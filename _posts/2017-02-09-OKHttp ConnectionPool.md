# OKHttp ConnectionPool
## OkHttpClient
成员final ConnectionPool connectionPool。一个OkHttpClient拥有一个ConnectionPool所有OkHttpClient发出的网络连接请求都可以复用连接池中存在的相同网络请求，节约了网络连接的创建开销，提高了网络访问速度。
## RetryAndFollowUpInterceptor
用OkHttpClient的ConnectionPool构建一个StreamAllocation传递到RealInterceptorChain构造函数中，供责任链的下一个环节使用。
```
streamAllocation = new StreamAllocation(
        client.connectionPool(), createAddress(request.url()), callStackTrace);
response = ((RealInterceptorChain) chain).proceed(request, streamAllocation, null, null);
```
## ConnectInterceptor
从连接池中寻找连接，返回连接。   
```
HttpCodec httpCodec = streamAllocation.newStream(client, doExtensiveHealthChecks);
```
## StreamAllocation
private RealConnection connection;   

```
RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
          writeTimeout, connectionRetryEnabled, doExtensiveHealthChecks);

```
新连接直接返回，recycled连接判断连接是否可用,如果可用直接返回，不可以用关闭（noNewStreams()）连接继续寻找可用连接。   

```
private RealConnection findHealthyConnection(int connectTimeout, int readTimeout,
      int writeTimeout, boolean connectionRetryEnabled, boolean doExtensiveHealthChecks)
      throws IOException {
    while (true) {
      RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout,
          connectionRetryEnabled);

      // If this is a brand new connection, we can skip the extensive health checks.
      synchronized (connectionPool) {
        if (candidate.successCount == 0) {
          return candidate;
        }
      }

      // Do a (potentially slow) check to confirm that the pooled connection is still good. If it
      // isn't, take it out of the pool and start again.
      if (!candidate.isHealthy(doExtensiveHealthChecks)) {
        noNewStreams();
        continue;
      }

      return candidate;
    }
  }

```
为StreamAllocation寻找成员RealConnection connection，从连接池寻找recycled连接，或者创建新连接，加入到连接池，赋值给connection。   

```
/**
   * Returns a connection to host a new stream. This prefers the existing connection if it exists,
   * then the pool, finally building a new connection.
   */
  private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
      boolean connectionRetryEnabled) throws IOException {
    Route selectedRoute;
    synchronized (connectionPool) {
      if (released) throw new IllegalStateException("released");
      if (codec != null) throw new IllegalStateException("codec != null");
      if (canceled) throw new IOException("Canceled");

      RealConnection allocatedConnection = this.connection;
      if (allocatedConnection != null && !allocatedConnection.noNewStreams) {
        return allocatedConnection;
      }

      // Attempt to get a connection from the pool.
      RealConnection pooledConnection = Internal.instance.get(connectionPool, address, this);
      if (pooledConnection != null) {
        this.connection = pooledConnection;
        return pooledConnection;
      }

      selectedRoute = route;
    }

    if (selectedRoute == null) {
      selectedRoute = routeSelector.next();
      synchronized (connectionPool) {
        route = selectedRoute;
        refusedStreamCount = 0;
      }
    }
    RealConnection newConnection = new RealConnection(selectedRoute);

    synchronized (connectionPool) {
      acquire(newConnection);
      Internal.instance.put(connectionPool, newConnection);
      this.connection = newConnection;
      if (canceled) throw new IOException("Canceled");
    }

    newConnection.connect(connectTimeout, readTimeout, writeTimeout, address.connectionSpecs(),
        connectionRetryEnabled);
    routeDatabase().connected(newConnection.route());

    return newConnection;
  }
```
## ConnectionPool
private final Deque<RealConnection> connections = new ArrayDeque<>();   
管理连接，寻找recycled连接，添加新建连接，删除不可用连接。   
寻找可用连接,为可用连接添加StreamAllocation（streamAllocation.acquire(connection)），连接复用。   

```
RealConnection get(Address address, StreamAllocation streamAllocation) {
    assert (Thread.holdsLock(this));
    for (RealConnection connection : connections) {
      if (connection.allocations.size() < connection.allocationLimit
          && address.equals(connection.route().address)
          && !connection.noNewStreams) {
        streamAllocation.acquire(connection);
        return connection;
      }
    }
    return null;
  }
```
添加新连接，Deque<RealConnection> 连接队列加入新连接。   

```
void put(RealConnection connection) {
    assert (Thread.holdsLock(this));
    if (!cleanupRunning) {
      cleanupRunning = true;
      executor.execute(cleanupRunnable);
    }
    connections.add(connection);
  }
```
遍历RealConnection，清理与RealConnection相关的leaked allocation。   
寻找RealConnection longestIdleConnection = null，当idle的RealConnection超过设置时，移除longestIdleConnection。pruneAndGetAllocationCount返回0表示连接是idle。   

```
long cleanup(long now) {
    int inUseConnectionCount = 0;
    int idleConnectionCount = 0;
    RealConnection longestIdleConnection = null;
    long longestIdleDurationNs = Long.MIN_VALUE;

    // Find either a connection to evict, or the time that the next eviction is due.
    synchronized (this) {
      for (Iterator<RealConnection> i = connections.iterator(); i.hasNext(); ) {
        RealConnection connection = i.next();

        // If the connection is in use, keep searching.
        if (pruneAndGetAllocationCount(connection, now) > 0) {
          inUseConnectionCount++;
          continue;
        }

        idleConnectionCount++;

        // If the connection is ready to be evicted, we're done.
        long idleDurationNs = now - connection.idleAtNanos;
        if (idleDurationNs > longestIdleDurationNs) {
          longestIdleDurationNs = idleDurationNs;
          longestIdleConnection = connection;
        }
      }

      if (longestIdleDurationNs >= this.keepAliveDurationNs
          || idleConnectionCount > this.maxIdleConnections) {
        // We've found a connection to evict. Remove it from the list, then close it below (outside
        // of the synchronized block).
        connections.remove(longestIdleConnection);
      } else if (idleConnectionCount > 0) {
        // A connection will be ready to evict soon.
        return keepAliveDurationNs - longestIdleDurationNs;
      } else if (inUseConnectionCount > 0) {
        // All connections are in use. It'll be at least the keep alive duration 'til we run again.
        return keepAliveDurationNs;
      } else {
        // No connections, idle or in use.
        cleanupRunning = false;
        return -1;
      }
    }

```
一个RealConnection有多个StreamAllocation，当发现某个StreamAllocation已经不被上层引用时，删除这个引用。   

```
private int pruneAndGetAllocationCount(RealConnection connection, long now) {
    List<Reference<StreamAllocation>> references = connection.allocations;
    for (int i = 0; i < references.size(); ) {
      Reference<StreamAllocation> reference = references.get(i);

      if (reference.get() != null) {
        i++;
        continue;
      }

      // We've discovered a leaked allocation. This is an application bug.
      StreamAllocation.StreamAllocationReference streamAllocRef =
          (StreamAllocation.StreamAllocationReference) reference;
      String message = "A connection to " + connection.route().address().url()
          + " was leaked. Did you forget to close a response body?";
      Platform.get().logCloseableLeak(message, streamAllocRef.callStackTrace);

      references.remove(i);
      connection.noNewStreams = true;

      // If this was the last allocation, the connection is eligible for immediate eviction.
      if (references.isEmpty()) {
        connection.idleAtNanos = now - keepAliveDurationNs;
        return 0;
      }
    }

    return references.size();
  }
```
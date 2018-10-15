## Okhttp获取响应过程

[TOC]

获取相应的过程利用了责任链模式

**责任链模式**:使多个对象都有机会处理同一个请求，从而避免请求的发送者和接收者之间的耦合关系。将这些对象连成一条链，并沿着这条链传递该请求，直到有一个对象处理它为止。

简单理解:流水线工人

```java
Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
  }
```



> 1. RetryAndFollowUpInterceptor

- streamAllocation

> 2. BridgeInterceptor

- 用来构建request的头部信息,添加gzip压缩.

- 负责将一个用户请求转化为能够进行网络传输的网络请求,
- 网络请求回来的响应Response转化为用户可以使用的Response

> 3. CacheInterceptor

缓存拦截器,缩短获取响应的时间

> 4. ConnectionInterceptor 

负责建立一个可用的链接

> 5. CallServicesInterceptor

负责将请求写进网络中,请求从网络中获取响应,返回上一层

### okhttp缓存机制

```java
public CacheInterceptor(InternalCache cache) {
    this.cache = cache;
}
//cache是通过okhttpClient builder 设置的,如果开发者没有设置cache 则为null
```

##### 缓存机制,使用了DiskLruCache

only-if-cached 表示不进行网络请求，完全只使用缓存，若缓存不命中，则返回503错误

```java
@Override public Response intercept(Chain chain) throws IOException {
    //创建CacheInterceptor的时候,会设置一个cache,cache是通过okhttpClient builder 设置的,如果开发者没有设置cache 则为null
    Response cacheCandidate = cache != null
        //在缓存中通过请求来获取response
        ? cache.get(chain.request())
        : null;

    long now = System.currentTimeMillis();
	//创建缓存策略
    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
    Request networkRequest = strategy.networkRequest;
    Response cacheResponse = strategy.cacheResponse;

    if (cache != null) {
      cache.trackResponse(strategy);
    }

    if (cacheCandidate != null && cacheResponse == null) {
        //释放资源
      closeQuietly(cacheCandidate.body()); // The cache candidate wasn't applicable. Close it.
    }

    // If we're forbidden from using the network and the cache is insufficient, fail.
    if (networkRequest == null && cacheResponse == null) {
        //如果网络不可用,并且没有设置缓存,则返回一个504错误的Response
      return new Response.Builder()
          .request(chain.request())
          .protocol(Protocol.HTTP_1_1)
          .code(504)
          .message("Unsatisfiable Request (only-if-cached)")
          .body(Util.EMPTY_RESPONSE)
          .sentRequestAtMillis(-1L)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build();
    }

    // If we don't need the network, we're done.
    if (networkRequest == null) {
      return cacheResponse.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build();
    }

    Response networkResponse = null;
    try {
      networkResponse = chain.proceed(networkRequest);
    } finally {
      // If we're crashing on I/O or otherwise, don't leak the cache body.
      if (networkResponse == null && cacheCandidate != null) {
        closeQuietly(cacheCandidate.body());
      }
    }

    // If we have a cache response too, then we're doing a conditional get.
    if (cacheResponse != null) {
      if (networkResponse.code() == HTTP_NOT_MODIFIED) {
        Response response = cacheResponse.newBuilder()
            .headers(combine(cacheResponse.headers(), networkResponse.headers()))
            .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
            .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build();
        networkResponse.body().close();

        // Update the cache after combining headers but before stripping the
        // Content-Encoding header (as performed by initContentStream()).
        cache.trackConditionalCacheHit();
        cache.update(cacheResponse, response);
        return response;
      } else {
        closeQuietly(cacheResponse.body());
      }
    }

    Response response = networkResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();

    if (cache != null) {
        //如果设置了缓存,则将response缓存
      if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
        // Offer this request to the cache.
          //调用cache 的put方法
        CacheRequest cacheRequest = cache.put(response);
        return cacheWritingResponse(cacheRequest, response);
      }

      if (HttpMethod.invalidatesCache(networkRequest.method())) {
        try {
          cache.remove(networkRequest);
        } catch (IOException ignored) {
          // The cache cannot be written.
        }
      }
    }

    return response;
  }
```

Cache的put方法

```java
@Nullable
    CacheRequest put(Response response) {
        String requestMethod = response.request().method();

        //检测该请求的缓存是否有效
        if (HttpMethod.invalidatesCache(response.request().method())) {
            try {
                remove(response.request());
            } catch (IOException ignored) {
                // The cache cannot be written.
            }
            return null;
        }
        //只缓存GET方法对应的请求
        if (!requestMethod.equals("GET")) {
            // Don't cache non-GET responses. We're technically allowed to cache
            // HEAD requests and some POST requests, but the complexity of doing
            // so is high and the benefit is low.
            return null;
        }

        if (HttpHeaders.hasVaryAll(response)) {
            return null;
        }

        //将对应的Response尽行存储
        Entry entry = new Entry(response);
        DiskLruCache.Editor editor = null;
        try {
            editor = cache.edit(key(response.request().url()));
            if (editor == null) {
                return null;
            }
            entry.writeTo(editor);
            return new CacheRequestImpl(editor);
        } catch (IOException e) {
            abortQuietly(editor);
            return null;
        }
    }
```

缓存机制总结:

1. okhttp缓存是在初始化okhttpClient时设置的,不设置是没有缓存的
2. CacheInterceptor在工作的过程中:

- 设置了缓存
  - 缓存中存在值
    - 调用Cache的get方法,判断当前请求的缓存是否有效,如果有效则返回对应缓存,否则返回null
  - 缓存中没有值
    - 转发到ConnectInterceptor中进一步处理Request
    - 处理ConnectInterceptor返回结果,通过Cache中的put方法,将response通过DiskLruCache结合okio将Response的相关header和body进行存储
- 没有设置缓存
  - 网络可用 : 转发到ConnectInterceptor中进一步处理Request
  - 网络不可用 : 直接返回504的错误

### 网络连接

okhttp中的网络连接由`RealConnection`来管理,`RealConnection`是`Connection`的实现类,代表链接Socket的链接.

1. 在建立过程中区别是建立隧道模式还是建立Socket模式
2. 当链接建立完成之后,需要建立协议,相关协议 http1、http1.1、http2、https(需要建立TLS层)

几个重要的字段:

- protocal:协议,当在建立协议的过程中,这个成员变量会被赋值,所以在okhttp中可以通过判断这个字段是否为null来判断链接是否建立成功
- noNewStream:这个变量是一个boolean的,当这个值为true时,标示这个这个链接为不可用,同时这个connection永远不会创建新的流
- souce/sink:在建立协议的过程中这两个值会被赋值,来负责与服务器之间的`I/O`操作,httpCode的读写,就是通过这两个变量
- idleAtNanos:long类型的值,当这个链接(Connection)空闲的时候,开始计时,当链接空闲超过5分钟的时候,该链接会从链接池中清理
- final List<Reference<StreamAllocation>> allocations:当前链接中多少个任务

### 连接池

okHttp中链接池是通过ConnectionPool来完成的,该类负责关联http的链接,以便减少网络请求的延迟,同一个address将共享同一个链接,从而实现了链接复用的目标

```java
public ConnectionPool() {
    this(5, 5, TimeUnit.MINUTES);
  }

  public ConnectionPool(int maxIdleConnections, long keepAliveDuration, TimeUnit timeUnit) {
    this.maxIdleConnections = maxIdleConnections;
    this.keepAliveDurationNs = timeUnit.toNanos(keepAliveDuration);

    // Put a floor on the keep alive duration, otherwise cleanup will spin loop.
    if (keepAliveDuration <= 0) {
      throw new IllegalArgumentException("keepAliveDuration <= 0: " + keepAliveDuration);
    }
  }
```

几个重要的字段:

- int maxIdleConnections:最大空闲的链接数为5个

- long keepAliveDurationNs:空闲链接存活的时间为5分钟,是通过构造方法来确定的

- Runnable cleanupRunnable:清理任务,当往链接队列中添加链接的时候,清理任务开始执行

-  Deque<RealConnection> connections:存储链接的队列,是通过数组实现的顺序队列

- Executor executor = new ThreadPoolExecutor(0 /* corePoolSize */,
  ​      Integer.MAX_VALUE /* maximumPoolSize */, 60L /* keepAliveTime */, TimeUnit.SECONDS,
  ​      new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp ConnectionPool", true));

  线程池,用来执行清理任务,他在注视中是这样描述的

  `Background threads are used to cleanup expired connections. There will be at most a single thread running per connection pool. The thread pool executor permits the pool itself to be garbage collected.`

  后台线程用来清理过期的链接(空闲时长超过5分钟&空闲链接数大于5个时空闲时间最长).每个线程池中最多只有一个线程,并且这个线程池允许自身被垃圾回收.

既然链接池,肯定有对应的get和put方法,先看get方法

```java
@Nullable RealConnection get(Address address, StreamAllocation streamAllocation, Route route) {
    //用来检测是否持有当前对象锁
    assert (Thread.holdsLock(this));
    //遍历整个链接队列
    for (RealConnection connection : connections) {
        //如果有链接满足条件,则使用该链接
      if (connection.isEligible(address, route)) {
        streamAllocation.acquire(connection, true);
        return connection;
      }
    }
    //没有符合条件的链接返回null
    return null;
  }
```

put方法也是比较简单

```java
 void put(RealConnection connection) {
    assert (Thread.holdsLock(this));
     //判断时候正在进行清理任务,当任务清理完成之后会重置
    if (!cleanupRunning) {
      cleanupRunning = true;
        //通过线程池执行清理任务
      executor.execute(cleanupRunnable);
    }
     //将新的链接添加到链接队列中
    connections.add(connection);
  }
```

#### 清理工作

```java
private final Runnable cleanupRunnable = new Runnable() {
    @Override public void run() {
      while (true) {
          //清理工作是通过cleanup方法来完成的,参数为当前时间的ns数值,用来计算空闲时间用
        long waitNanos = cleanup(System.nanoTime());
          //当需要等待的纳秒数为-1时结束清理任务,也就是说,当前链接池中没有链接数了
        if (waitNanos == -1) return;
		//计算需要等待的毫秒数,等待规定时间之后,再运行清理任务
        if (waitNanos > 0) {
          long waitMillis = waitNanos / 1000000L;
          waitNanos -= (waitMillis * 1000000L);
          synchronized (ConnectionPool.this) {
            try {
              ConnectionPool.this.wait(waitMillis, (int) waitNanos);
            } catch (InterruptedException ignored) {
            }
          }
        }
      }
    }
  };
```

那具体的清理工作是放在了cleanup方法中了

```java
  long cleanup(long now) {
      //当前正在执行请求的链接数量
    int inUseConnectionCount = 0;
      //空闲的链接数量
    int idleConnectionCount = 0;
      //耗时最长的链接
    RealConnection longestIdleConnection = null;
      //初始化空闲时间
    long longestIdleDurationNs = Long.MIN_VALUE;
    synchronized (this) {
      for (Iterator<RealConnection> i = connections.iterator(); i.hasNext(); ) {
        RealConnection connection = i.next();
          
          //当该链接正在使用时,inUseConnectionCount累加,跳过本次循环后继续往下执行
        if (pruneAndGetAllocationCount(connection, now) > 0) {
          inUseConnectionCount++;
          continue;
        }
		//该链接空闲
        idleConnectionCount++;
          
		//计算该空闲链接的空闲时间
        long idleDurationNs = now - connection.idleAtNanos;
        if (idleDurationNs > longestIdleDurationNs) {
            //对最长空闲时间重新赋值,同时表示对应的链接
          longestIdleDurationNs = idleDurationNs;
          longestIdleConnection = connection;
        }
      }

      if (longestIdleDurationNs >= this.keepAliveDurationNs
          || idleConnectionCount > this.maxIdleConnections) {
          //如果链接数量大于5个或者最长空闲时间超过5分钟,那就移除对应的链接
        connections.remove(longestIdleConnection);
      } else if (idleConnectionCount > 0) {
        //链接池中存在空闲链接,但是这些空闲的链接数空闲时间均没有超过5分钟,那就返回相差的时间,等待下次清理
        return keepAliveDurationNs - longestIdleDurationNs;
      } else if (inUseConnectionCount > 0) {
		//所有的链接都在使用的中,那下次清理任务的周期是在5分钟之后
        return keepAliveDurationNs;
      } else {
        //链接队列中没有链接了,那就停止清理任务
        cleanupRunning = false;
        return -1;
      }
    }

    closeQuietly(longestIdleConnection.socket());

    // Cleanup again immediately.
    return 0;
  }
```

### StreamAllocation

负责统筹管理链接(RealConnection)、流(HttpCode)、请求(Call),在ConnecInterecptor中调用newStream的过程中创建流和链接

StreamAllocation既然可以将三者管理起来,肯定具有创建链接创建流的能力,通过RealConnection分析可以知道,流是通过RealConnection的newCodec来创建的,那么创建链接就显的格外重要.具体体现在了findConnection方法中

```java
private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
      int pingIntervalMillis, boolean connectionRetryEnabled) throws IOException {
    boolean foundPooledConnection = false;
    RealConnection result = null;
    Route selectedRoute = null;
    Connection releasedConnection;
    Socket toClose;
    synchronized (connectionPool) {
      if (released) throw new IllegalStateException("released");
      if (codec != null) throw new IllegalStateException("codec != null");
      if (canceled) throw new IOException("Canceled");

      // Attempt to use an already-allocated connection. We need to be careful here because our
      // already-allocated connection may have been restricted from creating new streams.
      //如果当前的链接可以复用,那我们就用当前的链接
      releasedConnection = this.connection;
      toClose = releaseIfNoNewStreams();
      if (this.connection != null) {
        // We had an already-allocated connection and it's good.
        result = this.connection;
        releasedConnection = null;
      }
      if (!reportedAcquired) {
        // If the connection was never reported acquired, don't report it as released!
        releasedConnection = null;
      }

      if (result == null) {
        // Attempt to get a connection from the pool.
          //如果当前链接不可用,我们就链接池中取出
        Internal.instance.get(connectionPool, address, this, null);
        if (connection != null) {
          foundPooledConnection = true;
          result = connection;
        } else {
          selectedRoute = route;
        }
      }
    }
    closeQuietly(toClose);

    if (releasedConnection != null) {
      eventListener.connectionReleased(call, releasedConnection);
    }
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result);
    }
    if (result != null) {
        //如果在当前链接可用或者链接池中有可用链接
      // If we found an already-allocated or pooled connection, we're done.
      return result;
    }

    // If we need a route selection, make one. This is a blocking operation.
    boolean newRouteSelection = false;
    if (selectedRoute == null && (routeSelection == null || !routeSelection.hasNext())) {
      newRouteSelection = true;
      routeSelection = routeSelector.next();
    }

    synchronized (connectionPool) {
      if (canceled) throw new IOException("Canceled");

      if (newRouteSelection) {
        // Now that we have a set of IP addresses, make another attempt at getting a connection from
        // the pool. This could match due to connection coalescing.
        List<Route> routes = routeSelection.getAll();
        for (int i = 0, size = routes.size(); i < size; i++) {
          Route route = routes.get(i);
          Internal.instance.get(connectionPool, address, this, route);
          if (connection != null) {
            foundPooledConnection = true;
            result = connection;
            this.route = route;
            break;
          }
        }
      }

      if (!foundPooledConnection) {
        if (selectedRoute == null) {
          selectedRoute = routeSelection.next();
        }

        // Create a connection and assign it to this allocation immediately. This makes it possible
        // for an asynchronous cancel() to interrupt the handshake we're about to do.
        route = selectedRoute;
        refusedStreamCount = 0;
        result = new RealConnection(connectionPool, selectedRoute);
        acquire(result, false);
      }
    }

    // If we found a pooled connection on the 2nd time around, we're done.
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result);
      return result;
    }

    // Do TCP + TLS handshakes. This is a blocking operation.
    result.connect(connectTimeout, readTimeout, writeTimeout, pingIntervalMillis,
        connectionRetryEnabled, call, eventListener);
    routeDatabase().connected(result.route());

    Socket socket = null;
    synchronized (connectionPool) {
      reportedAcquired = true;

      // Pool the connection.
      Internal.instance.put(connectionPool, result);

      // If another multiplexed connection to the same address was created concurrently, then
      // release this connection and acquire that one.
      if (result.isMultiplexed()) {
        socket = Internal.instance.deduplicate(connectionPool, address, this);
        result = connection;
      }
    }
    closeQuietly(socket);

    eventListener.connectionAcquired(call, result);
    return result;
  }
```

1、先找是否有已经存在的连接，如果有已经存在的连接，并且可以使用(!noNewStreams)则直接返回。
2、根据已知的address在connectionPool里面找，如果有连接，则返回
3、更换路由，更换线路，在connectionPool里面再次查找，如果有则返回。
4、如果以上条件都不满足则直接new一个RealConnection出来
5、new出来的RealConnection通过acquire关联到connection.allocations上
6、做去重判断，如果有重复的socket则关闭

```java
/**
   * Use this allocation to hold {@code connection}. Each call to this must be paired with a call to
   * {@link #release} on the same connection.
   */
  public void acquire(RealConnection connection, boolean reportedAcquired) {
    assert (Thread.holdsLock(connectionPool));
    if (this.connection != null) throw new IllegalStateException();

    this.connection = connection;
    this.reportedAcquired = reportedAcquired;
      //对应的链接添加引用计数
    connection.allocations.add(new StreamAllocationReference(this, callStackTrace));
  }

  /** Remove this allocation from the connection's list of allocations. */
  private void release(RealConnection connection) {
    for (int i = 0, size = connection.allocations.size(); i < size; i++) {
      Reference<StreamAllocation> reference = connection.allocations.get(i);
      if (reference.get() == this) {
          //减少对应的应用计数
        connection.allocations.remove(i);
        return;
      }
    }
    throw new IllegalStateException();
  }
```


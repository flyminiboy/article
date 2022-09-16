## 手撕Android开源库-okhttp源码解析（拦截,器）

一图镇楼

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9weQJL8tibVaahtn2Pc7aiaa6D3WgmjmvJeicSia2A1ZbLfypgPIe2w6RIUKCGvryEfNpmYicVjwhNPSV7Q/0?wx_fmt=png)

在上篇文章 [手撕Android开源库-okhttp（整体流程分析）](https://mp.weixin.qq.com/s?__biz=Mzg3NzU3OTQwOA==&amp;mid=2247484012&amp;idx=1&amp;sn=56a5edd23ebcba79808a273ccc6e6f74&amp;chksm=cf219f93f85616858a2e59f0b68ef1a2b8e30b8c7e0f3100dcadcd8dec7787e1a9ecb68f6a42&token=2088204321&lang=zh_CN#rd) 我们介绍了**OKhttp** 了整体流程，并且说了网络的核心流程在 **拦截器** 里面实现。今天我们就来分析这部分代码，由于内容太多，决定将**路由和连接池**这块内容单独再写一篇😶😶😶😶😶😶。

### 源码版本

* OKhttp 4.4.1

### 背景知识

我们先来简单的补充一下网络相关的知识。方便我们下面做代码分析。

#### 网络分层


应用层（http，https，quic，ftp）
传输层（TCP，UDP）
网络层（IP）
数据链路层
物理层

什么是http协议和HTTPS协议呢

HTTP
基于TCP协议
请求行
请求头
请求体

http1.1

http2.0

TCP和UDP

Socket 又是什么

客户端
服务端




### 连接

#### 小tip

* 底层I/O操作由 **Okio** 库提供。本系列不会对 **Okio** 做介绍，感兴趣的自己去了解吧。
* 底层I/O操作由 **Okio** 库提供。本系列不会对 **Okio** 做介绍，感兴趣的自己去了解吧。
* 底层I/O操作由 **Okio** 库提供。本系列不会对 **Okio** 做介绍，感兴趣的自己去了解吧。
* 简单说一句，**Okio** 定义了自己的一套继承链，**Source对应InputStream， Sink对应OutputStream**
* 简单说一句，**Okio** 定义了自己的一套继承链，**Source对应InputStream， Sink对应OutputStream**
* 简单说一句，**Okio** 定义了自己的一套继承链，**Source对应InputStream， Sink对应OutputStream**

#### RealConnection

负责底层 socket 连接。

```

  /** The low-level TCP socket. */
  // 一个低级别的负责 TCP 连接的 socket
  private var rawSocket: Socket? = null

  /**
   * The application layer socket. Either an [SSLSocket] layered over [rawSocket], or [rawSocket]
   * itself if this connection does not use SSL.
   */
   // 在 rawSocket 上层嵌套 SSLSocket
  private var socket: Socket? = null
  // TLS 
  private var handshake: Handshake? = null
  // 协议
  private var protocol: Protocol? = null
  // HTTP2 连接处理
  private var http2Connection: Http2Connection? = null
  // 输入输出流(就这么理解吧)
  private var source: BufferedSource? = null
  private var sink: BufferedSink? = null
```

#### Exchange

传输单个 HTTP 请求和响应对。 算是对 ExchangeCodec 的层级上进行了包装，主要是进行了事件的监听回调。我们在做网络监控设置 `EventListener` 。

#### ExchangeCodec

这是一个接口，约定行为编码http请求，解码http响应。有俩个实现类，分别对应**http1.1协议**和**http2.0协议**。

1. `Http1ExchangeCodec`

> 一个socket连接可以用来发送 HTTP/1.1 消息
> 
> 1. 发送请求头
> 
> 2.  打开一个 sink 写请求体
> 
> 3. 写入并关闭 sink
> 
> 4. 读取响应头
> 
> 5. 打开一个 source 读响应体
> 
> 6. 读取并关闭 source

2. `Http2ExchangeCodec` 

> 使用 HTTP/2 帧对请求和响应进行编码。

* Http2Connection
* Http2Stream

	* FramingSource
	* FramingSink

* Http2Writer
> 写入 HTTP/2 传输帧。
* Http2Reader
> Reads HTTP/2 transport frames.

#### ExchangeFinder

#### RealConnectionPool

连接池，后面会专门再写一篇进行介绍。

在上一篇的流程分析，我们说只有3.5个核心类。这篇😭😭😭😭😭😭。

### 源码分析

#### RealCall#getResponseWithInterceptorChain

```kotlin
  @Throws(IOException::class)
  internal fun getResponseWithInterceptorChain(): Response {
    // Build a full stack of interceptors.
    // 添加拦截器
    val interceptors = mutableListOf<Interceptor>()
    interceptors += client.interceptors
    interceptors += RetryAndFollowUpInterceptor(client)
    interceptors += BridgeInterceptor(client.cookieJar)
    interceptors += CacheInterceptor(client.cache)
    interceptors += ConnectInterceptor
    if (!forWebSocket) {
      interceptors += client.networkInterceptors
    }
    interceptors += CallServerInterceptor(forWebSocket)

    // 构建 RealInterceptorChain 对象
    val chain = RealInterceptorChain(
        call = this,
        interceptors = interceptors,
        index = 0,
        exchange = null,
        request = originalRequest,
        connectTimeoutMillis = client.connectTimeoutMillis,
        readTimeoutMillis = client.readTimeoutMillis,
        writeTimeoutMillis = client.writeTimeoutMillis
    )

    var calledNoMoreExchanges = false
    try {
      val response = chain.proceed(originalRequest)
      if (isCanceled()) {
        response.closeQuietly()
        throw IOException("Canceled")
      }
      return response
    } catch (e: IOException) {
      calledNoMoreExchanges = true
      throw noMoreExchanges(e) as Throwable
    } finally {
      if (!calledNoMoreExchanges) {
        noMoreExchanges(null)
      }
    }
  }
```

默认添加了五个拦截器

* `RetryAndFollowUpInterceptor` 失败重试，重定向拦截器
* `BridgeInterceptor` 从应用程序到网络的桥梁。根据用户请求构建网络请求，根据网络响应构建用户响应。
* `CacheInterceptor` 缓存相关的过滤器，负责读取缓存直接返回、更新缓存
* `ConnectInterceptor` 打开到目标服务器的连接。开始真正的网络请求。
* `CallServerInterceptor` 对服务进行网络调用。（写请求，获取响应）

**所有的网络请求底层逻辑都划分到这几个拦截器里面了。**

和之前的不同，这次我们从里向外发射的分析。

#### RealConnection#connect

```
  fun connect(
    connectTimeout: Int,
    readTimeout: Int,
    writeTimeout: Int,
    pingIntervalMillis: Int,
    connectionRetryEnabled: Boolean,
    call: Call,
    eventListener: EventListener
  ) {
    check(protocol == null) { "already connected" }

    var routeException: RouteException? = null
    val connectionSpecs = route.address.connectionSpecs
    val connectionSpecSelector = ConnectionSpecSelector(connectionSpecs)

    if (route.address.sslSocketFactory == null) {
      if (ConnectionSpec.CLEARTEXT !in connectionSpecs) {
        throw RouteException(UnknownServiceException(
            "CLEARTEXT communication not enabled for client"))
      }
      val host = route.address.url.host
      if (!Platform.get().isCleartextTrafficPermitted(host)) {
        throw RouteException(UnknownServiceException(
            "CLEARTEXT communication to $host not permitted by network security policy"))
      }
    } else {
      if (Protocol.H2_PRIOR_KNOWLEDGE in route.address.protocols) {
        throw RouteException(UnknownServiceException(
            "H2_PRIOR_KNOWLEDGE cannot be used with HTTPS"))
      }
    }

    while (true) {
      try {
        if (route.requiresTunnel()) {
        // 隧道代理
          connectTunnel(connectTimeout, readTimeout, writeTimeout, call, eventListener)
          if (rawSocket == null) {
            // We were unable to connect the tunnel but properly closed down our resources.
            break
          }
        } else {
        // 连接
          connectSocket(connectTimeout, readTimeout, call, eventListener)
        }
        // 构建协议
        establishProtocol(connectionSpecSelector, pingIntervalMillis, call, eventListener)
        eventListener.connectEnd(call, route.socketAddress, route.proxy, protocol)
        break
      } catch (e: IOException) {
        socket?.closeQuietly()
        rawSocket?.closeQuietly()
        socket = null
        rawSocket = null
        source = null
        sink = null
        handshake = null
        protocol = null
        http2Connection = null
        allocationLimit = 1

        eventListener.connectFailed(call, route.socketAddress, route.proxy, null, e)

        if (routeException == null) {
          routeException = RouteException(e)
        } else {
          routeException.addConnectException(e)
        }

        if (!connectionRetryEnabled || !connectionSpecSelector.connectionFailed(e)) {
          throw routeException
        }
      }
    }

    if (route.requiresTunnel() && rawSocket == null) {
      throw RouteException(ProtocolException(
          "Too many tunnel connections attempted: $MAX_TUNNEL_ATTEMPTS"))
    }

    idleAtNs = System.nanoTime()
  }
```

#### CallServerInterceptor



#### RetryAndFollowUpInterceptor 

上图


#### ExchangeFinder#findConnection

```

  private fun findConnection(
    connectTimeout: Int,
    readTimeout: Int,
    writeTimeout: Int,
    pingIntervalMillis: Int,
    connectionRetryEnabled: Boolean
  ): RealConnection {
    var foundPooledConnection = false
    var result: RealConnection? = null
    var selectedRoute: Route? = null
    var releasedConnection: RealConnection?
    val toClose: Socket?
    synchronized(connectionPool) {
      if (call.isCanceled()) throw IOException("Canceled")

      val callConnection = call.connection // changes within this overall method
      releasedConnection = callConnection
      toClose = if (callConnection != null && (callConnection.noNewExchanges ||
              !sameHostAndPort(callConnection.route().address.url))) {
        call.releaseConnectionNoEvents()
      } else {
        null
      }

      if (call.connection != null) {
        // We had an already-allocated connection and it's good.
        result = call.connection
        releasedConnection = null
      }

      if (result == null) {
        // The connection hasn't had any problems for this call.
        refusedStreamCount = 0
        connectionShutdownCount = 0
        otherFailureCount = 0

        // Attempt to get a connection from the pool.
        if (connectionPool.callAcquirePooledConnection(address, call, null, false)) {
          foundPooledConnection = true
          result = call.connection
        } else if (nextRouteToTry != null) {
          selectedRoute = nextRouteToTry
          nextRouteToTry = null
        }
      }
    }
    toClose?.closeQuietly()

    if (releasedConnection != null) {
      eventListener.connectionReleased(call, releasedConnection!!)
    }
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result!!)
    }
    if (result != null) {
      // If we found an already-allocated or pooled connection, we're done.
      return result!!
    }

    // If we need a route selection, make one. This is a blocking operation.
    var newRouteSelection = false
    if (selectedRoute == null && (routeSelection == null || !routeSelection!!.hasNext())) {
      var localRouteSelector = routeSelector
      if (localRouteSelector == null) {
        localRouteSelector = RouteSelector(address, call.client.routeDatabase, call, eventListener)
        this.routeSelector = localRouteSelector
      }
      newRouteSelection = true
      routeSelection = localRouteSelector.next()
    }

    var routes: List<Route>? = null
    synchronized(connectionPool) {
      if (call.isCanceled()) throw IOException("Canceled")

      if (newRouteSelection) {
        // Now that we have a set of IP addresses, make another attempt at getting a connection from
        // the pool. This could match due to connection coalescing.
        routes = routeSelection!!.routes
        if (connectionPool.callAcquirePooledConnection(address, call, routes, false)) {
          foundPooledConnection = true
          result = call.connection
        }
      }

      if (!foundPooledConnection) {
        if (selectedRoute == null) {
          selectedRoute = routeSelection!!.next()
        }

        // Create a connection and assign it to this allocation immediately. This makes it possible
        // for an asynchronous cancel() to interrupt the handshake we're about to do.
        result = RealConnection(connectionPool, selectedRoute!!)
        connectingConnection = result
      }
    }

    // If we found a pooled connection on the 2nd time around, we're done.
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result!!)
      return result!!
    }

    // Do TCP + TLS handshakes. This is a blocking operation.
    result!!.connect(
        connectTimeout,
        readTimeout,
        writeTimeout,
        pingIntervalMillis,
        connectionRetryEnabled,
        call,
        eventListener
    )
    call.client.routeDatabase.connected(result!!.route())

    var socket: Socket? = null
    synchronized(connectionPool) {
      connectingConnection = null
      // Last attempt at connection coalescing, which only occurs if we attempted multiple
      // concurrent connections to the same host.
      if (connectionPool.callAcquirePooledConnection(address, call, routes, true)) {
        // We lost the race! Close the connection we created and return the pooled connection.
        result!!.noNewExchanges = true
        socket = result!!.socket()
        result = call.connection

        // It's possible for us to obtain a coalesced connection that is immediately unhealthy. In
        // that case we will retry the route we just successfully connected with.
        nextRouteToTry = selectedRoute
      } else {
        connectionPool.put(result!!)
        call.acquireConnectionNoEvents(result!!)
      }
    }
    socket?.closeQuietly()

    eventListener.connectionAcquired(call, result!!)
    return result!!
  }

```



* callAcquirePooledConnection

```

```

* initExchange

```

```



### 参考

图解HTTP

[https://cloud.tencent.com/developer/article/1667342](https://cloud.tencent.com/developer/article/1667342)

[https://cloud.tencent.com/developer/article/1667344](https://cloud.tencent.com/developer/article/1667344)

[https://segmentfault.com/a/1190000022145519](https://segmentfault.com/a/1190000022145519)

极客时间-趣谈网络协议
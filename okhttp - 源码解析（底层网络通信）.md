## 手撕Android开源库-okhttp源码解析（拦截,器）

一图镇楼

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9weQJL8tibVaahtn2Pc7aiaa6D3WgmjmvJeicSia2A1ZbLfypgPIe2w6RIUKCGvryEfNpmYicVjwhNPSV7Q/0?wx_fmt=png)

在上篇文章 [手撕Android开源库-okhttp（整体流程分析）](https://mp.weixin.qq.com/s?__biz=Mzg3NzU3OTQwOA==&amp;mid=2247484012&amp;idx=1&amp;sn=56a5edd23ebcba79808a273ccc6e6f74&amp;chksm=cf219f93f85616858a2e59f0b68ef1a2b8e30b8c7e0f3100dcadcd8dec7787e1a9ecb68f6a42&token=2088204321&lang=zh_CN#rd) 我们介绍了**OKhttp** 了整体流程，并且说了网络的核心流程在 **拦截器** 里面实现。今天我们就来分析这部分代码，由于内容太多，决定将**路由和连接池**这块内容单独再写一篇😶😶😶😶😶😶。

### 源码版本

* OKhttp 4.4.1

### 需要了解的必备知识

我们这里只是简单的介绍需要那些基础知识，具体的知识大家自行学习吧，主要是面太大，不是一俩句话可以说清楚的。当然了主要是小弟学艺不精，不敢随意造次。

#### 网络分层

|  分层   | 代表  |
|  ----  | ----  |
| 应用层  | http，https，quic，ftp |
| 传输层  | TCP，UDP |
| 网络层  |  |
| 数据链路层  |  |
| 物理层  |  |

我们主要是涉及上面俩层，**应用层和传输层**。

#### 什么是HTTP协议和HTTPS协议

HTTP 是基于 TCP 协议的。

HTTPS = HTTP + SSL/TLS，通过 SSL证书来验证服务器的身份，并客户端和服务器之间的通信进行加密。

这里面还涉及非对称加密，对称加密，数字证书（包含公钥）。

公钥私钥主要用于传输对称加密的秘钥，而真正的双方大数据量的通信都是通过对称加密进行的。

#### HTTP版本

* HTTP 1.1

1. 引入了持久连接 keep-alive
2. 加入了管道机制

* HTTP 2.0

1. HTTP 2.0 会对 HTTP 的头进行一定的压缩，将原来每次都要携带的大量 key value 在两端建立一个索引表，对相同的头只发送索引表中的索引
2. HTTP 2.0 协议将一个 TCP 的连接中，切分成多个流，每个流都有自己的 ID
3. HTTP 2.0 还将所有的传输信息分割为更小的消息和帧，并对它们采用二进制格式编码


#### TCP和UDP

* TCP

一种面向连接的、可靠的、基于字节流的传输层通信协议

1. TCP 的三次握手
2. TCP 四次挥手

* UDP

是一种无连接的协议、无状态服务。

#### Socket 编程

前面说的都是协议，是一种规范，约定。Socket 就是将这种规范进行了实现。

Socket 编程进行的是端到端的通信。需要客户端和服务端。

盗图俩张。

* TCP Socket 交互

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9weiaKzAicj3VJ8pMxRde60O0Ntxny2TlfHZfcCFnjxp55E0XmialiaeESdyyMlynjBj4WpFZLdkrnhktg/0?wx_fmt=png)

* UDP Socket 交互

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9weiaKzAicj3VJ8pMxRde60O0NdFdYWjvYdH4jEZyu9GdAhIWDabDUS687pKkTicAHYibiarI2lWbt3aycA/0?wx_fmt=png)

### 本篇关键类介绍

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

传输单个 HTTP 请求和响应对。 算是在 ExchangeCodec 的层级上进行了包装，主要是进行了事件的监听回调。我们在做网络监控设置 `EventListener` 。

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

查找一个合适的连接。使用一下策略

> 1. 如果当前调用已经有一个可以满足请求的连接，则使用它。
> 
> 2. 如果连接池（RealConnectionPool）中有可以满足请求的连接，则使用它
> 
> 3. 如果没有现有连接，构建路由（可能需要 DNS 查找-阻塞操作）并再次执行步骤2，如果还是没有建立新连接

#### RealConnectionPool

连接池，后面会专门再写一篇进行介绍。

#### Route

路由，主要是封装了 `Address` `Proxy` `InetSocketAddress` ，后面会专门再写一篇进行介绍。

#### RouteSelector

路由选择器，后面会专门再写一篇进行介绍。

在上一篇的流程分析，我们说只有3.5个核心类。这篇😭😭😭😭😭😭。

### 源码分析

#### 小tip

* 底层I/O操作由 **Okio** 库提供。本系列不会对 **Okio** 做介绍，感兴趣的自己去了解吧。
* 底层I/O操作由 **Okio** 库提供。本系列不会对 **Okio** 做介绍，感兴趣的自己去了解吧。
* 底层I/O操作由 **Okio** 库提供。本系列不会对 **Okio** 做介绍，感兴趣的自己去了解吧。
* 简单说一句，**Okio** 定义了自己的一套继承链，**Source对应InputStream， Sink对应OutputStream**
* 简单说一句，**Okio** 定义了自己的一套继承链，**Source对应InputStream， Sink对应OutputStream**
* 简单说一句，**Okio** 定义了自己的一套继承链，**Source对应InputStream， Sink对应OutputStream**

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

#### CallServerInterceptor

这个没啥好说的。直接通过 `Exchange` 进行 I/O 操作。写请求，读响应。

#### ConnectInterceptor

```
object ConnectInterceptor : Interceptor {
  @Throws(IOException::class)
  override fun intercept(chain: Interceptor.Chain): Response {
    val realChain = chain as RealInterceptorChain
    // 核心代码
    val exchange = realChain.call.initExchange(chain)
    val connectedChain = realChain.copy(exchange = exchange)
    return connectedChain.proceed(realChain.request)
  }
}
```

😂😂😂😂😂😂，这个真正执行网络操作的拦截，只有这么几行。😄😄😄😄😄。

调用链比较多，但是逻辑比较简单我们直接贴一个图来说明了。

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9weiaKzAicj3VJ8pMxRde60O0NlUhaeKxFDzic6aLaCiaw5bqFRZoWTxVIGaGtMaxcV8rIndKiaIG6Jc86A/0?wx_fmt=png)

核心逻辑都在 **ExchangeFinder#findConnection** 里面。这部分代码主要是包括路由逻辑和连接池连接复用逻辑。我们在下一篇里面详细说明。

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

	// 得到当前调用的连接
      val callConnection = call.connection // changes within this overall method
      releasedConnection = callConnection
      // 判断连接是否可用
      toClose = if (callConnection != null && (callConnection.noNewExchanges ||
              !sameHostAndPort(callConnection.route().address.url))) {
              // 不可用进行连接释放 call.connection = null
        call.releaseConnectionNoEvents()
      } else {
        null
      }

      if (call.connection != null) {
        // 1 当前连接可用
        result = call.connection
        releasedConnection = null
      }

      if (result == null) {
        // The connection hasn't had any problems for this call.
        refusedStreamCount = 0
        connectionShutdownCount = 0
        otherFailureCount = 0

        // 2 尝试从连接池里面获取可用连接
        if (connectionPool.callAcquirePooledConnection(address, call, null, false)) {
          foundPooledConnection = true
          result = call.connection
        } else if (nextRouteToTry != null) {
          selectedRoute = nextRouteToTry
          nextRouteToTry = null
        }
      }
    }
    // 省略代码...
    if (result != null) {
      //如果我们找到一个已经分配或池化的连接，直接返回
      return result!!
    }

    // 3.构建新的路由选择器
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
        // 再次尝试从池中获取连接
        routes = routeSelection!!.routes
        if (connectionPool.callAcquirePooledConnection(address, call, routes, false)) {
          foundPooledConnection = true
          result = call.connection
        }
      }

	// 没有找到可用连接
      if (!foundPooledConnection) {
        if (selectedRoute == null) {
          selectedRoute = routeSelection!!.next()
        }

        // 新建连接
        result = RealConnection(connectionPool, selectedRoute!!)
        connectingConnection = result
      }
    }

    // 如果我们找到一个已经池化的连接，直接返回
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result!!)
      return result!!
    }

    
    // 进行socket （TCP+TLS）连接
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
      	  // 符合条件的连接放到连接池
        connectionPool.put(result!!)
        call.acquireConnectionNoEvents(result!!)
      }
    }
    socket?.closeQuietly()

    eventListener.connectionAcquired(call, result!!)
    return result!!
  }
```

这个块代码更多的分析我们会在下一篇路由和连接池里面做分析。这里大家只需要知道里面的流程即可。

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
    
    // 省略代码...

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
        // 连接  socket 
        // 判断是否设置代理
        //  不设置    public Socket createSocket() {
        //          return new Socket();
        //      } 
        //  设置 Socket(proxy)
        //  socket.connect(address, connectTimeout)
          connectSocket(connectTimeout, readTimeout, call, eventListener)
        }
        // 建立协议 主要是干俩个事，
        // 1. http2
        // 1. TLS
        establishProtocol(connectionSpecSelector, pingIntervalMillis, call, eventListener)
        eventListener.connectEnd(call, route.socketAddress, route.proxy, protocol)
        break
      } catch (e: IOException) {
        // 省略代码...
      }
    }

	// 省略代码...
   
  }
```

最后会根据不同的协议来构建不同的 `ExchangeCodec` 

```
  internal fun newCodec(client: OkHttpClient, chain: RealInterceptorChain): ExchangeCodec {
    val socket = this.socket!!
    val source = this.source!!
    val sink = this.sink!!
    val http2Connection = this.http2Connection

    return if (http2Connection != null) {
      Http2ExchangeCodec(client, this, chain, http2Connection)
    } else {
      socket.soTimeout = chain.readTimeoutMillis()
      source.timeout().timeout(chain.readTimeoutMillis.toLong(), MILLISECONDS)
      sink.timeout().timeout(chain.writeTimeoutMillis.toLong(), MILLISECONDS)
      Http1ExchangeCodec(client, this, source, sink)
    }
  }
```

代码很简单了就没有说明的必要了。

#### CacheInterceptor 和 BridgeInterceptor

这俩个也比较简单，不再说明了。

`BridgeInterceptor` 这个算是一个翻译器吧，把我们配置的请求，翻译为网络底层传输的格式；将网络返回的数据，翻译成我们需要的响应。所以起到了一个桥接的作用哈。

#### RetryAndFollowUpInterceptor 

同样我们贴个图看看。打倒一切纸老虎。

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9weiaKzAicj3VJ8pMxRde60O0NSvdklbHtAyE7OqtLNNEyzJUKSJxPHAnbwibrDJh6jNF5twmBTq4ibY6w/0?wx_fmt=png)


所以呢，这里面就是一堆判断。但是核心就俩个

1. 异常是否重试
2. 判断响应码是否重定向

```
  override fun intercept(chain: Interceptor.Chain): Response {
    val realChain = chain as RealInterceptorChain
    var request = chain.request
    val call = realChain.call
    var followUpCount = 0
    var priorResponse: Response? = null
    var newExchangeFinder = true
    while (true) { // 这是一个死循环
    	// 本质就一个作用，构建 ExchangeFinder 处在死循环中，所以只有在第一次才会构建。
      call.enterNetworkInterceptorExchange(request, newExchangeFinder)

      var response: Response
      var closeActiveExchange = true
      try {
        if (call.isCanceled()) {
          throw IOException("Canceled")
        }

        try {
          response = realChain.proceed(request)
          newExchangeFinder = true
        } catch (e: RouteException) {
          // 单个路由连接出错，进行恢复操作
          if (!recover(e.lastConnectException, call, request, requestSendStarted = false)) {
            throw e.firstConnectException
          }
          // 这个是控制是否构建 ExchangeFinder 的标记
          newExchangeFinder = false
          continue // 直接开始下一次循环==》重试开始 这个地方理解好
        } catch (e: IOException) {
          // 与服务通信失败，进行重试
          if (!recover(e, call, request, requestSendStarted = e !is ConnectionShutdownException)) {
            throw e
          }
          // 这个是控制是否构建 ExchangeFinder 的标记
          newExchangeFinder = false
          continue // 直接开始下一次循环==》重试开始
        }

        // 省略代码...
        
        	  // 这个里面会处理身份验证，重定向，请求超时
	        val followUp = followUpRequest(response, exchange)
	        
      		 // 省略代码...	  
      } finally {
        call.exitNetworkInterceptorExchange(closeActiveExchange)
      }
    }
  }
```

* 失败重试

```
  private fun recover(
    e: IOException,
    call: RealCall,
    userRequest: Request,
    requestSendStarted: Boolean
  ): Boolean {
    // 禁止失败重试
    if (!client.retryOnConnectionFailure) return false

    // 请求已经开始并且是只期望一次的调用
    if (requestSendStarted && requestIsOneShot(e, userRequest)) return false

    // 不可恢复，致命异常
    // 协议问题，不可恢复
    // 非超时中断异常
    // 如果问题是来自 X509TrustManager 的 CertificateException 
    // 证书异常
    if (!isRecoverable(e, requestSendStarted)) return false

    // 没有更多的路由进行重试尝试
    if (!call.retryAfterFailure()) return false

    // 开始恢复
    return true
  }
```


* 截取部分代码 `followUpRequest` - 重定向

```
      HTTP_PERM_REDIRECT, HTTP_TEMP_REDIRECT -> {
        // "If the 307 or 308 status code is received in response to a request other than GET
        // or HEAD, the user agent MUST NOT automatically redirect the request"
        if (method != "GET" && method != "HEAD") {
          return null
        }
        // 构建重定向请求
        return buildRedirectRequest(userResponse, method)
      }
		// 300 301 302 303 
      HTTP_MULT_CHOICE, HTTP_MOVED_PERM, HTTP_MOVED_TEMP, HTTP_SEE_OTHER -> {
      	  // 构建重定向请求
        return buildRedirectRequest(userResponse, method)
      }
```

**这个里面一定不要忽视最外层是一个 while(true) 的循环模式啊，所以重试和重定向，不需要做其他的，代码自然会执行。**

### 参考

图解HTTP

[https://cloud.tencent.com/developer/article/1667342](https://cloud.tencent.com/developer/article/1667342)

[https://cloud.tencent.com/developer/article/1667344](https://cloud.tencent.com/developer/article/1667344)

[https://segmentfault.com/a/1190000022145519](https://segmentfault.com/a/1190000022145519)

极客时间-趣谈网络协议
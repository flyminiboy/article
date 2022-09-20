## 手撕OKhttp源码-连接池和路由

首先要明白，连接池复用连接是底层的**TCP连接**，体现在代码里面是 `RealConnection` 里面的 `Socket`，并不是 `Call`。

其次我们还要理清楚`Call`，`Route`，`RealConnection`它们的关系。

本篇就不赘述一些必备的基础知识了，当然了主要是小弟也是临时抱佛脚学的，不敢造次。大家懂的都懂，自行去学习网络相关的就行了。本篇主要是HTTP版本迭代过程中关于连接的优化吧。

比如HTTP1.1 默认开启 **keep-alive 长连接**，引入**管道模式**；HTTP2 多路复用，二进制流，帧。小弟不懂，小弟不懂，自行学习吧。无法抛转了，只能大家自救了。

言归正传，搞清楚`Call`，`Route`，`RealConnection`这仨个的关系，上图。

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wdFLgiaUk7jNmUbOW7ibGUA2LfibmqcUcy37UCG2HYQ9Jc8PtXeke0CGawefXicUh1f5YPgNkNvzgoSiaw/0?wx_fmt=png)

所以连接 Client 和 Server 是 `RealConnection` (归根结底还是Socket)。`Call` 是应用层的一个封装，可以携带我们的 请求信息（`Request`）, `Route` 封装了连接从Client到Sever的信息，告诉 `RealConnection` 连接到哪里。

到这里大家应该都明白了吧，连接说的是 `RealConnection`。同样连接的复用也是指复用 `RealConnection`，只不过是不同的HTTP版本有不同的实现方式。既然要复用那就必然要有一个连接池来管理这些连接了。

### 连接池

#### RealConnection

```
  // 此连接承载的 Call  
  val calls = mutableListOf<Reference<RealCall>>()
  
  // 简单的理解，是否允许当前连接被其他的 Exchange 复用，TRUE不可以在此连接上创建新的 Exchange，FALSE可以在此连接上创建新的 Exchange
  var noNewExchanges = false
```

#### ConnectionPool

管理 HTTP 和 HTTP/2 连接的重用以减少网络延迟。

其实它主要是做了一个配置信息，真正的管理操作在 `RealConnectionPool`。配置了最大的闲置连接数和keepAlive时间。

```
  // 最大的闲置连接数，keepAlive 设置
  constructor() : this(5, 5, TimeUnit.MINUTES)
```

#### RealConnectionPool

真正管理连接的地方。

```
  // 用来清除过期的连接的任务队列
  private val cleanupQueue: TaskQueue = taskRunner.newQueue()
  private val cleanupTask = object : Task("$okHttpName ConnectionPool") {
    override fun runOnce() = cleanup(System.nanoTime())
  }

  // 双端队列，保存连接
  private val connections = ArrayDeque<RealConnection>()
```

相关类介绍到这里，下面我们看看连接是如何复用的。

所有的连接复用逻辑都在 `ExchangeFinder#findConnection` 中实现。下面我们就将这部分代码拿出来遛一遛。

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

	// 省略call已经建立连接，继续使用此连接进行请求的逻辑判断...

      if (result == null) {
        // The connection hasn't had any problems for this call.
        refusedStreamCount = 0
        connectionShutdownCount = 0
        otherFailureCount = 0

        // 这个地方开始从本地连接池获取
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

    // 判断是否需要构建 RouteSelector.Selection（一组选定的路由-Route）
    var newRouteSelection = false
    if (selectedRoute == null && (routeSelection == null || !routeSelection!!.hasNext())) {
      var localRouteSelector = routeSelector
      if (localRouteSelector == null) {
        localRouteSelector = RouteSelector(address, call.client.routeDatabase, call, eventListener)
        this.routeSelector = localRouteSelector
      }
      newRouteSelection = true
      // 这一步会完成DNS解析
      	// 得到一组路由
      routeSelection = localRouteSelector.next()
    }

    var routes: List<Route>? = null
    synchronized(connectionPool) {
      if (call.isCanceled()) throw IOException("Canceled")

	// 如果有了一组新的路由
      if (newRouteSelection) {
        // 用新的路由再次去连接池里面做匹配
        routes = routeSelection!!.routes
        if (connectionPool.callAcquirePooledConnection(address, call, routes, false)) {
          foundPooledConnection = true
          result = call.connection
        }
      }

	// 没有在连接池找到可用连接
      if (!foundPooledConnection) {
        if (selectedRoute == null) {
        // 取第一个可用路由
          selectedRoute = routeSelection!!.next()
        }

        // 构建一个 RealConnection
        result = RealConnection(connectionPool, selectedRoute!!)
        connectingConnection = result
      }
    }

    // If we found a pooled connection on the 2nd time around, we're done.
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result!!)
      return result!!
    }

    // 省略连接代码...
    
        synchronized(connectionPool) {
      connectingConnection = null
      // Last attempt at connection coalescing, which only occurs if we attempted multiple
      // concurrent connections to the same host.
      if (connectionPool.callAcquirePooledConnection(address, call, routes, true)) {
        // 省略 ...
      } else {
      	// 将当前连接放到连接池中
        connectionPool.put(result!!)
        call.acquireConnectionNoEvents(result!!)
      }
    }
    
    return result!!
  }
```

```
  fun callAcquirePooledConnection(
    address: Address,
    call: RealCall,
    routes: List<Route>?,
    requireMultiplexed: Boolean
  ): Boolean {
    this.assertThreadHoldsLock()
	// 遍历队列
    for (connection in connections) {
    	// HTTP/2 连接 的一个判断，当前的请求是基于HTTP/2 协议，当前连接也需要是一个HTTP/2 连接
      if (requireMultiplexed && !connection.isMultiplexed) continue
      	// 判断当前的请求地址是否可以复用当前的连接
      if (!connection.isEligible(address, routes)) continue
      call.acquireConnectionNoEvents(connection)
      return true
    }
    return false
  }
```

#### RealConnection#isEligible

1. 是否超过当前连接最大的承载量。

这个最大承载量的设定，和HTTP1.1 HTTP2.0 关于连接复用的实现相关。

这个地方说一个不恰当的例子吧。

HTTP1.1的复用和线程池里面线程复用一样，只不过是逻辑存在一个主动和被动区别。
但是核心一致，都需要上一个任务完成，才可以复用该连接或者线程进行下一个。

HTTP2.0是同一个连接可以同时支持多个请求。

2. 除了host其他属性是否一致


```
  internal fun isEligible(address: Address, routes: List<Route>?): Boolean {
    // If this connection is not accepting new exchanges, we're done.
    if (calls.size >= allocationLimit || noNewExchanges) return false

    // If the non-host fields of the address don't overlap, we're done.
    if (!this.route.address.equalsNonHost(address)) return false

    // If the host exactly matches, we're done: this connection can carry the address.
    if (address.url.host == this.route().address.url.host) {
      return true // This connection is a perfect match.
    }
    
    // 下面就是http2里面多路复用的特性了

    // At this point we don't have a hostname match. But we still be able to carry the request if
    // our connection coalescing requirements are met. See also:
    // https://hpbn.co/optimizing-application-delivery/#eliminate-domain-sharding
    // https://daniel.haxx.se/blog/2016/08/18/http2-connection-coalescing/

    // 1. This connection must be HTTP/2.
    if (http2Connection == null) return false

    // 2. The routes must share an IP address.
    if (routes == null || !routeMatchesAny(routes)) return false

    // 3. This connection's server certificate's must cover the new host.
    if (address.hostnameVerifier !== OkHostnameVerifier) return false
    if (!supportsUrl(address.url)) return false

    // 4. Certificate pinning must match the host.
    try {
      address.certificatePinner!!.check(address.url.host, handshake()!!.peerCertificates)
    } catch (_: SSLPeerUnverifiedException) {
      return false
    }

    return true // The caller's address can be carried by this connection.
  }
```

#### RealCall#acquireConnectionNoEvents

```
  fun acquireConnectionNoEvents(connection: RealConnection) {
    connectionPool.assertThreadHoldsLock()

    check(this.connection == null)
    this.connection = connection
    // 将当前的 Call 对象添加被复用的连接维护的集合中 CallReference是弱引用 
    connection.calls.add(CallReference(this, callStackTrace))
  }
```

### 路由

一图明白路由从哪里来

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wdFLgiaUk7jNmUbOW7ibGUA2LS22cyJLH9VPw4qcLq0UXSIzt1cGX979ibCQWV6WM06ibtsp0phewIo2Q/0?wx_fmt=png)

这张图全网绝对独一份，描述了路由的由来。最左侧的使我们在初始化`OkHttpClient`和配置`Request`完成的。

后面就是进行组装，核心逻辑都在 **RouteSelector#next** 方法中。在看这个方法之前，我们先介绍一下相关类。

#### Route

连接用于到达抽象源服务器的具体路由。

主要包括选项

1. **HTTP 代理**，可以为客户端显式配置代理服务器。 否则使用代理选择器
2. **IP 地址**，无论是直接连接到源服务器还是代理，打开**socket**都需要一个 IP 地址。 DNS 服务器可能会返回多个 IP 地址进行尝试。

每条路线都是这些选项的特定选择。

#### Selection

一组选定的路由集合。

```
   // 选定的路由集合
  class Selection(val routes: List<Route>) {
    // 记录下一个路由在当前集合的位置
    private var nextRouteIndex = 0
    // 当前选定的路由集合是否有下一个路由	
    operator fun hasNext(): Boolean = nextRouteIndex < routes.size
    // 获取下一个路由
    operator fun next(): Route {
      if (!hasNext()) throw NoSuchElementException()
      return routes[nextRouteIndex++]
    }
  }
```

#### RouteSelector

路由选择器，选择连接到源服务器的路由。

#### RouteDatabase

黑名单，保存了一组失败的路由。

```
// 保存失败路由的集合
private val failedRoutes = mutableSetOf<Route>()
```

#### RouteSelector#next

虽说是核心逻辑，但是代码量很小，逻辑也很简单。相关说明都在代码注释中了。

```
  operator fun next(): Selection {
    if (!hasNext()) throw NoSuchElementException()

    // Compute the next set of routes to attempt.
    val routes = mutableListOf<Route>()
    while (hasNextProxy()) { // 默认只有一个 Proxy.NO_PROXY 

	   // 最重要的是完成DNS解析
      val proxy = nextProxy()
      // 遍历DNS解析之后的结果
      for (inetSocketAddress in inetSocketAddresses) {
      	 // 构建路由
        val route = Route(address, proxy, inetSocketAddress)
        // 判断当前路由是否在黑名单
        	  // 在黑名单则保存为备选路由
        	  // 不在黑名单则直接保存当前路由
        if (routeDatabase.shouldPostpone(route)) {
          postponedRoutes += route
        } else {
          routes += route
        }
      }

      if (routes.isNotEmpty()) {
        break
      }
    }

    if (routes.isEmpty()) {
    	// 如果没有找到合适的路由，则将之前失败的路由地址进行再次尝试
      routes += postponedRoutes
      postponedRoutes.clear()
    }

	 // 最终将符合条件的路由打包返回
    return Selection(routes)
  }
```

### 参考

[https://jishuin.proginn.com/p/763bfbd5a2d7](https://jishuin.proginn.com/p/763bfbd5a2d7)
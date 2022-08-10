## okhttp - 源码解析（整体流程分析）

### 流程图



### 核心类

#### OkHttpClient

#### OkHttpCall

```
  private okhttp3.Call createRawCall() throws IOException {
    okhttp3.Call call = callFactory.newCall(requestFactory.create(args));
    if (call == null) {
      throw new NullPointerException("Call.Factory returned null.");
    }
    return call;
  }
```

```
  override fun newCall(request: Request): Call = RealCall(this, request, forWebSocket = false)
```

#### RealCall

```
  override fun execute(): Response {
    synchronized(this) {
      check(!executed) { "Already Executed" }
      executed = true
    }
    timeout.enter()
    callStart()
    try {
    	// 关键代码
      client.dispatcher.executed(this)
      return getResponseWithInterceptorChain()
    } finally {
    	// 关键代码
      client.dispatcher.finished(this)
    }
  }

```

```
  override fun enqueue(responseCallback: Callback) {
    synchronized(this) {
      check(!executed) { "Already Executed" }
      executed = true
    }
    callStart()
    // 关键代码
    client.dispatcher.enqueue(AsyncCall(responseCallback))
  }
```

##### getResponseWithInterceptorChain 

请求所有的核心逻辑都在这个方法内部完成

```
  internal fun getResponseWithInterceptorChain(): Response {
    // Build a full stack of interceptors.
    // 关键代码
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

#### AsyncCall

本质 `Runnable`

```
    override fun run() {
      threadName("OkHttp ${redactedUrl()}") {
        var signalledCallback = false
        timeout.enter()
        try {
        // 关键代码
          val response = getResponseWithInterceptorChain()
          signalledCallback = true
          responseCallback.onResponse(this@RealCall, response)
        } catch (e: IOException) {
          if (signalledCallback) {
            // Do not signal the callback twice!
            Platform.get().log("Callback failure for ${toLoggableString()}", Platform.INFO, e)
          } else {
            responseCallback.onFailure(this@RealCall, e)
          }
        } catch (t: Throwable) {
          cancel()
          if (!signalledCallback) {
            val canceledException = IOException("canceled due to $t")
            canceledException.addSuppressed(t)
            responseCallback.onFailure(this@RealCall, canceledException)
          }
          throw t
        } finally {
        // 关键代码
          client.dispatcher.finished(this)
        }
      }
    }
```

### Interceptor 拦截器

#### 复杂度 五个⭐️

默认拦截器实现了网络请求的底层逻辑

* `RetryAndFollowUpInterceptor`
* `BridgeInterceptor`
* `CacheInterceptor`
* `ConnectInterceptor`
* `CallServerInterceptor`

> 首先拦截这块首先是责任链模式

### Dispatcher 分发器 

#### 复杂度 俩个⭐️

这个主要是控制异常请求的策略。



### RealConnectionPool 连接池

#### 复杂度 三个⭐️

多路复用的逻辑








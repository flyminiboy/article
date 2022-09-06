## okhttp - 源码解析（整体流程分析）

关于 **OKhttp** 预计分俩篇文章进行分析，第一篇我们先把流程捋顺了，第二篇我们对于网络部分做一下分析。

### 核心流程图

一图镇楼

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9weQJL8tibVaahtn2Pc7aiaa6D3WgmjmvJeicSia2A1ZbLfypgPIe2w6RIUKCGvryEfNpmYicVjwhNPSV7Q/0?wx_fmt=png)

通过上面的流程图，我们可以看到其实 **OKhttp** 整个网络执行过程，也就3.5个关键类。

* `RealCall` 实现 `Call` 接口。主要是负责执行请求，内部会维护连接，请求和响应。
* `Dispatcher` 主要是用来调度**异步任务**，同时会控制异步任务的执行流量（最大连接数和同域名最大连接数）
* `Interceptor`是一个接口，拦截器。默认提供的拦截器是实现底层网络通信的核心。我们可以自定义拦截器，统一请求头，公参添加，网络日志打印等。
* `AsyncCall` 实现 `Runnable` 接口。它最多只能算半个关键类啊。主要是在由`Dispatcher`调度异步任务的时候，交给 `Executor` 服务框架执行的。

下面我们再通过源码看看这几个类。

### 关键类

#### RealCall

* 同步请求

```
  override fun execute(): Response {
    synchronized(this) {
      check(!executed) { "Already Executed" }
      executed = true
    }
    // 关键代码
    timeout.enter()
    callStart()
    try {
    	// 关键代码
      client.dispatcher.executed(this)
      // 关键代码
      return getResponseWithInterceptorChain()
    } finally {
    	// 关键代码
      client.dispatcher.finished(this)
    }
  }

```

这段代码已经很简单了，没啥需要说明的。注释的几个地方，只需要记住就行。


* 异步请求

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

同上。

* 执行请求

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

这个里面是网络请求的所有逻辑，我们会在下一篇文章详细说明。这篇我们只需要记得这个方法是请求的核心逻辑。

#### AsyncCall 是 RealCall 内部类，实现了 Runnable 接口

* 将异步调用排入队列

```
    fun executeOn(executorService: ExecutorService) {
      client.dispatcher.assertThreadDoesntHoldLock()

      var success = false
      try {
      	 // 关键代码
        executorService.execute(this)
        success = true
      } catch (e: RejectedExecutionException) {
        val ioException = InterruptedIOException("executor rejected")
        ioException.initCause(e)
        noMoreExchanges(ioException)
        responseCallback.onFailure(this@RealCall, ioException)
      } finally {
       // 关键代码
        if (!success) {
          client.dispatcher.finished(this) // This call is no longer running!
        }
      }
    }
```

* 执行

```
    override fun run() {
      threadName("OkHttp ${redactedUrl()}") {
        var signalledCallback = false
        // 关键代码
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

综上不管是同步还是异步请求，我们都可以归纳为四部

1. 会启动一个后台线程做超时任务，
2. 通过分发器进行请求调度
3. 执行请求
4. 最终通过分发器进行异步请求调度

#### Dispatcher

* ExecutorService

```
  @get:Synchronized
  @get:JvmName("executorService") val executorService: ExecutorService
    get() {
      if (executorServiceOrNull == null) {
        executorServiceOrNull = ThreadPoolExecutor(0, Int.MAX_VALUE, 60, TimeUnit.SECONDS,
            SynchronousQueue(), threadFactory("$okHttpName Dispatcher", false))
      }
      return executorServiceOrNull!!
    }
   
```

构建了一个线程池。可以注意下这个线程池的配置。我们看看 `Executors` 框架里面为我们提供的 **CachedThreadPool** 。

```
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```

这个地方不明白的小伙伴可以参考小弟之前写的关于线程池的文章。

[ThreadPoolExecutor 浅析](https://mp.weixin.qq.com/s?__biz=Mzg3NzU3OTQwOA==&amp;mid=2247483901&amp;idx=1&amp;sn=aae3448b6cc3aa711d4380970a3b9182&amp;chksm=cf219c02f8561514c75a12b673e9af38efd243a8cc23626a15f98f204e124a4484a9bfcb9a9e&token=2088204321&lang=zh_CN#rd)

* 调度异步请求

```
  internal fun enqueue(call: AsyncCall) {
    synchronized(this) {
      readyAsyncCalls.add(call)

      // Mutate the AsyncCall so that it shares the AtomicInteger of an existing running call to
      // the same host.
      if (!call.call.forWebSocket) {
        val existingCall = findExistingCallWithHost(call.host)
        if (existingCall != null) call.reuseCallsPerHostFrom(existingCall)
      }
    }
    promoteAndExecute()
  }

  private fun findExistingCallWithHost(host: String): AsyncCall? {
    for (existingCall in runningAsyncCalls) {
      if (existingCall.host == host) return existingCall
    }
    for (existingCall in readyAsyncCalls) {
      if (existingCall.host == host) return existingCall
    }
    return null
  }
```

这俩个方法干了俩件事。

1. 将当前请求加到准备执行队列
2. 查找同域名请求，记录个数。

* 异步调度核心逻辑

```
  private fun promoteAndExecute(): Boolean {
    this.assertThreadDoesntHoldLock()

		// 关键代码1
    val executableCalls = mutableListOf<AsyncCall>()
    val isRunning: Boolean
    synchronized(this) {
      val i = readyAsyncCalls.iterator()
      while (i.hasNext()) {
        val asyncCall = i.next()

			// 关键代码2
        if (runningAsyncCalls.size >= this.maxRequests) break // Max capacity.
        if (asyncCall.callsPerHost.get() >= this.maxRequestsPerHost) continue // Host max capacity.

			// 关键代码3
        i.remove()
        asyncCall.callsPerHost.incrementAndGet()
        executableCalls.add(asyncCall)
        runningAsyncCalls.add(asyncCall)
      }
      isRunning = runningCallsCount() > 0
    }

	// 遍历可执行集合，开始执行请求
    for (i in 0 until executableCalls.size) {
      val asyncCall = executableCalls[i]
      asyncCall.executeOn(executorService)
    }

    return isRunning
  }
```

Q:思考关键代码1，为什么又用一个临时的集合来保存可以执行的异步请求呢？

A:全局维护的执行队列中包含同步请求

关键代码2是异步调度控制流量的地方，首先是控制最大请求数（默认64），然后是控制同域名最大请求数（默认5）。

关键代码3是维护全局的队列，将当前请求从准备执行队列移除，添加到执行队列，并且加到临时的可执行的异步请求集合中，当前域名请求数进行累加操作。

* 同步请求

```
  @Synchronized internal fun executed(call: RealCall) {
    runningSyncCalls.add(call)
  }
```

可以看到同步请求，调度器只是将当前的请求直接加入到了执行队列中，而且`Dispatcher`说明也说了只会调度**异步任务**，那么我们思考一个问题，这个同步调用有什么实际意义吗？

这是一个很小的面试考点，容易懵一下。其实这个主要在异步调度的时候做流量控制的时候，得到正确的执行请求数据。

* 执行完成

```
  private fun <T> finished(calls: Deque<T>, call: T) {
    val idleCallback: Runnable?
    synchronized(this) {
      if (!calls.remove(call)) throw AssertionError("Call wasn't in-flight!")
      idleCallback = this.idleCallback
    }

	// 关键代码
    val isRunning = promoteAndExecute()

	// 关键代码
    if (!isRunning && idleCallback != null) {
      idleCallback.run()
    }
  }
```

这个地方的设计熟悉线程池的小伙伴一定不陌生，线程池在线程复用也是类似的逻辑。

这个地方也涉及一个小的面试考点。调度器里面是如何将请求在俩个队列中进行操作的。就是在每个请求完成的时候，再次去执行异步请求调度。

**本篇内容略水，见谅见谅。**

**本篇内容略水，见谅见谅。**

**本篇内容略水，见谅见谅。**








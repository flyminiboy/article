## okhttp - 源码解析（底层网络通信）

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

这个主要是控制异步请求的策略。

控制流量和使用 Executors 框架将请求放在子线程去执行。



### RealConnectionPool 连接池

#### 复杂度 三个⭐️

多路复用的逻辑

#### 连接池背景

http1.x
keep-alive机制

http2.0
多路复用机制

#### 连接池维护

双端队列
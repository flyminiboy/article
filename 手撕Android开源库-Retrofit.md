## 手撕Android开源库-Retrofit

Retrofit 只是在OKHttp的上层进行了一次封装，帮助我们更加便捷的访问网络。所以它并不复杂，但是它里面涉及到大量的设计模式，所以我们分析Retrofit更多的是从架构设计层面来学习。

建造者模式
工厂模式
外观模式
策略模式
适配器模式
装饰器模式
代理模式

这个里面装饰器模式和代理模式可能理解起来比较混乱，这个小弟也是看书包括查了一些资料，结合知乎一个回答和之前看马士兵老师的讲解，简单说一下。

我们先来说一下代理模式哈

比如我们举办一个活动要邀请我们的顶流杰伦唱歌（毕竟夜曲一响，上台领奖），那我们指定是无法直接联系到杰伦的，我们只能通过杰伦的经纪公司，经纪公司代理杰伦的商演业务，在和杰伦的经纪公司谈妥了以后，最终上台唱歌还是杰伦哈。这就是代理模式。

下面我们再来说一下装饰器模式



其他包括Java注解和动态代理，MethodHandles，注解部分涉及到通用的运行时注解+缓存的策略（避免了每次都去解析注解），CompletableFuture（Future升级版，Java8新特性）。

```
  private final Map<Method, ServiceMethod<?>> serviceMethodCache = new ConcurrentHashMap<>();
```


限制

Android 7.0 API 24 支持Java8

Java8

invokeDefaultMethod

Android 8.0 API 26

### 代理模式，装饰器模式

动态代理

注解

```
  ServiceMethod<?> loadServiceMethod(Method method) {
    // 先从缓存中获取服务方法调用
    ServiceMethod<?> result = serviceMethodCache.get(method);
    if (result != null) return result;

    // 解析方法，主要是通过注解生成
    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        result = ServiceMethod.parseAnnotations(this, method);
        // 保存映射关系
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }
```


### Proxy

### MethodHandles

### Lookup

### 源码 

#### OkHttpCall

```
  // 构建 okhttp Call 对象
  private okhttp3.Call createRawCall() throws IOException {
    // okhttp3.Call.Factory callFactory = this.callFactory;
    // if (callFactory == null) {
    //    callFactory = new OkHttpClient();
    // }
    okhttp3.Call call = callFactory.newCall(requestFactory.create(args));
    if (call == null) {
      throw new NullPointerException("Call.Factory returned null.");
    }
    return call;
  }
```

#### ServiceMethod 

```
  static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
    // 构建 request
    RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);

    Type returnType = method.getGenericReturnType();
    if (Utils.hasUnresolvableType(returnType)) {
      throw methodError(
          method,
          "Method return type must not include a type variable or wildcard: %s",
          returnType);
    }
    if (returnType == void.class) {
      throw methodError(method, "Service methods cannot return void.");
    }

    return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
  }
```

#### HttpServiceMethod

```
   // 执行
  @Override
  final @Nullable ReturnT invoke(Object[] args) {
    Call<ResponseT> call = new OkHttpCall<>(requestFactory, args, callFactory, responseConverter);
    return adapt(call, args);
  }
```

将接口方法的调用调整为 HTTP 调用


#### KotlinExtensions 

主要是支持协程

#### DefaultCallAdapterFactory

```
  @Override
  public @Nullable CallAdapter<?, ?> get(
      Type returnType, Annotation[] annotations, Retrofit retrofit) {
    if (getRawType(returnType) != Call.class) {
      return null;
    }
    if (!(returnType instanceof ParameterizedType)) {
      throw new IllegalArgumentException(
          "Call return type must be parameterized as Call<Foo> or Call<? extends Foo>");
    }
    final Type responseType = Utils.getParameterUpperBound(0, (ParameterizedType) returnType);

    final Executor executor =
        Utils.isAnnotationPresent(annotations, SkipCallbackExecutor.class)
            ? null
            : callbackExecutor;

    return new CallAdapter<Object, Call<?>>() {
      @Override
      public Type responseType() {
        return responseType;
      }

      @Override
      public Call<Object> adapt(Call<Object> call) {
        return executor == null ? call : new ExecutorCallbackCall<>(executor, call);
      }
    };
  }
```

#### CompletableFutureCallAdapterFactory

```
  @Override
  public @Nullable CallAdapter<?, ?> get(
      Type returnType, Annotation[] annotations, Retrofit retrofit) {
    if (getRawType(returnType) != CompletableFuture.class) {
      return null;
    }
    if (!(returnType instanceof ParameterizedType)) {
      throw new IllegalStateException(
          "CompletableFuture return type must be parameterized"
              + " as CompletableFuture<Foo> or CompletableFuture<? extends Foo>");
    }
    Type innerType = getParameterUpperBound(0, (ParameterizedType) returnType);

    if (getRawType(innerType) != Response.class) {
      // Generic type is not Response<T>. Use it for body-only adapter.
      return new BodyCallAdapter<>(innerType);
    }

    // Generic type is Response<T>. Extract T and create the Response version of the adapter.
    if (!(innerType instanceof ParameterizedType)) {
      throw new IllegalStateException(
          "Response must be parameterized" + " as Response<Foo> or Response<? extends Foo>");
    }
    Type responseType = getParameterUpperBound(0, (ParameterizedType) innerType);
    return new ResponseCallAdapter<>(responseType);
  }
```
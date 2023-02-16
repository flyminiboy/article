## 手撕Android开源库-Retrofit

Retrofit 只是在OKHttp的上层进行了一次封装，帮助我们更加便捷的访问网络。所以它并不复杂，但是它里面涉及到大量的设计模式，所以我们分析Retrofit更多的是从架构设计层面来学习。

### 设计模式

外观模式

建造者模式

策略模式

适配器模式

装饰器模式

代理模式

工厂模式

这个里面装饰器模式和代理模式可能理解起来比较混乱，这个小弟也是看书包括查了一些资料，结合知乎一个回答和之前看马士兵老师的讲解，简单说一下。

* 代理模式

> 比如我们举办一个活动要邀请我们的顶流杰伦唱歌（毕竟夜曲一响，上台领奖），那我们指定是无法直接联系到杰伦的，我们只能通过杰伦的经纪公司，经纪公司代理杰伦的商演业务，在和杰伦的经纪公司谈妥了以后，最终上台唱歌还是杰伦哈。这就是代理模式。

* 装饰器模式

> 说一个我们大家比较熟悉的电影复仇者联盟，他里面有一个钢铁侠斯塔克。斯塔克本身是不具备很强的战斗力的，就是一个天才发明家，但是他有好多机甲。当他穿上这些机甲就拥有了超级无敌的能力，开始战斗的时候，各种起飞，激光这些是机甲的能力，并不是斯塔克的能力。这就是装饰器模式。

### 源码版本

Retrofit 2.9.0


### Java

#### Java反射

Class

Constructor
构造器
Field
属性
Method
方法

#### Java 泛型

**泛型是JDK1.5最大变化之一**

**泛型实现了参数化类型的概念**，但是Java是**伪泛型**，主要是为了兼容之前非泛化的代码。

**参数化类型**，这五个字一定要好好理解。拆开理解，参数化，类型。参数化一个动词，类型一个名称，总结就是我们要让类型通过参数的形式来动态配置。

这里我们来做一个类比，通过我们平时说参数最多的就是方法调用。我们说我们在调用一个方法会关注这个方法的传参。比如下面这个

```
public void run(int a) {
	int b = a ++;
} 
```

当我们在执行这个`run`方法的时候，参数 **a** 的具体数值，是有具体的调用方传递进来的。**a**是参数，调用方传递的1，2这样的数值是具体的值。

而泛型让**类型参数化**，比如我们容器`ArrayList<E>`，**E是泛型，也是这个容器可以保存的类型**。在实例化的时候，我们会发现这个**E**又好像是一个参数，我们可以具体指定为`String`,`Int`这样的类型。

所以**泛型核心概念是告诉编译器想使用什么类型。**

#### 协变，逆变

这个是一个计算机通用术语。可以直接参考维基百科。

里氏替换原则（向上类型转换），父类出现的地方，可以直接替换一个子类，行为不会变动，出错。

假设现在有三个类

`Apple`，`RedApple`，`GreenApple`，其中`RedApple`和`GreenApple`继承`Apple`

ArrayList<? extends Apple>    ? extends Apple 子类通配符，表示Apple或者Apple的子类，编译阶段实际可以对应

`ArrayList<RedApple>`
`ArrayList<GreenApple>`

多种可能。

所以不支持add操作，编译器无法确定具体类型，不能把一个GreenApple添加到保存RedApple的容器中

ArrayList<? super Apple> ? super Apple 超类通配符 表示Apple或者Apple的超类，编译阶段实际可以对应

ArrayList<Apple>

可以add，但是通过get操作获取数据的时候，无法确定具体类型

**引用《Effective Java》PECS “producer-extends, consumer-super”。**

> extends 是生产者
> 
> super 是消费者



#### 泛型擦除（Java伪泛型）

可以声明一个ArrayList.class
但是不能声明一个ArrayList<String>.class。

```
 ArrayList<String> a = new ArrayList();
 ArrayList<Integer> b = new ArrayList();
 Class c1 = a.getClass();
 Class c2 = b.getClass();
```

对应字节码

```
   L0
    LINENUMBER 10 L0
    NEW java/util/ArrayList
    DUP
    INVOKESPECIAL java/util/ArrayList.<init> ()V
    ASTORE 1
   L1
    LINENUMBER 11 L1
    NEW java/util/ArrayList
    DUP
    INVOKESPECIAL java/util/ArrayList.<init> ()V
    ASTORE 2
   L2
    LINENUMBER 12 L2
    ALOAD 1
    INVOKEVIRTUAL java/lang/Object.getClass ()Ljava/lang/Class;
    ASTORE 3
   L3
    LINENUMBER 13 L3
    ALOAD 2
    INVOKEVIRTUAL java/lang/Object.getClass ()Ljava/lang/Class;
    ASTORE 4
```

泛型边界

extends

```
public class Test<T extends Test> {
    T t;
}
```

字节码

```
public class Test {

  // compiled from: Test.java
  // access flags 0x8
  static INNERCLASS Test$1 null null

  // access flags 0x0
  // signature TT;
  // declaration: t extends T
  LTest; t
}
```

Type

```
public interface Type {
    default String getTypeName() {
        return this.toString();
    }
}
```

让我们看看 Class 的继承关系

```
public final class Class<T> implements Serializable, GenericDeclaration, Type, AnnotatedElement
```

`Class` 实现 `Type` 接口

getGenericSuperclass 获得带有泛型的父类



Type

Type 是 Java 编程语言中所有类型的通用超接口。 这些包括原始类型、参数化类型、数组类型、类型变量和原始类型。



原始类型（raw types）Class

参数化类型（parameterized types）ParameterizedType 泛型
	getActualTypeArguments 获取参数化类型的数组
	

类型变量（type variables）TypeVariable

泛型的泛型参数就是TypeVariable；当父类使用子类的泛型参数指定自身的泛型参数时；或者泛型属性定义在泛型类A<T>中，并使用泛型类A<T>的泛型参数T时，其泛型参数都会被编译器定为泛型变量TypeVariable，而不是被擦除

比较拗口，大家多读几遍，好好理解。

数组类型（array types）GenericArrayType

基本类型（primitive types）Class

泛型通配符 WildcardType





#### JDK 动态代理

JDK动态代理主要涉及两个类：`java.lang.reflect.Proxy` 和 `java.lang.reflect.InvocationHandler`

* `java.lang.reflect.Proxy`

* `java.lang.reflect.InvocationHandler`

示例代码

```

```

看一下控制台输出

#### MethodHandles



#### 注解



#### CompletableFuture

限制

Android 7.0 API 24 支持Java8

Java8

invokeDefaultMethod

Android 8.0 API 26

动态代理



注解部分涉及到通用的运行时注解+缓存的策略（避免了每次都去解析注解）

```
  private final Map<Method, ServiceMethod<?>> serviceMethodCache = new ConcurrentHashMap<>();
```

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
    // 解析注解
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
static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
      Retrofit retrofit, Method method, RequestFactory requestFactory) {
    boolean isKotlinSuspendFunction = requestFactory.isKotlinSuspendFunction;
    boolean continuationWantsResponse = false;
    boolean continuationBodyNullable = false;

    Annotation[] annotations = method.getAnnotations();
    Type adapterType;
    // 支持kotlin协程
    if (isKotlinSuspendFunction) {
      Type[] parameterTypes = method.getGenericParameterTypes();
      Type responseType =
          Utils.getParameterLowerBound(
              0, (ParameterizedType) parameterTypes[parameterTypes.length - 1]);
      if (getRawType(responseType) == Response.class && responseType instanceof ParameterizedType) {
        // Unwrap the actual body type from Response<T>.
        responseType = Utils.getParameterUpperBound(0, (ParameterizedType) responseType);
        continuationWantsResponse = true;
      } else {
        // TODO figure out if type is nullable or not
        // Metadata metadata = method.getDeclaringClass().getAnnotation(Metadata.class)
        // Find the entry for method
        // Determine if return type is nullable or not
      }

      adapterType = new Utils.ParameterizedTypeImpl(null, Call.class, responseType);
      annotations = SkipCallbackExecutorImpl.ensurePresent(annotations);
    } else { 
    	// 记录返回值类型
      adapterType = method.getGenericReturnType();
    }

    // Call 是适配器
    CallAdapter<ResponseT, ReturnT> callAdapter =
        createCallAdapter(retrofit, method, adapterType, annotations);
    Type responseType = callAdapter.responseType();
    if (responseType == okhttp3.Response.class) {
      throw methodError(
          method,
          "'"
              + getRawType(responseType).getName()
              + "' is not a valid response body type. Did you mean ResponseBody?");
    }
    if (responseType == Response.class) {
      throw methodError(method, "Response must include generic type (e.g., Response<String>)");
    }
    // TODO support Unit for Kotlin?
    if (requestFactory.httpMethod.equals("HEAD") && !Void.class.equals(responseType)) {
      throw methodError(method, "HEAD method must use Void as response type.");
    }

    Converter<ResponseBody, ResponseT> responseConverter =
        createResponseConverter(retrofit, method, responseType);

    okhttp3.Call.Factory callFactory = retrofit.callFactory;
    if (!isKotlinSuspendFunction) {
      return new CallAdapted<>(requestFactory, callFactory, responseConverter, callAdapter);
    } else if (continuationWantsResponse) {
      //noinspection unchecked Kotlin compiler guarantees ReturnT to be Object.
      return (HttpServiceMethod<ResponseT, ReturnT>)
          new SuspendForResponse<>(
              requestFactory,
              callFactory,
              responseConverter,
              (CallAdapter<ResponseT, Call<ResponseT>>) callAdapter);
    } else {
      //noinspection unchecked Kotlin compiler guarantees ReturnT to be Object.
      return (HttpServiceMethod<ResponseT, ReturnT>)
          new SuspendForBody<>(
              requestFactory,
              callFactory,
              responseConverter,
              (CallAdapter<ResponseT, Call<ResponseT>>) callAdapter,
              continuationBodyNullable);
    }
  }
```

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

#### RequestFactory

解析注解构建 `Request`

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


参考

<<Effective Java>>

<<Think in Java>>

[https://zh.wikipedia.org/wiki/%E5%8D%8F%E5%8F%98%E4%B8%8E%E9%80%86%E5%8F%98](https://zh.wikipedia.org/wiki/%E5%8D%8F%E5%8F%98%E4%B8%8E%E9%80%86%E5%8F%98)

[https://www.zhihu.com/question/41988550](https://www.zhihu.com/question/41988550)

<<设计模式之禅>>

[http://static.kancloud.cn/sstd521/design/193627](http://static.kancloud.cn/sstd521/design/193627)
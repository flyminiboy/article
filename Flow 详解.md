## Flow 从入门到放弃系列1

### 基础概念

#### 构造器

**相关kotlin文件**

* Flow.kt
	
	接口和抽象类定义
	
* Builders.kt

	快速构造`Flow`

#### 操作符

##### 中间操作符

* Transform.kt

	定义了各种操作符
	
* Emitters.kt

	做内部变换

##### 终止操作符

* Collect.kt 

	终止 Flow 数据流，并且接收这些数据
	

#### 控制器（连接操作符）

* FlowCollector.lt

	连接操作符

#### 很简单的一张图来说明一切问题。

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wc7VaVQNQa94Gj3bhUYicvW6yicLL5gkicV5HoPUSu3Oc3VLkIUO91wpk1OwIwBxFic3CibFBoIEfBRGbg/640?wx_fmt=png)

### Flow 声明周期

* onStart
* onCompletion

这里主要说明的是Flow的生命周期表现上也是一个中间操作符

```kotlin
public fun <T> Flow<T>.onStart(
    action: suspend FlowCollector<T>.() -> Unit
): Flow<T> = unsafeFlow { // Note: unsafe flow is used here, but safe collector is used to invoke start action
    val safeCollector = SafeCollector<T>(this, currentCoroutineContext())
    try {
        safeCollector.action() // 关键代码1
    } finally {
        safeCollector.releaseIntercepted()
    }
    collect(this) // 关键代码2
}
```

```kotlin
public fun <T> Flow<T>.onCompletion(
    action: suspend FlowCollector<T>.(cause: Throwable?) -> Unit
): Flow<T> = unsafeFlow { // Note: unsafe flow is used here, but safe collector is used to invoke completion action
    try {
        collect(this) // 关键代码1
    } catch (e: Throwable) { // 关键代码3
        /*
         * Use throwing collector to prevent any emissions from the
         * completion sequence when downstream has failed, otherwise it may
         * lead to a non-sequential behaviour impossible with `finally`
         */
        ThrowingCollector(e).invokeSafely(action, e) // 关键代码4
        throw e
    }
    // Normal completion
    val sc = SafeCollector(this, currentCoroutineContext())
    try {
        sc.action(null) // 关键代码2
    } finally {
        sc.releaseIntercepted()
    }
}
```

通过看俩个生命周期的实现，和分别标注的俩处注释。可以看到生命周期的执行顺序并不是严格按照上下游来执行的。

`onStart` 保证在 `collect` 之前执行，`onCompletion` 在 `collect` 之后执行

`onCompletion` 再多说一句，在**Flow执行完毕**，**Flow异常**，**Flow取消**这几种情况都会执行。

注意看我们标注的**关键代码3**和**关键代码4**，捕获一切的`Throwable`对象，在构造`ThrowingCollector`执行`invokeSafely`的时候会将`onCompletion`的**lamada表达式**传递进去。所以异常会被执行肯定不会有疑问。

那么Flow被取消也会执行什么鬼啊？？？

这个需要小伙伴的协程的基础知识了，协程在Job在取消的时候默认是会抛一个`JobCancellationException`的异常。

哦吼哦吼哦吼！！！

### 示例代码

```kotlin
flow { // 关键代码1 ==> FlowCollector
    emit(1)
    emit(2)
    emit(3)
    emit(4)
}.collect { // 关键代码2 ==> FlowCollector
    Log.e("TAG", "flow: $it")
}
```

### 分析示例代码

注释1 

```kotlin
public fun <T> flow(@BuilderInference block: suspend FlowCollector<T>.() -> Unit): Flow<T> = SafeFlow(block) 

public fun interface FlowCollector<in T> {

    /**
     * Collects the value emitted by the upstream.
     * This method is not thread-safe and should not be invoked concurrently.
     */
    public suspend fun emit(value: T)
}
```

构造一个`Flow` => `SafeFlow`

传递的**lamada表达式**是一个`FlowCollector`的扩展函数，**按照接口理解**

注释2

调用`collect`方法，传递一个**lamada表达式**，直接按照接口理解就行了。

```kotlin
public interface Flow<out T> {
    public suspend fun collect(collector: FlowCollector<T>)
}

public abstract class AbstractFlow<T> : Flow<T>, CancellableFlow<T> {

    public final override suspend fun collect(collector: FlowCollector<T>) {
        val safeCollector = SafeCollector(collector, coroutineContext) // 关键代码3
        try {
            collectSafely(safeCollector)
        } finally {
            safeCollector.releaseIntercepted()
        }
    }

    public abstract suspend fun collectSafely(collector: FlowCollector<T>)
}

private class SafeFlow<T>(private val block: suspend FlowCollector<T>.() -> Unit) : AbstractFlow<T>() {
    override suspend fun collectSafely(collector: FlowCollector<T>) {
        collector.block() // 关键代码4
    }
}
```

注释3

构建了`SafeCollector`,并且他会持有外层传递进来的的`FlowCollector`（就是那个**lamada表达式**）和一个**协程上下文**（下一篇文章讲述）。这俩个很关键要记住，后面都要用到。

注释4

调用在构造`Flow`的时候传递的**lamada表达式**.这个地方一定要搞清楚，很重要。

调用在构造`Flow`的时候传递的**lamada表达式**.这个地方一定要搞清楚，很重要。

调用在构造`Flow`的时候传递的**lamada表达式**.这个地方一定要搞清楚，很重要。

`collectSafely` 方法接收的 `FlowCollector` 是 `collect` 中构建的 `SafeCollector`，`SafeCollector` 持有的是 调用 `collect` 时候传递进来的 **lamada表达式**，然后 `collectSafely` 调用 `collector.block()` 是调用的构造 `Flow` 的时候传递进来的 **lamada表达式**

这个地方一定要搞清楚，

首先他对于我们理解**flow工作流**很重要

其次也是重要的一个热点，当我们在讨论`flow`和`channel`的时候，总会说到**flow是冷的**，**channel是热的**。

可以看到`flow`只有在`collect`的调用的时候，才会真正的去**执行生成和消费**。

```kotlin

private val emitFun =
    FlowCollector<Any?>::emit as Function3<FlowCollector<Any?>, Any?, Continuation<Unit>, Any?> //关键代码7

internal actual class SafeCollector<T> actual constructor(
    @JvmField internal actual val collector: FlowCollector<T>,
    @JvmField internal actual val collectContext: CoroutineContext
) : FlowCollector<T>, ContinuationImpl(NoOpContinuation, EmptyCoroutineContext), CoroutineStackFrame {

    override val context: CoroutineContext
        get() = lastEmissionContext ?: EmptyCoroutineContext // 关键代码9

    override suspend fun emit(value: T) { // 关键代码5
        return suspendCoroutineUninterceptedOrReturn sc@{ uCont ->
            try {
                emit(uCont, value)
            } catch (e: Throwable) { // 关键代码12
                lastEmissionContext = DownstreamExceptionContext(e, uCont.context)
                throw e
            }
        }
    }

    private fun emit(uCont: Continuation<Unit>, value: T): Any? { // 关键代码6
        val currentContext = uCont.context // 关键代码10
        currentContext.ensureActive()
        // This check is triggered once per flow on happy path.
        val previousContext = lastEmissionContext
        if (previousContext !== currentContext) { // 关键代码11
            checkContext(currentContext, previousContext, value)
            lastEmissionContext = currentContext
        }
        completion = uCont
        val result = emitFun(collector as FlowCollector<Any?>, value, this as Continuation<Unit>) // 关键代码8
        if (result != COROUTINE_SUSPENDED) {
            completion = null
        }
        return result
    }

    private fun checkContext(
        currentContext: CoroutineContext,
        previousContext: CoroutineContext?,
        value: T
    ) { 
        if (previousContext is DownstreamExceptionContext) {
            exceptionTransparencyViolated(previousContext, value)
        }
        checkContext(currentContext)
    }
}
```

注释7和注释8重点说一下。

```
FlowCollector::emit，这是函数引用的语法，代表了它就是 FlowCollector 的 emit() 方法，
它的类型是Function3<FlowCollector<Any?>, Any?, Continuation<Unit>, Any?>

所以注释8的位置，等价直接调用下游的emit方法。
```

这里解释一下为什么`emitFun`是调用下游的`emit`的方法。这就需要一些**kotlin**的基础知识了。

我们平时写的函数类型() -> Unit其实就对应了 **Function0**，也就是：没有参数的函数类型。所以，这里的 **Function3** 其实就代表了三个参数的函数类型

然后emit只有一个参数你骗我，怎么能是调用下游的emit方法呢？？？

回顾之前emit的函数定义

```kotlin
public fun interface FlowCollector<in T> {

    /**
     * Collects the value emitted by the upstream.
     * This method is not thread-safe and should not be invoked concurrently.
     */
    public suspend fun emit(value: T)
}
```

然后理解好俩个**kotlin**在实际调用的本质

1. 函数引用
2. 协程

```kotlin
suspend fun BaseViewModel<*>.emit(value:Int){
    
}
```

这个函数有几个参数？？？

我们看看最终的Java代码

```java
   @Nullable
   public static final Object emit(@NotNull BaseViewModel $this$emit, int value, @NotNull Continuation $completion) {
      return Unit.INSTANCE;
   }
```

what's fuck

what's fuck

what's fuck

是的没错是三个。想必现在不需要我多说了吧。

下一篇我们聊一下flow的实战，协程作用域的切换和异常的捕获俩部分。敬请期待。











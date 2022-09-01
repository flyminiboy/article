## LeakCanary

### 代码警告⚠️

代码恐惧症请慎重阅读。本文充斥大量源代码

代码恐惧症请慎重阅读。本文充斥大量源代码

代码恐惧症请慎重阅读。本文充斥大量源代码

### 内容超长警告⚠️

### 必要背景知识

#### 引用（Reference）

* 强引用（没有对应的类型表示，也就是说强引用是普遍存在的）

* 软引用（SoftReference）

* 弱引用（WeakReference）

* 虚引用（PhantomReference）

不同的引用类型，主要体现的是**对象不同的可达性（reachable）状态和对垃圾收集的影响**。

所有引用类型都是抽象类` java.lang.ref.Reference `的子类。它提供了` get() `方法获取原有对象

特殊 **虚引用**，重写了 `get()` 方法，返回 `null`

```
public T get() {
    return null;
}
```

**`Reference`简单代码结构**

```
public abstract class Reference<T> {

    private T referent; // 原有对象
    volatile ReferenceQueue<? super T> queue;
    
    volatile Reference next;
    private transient Reference<T> discovered;
    private static final Object processPendingLock = new Object();
    private static boolean processPendingActive = false;
    ...
}
```

**Java 定义的不同可达性级别（reachability level）**

* 强可达（Strongly Reachable）就是当一个对象可以有一个或多个线程可以不通过各种引用访问到的状态

* 软可达（Softly Reachable）只能通过软引用才能访问到对象的状态

* 弱可达（Weakly Reachable）只能通过弱引用访问时的状态，这是十分临近 finalize 状态的时机，当弱引用被清除的时候，就符合 finalize 的条件了

* 虚可达（Phantom Reachable）没有强、软、弱引用关联，并且 finalize 过了，只有幻象引用指向这个对象的时候

#### 引用队列（ReferenceQueue）

在创建各种引用并关联到相应对象时，可以选择是否需要关联**引用队列**，`JVM` 会在**特定时机**将引用 enqueue 到队列里，我们可以从队列里获取引用（remove 方法在这里实际是有获取的意思）进行相关后续逻辑。


**特定时机** 是什么时候？

当`Reference`实例持有的原有对象`referent`要被回收的时候，Reference实例会被放入引用队列。

**简单通过一张图来描述**

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9weu1icO7dZJGW2tU1rR32POwUTia3j4BUIBhiaSZwk3VvTky1MgPQBI1l9n0N44vZBwibKPc10lBIQBQw/0?wx_fmt=png)


我们再来写一段代码演示一下。

```java
    public static void main(String[] args) {

        try {
            B b = new B(); // 强引用 就是我们最常见的普通对象引用
            ReferenceQueue<B> queue = new ReferenceQueue<>();

            System.out.println(b);

            // 创建一个弱引用
            // 关联到 B 对象 和 queue 引用队列
            // 这个地方建立了俩个关系，首先是 B 对象有一个弱引用可达，其次是 WeakReference 这个对象关联到了引用队列，有JVM在特定时机，将WeakReference这个对象加入队列中
            WeakReference<B> wrt = new WeakReference<>(b, queue);
            System.out.println("关联 - " + wrt + "| referent -" + wrt.get());

            Reference temp;
            while ((temp = queue.poll()) != null) {
                System.out.println("检测引用队列 " + temp);
            }

            b = null; // 如果没有其他的引用关系，只要超过了引用的作用域或者显式地将相应（强）引用赋值为 null，就是可以被垃圾回收器收集的了

            Runtime.getRuntime().gc(); // 手动触发GC 但是具体回收还要是垃圾收集的策略
            // 此时不同的引用会触发不同的策略

            Thread.sleep(100); // 给gc之后入队一点时间
            System.out.println("gc之后 " + wrt.get());
            while ((temp = queue.poll()) != null) {
                System.out.println("检测引用队列 引用 - " + temp + "| referent -" + temp.get());
            }
            System.runFinalization();
        } catch (Exception e) {
            e.printStackTrace();
        }
	}
```

输出信息

```
com.fly.doc.B@3d012ddd
关联 - java.lang.ref.WeakReference@6f2b958e| referent -com.fly.doc.B@3d012ddd
B finalize
gc之后 null
检测引用队列 引用 - java.lang.ref.WeakReference@6f2b958e| referent -null
```

我们通过一张图来说明上面的流程

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wdmr2QicsdPSibicpJThjpOoyIX6py00qlOcljAPZ94MMOgAek6hngwqvSjn0l91OS3AvKtNOfKibEoeQ/0?wx_fmt=png)

```java
    public static void main(String[] args) {

        try {
            B b = new B(); // 强引用 就是我们最常见的普通对象引用
            ReferenceQueue<B> queue = new ReferenceQueue<>();

            System.out.println(b);

            A a = new A(b);

            // 创建一个弱引用
            // 关联到 B 对象 和 queue 引用队列
            // 这个地方建立了俩个关系，首先是 B 对象有一个弱引用可达，其次是 WeakReference 这个对象关联到了引用队列，有JVM在特定时机，将WeakReference这个对象加入队列中
            WeakReference<B> wrt = new WeakReference<>(b, queue);
            System.out.println("关联 - " + wrt + "| referent -" + wrt.get());

            Reference temp;
            while ((temp = queue.poll()) != null) {
                System.out.println("检测引用队列 " + temp);
            }

            // A 还持有引用
            b = null; // 如果没有其他的引用关系，只要超过了引用的作用域或者显式地将相应（强）引用赋值为 null，就是可以被垃圾收集的了

            Runtime.getRuntime().gc(); // 手动触发GC 但是具体回收还要是垃圾收集的策略
            // 此时不同的引用会触发不同的策略

            // 此时 B 对象还被 A对象持有，A是强可达 所以B也是强可达
            Thread.sleep(100); // 给gc之后入队一点时间
            System.out.println("gc之后 " + wrt.get());
            System.out.println("gc之后 A" + a.b);
            while ((temp = queue.poll()) != null) {
                System.out.println("检测引用队列 引用 - " + temp + "| referent -" + temp.get());
            }
            System.runFinalization();

            System.out.println("-----------------------");
            a = null;
            Runtime.getRuntime().gc();
            Thread.sleep(1000); // 给gc之后入队一点时间
            System.out.println("gc之后 " + wrt.get());
            while ((temp = queue.poll()) != null) {
                System.out.println("检测引用 回收 - " + temp.get());
            }
            System.runFinalization();
        } catch (Exception e) {
            e.printStackTrace();
        }
        }
```

我们通过一张图来说明上面的流程

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wdmr2QicsdPSibicpJThjpOoyIfN3vY8yQ8d2lqCaAqqr5jaREOU4Ru3rhJ2vKmEVyEslbHejDQkEbFA/0?wx_fmt=png)

**好了关于背景知识我们就补充到这里，可能有不严谨的地方，还望小伙伴们多指点。**

**下面我们来分析 `LeakCanary`**

### 地址

[https://github.com/square/leakcanary](https://github.com/square/leakcanary)

首先我们要知道这是个什么库啊。**what，do what!!!**

**Android 的内存泄漏检测库。**

**Android 的内存泄漏检测库。**

**Android 的内存泄漏检测库。**

默认检测 `activity`，`fragment`，`viewmodel`，`view`，`service`。

**为什么会发生内存泄漏？**

大白话来说，就是我们认为一个对象已经走到了生命尽头，此时如果垃圾回收器进行了回收操作，就应该把这个对象做回收处理。但是因为其他原因（比如其他生命周期更长的对象持有了个该对象）导致无法被回收，从而导致了内存泄漏了。

现在我们结合前面的知识，大家自己有没有一丢丢启发呢？比如以 activity 为例。

通常我们认为 `activity` 在 **destory** 之后就可以被回收了。所以我们可以在 `onDestory` 回调中把 activity 的实例保存到一个`WeakReference`中，并且把`WeakReference`对象关联到**引用队列**。
然后我们可以开启一个定时任务，在任务里面做俩件事情，一个是主动触发**GC**，一个是检测**引用队列**，如果**引用队列**有刚才新建的`WeakReference`对象，则说明`activity`对象已经被回收，如果没有则说明对象无法被回调，可能已经泄漏了。

### 初始化

**LeakCanary** 通过 `ContentProvider` 实现的初始化。

```kotlin
internal sealed class AppWatcherInstaller : ContentProvider() {

  override fun onCreate(): Boolean {
    val application = context!!.applicationContext as Application
    AppWatcher.manualInstall(application)
    return true
  }
}
```

```xml
    <application>
        <provider
            android:name="leakcanary.internal.AppWatcherInstaller$MainProcess"
            android:authorities="${applicationId}.leakcanary-installer"
            android:enabled="@bool/leak_canary_watcher_auto_install"
            android:exported="false" />
    </application>
```

接下来我们通过代码跟踪来分析整个流程。

### 流程

#### AppWatcher

```kotlin

  fun manualInstall(
    application: Application,
    retainedDelayMillis: Long = TimeUnit.SECONDS.toMillis(5),
    // 关键代码1
    watchersToInstall: List<InstallableWatcher> = appDefaultWatchers(application)
  ) {
    checkMainThread()
    if (isInstalled) {
      throw IllegalStateException(
        "AppWatcher already installed, see exception cause for prior install call", installCause
      )
    }
    check(retainedDelayMillis >= 0) {
      "retainedDelayMillis $retainedDelayMillis must be at least 0 ms"
    }
    installCause = RuntimeException("manualInstall() first called here")
    this.retainedDelayMillis = retainedDelayMillis
    if (application.isDebuggableBuild) {
      LogcatSharkLog.install()
    }
    // Requires AppWatcher.objectWatcher to be set
    LeakCanaryDelegate.loadLeakCanary(application)

	// 关键代码2
    watchersToInstall.forEach {
      it.install()
    }
  }

  fun appDefaultWatchers(
    application: Application,
    reachabilityWatcher: ReachabilityWatcher = objectWatcher
  ): List<InstallableWatcher> {
    return listOf(
      ActivityWatcher(application, reachabilityWatcher),
      FragmentAndViewModelWatcher(application, reachabilityWatcher),
      RootViewWatcher(reachabilityWatcher),
      ServiceWatcher(reachabilityWatcher)
    )
  }
```

标注的俩处注释的地方，就是这块代码的核心。没有什么需要说明的，一个地方那个提供了默认的 `InstallableWatcher`，另一个地方直接遍历执行了 `InstallableWatcher` 的 `install` 方法

#### InstallableWatcher

```
interface InstallableWatcher {

  fun install()

  fun uninstall()
}
```



默认四个`InstallableWatcher`对应的 `activity`，`fragment`和`viewmodel`，`view`，`service`。


	ActivityWatcher : InstallableWatcher
	FragmentAndViewModelWatcher : InstallableWatcher
	RootViewWatcher : InstallableWatcher
	ServiceWatcher : InstallableWatcher

下面以 `ActivityWatcher` 为例

#### ActivityWatcher

```
class ActivityWatcher(
  private val application: Application,
  private val reachabilityWatcher: ReachabilityWatcher
) : InstallableWatcher {

  private val lifecycleCallbacks =
    object : Application.ActivityLifecycleCallbacks by noOpDelegate() {
      override fun onActivityDestroyed(activity: Activity) {
      // 关键代码2
        reachabilityWatcher.expectWeaklyReachable(
          activity, "${activity::class.java.name} received Activity#onDestroy() callback"
        )
      }
    }

  override fun install() {
  // 关键代码1
    application.registerActivityLifecycleCallbacks(lifecycleCallbacks)
  }

  override fun uninstall() {
    application.unregisterActivityLifecycleCallbacks(lifecycleCallbacks)
  }
}
```

逻辑已经很清晰了。`install` 时候注册了`activity` 生命周期回调。在 `onActivityDestroyed` 的时候进行逻辑处理。

#### ReachabilityWatcher

```kotlin
fun interface ReachabilityWatcher {

  /**
   * Expects the provided [watchedObject] to become weakly reachable soon. If not,
   * [watchedObject] will be considered retained.
   */
  fun expectWeaklyReachable(
    watchedObject: Any,
    description: String
  )
}
```

#### ObjectWatcher : ReachabilityWatcher

```kotlin
  @Synchronized override fun expectWeaklyReachable(
    watchedObject: Any,
    description: String
  ) {
    if (!isEnabled()) {
      return
    }
    // 移除弱可达对象
    removeWeaklyReachableObjects()
    val key = UUID.randomUUID()
      .toString()
    val watchUptimeMillis = clock.uptimeMillis()
    // 关键代码1
    	 //   private val queue = ReferenceQueue<Any>()
    val reference =
      KeyedWeakReference(watchedObject, key, description, watchUptimeMillis, queue)
    SharkLog.d {
      "Watching " +
        (if (watchedObject is Class<*>) watchedObject.toString() else "instance of ${watchedObject.javaClass.name}") +
        (if (description.isNotEmpty()) " ($description)" else "") +
        " with key $key"
    }
	
	// k-v 映射关系
    watchedObjects[key] = reference
    // 关键代码2 轮训机制，检测对象是否被回收
    checkRetainedExecutor.execute {
      moveToRetained(key)
    }
  }
```


我们注释 关键代码1 的位置，生成了一个弱引用，并且与一个引用队列关联。

关键代码2 的位置会做一个固定时间间隔的轮训。

我们继续跟进代码，看看轮训内部具体做了什么。


```kotlin
  @Synchronized private fun moveToRetained(key: String) {
    removeWeaklyReachableObjects() // 移除弱可达对象
    // 
    val retainedRef = watchedObjects[key] // 取当前弱引用对象，在expectWeaklyReachable 的时候做过映射关系
    if (retainedRef != null) {
      retainedRef.retainedUptimeMillis = clock.uptimeMillis()
      onObjectRetainedListeners.forEach { it.onObjectRetained() }
    }
  }
```

```kotlin
  private fun removeWeaklyReachableObjects() {
    // WeakReferences are enqueued as soon as the object to which they point to becomes weakly
    // reachable. This is before finalization or garbage collection has actually happened.
    var ref: KeyedWeakReference?
    do {
      ref = queue.poll() as KeyedWeakReference?
      if (ref != null) {
        watchedObjects.remove(ref.key)
      }
    } while (ref != null)
  }
```


```kotlin 
  private fun checkRetainedObjects() {
    val iCanHasHeap = HeapDumpControl.iCanHasHeap()

    val config = configProvider()

    if (iCanHasHeap is Nope) {
      if (iCanHasHeap is NotifyingNope) {
        // Before notifying that we can't dump heap, let's check if we still have retained object.
        // 获取保留对象的数量，通过过滤map内部内容做判断
        var retainedReferenceCount = objectWatcher.retainedObjectCount

        if (retainedReferenceCount > 0) {
          gcTrigger.runGc() // 主动触发垃圾回收机制，将弱引用对象加入到关联的引用队列
          retainedReferenceCount = objectWatcher.retainedObjectCount
        }

        val nopeReason = iCanHasHeap.reason()
        val wouldDump = !checkRetainedCount(
          retainedReferenceCount, config.retainedVisibleThreshold, nopeReason
        )

        if (wouldDump) {
          val uppercaseReason = nopeReason[0].toUpperCase() + nopeReason.substring(1)
          onRetainInstanceListener.onEvent(DumpingDisabled(uppercaseReason))
          showRetainedCountNotification(
            objectCount = retainedReferenceCount,
            contentText = uppercaseReason
          )
        }
      } else {
        SharkLog.d {
          application.getString(
            R.string.leak_canary_heap_dump_disabled_text, iCanHasHeap.reason()
          )
        }
      }
      return
    }

    var retainedReferenceCount = objectWatcher.retainedObjectCount

    if (retainedReferenceCount > 0) {
    // 关键代码
      gcTrigger.runGc()
      retainedReferenceCount = objectWatcher.retainedObjectCount
    }

    if (checkRetainedCount(retainedReferenceCount, config.retainedVisibleThreshold)) return

    val now = SystemClock.uptimeMillis()
    val elapsedSinceLastDumpMillis = now - lastHeapDumpUptimeMillis
    if (elapsedSinceLastDumpMillis < WAIT_BETWEEN_HEAP_DUMPS_MILLIS) {
      onRetainInstanceListener.onEvent(DumpHappenedRecently)
      showRetainedCountNotification(
        objectCount = retainedReferenceCount,
        contentText = application.getString(R.string.leak_canary_notification_retained_dump_wait)
      )
      scheduleRetainedObjectCheck(
        delayMillis = WAIT_BETWEEN_HEAP_DUMPS_MILLIS - elapsedSinceLastDumpMillis
      )
      return
    }

    dismissRetainedCountNotification()
    val visibility = if (applicationVisible) "visible" else "not visible"
    dumpHeap(
      retainedReferenceCount = retainedReferenceCount,
      retry = true,
      reason = "$retainedReferenceCount retained objects, app is $visibility"
    )
  }
```

#### GcTrigger GC触发器

默认实现直接用的AOSP里面代码

```
  object Default : GcTrigger {
    override fun runGc() {
      // Code taken from AOSP FinalizationTest:
      // https://android.googlesource.com/platform/libcore/+/master/support/src/test/java/libcore/
      // java/lang/ref/FinalizationTester.java
      // System.gc() does not garbage collect every time. Runtime.gc() is
      // more likely to perform a gc.
      Runtime.getRuntime()
        .gc()
      enqueueReferences()
      System.runFinalization()
    }

    private fun enqueueReferences() {
      // Hack. We don't have a programmatic way to wait for the reference queue daemon to move
      // references to the appropriate queues.
      try {
        Thread.sleep(100)
      } catch (e: InterruptedException) {
        throw AssertionError()
      }
    }
  }
```

代码基本上没有什么需要特殊说明的地方，需要注意的地方都加了注释，不理解的还是多看上面的背景知识介绍吧。整体逻辑和我们刚开始自己的分析是一致的。其他的几个也是类似的逻辑，无非是需要确定自己的 destory 位置。

另

* 如何保存内存泄漏内存文件
* 如何分析内存泄漏文件
* 如何展示内存泄漏堆栈到ui中

这几个不属于核心内存泄漏知识逻辑，不再进行说明。

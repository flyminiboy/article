## LeakCannary

### 代码警告⚠️

代码恐惧症请慎重阅读。本文充斥大量源代码

代码恐惧症请慎重阅读。本文充斥大量源代码

代码恐惧症请慎重阅读。本文充斥大量源代码

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

**Java 定义的不同可达性级别（reachability level）**

* 强可达（Strongly Reachable）就是当一个对象可以有一个或多个线程可以不通过各种引用访问到的状态

* 软可达（Softly Reachable）只能通过软引用才能访问到对象的状态

* 弱可达（Weakly Reachable）只能通过弱引用访问时的状态，这是十分临近 finalize 状态的时机，当弱引用被清除的时候，就符合 finalize 的条件了

* 虚可达（Phantom Reachable）没有强、软、弱引用关联，并且 finalize 过了，只有幻象引用指向这个对象的时候

#### 引用队列（ReferenceQueue）

在创建各种引用并关联到相应对象时，可以选择是否需要关联**引用队列**，`JVM` 会在特定时机将**引用** enqueue 到队列里，我们可以从队列里获取引用（remove 方法在这里实际是有获取的意思）进行相关后续逻辑。




AppWatcher

```kotlin
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

ObjectWatcher : ReachabilityWatcher

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

> WeakReferences 在它们指向的对象变得弱可达时立即入队
> 

关键代码2 的位置会做一个固定时间间隔的轮训。我们继续跟进代码，看看轮训内部具体做了什么。


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

```
interface InstallableWatcher {

  fun install()

  fun uninstall()
}
```

ActivityWatcher : InstallableWatcher
FragmentAndViewModelWatcher : InstallableWatcher
RootViewWatcher : InstallableWatcher
ServiceWatcher : InstallableWatcher

默认四个  
会在对应的 activity，fragment，viewmodel，view，service销毁的时候，将对象通过一个软引用关联到一个引用队列中。

GcTrigger GC触发器

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

软引用+引用队列




我们在创建各种引用并关联到相应对象时，可以选择是否需要关联引用队列。**JVM** 会在特定时机将引用 enqueue 到队列里，我们可以从队列里获取引用（remove 方法在这里实际是有获取的意思）进行相关后续逻辑


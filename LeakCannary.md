## LeakCannary


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
    removeWeaklyReachableObjects()
    val key = UUID.randomUUID()
      .toString()
    val watchUptimeMillis = clock.uptimeMillis()
    // 关键代码1
    val reference =
      KeyedWeakReference(watchedObject, key, description, watchUptimeMillis, queue)
    SharkLog.d {
      "Watching " +
        (if (watchedObject is Class<*>) watchedObject.toString() else "instance of ${watchedObject.javaClass.name}") +
        (if (description.isNotEmpty()) " ($description)" else "") +
        " with key $key"
    }

    watchedObjects[key] = reference
    // 关键代码2 轮训机制，检测对象是否被回收
    checkRetainedExecutor.execute {
      moveToRetained(key)
    }
  }
```



```kotlin
  @Synchronized private fun moveToRetained(key: String) {
    removeWeaklyReachableObjects()
    val retainedRef = watchedObjects[key]
    if (retainedRef != null) {
      retainedRef.retainedUptimeMillis = clock.uptimeMillis()
      onObjectRetainedListeners.forEach { it.onObjectRetained() }
    }
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

软引用+引用队列

强引用

软引用

弱引用

虚引用


我们在创建各种引用并关联到相应对象时，可以选择是否需要关联引用队列。**JVM** 会在特定时机将引用 enqueue 到队列里，我们可以从队列里获取引用（remove 方法在这里实际是有获取的意思）进行相关后续逻辑
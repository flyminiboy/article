## Lifecycle 详解

**类图镇楼**

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wdiazGJDm28uz1Tuq3JicxAVWf9m8dxSuibqviarWt4loUyXZ1X02pVypZokymqBmsxCkZavib4FeGb9EA/0?wx_fmt=png)

今天和小伙伴们一起学习一下`Jetpack Lifecycle`

先来一段官方介绍。

生命周期感知型组件可执行操作来响应另一个组件（如 `Activity` 和 `Fragment`）的生命周期状态的变化

`androidx.lifecycle` 软件包提供了可用于构建生命周期感知型组件的类和接口 - 这些组件可以根据 activity 或 fragment 的当前生命周期状态自动调整其行为。

如何使用很简单了，就不再介绍了，小伙伴自己官网了解了

[https://developer.android.com/topic/libraries/architecture/lifecycle?hl=zh-cn](https://developer.android.com/topic/libraries/architecture/lifecycle?hl=zh-cn)

也可以看郭霖大神的**第一行代码**。

下面我们主要是介绍一下它的内部原理，如何实现生命周期的感知。

分析过后再结合我们的类图看，我相信大家不仅仅是学习到他的原理，我觉得更重要的是学习到一种架构的思想。

### 核心类和接口

* Lifecycle 抽象类
* LifecycleObserver 接口
* LifecycleOwner 接口

#### Lifecycle 核心代码

```java

    @MainThread
    public abstract void addObserver(@NonNull LifecycleObserver observer);

    @MainThread
    public abstract void removeObserver(@NonNull LifecycleObserver observer);

    @MainThread
    @NonNull
    public abstract State getCurrentState();
```

很明显的一个观察者模式


#### LifecycleObserver

```java
/**
 * Marks a class as a LifecycleObserver. Don't use this interface directly. Instead implement either
 * {@link DefaultLifecycleObserver} or {@link LifecycleEventObserver} to be notified about
 * lifecycle events.
 *
 * @see Lifecycle Lifecycle - for samples and usage patterns.
 */
@SuppressWarnings("WeakerAccess")
public interface LifecycleObserver {

}
```

简单的一个接口定义，并且注释中说明，在实际应用过程，我们最好的实践是根据自己的业务去选择使用`DefaultLifecycleObserver`或者`LifecycleEventObserver`


#### LifecycleOwner

```java
public interface LifecycleOwner {
    /**
     * Returns the Lifecycle of the provider.
     *
     * @return The lifecycle of the provider.
     */
    @NonNull
    Lifecycle getLifecycle();
}
```

同样简单的一个接口定义，只干一件事情，就是提供一个`Lifecycle`。

所以从架构层设计定义就是完美的体现了清晰明了。

现在我们不去看实现。就根据这一层的定义，让我们自己去实现大家会怎么做。我们怎么去监听到如`Activity` 和 `Fragment`的生命周期呢。如果大家对

androidx.core.app.ComponentActivity LifecycleOwner

```java

    private LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);

    @NonNull
    @Override
    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }
```

androidx.activity.ComponentActivity


ReportFragment





```java
    public static void injectIfNeededIn(Activity activity) {
        if (Build.VERSION.SDK_INT >= 29) {
            // On API 29+, we can register for the correct Lifecycle callbacks directly
            LifecycleCallbacks.registerIn(activity);
        }
        // Prior to API 29 and to maintain compatibility with older versions of
        // ProcessLifecycleOwner (which may not be updated when lifecycle-runtime is updated and
        // need to support activities that don't extend from FragmentActivity from support lib),
        // use a framework fragment to get the correct timing of Lifecycle events
        android.app.FragmentManager manager = activity.getFragmentManager();
        if (manager.findFragmentByTag(REPORT_FRAGMENT_TAG) == null) {
            manager.beginTransaction().add(new ReportFragment(), REPORT_FRAGMENT_TAG).commit();
            // Hopefully, we are the first to make a transaction.
            manager.executePendingTransactions();
        }
    }
```

 >=29 registerActivityLifecycleCallbacks

 <29 将fragment添加到activity，依赖fragment的生命周期
 
 
 LifecycleRegistry 处理逻辑核心类
 
 ObserverWithState
 
 
 
 ```java
     public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
        enforceMainThreadIfNeeded("handleLifecycleEvent");
        moveToState(event.getTargetState());
    }
 ```
 
 ```java
     private void moveToState(State next) {
        if (mState == next) {
            return;
        }
        mState = next;
        if (mHandlingEvent || mAddingObserverCounter != 0) {
            mNewEventOccurred = true;
            // we will figure out what to do on upper level.
            return;
        }
        mHandlingEvent = true;
        sync();
        mHandlingEvent = false;
    }
 ```
 
 ```java
 
     private void forwardPass(LifecycleOwner lifecycleOwner) {
        Iterator<Map.Entry<LifecycleObserver, ObserverWithState>> ascendingIterator =
                mObserverMap.iteratorWithAdditions();
        while (ascendingIterator.hasNext() && !mNewEventOccurred) {
            Map.Entry<LifecycleObserver, ObserverWithState> entry = ascendingIterator.next();
            ObserverWithState observer = entry.getValue();
            while ((observer.mState.compareTo(mState) < 0 && !mNewEventOccurred
                    && mObserverMap.contains(entry.getKey()))) {
                pushParentState(observer.mState);
                final Event event = Event.upFrom(observer.mState);
                if (event == null) {
                    throw new IllegalStateException("no event up from " + observer.mState);
                }
                observer.dispatchEvent(lifecycleOwner, event);
                popParentState();
            }
        }
    }

    private void backwardPass(LifecycleOwner lifecycleOwner) {
        Iterator<Map.Entry<LifecycleObserver, ObserverWithState>> descendingIterator =
                mObserverMap.descendingIterator();
        while (descendingIterator.hasNext() && !mNewEventOccurred) {
            Map.Entry<LifecycleObserver, ObserverWithState> entry = descendingIterator.next();
            ObserverWithState observer = entry.getValue();
            while ((observer.mState.compareTo(mState) > 0 && !mNewEventOccurred
                    && mObserverMap.contains(entry.getKey()))) {
                Event event = Event.downFrom(observer.mState);
                if (event == null) {
                    throw new IllegalStateException("no event down from " + observer.mState);
                }
                pushParentState(event.getTargetState());
                observer.dispatchEvent(lifecycleOwner, event);
                popParentState();
            }
        }
    }
 
     private void sync() {
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            throw new IllegalStateException("LifecycleOwner of this LifecycleRegistry is already"
                    + "garbage collected. It is too late to change lifecycle state.");
        }
        while (!isSynced()) {
            mNewEventOccurred = false;
            // no need to check eldest for nullability, because isSynced does it for us.
            if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
                backwardPass(lifecycleOwner);
            }
            Map.Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
            if (!mNewEventOccurred && newest != null
                    && mState.compareTo(newest.getValue().mState) > 0) {
                forwardPass(lifecycleOwner);
            }
        }
        mNewEventOccurred = false;
    }
 ```
 
 特殊处理，全局。初始化  AppInitializer-》doInitialize -》 ProcessLifecycleInitializer-》create-》ProcessLifecycleOwner-》init
 ProcessLifecycleOwner
 
 
 
 
 
 
 
 
 
 
 
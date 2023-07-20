## Lifecycle 详解

**类图镇楼**

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wdiazGJDm28uz1Tuq3JicxAVWf9m8dxSuibqviarWt4loUyXZ1X02pVypZokymqBmsxCkZavib4FeGb9EA/0?wx_fmt=png)

**官方事件和状态流转图**

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wftB9N26wUYnvxjoTNh5St0hKswBHPu4DibQJUX8d8V8ELibSjuemHYibDxO96xQKpewE68MGqaF3Jibg/0?wx_fmt=png)

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

#### Lifecycle 

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

现在我们不去看实现。就根据这一层的定义，让我们自己去实现大家会怎么做。我们怎么去监听到如`Activity`的生命周期呢。如果大家对`Glide`熟悉，想必很容易就会想到通过添加一个透明的fragment来实现这个需求。现在生命周期监听到了，我们再要回调里面去把事件分发出去。很明显`Lifecycle`是做这个事情的，因为他可以管理`LifecycleObserver`。所以我们的要实现`LifecycleOwner`，提供一个`Lifecycle`。最后就差`Lifecycle`，他是一个抽象类，那么我们就自己定义一个`Lifecycle`的子类来实现具体的逻辑。

下面我们就看看Android的官方实现了。

### 核心类和接口

* androidx.activity.ComponentActivity
* androidx.core.app.ComponentActivity 
* ReportFragment 
* LifecycleRegistry


#### androidx.core.app.ComponentActivity

实现 `LifecycleOwner` 接口

```java

    private LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);

    @NonNull
    @Override
    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }
    
    @SuppressLint("RestrictedApi")
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ReportFragment.injectIfNeededIn(this);
    }
    
```

#### ReportFragment


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

哇咔咔，可以看到实现和我们自己的思路是一致的。只不过官方在高版本上（>29）使用了Android提供的更先进的API。

下面我们就看看他在处理生命周期分发做了什么。

```java
    static void dispatch(@NonNull Activity activity, @NonNull Lifecycle.Event event) {
        if (activity instanceof LifecycleRegistryOwner) { // 这个根据源码注释已经被标记Deprecated，不会用到（自定义不包括）。和LifecycleOwner一模一样的逻辑，如果我们要自定义也没有必要用一个被标记为Deprecated的接口了。完全可以用LifecycleOwner平替
            ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
            return;
        }

        if (activity instanceof LifecycleOwner) {
            Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
            if (lifecycle instanceof LifecycleRegistry) {
                ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
            }
        }
    }
```

具体的生命周期中的事件（`Lifecycle.Event`）和状态（`Lifecycle.State`）,小伙伴自行看一下代码和官方的事件状态流转图。

这里可以看到所有事件都最终由`LifecycleRegistry`这个类的`handleLifecycleEvent`去处理了。还记得我们前面在介绍`androidx.core.app.ComponentActivity`继承`Lifecycle`时，实现了抽象方法`getLifecycle`返回的对象吗？
**就是一个`LifecycleRegistry`对象。**

可以看到前面都是流程上的逻辑，下面我们看看`LifecycleRegistry`，看看具体的处理逻辑。
 
 
#### LifecycleRegistry 继承了 Lifecycle
  
 
 ```java
 
 // 用列表(LinkedList)模拟的map结构
 private FastSafeIterableMap<LifecycleObserver, ObserverWithState> mObserverMap =
            new FastSafeIterableMap<>();
 
     public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
        enforceMainThreadIfNeeded("handleLifecycleEvent");
        moveToState(event.getTargetState());
    }
    
    // 维护LifecycleObserver 和 State 的关系，并且做事件分发
    static class ObserverWithState {
        State mState;
        LifecycleEventObserver mLifecycleObserver;

        ObserverWithState(LifecycleObserver observer, State initialState) {
            mLifecycleObserver = Lifecycling.lifecycleEventObserver(observer);
            mState = initialState;
        }

        void dispatchEvent(LifecycleOwner owner, Event event) {
            State newState = event.getTargetState();
            mState = min(mState, newState);
            mLifecycleObserver.onStateChanged(owner, event);
            mState = newState;
        }
    }  
    
    // 添加观察者，LifecycleObserver
    public void addObserver(@NonNull LifecycleObserver observer) {
        enforceMainThreadIfNeeded("addObserver");
        // 初始化状态
        State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
		 // 将当前的	LifecycleObserver 和 状态关联起来维护到一个类里面
        ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
        // 用一个map（本质是一个集合LinkedList）将当前的    LifecycleObserver 和   ObserverWithState建立一个映射关系
        ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);

        if (previous != null) {
            return;
        }
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            // it is null we should be destroyed. Fallback quickly
            return;
        }

        boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
        // 计算目标状态，主要是和上一个LifecycleObserver的状态和状态集合中最后一个状态做对比（双保险）
        State targetState = calculateTargetState(observer);
        mAddingObserverCounter++; 
        while ((statefulObserver.mState.compareTo(targetState) < 0
                && mObserverMap.contains(observer))) {
            // 将当前状态添加到集合的最后保存起来 
            pushParentState(statefulObserver.mState);
            final Event event = Event.upFrom(statefulObserver.mState);
            if (event == null) {
                throw new IllegalStateException("no event up from " + statefulObserver.mState);
            }
            // 将当前的事件分发出去
                     // 会修改 statefulObserver.mState 的状态
            statefulObserver.dispatchEvent(lifecycleOwner, event);
            // 将最后一个状态从集合移除
            popParentState();
            // 重新计算目标状态
            targetState = calculateTargetState(observer);
        }

        if (!isReentrance) {
            // 对其他的观察者做一次状态同步
            sync();
        }
        mAddingObserverCounter--;
    }      
    
 ```
 
 `handleLifecycleEvent`中会接收到一个生命周期相关的事件（`Lifecycle.Event`），由该事件类我们可以得到一个对应的状态（`Lifecycle.State`）。
 
 ```java
         public State getTargetState() {
            switch (this) {
                case ON_CREATE:
                case ON_STOP:
                    return State.CREATED;
                case ON_START:
                case ON_PAUSE:
                    return State.STARTED;
                case ON_RESUME:
                    return State.RESUMED;
                case ON_DESTROY:
                    return State.DESTROYED;
                case ON_ANY:
                    break;
            }
            throw new IllegalArgumentException(this + " has no target state");
        }
 ```
 
 拿到具体的状态会继续去判断状态是如何流转的。
 
 
 ```java
     private void moveToState(State next) {
        if (mState == next) { // 当前状态和目标状态一致，不需要做处理
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
     // 
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
    // 降序迭代器
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
 
 	  // 主要做状态的变化判断是是回退还是前进
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
    
    // 判断是否同步
    private boolean isSynced() {
        if (mObserverMap.size() == 0) {
            return true;
        }
        State eldestObserverState = mObserverMap.eldest().getValue().mState;
        State newestObserverState = mObserverMap.newest().getValue().mState;
        // 列表第一个和最后一个数据的状态一致，当前的状态和最新的状态一致
        return eldestObserverState == newestObserverState && mState == newestObserverState;
    }   
 ```
 
相关重点内容已经在代码注释中国 
 
好了到这里原理我们就解析结束了。
 
感兴趣的可以继续向下看看，有一个和`retrofit`很类似的知识点，就是关于注解的处理，会做一次缓存来完成效率提升。这里就不再分析了，因为相关注解已经被**Google**标记为**Deprecated**。
 
最后我们在说一个特殊的处理`ProcessLifecycleOwner`。在做一些业务需求或者数据埋点之类的需求，我们有时候会关注应用从前台到后台，后台再切回前台这样一个变化。我们会自己在`application`里面调用`registerActivityLifecycleCallbacks`，通过一个计数器来实现相关需求，代码很简单但是代码行数比较多。这样看起来就很不美观。现在Google帮助我们实现了这样的一个逻辑。ProcessLifecycleOwner帮助我们全局实现了计数器的逻辑，我们只需要在我们关注的回调里面做对应的事情即可。具体是怎么做到的，这里给出一个调用链，大家自行去看源码吧。
 
**AppInitializer-》doInitialize -》 ProcessLifecycleInitializer-》create-》ProcessLifecycleOwner-》init-》attach**
 
 
 
 
 
 
 
 
 
 
 
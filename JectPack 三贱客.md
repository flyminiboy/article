## JectPack 三贱客




观察者

lifecycle

定义一个具有 Android 生命周期的对象
	
Event

State 
	Lifecycle states
	
LifecycleRegistry extends lifecycle

lifecycleower
	具有 Android 生命周期的类。

livedata

ObserverWrapper
	Observer



底层用一个fragment和activity做绑定，进行生命周期的关系



setValue 

底层就是

private volatile Object mData;

数据销毁 生命周期 destory 

volatile 很关键

数据倒灌

数据变化 -> 立即响应

数据变化 -> 不响应 -> 生命周期变化-> 响应

粘性事件

收到注册之前的事件

解决

hook 


viewmodel

viewmodelstore
map结构







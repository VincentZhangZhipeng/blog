# RunLoop笔记

## RunLoop模型与结构
> RunLoop就是一个事件循环，在接收到退出信号之前，一直在监听。

1. RunLoop基本模型的伪代码
	
	```objc
	void main() {
	 	while(AppIsRunnig) {
	 		...
	 		
	 		if (someCondition) {
	 			AppIsRunning = false;
	 		}
	 	}
	 	exit();
	 }
```

2. RunLoop和RunLoopMode的结构定义
	
	```objc
	struct __CFRunLoop {
	    ...
	   	 CFMutableSetRef _commonModes;  // 标记为commonMode的RunLoop类型添加到这个集合
	    CFMutableSetRef _commonModeItems; // 
	    CFRunLoopModeRef _currentMode; // 每个运行的RunLoop只能指定的一个Mode
	    CFMutableSetRef _modes;
	    ...
	};
	
	struct __CFRunLoopMode {
		...
	    pthread_mutex_t _lock;	/* must have the run loop locked before locking this */
	    CFMutableSetRef _sources0;	// source0事件的集合
	    CFMutableSetRef _sources1;   // source1事件的集合
	    CFMutableArrayRef _observers;  // 观察者集合
	    CFMutableArrayRef _timers;	// 时间触发器集合
	    ...
	};
	```
3. RunLoop与其基本对象的关系
> RunLoop包含了若干个RunLoopMode，但是当前只能运行一个RunLoopMode，如要切换，要先退出RunLoop再指定一个mode再次进入RunLoop。而每个RunLoopMode包含若干source、观察者和时间触发器。



## RunLoop的基本对象
> 每个CFRunLoopMode中，都含有mode items，即source、timer和Observer中的一个或多个。若RunLoopMode没有item，则退出RunLoop。

### Source
1. source分为两类，source0和source1。source0只含有回调指针，处理如UIEvent，CFSocket这类事件。它只能手动唤醒。
2. source1则是有一个mach port和回调指针，能被Mach内核传递的信息唤醒。触摸事件其实是source1接收系统事件后在回调 \_\_IOHIDEventSystemClientQueueCallback() 内触发的 Source0，Source0 再触发的 \_UIApplicationHandleEventQueue()。
3. RunLoop其核心其实就是利用mach\_msg()方法，使RunLoop进入休眠状态。此时，该方法会将用户态切换到内核态。而内核态再调用mach\_msg()方法则会唤醒RunLoop，处理source1的相关回调。

### Timer
包含一个时间长度和一个回调指针。当它加入到RunLoop时，RunLoop会注册对应的时间点，当时间点到时，RunLoop会被唤醒以执行那个回调。

### Observer
每个Observer（观察者）都包含了一个回调，每次RunLoop的状态发生变化时，观察者就能通过回调接受到这个变化。观察者可以观测的时间点有以下几个：

```objc
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry         = (1UL << 0), // 即将进入Loop
    kCFRunLoopBeforeTimers  = (1UL << 1), // 即将处理 Timer
    kCFRunLoopBeforeSources = (1UL << 2), // 即将处理 Source
    kCFRunLoopBeforeWaiting = (1UL << 5), // 即将进入休眠
    kCFRunLoopAfterWaiting  = (1UL << 6), // 刚从休眠中唤醒
    kCFRunLoopExit          = (1UL << 7), // 即将退出Loop
};
```
CAAnimation是由观察触发回调来重绘，这将会在[RunLoop与UI优化的关系](#RunLoop与UI优化的关系)中讲到。
## RunLoop处理流程

## RunLoop与NSTimer

## RunLoop与UI优化的关系
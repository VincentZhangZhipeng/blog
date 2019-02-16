# RunLoop笔记
> 本文为网上关于RunLoop博文的读后感与整理。

## RunLoop模型与结构
> RunLoop就是一个事件循环，在接收到退出信号之前，一直在监听。

1. RunLoop基本模型的伪代码
	
	```objc
	void main() {
	 	while(AppIsRunnig) {
	 		...
	 		handelEvents();
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
4. RunLoop Source可以被添加到多个mode中去。

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
CAAnimation是由观察者触发回调来重绘，这将会在[RunLoop与UI优化的关系](#RunLoop与UI优化的关系)中讲到。

## RunLoop处理流程
> 原文：[The Run Loop Sequence of Events](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW23)。

1. 即将进入RunLoop。
2. 此处开始是RunLoop的事件循环。通知Observer即将处理Timer的信息。
3. 通知Observer即将处理source0。
4. 处理source0。
5. 如果有source1待处理，马上跳到步骤9。
6. RunLoop无其他操作，通知Observer即将进入休眠。
7. RunLoop进入休眠，等待source、timer事件或者RunLoop设置的超时时间已到唤醒。
8. Observer监听到RunLoop已经被唤醒。
9. 处理待办事项，顺序为：timer触发事件，然后超时时间过期的事件，最后处理source1事件。
10. 通知Observer已经退出RunLoop。

## RunLoop与NSTimer
### CFRunLoopTimerRef结构体
CFRunLoopTimerRef与NSTimer是toll-free bridge关系。首先来看看CFRunLoopTimerRef的结构体：

```objc
typedef struct CF_BRIDGED_MUTABLE_TYPE(NSTimer) __CFRunLoopTimer * CFRunLoopTimerRef;

struct __CFRunLoopTimer {
    CFRuntimeBase _base;
    uint16_t _bits;
    pthread_mutex_t _lock;
    CFRunLoopRef _runLoop;
    CFMutableSetRef _rlModes;
    CFAbsoluteTime _nextFireDate;
    CFTimeInterval _interval;       // 人为设定的时间间隔
    CFTimeInterval _tolerance;      // 人为设定的宽容度（与_interval相加得到最晚触发时间）
    uint64_t _fireTSR;          	   // Timmer的真正触发时间
    CFIndex _order;         /* immutable */
    CFRunLoopTimerCallBack _callout;    /* immutable */
    CFRunLoopTimerContext _context; /* immutable, except invalidation */
};
```
### NSTimer的触发结论
1. 如果当前时间+timeInterval  > 设定触发的时间（runloop任务很重时，会错过执行时间），timer会在runloop闲时继续执行。我们人手设定的触发时间\_interval和\_tolerace只是为了方便计算timer的触发时间_fireTSR。
2. 时间间隔比较小，或者对精度比较高，不适宜用NSTimer来计算时间。
3. 设置了tolerance的NSTimer，对时间精度有一定损耗，但是对提高续航有明显帮助。
4. 对于重复的NSTimer，其多次触发的时刻不是一开始算好的，而是timer触发后计算的。但是计算时参考的是上次应当触发的时间_fireTSR，因此计算出的下次触发的时刻不会有误差。这保证了timer不会出现误差叠加。
5. _fireTSR即为理论触发值。

### NSTimer添加到runloop中的过程
1. 入口函数

	`void CFRunLoopAddTimer(CFRunLoopRef rl, CFRunLoopTimerRef rlt, CFStringRef modeName)`，针对添加到`kCFRunLoopCommonModes`的定时器，递归调用该方法，添加timer到每个runloop的commonMode事件里面。
	
	而对于指定了runloop mode类型的timer，用`__CFRepositionTimerInMode`函数，为timer的runloop属性赋值，并且根据`_fireTSR`的值，为`_timers`重新排序。

2. 排序函数

	`__CFRepositionTimerInMode`用于将`rlt`插入到`timers`正确的index中。而其index的计算则是通过`__CFRunLoopInsertionIndexInTimerArray`函数，根据`rlt`的_fireTSR字段，利用二分查找的算法计算出来。

3. 注册函数

	`__CFArmNextTimerInMode`函数的作用是根据mode中的最前面的那个timer的触发时间，将其通过`dispatch_source_set_runloop_timer`或者`mk_timer`的方式注册。
	
	对于有tolerance的NSTimer，其最终注册成了一个GCD Timer，只不过最终定时器fire的时候，会再通过RunLoop那一层，调用RunLoopTimer中保存的回调。


4. Timer触发

	RunLoop在处理完各种事件后，调用__CFRunLoopServiceMachPort函数等待接收mach port消息并进入休眠。
	
	Runloop被mach port唤醒以后，调用`__CFRunLoopDoTimers`函数，取出所有_fireTSR（理论触发值）小于当前系统时间的rlt（runloop timer），然后调用`__CFRunLoopDoTimers`。
	
	`__CFRunLoopDoTimers`主要是调用NSTimer预先设置好的`callout`方法，并且根据`_fireTSR`计算出下次的`_fireTSR`，新的`_fireTSR`必须晚于当前系统时刻，且会根据新的`_fireTSR`，重新排序`_timers`。
	
	
	NSTimer回调不会被Runloop阻塞掉，但是如果RunLoop忙的时间过长，以至于收到mach-port消息时，已经过了下次的理论触发点，则系统在`__CFRunLoopDoTimer`逻辑中计算_fireTSR的时候，会找到晚于当前时刻的那个理应触发点，作为`_fireTSR`。就是下面这一小段代码：`while (nextFireTSR <= currentTSR) {nextFireTSR += intervalTSR;}`。因此，如果RunLoop的忙的时间很长，长度达到了好多个timeInteval，则忙的这段时间内的timer回调只会被触发一次。

## RunLoop与UI优化

### 常用RunLoopMode
| mode| 描述 |
| :------| :------: |
| kCFRunLoopDefaultMode | 系统默认创建的RunLoop mode，App一般运行在这个mode中 | 
| UITrackingRunLoopMode | 主要用于监听UIScrollView的滑动状态，处理优先级高于default mode |
| kCFRunLoopCommonModes | 占位mode，添加到这个mode的mode items会自动同步到标记了common mode的其他mode中|

系统还默认注册了`UIInitializationRunLoopMode`（刚启动App的第一个mode，启动后不再使用）和 `GSEventReceiveRunLoopMode`（接收系统事件）。

`kCFRunLoopDefaultMode`和`UITrackingRunLoopMode`均是标记了`kCFRunLoopCommonModes`的mode，当需要一边滑动一边定时的时候，可以将NSTimer分别添加到上面两个模式，或者添加到kCFRunLoopCommonModes即可。

### RunLoop和AutoReleasePool的关系

App启动后，主线程RunLoop注册了两个回调都为`_wrapRunLoopWithAutoreleasePoolHandler()`的Observer。

第一个Observer监听Entry，优先级最高，保证回调在所有回调之前，用于创建`autoReleasePool`。

第二个Observer监听`BeforeWating`和`Exit`两个状态。优先级最低，为了保证其回调在所有回调之后。监听`BeforeWating`会调用`_objc_autoreleasePoolPop()`和`_objc_autoReleasePoolPush()`，旧线程池的销毁和新线程池的创建。监听`Exit`则会调用`_objc_autoreleasePoolPop()`，释放线程池。

结论：在两次Runloop进入Sleep之间，通过CFRunloopObeserverRef通知，让AutoReleasePool分别push和pool。

### RunLoop与GCD关系

CGD中,执行`dispatch_async(dispatch_get_main_queue(), block)` 时，`libDispatch`通知主线程RunLoop，让其执行发送过来的block。而分发到其他线程的block仍然由`libDispatch`管理。

### 用RunLoop优化界面更新
UIView或CALayer的变更，如frame的变动，视图层级的变化，或者手动调用`setNeedsLayout/setNeedsDisplay`，都会被标记为待处理，然后提交到一个全局容器中。

前面提到的`BeforeWating`和`Exit`这两个监听，都会回调`_ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv()`，所有在之前被标记为待处理的UIView或者CALayer，都会在这时候更新UI。

利用界面更新的原理，可以改善App的卡顿问题，目的都是利用RunLoop运行时的空闲时间来更新UI，避免繁重任务时更新导致卡顿。以`AsyncDisplayKit`和`UITableView+FDTemplateLayoutCell`为例子。

#### 1. `AsyncDisplayKit`

UI相关操作一般分为以下三部分：
1. 排版是指计算视图的frame，计算文本的高度，计算子视图的排版。
2. 绘制一般是指文本绘制（coreText）、图片绘制（包括图片解压）、元素绘制（Quartz）。
3. UI操作则是`UIView/CALayer`创建、设置属性和销毁。

前两者可以放到其他线程，而第三步必须放到主线程。`AsyncDisplayKit`的思路是即将可以放到后台的所有耗时操作，都放到后台，否则，尽量推迟，等待主线程空闲。

`AsyncDisplayKit`仿照 QuartzCore/UIKit 框架的模式，实现了一套类似的界面更新的机制：即在主线程的 RunLoop 中添加一个 Observer，监听了 kCFRunLoopBeforeWaiting 和 kCFRunLoopExit 事件，在收到回调时，遍历所有之前放入队列的待处理的任务，然后一一执行。



#### 2. `UITableView+FDTemplateLayoutCell`

其是一个优化`UITableView`的第三方库，提供了高度缓存的一系列操作。

其中，`UITableView+FDTemplateLayoutCell`提供的高度预缓存操作，就利用了RunLoop的循环周期，当RunLoop处于`NSDefaultRunLoopMode`（不与`UITrackingRunLoopMode`竞争资源），和接收到`BeforeWatting`的监听时，才进行高度预缓存操作。

`UITableView+FDTemplateLayoutCell`为了避免RunLoop的负荷过重，分解成了若干个任务，新建不同的RunLoop执行。

利用`-performSelector:onThread:withObject:waitUntilDone:modes:`，可以创建source0并添加到指定的`NSThread`所在的RunLoop中。




## Reference
1. [深入理解RunLoop](https://blog.ibireme.com/2015/05/18/runloop/)
2. [从RunLoop源码探索NSTimer的实现原理](https://juejin.im/entry/59b62840f265da064a0f166a)
3. [优化UITableViewCell高度计算](http://blog.sunnyxx.com/2015/05/17/cell-height-calculation/)
---
layout: post
tags:
  - iOS
title: 'iOS RunLoop'
category: iOS
---
# RunLoop学习

## 前言

大家开始编程的时候,第一个程序一般都是打印hello world.回想一下,控制台输出对应字符串以后,是不是整个程序就结束了?我们后面开发的各种APP本质上也就是复杂的hello world.那如果都像打印hello world一样,那APP岂不是打开就闪退?这明显不符合我们的要求,这时候你可能回想,用while循环啊...恭喜你,就是这个思路.而runloop最直接的翻译就是跑一个循环.

不过如果单纯跑一个while循环,那将极其耗费资源.所以各个平台产生了不同的事件模型处理,能在闲置时休眠,有任务时被立即唤醒.而runloop就是iOS平台下的事件模型处理.

我们将runloop当成是跑一个while(1)循环以后,那么他和线程的关系就一目了然了.一个线程里面放两个while(1)是不是吃饱了没事干?所以我们大胆得出一个结论,runloop与线程是一一对应关系.当然如果你只是想开一个线程随便用用.做完事情就销毁掉,那用runloop是不是太重量了?所以runloop就采用了懒加载的方式,第一次访问的时候创建.苹果爸爸怕我们滥用,还设定了一些规则.比如只能在一个线程的内部获取runloop(主线程除外)等.

### 应用

通过前言,我们对runloop有了大致的印象,那现在考虑下我们应该在什么场景下使用runloop?
* 主线程. 我们APP能常驻运行肯定是开启runloop作为支撑的,我们在主线程下对runloop的应用应该是在系统现有对runloop的事件处理机制下进行自定义处理.
* 子线程. 之前提过runloop采用懒加载.是因为不是每个线程都需要重量的runloop来执行任务.比如使用线程执行一些预先设定好的运行时间较长的任务,可能就不需要开启runloop.runloop应该是为了 - 想要与线程更多交互的场景准备的
1)使用input source和其他线程通信
2)线程中使用timer
3)使用任何performSelectore系列方法
4)让线程执行周期性任务

## RunLoop实现

高能预警 - 下面的部分可能有点枯燥,作者能力有限,也无法从设计意图上抽象出为什么要这样实现.有大神可以考虑总结下.

runloop主要职能是围绕着事件和消息的处理.细分一下.大致可以分为sources, timers, observers.
下面会稍微展开,下图是官网上截取的图片,可以辅助理解.

![runloop-source](https://raw.githubusercontent.com/HighmoreXu/BlogImage/master/images/runloop.jpg "runloop-source")

可以看出事件源主要有两种: input source传送来自其他应用或线程的异步事件/消息;Timer Source基于定时器的同步事件,可以定时或重复发送.


### Input sources

监控其他线程或者其他应用的消息然后分发事件给线程(异步)

由上图可以看出,输入源大致可以分为基于端口的输入源及自定义数据源两种类型.两种输入源大体一致,比较大的差异就是触发方式不同.基于端口的事件源会自动由内核发信号,而自定义事件源要被其他线程手动发送信号.

#### Custom Input Sources(Source0)
包含一个回调(函数指针),不能主动触发事件.使用时,需要先调用CFRunLoopSourceSignal(source)，将这个 Source 标记为待处理，然后手动调用 CFRunLoopWakeUp(runloop) 来唤醒 RunLoop，让其处理这个事件。
APP内部事件,APP自己负责管理(触发) 如UIEvent, CFSocket.

#### Port-Based Sources(Source1)
包含了一个 mach_port 和一个回调（函数指针），被用于通过内核和其他线程相互发送消息.
由runloop和内核管理,Mach-Port驱动 如CFMachPort, CFMessagePort.

#### perform selector source
perform系列的方法

### timer sources
在未来一个预定的时间向线程同步分发事件.CFRunLoopTimerRef和NSTimer是toll-free bridged的.
换句话说,我们研究的timer souces也就是NSTimer. NSTimer的实现机制是Monotonic Timer,是基于CPU中断的. Timer受Runloop影响,加入到Runloop时,Runloop会注册对应的时间点,当时间点到时，RunLoop会被唤醒以执行那个回调。如果线程阻塞或者不在这个Mode下，触发点将不会执行，一直等到下一个周期时间点触发。

### RunLoop Observer

```
enum CFRunLoopActivity {
    kCFRunLoopEntry                     = (1 << 0),    // 即将进入Loop   
    kCFRunLoopBeforeTimers 		= (1 << 1),    // 即将处理 Timer    	
    kCFRunLoopBeforeSources		= (1 << 2),    // 即将处理 Source  
    kCFRunLoopBeforeWaiting		= (1 << 5),    // 即将进入休眠     
    kCFRunLoopAfterWaiting 		= (1 << 6),    // 刚从休眠中唤醒   
    kCFRunLoopExit                      = (1 << 7),    // 即将退出Loop  
    kCFRunLoopAllActivities		= 0x0FFFFFFFU  // 包含上面所有状态  
};
typedef enum CFRunLoopActivity CFRunLoopActivity;
```
开发者可以注册成为run-loop的观察者.当runloop位于上面不同的状态时,观察者可以执行相应的回调进行处理.


### RunLoop结构

上面所说的souces, timers, observers组成了RunLoopMode, RunLoop与Mode是一对多关系,不过同一时刻只能以一种Mode运行RunLoop,想要切换,需要退出当前的Mode,然后重启RunLoop.

```
struct __CFRunLoopMode {
    CFStringRef _name;            // Mode Name, 例如 @"kCFRunLoopDefaultMode"
    CFMutableSetRef _sources0;    // source0 set ，非基于Port的，接收点击事件，触摸事件等APP 内部事件
    CFMutableSetRef _sources1;    // source1 set，基于Port的，通过内核和其他线程通信，接收，分发系统事件
    CFMutableArrayRef _observers; // Array
    CFMutableArrayRef _timers;    // Array
    ...
};
```
RunLoop结构体内容分别对应之前讲的内容.

```
struct __CFRunLoop {
    CFMutableSetRef _commonModes;     // Set
    CFMutableSetRef _commonModeItems; // Set<Source/Observer/Timer>
    CFRunLoopModeRef _currentMode;    // Current Runloop Mode
    CFMutableSetRef _modes;           // Set
    ...
};
```

currentMode对应同一时刻一个Mode运行的结论.
modes存储所有的Mode,以name字符串作为标识.
那common开头的成员又发挥什么作用呢?


#### Mode
* kCFDefaultRunLoopMode App的默认Mode，通常主线程是在这个Mode下运行
* UITrackingRunLoopMode 界面跟踪Mode，用于ScrollView追踪触摸滑动，保证界面滑动时不受其他Mode影响
* UIInitializationRunLoopMode 在刚启动App时第进入的第一个Mode，启动完成后就不再使用
* GSEventReceiveRunLoopMode 接受系统事件的内部Mode，通常用不到
* kCFRunLoopCommonModes 这是一个占位用的Mode，不是一种真正的Mode

苹果的mode模式可以说是一大特色, 这个也是苹果保持顺滑的一大利器.比如在滑动列表时,就只进行UITrackingRunLoopMode相关的计算,列表滑动结束以后就切换回kCFDefaultRunLoopMode处理default mode下的sources,timers,observers.


kCFRunLoopCommonModes既然是占位符.那为什么有以下写法呢?
```
[[NSRunLoop mainRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
```
commonModes可以理解为存放所有具有"common"属性的Mode的集合,每当RunLoop的内容发生变化时,RunLoop都会将 _commonModeItems 里的 Source/Observer/Timer 同步到具有 “Common” 标记的所有Mode里.
所以我们实际上是把timer加入到了顶层的 RunLoop 的 commonModeItems中.后续再自动被更新到所有具有"common"属性的Mode里去.

因此,我个人的理解common即公有的需要监测的sources,timers,observers集合.

## RunLoop流程

![RunLoop流程](https://raw.githubusercontent.com/HighmoreXu/BlogImage/master/RunLoop_1.png "RunLoop流程")

```
/// 用DefaultMode启动
void CFRunLoopRun(void) {
    CFRunLoopRunSpecific(CFRunLoopGetCurrent(), kCFRunLoopDefaultMode, 1.0e10, false);
}
 
/// 用指定的Mode启动，允许设置RunLoop超时时间
int CFRunLoopRunInMode(CFStringRef modeName, CFTimeInterval seconds, Boolean stopAfterHandle) {
    return CFRunLoopRunSpecific(CFRunLoopGetCurrent(), modeName, seconds, returnAfterSourceHandled);
}
 
/// RunLoop的实现
int CFRunLoopRunSpecific(runloop, modeName, seconds, stopAfterHandle) {
    
    /// 首先根据modeName找到对应mode
    CFRunLoopModeRef currentMode = __CFRunLoopFindMode(runloop, modeName, false);
    /// 如果mode里没有source/timer/observer, 直接返回。
    if (__CFRunLoopModeIsEmpty(currentMode)) return;
    
    /// 1. 通知 Observers: RunLoop 即将进入 loop。
    __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopEntry);
    
    /// 内部函数，进入loop
    __CFRunLoopRun(runloop, currentMode, seconds, returnAfterSourceHandled) {
        
        Boolean sourceHandledThisLoop = NO;
        int retVal = 0;
        do {
 
            /// 2. 通知 Observers: RunLoop 即将触发 Timer 回调。
            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeTimers);
            /// 3. 通知 Observers: RunLoop 即将触发 Source0 (非port) 回调。
            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeSources);
            /// 执行被加入的block
            __CFRunLoopDoBlocks(runloop, currentMode);
            
            /// 4. RunLoop 触发 Source0 (非port) 回调。
            sourceHandledThisLoop = __CFRunLoopDoSources0(runloop, currentMode, stopAfterHandle);
            /// 执行被加入的block
            __CFRunLoopDoBlocks(runloop, currentMode);
 
            /// 5. 如果有 Source1 (基于port) 处于 ready 状态，直接处理这个 Source1 然后跳转去处理消息。
            if (__Source0DidDispatchPortLastTime) {
                Boolean hasMsg = __CFRunLoopServiceMachPort(dispatchPort, &msg)
                if (hasMsg) goto handle_msg;
            }
            
            /// 通知 Observers: RunLoop 的线程即将进入休眠(sleep)。
            if (!sourceHandledThisLoop) {
                __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeWaiting);
            }
            
            /// 7. 调用 mach_msg 等待接受 mach_port 的消息。线程将进入休眠, 直到被下面某一个事件唤醒。
            /// • 一个基于 port 的Source 的事件。
            /// • 一个 Timer 到时间了
            /// • RunLoop 自身的超时时间到了
            /// • 被其他什么调用者手动唤醒
            __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort) {
                mach_msg(msg, MACH_RCV_MSG, port); // thread wait for receive msg
            }
 
            /// 8. 通知 Observers: RunLoop 的线程刚刚被唤醒了。
            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopAfterWaiting);
            
            /// 收到消息，处理消息。
            handle_msg:
 
            /// 9.1 如果一个 Timer 到时间了，触发这个Timer的回调。
            if (msg_is_timer) {
                __CFRunLoopDoTimers(runloop, currentMode, mach_absolute_time())
            } 
 
            /// 9.2 如果有dispatch到main_queue的block，执行block。
            else if (msg_is_dispatch) {
                __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);
            } 
 
            /// 9.3 如果一个 Source1 (基于port) 发出事件了，处理这个事件
            else {
                CFRunLoopSourceRef source1 = __CFRunLoopModeFindSourceForMachPort(runloop, currentMode, livePort);
                sourceHandledThisLoop = __CFRunLoopDoSource1(runloop, currentMode, source1, msg);
                if (sourceHandledThisLoop) {
                    mach_msg(reply, MACH_SEND_MSG, reply);
                }
            }
            
            /// 执行加入到Loop的block
            __CFRunLoopDoBlocks(runloop, currentMode);
            
 
            if (sourceHandledThisLoop && stopAfterHandle) {
                /// 进入loop时参数说处理完事件就返回。
                retVal = kCFRunLoopRunHandledSource;
            } else if (timeout) {
                /// 超出传入参数标记的超时时间了
                retVal = kCFRunLoopRunTimedOut;
            } else if (__CFRunLoopIsStopped(runloop)) {
                /// 被外部调用者强制停止了
                retVal = kCFRunLoopRunStopped;
            } else if (__CFRunLoopModeIsEmpty(runloop, currentMode)) {
                /// source/timer/observer一个都没有了
                retVal = kCFRunLoopRunFinished;
            }
            
            /// 如果没超时，mode里没空，loop也没被停止，那继续loop。
        } while (retVal == 0);
    }
    
    /// 10. 通知 Observers: RunLoop 即将退出。
    __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);
}
```

结合代码和图片,大致可以总结为以下步骤:

如果RunLoop没有处理的Mode,直接退出.

1. 通知observer,RunLoop已经启动,马上进入Loop循环.
2. 通知observer,RunLoop即将触发Timer回调.(kCFRunLoopBeforeTimers)
3. 通知observer,RunLoop即将触发source0回调.(kCFRunLoopBeforeSources)
4. RunLoop触发就绪的Source0回调.
5. 如果有source1处于等待状态,直接处理这个source1,跳转到第9.
6. 通知observer 线程即将休眠.
7. 让线程休眠,直到被下列条件唤醒:
	1) 有source0的事件到达
	2) timer触发
	3) RunLoop设定的超时时间到了
	4) RunLoop被手动唤醒
8. 通知observer 线程刚刚被唤醒
9. 处理代决事件
	1) 如果有一个timer时间到了,触发timer回调
	2) 如果有dispatch到main_queue的block,执行block
	3) 如果有source1事件触发,处理source1事件
	事件处理完成判断是否需要跳出loop
	1) 进入loop说明处理完成就返回
	2) 超时
	3) 被外部强制停止
	4) mode为空
10. 系统通知观察者,RunLoop即将退出

## RunLoop应用

### AutoReleasePool
APP启动以后,苹果在主线程注册了两个observer,其回调都是_wrapRunLoopWithAutoreleasePoolHandler()
第一个observer监视Entry,回调会调用_objc_autoreleasePoolPush.其 order 是-2147483647，优先级最高，保证创建释放池发生在其他所有回调之前.
第二个observer监视了两个事件:BeforeWaiting(准备进入休眠) 时调用_objc_autoreleasePoolPop() 和 _objc_autoreleasePoolPush() 释放旧的池并创建新池；Exit(即将退出Loop) 时调用 _objc_autoreleasePoolPop() 来释放自动释放池。这个 Observer 的 order是2147483647，优先级最低，保证其释放池子发生在其他所有回调之后。

### 事件响应
source1事件源来接收系统事件,其回调函数为__IOHIDEventSystemClientQueueCallback().
当一个硬件事件(触摸/锁屏/摇晃等)发生后, 首先由 IOKit.framework 生成一个 IOHIDEvent 事件并由 SpringBoard 接收.SpringBoard 只接收按键(锁屏/静音等)，触摸，加速，接近传感器等几种 Event，随后用 mach port 转发给需要的App进程.随后苹果注册的那个 Source1 就会触发回调，并调用 _UIApplicationHandleEventQueue() 进行应用内部的分发.

_UIApplicationHandleEventQueue() 会把 IOHIDEvent 处理并包装成 UIEvent 进行处理或分发，其中包括识别 UIGesture/处理屏幕旋转/发送给 UIWindow 等。通常事件比如 UIButton 点击、touchesBegin/Move/End/Cancel 事件都是在这个回调中完成的。

### 手势识别
当上面的 _UIApplicationHandleEventQueue() 识别了一个手势时，其首先会调用 Cancel 将当前的 touchesBegin/Move/End 系列回调打断。随后系统将对应的 UIGestureRecognizer 标记为待处理。

苹果注册了一个 Observer 监测 BeforeWaiting (Loop即将进入休眠) 事件，这个Observer的回调函数是 _UIGestureRecognizerUpdateObserver()，其内部会获取所有刚被标记为待处理的 GestureRecognizer，并执行GestureRecognizer的回调。

当有 UIGestureRecognizer 的变化(创建/销毁/状态改变)时，这个回调都会进行相应处理。

### 界面更新
当在操作 UI 时，比如改变了 Frame、更新了 UIView/CALayer 的层次时，或者手动调用了 UIView/CALayer 的 setNeedsLayout/setNeedsDisplay方法后，这个 UIView/CALayer 就被标记为待处理，并被提交到一个全局的容器去。

苹果注册了一个 Observer 监听 BeforeWaiting(即将进入休眠) 和 Exit (即将退出Loop) 事件，回调去执行一个很长的函数：
_ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv()。这个函数里会遍历所有待处理的 UIView/CAlayer 以执行实际的绘制和调整，并更新 UI 界面。

### 定时器
NSTimer. 之前简单分析过了.引用链接里面有个专门的分析,可以看下.

经典问题 - 定时器滚动时失效


### PerformSelector

## 引用
[apple runloop](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW23)

[the-secret-world-of-nstimer](https://medium.com/@danielemargutti/the-secret-world-of-nstimer-708f508c9eb)

[behind-autorelease/](https://blog.sunnyxx.com/2014/10/15/behind-autorelease/)

[ibireme runloop](https://blog.ibireme.com/2015/05/18/runloop/)




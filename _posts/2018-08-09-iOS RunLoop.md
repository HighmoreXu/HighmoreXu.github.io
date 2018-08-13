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


### input source

监控其他线程或者其他应用的消息然后分发事件给线程(异步)

由上图可以看出,输入源大致可以分为基于端口的输入源及自定义数据源两种类型.两种输入源大体一致,比较大的差异就是触发方式不同.基于端口的事件源会自动由内核发信号,而自定义事件源要被其他线程手动发送信号.

#### Custom Input Sources(Source0)
包含一个回调(函数指针),不能主动触发事件.使用时,需要先调用CFRunLoopSourceSignal(source)，将这个 Source 标记为待处理，然后手动调用 CFRunLoopWakeUp(runloop) 来唤醒 RunLoop，让其处理这个事件。

#### Port-Based Sources(Source1)
包含了一个 mach_port 和一个回调（函数指针），被用于通过内核和其他线程相互发送消息.

```
struct __CFRunLoopMode {
    CFStringRef _name;            // Mode Name, 例如 @"kCFRunLoopDefaultMode"
    CFMutableSetRef _sources0;    // source0 set ，非基于Port的，接收点击事件，触摸事件等APP 内部事件
    CFMutableSetRef _sources1;    // source1 set，基于Port的，通过内核和其他线程通信，接收，分发系统事件
    CFMutableArrayRef _observers; // Array
    CFMutableArrayRef _timers;    // Array
    ...
};
 
struct __CFRunLoop {
    CFMutableSetRef _commonModes;     // Set
    CFMutableSetRef _commonModeItems; // Set<Source/Observer/Timer>
    CFRunLoopModeRef _currentMode;    // Current Runloop Mode
    CFMutableSetRef _modes;           // Set
    ...
};
```

#### perform selector source


#### timer source

上面代码就是runloop的基本结构.看起来有点抽象,不过其实蛮好理解的.
我们之前提过runloop就是iOS平台下的event loop.管理了所需要处理的事件和消息.这边对应到上述结构其实就是sources,timers,observers.下面分别展开提一下.

### 输入源
输入源会异步的向你的线程分发事件.输入源大致可以分为两种类型,基于端口的源,还有自定义的源.
基于端口的输入源监视应用程序的Mach端口,自定义输入源监视自定义事件源.两者大体类似,只是触发方式不一样,基于端口的源由系统内核自动触发,而自定义源则由另外的线程来触发.
source0, source1就对应上诉两种不同的输入源.source0对应自定义输入源,source1对应端口源.








## 引用
[apple runloop](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW23)

[the-secret-world-of-nstimer](https://medium.com/@danielemargutti/the-secret-world-of-nstimer-708f508c9eb)



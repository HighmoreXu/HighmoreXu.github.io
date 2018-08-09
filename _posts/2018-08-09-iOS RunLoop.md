---
title: 'iOS RunLoop'
layout: post
tags:
  - iOS
category: iOS
---
# RunLoop学习

## 前言

大家开始编程的时候,第一个程序一般都是打印hello world.回想一下,控制台输出对应字符串以后,是不是整个程序就结束了?我们后面开发的各种APP本质上也就是复杂的hello world.那如果都像打印hello world一样,那APP岂不是打开就闪退?这明显不符合我们的要求,这时候你可能回想,用while循环啊...恭喜你,就是这个思路.而runloop最直接的翻译,就是跑一个循环.

不过如果单纯跑一个while循环,那将极其耗费资源.所以各个平台产生了不同的事件模型处理,能在闲置时休眠,有任务时被立即唤醒.而runloop就是iOS平台下的事件模型处理.

我们将runloop当成是跑一个while(1)循环以后,那么他和线程的关系就一目了然了.一个线程里面放两个while(1)是不是吃饱了没事干?所以我们大胆得出一个结论,runloop与线程是一一对应关系.当然如果你只是想开一个线程随便用用.做完事情就销毁掉,那用runloop是不是太重量了?所以runloop就采用了懒加载的方式,第一次访问的时候创建.苹果爸爸怕我们滥用,还设定了一些规则.比如只能在一个线程的内部获取runloop(主线程除外)等.

## RunLoop实现

```
struct __CFRunLoopMode {
    CFStringRef _name;            // Mode Name, 例如 @"kCFRunLoopDefaultMode"
    CFMutableSetRef _sources0;    // Set
    CFMutableSetRef _sources1;    // Set
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

上面代码就是runloop的基本结构.有点抽象,不过其实蛮好理解的.
我们之前提过,runloop就是iOS平台下的event loop.管理了所需要处理的事件和消息.这边对应到上诉结构其实就是source,timers, observers.下面分别展开提一下.

### 输入源






## 引用
[apple runloop](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW23)



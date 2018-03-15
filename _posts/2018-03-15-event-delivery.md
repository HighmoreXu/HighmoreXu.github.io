---
title: 事件分发
layout: post
tags:
  - iOS
published: true
category: iOS
---
* 一般来讲,线程执行完任务以后就会退出.这和如今我们使用App的方式肯定是相违背的,那如何解决这个问题呢? App采用Event Loop的方式,使应用能主线程能随时处理事件且不退出.
* App启动时, UIApplicationMain就会创建一个单例的UIApplication对象,该对象就负责处理和分发系统发送到事件队列中的事件.

那事件被捕获以后,如何传递?如何响应呢?这就是本篇文章的主要总结内容.水平有限,有错请帮忙指出.

事件队列中的数据源大概分为以下三类:
1. UIControl Actions - action / target那一套
2. User Events - 触屏, 摇晃等等
3. System Events - 低内存, 旋转机身等等.

## UIControl Actions

UIControl是一个抽象基类, 为子类提供了一些通用的接口和一些基础的实现.不能直接用于实例化对象.
UIControl存储一个可变数组 _targetActions用于存放私有类UIControlTargetAction. 私有类包含了target, action, eventMask等基本要素. 当调用addTarget方法后, 会自动生成UIControlTargetAction存放人可变数组当中.UIControlTargetAction的target是弱引用持有的关系,否则会造成循环引用的问题.
```
func addTarget(_ target: Any?, 
        action: Selector, 
           for controlEvents: UIControlEvents)
```
当对应事件发生时(sendActionsForControlEvents 或者 控件交互触发), UIControl会调用sendAction:to:forEvent:来将行为消息转发到UIApplication对象, UIApplication对象再调用sendAction:to:fromSender:forEvent:将消息转发到指定的target上, 如果target为nil, 那则会将事件转发到响应链第一个实现了action的对象上(view->superview->...->controller->UIWindow->UIApplication->Appdelegate),如果都没找到就丢弃.这也就是我们常说的Nil-Targeted Actions.

这个技术可以一定程度上解耦某些东西, 可以充分利用响应链来实现我们自己的数据流传递, 但是和通知类似, 也伴随风险, 使用时要考虑是否符合自身场景.详细例子可以参考文末资料.

同样利用这种sendAction的传输机制, 你也可以选择复写该函数, 从而替换某些target或者action.具体例子同见文末资料.

## User Events

该事件主要来源于与用户的交互. 大体可分为两类: 触摸事件, 移动事件. 
Apple设计了一个UIResponder类为这些事件定义了接口,我们日常开发所使用的UIApplication, UIView, UIViewController等都是直接继承UIResponder,所以都具响应事件能力.再配合事件响应链机制大致就构成了这个版块.


### 响应链

即按照视图层级结构链接所有的响应者而最终形成的链式关系.

#### UIResponder
提供了一系列方法来管理响应链. 
```
- (BOOL)isFirstResponder
- (BOOL)becomeFirstResponder
- (BOOL)canBecomeFirstResponder
- (BOOL)resignFirstResponder
- (BOOL)canResignFirstResponder
```
UIResponder的nextResponder不保存或设置下一个响应者, 方法默认实现是nil. 子类的实现必须重写这个方法来设置下一个响应者. UIView实现返回管理它的UIViewController或者其父视图. UIViewController返回它的视图的父视图. UIWindow返回app对象.UIApplication实现返回nil.
这也是响应链的构成原理及顺序,可以通过更改对应方法来满足自定义需求, 也代表响应链是在构造视图结构时生成的.

注意:
1. 响应对象只能在当前响应者能放弃第一响应者且自身能成为第一响应者才可成为第一响应者.
2. 必须处于当前视图层次结构.

#### 触摸事件

当用户触发事件后, UIKit会创建事件对象UIEvent,然后该对象会被放入事件队列中,然后UIApplication会按照先进先出的顺序读取并分发事件队列中的事件.
除触摸事件以外的事件,都会被分发给第一响应者处理,如果第一响应者无法处理,则根据响应链依次传递.对于触摸事件,则会先给UIWindow, UIWindow会利用hit-testing来寻找最合适处理该事件的对象.

* 什么是hit-testing?

检测触摸事件是否发生在视图的边界内.
使用的搜索方法是逆前序深度遍历, 即当触摸事件发生在多个视图的重叠部分时,根据算法将最先检测最右子树的最深视图,而该视图位于界面最前端. 这样可以省去部分搜索操作.

```
- (UIView *)hitTest:(CGPoint)point 
          withEvent:(UIEvent *)event;
- (BOOL)pointInside:(CGPoint)point 
          withEvent:(UIEvent *)event;
```
主要是调用上面两个API来检测hit-view, 下面代码猜测其可能的内部实现,引用自文末资料.
```
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event {
    if (!self.isUserInteractionEnabled || self.isHidden || self.alpha <= 0.01) {
        return nil;
    }
    if ([self pointInside:point withEvent:event]) {
        for (UIView *subview in [self.subviews reverseObjectEnumerator]) {
            CGPoint convertedPoint = [subview convertPoint:point fromView:self];
            UIView *hitTestView = [subview hitTest:convertedPoint withEvent:event];
            if (hitTestView) {
                return hitTestView;
            }
        }
        return self;
    }
    return nil;
}
```
即首先视图得允许接收touch, 然后对subview进行逆序遍历.当无子视图且触摸点在当前view的范围内,则返回self,找到hit-view.

* 事件传递流程

之前hit-testing步骤找到的对象优先尝试处理触摸事件,如果不能处理,则将时间传递到响应链的下一个响应者.以此类推.

```
- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event;
- (void)touchesMoved:(NSSet *)touches withEvent:(UIEvent *)event;
- (void)touchesEnded:(NSSet *)touches withEvent:(UIEvent *)event;
- (void)touchesCancelled:(NSSet *)touches withEvent:(UIEvent *)event;
```
大致会按照如下过程传递.
1. hit-testing找到的view
2. view的superview
3. 如果已经到控制器的rootview还不能处理touch event 则viewcontroller
4. 如果控制器已经是root controller则window
5. application
6. app delegate

其中如果包含手势处理, 也会影响事件的传递.
大致来讲, 如果当前视图的touch被gesture recognizer识别了,那视图会调用
自己的touchesCancelled:withEvent:来取消. 即手势识别优先touch识别.

#### 移动事件

```
// 移动事件开始
- (void)motionBegan:(UIEventSubtype)motion withEvent:(UIEvent *)event
// 移动事件结束
- (void)motionEnded:(UIEventSubtype)motion withEvent:(UIEvent *)event
// 取消移动事件
- (void)motionCancelled:(UIEventSubtype)motion withEvent:(UIEvent *)event
```


#### 远程控制事件

```
- (void)remoteControlReceivedWithEvent:(UIEvent *)event
```
1. 事件来源主要是一些外部的控件, 比如耳机等. 接收这可以检查事件的子类型来确定命令UIEventSubtypeRemoteControlPlay
2. 要运行远程控制事件的分发,必须调用UIApplication的beginReceivingRemoteControlEvents.关闭则调用endReceivingRemoteControlEvents.


#### 其他

1. 获取Undo管理器 
2. 验证命令, 比如文本输入框的复制粘贴等.
```
- (BOOL)canPerformAction:(SEL)action withSender:(id)sender
- (id)targetForAction:(SEL)action withSender:(id)sender
```
3. 外部键盘快捷键
```
property(nonatomic, readonly) NSArray *keyCommands
```
4. User Activities, 比如UIResponder即切换设备以后可以继续做某件事.

###  System Events

事件会被发送给application单例, application单例会分发给App的delegate从而处理对应的事件.

参考:
[understanding-cocoa-and-cocoa-touch-responder-chain](https://medium.com/ios-os-x-development/understanding-cocoa-and-cocoa-touch-responder-chain-12fe558ebe97)

[event-delivery-on-ios-part-1](https://medium.com/bpxl-craft/event-delivery-on-ios-part-1-8e68b3a3f423)

[event-delivery-on-ios-part-2](https://medium.com/bpxl-craft/event-delivery-on-ios-part-2-13f6246a88b5)

[event-delivery-on-ios-part-3](https://medium.com/bpxl-craft/event-delivery-on-ios-part-3-14463fba84b4)

[cocoa-uikit-uicontrol](http://southpeak.github.io/2015/12/13/cocoa-uikit-uicontrol/)

[cocoa-uikit-uiresponder](http://southpeak.github.io/2015/03/07/cocoa-uikit-uiresponder/)

[Hit-Testing](https://zhongwuzw.github.io/2016/09/12/iOS%E4%BA%8B%E4%BB%B6%E5%A4%84%E7%90%86%E4%B9%8BHit-Testing/)

[developer.apple.com](https://developer.apple.com/documentation/uikit/uicontrol)

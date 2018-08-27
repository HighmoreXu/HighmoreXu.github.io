---
title: AutoReleasePool
layout: post
tags: []
category: iOS
---
## 前言

AutoReleasePool字面翻译过来是"自动释放池".使用场景主要是在需要延迟释放某些对象的情况时,可以把他们放到对应的AutoReleasePool中,等AutoReleasePool生命周期结束时一起释放.这些对象会被发送autorelease消息.

下面简单分析下AutoReleasePool的实现

## 函数调用栈分析

从main函数开始
```
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```
整个iOS的应用都是包含在一个自动释放池block中的.

## autoreleasepool

可使用clang -rewrite-objc main.m命令让编译器重写文件
转化后可得到如下关键代码
```
struct __AtAutoreleasePool {
  __AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();}
  ~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}
  void * atautoreleasepoolobj;
};
```
意图很清晰,即构造的时候调用push, 析构的时候调用pop.
转换以后的main如下所示

```
int main(int argc, const char * argv[]) {
    {
        void * atautoreleasepoolobj = objc_autoreleasePoolPush();
        // do whatever you want
        objc_autoreleasePoolPop(atautoreleasepoolobj);
    }
    return 0;
}
```

继续转到定义看下Push和Pop的具体实现
```
void *objc_autoreleasePoolPush(void) {
    return AutoreleasePoolPage::push();
}
void objc_autoreleasePoolPop(void *ctxt) {
    AutoreleasePoolPage::pop(ctxt);
}
```
可以看到关键性的类AutoreleasePoolPage.

### AutoreleasePoolPage分析
```
class AutoreleasePoolPage {
    magic_t const magic;
    id *next;
    pthread_t const thread;
    AutoreleasePoolPage * const parent;
    AutoreleasePoolPage *child;
    uint32_t const depth;
    uint32_t hiwat;
};
```
上面是AutoreleasePoolPage的具体实现.我们简单分析一下.

1. AutoReleasePool没有单独的结构,是由若干个AutoReleasePoolPage以双向链表的形式组合而成的(可以看到类中的parent和child指针).一个AutoReleasePoolPage的空间被占满时,会新建一个AutoReleasePoolPage对象,连接链表,后来的autorelease对象在新的page中加入

2. AutoreleasePoolPage每个对象会开辟4096个字节内存(也就是虚拟内存一页的大小).

![autoreleasepage结构](https://raw.githubusercontent.com/HighmoreXu/BlogImage/master/images/objc-autorelease-page-in-memory.png "autoreleasepage结构")

其中56bit用于存储AutoreleasePoolPage的成员变量,剩下的都用来存储加入到自动释放池中的对象.
begin, end两个类的实例方法用于获取存储自动释放对象的边界.
next指向了下一个为空的内存地址,如果next指向的地址被加入一个objct,它就会自动移动到下一个为空的内存地址中.

3. POOL_BOUNDARY(边界对象)

`#define POOL_BOUNDARY nil`
实际上只是nil的别名.
上面介绍加入自动释放对象时,是通过next指针的移动来确定插入的位置.POOL_BOUNDARY与之相反,对应的在删除时.

每个自动释放池初始化调用objc_autoreleasePoolPush时,都会把一个POOL_BOUNDARY push到自动释放池的栈顶,并且返回这个POOL_BOUNDARY边界对象.当Pop时再将该对象传入,就可以清空中间所有加入的自动释放对象.

```
//atautoreleasepoolobj 就是POOL_BOUNDARY
int main(int argc, const char * argv[]) {
    {
        void * atautoreleasepoolobj = objc_autoreleasePoolPush();
        // do whatever you want
        objc_autoreleasePoolPop(atautoreleasepoolobj);
    }
    return 0;
}
```

4. AutoReleasePool是线程一一对应的,类中的thread指针指向当前线程
5. magin 用于对当前AutoreleasePoolPage完整性的校验

### 重点函数分析

#### 添加AutoRelease对象
```
void *objc_autoreleasePoolPush(void) {
    return AutoreleasePoolPage::push();
}
```

```
static inline void *push() {
   return autoreleaseFast(POOL_BOUNDARY);
}
```

```
static inline id *autoreleaseFast(id obj)
{
   AutoreleasePoolPage *page = hotPage();
   if (page && !page->full()) {
       return page->add(obj);
   } else if (page) {
       return autoreleaseFullPage(obj, page);
   } else {
       return autoreleaseNoPage(obj);
   }
}
```

回想之前提的,AutoReleasePool是由若干个Page以双向链表的方式连接起来的,那上面的逻辑就很清晰了.
首先获取当前所在的Page(hotPage).如果当前的Page没有满,则直接把AutoRelease加到当前Page.如果满了,则新添加一个Page,再把AutoRelease对象添加到新的Page中.如果当前Page是空的,则创建一个Page,再加入AutoRelease.

```
static id *autoreleaseFullPage(id obj, AutoreleasePoolPage *page) {
    do {
        if (page->child) page = page->child;
        else page = new AutoreleasePoolPage(page);
    } while (page->full());

    setHotPage(page);
    return page->add(obj);
}
```
查找未满的page,设置为hotPage,然后将AutoRelease对象添加到新的Page中.

```
static id *autoreleaseNoPage(id obj) {
    AutoreleasePoolPage *page = new AutoreleasePoolPage(nil);
    setHotPage(page);

    if (obj != POOL_BOUNDARY) {
        page->addPOOL_BOUNDARY);
    }

    return page->add(obj);
}
```
从头开始构建自动释放池的双向链表

#### 移除AutoRelease对象

```
void objc_autoreleasePoolPop(void *ctxt) {
    AutoreleasePoolPage::pop(ctxt);
}
```

```
static inline void pop(void *token) {
    AutoreleasePoolPage *page = pageForPointer(token);
    id *stop = (id *)token;

    page->releaseUntil(stop);

    if (page->child) {
        if (page->lessThanHalfFull()) {
            page->child->kill();
        } else if (page->child->child) {
            page->child->child->kill();
        }
    }
}
```
在当前page存放的对象大于一半时，会保留一个空的子page，
这样估计是为了可能马上需要新建page节省创建page的开销。


我们再来详细看下陌生方法的实现.
```
static AutoreleasePoolPage *pageForPointer(const void *p) {
    return pageForPointer((uintptr_t)p);
}

static AutoreleasePoolPage *pageForPointer(uintptr_t p) {
    AutoreleasePoolPage *result;
    uintptr_t offset = p % SIZE;

    assert(offset >= sizeof(AutoreleasePoolPage));

    result = (AutoreleasePoolPage *)(p - offset);
    result->fastcheck();

    return result;
}
```
之前说过,每个page占4096. 与4096取模,得到当前指针的偏移量.因为所有的 AutoreleasePoolPage 在内存中都是对齐的.最后调用fastCheck()来检查当前的result是不是一个AutoreleasePoolPage(检查magic_t 结构体中的某个成员是否为 0xA1A1A1A1).

```
void releaseUntil(id *stop) {
    while (this->next != stop) {
        AutoreleasePoolPage *page = hotPage();

        while (page->empty()) {
            page = page->parent;
            setHotPage(page);
        }

        page->unprotect();
        id obj = *--page->next;
        memset((void*)page->next, SCRIBBLE, sizeof(*page->next));
        page->protect();

        if (obj != POOL_BOUNDARY) {
            objc_release(obj);
        }
    }

    setHotPage(this);
}
```
```
void kill() {
    AutoreleasePoolPage *page = this;
    while (page->child) page = page->child;

    AutoreleasePoolPage *deathptr;
    do {
        deathptr = page;
        page = page->parent;
        if (page) {
            page->unprotect();
            page->child = nil;
            page->protect();
        }
        delete deathptr;
    } while (deathptr != this);
}
```
删除当前page及所有child的page.

## autorelease方法

```
- [NSObject autorelease]
└── id objc_object::rootAutorelease()
    └── id objc_object::rootAutorelease2()
        └── static id AutoreleasePoolPage::autorelease(id obj)
            └── static id AutoreleasePoolPage::autoreleaseFast(id obj)
                ├── id *add(id obj)
                ├── static id *autoreleaseFullPage(id obj, AutoreleasePoolPage *page)
                │   ├── AutoreleasePoolPage(AutoreleasePoolPage *newParent)
                │   └── id *add(id obj)
                └── static id *autoreleaseNoPage(id obj)
                    ├── AutoreleasePoolPage(AutoreleasePoolPage *newParent)
                    └── id *add(id obj)
```

看懂之前的代码流程,这个函数调用栈也就很清晰了,对象调用autorelease最终就是将当前对象加入到autoreleasepage中.


## 总结

* AutoReleasePool是由AutoreleasePoolPage以双向链表来实现的.
* 当对象调用autorelease方法时,会将对象加入到AutoreleasePoolPage中
* Push和Pop分别从自动释放池中增加和删除autorelease对象.
* APP启动以后,苹果在主线程注册Observer. 第一个Observer监测Entry, 会调用Push.第二个observer监测BeforeWaiting,会调用Pop和Push.Exit时地阿偶用Pop来释放自动释放池.


## 引用

[自动释放池的前世今生 ---- 深入解析 autoreleasepool](https://draveness.me/autoreleasepool)





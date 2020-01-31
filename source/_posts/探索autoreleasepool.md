---
title: 探索autoreleasepool
date: 2020-01-17 19:21:53
tags: iOS
categories: iOS
---

# 前言
本文纯属是根据前人对`autoreleasepool`的分析学习和苹果文档、源码的一次学习笔记，内容大部分来自引用。本人所做的工作仅是按照前人的笔记手动实践一遍梳理原理并记录供日后方便回顾。

通常来说，研究`Objective-C`必备的源码

[objc4](https://opensource.apple.com/source/objc4/)

# 引出autoreleasepool
iOS应用程序在默认创建时，`main`函数的内容都有一个`autorelease`块包裹函数体

```objc
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```

从iOS内存管理的内容可以得知，这个自动释放池块对应着主线程，伴随着整个应用程序生命周期，当我们手动退出应用程序，整个自动释放池内的对象都将被释放，因此不会出现内存泄漏。

另外，从`objc4`源码对`autorelease pool`实现中的注释中可以获得一些相关信息

```
/***********************************************************************
   Autorelease pool implementation

   A thread's autorelease pool is a stack of pointers. 
   Each pointer is either an object to release, or POOL_BOUNDARY which is 
     an autorelease pool boundary.
   A pool token is a pointer to the POOL_BOUNDARY for that pool. When 
     the pool is popped, every object hotter than the sentinel is released.
   The stack is divided into a doubly-linked list of pages. Pages are added 
     and deleted as necessary. 
   Thread-local storage points to the hot page, where newly autoreleased 
     objects are stored. 
**********************************************************************/
```

以下是我生硬的翻译

```
一个线程的自动释放池是一个保存很多指针的栈。
每一个指针指向的要么是需要释放的对象，要么是自动释放池的边界POOL_BOUNDARY
自动释放池的token指向该池的POOL_BOUNDARY,当自动释放池销毁时，所有比POOL_BOUNDARY（边界对象）高的对象都会被released
自动释放池栈由双向链表构成，链表结点是一个page(对应下文的AutoreleasePoolPage)，page可以按需添加或删除
线程本地存储指向hotpage，所谓hot page是指最近有autoreleased对象被存储进来的page
```



# 研究autoreleasepool

从顶层代码往下研究，逐步追溯到原理实现层。

使用命令用编译器将`main.m`转换为底层处理代码

```sh
clang -x objective-c -rewrite-objc -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator13.2.sdk main.m
```

查看生成的`main.cpp`，得到以下关键代码

```c
struct __AtAutoreleasePool {
  __AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();}
  ~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}
  void * atautoreleasepoolobj;
};

int main(int argc, char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 
        return UIApplicationMain(argc, argv, __null, NSStringFromClass(((Class (*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("AppDelegate"), sel_registerName("class"))));
    }
}
```

从中可以得到的信息是，`@autoreleasepool`转换成了一个C++结构体实例,而该结构体为`__AtAutoreleasePool`

`__AtAutoreleasePool`结构体中分别有构造函数`__AtAutoreleasePool ()`和析构函数`~__AtAutoreleasePool()`以及`atautoreleasepoolobj `成员变量

结合转换代码，根据局部变量超出作用域的规则，可手动将`main`函数改写成以下形式

```objc
int main(int argc, char * argv[]) {
    {
        void * atautoreleasepoolobj = objc_autoreleasePoolPush();
        
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
        
        objc_autoreleasePoolPop(atautoreleasepoolobj);
    }
    return 0;
}
```

得到以上代码后，我们就可以顺藤摸瓜，沿着`objc_autoreleasePoolPush()`以及`objc_autoreleasePoolPop()`继续深入

由于是`objc`前缀的函数，我们可以想到从`objc4`的源码中搜索，搜索发现如下

```c
void * objc_autoreleasePoolPush(void) {
    return AutoreleasePoolPage::push();
}

void objc_autoreleasePoolPop(void *ctxt) {
    AutoreleasePoolPage::pop(ctxt);
}
```

同样在源码中搜索可以发现`push()`和`pop()`都是C++类`AutoreleasePoolPage`内的静态方法,接下来分析该类及其两个函数

## AutoreleasePoolPage

根据源码中`autoreleasepool`实现的注释可知,每一个自动释放池由结点为`AutoreleasePoolPage`的双向链表组成，在源码中得到`AutoreleasePoolPage`类的定义

```c++
class AutoreleasePoolPage {
    // EMPTY_POOL_PLACEHOLDER is stored in TLS when exactly one pool is 
    // pushed and it has never contained any objects. This saves memory 
    // when the top level (i.e. libdispatch) pushes and pops pools but 
    // never uses them.
#   define EMPTY_POOL_PLACEHOLDER ((id*)1)

#   define POOL_BOUNDARY nil
    static pthread_key_t const key = AUTORELEASE_POOL_KEY;
    static uint8_t const SCRIBBLE = 0xA3;  // 0xA3A3A3A3 after releasing
    static size_t const SIZE = 
#if PROTECT_AUTORELEASEPOOL
        PAGE_MAX_SIZE;  // must be multiple of vm page size
#else
        PAGE_MAX_SIZE;  // size and alignment, power of 2
#endif
    static size_t const COUNT = SIZE / sizeof(id);

    magic_t const magic;
    id *next;
    pthread_t const thread;
    AutoreleasePoolPage * const parent;
    AutoreleasePoolPage *child;
    uint32_t const depth;
    uint32_t hiwat;

    ...
}
```

* PAGE_MAX_SIZE page的大小以及对齐，2的幂 (此处为4096字节，也就是4KB，虚拟内存的一页)
* magic 用于对当前AutoreleasePoolPage完整性的校验
* thread 保存当前page所在的线程
* parent、child 双向链表使用的指针

--- 

虚拟化一个page，它的结构如下

<div align=center> 	<img src="https://kinkenyuen.oss-cn-shenzhen.aliyuncs.com/images/page结构.png" width="800" /> 	<div style="color:orange; border-bottom: 1px solid #d9d9d9;     	display: inline-block;     	color: #999;     	padding: 2px;">page结构</div> </div>

接下来是对该结构的一些解释

```c++
AutoreleasePoolPage(AutoreleasePoolPage *newParent) 
    : magic(), next(begin()), thread(pthread_self()),
      parent(newParent), child(nil), 
      depth(parent ? 1+parent->depth : 0), 
      hiwat(parent ? parent->hiwat : 0)
{ 
    if (parent) {
        parent->check();
        assert(!parent->child);
        parent->unprotect();
        parent->child = this;
        parent->protect();
    }
    protect();
}

id * begin() {
    return (id *) ((uint8_t *)this+sizeof(*this));
}

id * end() {
    return (id *) ((uint8_t *)this+SIZE);
}
```

根据类实例的构造函数和`begin()`、`end()`实例方法，`next`指针指向类实例所占内存的下一位置。

可以发现前人的文章提到了一个**哨兵对象**,但在最新版的`objc4`里已经找不到，取而代之的是
`POOL_BOUNDARY`,这里我暂且称之为**边界对象**,同样地，它是个`nil`的别名。

这个`POOL_BOUNDARY`在`page`压入第一个对象指针时（也是`page`初始化的时候）被压入，并且返回这个**边界对象**。

```c++
static inline void *push() {
    id *dest;
    if (DebugPoolAllocation) {
        // Each autorelease pool starts on a new pool page.
        dest = autoreleaseNewPage(POOL_BOUNDARY);
    } else {
        dest = autoreleaseFast(POOL_BOUNDARY);
    }
    assert(dest == EMPTY_POOL_PLACEHOLDER || *dest == POOL_BOUNDARY);
    return dest;
}
    
static __attribute__((noinline)) id *autoreleaseNewPage(id obj) {
    AutoreleasePoolPage *page = hotPage();
    if (page) return autoreleaseFullPage(obj, page);
    else return autoreleaseNoPage(obj);
}
    
static __attribute__((noinline)) id *autoreleaseNoPage(id obj) {
    // "No page" could mean no pool has been pushed
    // or an empty placeholder pool has been pushed and has no contents yet
    assert(!hotPage());

    bool pushExtraBoundary = false;
    if (haveEmptyPoolPlaceholder()) {
        // We are pushing a second pool over the empty placeholder pool
        // or pushing the first object into the empty placeholder pool.
        // Before doing that, push a pool boundary on behalf of the pool 
        // that is currently represented by the empty placeholder.
        pushExtraBoundary = true;
    }
    else if (obj != POOL_BOUNDARY  &&  DebugMissingPools) {
        // We are pushing an object with no pool in place, 
        // and no-pool debugging was requested by environment.
        _objc_inform("MISSING POOLS: (%p) Object %p of class %s "
                     "autoreleased with no pool in place - "
                     "just leaking - break on "
                     "objc_autoreleaseNoPool() to debug", 
                     pthread_self(), (void*)obj, object_getClassName(obj));
        objc_autoreleaseNoPool(obj);
        return nil;
    }
    else if (obj == POOL_BOUNDARY  &&  !DebugPoolAllocation) {
        // We are pushing a pool with no pool in place,
        // and alloc-per-pool debugging was not requested.
        // Install and return the empty pool placeholder.
        return setEmptyPoolPlaceholder();
    }

    // We are pushing an object or a non-placeholder'd pool.

    // Install the first page.
    AutoreleasePoolPage *page = new AutoreleasePoolPage(nil);
    setHotPage(page);
    
    // Push a boundary on behalf of the previously-placeholder'd pool.
    if (pushExtraBoundary) {
        page->add(POOL_BOUNDARY);
    }
    
    // Push the requested object or pool.
    return page->add(obj);
}

id *add(id obj) {
    assert(!full());
    unprotect();
    id *ret = next;  // faster than `return next-1` because of aliasing
    *next++ = obj;
    protect();
    return ret;
}
```

因此`atautoreleasepoolobj`就是`POOL_BOUNDARY`

```c
void * atautoreleasepoolobj = objc_autoreleasePoolPush();
```

对`AutoreleasePoolPage`的结构有了一定的了解后，接着对`push()`和`pop()`进行梳理

### objc_autoreleasePoolPush

回顾该函数

```c
void *objc_autoreleasePoolPush(void) {
    return AutoreleasePoolPage::push();
}
```

它调用`AutoreleasePoolPage`的类方法`push`

```c++
static inline void *push()  {
    id *dest;
    if (DebugPoolAllocation) {
        // Each autorelease pool starts on a new pool page.
        dest = autoreleaseNewPage(POOL_BOUNDARY);
    } else {
        dest = autoreleaseFast(POOL_BOUNDARY);
    }
    assert(dest == EMPTY_POOL_PLACEHOLDER || *dest == POOL_BOUNDARY);
    return dest;
}
```

在这里会进入一个比较关键的方法`autoreleaseFast`, 并传入边界对象
 
```c++
static inline id *autoreleaseFast(id obj) {
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
有三个分支，分别为

* 存在hotpage且page不满
	* 调用add方法将对象指针压入AutoreleasePoolPage结构中
* 存在hotpage但page已满
	* 调用autoreleaseFullPage新建一个结点page
	* 调用add方法将对象指针压入AutoreleasePoolPage结构中
* 无hotpage
	* 调用autoreleaseNoPage	创建一个page，并设置其为hotpage
	* 调用add方法将对象指针压入AutoreleasePoolPage结构中

>hotpage可以理解为当前正在使用、活跃的AutoreleasePoolPage

`add`方法很简单，具体操作就是将对象指针放入`page`栈中，并移动`next`指针

### objc_autoreleasePoolPop

```c++
static inline void pop(void *token) {
    AutoreleasePoolPage *page;
    id *stop;

    if (token == (void*)EMPTY_POOL_PLACEHOLDER) {
        // Popping the top-level placeholder pool.
        if (hotPage()) {
            // Pool was used. Pop its contents normally.
            // Pool pages remain allocated for re-use as usual.
            pop(coldPage()->begin());
        } else {
            // Pool was never used. Clear the placeholder.
            setHotPage(nil);
        }
        return;
    }

    page = pageForPointer(token);
    stop = (id *)token;
    if (*stop != POOL_BOUNDARY) {
        if (stop == page->begin()  &&  !page->parent) {
            // Start of coldest page may correctly not be POOL_BOUNDARY:
            // 1. top-level pool is popped, leaving the cold page in place
            // 2. an object is autoreleased with no pool
        } else {
            // Error. For bincompat purposes this is not 
            // fatal in executables built with old SDKs.
            return badPop(token);
        }
    }

    if (PrintPoolHiwat) printHiwat();

    page->releaseUntil(stop);

    // memory: delete empty children
    if (DebugPoolAllocation  &&  page->empty()) {
        // special case: delete everything during page-per-pool debugging
        AutoreleasePoolPage *parent = page->parent;
        page->kill();
        setHotPage(parent);
    } else if (DebugMissingPools  &&  page->empty()  &&  !page->parent) {
        // special case: delete everything for pop(top) 
        // when debugging missing autorelease pools
        page->kill();
        setHotPage(nil);
    } 
    else if (page->child) {
        // hysteresis: keep one empty child if page is more than half full
        if (page->lessThanHalfFull()) {
            page->child->kill();
        }
        else if (page->child->child) {
            page->child->child->kill();
        }
    }
}
```

自动释放池析构时调用上述`pop`方法，传入的参数为**边界对象**,该静态方法共做了三件主要事情

1. 使用pageForPointer获取当前边界对象所在的page
2. 将stop标记设置为当前边界对象的位置，调用releaseUntil，直到stop位置
3. 调用child的kill方法清理链表中的子结点

`releaseUntil`方法遍历栈，获取放在`pool`对象的指针，调用`objc_release`函数释放对象所占内存

## autorelease方法
根据`objc4`得出`autorelease`方法的调用栈，如下

```c
autorelease -> rootAutorelease -> rootAutorelease2 -> AutoreleasePoolPage::autorelease -> autoreleaseFast
```

最终会调用`autoreleaseFast`方法，将对象指针压入当前`AutoreleasePoolPage`中

## 小结
梳理了一遍自动释放池的实现和`autorelease`方法，对iOS管理模型又加深了一点理解

* 自动释放池是由 AutoreleasePoolPage 以双向链表的方式实现的
* 当对象调用 autorelease 方法时，会将对象加入 AutoreleasePoolPage 的栈中
* 调用 AutoreleasePoolPage::pop 方法会向栈中的对象发送 release 消息

## Reference

[自动释放池的前世今生 ---- 深入解析 autoreleasepool](https://draveness.me/autoreleasepool)

[黑幕背后的Autorelease](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/)











---
title: iOS内存管理
date: 2020-01-16 19:27:19
tags: iOS
categories: iOS
---

# 内存使用
应用的内存消耗主要分为两部分：栈大小和堆大小

## 栈大小

应用中的每个线程都有专用的栈空间，栈可以在线程存在期间自由使用。线程的最大栈空间很小，因此有很多限制

* 限制递归调用的最大方法数
* 方法中使用的参数个数和内部变量个数
* 视图层级的最大深度
* ...

## 堆大小

每个进程的所有线程共享一个堆。OC对象如NSString、UIImage、视图等都会消耗堆内存。**大多数情况下，OC对象存放在堆中**,**与通过类创建的对象相关的所有数据也都存放在堆中**。

# 内存管理模型

苹果LLVM官方文档使用术语`持有关系`和`引用计数`来描述OC对象的内存管理。

如果一个对象处于被持有状态，那么它占用的内存就不能被回收。

当一个对象在某个方法内部创建时，那么该方法就持有这个对象。如果对象从方法中返回，则方法调用者声称建立了持有关系。对象"赋值"给其他变量，对应的变量同样声称建立了持有关系。

一个对象的持有者数量称之为引用计数，当持有者与被持有对象建立持有关系时，被持有对象的引用计数增加1，同理，解除持有关系时，引用计数减少1，当引用计数降为0时，该对象被释放，对应相关的内存会被回收。

管理形式分为手动引用计数(`manual reference counting,MRC`)与自动引用计数(`automatic reference counting, ARC`)

## MRC 
虽然MRC现在已经十分罕见，但理解MRC对内存管理有很大帮助

```objc
@interface MyObject : NSObject
@end

@implementation MyObject
- (void)dealloc
{
    NSLog(@"%s",__func__);
    [super dealloc];
}
@end

@interface ViewController ()
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    MyObject *obj = [[MyObject alloc] init];//1
    NSLog(@"retainCount : %lu",obj.retainCount);	
    MyObject *objRetained = [obj retain];//2
    NSLog(@"retainCount : %lu",obj.retainCount);
    [objRetained release];//3
    NSLog(@"retainCount : %lu",obj.retainCount);
    [obj release];//4
}

@end
```
```
2020-01-16 11:21:35.447154+0800 block[999:19673] retainCount : 1
2020-01-16 11:21:35.447242+0800 block[999:19673] retainCount : 2
2020-01-16 11:21:35.447270+0800 block[999:19673] retainCount : 1
2020-01-16 11:21:35.447411+0800 block[999:19673] -[MyObject dealloc]
```

1. 创建对象,obj建立了持有关系，引用计数为1
2. objRetained建立了持有关系，引用计数增加为2
3. objRetained解除了持有关系，引用计数降为1
4. obj解除持有关系，引用计数降为0，对象调用析构方法dealloc

___

方法中的引用计数

```objc
@interface MyObject : NSObject
- (NSString *)name;
@end

@implementation MyObject

- (NSString *)name {
    NSString *result = [[NSString alloc] initWithFormat:@"%@\n",@"kinken"];//1
    return result;
}

- (void)dealloc
{
    NSLog(@"%s",__func__);
    [super dealloc];
}
@end

@interface ViewController ()
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    MyObject *obj = [[MyObject alloc] init];
    NSString *name = [obj name];//2
    NSLog(@"obj's name : %@",name);
    [name release];//3
    [obj release];
}
```
```
2020-01-16 12:43:50.861654+0800 block[1026:28263] obj's name : kinken
2020-01-16 12:43:50.861756+0800 block[1026:28263] -[MyObject dealloc]
```

1. 首次创建对象，`name`指向内存的引用计数为1,`name`方法持有`NSString`对象
2. 通过`name`指向的内存引用计数仍然为1。在`viewDidLoad`方法中通过调用`[obj name]`引用了对象，获得了该对象，此时不应该再次持有(`retain`)。(`name`方法内部通过`alloc`方法创建对象已经`retain`建立持有关系)
3. 使用完毕，解除持有关系，引用计数为0

`viewDidLoad`并不知道`name`方法是创建了一个新的对象还是重用旧的对象。但是它知道对象的引用计数加1后会被返回，因此这里没有继续持有(`retain`)`name`。

我的理解是当对象被作为返回值返回时，内部会建立一个新的持有关系（当前例子中`Myobject`类的`name`方法内部创建`NSString`类的对象并持有），你不需要再次`retain`来再次持有，只需要在使用完毕后手动`release`对象。

## Autoreleasing Objects

自动释放对象（`Autoreleasing Objects`），让你解除对一个对象的持有关系，但延后对它的销毁。当在方法中创建一个对象并且需要将其返回时，自动释放就非常有用。自动释放可以帮助MRC管理对象的生命周期。

沿用上一节的例子，从表面上看，没有什么能够表示`name`方法持有了返回的字符串对象，因此在`viewDidLoad`方法内也不应该`release`返回的字符串，这有可能会导致内存泄漏（释放一块未知的内存区域）。而加入`[name release]`的原因是我们事先知道name方法内部是通过`alloc`创建了对象并持有(`retain`)。

类似于这种**创建对象并持有**的情况有以下

* alloc
* new 
* copy
* mutableCopy

那么，在不知道方法内部是如何创建对象的情况下，外部的方法调用者如何正确管理返回值对象的生命周期

有两种可能的解决方案

* 不使用alloc或相关的方法
* 对返回的对象使用使用autorelease延时释放

第一种

```objc
@interface MyObject : NSObject
- (NSString *)name;
@end

@implementation MyObject

- (NSString *)name {
    NSString *result = [NSString stringWithFormat:@"%@",@"kinken"];//1
    return result;
}

- (void)dealloc
{
    NSLog(@"%s",__func__);
    [super dealloc];
}
@end

@interface ViewController ()
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    MyObject *obj = [[MyObject alloc] init];
    NSString *name = [obj name];
    NSLog(@"obj's name : %@",name);
    //2
    [obj release];
}
```

1. 不使用alloc方法
2. 实际上stringWithFormat返回的是一个autorelease对象，返回的时候已经解除持有关系，只是还没有销毁

第一种方案实际上是第二种方案的一个例子

---

第二种

`NSObject`协议定义了`autorelease`方法，可在从方法中返回对象时使用它返回一个`autorelease`对象

```objc
- (NSString *)name {
    NSString *result = [[[NSString alloc] initWithFormat:@"%@\n",@"kinken"] autorelease];
    return result;
}
```
* 通过alloc创建对象并持有
* autorelease表示解除持有关系，但是延时释放，即允许方法的调用者在对象被释放之前使用该对象

## Autorelease Pool Blocks
自动释放池块(`Autorelease Pool Blocks`)是允许你解除对一个对象的持有关系，但可避免对象立即被销毁回收的一个工具。

它能确保在块内创建的对象在块完成时被回收。这在创建了多个对象的场景中非常有用，因为在块中可以提早释放不再需要的对象，从而降低内存用量。

iOS中常见的代码

```objc
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```

用`@autoreleasepool{}`表示自动释放池块，块中收到`autorelease`消息的所有对象都会在`autoreleasepool`块结束时收到`release`消息。每调用一次`autorelease`会向对象发送一次`release`消息，这意味着如果一 个对象收到了不止一次的 `autorelease `消息，那它也会多次收到` release `消息。

`main`方法的代码中可以发现，整个应用的代码都在自动释放池块中，这意味着应用程序关闭退出后，所有的`autorelease`对象最后都会被回收,不会导致内存泄漏。

---

嵌套`autoreleasepool`块

```objc
@autoreleasepool { 
    // 一些代码 
    @autoreleasepool {
    // 更多代码 
    }
}
```

这样操作的目的是**提前执行对象的销毁回收**

>Cocoa 框架希望代码能在 autoreleasepool 块内执行，否则 autorelease 对象将无法被 释放，从而导致应用发生内存泄漏。

>AppKit 和 UIKit 框架将事件 - 循环的迭代放入了 autoreleasepool 块中。因此，通常 不需要你自己再创建 autoreleasepool 块了。

---

说说手动创建`autoreleasepool`的情况

* 当有需要创建很多临时对象的循环时，在循环中使用`autoreleasepool`提早释放内存，降低内存使用总和峰值
* 当你手动创建一个线程时，每个线程都需要有它自己的`autoreleasepool`栈。主线程用自己的`autoreleasepool`启动，因为它来自统一生成的代码(类似于上述的main)。然而，对于任何自定义的线程，在线程执行开始的时候，必须创建线程对应的`autoreleasepool`，否则会有内存泄漏。
* 当你编写一个不是基于UI框架的程序，如命令行工具

情况一，循环中的`autoreleasepool `

```objc
//代码块1
{
    @autoreleasepool {
        NSUInteger *userCount = userDatabase.userCount;
        for(NSUInteger *i = 0; i < userCount; i++) {
            Person *p = [userDatabase userAtIndex:i];
            NSString *fname = p.fname; if(fname == nil) {
                fname = [self askUserForFirstName];
            }
            NSString *lname = p.lname; if(lname == nil) {
                lname = [self askUserForLastName];
            }
            //...
            [userDatabase updateUser:p];
        }
    }
}
```

```objc
//代码块2
{
    @autoreleasepool {
        NSUInteger *userCount = userDatabase.userCount; 
        for(NSUInteger *i = 0; i < userCount; i++) {
            @autoreleasepool {
                Person *p = [userDatabase userAtIndex:i];
                NSString *fname = p.fname; if(fname == nil) {
                    fname = [self askUserForFirstName];
                }
                NSString *lname = p.lname; if(lname == nil) {
                    lname = [self askUserForLastName];
                }
                //...
                [userDatabase updateUser:p];
            }
        }
    }
}
```

1. 代码块1只有一个`autoreleasepool`,内存释放回收需要在所有的迭代完成之后才开始进行
2. 代码块2在循环中内嵌`autoreleasepool`确保每次循环迭代完成后**立即释放回收对象内存**，从而减少内存最大用量

情况二，自定义线程中必须创建自己的`autoreleasepool`

```objc
- (void)viewDidLoad {
    [super viewDidLoad];

    NSThread *newThread = [[NSThread alloc] initWithTarget:self selector:@selector(newThreadStart:) object:nil];
    [newThread start];
}

- (void)newThreadStart:(id)obj {
    @autoreleasepool {
        NSLog(@"newThreadStart");
    }
}
```

## ARC
使用MRC需要开发人员在适当位置频繁使用`retain`、`release`和`autorelease`,一方面需要开发人员十分熟悉MRC的内存管理规则，另一方面需要花费很多精力编写重复的代码。ARC解决方案在这样的环境下提出，目前OC与Swift均使用ARC。

ARC是一种编译器特性,它分析对象在代码中的生命周期，并在编译期自动插入适合的内存管理代码(`retain`,`release`等)。同时编译期还会生成适合的`dealloc`方法对对象内存做清理工作。这大大减轻了开发人员的内存管理负担。

---

为了规范ARC内存管理的模型，使用ARC需要遵循的一些规则

* 不能实现或调用 retain、release、autorelease 或 retainCount 方法。这一限制不仅针 对对象，对选择器同样有效。因此，[obj release]或@selector(retain)是编译时的错误。
* 可以实现 dealloc 方法，但不能调用它们。不仅不能调用其他对象的 dealloc 方法，也
不能调用超类。[super dealloc] 是编译时的错误。但你仍然可以对 Core Foundation 类型的对象调用 CFRetain、CFRelease 等相关方法。
* 不能调用 NSAllocateObject 和 NSDeallocateObject 方法。应使用 alloc 方法创建对象， 运行时负责回收对象
* 不能在 C 语言的结构体内使用对象指针。
* 不能在 id 类型和 void * 类型之间自动转换。如果需要，那么你必须做显式转换。
* 不能使用 NSAutoreleasePool，要替换使用 autoreleasepool 块。
* 不能使用 NSZone 内存区域。
* 属性的访问器名称不能以 new 开头，以确保与 MRC 的互操作性。

### 引用类型
* 强引用，默认的引用类型。被强引用指向的对象内存不会被释放。强引用会对引用计数加1，从而延长对象的生命周期
* 弱引用，不会增加引用计数类型，因而不会延长对象的生命周期。

#### 变量限定符
ARC为变量提供4种生命周期限定符

* \_\_strong，默认限定符，无需显式引入。只要有强引用指向，对象就会长时间驻留在内存堆中。可以将__strong理解为retain调用的ARC版本
* \_\_weak，表示引用不会保持被引用对象的存货。当没有强引用指向对象，弱引用会被置为nil。可将\_\_weak 看作是 assign 操作符的 ARC 版本，只是对象被回收时，__weak具有安全性——指针将自动被设置为 nil。
* \_\_unsafe_unretained, 与 \_\_weak 类似，只是当没有强引用指向对象时，\_\_unsafe_unretained 不会被置为 nil。 可将其看作 assign 操作符的 ARC 版本。
* \_\_autoreleasing，用于由引用使用id *传递的消息参数。它预期了autorelease方法会在传递参数的方法中被调用

第四点解释

```objc
- (BOOL)moveItemAtPath:(NSString *)srcPath toPath:(NSString *)dstPath error:(NSError **)error API_AVAILABLE(macos(10.5), ios(2.0), watchos(2.0), tvos(9.0));

编译时实际是

- (BOOL)moveItemAtPath:(NSString *)srcPath toPath:(NSString *)dstPath error:(NSError * __autoreleasing*)error API_AVAILABLE(macos(10.5), ios(2.0), watchos(2.0), tvos(9.0));
```

变量限定符的使用情况

```objc
MyObject * __strong obj1 = [[MyObject alloc] init];//1
MyObject * __weak obj2 = [[MyObject alloc] init];//2
MyObject * __unsafe_unretained obj3 = [[MyObject alloc] init];//3
MyObject * __autoreleasing obj = [[MyObject alloc] init];//4
```

1. 创建对象后引用计数为 1，并且对象在 obj1引用期间不会被回收。
2. 创建对象后引用计数为 0，对象会被立即释放，且 obj2 将被设置为 nil。
3. 创建对象后引用计数为 1，对象会被立即释放，但 obj3不会被设置为 nil。
4. 创建对象后引用计数为 1，当方法返回时对象会被自动释放。

#### 属性限定符
属性声明根据引用类型新增两个持有关系限定符:`strong`和`weak`。另外`assign`的语义也更新了。总的来说共有6个限定符。

* strong，默认，指定了__strong强引用关系
* weak，指定__weak弱引用关系
* assign，MRC环境是默认的持有关系限定符，ARC环境是__unsafe_unretained关系
* copy，指定__strong的强引用关系同时，还表示使用setter中的copy语义(对象的copy)
* retain，指定__strong强引用关系
* unsafe_unretained,指定了\_\_unsafe_unretained关系

`assign`和`unsafe_unretained`只进行值复制而没有任何实质性的检查，所以它们仅用于值类型(BOOL、NSInteger等等)。避免将它们用于引用类型或指针类型

属性限定符的使用情况

```objc
@property (nonatomic, strong) UIView *myView;
@property (nonatomic, weak) id<UIApplicationDelegate> delegate;
@property (nonatomic, assign) UIView *myWrongView;//1
@property (nonatomic, assign) BOOL *myFlag;
@property (nonatomic, copy) NSString *myName;
@property (nonatomic, retain) UIView *mySecondView;//2
@property (nonatomic, unsafe_unretained) UIView *myThirdView;
```

1. 错误将assign用于指针
2. 老古董用法，现在基本不会出现

## 内存管理规则

苹果[官方文档](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmRules.html#//apple_ref/doc/uid/20000994-BAJHFBGH)中描述，内存管理的四个基本规则

* 你拥有所有自己创建的对象，如通过new、alloc、copy或mutableCopy创建
* 用MRC中的retain或ARC中的__strong引用来拥有任何对象的持有关系
* 当不需要某个对象时，使用MRC中的release来放弃对该对象的持有关系，而在ARC中无需任何特殊操作，持有关系会在对象失去最后的引用（通常为方法中的最后一行代码）时销毁回收。

# Reference
* 《High Performance iOS Apps》
* 《Objective-C高级编程》

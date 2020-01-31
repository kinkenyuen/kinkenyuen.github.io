---
title: Block
date: 2020-01-11 20:58:22
tags: iOS
categories: iOS
---

# block

block是OC对闭包的实现，闭包的定义如下:
>在计算机科学中，闭包（英语：Closure），又称词法闭包（Lexical Closure）或函数闭包（function closures），是引用了自由变量的函数。这个被引用的自由变量将和这个函数一同存在，即使已经离开了创造它的环境也不例外。所以，有另一种说法认为闭包是由函数和与其相关的引用环境组合而成的实体。闭包在运行时可以有多个实例，不同的引用环境和相同的函数组合可以产生不同的实例。

block用来实现匿名函数的特性，是一种特殊数据类型，可以定义为变量、作为参数、作为返回值，一般用来保存一段代码，在需要的时候回调，在iOS中广泛使用，如GCD、动画变换、网络回调等等

# block基础
表达式

`returnType (^blockName)(parameterTypes)`

## block变量声明

```objc
//声明无返回值，参数为空，名为aBlock的block变量
void(^aBlock)(void);

//声明无返回值，参数为一个NSString对象，名为aBlock的block变量
void(^aBlock)(NSString *aString);

//其中形参名可省略
void(^aBlock)(NSString *);
```

## block变量赋值
本质 ： block变量指向block结构体实例

```
block变量 = ^(参数列表){
    //code	
}
```

```objc
aBlock = ^(NSString *aString){
    NSLog(@"aString:%@",aString);
};
```

>block变量的赋值格式可以是: `block变量 = ^返回值类型(参数列表){函数体};`不过通常情况下都将返回值类型省略,因为编译器可以从存储代码块的变量中确定返回值的类型

## 声明block变量的同时进行赋值

在Xcode中输入`inlineBlock`代码提示如下:

```objc
returnType(^blockName)(parameterTypes) = ^(parameters) {
    statements
};
```

```objc
void(^aBlock)(NSString *) = ^(NSString *aString) {
    NSLog(@"aString:%@",aString);
};
```

## 调用block
通常传入某类实例内部以供回调,一般不会在同一处给block变量赋值以及调用，开发中常用作传递信息（也就是常说的回调),此处仅用于展示。

```objc
void(^aBlock)(NSString *) = ^(NSString *aString) {
    NSLog(@"aString:%@",aString);
};
    
aBlock(@"kinken");
```

## block类型定义
使用`typedef`使block的类型定义变得更加清晰,在Xcode输入`typedefBlock`代码提示快捷键

`typedef returnType(^name)(arguments); `

`typedef void(^aBlockType)(NSString *);`

`aBlockType`即表示`void(^)(NSString *)`类型

```objc
aBlockType aBlock = ^(NSString *aString) {
    NSLog(@"aString:%@",aString);
}
```

## block作函数参数
### C函数参数
```c
//一个使用block作为c函数形参
void blockPara(NSString *(^aBlock)(NSString *)) {
    NSLog(@"log:%@",aBlock(@"ken"));
}

//定义block变量并赋值
NSString * (^aBlock)(NSString *) = ^(NSString *aString) {
    return aString;
};
    
//调用c函数，传入block
blockPara(aBlock);
```

### OC函数参数

```objc
- (void)blockPara:(NSString * (^)(NSString *))aBlock {
    NSLog(@"log:%@",aBlock(@"ken"));
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    NSString * (^aBlock)(NSString *) = ^(NSString *aString) {
        return aString;
    };
    
    [self blockPara:aBlock];
    //或者这样直接传block块，比较常用
    [self blockPara:^NSString *(NSString *aString) {
        return aString;
    }];
}
```

## block内访问外部局部变量
在block中可以访问block之外的局部变量，默认情况下，block访问的仅仅是变量的瞬时值，即在声明block变量并赋值之后，访问的局部变量值已经确定，若在block调用前修改局部变量的值，不会影响到block内访问的那个值

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    int var = 100;
    void(^aBlock)(void) = ^(void) {
        NSLog(@"%d",var);
    };
    var = 0;
    
    aBlock();
}

//控制台
2019-02-15 17:34:10.061709+0800 Test[4963:231428] 100
```

## block内修改外部局部变量

在block中不能直接修改外部局部变量,会报一个`Variable is not assignable (missing __block type specifier)`错误。

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    int var = 100;
    void(^aBlock)(void) = ^(void) {
        var++;//Variable is not assignable (missing __block type specifier)
        NSLog(@"var:%d",var);
    };
    var = 0;
    
    aBlock();
}
```
若需要在block内部使用并修改block外部的局部变量值，需要使用`__block`修饰符修饰该局部变量。原因可以在[__block修饰符](#jump)找到

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    __block int var = 100;
    void(^aBlock)(void) = ^(void) {
        var++;
        NSLog(@"var:%d",var);
    };
    var = 0;
    
    aBlock();
}

//控制台
2019-02-15 19:03:19.104961+0800 Test[6996:285534] var:1
```

## block内访问与修改全局变量

```objc
int var = 100;

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    void(^aBlock)(void) = ^(void) {
        var++;
        NSLog(@"var:%d",var);
    };
    aBlock();
}

//控制台
2019-02-15 19:08:06.800896+0800 Test[7132:289302] var:101
```
block中能直接访问并修改全局变量，原因在于`程序装载到内存后，全局变量存储在全局数据区，生命周期直至程序结束`，当然这仅是很浅显的原因，具体涉及到block是**如何捕获变量**

## block内访问与修改静态变量

```objc
//static int var = 100;

@implementation ViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    static int var = 100;
    void(^aBlock)(void) = ^(void) {
        var++;
        NSLog(@"var:%d",var);
    };
    var = 0;
    aBlock();
}

@end

//控制台
2019-02-15 19:10:59.184450+0800 Test[7208:291713] var:1
```
block中能直接访问并修改静态变量，原因同样涉及到block是**如何捕获变量**

## block内捕获OC对象
```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    NSMutableArray *mArray = [[NSMutableArray alloc] init];
    void(^aBlock)(void) = ^(void) {
        NSObject *obj = [[NSObject alloc] init];
        [mArray addObject:obj];
        NSLog(@"array: %@",mArray);
    };
    aBlock();
}
```
这样是可行的，因为此处block捕获的是`结构体实例指针`，虽然对局部变量mArray赋值会产生编译错误，但使用捕获的值不会有任何问题。以下对局部变量mArray赋值则会产生错误

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    NSMutableArray *mArray = [[NSMutableArray alloc] init];
    void(^aBlock)(void) = ^(void) {
        mArray = [NSMutableArray array]; 	//error
    };
    aBlock();
}
```

## block捕获C数组
block内捕获C数组必须以指针形式捕获,**因为block没有实现对C数组的捕获**

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
//    char *word = "Hello";	//correct
    char word[] = "Hello";		//error
    void(^aBlock)(void) = ^(void) {
        printf("word : %s",word);
    };
    aBlock();
}
```

# block底层
## 代码转换
编译器(Xcode使用clang)在处理block语法时，会将其转换为普通的C语言代码来处理，而编译器本身支持通过命令选项将block语法转换为C/C++源代码表示，因此可从此转换入手分析。

```objc
int main(int argc, const char * argv[]) {
    void (^aBlock)(void) = ^ {
        NSLog(@"block body");
    };
    
    aBlock();
    
    return 0;
}
```

`clang -rewrite-objc main.m`

通过转换生成的代码很多，关注以下转换代码

```c
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_ny_1k2gy9p127s6z2jr11ffpx780000gn_T_main_2363f0_mi_0);
}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};

int main(int argc, const char * argv[]) {
    void (*aBlock)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));

    ((void (*)(__block_impl *))((__block_impl *)aBlock)->FuncPtr)((__block_impl *)aBlock);

    return 0;
}
```
## 分析
### 数据结构与实例化
block语法块转换如下:

```c
^ { NSLog(@"block body");};
```

```c
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_ny_1k2gy9p127s6z2jr11ffpx780000gn_T_main_2363f0_mi_0);
}
```
可以看出**block语法块内的函数体**实际上被作为**C函数**处理

* 函数名由编译器clang根据block语法块出现在的函数(此处为`main`)与出现顺序给出
* 函数参数`__cself`是`__main_block_impl_0 `类型的结构体指针，它相当于C++实例方法中指向实例自身变量`this`或OC实例方法中指向对象自身的变量`self`,`__cself`则指向block语法块结构体实例（指向block值的变量）

---
上述提到block语法块结构体实例，指的是一个block语法块底层是通过**结构体定义**以及**实例化**来处理

回顾main函数看block语法块

```objc
void (^aBlock)(void) = ^ {
    NSLog(@"block body");
};
```
对应转换代码:

```c
void (*aBlock)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));
```
不难看出，此处实例化了一个`__main_block_impl_0 `结构体并将地址(指针)赋值给aBlock变量（也就是上述提到的`__cself`指针，他们指向同一个结构体实例）

回顾关注的转换代码，可将三个结构体定义合并成一个，如下:

```c
struct __main_block_impl_0 {
    void *isa;
    int Flags;
    int Reserved;
    void *FuncPtr;
    size_t reserved;
    size_t Block_size;
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```
字段|含义
---|---
isa|isa指针，实现实例对象相关的功能，OC对象都有该指针
Flags|用于按 bit 位表示一些 block 的附加信息,block copy的时候会用到|
Reserved|保留字段，用于今后版本升级
FuncPtr|函数指针，指向具体的 block 实现的函数调用地址
Block_size|block实例的大小，也就是`__main_block_impl_0`结构体的大小

这是编译器生成的对block描述的数据结构体,更详细的定义可在Apple开源代码[Block_private.h](https://opensource.apple.com/source/libclosure/libclosure-65/Block_private.h)中找到

```c
/*
 * Block_private.h
 *
 * SPI for Blocks
 *
 * Copyright (c) 2008-2010 Apple Inc. All rights reserved.
 *
 * @APPLE_LLVM_LICENSE_HEADER@
 *
 */

#ifndef _BLOCK_PRIVATE_H_
#define _BLOCK_PRIVATE_H_

#include <Availability.h>
#include <AvailabilityMacros.h>
#include <TargetConditionals.h>

#include <stdbool.h>
#include <stdint.h>
#include <stdio.h>

#include <Block.h>

#if __cplusplus
extern "C" {
#endif


// Values for Block_layout->flags to describe block objects
enum {
    BLOCK_DEALLOCATING =      (0x0001),  // runtime
    BLOCK_REFCOUNT_MASK =     (0xfffe),  // runtime
    BLOCK_NEEDS_FREE =        (1 << 24), // runtime
    BLOCK_HAS_COPY_DISPOSE =  (1 << 25), // compiler
    BLOCK_HAS_CTOR =          (1 << 26), // compiler: helpers have C++ code
    BLOCK_IS_GC =             (1 << 27), // runtime
    BLOCK_IS_GLOBAL =         (1 << 28), // compiler
    BLOCK_USE_STRET =         (1 << 29), // compiler: undefined if !BLOCK_HAS_SIGNATURE
    BLOCK_HAS_SIGNATURE  =    (1 << 30), // compiler
    BLOCK_HAS_EXTENDED_LAYOUT=(1 << 31)  // compiler
};

#define BLOCK_DESCRIPTOR_1 1
struct Block_descriptor_1 {
    uintptr_t reserved;
    uintptr_t size;
};

#define BLOCK_DESCRIPTOR_2 1
struct Block_descriptor_2 {
    // requires BLOCK_HAS_COPY_DISPOSE
    void (*copy)(void *dst, const void *src);
    void (*dispose)(const void *);
};

#define BLOCK_DESCRIPTOR_3 1
struct Block_descriptor_3 {
    // requires BLOCK_HAS_SIGNATURE
    const char *signature;
    const char *layout;     // contents depend on BLOCK_HAS_EXTENDED_LAYOUT
};

struct Block_layout {
    void *isa;
    volatile int32_t flags; // contains ref count
    int32_t reserved; 
    void (*invoke)(void *, ...);
    struct Block_descriptor_1 *descriptor;
    // imported variables
};


// Values for Block_byref->flags to describe __block variables
enum {
    // Byref refcount must use the same bits as Block_layout's refcount.
    // BLOCK_DEALLOCATING =      (0x0001),  // runtime
    // BLOCK_REFCOUNT_MASK =     (0xfffe),  // runtime

    BLOCK_BYREF_LAYOUT_MASK =       (0xf << 28), // compiler
    BLOCK_BYREF_LAYOUT_EXTENDED =   (  1 << 28), // compiler
    BLOCK_BYREF_LAYOUT_NON_OBJECT = (  2 << 28), // compiler
    BLOCK_BYREF_LAYOUT_STRONG =     (  3 << 28), // compiler
    BLOCK_BYREF_LAYOUT_WEAK =       (  4 << 28), // compiler
    BLOCK_BYREF_LAYOUT_UNRETAINED = (  5 << 28), // compiler

    BLOCK_BYREF_IS_GC =             (  1 << 27), // runtime

    BLOCK_BYREF_HAS_COPY_DISPOSE =  (  1 << 25), // compiler
    BLOCK_BYREF_NEEDS_FREE =        (  1 << 24), // runtime
};

struct Block_byref {
    void *isa;
    struct Block_byref *forwarding;
    volatile int32_t flags; // contains ref count
    uint32_t size;
};

struct Block_byref_2 {
    // requires BLOCK_BYREF_HAS_COPY_DISPOSE
    void (*byref_keep)(struct Block_byref *dst, struct Block_byref *src);
    void (*byref_destroy)(struct Block_byref *);
};

struct Block_byref_3 {
    // requires BLOCK_BYREF_LAYOUT_EXTENDED
    const char *layout;
};


// Extended layout encoding.

// Values for Block_descriptor_3->layout with BLOCK_HAS_EXTENDED_LAYOUT
// and for Block_byref_3->layout with BLOCK_BYREF_LAYOUT_EXTENDED

// If the layout field is less than 0x1000, then it is a compact encoding 
// of the form 0xXYZ: X strong pointers, then Y byref pointers, 
// then Z weak pointers.

// If the layout field is 0x1000 or greater, it points to a 
// string of layout bytes. Each byte is of the form 0xPN.
// Operator P is from the list below. Value N is a parameter for the operator.
// Byte 0x00 terminates the layout; remaining block data is non-pointer bytes.

enum {
    BLOCK_LAYOUT_ESCAPE = 0, // N=0 halt, rest is non-pointer. N!=0 reserved.
    BLOCK_LAYOUT_NON_OBJECT_BYTES = 1,    // N bytes non-objects
    BLOCK_LAYOUT_NON_OBJECT_WORDS = 2,    // N words non-objects
    BLOCK_LAYOUT_STRONG           = 3,    // N words strong pointers
    BLOCK_LAYOUT_BYREF            = 4,    // N words byref pointers
    BLOCK_LAYOUT_WEAK             = 5,    // N words weak pointers
    BLOCK_LAYOUT_UNRETAINED       = 6,    // N words unretained pointers
    BLOCK_LAYOUT_UNKNOWN_WORDS_7  = 7,    // N words, reserved
    BLOCK_LAYOUT_UNKNOWN_WORDS_8  = 8,    // N words, reserved
    BLOCK_LAYOUT_UNKNOWN_WORDS_9  = 9,    // N words, reserved
    BLOCK_LAYOUT_UNKNOWN_WORDS_A  = 0xA,  // N words, reserved
    BLOCK_LAYOUT_UNUSED_B         = 0xB,  // unspecified, reserved
    BLOCK_LAYOUT_UNUSED_C         = 0xC,  // unspecified, reserved
    BLOCK_LAYOUT_UNUSED_D         = 0xD,  // unspecified, reserved
    BLOCK_LAYOUT_UNUSED_E         = 0xE,  // unspecified, reserved
    BLOCK_LAYOUT_UNUSED_F         = 0xF,  // unspecified, reserved
};


// Runtime support functions used by compiler when generating copy/dispose helpers

// Values for _Block_object_assign() and _Block_object_dispose() parameters
enum {
    // see function implementation for a more complete description of these fields and combinations
    BLOCK_FIELD_IS_OBJECT   =  3,  // id, NSObject, __attribute__((NSObject)), block, ...
    BLOCK_FIELD_IS_BLOCK    =  7,  // a block variable
    BLOCK_FIELD_IS_BYREF    =  8,  // the on stack structure holding the __block variable
    BLOCK_FIELD_IS_WEAK     = 16,  // declared __weak, only used in byref copy helpers
    BLOCK_BYREF_CALLER      = 128, // called from __block (byref) copy/dispose support routines.
};

enum {
    BLOCK_ALL_COPY_DISPOSE_FLAGS = 
        BLOCK_FIELD_IS_OBJECT | BLOCK_FIELD_IS_BLOCK | BLOCK_FIELD_IS_BYREF |
        BLOCK_FIELD_IS_WEAK | BLOCK_BYREF_CALLER
};

// Runtime entry point called by compiler when assigning objects inside copy helper routines
BLOCK_EXPORT void _Block_object_assign(void *destAddr, const void *object, const int flags);
    // BLOCK_FIELD_IS_BYREF is only used from within block copy helpers


// runtime entry point called by the compiler when disposing of objects inside dispose helper routine
BLOCK_EXPORT void _Block_object_dispose(const void *object, const int flags);


// Other support functions

// runtime entry to get total size of a closure
BLOCK_EXPORT size_t Block_size(void *aBlock);

// indicates whether block was compiled with compiler that sets the ABI related metadata bits
BLOCK_EXPORT bool _Block_has_signature(void *aBlock)
    __OSX_AVAILABLE_STARTING(__MAC_10_7, __IPHONE_4_3);

// returns TRUE if return value of block is on the stack, FALSE otherwise
BLOCK_EXPORT bool _Block_use_stret(void *aBlock)
    __OSX_AVAILABLE_STARTING(__MAC_10_7, __IPHONE_4_3);

// Returns a string describing the block's parameter and return types.
// The encoding scheme is the same as Objective-C @encode.
// Returns NULL for blocks compiled with some compilers.
BLOCK_EXPORT const char * _Block_signature(void *aBlock)
    __OSX_AVAILABLE_STARTING(__MAC_10_7, __IPHONE_4_3);

// Returns a string describing the block's GC layout.
// This uses the GC skip/scan encoding.
// May return NULL.
BLOCK_EXPORT const char * _Block_layout(void *aBlock)
    __OSX_AVAILABLE_STARTING(__MAC_10_7, __IPHONE_4_3);

// Returns a string describing the block's layout.
// This uses the "extended layout" form described above.
// May return NULL.
BLOCK_EXPORT const char * _Block_extended_layout(void *aBlock)
    __OSX_AVAILABLE_STARTING(__MAC_10_8, __IPHONE_7_0);

// Callable only from the ARR weak subsystem while in exclusion zone
BLOCK_EXPORT bool _Block_tryRetain(const void *aBlock)
    __OSX_AVAILABLE_STARTING(__MAC_10_7, __IPHONE_4_3);

// Callable only from the ARR weak subsystem while in exclusion zone
BLOCK_EXPORT bool _Block_isDeallocating(const void *aBlock)
    __OSX_AVAILABLE_STARTING(__MAC_10_7, __IPHONE_4_3);


// the raw data space for runtime classes for blocks
// class+meta used for stack, malloc, and collectable based blocks
BLOCK_EXPORT void * _NSConcreteMallocBlock[32]
    __OSX_AVAILABLE_STARTING(__MAC_10_6, __IPHONE_3_2);
BLOCK_EXPORT void * _NSConcreteAutoBlock[32]
    __OSX_AVAILABLE_STARTING(__MAC_10_6, __IPHONE_3_2);
BLOCK_EXPORT void * _NSConcreteFinalizingBlock[32]
    __OSX_AVAILABLE_STARTING(__MAC_10_6, __IPHONE_3_2);
BLOCK_EXPORT void * _NSConcreteWeakBlockVariable[32]
    __OSX_AVAILABLE_STARTING(__MAC_10_6, __IPHONE_3_2);
// declared in Block.h
// BLOCK_EXPORT void * _NSConcreteGlobalBlock[32];
// BLOCK_EXPORT void * _NSConcreteStackBlock[32];


// the intercept routines that must be used under GC
BLOCK_EXPORT void _Block_use_GC( void *(*alloc)(const unsigned long, const bool isOne, const bool isObject),
                                  void (*setHasRefcount)(const void *, const bool),
                                  void (*gc_assign_strong)(void *, void **),
                                  void (*gc_assign_weak)(const void *, void *),
                                  void (*gc_memmove)(void *, void *, unsigned long));

// earlier version, now simply transitional
BLOCK_EXPORT void _Block_use_GC5( void *(*alloc)(const unsigned long, const bool isOne, const bool isObject),
                                  void (*setHasRefcount)(const void *, const bool),
                                  void (*gc_assign_strong)(void *, void **),
                                  void (*gc_assign_weak)(const void *, void *));

BLOCK_EXPORT void _Block_use_RR( void (*retain)(const void *),
                                 void (*release)(const void *));

struct Block_callbacks_RR {
    size_t  size;                   // size == sizeof(struct Block_callbacks_RR)
    void  (*retain)(const void *);
    void  (*release)(const void *);
    void  (*destructInstance)(const void *);
};
typedef struct Block_callbacks_RR Block_callbacks_RR;

BLOCK_EXPORT void _Block_use_RR2(const Block_callbacks_RR *callbacks);

// make a collectable GC heap based Block.  Not useful under non-GC.
BLOCK_EXPORT void *_Block_copy_collectable(const void *aBlock);

// thread-unsafe diagnostic
BLOCK_EXPORT const char *_Block_dump(const void *block);


// Obsolete

// first layout
struct Block_basic {
    void *isa;
    int Block_flags;  // int32_t
    int Block_size; // XXX should be packed into Block_flags
    void (*Block_invoke)(void *);
    void (*Block_copy)(void *dst, void *src);  // iff BLOCK_HAS_COPY_DISPOSE
    void (*Block_dispose)(void *);             // iff BLOCK_HAS_COPY_DISPOSE
    //long params[0];  // where const imports, __block storage references, etc. get laid down
} __attribute__((deprecated));


#if __cplusplus
}
#endif


#endif
```

对block的结构有了一定了解，再看`__main_block_impl_0 `构造函数

```c
__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
```

参数`fp`对应block内部函数体实现的首地址（也就是block实现的入口地址）

参数`desc`为block描述结构体指针

参数`flags`可选，默认值为0

内部语句表明将block初始化为`_NSConcreteStackBlock`类型，并绑定block实现对应的C函数指针

以下实例化调用，参数1为block对应的C函数指针，参数2为以静态全局变量初始化的`__mian_block_desc_0`结构体指针

```c
((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));
```

### block调用原理

```c
aBlock();
```

```c
((void (*)(__block_impl *))((__block_impl *)aBlock)->FuncPtr)((__block_impl *)aBlock);

//去掉类型转换前缀
aBlock->FuncPtr(aBlock);
```

不难看出，block调用即是通过结构体实例指针aBlock访问结构体成员函数指针`FuncPtr`来调用，同时将自身(`__cself `指针)传入

### 捕获变量
#### 捕获局部变量
```c
int main(int argc, const char * argv[]) {
    int var = 100;
    void (^aBlock)(void) = ^ {
        printf("var : %d\n",var);
    };
    var = 0;
    aBlock();
    return 0;
}
```
同样通过clang转换得到以下关键部分:

```c
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int var;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _var, int flags=0) : var(_var) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

void (*aBlock)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, var));
```
从转换代码看出，block的结构体增加了一成员var，并且关键部分在于block结构体实例化时传入var是**值传递**。

总的来说，所谓"捕获局部变量值"，意思是解析block语法块的时候，block语法块内使用的局部变量值被保存到block的结构体实例中

#### 处理非局部变量
在2.9与2.10节中提到block访问与修改全局静态变量、全局变量、静态变量，以下同样通过转换代码查看:

```c
static int static_global_val = 100;
int global_val = 101;

int main(int argc, const char * argv[]) {
    static int static_val = 102;
    void (^aBlock)(void) = ^ {
        printf("static_global_val : %d\n",static_global_val);
        printf("global_val : %d\n",global_val);
        printf("static_val : %d\n",static_val);
    };
    static_global_val = 0;
    global_val = 1;
    static_val = 2;
    aBlock();
    return 0;
}
```


```c
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int *static_val;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int *_static_val, int flags=0) : static_val(_static_val) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

void (*aBlock)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, &static_val));

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    int *static_val = __cself->static_val; // bound by copy
    printf("static_global_val : %d\n",static_global_val);
    printf("global_val : %d\n",global_val);
    printf("static_val : %d\n",(*static_val));
}
```

block底层对静态全局变量、全局变量并没有将其添加到自身结构体定义中，这一点很容易理解，就像往常编写C程序，这两类变量在整个程序生命周期中都可以访问并赋值修改。**静态变量在超出函数作用域后仍然可以使用，因此block捕获静态变量static_val的指针，并在需要访问或修改的时候通过`int *static_val = __cself->static_val`指针来操作。其实这在另一方面也解释了为什么局部变量不能这样做？原因在于局部变量超出作用域后就释放，block不能捕获其指针来进行访问或修改。**

要想在block内部访问或修改原来的局部变量，就需要使用`__block`

####  <span id="jump">__block修饰符</span>

__block类似于static、auto修饰符，可以用于给变量设置其存储的区域。例如，auto表示将变量存储在栈中，static表示将变量存储在数据区。

同样地，将用__block修饰的变量实例代码转换，得到如下:

```c
int main(int argc, const char * argv[]) {
    __block int __block_val = 100;
    void (^aBlock)(void) = ^ {
        __block_val = 1;
        printf("__block_val : %d\n",__block_val);
    };
    aBlock();
    printf("__block_val : %d\n",__block_val);
    return 0;
}
```

```c
struct __Block_byref___block_val_0 {
  void *__isa;
__Block_byref___block_val_0 *__forwarding;	//指向自身的指针
 int __flags;
 int __size;
 int __block_val;
};

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __Block_byref___block_val_0 *__block_val; // by ref
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref___block_val_0 *___block_val, int flags=0) : __block_val(___block_val->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  __Block_byref___block_val_0 *__block_val = __cself->__block_val; // bound by ref

        (__block_val->__forwarding->__block_val) = 1;
        printf("__block_val : %d\n",(__block_val->__forwarding->__block_val));
    }
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->__block_val, (void*)src->__block_val, 8/*BLOCK_FIELD_IS_BYREF*/);}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->__block_val, 8/*BLOCK_FIELD_IS_BYREF*/);}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};
int main(int argc, const char * argv[]) {
	//初始化__block变量对应的结构体
    __attribute__((__blocks__(byref))) __Block_byref___block_val_0 __block_val = {(void*)0,(__Block_byref___block_val_0 *)&__block_val, 0, sizeof(__Block_byref___block_val_0), 100};
    //初始化block结构体,同时将__block变量对应的结构体指针传入保存
    void (*aBlock)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, (__Block_byref___block_val_0 *)&__block_val, 570425344));
    ((void (*)(__block_impl *))((__block_impl *)aBlock)->FuncPtr)((__block_impl *)aBlock);
    printf("__block_val : %d\n",(__block_val.__forwarding->__block_val));
    return 0;
}
```
可以看出

```c
__block int __block_val = 100;
```
转换后，变成了初始化一个`__Block_byref___block_val_0 `结构体变量`__block_val`，并将值100传入。在block结构体初始化时将`__block_val`指针传入保存，查看此处block结构体定义，新增了一个`__Block_byref___block_val_0`类型的结构体指针,再查看`__main_block_func_0`函数，使用了该指针来访问`__block_val`这个用__block修饰的变量。

**此处`__forwarding`指针十分巧妙，它的作用是使__block变量无论是在栈上或是在堆上都能够被block正确访问，原理在后续章节**

**在block结构体定义中，它并没有像捕获局部变量那样，将`__Block_byref___block_val_0`以结构体的形式定义，而是通过指针方式,这样做是为了让多个block可以访问同一__block变量而无需逐一实例化多个__block变量的结构体**


### 存储域

上述示例的block与__block变量都在main函数内声明定义，它们的本质是:

名称|实质|
---|---|
block|栈上的block结构体实例
__block变量|栈上__block变量结构体实例|

#### block的存储域
从block结构体中的isa指针值可以发现`_NSConcreteStackBlock`，此类block在栈上实例化。按照block的存储域划分，有以下三种:

* _NSConcreteStackBlock
* _NSConcreteGlobalBlock
* _NSConcreteMallocBlock

程序运行时它们的内存存储情况如下

<div align=center> 	<img src="https://kinkenyuen.oss-cn-shenzhen.aliyuncs.com/images/block存储域.png" width=50% /> 	<br> 	<div style="color:orange; border-bottom: 1px solid #d9d9d9;     	display: inline-block;     	color: #999;     	padding: 2px;">block存储域</div> </div>

##### _NSConcreteGlobalBlock
因为使用全局变量的地方不能使用局部变量，因此此类block不存在对局部变量的捕获，在程序中只需要一个实例，将全局block存放在数据区域显然是没有问题。
以下两种情况存在的block均为全局block

* block定义在全局区（类似于全局变量）
* block定义的语法块中没有使用或捕获局部变量

##### _NSConcreteMallocBlock

栈block在超出变量作用域后，block结构体实例就被释放(__block变量同理)。而`_NSConcreteMallocBlock`即将block结构体实例和__block变量结构体实例从栈上复制到堆上，来使它们在超出作用域后，仍可以继续存在。

在ARC环境下，大多数情况下编译器会自动帮我们生成复制block到堆上的代码。至于底层是如何copy，这里不展开分析。

#### \_\_block变量的存储域

在block捕获\_\_block变量的情况下，__block变量会受block从栈复制到堆上影响，如下

__block的存储域 | block从栈复制到堆上后的影响|
---|---
栈 | 从栈复制到堆并被block持有|
堆| 被block持有|

<div align=center> 	<img src="https://kinkenyuen.oss-cn-shenzhen.aliyuncs.com/images/block拷贝时__block变量行为.jpg" width=50% /> 	<br> 	<div style="color:orange; border-bottom: 1px solid #d9d9d9;     	display: inline-block;     	color: #999;     	padding: 2px;">block拷贝时__block变量行为</div> </div>

大多数情况下，block最先都在栈上实例化，\_\_block变量也是在栈上实例化，当一个__block变量被多个block使用时，第一个被复制到堆上的block持有\_\_block实例，之后剩下的block复制到堆上时，对\_\_block实例增加引用计数。

block与__block变量在堆上的内存管理与OC对象的引用计数方式相同。

### \_\_forwarding的作用 
在3.2.3.3 \_\_block修饰符一节中出现的\_\_forwarding指针，结合上面\_\_block变量实例的内存行为分析就很好理解了。转换得到以下代码:

```c
int main(int argc, const char * argv[]) {
    __block int __block_val = 1;
    void (^aBlock)(void) = ^ {
        __block_val++;
    };
    //执行到此处，由于ARC环境，block从栈复制到堆上,__block变量也复制到堆上
    //__block变量在复制到堆上后，更新__forwarding值，使其指向堆上的__block变量实例
    __block_val++;//可以从下面的转换代码中看出，这里的自增实质是对堆上的__block变量结构体成员__block_val自增
    aBlock();
    printf("__block_val : %d\n",__block_val);//这里看似访问的是栈上的__block变量，实质也是访问堆上的
    return 0;
}
```

```c
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    __Block_byref___block_val_0 *__block_val = __cself->__block_val; // bound by ref
    (__block_val->__forwarding->__block_val)++;
}

int main(int argc, const char * argv[]) {
    __attribute__((__blocks__(byref))) __Block_byref___block_val_0 __block_val = {(void*)0,(__Block_byref___block_val_0 *)&__block_val, 0, sizeof(__Block_byref___block_val_0), 1};
    void (*aBlock)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, (__Block_byref___block_val_0 *)&__block_val, 570425344));
    (__block_val.__forwarding->__block_val)++;
    ((void (*)(__block_impl *))((__block_impl *)aBlock)->FuncPtr)((__block_impl *)aBlock);
    printf("__block_val : %d\n",(__block_val.__forwarding->__block_val));
    return 0;
}
```

通过`__forwarding`的处理，就可以确保block语法块内或外，访问的的__block变量均为同一个，大多数情况下是在堆中的实例。

### 捕获对象

```objc
{
	id array = [[NSMutableArray alloc] init];
}
```
默认情况下,array变量在超出作用域后就被释放，在堆上的NSMutableArray对象由于没有强引用也会被释放。
但是查看以下代码:

```objc
int main(int argc, const char * argv[]) {
    void (^blk)(id);
    {
        id array = [[NSMutableArray alloc] init];
        blk = ^ (id obj) {
            [array addObject:obj];
            NSLog(@"array count : %ld",[array count]);
        };
    }

    blk([[NSObject alloc] init]);
    blk([[NSObject alloc] init]);
    blk([[NSObject alloc] init]);
    
    return 0;
}
```
代码运行正常，也就是array在超出其变量作用域后，仍能在block语法块内存在被block访问赋值,这意味着block引用了该array对象

转换代码

```c
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  id array;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, id _array, int flags=0) : array(_array) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself, id obj) {
  id array = __cself->array; // bound by copy

            ((void (*)(id, SEL, ObjectType _Nonnull))(void *)objc_msgSend)((id)array, sel_registerName("addObject:"), (id)obj);
            NSLog((NSString *)&__NSConstantStringImpl__var_folders_fv_st7q8xkx05g6szwd247tjc8c0000gn_T_main_d9cea7_mi_0,((NSUInteger (*)(id, SEL))(void *)objc_msgSend)((id)array, sel_registerName("count")));
}

static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->array, (void*)src->array, 3/*BLOCK_FIELD_IS_OBJECT*/);}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->array, 3/*BLOCK_FIELD_IS_OBJECT*/);}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};

int main(int argc, const char * argv[]) {
    void (*blk)(id);
    {
        id array = ((NSMutableArray *(*)(id, SEL))(void *)objc_msgSend)((id)((NSMutableArray *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSMutableArray"), sel_registerName("alloc")), sel_registerName("init"));
        blk = ((void (*)(id))&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, array, 570425344));
    }

    ((void (*)(__block_impl *, id))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk, ((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("alloc")), sel_registerName("init")));
    ((void (*)(__block_impl *, id))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk, ((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("alloc")), sel_registerName("init")));
    ((void (*)(__block_impl *, id))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk, ((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("alloc")), sel_registerName("init")));

    return 0;
}
```
从以上发现，block捕获的array成为了blcok结构体中的成员变量,实际上结构体中的array变量含有__strong修饰符，编译器在ARC环境下隐藏了。在OC中，C结构体不能含有__stroing修饰的变量，因为编译器不知道何时对C结构体进行初始化和释放，也就是说C结构体不归ARC管理。

但是OC的运行时库可以在恰当时机将block从栈复制到堆上以及释放堆上的block(block在ARC环境下编译器做的处理)，由于这一原因，block的结构体中可以有__strong或__weak修饰的变量，编译器也能很好地对这些变量进行内存管理。

为了管理block结构体中的这些变量，使用了`__main_block_desc_0 `结构体中新增的两个成员变量，分别为`copy`与`dispose`函数指针

`copy`指针指向`__main_block_copy_0`函数，而`__main_block_copy_0`函数使用`_Block_object_assign`函数**将array赋值到block结构体成员变量array中并持有该对象**
`_Block_object_assign`相当于`retain`并赋值。这一步的表现即为block引用持有了对象array

`dispose`指针指向`__main_block_dispose_0 `函数，而`__main_block_dispose_0 `函数使用`_Block_object_dispose `函数**释放赋值在block结构体成员变量array中的对象**
`_Block_object_dispose `相当于`release`实例方法

从转换代码中没有发现以上两个函数的调用处，原因是它们是**在block从栈复制到堆以及堆上block释放时才会调用**。

block从栈上复制到堆的时机

* 调用block的copy实例方法
* block作为函数返回值返回
* 将block复制给带__strong修饰符id类型的类或block类型成员变量时
* 在方法名中含有usingBlock的Cocoa框架方法或GCD的API中传递block时

在以上情况下的block都会在底层调用`_Block_copy`函数从栈上复制到堆

回顾`__block修饰符`一节，其实转换代码也使用了`copy`和`dispose`函数,不同的是其中一个枚举参数


对象| BLOCK_FIELD_IS_OBJECT|
---|---|
__block变量| BLOCK_FIELD_IS_BYREF|

更多的枚举类型可以在前面的`Block_private.h`中内容找到

# 参考
《Objective-C高级编程》

















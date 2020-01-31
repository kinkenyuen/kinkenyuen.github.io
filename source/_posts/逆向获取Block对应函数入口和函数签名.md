---
title: 逆向获取Block对应函数入口和函数签名
date: 2020-01-14 19:37:46
tags: 逆向 
categories: 逆向
---

# 逆向获取block对应函数入口和函数签名

在逆向分析App的时候，有时会遇到某个关键方法中传入一个block参数来做回调，class-dump无法解析出block的类型以及函数签名。了解过block本质及其内存模型后，就可以通过lldb动态调试来获取目标信息

# block的内存结构

在LLVM文档中，找到block的实现规范[Block Implementation Specification](http://clang.llvm.org/docs/Block-ABI-Apple.html),找到block内存结构的定义

```c
struct Block_literal_1 {
    void *isa; // initialized to &_NSConcreteStackBlock or &_NSConcreteGlobalBlock
    int flags;
    int reserved;
    void (*invoke)(void *, ...);
    struct Block_descriptor_1 {
    unsigned long int reserved;         // NULL
        unsigned long int size;         // sizeof(struct Block_literal_1)
        // optional helper functions
        void (*copy_helper)(void *dst, void *src);     // IFF (1<<25)
        void (*dispose_helper)(void *src);             // IFF (1<<25)
        // required ABI.2010.3.16
        const char *signature;                         // IFF (1<<30)
    } *descriptor;
    // imported variables
};
```

其中block对应的实现函数地址入口和函数签名分别在`void (*invoke)(void *, ...);`和`descriptor `中的` const char *signature; `中保存

# 实例

某app中的某个带block参数的方法

`- (void)parseRequest:(id)arg1 result:(id)arg2 completion:(id)arg3;`

祭出debugserver和lldb

<br>

<div align=center> 	<img src="https://kinkenyuen.oss-cn-shenzhen.aliyuncs.com/images/debugserver启动.png" width="800" /> 	<div style="color:orange; border-bottom: 1px solid #d9d9d9;     	display: inline-block;     	color: #999;     	padding: 2px;">debugserver启动</div> </div>

<br>

<div align=center> 	<img src="https://kinkenyuen.oss-cn-shenzhen.aliyuncs.com/images/LLDB connect.png" width="800" /> 	<div style="color:orange; border-bottom: 1px solid #d9d9d9;     	display: inline-block;     	color: #999;     	padding: 2px;">LLDB connect</div> </div>

<br>

<div align=center> 	<img src="https://kinkenyuen.oss-cn-shenzhen.aliyuncs.com/images/定位目标方法内存地址.png" width="800" /> 	<div style="color:orange; border-bottom: 1px solid #d9d9d9;     	display: inline-block;     	color: #999;     	padding: 2px;">定位目标方法内存地址</div> </div>

<br>

<div align=center> 	<img src="https://kinkenyuen.oss-cn-shenzhen.aliyuncs.com/images/断点并触发.png" width="800" /> 	<div style="color:orange; border-bottom: 1px solid #d9d9d9;     	display: inline-block;     	color: #999;     	padding: 2px;">断点并触发</div> </div>

<br>

<div align=center> 	<img src="https://kinkenyuen.oss-cn-shenzhen.aliyuncs.com/images/获取block参数对象.png" width="800" /> 	<div style="color:orange; border-bottom: 1px solid #d9d9d9;     	display: inline-block;     	color: #999;     	padding: 2px;">获取block参数对象</div> </div>

<br>

<div align=center> 	<img src="https://kinkenyuen.oss-cn-shenzhen.aliyuncs.com/images/block内存实例.png" width="800" /> 	<div style="color:orange; border-bottom: 1px solid #d9d9d9;     	display: inline-block;     	color: #999;     	padding: 2px;">block内存实例</div> </div>

<br>

<div align=center> 	<img src="https://kinkenyuen.oss-cn-shenzhen.aliyuncs.com/images/检查block是否有函数签名.png" width="800" /> 	<div style="color:orange; border-bottom: 1px solid #d9d9d9;     	display: inline-block;     	color: #999;     	padding: 2px;">检查block是否有函数签名</div> </div>

<br>

<div align=center> 	<img src="https://kinkenyuen.oss-cn-shenzhen.aliyuncs.com/images/检查block是否有copy和dispose函数指针.png" width="800" /><div style="color:orange; border-bottom: 1px solid #d9d9d9;     	display: inline-block;     	color: #999;     	padding: 2px;">检查block是否有copy和dispose函数指针</div> </div>

<br>

<div align=center> 	<img src="https://kinkenyuen.oss-cn-shenzhen.aliyuncs.com/images/descriptor布局.png" width="800" /> 	<div style="color:orange; border-bottom: 1px solid #d9d9d9;     	display: inline-block;     	color: #999;     	padding: 2px;">descriptor布局</div> </div>

<br>

<div align=center> 	<img src="https://kinkenyuen.oss-cn-shenzhen.aliyuncs.com/images/函数签名信息.png" width="800" /> 	<div style="color:orange; border-bottom: 1px solid #d9d9d9;     	display: inline-block;     	color: #999;     	padding: 2px;">函数签名信息</div> </div>

<br>

最后得到的函数签名字符编码，可在官方[Type Encoding](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html)中找到

从图中可以看出，block参数个数为3，block 的返回值为`void`,第一个参数为block自身，通常在block语法参数列表省略，第二个参数为`NSArray`类型，第三个参数为`BOOL`类型

# 参考

[通过逆向深入理解 Block 的内存模型](http://www.swiftyper.com/2016/12/16/debuging-objective-c-blocks-in-lldb/)

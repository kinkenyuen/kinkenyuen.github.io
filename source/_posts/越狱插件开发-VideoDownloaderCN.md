---
title: 越狱插件开发-VideoDownloaderCN
date: 2019-12-31 16:51:17
tags: 逆向
categories: 逆向
---

# 前言

原创文章搬运


学习逆向有一段时间了，想着写个iOS Jailbreak Tweak练练手，平时比较喜欢看视频，看到比较搞笑的视频想保存下载发给好友，无奈无法下载，于是有了这个**VideoDownloaderCN**插件。

**声明 ： 插件仅用于技术研究**

<div align=center>
	<img src="https://kinkenyuen.oss-cn-shenzhen.aliyuncs.com/images/tweak.png" width=50% />
	<br>
	<div style="color:orange; border-bottom: 1px solid #d9d9d9;
    	display: inline-block;
    	color: #999;
    	padding: 2px;">插件界面
	</div>
</div>

# 分析
最近分析几个App上的视频播放，基本上就是一个View上有个播放组件，而这个View或者它的若干个nextResponder持有这个视频的url，于是有这样的思路：

* 动态分析定位得到视频的URL
* 在View的构造方法内添加一个手势弹出下载
* 手势对应视频下载的方法实现，最后移动到系统相册

分析方法用到:**Cycript动态调试**、**Reveal界面分析**、**class-dump头文件**

分析过程不在这里描述，主要mark一下Tweak的构建工程.

# Theos

## 安装配置
网络上有非常多的教程，但是我推荐查看官方的.[Theos installation](https://github.com/theos/theos/wiki/Installation-iOS)

## 创建工程
> `nic.pl`

<div align=center> 	<img src="https://kinkenyuen.oss-cn-shenzhen.aliyuncs.com/images/creat_tweak.png" width=600 /> 	<br> 	<div style="color:orange; border-bottom: 1px solid #d9d9d9;     	display: inline-block;     	color: #999;     	padding: 2px;">创建插件</div> </div>

> 目录介绍

<div align=center> 	<img src="https://kinkenyuen.oss-cn-shenzhen.aliyuncs.com/images/mulu.png" width=600 /> 	<br> 	<div style="color:orange; border-bottom: 1px solid #d9d9d9;     	display: inline-block;     	color: #999;     	padding: 2px;">目录介绍</div> </div>

> Makefile

<div align=center> 	<img src="https://kinkenyuen.oss-cn-shenzhen.aliyuncs.com/images/makefile.png" width=600 /> 	<br> 	<div style="color:orange; border-bottom: 1px solid #d9d9d9;     	display: inline-block;     	color: #999;     	padding: 2px;">makefile</div> </div>

> Tweak.xm

<div align=center> 	<img src="https://kinkenyuen.oss-cn-shenzhen.aliyuncs.com/images/tweakxm.png" width=600 /> 	<br> 	<div style="color:orange; border-bottom: 1px solid #d9d9d9;     	display: inline-block;     	color: #999;     	padding: 2px;">xm</div> </div>

此处我使用了多个xm文件来区分每个App的注入代码，具体查看[github](https://github.com/kinkenyuen/VideoDownloaderCN)

> *.plist

<div align=center> 	<img src="https://kinkenyuen.oss-cn-shenzhen.aliyuncs.com/images/plist.png" width="375" /> 	<br> 	<div style="color:orange; border-bottom: 1px solid #d9d9d9;     	display: inline-block;     	color: #999;     	padding: 2px;">plist</div> </div>

## 编译Tweak
做好了基础配置以及编写好xm之后，就可以通过`make`命令编译，我遇到了一个错误，如下：

<div align=center> 	<img src="https://kinkenyuen.oss-cn-shenzhen.aliyuncs.com/images/makeerror.png" width=600 /> 	<br> 	<div style="color:orange; border-bottom: 1px solid #d9d9d9;     	display: inline-block;     	color: #999;     	padding: 2px;">编译错误</div> </div>

修改`Makefile`再次编译

<div align=center> 	<img src="https://kinkenyuen.oss-cn-shenzhen.aliyuncs.com/images/xiugaimakefile.png" width=600 /> 	<br> 	<div style="color:orange; border-bottom: 1px solid #d9d9d9;     	display: inline-block;     	color: #999;     	padding: 2px;">修改编译设置</div> </div>

<div align=center> 	<img src="https://kinkenyuen.oss-cn-shenzhen.aliyuncs.com/images/makesuccess.png" width=600 /> 	<br> 	<div style="color:orange; border-bottom: 1px solid #d9d9d9;     	display: inline-block;     	color: #999;     	padding: 2px;">再次编译</div> </div>

`make package`生成deb

<div align=center> 	<img src="https://kinkenyuen.oss-cn-shenzhen.aliyuncs.com/images/makepackage.png" width=600 /> 	<br> 	<div style="color:orange; border-bottom: 1px solid #d9d9d9;     	display: inline-block;     	color: #999;     	padding: 2px;">makepackage</div> </div>

## 安装Tweak

`make install`命令安装到设备,安装前需要配置一些必要参数,将以下两行参数配置到`Makefile`是一种方法，意思是通过本地USB方式，端口2222安装到手机，我的做法是配置好写在`.bash_profile`或`.zshrc`内，这样不用每次在`Makefile`内编写（重要提示：出现Error请详细检查theos配置，手机IP、端口是否映射，ssh是否正常登陆）

>`export THEOS_DEVICE_IP=localhost`
>
>`export THEOS_DEVICE_PORT=2222`

当然安装deb方法不止一种，当你打包出`deb`后

1. 可以使用`CyDown`这个插件开一个`ftp`，然后PC传`deb`过去，接着手机端打开`cydia`找到那个deb安装。
2. 可以使用`scp`命令传到手机端，接着手机终端`dpkg -i`安装,具体做法请自行搜索详细教程。

至此，一个Tweak在iOS越狱设备上

<div align=center> 	<img src="https://kinkenyuen.oss-cn-shenzhen.aliyuncs.com/images/install.jpeg" width=600 /> 	<br> 	<div style="color:orange; border-bottom: 1px solid #d9d9d9;     	display: inline-block;     	color: #999;     	padding: 2px;">安装</div> </div>

# 插件设置项Preference Bundle
一个`tweak`可能要设置一些选项，就像`App Store上`的`App`一样，在设置应用里面可以设置，在`theos`里，可以通过创建`Preference Bundle`来为插件提供设置界面,有点类似于`Xcode`里的`Setting Bundle`,`Preference Bundle`安装到手机后会在`/Library/PreferenceBundles/`目录生成一个对应的`bundle`,此`bundle`会基于`PreferenceLoader`注入到设置应用(`Setting.app`)，而`PreferenceLoader`是由Dustin Howett开发的基于`Mobile Substrate`的工具，主要为插件在系统设置界面添加一个设置入口。

## 为插件创建Preference Bundle
一般的做法为：在原插件目录使用`theos`创建

<div align=center> 	<img src="https://kinkenyuen.oss-cn-shenzhen.aliyuncs.com/images/createprefs.png" width=600 /> 	<br> 	<div style="color:orange; border-bottom: 1px solid #d9d9d9;     	display: inline-block;     	color: #999;     	padding: 2px;">创建PreferenceBundle</div> </div>

<div align=center> 	<img src="https://kinkenyuen.oss-cn-shenzhen.aliyuncs.com/images/prefssubproject.png" width=600 /> 	<br> 	<div style="color:orange; border-bottom: 1px solid #d9d9d9;     	display: inline-block;     	color: #999;     	padding: 2px;">PreferenceBundle sub project</div> </div>

创建`preference bundle`后新生成目录下的文件介绍如下:

文件        | 作用/含义
---         |---
entry.plist |为插件在系统设置应用界面添加一个入口，一般修改`icon`与`label`即可
XXXRootListController|XXXRootListController必须继承PSListController或者PSViewController，且必须实现- (id)specifiers方法，因为PSListController依赖_specifiers来获得metadata和group
Makefile    |`preference bundle`的`Makefile`，一般不用过多修改与操作，编译Tweak的Makefile会跟随着一起编译|
Resources文件夹下的文件如下 | |
Info.plist  |主要记录这个`preference bundle`的配置信息，一般不用修改
Root.plist  |重点编写的文件，主要配置插件界面的UI元素，XML格式，好像还有一种类似JSON格式的|

关于`Preference Bundle`的更多配置方法，参考：

[PreferenceBundles](http://iphonedevwiki.net/index.php/PreferenceBundles)

[Preferences specifier plist](https://iphonedevwiki.net/index.php/Preferences_specifier_plist#PSSpecifier_runtime_properties_of_plist_keys)

[让 iPhone 上显示学期周数（五） — — 增加用户配置界面](https://medium.com/@wangjinli/%E8%AE%A9-iphone-%E4%B8%8A%E6%98%BE%E7%A4%BA%E5%AD%A6%E6%9C%9F%E5%91%A8%E6%95%B0-%E4%BA%94-%E5%A2%9E%E5%8A%A0%E7%94%A8%E6%88%B7%E9%85%8D%E7%BD%AE%E7%95%8C%E9%9D%A2-32892a6eb97)

[制作 Preference Bundle 插件菜单](https://weibo.com/p/1001603804327025185325?from=page_100505_profile&wvr=6&mod=wenzhangmod)

[Part 6: Preferences, Preferences, a little Tweak, and Heaps of More Preferences](https://github.com/derv82/Exchangent/wiki/Part-6:-Preferences,-Preferences,-a-little-Tweak,-and-Heaps-of-More-Preferences)
 
[Theos - Preference Bundle Tutorial iOS 8 - iPhone, iPad](https://www.youtube.com/watch?v=nBQuz7TvecA)

网上关于`Preference Bundle`的用法中文介绍很少，我是结合`iPhonewiki`、视频以及一些开源插件的`Preference Bundle`配置学习，真的是花了不少时间~~~,具体的配置我不放上来了，可以去我的[github](https://github.com/kinkenyuen/VideoDownloaderCN)看一下，我尽量注释说明了配置的含义。

推荐一个开源插件Repo：[Open-Source-Tweaks](https://github.com/LacertosusRepo/Open-Source-Tweaks)

# 遇到的坑
`theos`创建`preference bundle`后编译不通过,原因是`theos`找不到对应的库

<div align=center> 	<img src="https://kinkenyuen.oss-cn-shenzhen.aliyuncs.com/images/theoskeng.png" width=600 /> 	<br> 	<div style="color:orange; border-bottom: 1px solid #d9d9d9;     	display: inline-block;     	color: #999;     	padding: 2px;">找不到私有库</div> </div>

解决方法:
手动下载`theos`需要的`sdks`，网上已经有人`Patch`好了，将从[theos sdks](https://github.com/theos/sdks)下载的`sdks`放到`theos`的`/sdks`目录下。

再次编译还是错误,原因是目前Xcode10.1已经使用iOS12.1的sdks，搜了一番没有找到`theos`用的，于是我将插件支持版本调低一点,可以编译通过,`Tweak`的`Makefile`添加:

>`export TARGET = iphone:clang:11.2:8.0`

最低版本8.0，最高版本11.2，反正iOS12的越狱大神还没`release`,**`why so serious?`**

参考:[Xcode10.x theos doesn't work](https://www.reddit.com/r/jailbreakdevelopers/comments/9k85yj/)

# 总结
本次主要学习`theos`开发`iPhone tweak`的操作以及为`tweak`增加设置入口，因为自己的插件需要对多个App注入，就想增加开关来控制插件是否生效，顺便学习一下`Preference` `Bundle`.
整个开发过程回顾一下大概是:

1. `Cycript`调试拿到视频`URL`的成员变量
2. 查看头文件查看属性与方法，`hook`初始化方法添加触发手势
3. 编写`Tweak`调试测试，适配多个视频场景
4. 编写`Tweak`的`Preference Bundle`控制插件开关（花了不少时间）
5. 整理

# 参考

[Tweak开发：给调音量增加震动反馈](https://bbs.pediy.com/thread-223122.htm#%E5%8F%82%E8%80%83%E9%93%BE%E6%8E%A5)
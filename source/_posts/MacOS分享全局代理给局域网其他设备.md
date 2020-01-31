---
title: MacOS分享全局代理给局域网其他设备
date: 2020-01-31 21:42:22
tags: MacOS
categories: MacOS
---

# 为什么要这么做
在出租屋里的路由器配置了SSR，所有设备科学上网已经很方便了。回到老家后，出现了一些需要科学上网的场景却又很难实现，比如玩switch游戏时需要科学上网，懵逼了。于是想到了自己的MBP有SSR，能开启全局代理然后分享出来吗？

# 正题
## 配置
这个方法实现了的效果是，同一局域网内的设备使用MBP上的全局代理，这里不说SSR怎么配置。

下载安装[privoxy](http://www.privoxy.org/sf-download-mirror/),注意对应平台。根据环境，我使用的是`Privoxy 3.0.26 64 bit.pkg`这个包。

下载安装完毕后，修改一些需要自定义的配置，打开`/usr/local/etc/privoxy`目录下的`config`文件，搜索`forward-socks5t`,并将端口号改为自己SSR里配置的端口（下面的1086是笔者使用的端口）

```plain
#  Examples:
#
#      From the company example.com, direct connections are made to
#      all "internal" domains, but everything outbound goes through
#      their ISP's proxy by way of example.com's corporate SOCKS 4A
#      gateway to the Internet.
#
#        forward-socks4a   /              socks-gw.example.com:1080  www-cache.isp.example.net:8080
#        forward           .example.com   .
#
#      A rule that uses a SOCKS 4 gateway for all destinations but no
#      HTTP parent looks like this:
#
#        forward-socks4   /               socks-gw.example.com:1080  .
#
#      To chain Privoxy and Tor, both running on the same system, you
#      would use something like:
#
        forward-socks5t   /               127.0.0.1:1086 .
#
#      Note that if you got Tor through one of the bundles, you may
#      have to change the port from 9050 to 9150 (or even another
#      one). For details, please check the documentation on the Tor
#      website.
```

继续搜索`listen-address`，将`127.0.0.1`修改为`0.0.0.0`,端口任意修改为未占用的（笔者使用的是2134）

```plain
#  Example:
#
#      Suppose you are running Privoxy on a machine which has the
#      address 192.168.0.1 on your local private network
#      (192.168.0.0) and has another outside connection with a
#      different address. You want it to serve requests from inside
#      only:
#
#        listen-address  192.168.0.1:8118
#
#      Suppose you are running Privoxy on an IPv6-capable machine and
#      you want it to listen on the IPv6 address of the loopback
#      device:
#
#        listen-address [::1]:8118
#
listen-address  0.0.0.0:2134
#
```

## 开启服务

```
# 进入 Privoxy 开启关闭脚本的文件
cd /Applications/Privoxy
# 开启 Privoxy:
sudo ./startPrivoxy.sh
# 关闭 Privoxy:
sudo ./stopPrivoxy.sh

```

## 设备共享

将需要共享全局代理的设备连入与MBP同一网络，然后修改无线局域网配置。将MBP的IP地址和`listen-address`配置的端口填入HTTP代理中

<div align=center> 	<img src=https://kinkenyuen.oss-cn-shenzhen.aliyuncs.com/images/设置代理.jpg width=50% /> 	<br> 	<div style="color:orange; border-bottom: 1px solid #d9d9d9;     	display: inline-block;     	color: #999;     	padding: 2px;">设置代理</div> </div>

<div align=center> 	<img src=https://kinkenyuen.oss-cn-shenzhen.aliyuncs.com/images/使用代理.jpg width=50% /> 	<br> 	<div style="color:orange; border-bottom: 1px solid #d9d9d9;     	display: inline-block;     	color: #999;     	padding: 2px;">使用代理</div> </div>








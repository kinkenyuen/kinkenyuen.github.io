---
title: 简单配置iTerm2与Zsh
date: 2020-01-04 13:03:10
tags: Macos
categories: Macos
---

# iTerm2

[iTerm2](https://iterm2.com/)

官网下载直接安装即可

主题配置使用Dark Background

## 安装Powerline字体
为了终端下能正确的显示fancy字符，需要安装powerline字体，这样，这些fancy字符不至于显示为乱码。 GitHub上已经有制作好的Powerline字体，可以下载了直接安装到系统：

> Powerline字体下载: [https://github.com/powerline/fonts](https://github.com/powerline/fonts)

安装好之后，就可以选择一款你喜欢的Powerline字体了：
`Preferences` -> `Profiles` -> `Text` -> `Font`

## 修改打开窗口的默认位置

`Preference`->` Profiles` -> `Window` -> `Style`

## 修改背景透明度

`Preference`->` Profiles` -> `Window` -> `Transparency`

快捷键`cmd + U`切换透明与非透明

# Zsh与Oh My Zsh
查看支持的shell

`cat /etc/shells`

```
# List of acceptable shells for chpass(1).
# Ftpd will not allow users to connect who are not using
# one of these shells.

/bin/bash
/bin/csh
/bin/ksh
/bin/sh
/bin/tcsh
/bin/zsh
```

Macos 系统默认安装了zsh

安装Oh My Zsh ,通过[官网](https://ohmyz.sh/)的脚本安装

```
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

修改Oh My Zsh主题
默认主题为robbyrussell，可以通过修改 ~/.zshrc 文件配置

```
ZSH_THEME="robbyrussell"
```

## 增强插件
### 命令提示
克隆`zsh-autosuggestions`到`plugins`目录

```
git clone git://github.com/zsh-users/zsh-autosuggestions $ZSH_CUSTOM/plugins/zsh-autosuggestions
```

修改`.zshrc`配置文件

```
plugins=(zsh-autosuggestions git)
```

### 语法高亮
`homebrew`安装 `zsh-syntax-highlighting` 插件

```
brew install zsh-syntax-highlighting
```

修改`.zshrc`配置文件,添加一行内容

```
source /usr/local/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
```

# 配合Go2Shell 快速打开终端

[官网地址](https://zipzapmac.com/Go2Shell)

下载后打开选择默认终端，然后安装到`Finder`菜单栏即可



---
layout: post
title: VirtualBox使用注意事项
categories: VirtualBox
keywords: VirtualBox Network
permalink: /Ubuntu/vbAttention
---

本文主要介绍在使用VirtualBox安装虚拟机系统的一些注意事项，土豪慎入😂。。。

**目录**

* TOC
{:toc}

### Ubuntu安装界面无法全屏
安装Ubuntu系统过程中，点击全屏，整个窗口全屏了，但Ubuntu安装界面窗口并未随之扩展(如下图
绿色方框中的界面并未全屏显示)，导致系统安装界面显示不全，如：下图红色小圈显示的是返回按钮，
确定按钮不显示，无法手动点击。

![界面无法全屏](/images/posts/virtualbox/notfullscreen.jpg "界面无法全屏")

网上提到了几种解决方案，尝试后无法解决该问题
- 调整分辨率，尽管界面文字都调大了，但仍然无法解决显示不全的问题。
- 安装VirtualBox增强功能包（前提：只有在Ubuntu系统安装成功后才可安装该功能包，否则会报错）。前提与当前问题不匹配

尝试了N种方案后，最终的解决方案非常非常之简单，吐了一大口血😂，就是tab键（MacBook Pro上的`->|`键）
切换到确认按钮点击确认键即可！！！想想就想拿脑袋撞豆腐，傻不隆咚的。

最后来说下在Ubuntu系统成功后，我们来看下如何安装VirtualBox增强功能包，每版本VirtualBox对应有各自版本的增强包，
此处我使用的当前的最新版VirtualBox 6.1.12：
1. 在[VirtualBox官网下载页面][1]下载适配Mac OS的最新版VirtualBox 6.1.12（VirtualBox-6.1.12-139181-OSX.dmg），
并对应下载VirtualBox 6.1.12增强包（Oracle_VM_VirtualBox_Extension_Pack-6.1.12.vbox-extpack.iso）
2. 在VirtualBox中启动Ubuntu系统，选择"VirtualBox" - "偏好设置..." - "扩展"，选择"添加新包"。

![添加增强包](/images/posts/virtualbox/extension1.jpg "添加增强包")

3. 在Ubuntu虚拟机中，在终端输入 `apt install virtualbox-guest-utils`完成安装，虚拟机可全屏显示。

### 配置共享目录
搭建在VirtualBox上的Ubuntu虚拟机，如何与MacBook Pro宿主机共享文件？接下来就来看看共享目录具体配置
1. 在VirtualBox中选中待配置的虚拟机，选择"配置" - "共享文件夹" - "添加"，添加对应属性。

![配置共享目录1](/images/posts/virtualbox/share1.jpg "配置共享目录1")
![配置共享目录2](/images/posts/virtualbox/share2.jpg "配置共享目录2")

2. 启动Ubuntu系统，在终端输入`mkdir /mnt/share && mount -t vboxsf share /mnt/share`完成挂载
3. 配置开机自启动挂载，在终端输入`vim /etc/rc.local`编辑文件，然后将`mount -t vboxsf share /mnt/share`
内容写入文件`exit 0`之前，保存即可。

此时，宿主机/Users/wewe/share目录成功挂载到虚拟机/mnt/share目录下，增多其它虚拟机，可共用同一个宿主机共享目录。


### VirtualBox为虚拟机配置提供的几种网络类型

Reference links:
[1] https://www.virtualbox.org/wiki/Downloads
---
layout: post
title: 苹果电脑使用小技巧
categories: MacBookProTips
keywords: macbookProTips
permalink: /macbookProTips
---

日常在使用苹果电脑时，比如：安装环境，快捷键等，时常会发现有些常用的问题因为时间久了或系统更新了被遗忘。
本篇博客主要是用于记录那些惯常发生的问题和使用小技巧，从而避免耗费大量的时间用于同个问题的解答。

**目录**

* TOC
{:toc}

### 环境变量永久生效
重装OS 10.15版本后，系统shell推荐使用zsh，不再使用原来的bash，因此原来通过<b>~/.bash_profile</b>配置的
Java, Go, Maven等环境变量也顺势需要替换为新的配置文件，但新的配置文件不是<b>～/.zsh_profile</b>，该配置
文件配置了环境变量，使用`source ～/.zsh_profile`命令可以使环境变量生效，但仅限于当前的终端生效，
新开终端会失效，因此，若想配置的环境变量永久生效，应在用户主目录（～）下创建<b>.zshrc</b>配置文件，在该配置文件
配置所需环境变量，使用`source ～/.zshrc`命令即可使环境变量永久生效。

### 从访达（Finder）进入系统目录
打开Finder通常只显示桌面，下载等目录，若用户想进入主目录或者系统目录，如：/usr/bin，会发现在Finder上难以找到
操作入口，此时，可以使用`command+shift+G`快捷键，接着在跳出的输入框中输入要进入的目录，如：/usr/bin即可。
好处：用户若需要拷贝系统目录文件就不必仅依赖终端去操作，在图形界面上操作即可。
坏处：切忌随意删除系统文件，容易搞崩机器。

### 启动台（Launchpad)删除残留无用图标
- 打开访达（Finder），使用`command+shift+G`快捷键，在弹框中输入"/private/var/folders"，进入访达（Finder）窗口
- 在访达（Finder）窗口搜索栏输入"com.apple.dock.launchpad"，搜索范围点击选择“folders”
- 接着进入“com.apple.dock.launchpad” 文件夹，找到 “db”文件夹
- 选中“db”文件夹，在右边弹出目录，选择"新建位于文件夹位置的终端窗口"（也可以通过终端直接cd全路径进入该db目录)

  ![打开指定位置终端窗口](/images/posts/macbookProTips/open_terminal.jpg "打开指定位置终端窗口")
- 在终端输入`sqlite3 db "delete from apps where title='yourRemoveAppName';"&&killall Dock`完成删除

  注： yourRemoveAppName即为你需要删除的应用名称

### 屏幕截图
- 方法1. 系统默认使用`command+shift+4`快捷键进行屏幕截图
- 方法2. 登陆微信客户端，使用`control+command+A`快捷键进行截图

### 持续更新中。。。


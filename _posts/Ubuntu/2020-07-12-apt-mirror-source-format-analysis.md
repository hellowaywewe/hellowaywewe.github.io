---
layout: post
title: APT镜像软件源更新及格式解析
categories: Ubuntu
keywords: Mirrors APT
permalink: /Ubuntu/mirrorSource
---

Apt是Ubuntu的包管理工具，对于经常操作Ubuntu系统的开发者来说，常常需要使用apt-get命令安装软件，
但由于系统默认的软件源地址是国外的，受限于墙和网络下载速度等原因，开发者往往需要更换源地址，将国外
的镜像源地址更换为国内的镜像源地址，过去我一直是直接搜索别人现成可使用的源地址，然后手动替换保存，
今天我们就一探究竟，看下（/etc/apt/sources.list）镜像源文件的具体内容和格式。

**目录**

* TOC
{:toc}

我们先来看下镜像源是如何更换更新的，再来细看源文件的内容格式。

### 镜像源更新
##### 获取 Ubuntu 发行版代号
每个Ubuntu发行版都对应有各自的代号（Codename），在终端输入`lsb_release -a`命令获得Codename
后续可通过Codename查找对应的国内源：
![Ubuntu发行版代号](/images/posts/ubuntu/ubuntu_codename.png "Ubuntu发行版代号")

##### 更换国内源
国内目前有许多稳定可用的镜像源，如：
```
# 清华大学源
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial multiverse
deb-src http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial multiverse
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-updates multiverse
deb-src http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-updates multiverse

# 阿里云源
deb http://mirrors.aliyun.com/ubuntu/ xenial multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ xenial multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates multiverse

# 中科大源
deb https://mirrors.ustc.edu.cn/ubuntu/ xenial multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ xenial multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ xenial-updates multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ xenial-updates multiverse

# 网易163源
deb http://mirrors.163.com/ubuntu/ xenial multiverse
deb-src http://mirrors.163.com/ubuntu/ xenial multiverse
deb http://mirrors.163.com/ubuntu/ xenial-updates multiverse
deb-src http://mirrors.163.com/ubuntu/ xenial-updates multiverse
```
在修改Ubuntu 的源文件（/etc/apt/sources.list）前先备份（/etc/apt/sources.list.bak），
在终端中使用`vim /etc/apt/sources.list`命令编辑替换对应的国内源，通常被替换的部位如下图所示，
最后保存，执行`apt-get update`命令更新即可。
![Ubuntu国内镜像源](/images/posts/ubuntu/fast_source.jpg "Ubuntu国内镜像源")
升级后，会发现软件下载安装速度提升很多。

### 镜像源文件解析
Ubuntu的镜像源分为官方源（即上述所说的/etc/apt/sources.list）和ppa(/etc/apt/sources.list.d/)，
可以将ppa理解为个人软件源，是为Ubuntu用户提供的维护Ubuntu开发者的平台，开发者可以自由参与Ubuntu或相关
自由软件的开发等工作，并将更新的软件包存放在https://launchpad.net/网站中。我们通常所说的换源，指的是
更换官方源，下面我们就来看下/etc/apt/sources.list官方源文件内容格式：
```
deb http://archive.ubuntu.com/ubuntu/ xenial multiverse
deb-src http://archive.ubuntu.com/ubuntu/ xenial multiverse
deb http://archive.ubuntu.com/ubuntu/ xenial-updates multiverse
deb-src http://archive.ubuntu.com/ubuntu/ xenial-updates multiverse
```
整个软件源格式分为4个部分：
1. 软件包格式, 分为deb(二进制包)和deb-src(源代码包)两种。
2. 软件源镜像站地址，支持https，http和cdrom等下载。
3. 发行版本代号(Codename)及相关安全性更新分类。 （在镜像源目录结构中对应有各自的目录）
4. 按照软件自由度分类而来，分为main,restricted,universe和multiverse等。（在镜像源目录结构中对应有
各自的目录）
```
main: "基本组件"，仅包含符合Ubuntu协议要求并由Ubuntu团队维护的软件。
restricted: "受限组件"，包含重要的但并不具备合适自由协议的软件，如显卡驱动，仍由Ubuntu团队维护。
universe: "社区维护组件"，种类多，不由Ubuntu团队维护，由相关社区团队维护。
multiverse: "非社区维护组件"，不由Ubuntu团队维护，通常为商业公司编写维护的软件。
```
以ubuntu国外官方源为例，在浏览器输入`http://archive.ubuntu.com/ubuntu/`网址，Ubuntu镜像源目录
结构如下图所示：
![Ubuntu镜像源目录结构](/images/posts/ubuntu/ubuntu_source_dir.jpg "Ubuntu镜像源目录结构")

其中，
1. dists目录：包含Ubuntu发行版及附加仓库目录（如：xenial，xenial-updates，xenial-security等）
![Ubuntu镜像源dists目录结构](/images/posts/ubuntu/ubuntu_dists_dir.jpg "Ubuntu镜像源dists目录结构")
![Ubuntu镜像源二进制软件包](/images/posts/ubuntu/ubuntu_binary_dir.jpg "Ubuntu镜像源二进制软件包")
2. pool目录：包含Ubuntu发行版及已发布软件包的物理地址
![Ubuntu镜像源pool目录结构](/images/posts/ubuntu/ubuntu_pool_dir.jpg "Ubuntu镜像源pool目录结构")

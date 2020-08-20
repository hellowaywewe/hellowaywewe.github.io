---
layout: post
title: VirtualBox虚拟机的4种网络类型
categories: VirtualBox
keywords: VirtualBox Network
permalink: /Ubuntu/vboxNetwork
---

VirtualBox为虚拟机配置提供了多种网络类型，我们在使用VirtualBox部署虚拟机后，常常需要为虚拟机配置网络。
根据不同的应用场景（如：虚拟机是否需要联网，虚拟机间是否需要互通，虚拟机与宿主机间是否需要互通等），我们
往往需要选择配置不同的网络类型，这就要求我们对不同的网络类型有一定的认识。今天主要为大家介绍其中应用比较
广泛的3种，分别是：网络地址转换（NAT模式），桥接网卡（Bridged模式）和仅主机网络（Host-only模式）。
如果有不对或不足的，欢迎纠错补充。

**目录**

* TOC
{:toc}

### 系统介绍
Memory | :-: | 16G
OS | :-: | Mac OS 10.15.5
Software | :-: | VirtualBox 6.1.12, Guest OS: Ubuntu 16.04

### 网络地址转换（NAT模式）
##### 原理
配置NAT模式的虚拟机会生成各自的NAT Engine，用于网络地址转换。VirtualBox默认为虚拟机配置NAT模式，
会提供一个虚拟路由器，为虚拟机分配：
- IP地址（如：10.0.2.15）
- 子网掩码255.255.255.0
- 广播地址10.0.2.255
- 网关10.0.2.2
- DNS服务器与主机中的相同
- DHCP服务器10.0.2.2

其中，
- 10.0.2.2分配给主机，即利用主机充当网关，虚拟机可访问外网。
- 虚拟机通过10.0.2.2能与主机通信，但是主机无法访问虚拟机（需要用端口转接才能访问）。
- 虚拟机间不通信，即使它们的IP地址都是10.0.2.15，即使我们手动为其设置同个网段的不同IP，即使他们的网关IP相同，
虚拟机间也无法通信。原因是：他们属于不同的NAT Engine，只是恰好分配了IP值相同的网关，但实际上它们不在同一个网络中。

![NAT模式](/images/posts/virtualbox/NAT.png "NAT模式")

##### 特点及应用场景
1. 特点如下
- 如果主机可以访问外网，虚拟机可以访问外网
- 虚拟机之间不能ping通
- 主机不能ping通虚拟机
- 虚拟机可以ping通主机（此时主机充当虚拟机网关，可ping通）

操作示例：
虚拟机的IP为10.0.2.15，主机的IP为172.20.10.3。
1) 在虚拟机中ping 172.20.10.3，可正常通信
![NAT模式示例1](/images/posts/virtualbox/nat_ping1.png "NAT模式示例1")

2) 在Mac主机中ping 10.0.2.15，则无法ping通
![NAT模式示例2](/images/posts/virtualbox/nat_ping2.png "NAT模式示例2")

2. 适用场景
- 虚拟机只要求可以上网，无其它特殊要求。
- IP有限，不能占用过多的内网IP。

##### VirtualBox配置操作
1) VirtualBox上选中指定的虚拟机进行网络设置：“连接方式” 选择 “网络地址转换（NAT)"。
![NAT模式配置操作](/images/posts/virtualbox/nat_op.png "NAT模式配置操作")

2) 设置成功后，开启虚拟机，虚拟机自动获取IP，统一为10.0.2.15，网关为10.0.2.2。

注：若需手动更改IP，则使用命令： `ifconfig enp0s3 10.0.2.100 netmask 255.255.255.0`，
其中enp0s3为虚拟机网卡名称，可通过`ifconfig`命令查询本虚拟机的网卡等信息。
可通过`ping –c 3 baidu.com`查看本机是否可以访问外网，其中`-c 3`表示ping包时发送3个包。

### 桥接网卡（Bridged模式）
##### 原理
通过主机网卡，架设一条桥，直接连入到网络中。每台虚拟机都能分配到一个独立的网络IP，所有网络功能完全
和网络中真实的机器一样。主机与虚拟机间，虚拟机与虚拟机间可相互通信。虚拟机没有独立的硬件，它需要
借助主机的网卡，因此，主机若断开网络，虚拟机也就断网了。配置了Bridged模式的虚拟机，若无法通过主机所
在网络中的DHCP服务获取可访问外网的IP地址，则需要手动为虚拟机和主机配置同个网段的IP。
![Bridged模式](/images/posts/virtualbox/BRIDGED.png "Bridged模式")

##### 特点及应用场景
1. 特点如下
- 如果主机可以上网，虚拟机可以上网
- 虚拟机之间可以ping通
- 虚拟机可以ping通主机
- 主机可以ping通虚拟机
上述特性都要求：主机可以上网。若主机不可以上网，则该特性失效。

2. 适用场景
- 虚拟机需要访问外网，且虚拟机间可通信，
- 适用主机IP网段保持固定不变的场景。
- 不适用于生产环境的集群部署（一旦主机切换网络，虚拟机需保证和主机属于同个网段，需要随之更新虚拟机IP）。

##### VirtualBox配置操作
VirtualBox上选中指定的虚拟机进行网络设置：“连接方式” 选择 “桥接网卡”。
“界面名称” 选择 （如果你的笔记本有无线网卡和有线网卡，需要根据当前的上网方式对应选择）。
![Bridged模式配置操作](/images/posts/virtualbox/bridged_op.jpg "Bridged模式配置操作")

注：虚拟机IP自动获取，若无法自动获取，可手动配置，须与主机IP在同一网段内，网关与主机网关相同。
如：主机IP为172.20.10.2，子网掩码为255.255.255.240，网关为172.20.10.1，则手动设置虚拟
机的IP必须为172.20.10.3到172.20.10.14中的一个，网关和DNS服务器都为主机的网关172.20.0.1，
否则虚拟机无法访问外网。
（手动配置方法可参见[Ubuntu常用基础知识点](http://we.wewelove.cn/Ubuntu/generalBasics) 一文）

### 仅主机网络（Host-only模式）
##### 原理
采用Host-Only网络模式构建一个主机与各个虚拟机的局域网，VirtualBox会为主机及虚拟机都虚拟出一块
用于Host-Only连接的网卡（如下图中的vboxnet0），虚拟机以该网卡IP作为网关，使得主机与虚拟机间，
虚拟机与虚拟机间都可相互通信，但虚拟机无法访问外网。
![Host-Only模式](/images/posts/virtualbox/HOSTONLY.png "Host-Only模式")

##### 特点及应用场景
1. 特点如下
- 虚拟机不可以上网
- 虚拟机间可以ping通
- 虚拟机可以ping通主机（注意虚拟机与主机通信是通过虚拟网卡）
- 主机可以ping通虚拟机

2. 适用场景
- 无论主机可否访问外网，都可采用host-only模式模拟出局域网，使得所有虚拟机可以互通。
- 虚拟机不需要访问外网

##### VirtualBox配置操作
1) VirtualBox上选中指定的虚拟机进行网络设置：“连接方式” 选择 “仅主机（Host-Only）网络”。
“界面名称” 选择 “vboxnet0”。
![Host-Only模式配置操作](/images/posts/virtualbox/hostonly_op1.png "Host-Only模式配置操作")

2) 上述vboxnet0是由VirtualBox配置的，选中“vboxnet0” 点击红色箭头所指的“配置”图标，进入“主机虚拟界面”，
配置vboxnet0的IP和子网掩码。紧接着，点击“DHCP服务器”进行配置。
![Host-Only模式配置操作2](/images/posts/virtualbox/hostonly_op2.png "Host-Only模式配置操作2")
![Host-Only模式配置操作3](/images/posts/virtualbox/hostonly_op3.png "Host-Only模式配置操作3")
![Host-Only模式配置操作4](/images/posts/virtualbox/hostonly_op4.png "Host-Only模式配置操作4")

虚拟机IP可自动获取，也可以自己手动配置，虚拟机网关为主机中虚拟网卡vboxnet0的IP，如：192.168.99.1，
虚拟机IP与vboxnet0虚拟网卡IP同属于一个网段即可，如：192.168.99.100。

### 亲手实践小案例
我们必须先清楚自己要配置的机器情况及架构，明确自己的网络选型（如网络基本模式介绍模块所述），根据自己的需要
选择合适的网络模式进行配置。

##### 待实现的场景需求
- 集群架构：下图是使用VirtualBox创建的四台虚拟机（分别是：ceph-admin、ceph-mon、spdkubuntu-osd0、
spdkubuntu-osd1），都安装有ubuntu 16.04的操作系统，搭载在Mac宿主机上。
![Ceph集群整体架构](/images/posts/virtualbox/ceph_1.png "Ceph集群整体架构")

- 场景需求：搭建 1+3 Ceph集群，要求四台虚拟机均可访问外网，同时虚拟机之间可相互通信。

##### 选型分析
根据场景需求描述，单独使用NAT模式和Host-Only模式都是不满足要求的，因为NAT模式可实现虚拟机访问外网，
但不可实现虚拟机间的通信，而Host-Only模式可实现虚拟机间通信，但不可实现虚拟机访问外网。
桥接模式可实现虚拟机访问外网，同时实现虚拟机之间的通信，但由于Mac宿主机使用Wifi和热点联网，经常需要
更换访问外网的IP，因此每次更换IP时，虚拟机网络都需要重新配置，对ceph集群来说，并不适用。我们需要固定
不变的IP用于虚拟机之间的通信。

综上所述，我们可以采用双网卡，结合NAT模式和Host-Only模式去进行网络配置。

##### 动手配置
1. NAT模式配置
![动手配置1](/images/posts/virtualbox/try1.png "动手配置1")
![动手配置2](/images/posts/virtualbox/try2.png "动手配置2")

2. Host-Only模式配置
![动手配置3](/images/posts/virtualbox/try3.png "动手配置3")
![动手配置4](/images/posts/virtualbox/try4.png "动手配置4")

3. 检验结果
配置完成之后，在终端中重置网卡：
```shell
ifconfig enp0s3 down
ifconfig enp0s8 down
ifconfig enp0s3 up
ifconfig enp0s8 up
``` 
最后输入`ifconfig`，验证配置成功,然后通过ping命令验证通信是否成功。
![检验结果](/images/posts/virtualbox/res_check.png "检验结果")

注：若出现主机无法ping通虚拟机的情况，请首先确认虚拟机防火墙已关闭。。








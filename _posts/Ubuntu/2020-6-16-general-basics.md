---
layout: post
title: Ubuntu常用基础知识点
categories: Ubuntu
keywords: Basics
permalink: /Ubuntu/generalBasics
---

在使用Ubuntu系统搭建安装环境时，往往会有一些通用的知识点，以此博客作为起点，开始整理，以免因为时间久了被遗忘，减少时间查询成本。

**目录**

* TOC
{:toc}

### 手动配置虚拟主机静态IP
```
# 编辑/ect/network/interfaces文件
vim /ect/network/interfaces
# 在文件末尾追加下述配置信息，然后保存
auto enp0s3
iface enp0s3 inet static
address 192.168.43.174
netmask 255.255.255.0
gateway 192.168.43.1
# 重启网络服务使之生效
/etc/init.d/networking restart
```
此时，通过`ping -c 1 baidu.com`会发现通过域名无法ping通，但`ping -c 1 64.31.6.54`通过百度IP是可以ping通的，这说明配置的
主机静态IP（192.168.43.174）已经可以联网，还需要配置DNS进行域名解析。

### 手动为虚拟主机配置DNS服务器
```
# 编辑/ect/network/interfaces文件
vim /ect/network/interfaces
# 在文件末尾追加下述配置信息,可设置一个（nameserver）或多个(nameservers),多个DNS地址通过空格分开，然后保存
nameservers 114.114.114.114 114.114.115.115 223.5.5.5 223.6.6.6
# 重启网络服务使之生效
/etc/init.d/networking restart
```
下图为静态IP和DNS服务器配置
![静态IP和DNS服务器配置](/images/posts/ubuntu/network.jpg "静态IP和DNS服务器配置")
注：
- 由于本机是搭建在virtualbox上的虚拟机，因此DNS与物理主机的DNS保持一致即可,此时，`ping -c 1 baidu.com`通过域名可以ping通。
- 看了不少文档介绍说可在/etc/resolv.conf文件中追加DNS配置信息，但机器重启后该文件会被覆盖失效，解决方法除上述方案外，还可直接在
启动脚本/etc/rc.local中将DNS服务器地址写入/etc/resolv.conf：`echo "nameservers x.x.x.x" >/etc/resolv.conf`,这样开机
可重新配置并启动。

### 修改虚拟主机名
```
# 编辑/ect/hostname文件
vim /ect/hostname
# 在文件中修改主机名（如：kube-master），然后保存
kube-master
```

### 添加虚拟主机IP与虚拟主机名映射关系
```
# 编辑/ect/hosts文件
vim /ect/hosts
# 在文件末尾追加对应IP与主机名，然后保存
192.168.43.174 kube-master
192.168.43.175 kube-node1
```

### 防火墙启动/关闭/重启/状态查看
```
systemctl start firewalld.service
systemctl stop firewalld.service
systemctl restart firewalld.service
systemctl status firewalld.service
```

### NTP时钟同步
在同一局域网内搭建集群时，在分布式调度/定时等作业中，常常需要确保几台机器的时间同步，保证数据的实时同步。选择局域网里的一台服务器作为ntp服务器，
比如在部署K8S集群时，选择k8s-master主节点作为NTP服务器，其他k8s-node节点安装ntpdate客户端，以k8s-master作为基准时钟服务器进行时间同步。

##### date命令查看时间日期及时区信息
```
date -R
```
![查看时间信息](/images/posts/ubuntu/time_op1.jpg "查看时间信息")

##### cat命令查看时区
```
cat /etc/timezone
```
![查看时区](/images/posts/ubuntu/time_op1_1.jpg "查看时区")

##### 修改时区方案1：date命令修改时区，按照图片中提示的步骤逐一操作
```
tzselect
```
![修改时区1](/images/posts/ubuntu/time_op2.jpg "修改时区1")
![修改时区2](/images/posts/ubuntu/time_op3.jpg "修改时区2")
tzselect命令用于选择时区。要注意的是tzselect只是帮我们把选择的时区显示出来，并不会实际生效，我们需要用上图tzselect生成的提示
去修改profile文件使得修改时区生效。
```
# 编辑/etc/profile文件。
vim /etc/profile
# 在文件末尾添加tzselect生成的提示：TZ='Asia/Shanghai'; export TZ，然后保存文件
TZ='Asia/Shanghai'; export TZ
# 使配置生效
source /etc/profile
```
![时区配置生效](/images/posts/ubuntu/time_op4.jpg "时区配置生效")
![检查时区修改结果](/images/posts/ubuntu/time_op5.jpg "检查时区修改结果")
使用`date -R`命令，发现时区由原来的+0900修正为+0800。

##### 修改时区方案2：为虚拟主机设置使用亚洲/上海（+8）时区
```
# 将系统时区文件链接到/usr/share/zoneinfo/Asia/Shanghai文件，或者直接替换系统时区文件，重启生效。
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

##### 安装及配置ntp服务器
```
# 在k8s-master（192.168.43.174）机器上安装NTP
apt-get -y install ntp
# 启动NTP服务器
/etc/init.d/ntp start
```
![ntp服务器安装](/images/posts/ubuntu/ntp_install.jpg "ntp服务器安装")
![ntp服务器启动](/images/posts/ubuntu/ntp_start.png "ntp服务器启动")

##### 安装ntpdate客户端执行时钟同步
```
# 在k8s-node1（192.168.43.175）机器上安装ntpdate
apt-get -y install ntpdate
# 执行时钟同步，与k8s-master时间一致
ntpdate k8s-master 或者 ntpdate 192.168.43.174
```
![ntpdate客户端安装](/images/posts/ubuntu/ntpdate_install.jpg "ntpdate客户端安装")
![ntpdate时间同步1](/images/posts/ubuntu/ntpdate_sync1.png "ntpdate时间同步1")
![ntpdate时间同步2](/images/posts/ubuntu/ntpdate_sync2.png "ntpdate时间同步2")
如果没有自定义安装NTP服务器，可使用已有开放的NTP服务器作为同步（如阿里云NTP服务器，`ntpdate ntp1.aliyun.com`），建议自建NTP服务器。
![ntpdate阿里云时间同步](/images/posts/ubuntu/ntpdate_sync3.png "ntpdate阿里云时间同步")

### Crontab定时任务

### 持续更新

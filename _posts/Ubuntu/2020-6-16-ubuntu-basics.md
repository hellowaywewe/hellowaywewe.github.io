---
layout: post
title: Ubuntu常用基础知识点
categories: Ubuntu
keywords: Basics
permalink: /Ubuntu/generalBasics
---

在使用Ubuntu系统搭建安装环境时，往往会有一些通用的知识点，以此博客作为起点，开始整理，以免因为
时间久了被遗忘，减少时间查询成本。

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
此时，通过`ping -c 1 baidu.com`会发现通过域名无法ping通，但`ping -c 1 64.31.6.54`
通过百度IP是可以ping通的，这说明配置的主机静态IP（192.168.43.174）已经可以联网，还需要
配置DNS进行域名解析。

### 手动为虚拟主机配置DNS服务器
```
# 编辑/ect/network/interfaces文件
vim /ect/network/interfaces
# 在文件末尾追加下述配置信息,可设置一个（nameserver）或多个(nameservers),
多个DNS地址通过空格分开，然后保存
nameservers 114.114.114.114 114.114.115.115 223.5.5.5 223.6.6.6
# 重启网络服务使之生效
/etc/init.d/networking restart
```
下图为静态IP和DNS服务器配置
![静态IP和DNS服务器配置](/images/posts/ubuntu/network.jpg "静态IP和DNS服务器配置")
注：
- 由于本机是搭建在virtualbox上的虚拟机，因此DNS与物理主机的DNS保持一致即可，此时，
`ping -c 1 baidu.com`通过域名可以ping通。
- 看了不少文档介绍说可在/etc/resolv.conf文件中追加DNS配置信息，但机器重启后该文件会被覆盖失效，
解决方法除上述方案外，还可直接在启动脚本/etc/rc.local中将DNS服务器地址写入/etc/resolv.conf：
`echo "nameservers x.x.x.x" >/etc/resolv.conf`,这样开机可重新配置并启动。

### 修改虚拟主机名
```
# 编辑/ect/hostname文件
vim /ect/hostname
# 在文件中修改主机名（如：kube-master），然后保存
kube-master
```

### 添加虚拟主机IP与虚拟主机名映射关系
```shell
# 编辑/ect/hosts文件
vim /ect/hosts
# 在文件末尾追加对应IP与主机名，然后保存
192.168.43.174 kube-master
192.168.43.175 kube-node1
```

### 防火墙操作
Ubuntu防火墙默认使用ufw,是关闭状态，CentOS7防火墙默认使用firewalld

##### 防火墙启动/关闭/重启/状态查看
```shell
ufw enable
ufw disable
ufw reload
ufw status
```

##### 防火墙打开/关闭某个端口/
```shell
# 允许/禁止所有的外部IP访问本机的25/tcp (smtp)端口
ufw allow smtp
ufw deny smtp

# 允许/禁止所有的外部IP访问本机的22/tcp (ssh)端口
ufw allow 22/tcp
ufw deny 22/tcp

# 允许/禁止外部访问53端口(tcp/udp)
ufw allow 53
ufw deny 53

# 允许/禁止此IP访问本机所有端口
ufw allow from 192.168.43.174
ufw deny from 192.168.43.174
```

##### 删除防火墙规则
```shell 
ufw delete allow smtp
ufw delete deny from 192.168.43.174
```

### NTP时钟同步
在同一局域网内搭建集群时，在分布式调度/定时等作业中，常常需要确保几台机器的时间同步，保证数据的实时
同步。选择局域网里的一台服务器作为ntp服务器，比如在部署K8S集群时，选择k8s-master主节点作为NTP服
务器，其他k8s-node节点安装ntpdate客户端，以k8s-master作为基准时钟服务器进行时间同步。

##### date命令查看时间日期及时区信息
```shell
date -R
```
![查看时间信息](/images/posts/ubuntu/time_op1.jpg "查看时间信息")

##### cat命令查看时区
```shell
cat /etc/timezone
```
![查看时区](/images/posts/ubuntu/time_op1_1.jpg "查看时区")

##### 修改时区方案1：date命令修改时区，按照图片中提示的步骤逐一操作
```shell
tzselect
```
![修改时区1](/images/posts/ubuntu/time_op2.jpg "修改时区1")
![修改时区2](/images/posts/ubuntu/time_op3.jpg "修改时区2")
tzselect命令用于选择时区。要注意的是tzselect只是帮我们把选择的时区显示出来，并不会实际生效，
我们需要用上图tzselect生成的提示去修改profile文件使得修改时区生效。
```shell
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
```shell
# 将系统时区文件链接到/usr/share/zoneinfo/Asia/Shanghai文件，
或者直接替换系统时区文件，重启生效。
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

##### 安装及配置ntp服务器
```shell
# 在k8s-master（192.168.43.174）机器上安装NTP
apt-get -y install ntp
# 启动NTP服务器
/etc/init.d/ntp start
```
![ntp服务器安装](/images/posts/ubuntu/ntp_install.jpg "ntp服务器安装")
![ntp服务器启动](/images/posts/ubuntu/ntp_start.png "ntp服务器启动")

##### 安装ntpdate客户端执行时钟同步
```shell
# 在k8s-node1（192.168.43.175）机器上安装ntpdate
apt-get -y install ntpdate
# 执行时钟同步，与k8s-master时间一致
ntpdate k8s-master 或者 ntpdate 192.168.43.174
```
![ntpdate客户端安装](/images/posts/ubuntu/ntpdate_install.jpg "ntpdate客户端安装")
![ntpdate时间同步1](/images/posts/ubuntu/ntpdate_sync1.png "ntpdate时间同步1")
![ntpdate时间同步2](/images/posts/ubuntu/ntpdate_sync2.png "ntpdate时间同步2")


如果没有自定义安装NTP服务器，可使用已有开放的NTP服务器作为同步（如阿里云NTP服务器，
`ntpdate ntp1.aliyun.com`），建议自建NTP服务器。
![ntpdate阿里云时间同步](/images/posts/ubuntu/ntpdate_sync3.png "ntpdate阿里云时间同步")

### Crontab定时任务
Crontab主要用于定期执行程序
##### 安装并启动Crontab服务
```shell
apt-get -y install cron
service cron start
```

##### 开启Crontab日志
Ubuntu默认不开启cron日志（/var/log下没有cron.log文件）
```shell
# 编辑/etc/rsyslog.d/50-default.conf文件
vim /etc/rsyslog.d/50-default.conf
# 将cron前面的注释符去掉
cron.* /var/log/cron.log
```
![开启cron日志](/images/posts/ubuntu/crontab_log.jpg "开启cron日志")

##### Crontab任务配置
```shell
# 编辑新任务方案1，配置完成后需重启cron进程：
crontab -e
# 编辑新任务方案2，编辑/etc/crontab文件,需要重启系统才可生效
vim /etc/crontab
```
##### Crontab任务解析
任务时间格式：m h dom mon dow user  command，其中，
- m: 表示分钟，为`*`表示每分钟执行，为`a-b`表示在a分钟到b分钟这段时间内都要执行，
为`*/n`表示每 n 分钟执行一次，为`a, b, c,...`表示第 a, b, c,... 分钟要执行
- h: 表示小时，为`*`表示每小时执行，为`a-b`表示在a小时到b小时这段时间内都要执行，
为`*/n`表示每 n 小时执行一次，，为`a, b, c,...`表示第 a, b, c,... 小时要执行
- dom: 表示每个月的第几日（号）
- mon 表示月份
- dow 表示每个星期的第几天，星期几。
- user 执行程序的用户
- command 表示要执行的命令/程序。
   
##### 一个简单的Crontab例子
定时任务：每天上午9点半kube-node1机器与kube-master同步时间
用`crontab -e`命令编辑，然后在末尾添加同步任务`30 9 * * * root  /usr/sbin/ntpdate kube-master`
保存后使用`crontab -l`查看是否编辑成功，然后使用`/etc/init.d/cron restart`重启下cron服务即可。

![Crontab例子](/images/posts/ubuntu/crontab.jpg "Crontab例子")

### 查看/释放端口占用情况
查看当前系统所有被占用端口
```
netstat -tln
```

查看指定端口(如：8080)是否被占用
```
netstat -tln ｜ grep 8080
```

释放被占用端口
```
# 根据端口（如：8080）查询进程，注意带上冒号
lsof -i :8080

# 根据上述查询到的进程号(如：1000)，杀死该进程进而释放被占用端口
kill -9 1000 或者 kill 1000
```

### 持续更新。。。

---
layout: post
title: 基于kubeadm工具部署Kubernetes集群
categories: K8s
keywords: Kubernetes kubeadm
permalink: /Ubuntu/k8sDeploy
---

目前官方提供了3种部署Kubernetes（下面简写k8s）的方式：minikube，kubeadm和二进制包。
- minikube：用于自动化部署单节点k8s的工具，主要用于测试环境学习使用，不可用于生产环境
- kubeadm：用于自动化部署多节点k8s集群的工具，可用于生产环境，使用简单，利于初学者入门
- 二进制包：手动安装，非自动化部署，可用于生产环境需要感知每个组件及组件间的配置细节，遇
到问题较难排查，但有利于全面理解k8s

本文为初学者展示如何使用kubeadm工具快速部署k8s集群，帮助初学者快速入门实践。

**目录**

* TOC
{:toc}

### 系统介绍

HOSTNAME | IP ADDR | 虚拟机OS | Specifications 
:-: | :-: | :-: | :-: 
kube-master | 192.168.43.174 | ubunru 16.04 | 1核4G 
kube-node1 | 192.168.43.175 | ubunru 16.04 | 1核4G

注：CPU/内存要求至少2核/2G，但由于我们是用于测试操作用，不做生产用，选用1核也可以完成部署。

### 基础环境准备
本文直接使用root用户搭建k8s集群

##### 防火墙关闭（2台主机）
修改方式参见 [Ubuntu常用基础知识点](http://we.wewelove.cn/Ubuntu/generalBasics) 一文

##### 修改对应主机名（2台主机）
修改方式参见 [Ubuntu常用基础知识点](http://we.wewelove.cn/Ubuntu/generalBasics) 一文

##### 新增对应主机名及映射IP（2台主机）
修改方式参见 [Ubuntu常用基础知识点](http://we.wewelove.cn/Ubuntu/generalBasics) 一文

##### 时钟同步（2台主机）
修改方式参见 [Ubuntu常用基础知识点](http://we.wewelove.cn/Ubuntu/generalBasics) 一文：
- 在k8s-master（192.168.43.174）机器上安装NTP
- 在k8s-node1（192.168.43.175）机器上安装ntpdate
- 在k8s-node1配置Crontab定时同步任务（可选）

##### 禁用swap（2台主机）
关闭swap即禁用虚拟内存，主要出于对性能的考虑
```shell
# 打开/etc/fstab文件，将swap那一行内容注释掉，然后关闭swap
sed -i '/swap/ s/^/#/' /etc/fstab
swapoff -a
```

### 正式部署
##### Docker安装（2台主机，版本：18.06.2~ce~3-0~ubuntu）
```shell
# 安装一些必要的系统工具
apt-get update
apt-get -y install apt-transport-https ca-certificates curl software-properties-common
# 安装GPG证书
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
# 写入软件源信息
add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
# 更新并安装指定版本Docker-CE
apt-get -y install docker-ce=18.06.2~ce~3-0~ubuntu
```
使用`docker info `指令查看docker安装成功后的系统信息

![查看docker信息1](/images/posts/kubernetes/docker_info.png "查看docker信息1")

观察上图，会发现docker的Cgroup driver是cgroupfs，而k8s集群部署建议Cgroup driver修改为systemd，
使用systemd来进行资源控制与管理，cgroupfs和systemd都是对cgroup（用于系统资源隔离）原生接口的一种封
装，但systemd更加简单，且被主流Linux发行版所支持，所以，可进行如下修改：
```shell
vim /etc/docker/daemon.json

# 添加如下内容
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}

# 重启docker
systemctl restart docker
```

再次使用`docker info `指令查看docker的Cgroup driver属性是否已修改为systemd

![查看docker信息2](/images/posts/kubernetes/docker_info_change.png "查看docker信息2")

##### kubelet，kubeadm和kubectl安装(2台主机，版本：1.15.0-00)
使用kubeadm工具部署k8s集群，需要访问google拉取kubelet等相关组件的镜像，由于镜像站点在国外，访问速度慢，
镜像拉取常常不成功，导致集群自动化部署失败。因此，国内用户可选用访问 [阿里云镜像站点](https://developer.aliyun.com/mirror/)
获取k8s镜像。
```
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF  
apt-get update
apt-get install -y kubelet=1.15.0-00 kubeadm=1.15.0-00 kubectl=1.15.0-00
# 确保版本不会被自动更新
apt-mark hold kubelet=1.15.0-00 kubeadm=1.15.0-00 kubectl=1.15.0-00
```

##### k8s集群初始化（仅master节点）
指令执行输出的信息，会保存到/etc/kube-server-key文件中
```shell
kubeadm init  --image-repository registry.aliyuncs.com/google_containers --kubernetes-version v1.15.0 --pod-network-cidr=192.168.0.0/16 | tee /etc/kube-server-key
```
部分参数说明：
1. --image-repository：指定镜像源，为避免拉取镜像超时，此处使用阿里云源，网速快则几分钟后就可以看见成功日志输入
2. --kubernetes-version: 指定版本
3. --pod-network-cidr: 指定pod网络地址，为内网网段。calico网络组件建议使用`192.168.*.*`网段，flannel则使用`10.*.*.*`网段

注：执行上述命令若报出如下警告和error信息

![k8s初始化信息](/images/posts/kubernetes/k8s_init_error.jpg "k8s初始化信息")

对应的解决方法：
1. cgroupfs驱动问题，则需要设置docker的cgroup驱动为systemd，修改方法参考Docker安装小节。
2. NumCPU错误是由于系统中只有1个CPU，而k8s安装要求至少2个CPU导致，可在kubeadm init命令后面添加`--ignore-preflight-errors=NumCPU`参数，在飞检时忽略NumCPU错误即可。

执行kubeadm init指令后输出信息如下，按照信息中的提示继续往下执行：

![k8s初始化输出信息](/images/posts/kubernetes/k8s_init_output.jpg "k8s初始化输出信息")

##### 拷贝kubeconfig文件（仅master节点）
拷贝kubeconfig文件到用户主目录的.kube目录，并将该文件的所有者授予该用户
```shell
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```
其中，`$(id -u)`和`$(id -g)`表示当前用户的用户id和组id

##### 安装calico网络插件（仅master节点）
安装calico网络插件，实现pod间通信，可在执行下述命令前先准备好对应镜像，也可在执行下述命令后发现网络不佳时再拉取镜像
```shell
kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/calico.yaml
```
 `` 
- 由于镜像下载速度慢，执行上述命令会一直提示失败重连，解决方法：可使用`docker pull`命令事先拉取calico.yaml的几个镜像
```shell
docker pull calico/cni:v3.8.9
docker pull calico/pod2daemon-flexvol:v3.8.9
docker pull calico/node:v3.8.9
docker pull calico/kube-controllers:v3.8.9
```
calico版本镜像可访问 [calico.yaml](https://docs.projectcalico.org/v3.8/manifests/calico.yaml) 文件找到images属性获得

- 查看kube-system命名空间下的pod状态

```shell
kubectl get pod -n kube-system
```
状态显示全为Running即为安装成功:

![calico安装成功](/images/posts/kubernetes/calico_ok.png "calico安装成功")

##### node节点加入集群（仅node节点）
- 在master节点查看加入节点命令
```shell
cat /etc/kube-server-key | tail -2
```
![节点加入](/images/posts/kubernetes/k8s_join.jpg "节点加入")

- 执行命令，加入节点（仅node节点）
```shell
kubeadm join 192.168.43.174:6443 --token obwwzs.teqoezbgxk850uyb \
    --discovery-token-ca-cert-hash sha256:aab8882230c446a5e81dd57600f99e034f7e0413f5c927298e578eaa64cf0cc1
```

- 在master节点查看集群状态，若网速快，等待几分钟即可。
![查看集群启动状态](/images/posts/kubernetes/k8s_pods.png "查看集群启动状态")

观察上图，发现master节点版本是v1.15.0，虽然不影响集群的使用，但我们想升级下版本，统一为v1.15.2

##### 集群升级
```shell
# 先移除版本不自动升级的标签
apt-mark unhold kubelet=1.15.0-00 kubeadm=1.15.0-00 kubectl=1.15.0-00

# 覆盖安装待升级版本的组件
apt-get install -y kubelet=1.15.2-00 kubeadm=1.15.2-00 kubectl=1.15.2-00

# 升级后集群版本不自动升级
apt-mark hold kubelet=1.15.2-00 kubeadm=1.15.2-00 kubectl=1.15.2-00

# 在所有节点（包括 master、node 节点）执行命令
systemctl daemon-reload 
systemctl restart kubelet
```
同样，在master节点查看集群状态已经，master节点已经从1.15.0升级到1.15.2版本

![查看集群升级状态](/images/posts/kubernetes/k8s_upgrade.png "查看集群升级状态")



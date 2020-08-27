---
layout: post
title: Kubernetes应用部署及可视化展示
categories: K8s
keywords: kubernetes dashboard
permalink: /Ubuntu/k8sUse
---


在上篇 [基于kubeadm工具部署Kubernetes集群](https://we.wewelove.cn/Ubuntu/k8sDeploy) 博客中，我们已经使用 kubeadm 工具成功搭建了 k8s 集群，
接下来我们就来看看如何在搭建好的集群上部署应用，并使用 Web UI 可视化界面查看集群中应用的运行状态，帮助开发者快速分析集群的健康状态。

**目录**

* TOC
{:toc}

### 系统介绍

HOSTNAME | IP ADDR | 虚拟机OS | Specifications 
:-: | :-: | :-: | :-: 
kube-master | 192.168.43.174 | ubunru 16.04 | 1核4G 
kube-node1 | 192.168.43.175 | ubunru 16.04 | 1核4G

注：本文直接使用root用户部署应用，Pod 的内网网段 IP 和子网掩码 是 `192.168.*.*/16`。

### 应用部署
使用 k8s 集群部署 hello 应用，打印出" Hello World！！"字样。注：下述文件统一放在一个目录中（如：/mnt/share）

##### hello 应用实现
- 使用 C++ 编程语言编写简单的hello-world.c文件，并使用命令`gcc hello-world.c -o hello-world`编译生成hello-world二进制可执行文件。

```shell
#include <stdio.h>
#include <unistd.h>

int main(int argc, char *argv[])
{
    int i = 0;
    while( i < 3600 )
    {
        printf("Hello World!!\n");
        i++;
    }
    sleep(i);
    return 0;
}
```

- 编写Dockerfile，制作容器镜像（将生成的hello-world二进制可执行文件拷贝至容器中执行）

```shell
FROM ubuntu:16.04

MAINTAINER yourName yourEmail

RUN mkdir data
COPY hello-world data
ENTRYPOINT ["data/hello-world"]
```

- 编译镜像，并将其上传到个人的 [DockerHub](https://hub.docker.com/) ，此处默认已申请DockerHub账号

```shell
# 登陆DockerHub，输入在DockerHub申请注册的账号密码
doker login

# 编译Dockerfile，生成镜像
- 其中，hello_world是镜像名
- 2.0是版本tag
- .是指定镜像构建过程中的上下文环境的目录，docker build 可将这个指定路径下的文件打包上传到Docker引擎
docker build -t yourDockerHubAccount/hello_world:2.0 .

# 将生成的镜像上传到个人的DockerHub
docker push yourDockerHubAccount/hello_world:2.0
```

- 编写k8s应用部署文件hello.yaml

```shell
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: hello
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: hello
    spec:
      containers:
      - name: hello-world
        image: yourDockerHubAccount/hello_world:2.0
        imagePullPolicy: IfNotPresent
```

- 启动应用（仅master节点）

```shell
kubectl apply -f hello.yaml
```

- 查看应用状态，-o wide表示输出详情

```shell
kubectl get pods -o wide
```
![查看应用状态](/images/posts/kubernetes/app_status.jpg "查看应用状态")

观察上图可见，该应用pod运行在k8s-node1节点上，分配IP：192.168.9.71，属于192.168.*.*网段。

运行`ping -c 1 192.168.9.71`命令可ping通，说明calico网络插件运行正常。

![calico状态](/images/posts/kubernetes/calico_status.jpg "calico状态")

测试pod的中的容器是否在运行

```shell
kubectl exec -it hello-8464b48485-4znrz -- ps -aux | grep hello
```
![程序运行状态](/images/posts/kubernetes/prog_status.jpg "程序运行状态")

观察上图可见,pod的中的容器在执行hello-world程序，因文件名过长被裁切省略显示。

执行输出的结果和运行状态可查看对应pod的日志

```shell
kubectl logs hello-8464b48485-4znrz
```

![pod日志](/images/posts/kubernetes/pod_logs.jpg "pod日志")

### Dashboard 可视化展示
Dashboard 是基于网页的 Kubernetes 用户界面，囊括了 kubectl 命令行工具几乎所有的功能。
你可以使用 Dashboard 在 WEB 界面浏览运行在集群中的应用概况和节点负载等信息，也可以直接
在 WEB 界面将容器应用部署到 Kubernetes 集群中，管理集群资源（如：创建和修改Deployment，
Job等资源，还可以对 Deployment 实现弹性扩缩、滚动升级、重启 Pod等操作）。

##### Dashboard 安装
在网上查找很多教程都使用 [faasx镜像站的kubernetes-dashboard.yaml文件](http://mirror.faasx.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml)，
但在使用时，会发现yaml文件中的 `reg.qiniu.com/k8s/kubernetes-dashboard-amd64:v1.8.3`
镜像下载不下来。

解决方法：将kubernetes-dashboard.yaml的内容复制到本地，然后将其中的image的dashboard镜像
修改为对应版本的阿里云镜像 `registry.cn-hangzhou.aliyuncs.com/kubeapps/k8s-gcr-kubernetes-dashboard-amd64:v1.8.3`
![dashboard镜像](/images/posts/kubernetes/dashboard_image.jpg "dashboard镜像")

然后在master节点执行：

```shell
# 此处的kubernetes-dashboard.yaml是经本地修改过的
kubectl apply -f kubernetes-dashboard.yaml
```

若发现，节点镜像拉取很慢导致启动不成功，因为此处只有node1工作节点，默认镜像会在node1节点启动，如果着急
看效果，可在node1节点手动执行 `docker pull reg.qiniu.com/k8s/kubernetes-dashboard-amd64:v1.8.3`
命令拉取镜像。

##### 查看 Dashboard 是否成功启动

```shell
kubectl get pod --namespace=kube-system
kubectl get deployment kubernetes-dashboard --namespace=kube-system 
kubectl get service kubernetes-dashboard --namespace=kube-system 
```

![dashboard启动确认](/images/posts/kubernetes/dashboard_status.jpg "dashboard启动确认")

##### 设置外部访问权限

```shell
# 允许外部全部用户访问（会占用阻塞终端）
kubectl proxy --address='0.0.0.0'  --accept-hosts='^*$'

# 编写配置文件 dashboard-admin.yaml，为 Dashboard 配置登陆权限
vim dashboard-admin.yaml
# 添加如下内容，为 Dashboard 用户赋予 admin 权限
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
  labels: 
     k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system

# 执行kubectl apply应用权限使之生效
kubectl apply -f dashboard-admin.yml
```

##### 浏览器访问
在浏览器 WEB 页面访问Dashboard查看集群状态。

```shell
# 通过在浏览器输入下述地址，其中yourMasterNodeIP为为kube-master节点的IP：192.168.43.174
http://yourMasterNodeIP:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/.
```

然后直接点击`登录页面`的`跳过`

![dashboard登陆页面](/images/posts/kubernetes/dashboard_ui_login.jpg "dashboard登陆页面")

随后进入Dashboard，就可以查看集群的信息了，页面如下：

![dashboard主页面](/images/posts/kubernetes/dashboard_ui_home.png "dashboard主页面")

下图可以看见hello应用启动的一个状态变化，由于镜像的下载速度慢导致的错误，后来在node1节点手动拉好镜像，解决了该问题。

![dashboard应用状态](/images/posts/kubernetes/dashboard_board_app.png "dashboard应用状态")

### 小知识积累
##### Kubernetes命名空间
在默认情况下，新搭建的集群有三个命名空间：
1. default: 向集群中添加对象而不提供命名空间，默认都会放入此空间
2. kube-system: 用于存放Kubernetes组件，通常情况，该空间不添加运行普通的工作负载，由系统直接管理
3. kube-public: 对所有用户可读，由Kubernetes自己管理
      
##### kubectl命令补全（仅master节点）

```shell
apt-get install bash-completion

# 编辑~/.bashrc文件
vim ~/.bashrc
# 添加如下内容
source <(kubectl completion bash)

# 使配置生效
source  ~/.bashrc
```













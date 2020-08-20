---
layout: post
title: GPG加密软件安装
categories: Ubuntu
keywords: APT GPG
permalink: /Ubuntu/aptGpg
---

正如上文《APT镜像软件源更新及格式解析》所述，apt是Ubuntu的包管理工具，为了加快下载速度，
我们在使用apt安装软件时，常常需要将默认的软件源替换成国内的镜像源。除此之外，有些
软件（如：前面所说的ppa软件）还需要先获取该软件包的GPG公钥，才能成功安装。
关于[GPG加解密知识学习](https://www.ruanyifeng.com/blog/2013/07/gpg.html?tdsourcetag=s_pctim_aiomsg)
的入门不作为本文重点，如需了解，可自行查阅相关资料。

**目录**

* TOC
{:toc}

相信经常操作Ubuntu系统的开发者对下面的指令并不陌生：
```
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl
```
上述指令即是使用apt工具安装Kubernetes软件包(已经过GPG技术加密)。
1. curl执行获取Kubernetes安装所需公钥文件，并将公钥内容添加到apt-key，为后续安装做好准备。
2. 更换Kubernetes国内源，这里通过[阿里云镜像站](https://developer.aliyun.com/mirror/)
查询Kubernetes可用安装包，在/etc/apt/sources.list.d/目录下创建kubernetes.list文件，并将
所需源内容写入该文件保存即可。
3. 使用`apt-get`命令更新镜像源码，并安装Kubernetes。

我们来看看阿里云镜像站kubernetes的目录结构，在浏览器输入`https://mirrors.aliyun.com/kubernetes/`，目录结构如下：
![Kubernetes镜像目录](/images/posts/ubuntu/kubernetes_mirror_dir.jpg "Kubernetes镜像目录")
其中，dists目录存放发布的软件包（二进制包和源码包）
![Kubernetes二进制目录](/images/posts/ubuntu/kubernetes_binary_dir.jpg "Kubernetes二进制目录")，图中url的kubernetes-xenial
和main目录对应`deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main`的kubernetes-xenial和main




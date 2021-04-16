---
layout: post
title: 基于Ubuntu 16.04部署Jenkins
categories: Jenkins
keywords: Jenkins
permalink: /Ubuntu/jenkinsDeploy
---


写这篇博客之前，构思的是一个系列博客，大体想法是基于 Kubernetes 搭建 Jenkins 集群，然后联动 Github
或 Gitee 等代码托管平台构建自动化CI环境，对合入 master 分支的代码做预检查等操作。但这样会显得篇幅过长，对
新手用户来说，会有些吃力，毕竟一口吃不下一个大胖子，所以先借此篇博客给大家暖暖身，了解下Jenkins环境是如何
安装和使用的。网上此类安装文档也比较多，我就简单说下我是如何使用脚本快速安装的，这可能需要大家有一点脚本方面的基础。


**目录**

* TOC
{:toc}

### 基础环境准备
- 一台可联网的 `Ubuntu 16.04` 虚拟机（建议使用独立的公网IP，后续可做域名映射，配置HTTPS认证）
- 全程使用 `root` 用户部署
- 在安装Jenkins之前需要先安装好JAVA环境。

前提：此虚拟机是新建无污染的， 安装后需要先做一定更新，安装必要基础软件，属于用户可选操作

```shell
apt-get update
apt-get -y install vim
```


### 安装脚本文件准备
使用 `vim install_jenkins.sh` 命令创建该脚本文件，脚本内容如下：

```
#! /bin/bash

set -e

# 安装wget文件下载工具
apt-get -y install wget

# 安装OpenJDK
apt install -y openjdk-8-jdk

# 为OpenJDK配置环境变量，使用apt安装JDK的默认安装目录是/usr/lib/jvm/JDK版本
java_path=`ls /usr/lib/jvm/ | grep java-1.8.*`
cat >> /etc/profile <<EOF
export JAVA_HOME=/usr/lib/jvm/$java_path
export JRE_HOME=\${JAVA_HOME}/jre
export CLASSPATH=.:\${JAVA_HOME}/lib:\${JRE_HOME}/lib
export PATH=\${JAVA_HOME}/bin:\$PATH
EOF

# 使配置生效
source /etc/profile

# 安装Jenkins
wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -
echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list
apt-get update
apt-get -y install jenkins

# 启动Jenkins服务
service jenkins start

# 默认Jenkins端口是8080，出于安全考虑,可将其修改为其他端口，如：8090（可选）
sed -i 's/HTTP_PORT=8080/HTTP_PORT=8090/g' /etc/default/jenkins

# 重启Jenkins服务
service jenkins restart

# 防火墙设置，允许访问8090端口
ufw allow 8090
```

### 执行安装操作

```
# 为脚本文件分配可执行权限
chmod +x install_jenkins.sh

# 执行脚步
./install_jenkins.sh
```

### 验证Jenkins是否成功安装

```
# 查看jenkins服务启动状态，若显示active则表示服务已启动
service jenkins status
```

![查看Jenkins状态](/images/posts/jenkins/jenkins_status.png "查看Jenkins状态")

### 浏览器页面访问
在浏览器地址栏输入 `http://yourIP:8090/` ，其中，yourIP即为搭建该Jenkins所在的虚拟机的IP，效果如下：

![Jenkins页面1](/images/posts/jenkins/jenkins_ui1.png "Jenkins页面1")

根据上图提示，在虚拟机终端输入`cat /var/lib/jenkins/secrets/initialAdminPassword`命令显示密码:

![Jenkins_32位密码](/images/posts/jenkins/jenkins_passwd.png "Jenkins_32位密码")

将输出的32位字符密码拷贝到"管理员密码"字段中:

![Jenkins页面2](/images/posts/jenkins/jenkins_ui2.png "Jenkins页面2")

点击"继续"按钮，效果如下：

![Jenkins页面3](/images/posts/jenkins/jenkins_ui3.png "Jenkins页面3")

选择"安装推荐的插件"，若网速不佳，需要等待较长时间, 效果如下：

![Jenkins页面4](/images/posts/jenkins/jenkins_ui4.png "Jenkins页面4")

插件安装完成后，会跳出创建第一个管理员用户页面，可以跳过此步骤，直接点击`使用admin账号继续`按钮，
每次登陆使用`/var/lib/jenkins/secrets/initialAdminPassword`文件的初始密码作为admin继续，
此处，我们还是选择填写信息创建新的管理员用户。Jenkins默认使用HTTP协议，数据传输不安全，后续可以
考虑[使用Nginx反向代理为其配置HTTPS认证](https://we.wewelove.cn/Ubuntu/jenkinsHttps)。

![Jenkins页面5](/images/posts/jenkins/jenkins_ui5.png "Jenkins页面5")

管理员用户生成后，就进入实例配置页面，可以选择`现在不要`，考虑到后续会与Github联动，都需要用到，
此处我们选择`保存并完成`：

![Jenkins页面6](/images/posts/jenkins/jenkins_ui6.png "Jenkins页面6")

到此，Jenkins就算正式准备好了。

![Jenkins页面7](/images/posts/jenkins/jenkins_ui7.png "Jenkins页面7")

点击`开始使用Jenkins`按钮即可顺利访问Jenkins仪表板：

![Jenkins页面8](/images/posts/jenkins/jenkins_ui8.png "Jenkins页面8")

注：上述防火墙已经开启8090端口的访问权限，若使用的是云服务提供商（如：阿里云，华为云）的云虚拟机ECS，
仅仅开启防火墙的8090端口权限是不足的，还需要在云平台上手动添加安全组的入访问规则，否则，在浏览器访问
Jenkins时会报如下错：

```
Unable to round-trip http request to upstream: dial tcp yourIP:8090: i/o timeout
```

云平台设置安全组规则操作如下：

![安全组设置](/images/posts/jenkins/rule_port.png "安全组设置")
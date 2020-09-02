---
layout: post
title: Github/Gitee + Jenkins 实现自动化CI构建
categories: Jenkins
keywords: Jenkins Github Gitee  CI Automation
permalink: /Ubuntu/jenkinsCI
---


传统团队开发场景中，开发人员每次提交代码需要编译构建时，都需要在自己的环境中手动同步最新代码并进行编译，
这是十分繁琐且耗时的事。因此，越来越多的公司开始采用Github/Gitee + Jenkins 协作架构实现自动化CI构
建，其中，Github/Gitee作为代码仓库托管项目代码，Jenkins服务器用于项目任务自动化构建，完整流程可以描
述为：当开发者往Github/Gitee代码仓合入代码时，通过webhook机制会自动触发Jenkins从Github/Gitee代
码仓拉取最新的项目代码，然后进行编译构建。


**目录**

* TOC
{:toc}

### 概述

### 基础环境准备
1. 一台可联网的 `Ubuntu 16.04` 虚拟机，有独立的公网IP，搭建Jenkins服务器环境
2. 已有GitHUb/Gitee账号，并在上面创建一个使用maven构建的spring web项目工程代码仓
3. Jenkins服务器实现安装好openjDK 8，maven，git和mysql数据库等软件。(可根据自己的项目情况操作，若项目不涉及数据库操作，可不安装MySQL)

Jenkins服务器环境搭建参见 [基于Ubuntu 16.04部署Jenkins](https://we.wewelove.cn/Ubuntu/jenkinsDeploy) 一文；

使用IntelliJ IDEA 新建（New）一个maven项目，并使用Spring boot编写简单的web项目，
与数据库交付，实现用户的注册和登陆功能。具体项目代码参见【jenkinsDemo示例代码】(https://github.com/hellowaywewe/jenkinsDemo)
这里不详细展开，后面有时间再专门写一篇博客介绍如何使用IntelliJ构建web项目。

使用脚本完成OpenjDK 8，maven和git安装，此处使用 `vim env_prep.sh` 写一个脚本env_prep.sh，内容如下：

```
#!/bin/bash

set -e

# 使用apt工具安装OpenjDK 8 和 git
apt install -y openjdk-8-jdk git

# 配置 OpenjDK 8 环境
java_path=`ls /usr/lib/jvm/ | grep java-1.8.*`
cat >> /etc/profile <<EOF
export JAVA_HOME=/usr/lib/jvm/$java_path
export JRE_HOME=\${JAVA_HOME}/jre
export CLASSPATH=.:\${JAVA_HOME}/lib:\${JRE_HOME}/lib
export PATH=\${JAVA_HOME}/bin:\$PATH
EOF

# 安装 Maven 3.6.3
wget http://mirror.bit.edu.cn/apache/maven/maven-3/3.6.3/binaries/apache-maven-3.6.3-bin.tar.gz
mkdir -p /usr/local/maven && tar -zxvf apache-maven-3.6.3-bin.tar.gz -C /usr/local/maven --strip-components 1
rm apache-maven-3.6.3-bin.tar.gz

# 配置 Maven 环境
cat >>/etc/profile<<EOF
export M2_HOME=/usr/local/maven
export CLASSPATH=\$CLASSPATH:\$M2_HOME/lib
export PATH=\$PATH:\$M2_HOME/bin
EOF

# 使 OpenjDK 和 maven 配置生效
source /etc/profile

# 安装 MySQL 8.0.17及其相关依赖库
apt install -y libaio1 libmecab2 libjson-perl mecab-ipadic-utf8 mecab-utils mecab-ipadic
wget https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-server_8.0.17-1ubuntu16.04_amd64.deb-bundle.tar
mkdir -p mysql && tar -xvf mysql-server_8.0.17-1ubuntu16.04_amd64.deb-bundle.tar -C mysql
dpkg -i mysql/*.deb
rm -rf mysql && rm -f *.tar

# 配置 MySQL 环境
cat >> /etc/mysql/mysql.conf.d/mysqld.cnf <<EOF
bind_address          = 0.0.0.0
EOF

# 重启 MySQL 服务
service mysql restart
```

执行 `chomd +x env_prep.sh` 授予脚本可执行权限，然后直接在系统中以root用户执行 `./env_prep.sh` 命令即可。

注：
- 可通过 `java --version` 查看OpenJDK安装版本，即可知晓其是否成功安装
- 可通过 `mvn -v` 查看Maven安装版本，即可知晓其是否成功安装
- 可通过 `service mysql status` 查看MySQL的状态，显示active即为成功启动
- 创建用于jenkinsDemo网站工程的数据库和数据表

```
# 访问数据库
mysql -uroot -pyourPassword
# 创建jenkins_db数据库
CREATE DATABASE IF NOT EXISTS `jenkins_db`;

# 进入jenkins_db数据库
use jenkins_db;

# 创建user数据表
CREATE TABLE IF NOT EXISTS `user` (
  `user_id` int(11) NOT NULL AUTO_INCREMENT,
  `user_name` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
  `user_pwd` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
  PRIMARY KEY (`user_id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 3 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;
```

### Github 操作步骤
##### Github web 项目准备
在Github中创建jenkinsDemo项目仓，实现用户登陆/注册功能，后台接收前端页面的注册/登陆请求，
会与服务器数据库进行交互，非常简单的示例，仅用于测试。

当用户往jenkinsDemo仓合入代码时，github webhook会触发Jenkins启动构建任务，编译jenkinsDemo项目。

jenkinsDemo项目结构如下：

![jenkinsDemo项目结构](/images/posts/jenkins/demo_struceture.png "jenkinsDemo项目结构")

展示的页面效果如下：

![jenkinsDemo页面效果1](/images/posts/jenkins/demo_page1.jpg "jenkinsDemo页面效果1")

![jenkinsDemo页面效果2](/images/posts/jenkins/demo_page2.jpg "jenkinsDemo页面效果2")

![jenkinsDemo页面效果3](/images/posts/jenkins/demo_page3.jpg "jenkinsDemo页面效果3")

##### 在 Github 上 配置 Webhook 地址
1. 在浏览器中登陆github账号,打开待设置Webhook的github项目，此为jenkinsDemo
2. 然后点击工程主页面右上方的`setting`->再点击`Webhooks`->最后点击`Add webhook`,
如下页面显示：

![配置 Webhook 地址1](/images/posts/jenkins/demo_webhook1.png "配置 Webhook 地址1")

3. 在页面"Payload URL"处填写webhook地址->再点击底部的"Add webhook按钮"即完成webhook配置

![配置 Webhook 地址2](/images/posts/jenkins/demo_webhook2.png "配置 Webhook 地址2")

注：
- webhook地址即为Jenkins网站站点地址（服务器域名/IP + 端口）+ /github-webhook请求字段
- 一旦jenkinsDemo有代码提交，GitHub就会向此webhook地址发送请求，通知Jenkins构建任务

##### 在 Github 上生成 Personal access tokens

Jenkins访问GitHub工程项目时，有些操作需要用户授权才可以成功执行，因此，我们需要在GitHub上生成用户
授权访问的Personal access tokens，并将其授予Jenkins使用，生成步骤如下：

1. 登录GitHub，点击右上方用户头像图标，进入"Settings"页面，如下图所示：

![配置token1](/images/posts/jenkins/demo_token1.png "配置token1")

2. 接着点击左下角的"Developer settings"，如下图所示：

![配置token2](/images/posts/jenkins/demo_token2.png "配置token2")

3. 跳转至"Developer settings"页面，然后点击右上角的"Generate new token"按钮，如下图所示：

![配置token3](/images/posts/jenkins/demo_token3.png "配置token3")

4. 填写note说明，再勾选"repo"和"admin:repo_hook"，最后点击底部的"Generate token"按钮即可
产生新的access token具体设置如下图：

![配置token4](/images/posts/jenkins/demo_token4.png "配置token4")

该生成的token后面会用于Jenkins，生成的token如下图：

![配置token5](/images/posts/jenkins/demo_token5.png "配置token5")

注：务必记得将此token字符串拷贝备份，否则页面刷新后token会找不见，刷新页面如下。

![配置token6](/images/posts/jenkins/demo_token6.png "配置token6")

### Jenkins 操作步骤

##### 登录Jenkins主页
进入 [Jenkins站点主页](http://jenkins.wewelove.cn:8090/) ，若未登录会先跳转到登录界面，
输入管理员账号密码即可

![Jenkins主页](/images/posts/jenkins/jenkins_home.png "Jenkins主页")

##### Jenkins系统配置
1. 点击"系统管理->系统设置"，进入系统配置页面：

![Jenkins系统配置1](/images/posts/jenkins/jenkins_sysconf1.png "Jenkins系统配置1")

2. 找到"GitHub"，配置GitHub服务器，"API URL"填写"https://api.github.com”，“凭据"点击
"添加"下拉框，然后选中"Jenkins"选项，如下所示：

![Jenkins系统配置2](/images/posts/jenkins/jenkins_sysconf2.png "Jenkins系统配置2")

选中"Jenkins"选项后会弹出"Jenkins 凭据提供者"窗口，然后“类型"选择"Secret text”，"Secret"即为
上述在GitHub上生成的Personal access tokens，描述可随意填写，然后点击"添加"按钮，如下所示：

![Jenkins系统配置3](/images/posts/jenkins/jenkins_sysconf3.png "Jenkins系统配置3")

点击"添加"按钮后会跳转回GitHub服务器配置处，然后"凭据"选择前面成功添加的凭据，点击"连接测试"，测试
成功后勾选"管理Hook"，最后点击"保存"按钮，如下所示：

![Jenkins系统配置4](/images/posts/jenkins/jenkins_sysconf4.png "Jenkins系统配置4")

若还需要继续配置其他系统项，可点击"应用"按钮，最后完成全部配置再点击"保存"按钮。

### 新建Jenkins任务
由于jenkinnsDemo是使用maven构建的web项目，因此我们在jenkins里新建的任务需要是maven项目的，但
我们在新建任务时没有"新建一个maven项目"选项，所以我们在新建任务时需要先安装maven插件，

![maven插件配置1](/images/posts/jenkins/maven1.jpg "maven插件配置1")

##### 在Jenkins中安装maven插件

1. 点击"系统管理->插件管理"，如下所示：

![插件管理1](/images/posts/jenkins/jenkins_plugin1.png "插件管理1")

2. 进入插件管理页面，点击"可选插件"选项，然后在搜索栏输入"maven"开始搜索，在搜索结果中勾
选"Maven Integration"，然后点击"直接安装"按钮，如下所示：

![插件管理2](/images/posts/jenkins/jenkins_plugin2.png "插件管理2")

3. 等待插件安装完成

![插件管理3](/images/posts/jenkins/jenkins_plugin3.png "插件管理3")

##### 在Jenkins上完成Maven，JDK等工具配置

由于jenkinnsDemo任务构建完成后，需要在Jenkins服务器上运行，我们已经在该服务器上安装有Maven，JDK等
软件，现在只需在Jenkins上为对应软件工具指定安装目录（对应服务器上OpenJDK的安装目录）。

1. 首先，点击"系统管理->全局工具配置"，如下所示：

![全局工具配置1](/images/posts/jenkins/jenkins_tool1.png "全局工具配置")

2. 进入全局工具配置页面，找到"JDK"， 点击"JDK安装..."

![全局工具配置2](/images/posts/jenkins/jenkins_tool2.png "全局工具配置2")

3. 点击"新增JDK"

![全局工具配置3](/images/posts/jenkins/jenkins_tool3.png "全局工具配置3")

4. 不勾选"自动安装"选项，"JAVA_HOME"填入先前在Jenkins服务器中安装OpenJDK 8的安装路径, 然后
点击"应用"按钮，如下所示：

![全局工具配置4](/images/posts/jenkins/jenkins_tool4.png "全局工具配置4")

6. 找到"Maven"， 点击"Maven安装..."

![全局工具配置5](/images/posts/jenkins/jenkins_tool5.png "全局工具配置5")

7. 点击"新增Maven"

![全局工具配置6](/images/posts/jenkins/jenkins_tool6.png "全局工具配置6")

4. 不勾选"自动安装"选项，"MAVEN_HOME"填入先前在Jenkins服务器中安装maven 3.6.3的安装路径, 
然后点击"保存"按钮，如下所示：

![全局工具配置7](/images/posts/jenkins/jenkins_tool7.jpg "全局工具配置7")

##### 在Jenkins上构建新任务

首先需要新建jenkins任务，此处命名为test_jenkinsDemo,然后需要配置项目，主要分为以下几个部分配置：
在本次任务中主要配置"源码管理"，"构建触发器"，"构建环境"等

![构建任务0](/images/posts/jenkins/jenkins_task0.jpeg "构建任务0")

1. 新建任务

在Jenkins主页选择"新建任务"，进入如下页面：

![构建任务1](/images/posts/jenkins/jenkins_task1.png "构建任务1")

名称可以随意定义，然后选择"构建一个maven构建项目"，点击"确定"按钮,进入任务配置页面

2. 源码管理配置

所谓"源码管理"配置其实就是为构建的任务设置源码所在的路径,帮助Jenkins任务获取源代码，后面一旦源码仓
状态变化，就会触发任务轮询源码编译，如果不想设置或者后面有脚本可以自主管理可以选择none，具体步骤如下：

点击"源码管理"，接着选中"Git"选项，"Repository URL"填写项目仓地址（此处为jenkinsDemo的
github地址，可通过下图1复制过来，记得选https的地址），然后"Credentials"点击"添加"，如下图2所
示, 在弹出窗口（如下图3所示），"类型"选择"Username with password"，Username输入GitHub账号，
Password输入GitHub密码，描述可随意写，点击"添加"按钮。

![构建任务2](/images/posts/jenkins/jenkins_task2.png "构建任务2")

![构建任务3](/images/posts/jenkins/jenkins_task3.png "构建任务3")

![构建任务4](/images/posts/jenkins/jenkins_task4.png "构建任务4")

成功添加Credentials后，继续往下配置，"Credentials"选择刚添加的的Credentials，然后“源码库浏览器"
选择"githubweb”，"URL"输入项目主页：https://github.com/yourGithubAccount/jenkinsDemo，如
下所示：

![构建任务5](/images/posts/jenkins/jenkins_task5.png "构建任务5")

3. 构建触发器配置

"构建触发器"配置也比较好理解，其实就是配置任务构建的触发时机，任务何时会被触发，什么条件会触发等等，
具体步骤如下：

“构建触发器"中勾选"GitHub hook trigger for GiTScm polling”，该选项说明webhook插件检测到源
码有push操作就会触发构建，如下所示：

![构建任务6](/images/posts/jenkins/jenkins_task6.png "构建任务6")

4. 构建环境配置
勾选"Use secret text(s) or file(s)"，勾选后会在下方出现"绑定"，然后"凭据"选择我们先前配置过
的"github personal access tokens"，然后点击"保存"，如下所示：

![构建任务7](/images/posts/jenkins/jenkins_task7.png "构建任务7")

### 向Github项目仓提交代码触发Jenkins任务构建

1. 修改jenkinsDemo项目代码，并提交到Github jenkinsDemo仓,提交历史可通过`git log`指令查看：

![git log](/images/posts/jenkins/git_log.jpg "git log")

可见我们push了一段代码到Github jenkinsDemo仓，该代码修改commit说明为"modify pom.xml error"，
我们来看看这段代码的合入是否触发了Jenkins构建任务的编译执行。

2. 我们返回Jenkins主页面，可以看见我们之前构建的test_jenkinsDemo任务正在执行，如下所示：

![jenkins result 1](/images/posts/jenkins/jenkins_result1.png "jenkins result 1")

点击"test_jenkinsDemo"进入任务页面，然后点击"工作空间"，可看见工程项目编译后生成.jar文件，如下
所示：

![jenkins result 2](/images/posts/jenkins/jenkins_result2.png "jenkins result 2")

3. 返回上一页，仍然进入test_jenkinsDemo任务页面，然后点击最后一次构建，如下所示：

![jenkins result 3](/images/posts/jenkins/jenkins_result3.png "jenkins result 3")

在最后一次构建页面点击"控制台输出"，如下所示：

![jenkins log 1](/images/posts/jenkins/jenkins_log1.png "jenkins log 1")

输出内容中的"Commit message"与git log中提交的commit内容一致，说明是"modify pom.xml error"
这个commit里合入的代码触发了Jenkins任务编译执行。

![jenkins log 2](/images/posts/jenkins/jenkins_log2.png "jenkins log 2")

观察上图可见，任务编译成功。



























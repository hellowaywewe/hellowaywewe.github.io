---
layout: post
title: MindSpore Lite端侧图像分类应用安装部署及运行
categories: MindSpore
keywords: MindSpore Lite APP APK
permalink: /Ubuntu/liteAppBuild
---

Android Studio是用于安卓开发和调试的工具，在其上开发端测应用代码，然后基于Gradle自动化构建应用并运行。

**目录**

* TOC
{:toc}

## 本机系统信息

System | Specifications
:-: | :-:
Memory | 16G
OS | Mac OS 10.15.5
Software | Android Studio 4.1 <br /> JDK: 1.8.0

本文基于MAC电脑安装配置开发环境，若采用windows操作系统，大体思路不变，具体操作有所区别。

## 安装前准备

#### 事先安装并配置好JDK 1.8 环境

JDK(全称：Java Development Kit) 是 Java 语言的软件开发工具包(SDK)，MindSpore Lite APP 需要使用 JDK 编译 JAVA 源码, JDK 符合 1.8 版本即可。

1. 由于 JDK 的下载需要 Oracle 帐户，因此，我们在下载前需要先[注册](https://profile.oracle.com/myprofile/account/create-account.jspx)一个 Oracle 账户；
2. 然后打开[官网地址](https://www.oracle.com/java/technologies/javase/javase-jdk8-downloads.html)登录下载JDK安装包，可能访问会有些慢，若发现下载失败可多试几次，或打开VPN下载；
3. 直接双击 JDK 安装包，安装是图形界面一步步操作的，比较简单，网上教程也比较多，就不详细说了； 
4. 安装好JDK后，需要配置JDK环境变量

```shell
# 在mac命令终端打开并编辑~/.zshrc文件
vim ~/.zshrc

# 在~/.zshrc文件中添加如下内容，保存即可。其中JAVA_HOME为JDK的安装目录
export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_251.jdk/Contents/Home
export JRE_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_251.jdk/Contents/Home/jre
export PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin

# 使配置生效
source ~/.zshrc

# 输入java -version命令查看JDK是否配置成功，若输出如下信息，即为配置成功
java version "1.8.0_251"
Java(TM) SE Runtime Environment (build 1.8.0_251-b08)
Java HotSpot(TM) 64-Bit Server VM (build 25.251-b08, mixed mode)
```

JDK版本信息如下图所示：

![JDK版本信息](/images/posts/mindspore/lite/java_version.png "JDK版本信息")

#### 事先安装好Android Studio（推荐4.0版本以上）

Android Studio 是一个Android集成开发工具，用于安卓应用开发和调试。

1. 首先下载安装文件，由于服务器不在国内，[官方下载地址](https://developer.android.com/studio/)可能访问有些慢，可打开VPN下载，若没有VPN,可访问 [Android Studio 中文社区](http://www.android-studio.org/)下载；
2. 完成Android Studio安装及初始化操作

###### Android Studio安装

直接双击.zip安装压缩包（如下图所示），将其解压生成 `Android Studio.app`

![Android Studio安装1](/images/posts/mindspore/lite/studio_install1.jpeg "Android Studio安装1")

然后将 `Android Studio.app` 移动到 Mac 应用程序（ Application ）目录中，这样就可以在 Mac 启动台（ LaunchPad ）点击启动 Android Studio，如下所示：

![Android Studio安装2](/images/posts/mindspore/lite/studio_install2.jpeg "Android Studio安装2")

###### Android Studio初始化操作

打开Android Studio，如果是新安装用户会提示无法访问SDK，可以选择点击"Cancel"按钮跳过，后面再配置。

![Android Studio初始化1](/images/posts/mindspore/lite/studio_init1.png "Android Studio初始化1")

点击"Next"按钮，开始配置SDK:

![Android Studio初始化2](/images/posts/mindspore/lite/studio_init2.png "Android Studio初始化2")

默认安装Android 11，SDK版本为30，符合端侧应用"Android SDK >= 26"的要求，SDK存放目录可自定义，后面会用到，然后点击"Next"按钮：

![Android Studio初始化3](/images/posts/mindspore/lite/studio_init3.png "Android Studio初始化3")

点击"Finish"按钮，完成SDK配置。由下图可知，最好事先配置好JDK环境：

![Android Studio初始化4](/images/posts/mindspore/lite/studio_init4.png "Android Studio初始化4")

配置完成后，开始正式安装SDK：

![Android Studio初始化5](/images/posts/mindspore/lite/studio_init5.png "Android Studio初始化5")

点击"Finish"按钮，完成SDK安装。此次安装未开VPN，等待几分钟就完成了，这一步没安装成功也没关系，可在后续安装：

![Android Studio初始化6](/images/posts/mindspore/lite/studio_init6.png "Android Studio初始化6")

最终出现如下启动界面，即完成Android Studio初始化：

![Android Studio启动界面](/images/posts/mindspore/lite/android_studio_start.png "Android Studio启动界面")

## 正式配置 MindSpore Lite 端侧应用

#### 下载并打开 MindSpore Lite 源码

由于 MindSpore Lite APP（ 名为：image_classification ） 源码存放于MindSpore代码仓（ mindspore/model_zoo/official/lite/ ）中，因此，我们可以先使用 `git clone`命令下载 mindspore 项目

- 可访问 Gitee 或 Github 下载 mindspore项目，建议选择 Gitee，速度会快一些

```shell
# 从Gitee仓地址下载
git clone https://gitee.com/mindspore/mindspore.git

# 从Github仓地址下载
git clone https://github.com/mindspore-ai/mindspore.git
```

#### 在Android Studio里配置 MindSpore lite 端测项目

###### 打开端侧图像分类源码项目

在 Android Studio 启动窗口，选择 "Open an Existing Project" 打开已有的项目，如下所示：

![项目配置1](/images/posts/mindspore/lite/android_studio_op1.png "项目配置1")

在弹出框选择 "image_classification" 项目，点击 "Open" 按钮：

![项目配置2](/images/posts/mindspore/lite/android_studio_op2.png "项目配置2")

###### SDK Tools（如NDK，Cmake）安装

MindSpore Lite 端侧图像分类应用 "image_classification" 要求运行环境满足：
1. NDK 版本为 21.3
2. CMake 3.10.2

因此，我们需要在Android Studio安装对应版本NDK和CMake工具。

在 Android Studio 面板中，选择 "Preferences"，如下所示：

![SDK Tools安装1](/images/posts/mindspore/lite/sdk_conf1.png "SDK Tools安装1")

然后点击 "Appearance & Behavior"->"System Settings"->"Android SDK"，可见先前安装好的 Android 11, 支持 SDK 30。若需要安装个人所需的 Android platform 和 SDK，都可在此进行。"Android SDK Location" 即为先前初始化时为其配置的安装路径

![SDK Tools安装2](/images/posts/mindspore/lite/sdk_conf2.png "SDK Tools安装2")

点击 "SDK Tools"，选中 "NDK" 和 "Cmake" 选项，如下所示：

![SDK Tools安装3.1](/images/posts/mindspore/lite/sdk_conf3.png "SDK Tools安装3.1")

勾选右下角的 "Show Package Detail" 可查看对应的版本信息，勾选符合要求的安装版本，如下所示：

![SDK Tools安装3.2](/images/posts/mindspore/lite/sdk_conf3.jpeg "SDK Tools安装3.2")

点击 "OK"，弹出确认更改窗口：

![SDK Tools安装4](/images/posts/mindspore/lite/sdk_conf4.png "SDK Tools安装4")

点击 "OK"，弹出 License 协议窗口，选择 "Accept" 选项，点击 "Next" 按钮，开始安装 NDK 和 Cmake 等工具：

![SDK Tools安装5](/images/posts/mindspore/lite/sdk_conf5.png "SDK Tools安装5")

安装需等待几分钟，完成后点击 "FInish" 即可。

![SDK Tools安装6](/images/posts/mindspore/lite/sdk_conf6.png "SDK Tools安装6")

###### 配置 SDK，NDK 和 JDK 路径

在 Android Studio 面板中，点击 "File"->"Project Structure..."，如下所示：

![NDK配置1](/images/posts/mindspore/lite/ndk_conf1.png "NDK配置1")

点击 "SDK Location", "Android SDK Location" 为先前配置的SDK安装目录，"Android NDK Location" 为SDK安装目录的 ndk 目录，通过下拉选项选择：

![NDK配置2](/images/posts/mindspore/lite/ndk_conf2.png "NDK配置2")

"JDK Location" 为先前配置的 JDK 的安装目录，通过下拉框选择，点击 "OK" 完成配置：

![NDK配置3](/images/posts/mindspore/lite/ndk_conf2.png "NDK配置3")

###### 基于 Gradle 自动化构建 MindSpore lite 端侧应用

Gradle 是项目自动化构建工具，类似 Maven，却又不像 Maven 那样需要维护繁琐的 xml 配置，可帮助我们自动化构建 Android App。

在 Android Studio 面板中，点击 "File"->"Sync Project with Gradle Files"，如下所示：

![Gradle构建1](/images/posts/mindspore/lite/app_sync1.png "Gradle构建1")

开始自动化构建，Gradle 分多 task 执行构建任务，如下图所示：

![Gradle构建2](/images/posts/mindspore/lite/app_sync2.png "Gradle构建2")

编译需要下载许多依赖库文件等，会花费较多时间，取决于各位的网速和 RP，需要点耐心，建议可以趁此空隙喝杯咖啡，构建完成状态如下：

![Gradle构建3](/images/posts/mindspore/lite/app_sync3.png "Gradle构建3")

## 编译/运行 MindSpore lite 端侧应用

通常情况下，Android 应用的运行有三种形式：

1. 使用 AVD Manager 创建一个安卓模拟设备，在模拟出的安卓手机中运行应用。（受限于平台 AVD 对 ARM 的支持，暂不推荐使用该方法）
2. 编译生成 .apk 应用安装文件，将其传入安卓手机中即可安装执行。（简单高效，用户仅需下载该 apk 文件到自己的安卓手机中安装运行即可）
3. 用数据线直接连接真实的安卓手机运行应用。（手机需开启"USB调试"模式，配置较繁琐，一旦手机被电脑Android Studio识别，运行过程也算简单高效，方便调试）

下面就来说下后 2 种形式的具体的操作流程，首先说下如何生成 .apk 安装文件

在 Android Studio 面板中，点击 "Build"，浮动箭头指向 "Build Bundle(s) / APK(s)" 选项的右三角形处，然后点击 "Build APK(s)"，如下所示：

![apk生成1](/images/posts/mindspore/lite/app_build1.png "apk生成1")

bulid 过程中，download.gradle 文件会自动下载 libmindspore-lite.so 等库文件和 mobilenetv2.ms 模型文件到 app/libs/arm64-v8a 目录和 app/src/main/assets/model 目录下。

![apk生成2](/images/posts/mindspore/lite/app_build2.png "apk生成2")

编译成功会生成 app-debug.apk 文件，存放于 app/build/outputs/apk/debug 目录下，如下图所示：

![apk生成3](/images/posts/mindspore/lite/app_build3.png "apk生成3")

<a href="/images/posts/mindspore/lite/app-debug.apk" target="_blank">MindSpore lite端侧图片分类应用APK安装文件下载地址</a>，有需要的童鞋自取。

接下来，我们来看下如何直接连接真机运行应用，这里选用华为Meta 30 Pro安卓机。

通过数据线（意外发现苹果macbook的充电线可用）连接华为Meta 30 Pro安卓机和MacBook Pro笔记本电脑，可以不要求在同一网段，开启手机的"USB调试"模式，具体操作如下：

- 打开手机的"设置"->"关于手机"，然后连续点击6-8次"版本号"，输入锁屏密码，进入开发者模式；
- 点击手机的"系统和更新"（在"关于手机"上一行）->"开发人员选项"，打开"保持唤醒状态"，" USB调试"和"仅充电模式下允许 ADB调试"开关，然后点击"选择USB配置"选项，在弹出选项中选择"MTP（多媒体传输）"，开启USB调试；
- 此时手机桌面会弹出是否允许usb调试的弹窗，选择"是"即可；
- 成功连接后，华为手机就被添加到Android设备里了，点击运行按钮。

![app run](/images/posts/mindspore/lite/app_run.png "app run")

可查看视频演示，最终可看见 MindSpore lite 端测应用在手机中启动运行，通过摄像头检测到目标对象并为其分类：

<video src="/images/posts/mindspore/lite/phone_usb.mp4" width="800px" height="600px" controls="controls"></video>



















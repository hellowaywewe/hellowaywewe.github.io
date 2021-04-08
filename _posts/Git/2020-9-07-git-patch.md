---
layout: post
title: 使用git命令生成patch和打patch
categories: Git
keywords: Git Patch
permalink: /gitPatch
---


老是听身边的同事提到生成patch，打patch，但一直对什么是patch，用在什么场景中一知半解，最近正好在
做开源合规的整改工作，正好借此机会学习下。可以说，在日常的协同开发中，生成patch和打patch时常会用到。


**目录**

* TOC
{:toc}

在我们的工作中存在这样一种情况：在开源的项目中需要依赖第三方开源软件，根据自身项目的需求，
需对该第三方开源软件进行源码修改后再使用，修改的代码可不合入到远程主仓。在此种情况下，修改的代码
可通过生成patch和打patch完成。除了上述提到的开源软件依赖的场景，平时我们在工作中也比较经常会遇到
需要使用patch的场景，比如：我们使用git管理项目时，当与合作伙伴开发项目时，可以让他们使用git命令
将他们开发的代码生成patch给到我们，此时我们只需将该patch打入到我们的项目中即可。

下面我们就来聊一聊：什么是patch?如何生成patch和打patch?
通俗来说，patch中存储的内容其实就是你对代码的修改，生成patch其实就是将你对代码的修改记录并保存到patch
文件中，而打patch实际就是应用patch，将patch文件中对代码的修改，应用到项目源代码中。

这里默认大家熟悉使用git操作命令，使用git format-patch命令生成patch，使用git am命令打patch。

### 事先准备好源码项目
事先在Github平台上准备好待开发的项目源码，此处仅用于实践测试之用，因此简单创建一个Java项目即可，我已经事
先准备好，只需要下载到本地即可。
```
git clone https://github.com/hellowaywewe/fittest.git
```

首先我们通过git log命令查看commit提交记录

在远程主仓src/main/java源码目录中已有PersonA和PersonB,假设这并不符合我们的项目需求，
我们需要在在项目src/main/java源码目录下新增一个名为c的包，在该包下新建PersonC.java文件，
新增的代码文件，不需要合入远程主仓，只需要在本地主仓提交，并将该本地commit打成patch,提供给
其他人使用，PersonC.java文件代码如下：
```
package c;

public class PersonC {
    private String name;
    private int age;

    public PersonC(String name){
        this.name = name;
    }

    public PersonC(String name, int age){
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }
}
```
然后修改src/main/java/SayHello.java文件，在原有代码基础上新增如下代码：
```
import c.PersonC；
# 在main函数中添加
PersonC personC = new PersonC("C");
System.out.println("Hello, I am " + personC.getName());
```

在本地主仓提交代码,并不需要push到远程主仓。
```
git add ./
git commit -m 'add PersonC'

# 查看本地仓库提交记录，发现commit 85f9d0e6b即为我们新增PersonC的提交记录
git log
```
如下图所示：

![本地仓提交](/images/posts/git/git_patch1.jpeg "本地仓提交")


### 生成patch
使用git format-patch生成patch,常用操作如下：
```
# 生成最近的1次commit的patch
git format-patch HEAD^
# 生成最近的2次commit的patch
git format-patch HEAD^^
# 生成最近的3次commit的patch，依次类推，加^符号
git format-patch HEAD^^^
# 生成两个commit间的修改的patch（包含两个commit. <r1>和<r2>都是具体的commit号)
git format-patch <r1>..<r2>
# 生成单个commit的patch
git format-patch -1 <r1>
# 生成除commit之外后面修改的所有patch（不包含该commit）         
git format-patch <r1>
# 生成从根到r1提交的所有patch
git format-patch --root <r1>
```
因为add PersonC的提交记录是我们最新提交的，所以此处我们选用如下操作生成patch
```
git format-patch HEAD^
```
如下图所示：

![生成patch](/images/posts/git/git_patch2.png "生成patch")

### 打patch
patch生成后，接下来就是其他人如何将patch应用到他们本地的fittest项目中，也就是这里说的打patch。
git使用git am来执行打patch的操作,在使用git am之前，需要先`git am --abort` 一次，来放弃掉以前的
am信息，这样才可以进行一次全新的am，不然可能会报错。
git am 可以一次合并一个patch文件，或者一个目录下所有的patch(如：`git am mypatch/*.patch`)，
此处我们就新增了一个commit,生成一个patch,因此打一个patch文件即可，如下所示：
```
# 事先将生成的patch文件拷贝到个人的目录下（如：mypatch）
# 然后重新下载下远程主仓代码，该项目src/main/java目录下并没有c包，也没有PersonC类
# 为下载的主仓代码打patch，在新下载的项目中执行
git am ../mypatch/0001-add-PersonC.patch
```
如下图所示：

![打patch](/images/posts/git/git_patch3.png "打patch")

此时会发现src/main/java目录下存在c包，也有PersonC类了，确实是将我们前面add PersonC的提交记录生成
的patch应用进新下载的项目了。

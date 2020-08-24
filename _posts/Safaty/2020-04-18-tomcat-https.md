---
layout: post
title: 为Tomcat服务器配置https认证
categories: HTTPS
keywords: Https Tomcat Certbot Let's Encrypt 
permalink: /Ubuntu/tomcatHttps
---

为保证端到端数据传输安全，确保网站的真实性。近两年，Google、Baidu 等国内外互联网巨头都相继开始推
行启用 HTTPS 协议，Google 更是为了全球网站的 HTTPS 安全化，优化搜索引擎算法，让采用 HTTPS 的
网站在搜索排名中更靠前。下面我们就来了解下，如何使用 Let's Encrypt为 Tomcat 网站申请HTTPS安全
认证。


**目录**

* TOC
{:toc}

### HTTPS基本认识
HTTPS（全称：Hyper Text Transfer Protocol over Secure Socket Layer）是在HTTP的基础上加
入 SSL 安全套接层，通俗来说就是安全版 HTTP，它通过建立一个信息安全通道对传输的内容进行加密，确认网
站的真实性。

如果网站想使用 HTTPS 服务，则需要从信任的证书授权中心（CA，Certificate Authority ）为网站域名申
请 SSL 安全证书，并且将它部署到网站服务器上。一旦部署成功，网址的URL前缀会从http变成https，用户在访
问你的网站时，浏览器会在显示的网址前加一把小锁，表明访问的这个网站是安全的。
![HTTPS URL地址](/images/posts/safety/https_url.png "HTTPS URL地址")

### HTTPS认证申请

Let's Encrypt 是一个由非盈利组织提供的公共且免费 SSL 证书颁发机构 CA，其使用 certbot 官方推荐工
具为网站提供免费的 HTTPS 认证，支持和兼容 Chrome，FireFox 在内的主流浏览器。

目前 Let's Encrypt 免费 SSL 证书默认是90天有效期，但是我们可以到期自动续约（30天）或在即将到期前
为网站域名再重新申请一份为期90天的证书，重新申请不会影响当前网站所使用证书的安全性，每份证书使用90天，
到期/提前替换都可。
![HTTPS有效期](/images/posts/safety/https_time.jpg "HTTPS有效期")

接下来，我们就来一步步为网站域名申请HTTPS认证。注意，前提是已有可用的域名（可在GoDaddy或公有云提供商
处购买，如：阿里云，华为云），此处我使用的是阿里云。

##### 获取certbot-auto工具
下载certbot-auto，并将其设置可执行文件
```shell
wget https://dl.eff.org/certbot-auto
chmod a+x certbot-auto
```
![下载certbot-auto](/images/posts/safety/ssl_wget.png "下载certbot-auto")

![certbot-auto授权](/images/posts/safety/ssl_chmod.png "certbot-auto授权")

##### 生成HTTPS证书
可以使用多个`-d`参数为多个域名申请证书，也可以一次性使用`-d *.wewelove.cn`为你的多个二级/一级域名申
请认证，也可以使用`--email yourEmailAddr`设置你的个人邮箱接收认证信息，如：过期提醒。
```shell
./certbot-auto certonly -d we.wewelove.cn --manual --preferred-challenges dns --server https://acme-v02.api.letsencrypt.org/directory
```
输入`y`，点击确认

![生成HTTPS证书操作1](/images/posts/safety/ssl_op1.png "生成HTTPS证书操作1")

生成DNS TXT record
![生成HTTPS证书操作2](/images/posts/safety/ssl_op2.png "生成HTTPS证书操作2")

将上图生成的DNS TXT record填入[阿里云DNS解析](https://dns.console.aliyun.com/?spm=a2c4g.11186623.2.13.494940c6rNTXp8#/dns/setting/wewelove.cn)

![生成HTTPS证书操作3](/images/posts/safety/ssl_op3.jpeg "生成HTTPS证书操作3")

在对应的域名（we.wewelove.cn）中，点击`添加记录`新增一条TXT类型的记录

![生成HTTPS证书操作4](/images/posts/safety/ssl_op4.jpeg "生成HTTPS证书操作4")

在阿里云DNS解析设置完后，回到命令终端，点击确认，显示如下信息即为HTTPS证书生成成功。

![生成HTTPS证书检查1](/images/posts/safety/ssl_check1.png "生成HTTPS证书检查1")

输入`ls /etc/letsencrypt/live/yourDomain/`命令，在你的域名目录下确认生成4个.pem证书文件：
- cert.pem: 服务器端证书公钥
- chain.pem: 浏览器需要的所有证书但不包括服务端证书，比如根证书和中间证书
- fullchain.pem: 包括了cert.pem和chain.pem的内容
- privkey.pem: 证书的私钥

![生成HTTPS证书检查2](/images/posts/safety/ssl_check2.png "生成HTTPS证书检查2")

##### 证书续约
可设置crontab定时任务每月自动续约(有效期30天)证书
```shell
# 查看SSL证书的过期时间
./certbot-auto certificates

# 证书过期后执行才可生效，否则会提示证书未到续签期
./certbot-auto renew

# 对于提示未到续签期：cert not due for renewal，可以强制更新
./certbot-auto renew --force-renew
```
注：也可以选择重新生成一份新的证书，每次认证相互独立，互不影响，有效期都为90天，然后在服务器中
更换新生成的.pem密钥文件。

### 为Tomcat服务器配置HTTPS认证
将上述生成的privkey.pem，cert.pem和chain.pem三个文件拷贝到tomcat安装目录
（如:/usr/local/tomcat）下的conf文件夹下，然后配置Tomcat服务器（https默认使用443端口）
```shell
# 修改Tomcat服务器配置文件server.xml
cd /usr/local/tomcat/conf
vim server.xml

# 修改如下配置
<Connector port="80" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="443"/>
    <Connector port="443" protocol="org.apache.coyote.http11.Http11NioProtocol"
               maxThreads="150" SSLEnabled="true" >
        <SSLHostConfig>
            <Certificate certificateKeyFile="conf/privkey.pem"
                         certificateFile="conf/cert.pem"
                         certificateChainFile="conf/chain.pem"/>
        </SSLHostConfig>
    </Connector>
```
![Tomcat服务器配置](/images/posts/safety/https_conf.jpg "Tomcat服务器配置")


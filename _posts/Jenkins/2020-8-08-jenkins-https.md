---
layout: post
title: 使用Nginx反向代理为Jenkins配置HTTPS认证
categories: Jenkins
keywords: Jenkins HTTPS Nginx
permalink: /Ubuntu/jenkinsHttps
---


在生产环境中使用Jenkins，需要确保Jenkins站点访问的安全性，因此，本篇博文主要是在上篇博文
[基于Ubuntu 16.04部署Jenkins](https://we.wewelove.cn/Ubuntu/jenkinsDeploy)的
基础上，使用Nginx反向代理为事先搭建好的Jenkins站点配置HTTPS认证。


**目录**

* TOC
{:toc}

### 基础环境准备
1. 一台可联网的 `Ubuntu 16.04` 虚拟机，有独立的公网IP
2. 为Jenkins配置域名，可通过域名访问站点
3. 事先为域名申请好HTTPS安全认证证书
4. 全程使用 `root` 用户配置
5. 事先已安装好vim curl wget等基础软件工具。

针对第2点，可在云服务提供商（如：阿里云，阿里云）平台上的DNS解析服务中添加A类型映射记录（域名到IP的解析）:

![Jenkins域名解析](/images/posts/jenkins/jenkins_dns.jpg "Jenkins域名解析")

针对第3点，可参考[为Tomcat服务器配置https认证](https://we.wewelove.cn/Ubuntu/tomcatHttps)一文，使用Let’s Encrypt的
certbot-auto工具为Jenkins域名申请HTTPS认证，生成4个.pem证书文件:

![Jenkins认证证书](/images/posts/jenkins/jenkins_pem.jpeg "Jenkins认证证书")

针对第5点，若未安装好vim curl wget等基础软件，可通过以下命令安装：

```shell
apt-get update
apt-get -y install vim curl wget
```


### Nginx安装

```
# 安装指令
apt-get install nginx

# 启动服务
service nginx start

# 查看Nginx状态，显示active状态则启动成功
service nginx status
```

安装成功后，可以查看下属nginx文件位置：
- /usr/sbin/nginx：主程序存放位置
- /etc/nginx：配置文件的存放目录
- /usr/share/nginx：静态文件的存放目录
- /var/log/nginx：日志的存放目录
      
通过APT工具安装的的软件，会自动创建/etc/init.d/nginx服务脚本，可直接使用`service nginx { start | stop | restart | reload | force-reload | status | configtest | rotate | upgrade }`命令操作服务。
  

### 在Nginx中配置HTTPS认证

```
# 将cert.pem和privkey.pem两个文件拷贝到/etc/nginx/conf.d/目录下
cp /etc/letsencrypt/live/yourJenkinsDomain/cert.pem /etc/nginx/conf.d/
cp /etc/letsencrypt/live/yourJenkinsDomain/privkey.pem /etc/nginx/conf.d/

# 进入nginx配置目录
cd /etc/nginx/conf.d
# 新建ssl.conf文件
vim ssl.conf
# 在ssl.conf文件内增加如下内容，保存即可
server {
    listen 443;
    server_name yourJenkinsDomain; # yourJenkinsDomain改为已申请好认证证书的jenkins域名
    # 配置SSL
    ssl on;
    ssl_certificate /etc/nginx/conf.d/cert.pem; # 服务器公钥认证证书
    ssl_certificate_key /etc/nginx/conf.d/privkey.pem; # 私钥认证证书
    ssl_session_timeout 10m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4:!DH:!DHE;
    ssl_prefer_server_ciphers on;

    location / {
        proxy_pass http://localhost:8090; # 做端口转发
    }
    error_page 404 /404.html;
    error_page 500 502 503 504 /50x.html;
}
```

配置好HTTPS认证后，访问Jenkins站点加www和不加www是有很大区别的，存在跨域问题，因此我们还需要做重定向工作。
这样，不管用户访问哪个域名`http://www.yourJenkinsDomain`，`http://yourJenkinsDomain`，`https://www.yourJenkinsDomain`
`https://yourJenkinsDomain`都会跳转到`https://yourJenkinsDomain`

### 

```
# 继续编辑ssl.conf文件
/etc/nginx/conf.d/ssl.conf
# 在ssl.conf文件末尾追加如下内容，保存即可
server {
    listen 80;
    server_name www.yourJenkinsDomain;
    rewrite ^(.*)$ https://yourJenkinsDomain$1 permanent; 
}
server {
    listen 80;
    server_name yourJenkinsDomain;
    rewrite ^(.*)$ https://${server_name}$1 permanent; 
}
server {
    listen 443;
    server_name www.yourJenkinsDomain;
    rewrite ^(.*)$ https://yourJenkinsDomain$1 permanent; 
}
```

Nginx配置文件内容如下图：

![Nginx配置文件](/images/posts/jenkins/nginx_https_conf.png "Nginx配置文件")

### 重启Nginx服务

```
# 重加载nginx服务
service nginx reload

# 重启nginx服务
service nginx restart

# 查看Nginx状态，显示active状态则重启成功
service nginx status
```

![查看Nginx状态](/images/posts/jenkins/nginx_status.png "查看Nginx状态")


### 访问Jenkins站点验证HTTPS是否生效

在浏览器地址栏输入 `http:/yourJenkinsDomain` 和 `https:/yourJenkinsDomain`，效果如下：

![Jenkins认证页面](/images/posts/jenkins/jenkins_https_page.png "Jenkins认证页面")

观察上图可知，地址栏左边加了锁，表示https认证配置成功，可安全访问。

注意：打开防火墙允许80和443端口访问，打开云服务提供商的安全组入方向规则（80和443），才可正常访问。

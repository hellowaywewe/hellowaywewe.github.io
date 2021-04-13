---
layout: post
title: 集成Prometheus + Grafana 实现可视化监控系统
categories: Monitoring
keywords: Monitoring Prometheus Grafana
permalink: /Monitoring/basics
---

过去因为工作的关系，对Prometheus技术及社区做过洞察，原来以为监控系统再强大不还是监控系统，监控系统资源，
温度，内存那些指标吗？它还能造出花来？在使用了Prometheus+Grafana后，还真是感觉它能造出花来，除了监控
和及时有效的报警，它还非常”好看”，带给我们非常直观的视觉感受，在绚丽的界面中发现异常,感觉是技术与美的结合。

**目录**

* TOC
{:toc}

## 本机系统信息

HOSTNAME | IP ADDR | 虚拟机OS | Specifications 
:-: | :-: | :-: | :-: 
kube-master | 192.168.43.174 | ubunru 16.04 | 1核4G 

使用kube-master节点打造prometheus服务端和grafana服务端，并在其上运行node_exporter，向prometheus
暴露指标，由grafana进行高逼格的可视化展示。

## Prometheus 简介
Prometheus 是一款基于时序数据库的开源监控告警系统，使用Go语言开发，目前 Prometheus 已经广泛
用于 Kubernetes 集群的监控系统中，对云原生监控感兴趣的同学可以玩一玩Prometheus。

Prometheus 采用去中心化架构，支持独立部署，你可以快速地在短时间内搭建出一套监控系统。Prometheus 数据采集
方式灵活，在待采集目标处安装数据采集组件（Exporter），它会在目标处收集监控数据，并暴露出一个 HTTP 接口
供 Prometheus 查询，Prometheus 定时通过 Pull 的方式来采集数据，这和传统的 Push 模式不同。
不过 Prometheus 也提供了一种方式来支持 Push 模式，你可以将你的数据推送到 Push Gateway，Prometheus 通
过 Pull 的方式从 Push Gateway 获取数据。目前的 Exporter 已经可以采集绝大多数的第三方数据，比如 Docker、
HAProxy等，具体可访问[官网 Exporter 列表](https://prometheus.io/docs/instrumenting/exporters)。

下面我们来看下 Prometheus 的整体架构图，[图片来源](https://prometheus.io/docs/introduction/overview):

![Prometheus架构](/images/posts/monitoring/prometheus1.png "查看docker信息1")

Prometheus 包含几个关键的组件：Prometheus server、Pushgateway、Alertmanager、Web UI 等，但是大多数组件
都不是必需的，其中最核心的组件是 Prometheus server，它负责收集和存储指标数据，支持表达式查询和告警的生成。

接下来，我们就来手动安装下 Prometheus server。

#### Prometheus server 安装
目前Prometheus server 安装支持Docker, Ansible，二进制可执行文件解压等多种方式，我们这里选用二进制可执行文件
开箱即用方式安装Prometheus server。这里为了方便，全程使用`root`用户安装。

此处，选用Prometheus server 2.16.0(2020年2月)，从官方渠道下载并解压
```
wget https://github.com/prometheus/prometheus/releases/download/v2.16.0/prometheus-2.16.0.linux-amd64.tar.gz
mkdir -p /usr/local/prometheus && -tar -zxvf prometheus-2.16.0.linux-amd64.tar.gz -C /usr/local/prometheus --strip-components 1
```

然后切换到解压目录，检查 Prometheus 版本
```
cd /usr/local/prometheus
./prometheus --version
prometheus, version 2.16.0
...
```

最后，在后台运行 Prometheus server，若需要外网浏览器可访问，添加参数`--web.listen-address=0.0.0.0:9000`，监听9000端口
```
nohup ./prometheus --config.file=prometheus.yml --web.listen-address=0.0.0.0:9000 &
```

在浏览器可访问http://192.168.43.174:9000可访问如下页面
![Prometheus页面](/images/posts/monitoring/prometheus1.1.png "Prometheus页面")

#### Prometheus server 配置文件解读

Prometheus通过参数 --config.file 来指定配置文件，配置文件格式为 YAML。现在我们来看下默认配置文件 prometheus.yml 的内容：
```
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']
```

配置文件可分为4大块内容：
- global 块：全局配置，比如 scrape_interval 表示 Prometheus 多久抓取一次数据（默认15s），evaluation_interval 表示多久检测一次告警规则；
- alerting 块：Alertmanager 报警配置；
- rule_files 块：告警规则；
- scrape_config 块：定义 Prometheus 要抓取的目标，默认配置了一个名为 prometheus 的 job

Prometheus 启动时通过 HTTP 接口暴露自身的指标数据，相当于 Prometheus 自己监控自己，虽然在真实场景中没啥用处，
但通过这个例子我们可以学习如何使用 Prometheus。默认拉取指标的是9090端口。但在上述启动 Prometheus时，我们使用的是9000端口，
此时，在访问 http://localhost:9000/页面'Status'下拉菜单'Targets'时，发现服务是Down的状态，表明该端口服务未正常启动。
因此，我们想在不停服务时，选用热加载的方式修改配置文件端口号并更新服务
```
# 将targets: ['localhost:9090'] 改为 ['localhost:9000']
vim /usr/local/prometheus/prometheus.yml

# 热更新加载配置文件
curl -X POST http://127.0.0.1/-/reload
```
此时可能会报错，如下所示

![热加载失败](/images/posts/monitoring/prometheus2.png "热加载失败")

从 2.0 开始，Prometheus hot reload 热加载功能默认是关闭的，如需开启，需要在启动 Prometheus 时，添加 --web.enable-lifecycle 参数。
```
nohup ./prometheus --config.file=prometheus.yml --web.listen-address=0.0.0.0:9000 --web.enable-lifecycle &
```
此时，若 Prometheus 配置文件被修改，即可采用热更新方式实现在不停服务的情况下实现配置文件的重新加载。

然后再执行 `curl -X POST http://127.0.0.1/-/reload` 热更新加载命令，此时访问http://192.168.43.174:9000/metrics 
就可以查看 Prometheus 暴露的指标，如下所示

![指标](/images/posts/monitoring/prometheus4.jpg "指标")

#### Prometheus 指标说明
观察上图可知，在绿色方框中，promhttp_metric_handler_requests_total是指标名称，遵循一定的命名规范，
该指标表示抓取不同状态码的HTTP请求总数，其中，请求成功（即状态码是200）的次数是983次，服务器发生
异常（即状态码是500）的次数是0次。

HELP表示指标的说明描述，TYPE表示指标所属的类型，目前 Prometheus 采集的指标主要分为4类，存储的数据都是 float64 的一个数值
- Counter：用于计数，如：请求次数、任务完成数、错误发生次数，这个值只增不减
- Gauge：用于计数，可增可减，如：温度变化、内存使用率变化
- Histogram：直方图，或柱状图，可分组，提供 count 和 sum 的功能。如：请求耗时（耗时小于1s的请求次数，耗时1s-10s内的请求次数，超过10s的请求次数）。
- Summary：与Histogram 相似，区别在它提供了一个 quantiles 的功能，可按百分比划分跟踪结果。如：quantile 取值 0.8，表示取采样值里面的 80% 数据。

通常，这些类型在Exporter里实现并提供。
```
# HELP promhttp_metric_handler_requests_total Total number of scrapes by HTTP status code.
# TYPE promhttp_metric_handler_requests_total counter
promhttp_metric_handler_requests_total{code="200"} 983
promhttp_metric_handler_requests_total{code="500"} 0
promhttp_metric_handler_requests_total{code="503"} 0
```

虽然 Prometheus 提供的Web UI也可以查看不同指标的视图，但目前这个功能还比较简单，要实现一个强大的监控系统，
我们还需要支持更多更加多样的 Dashboard 仪表盘（如：曲线图、饼状图、TopN 等）。Prometheus 自身也开发了一套
仪表盘系统 PromDash，但很快就被废弃了，官方开始推荐使用 Grafana 来对 Prometheus 的指标数据进行可视化。

## Grafana 简介

Grafana 是一款用Go语言开发的开源数据可视化工具，可做数据监控和数据统计，并带有告警功能。其功能非常强大，界面酷炫，
可自定义创建控制面板，配置要显示的数据和显示方式，目前支持许多不同的数据源，如：Prometheus，InfluxDB、OpenTSDB、
Elasticsearch等，也支持众多的插件。

#### Grafana 安装
使用root用户，选用 Grafana 6.6.2(2020年2月)，从官方渠道下载并安装
```
apt-get install -y adduser libfontconfig1
wget https://dl.grafana.com/oss/release/grafana_6.6.2_amd64.deb
dpkg -i grafana_6.6.2_amd64.deb
```

然后启动Grafana服务，并设置开机自启
```
systemctl start grafana-server.service && systemctl enable grafana-server.service
```
如下所示：

![grafana启动](/images/posts/monitoring/grafana1.png "grafana启动")

然后在浏览器上输入 `http://192.168.43.174:3000/` 即可进入Grafana登陆界面，默认账户密码：`admin/admin` 如下所示：

![grafana登陆界面1](/images/posts/monitoring/grafana2.png "grafana登陆界面1")

点击登陆后会让用户修改密码，可跳过也可更改，自行选择即可，此处我选择修改，然后点击save按钮，如下所示：

![grafana登陆界面2](/images/posts/monitoring/grafana3.png "grafana登陆界面2")

然后，正式进入grafana界面，如下所示：

![grafana界面](/images/posts/monitoring/grafana4.png "grafana界面")

要使用 Grafana，第一步需要配置数据源，告诉 Grafana 从哪里取数据，点击 Add data source 进入数据源的配置页面，如下所示：

![grafana数据源配置界面1](/images/posts/monitoring/grafana5.png "grafana数据源配置界面1")

选择Prometheus时序数据库，如下所示：
     
![grafana数据源配置界面2](/images/posts/monitoring/grafana6.png "grafana数据源配置界面2")

依次填上所需配置，Name: Prometheus-test, URL: http://192.168.43.174:9000 (Prometheus sever 访问地址)，
然后点击 Save & Test，显示"Data source is working"，即为配置成功，如下所示：

![grafana数据源配置界面3](/images/posts/monitoring/grafana7.png "grafana数据源配置界面3")

#### Grafana导入JSON数据源图
导入画好的图，下载网上找的[JSON文件](https://grafana.com/api/dashboards/8919/revisions/11/download)，
进入Manage页面，准备导入。如下所示：

![导入json数据图1](/images/posts/monitoring/grafana8.png "导入json数据图1")
![导入json数据图2](/images/posts/monitoring/grafana9.png "导入json数据图2")

点击`Upload .json file`按钮，上传刚下载好的JSON文件，如下所示：

![导入json数据图2](/images/posts/monitoring/grafana10.png "导入json数据图2")

并为其配置名称（Name: Prometheus Dashboard）和数据源(上述配置的Prometheus-test)，然后点击`Import`按钮，如下所示：

![导入json数据图3](/images/posts/monitoring/grafana11.png "导入json数据图3")

此时，进入Prometheus Dashboard监控页面，页面面板很漂亮，但没有任何数据。回头看我们发现Prometheus未监控任何Exporter，
因此没有数据，如下所示：

![监控页面展示](/images/posts/monitoring/grafana12.png "监控页面展示")

## 安装node_exporter
为了解决上述Grafana面板没有监控数据的问题，我们需要为Prometheus server配置监控对象node_exporter，
从node_exporter处获取监控指标并展示。每台被监控的主机都要安装node_exporter，此处我们选用在安装有
prometheus sever的主机上安装node_exporter。

下载并安装node_exporter (v0.18.1)
```
wget https://github.com/prometheus/node_exporter/releases/download/v0.18.1/node_exporter-0.18.1.linux-amd64.tar.gz
mkdir /usr/local/node_exporter && tar -zxvf node_exporter-0.18.1.linux-amd64.tar.gz -C /usr/local/node_exporter --strip-components 1

# 后台启动node_exporter进程
cd /usr/local/node_exporter && nohup ./node_exporter &
```
如下所示：

![node_exporter启动1](/images/posts/monitoring/node_exporter1.png "node_exporter启动1")

此时，在/usr/local/node_exporter目录下使用`cat nohup.out`命令查看nohup.out文件，发现最后一行msg信息为
`INFO[0000] Listening on :9100     source="node_exporter.go:170"`

如下所示：
![node_exporter启动2](/images/posts/monitoring/node_exporter2.png "node_exporter启动2")

此时，在浏览器输入http://192.168.43.174:9100/metrics，可以看到已经采集的指标。

然后再修改prometheus服务端的配置文件，添加被监控主机（192.168.43.174:9100）
```
vim /usr/local/prometheus/prometheus.yml

# 在配置文件的末尾，添加如下信息，并保存退出
- job_name: 'nginx'
    static_configs:
    - targets: [192.168.43.174:9100]
```
如下所示：

![添加被监控主机](/images/posts/monitoring/node_exporter3.png "添加被监控主机")

热加载prometheus配置，使其无需重启prometheus server即可重新加载新配置
```
curl -X POST http://127.0.0.1/-/reload
```
如下所示：

![Prometheus热加载](/images/posts/monitoring/prometheus_reload.png "Prometheus热加载")

再访问 http://localhost:9000/页面'Status'下拉菜单'Targets'时，发现服务（192.168.43.174:9100）是Up的状态，
表明该服务已经被监控。如下所示：

![node_exporter监控主机](/images/posts/monitoring/prometheus_node_exporter.jpeg "node_exporter监控主机")

此时，我们再访问Grafana中的Prometheus Dashboard监控页面时，会发现面板上已经有监控数据了，如下所示：

![Grafana监控数据](/images/posts/monitoring/grafana_data.png "Grafana监控数据")

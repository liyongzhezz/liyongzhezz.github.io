---
title: 部署Prometheus、node-exporter和grafana
date: 2020-08-19 09:39:25
tags:
- Prometheus
categories:
- Prometheus
description: 部署prometheus、node-exporter和grafana服务
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1597811354534&di=4f9cf0a3bc115ebb13e67a946849bd8a&imgtype=0&src=http%3A%2F%2Ftm-image.tianyancha.com%2Ftm%2F1b2f7a725a0ef7361835201e25876cec.jpg
---



# 准备

## 时间同步

```bash
$ yum install -y ntpdate 
$ timedatectl set-timezone Asia/Shanghai 
$ ntpdate -u cn.pool.ntp.org 
$ echo "* * * * * /usr/sbin/ntpdate -u cn.pool.ntp.org >dev/null &" >> /var/spool/cron/root
```



<br>



# 安装Prometheus

## 下载

到prometheus的官网：https://prometheus.io/download/， 下载最新的版本，这里使用的是2.14.0：

```bash
$ wget https://github.com/prometheus/prometheus/releases/download/v2.14.0/prometheus-2.14.0.linux-amd64.tar.gz
```



## 安装

```bash
$ tar zxf prometheus-2.14.0.linux-amd64.tar.gz -C /usr/local/prometheus 
$ cd /usr/local/ 
$ mv prometheus-2.14.0.linux-amd64/ prometheus
```



## 运行

进入到prometheus目录下直接执行名为 prometheus 的二进制文件即可：

```bash
$ cd /usr/local/prometheus 
$ ./prometheus
```



>  当出现`Server is ready to receive web requests`的时候，说明prometheus已经启动成功。



这种方式是前台启动方式，如果要放在后台运行可以执行如下的命令：

```bash
$ nuhup ./prometheus &
```



使用prometheus命令启动的时候还支持以下的参数：

- `--config.file="/usr/local/prometheus/prometheus.yml"`：指定配置文件位置；
- `--web.read-timeout=5m`：获取数据时请求链接的最大等待时间，防止过多空闲链接浪费资源；
- `--web.max-connections=512`：获取数据时最多的连接数；
- `--storage.tsdb.retention=15d`：数据保留期限；
- `--storage.tsdb.path="/data"`：数据存储目录；
- `--query.timeout=2m`：查询数据超时时间；
- `--query.max-concurrency=20`：最大并发查询量； 
- `--web.enable-lifecycle`：热加载配置，支持通过命令 `curl -X POST http://localhost:9090/-/reload `的方式动态加载配置，而不用重启服务；

 

## 验证

启动后，prometheus默认监听 9090 端口：

```bash
$ netstat -ntlp | grep prome 
tcp6       0      0 :::9090                 :::*                    LISTEN      7898/./prometheus
```



然后可以打来浏览器访问本机的9090端口，就可以进入如下页面：

![](prome-web.png)



> prometheus本身是没有账号密码验证的，可以使用nginx的httppass等方式间接进行验证



<br>



# 安装node_exporter

## 下载

到prometheus的官网：https://prometheus.io/download/， 下载最新的版本，这里使用的是0.18.1：

```bash
$ wget https://github.com/prometheus/node_exporter/releases/download/v0.18.1/node_exporter-0.18.1.linux-amd64.tar.gz
```





## 安装

```bash
$ tar zxf node_exporter-0.18.1.linux-amd64.tar.gz -C /usr/local 
$ cd /usr/local/ 
$ mv node_exporter-0.18.1.linux-amd64 node_exporter
```





## 启动

```bash
$ cd /usr/local/node_exporter 
$ ./node_exporter
```



> 当出现 `Listening on :9100`  的时候表明启动成功了。



这样是前台启动的方式，想要放到后台启动，可以使用如下命令：

```bash
$ nobup ./node_exporter &
```



## 验证

node_exporter默认工作在9100端口：

```bash
$ netstat -ntlp | grep node_exp 
tcp6       0      0 :::9100                 :::*                    LISTEN      8647/./node_exporte
```



可以通过curl访问本机9100端口查看node_exporter的监控项：

```bash
$ curl localhost:9100/metrics
```



<br>



# 配置prometheus

配置文件位于解压后的目录，和启动程序同级。prometheus启动后会加载配置文件` prometheus.yml `。下面介绍下配置文件中的内容：

## global全局配置

- `scrape_interval`：数据采集时间间隔，默认是15秒；
- `evaluation_interval`：监控规则执行频率，默认15秒；



## alerting

报警相关配置，prometheus支持使用alertmanager进行报警，就在这里配置。当然也可以在grafana中配置报警。



## scrape_configs

这里定义的是数据抓取相关配置。

- `job_name`：任务名称；
- `static_config`：任务配置；
  - `targets`：任务目标位置（一般是ip+端口），可以并行写多个，逗号分隔；





## 添加node_exporter任务

在上边已经部署了node_exporter，那么这里我可以新增一个任务，然后指定目标为node_exporter，于是在  prometheus.yml 文件中添加如下的内容：

```yaml
- job_name: 'node_exporter'    
  static_configs:    
  - targets: ['192.168.1.106:9100']
```



这里新增了一个叫 `node_exporter` 的任务，目标为部署有node_exporter服务的服务器的9100端口；

重启prometheus：

```bash
$ kill -9 $(ps x | grep prome | grep -v grep | awk '{print $1}') 
$ nohup ./prometheus &

# 或者通过下面的命令动态加载配置
$ curl -X POST http://localhost:9090/-/reload
```



然后进入prometheus的页面，点击 `status --> targets`，看到定义的任务 node_exporter 如果是 UP 状态，就说明成功了。

![](prome-node.png)



<br>



# 使用daemonize方式将服务放入后台运行

daemonize是Unix系统后台守护进程管理软件，他更加正规，后台运行更加稳定。它是通过用户的启动脚本来启动后台进程。



## 安装daemonize

```bash
$ yum install -y gcc 
$ git clone git://github.com/bmc/daemonize.git 
$ cd daemonize 
$ sh configure 
$ make && make install
```





## 设置启动脚本

```bash
#!/bin/bash 

/usr/local/prometheus/prometheus --config.file="/usr/local/prometheus/prometheus.yml" --web.listen-address="0.0.0.0:9090" --web.read-timeout=5m --web.max-connections=10 --storage.tsdb.re tention=15d --storage.tsdb.path="/data/" --query.max-concurrency=20 --query.timeout=2m --web.enable-lifecycle
```



>  将上边的脚本保存到一个文件，例如我的是 `/root/prometheus-up.sh`  ，需要给这个脚本添加可执行权限。





## 运行后台服务

```bash
$ daemonize -c /root/ /root/prometheus-up.sh 
$ ps -ef | grep -v grep | grep peometheus
```



>  node_exporter同理，按照上边的配置即可。node_exporter有一些默认不开启的监控项可以在其github主页找到，并在启动的时候开启。



<br>



# 部署grafana

## 下载rpm包

可以从 grafana 官网获 https://grafana.com/grafana/download  取在最新的rpm包进行安装，这里我使用的是 6.5.2 版本：

```bash
$ wget https://dl.grafana.com/oss/release/grafana-6.5.2-1.x86_64.rpm
```





## 安装grafana

```bash
$ yum localinstall -y grafana-6.5.2-1.x86_64.rpm
```





## 启动grafana

```bash
$ systemctl start grafana-server 
$ systemctl enable grafana-server
```





## 验证

grafana默认监听在 3000端口：

```bash
$ netstat -ntlp | grep grafana 
tcp6       0      0 :::3000                 :::*                    LISTEN      15939/grafana-serve
```



然后通过浏览器访问本机的3000端口，输入默认密码：admin/admin，并且更改一下密码就可以进入grafana的主界面：

![](grafana.png)



## 添加数据源

第一次进入grafana，会要求添加一个数据源，在之前我已经部署好了 prometheus，地址为`192.168.1.106:9090`，所以这里我添加这个数据源。



点击` Add data source` ，在数据源类型中选择prometheus进入编辑表单，单后填入如下的内容：

![](datasource.png)



> 没有特殊要求的话，填入名称和url即可。



然后点击下边的 `save and test `进行连通性测试，当出现 `Data source is working` 的时候表示添成功。



可以在左侧侧边栏的 `Configconfiguration` 按钮下管理已经添加的数据源：

![](config.png)



## 添加图表

点击 `New dashboard `，然后选择 `Add Query ` ，此时我们创建了一张空的图表：

![](empdash.png)



下方是关于图表的一些配置信息，包括名称、数据源、计算公式等，这里以之前计算cpu使用率为例，来绘制图表，cpu使用率计算公式如下：

```bash
(1- (sum(increase(node_cpu_seconds_total{mode="idle"}[1m])) by (instance) / (sum(increase(node_cpu_seconds_total[1m])) by (instance)))) *  100
```



首先我们需要填入数据源和公式：

![](1.png)



然后进入下一步，设置图表类型以及X和Y轴：

![](2.png)

![](3.png)



最后，设置一下图表标题：

![](4.png)



这样，一张图表就添加完成了：

![](5.png)






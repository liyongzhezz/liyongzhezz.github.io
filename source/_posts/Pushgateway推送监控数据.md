---
title: Pushgateway推送监控数据
date: 2020-08-19 10:26:50
tags:
- Prometheus
categories:
- Prometheus
description: 安装pushgateway并推送监控数据到prometheus
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1597814166939&di=040c0d65c32b1f502c394b6075c7b571&imgtype=0&src=http%3A%2F%2Fpic1.zhimg.com%2Fv2-b18dd8a0cc614843760160720400d281_1200x500.jpg
---



# pushgateway介绍

pushgateway是一种采用push方式向prometheus推送监控数据的插件，它可以单独运行在任何节点上。



用户通过自定义脚本，把需要监控的数据发送到pushgateway，pushgateway再把数据推送到prometheus。

- pushgteway会形成一个单点瓶颈，如果有多个采集脚本向一个pushgateway发送数据，一旦pushgateway挂了，监控数据就没了；   
- pushgateway本身还是很稳定的，需要保证服务器不宕机基本没问题，只不过单点的话速度会慢；
- pushgateway不能对采集脚本发送的数据进行智能判断； 对监控数据采集脚本质量要求高；



<br>





# 安装pushgateway

## 下载

可以在prometheus官网 https://prometheus.io/download/  找到pushgateway最新的安装包，这里我使用的是1.0.0版本：

```bash
$ wget https://github.com/prometheus/pushgateway/releases/download/v1.0.0/pushgateway-1.0.0.linux-amd64.tar.gz 
$ tar zxf pushgateway-1.0.0.linux-amd64.tar.gz -C /usr/local/ 
$ cd /usr/local/ 
$ mv pushgateway-1.0.0 pushgateway
```



## prometheus配置pushgateway

编辑prometheus配置文件，增加如下内容，添加job：

```yaml
- job_name: 'pushgateway'  
  static_configs:    
  - targets: ['192.168.1.106:9091']
```



>  更新完配置后重启prometheus。





## 运行pushgateway

这里使用daemonize将其放入后台运行，首先编写脚本 `pushgateway-up.sh`：

```shell
#!/bin/bash 

/usr/local/pushgateway/pushgateway --web.listen-address="0.0.0.0:9091"
```



后台运行：

```bash
$ chmod +x pushgateway-up.sh 
$ daemonize -c /root/ /root/pushgateway-up.sh
```



## 验证

```bash
$ ps -ef | grep -v grep | grep pushgateway 
$ netstat -ntlp | grep pushgateway
```



<br>





# 推送数据到pushgateawy

## 编写脚本

首先编写一个脚本，这里是以获取`wati_connection`为例：

```shell
#!/bin/bash 

instance_name=`hostname` 
label="count_netstat_wait_connections" 

if [ $instance_name == 'localhost' ];then    
    echo "Must FQDN | HOSTNAME"    
    exit 1 
fi 

count_netstat_wait_connections=`netstat -an | grep -i wait | wc -l` 
echo "$label:$count_netstat_wait_connections" 

echo "$label count_netstat_wait_connections" | curl --data-binary @- http://192.168.1.106:9091/metrics/job/pushgateway/instance/$instance_name
```



在curl发送数据的url中比较重要的几个点：

- `job/pushgateway`：是prometheus中定义的job名称，我之前定义的是`pushgateway`，如果是其他名称请改动；
- `$instance_name`：不能为localhost，否则无法区分；





## 运行脚本

需要结合crontab定期运行脚本：

```bash
$ echo "* * * * * sh wait_connection.sh > /dev/null &" >> /var/spool/cron/root
```



>  crontab最小默认是1分钟一次，可以结合sleep实现每秒运行。






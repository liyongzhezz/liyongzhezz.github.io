---
title: Docker方式部署prometheus
date: 2020-09-05 18:44:49
tags:
- Prometheus
categories:
- Prometheus
- 服务部署
description: 使用docker方式部署prometheus
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1599312925466&di=b8408065486cd81238ab004bcba11897&imgtype=0&src=http%3A%2F%2Fimg.voycn.com%2Fimages%2F2019%2F09%2F88251df5ae97599e53312153dae3294e.jpg
---



# 使用官方镜像运行

直接执行下面的命令来运行官方镜像的prometheus：

```bash
$ docker run -d -p 9090:9090 --name=prometheus \
 -v  /root/prometheus/conf/:/etc/prometheus/  \
prom/prometheus 
```



在运行之前提前创建Prometheus配置文件`prometheus.yml`和Prometheus规则文件`rules.yml`放在`/root/prometheus/conf`下，挂在到容器中。



> Prometheus官方镜像没有开启热加载功能，而且时区相差八小时。



<br>



# 自制镜像运行

这里以`2.19.0`版本为例。



## 创建相关目录

首先创建一个根目录：

```bash
$ mkdir prometheus-2.9.0/
```



在这个目录下，有几个功能目录：

```bash
# 配置文件目录
$ mkdir prometheus-2.9.0/conf

# 监控规则目录
$ mkdir prometheus-2.9.0/conf/rules

# prometheus程序目录
$ mkdir prometheus-2.9.0/package
```



## 下载prometheus

在prometheus的github页面下载对应的版本：

```bash
$ wget https://github.com/prometheus/prometheus/releases/download/v2.19.0/prometheus-2.19.0.freebsd-amd64.tar.gz
```



将下载好的文件放到`package`目录下并解压：

```bash
$ mv prometheus-2.19.0.freebsd-amd64.tar.gz prometheus-2.9.0/package
$ cd prometheus-2.9.0/package
$ tar zxf prometheus-2.19.0.freebsd-amd64.tar.gz
```



此时的文件目录结构应该是这样的：

```bash
$ tree prometheus-2.9.0/
prometheus-2.9.0/
├── conf
    ├── rules
└── package
    ├── console_libraries
    ├── consoles
    ├── LICENSE
    ├── NOTICE
    ├── prometheus
    ├── prometheus.yml
    ├── tsdb
    └── promtool
```



## 创建启动脚本

在`conf`目录下创建`prometheus-start.sh`脚本作为服务的启动脚本：

```bash
#!/bin/bash
/bin/prometheus \
 --config.file=/data/prometheus/prometheus.yml \
 --storage.tsdb.path=/data/prometheus/data \
 --web.console.libraries=/data/prometheus/console_libraries \
 --web.enable-lifecycle \
 --web.console.templates=/data/prometheus/consoles
```



## 创建supervisord配置文件

这里通过`supervisord`服务来管理prometheus服务，在`conf`目录下创建名为`prometheus-start.conf`

的配置文件：

```ini
[program:prometheus]
command=sh /etc/supervisord.d/prometheus-start.sh   ; 程序启动命令
autostart=false     ; 在supervisord启动的时候不自动启动
startsecs=10        ; 启动10秒后没有异常退出，就表示进程正常启动了，默认1秒
autorestart=false   ; 关闭程序退出后自动重启，可选值：[unexpected,true,false]，默认为unexpected,表示进程意外杀死才重启
startretries=0      ; 启动失败自动重试次数，默认是3
user=root            ; 用哪个用户启动进程，默认是root
redirect_stderr=true            ; 把stderr重定向到stdout，默认false
stdout_logfile_maxbytes=20MB  ; stdout 日志文件大小，默认是50MB
stdout_logfile_backups=30        ; stdout 日志文件备份数，默认是10; 
# stdout 日志文件，需要注意当指定目录不存在时无法正常启动，所以需要手动创建目录(supervisord 会自动创建日志文件)
stdout_logfile=/data/prometheus/prometheus.log
stopasgroup=true
killasgroup=true
```



然后在`conf`目录下创建`supervisord`服务本身的配置文件`supervisord.conf`：

```ini
[unix_http_server]
file=/var/run/supervisor.sock   ; (the path to the socket file)
chmod=0700                       ; sockef file mode (default 0700)

[supervisord]
logfile=/var/log/supervisor/supervisord.log ; (main log file;default $CWD/supervisord.log)
pidfile=/var/run/supervisord.pid ; (supervisord pidfile;default supervisord.pid)
childlogdir=/var/log/supervisor            ; ('AUTO' child log dir, default $TEMP)
user=root
minfds=10240
minprocs=200

[rpcinterface:supervisor]
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface

[supervisorctl]
serverurl=unix:///var/run/supervisor.sock ; use a unix:// URL  for a unix socket

[program:sshd]
command=/usr/sbin/sshd -D
autostart=true
autorestart=true
stdout_logfile=/var/log/supervisor/ssh_out.log
stderr_logfile=/var/log/supervisor/ssh_err.log

[include]
files = /etc/supervisord.d/*.conf
```



## 创建容器启动脚本

在`conf`目录下创建名为`container-entrypoint`的文件，用于在容器启动后调用执行：

```bash
#!/bin/sh
set -x
if [ ! -d "/data/prometheus" ];then
    mkdir -p /data/prometheus/data
fi
mv /usr/local/src/* /data/prometheus/
exec /usr/bin/supervisord -n
exit
```



## 创建prometheus配置文件

在`conf`目录下创建`prometheus.yml`文件作为prometheus的配置文件：

```yaml
global:
  scrape_interval:   60s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 60s # Evaluate rules every 15 seconds. The default is every 1 minute.
alerting:
  alertmanagers:
  - static_configs:
    - targets: [ '192.168.133.110:9093']

rule_files:
  - "rules/host_sys.yml"

scrape_configs:
  - job_name: 'Host'
    static_configs:
      - targets: ['10.1.250.36:9100']
        labels:
          appname: 'DEV01_250.36'
  - job_name: 'prometheus'
    static_configs:
      - targets: [ '10.1.133.210:9090']
        labels:
          appname: 'Prometheus'
```

​	

> 这里示例写死了监控目标，实际上可以通过服务发现的方式来配置，例如consul



## 创建监控规则配置文件

在`conf/rules`下创建`service_down.yml`文件，用来配置监控报警规则：

```yaml
groups:
- name: servicedown
  rules:
  - alert: InstanceDown
    expr: up == 0
    for: 1m
    labels:
      name: instance
      severity: Critical
    annotations:
      summary: " {{ $labels.appname }}"
      description: " 服务停止运行 "
      value: "{{ $value }}"
```



## 创建Dockerfile

在顶层目录`prometheus-2.9.0/`下创建`Dockerfile`文件，用于构建镜像：

```dockerfile
FROM docker.io/centos:7
MAINTAINER from xxx@example.com

# install repo
RUN  rm -rf  /etc/yum.repos.d/*.repo
ADD  conf/CentOS7-Base-163.repo /etc/yum.repos.d/
ADD  conf/epel-7.repo           /etc/yum.repos.d/

# yum install
RUN yum install -q -y  openssh-server openssh-clients  net-tools \
  vim  supervisor && yum clean all

# install sshd
RUN  ssh-keygen -q -N "" -t rsa -f /etc/ssh/ssh_host_rsa_key \
  &&  ssh-keygen -q -N "" -t ecdsa -f /etc/ssh/ssh_host_ecdsa_key \
  &&  ssh-keygen -q -N "" -t ed25519 -f /etc/ssh/ssh_host_ed25519_key \
  &&  sed -i 's/#UseDNS yes/UseDNS no/g' /etc/ssh/sshd_config

# set UTF-8 and CST +0800
ENV  LANG=zh_CN.UTF-8 
RUN  echo "export LANG=zh_CN.UTF-8" >> /etc/profile.d/lang.sh \
    &&  ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
    && localedef -c -f UTF-8 -i zh_CN zh_CN.utf8

# install Prometheus
COPY  package/prometheus            /bin/prometheus
COPY  package/promtool              /bin/promtool
COPY  package/console_libraries/    /usr/local/src/console_libraries/
COPY  package/consoles/             /usr/local/src/consoles/
COPY  conf/prometheus.yml           /usr/local/src/prometheus.yml   
COPY  conf/rules/                   /usr/local/src/rules/

# create user
RUN  echo "root:root123456" | chpasswd 

# supervisord
ADD  conf/supervisord.conf            /etc/supervisord.conf
ADD  conf/prometheus-start.conf       /etc/supervisord.d/prometheus-start.conf
ADD  conf/container-entrypoint        /container-entrypoint
ADD  conf/prometheus-start.sh         /etc/supervisord.d/prometheus-start.sh
RUN  chmod +x /container-entrypoint

# cmd
CMD  ["/container-entrypoint"]
```



## 下载repo文件

在`conf`目录下下载一些基础的repo文件：

```bash
$ cd prometheus-2.9.0/conf
$ wget https://mirrors.163.com/.help/CentOS7-Base-163.repo
$ wget http://mirrors.aliyun.com/repo/epel-7.repo
```



此时的目录结构应该是这样的：

```bash
$ tree prometheus-2.9.0/
prometheus-2.9.0/
├── conf
│   ├── CentOS7-Base-163.repo
│   ├── container-entrypoint
│   ├── epel-7.repo
│   ├── prometheus-start.conf
│   ├── prometheus-start.sh
│   ├── prometheus.yml
│   ├── rules
│   │   └── service_down.yml
│   └── supervisord.conf
├── Dockerfile
└── package
    ├── console_libraries
    ├── consoles
    ├── LICENSE
    ├── NOTICE
    ├── prometheus
    ├── prometheus.yml
    ├── tsdb
    └── promtool
```



## 构建镜像并运行

```bash
# 构建镜像
$ docker build -t mybuild/prometheus:v2.9.0 .

# 运行容器
$ docker run -itd  -h prometheus139-210 -m 8g  --cpuset-cpus=28-31  --name=prometheus139-210 --network trust139  --ip=10.1.133.28  -v /data/works/prometheus139-210:/data  -p 9090:9090 mybuild/prometheus:v2.9.0
```



- `-h prometheus139-210`：设定容器的hostname为`prometheus139-210`；
- `-m 8g`：设定容器最大使用内存为8G；
- `--cpuset-cpus=28-31`：制定容器运行在编号为28到31的CPU上；
- `--name=prometheus139-210`：制定容器的名字；
- `--network`：设定容器使用的网络；
- `--ip`：容器固定IP；
- `-v`：挂在数据卷到容器中作为数据目录；
- `-p`：端口映射；



## 测试

容器启动后，访问本机的`9090`端口即可看到prometheus的页面了。



<br>


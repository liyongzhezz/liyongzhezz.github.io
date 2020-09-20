---
title: haproxy介绍和部署
date: 2020-09-20 11:55:31
tags:
- HAproxy
categories: 
- 负载均衡和高可用方案
- HAproxy
description: haproxy介绍和安装配置
cover: https://ss0.bdstatic.com/70cFvHSh_Q1YnxGkpoWK1HF6hhy/it/u=3429957096,767347968&fm=26&gp=0.jpg
---





# HAproxy介绍

## 什么是HAproxy

HAproxy是C语言编写的一个提供高可用性、负载均衡及四层（TCP）和七层（HTTP）应用代理的软件，特别适合用于负载大的web站点。



## HAproxy的特点

- 支持虚拟主机；
- 支持session保持、cookie的引导等；
- 支持后端服务健康检查，故障自动转移；
- 支持4、7层代理，更细粒度的七层负载均衡（对请求url和header中信息进行匹配）；
- 自带web管理页面；
- 能从进出两个方向修改http header信息；
- 支持多种复杂均衡算法；



## HAproxy和nginx对比

`角色定位`：nginx本身定位就是一个web server，而haproxy则是纯粹的负载均衡器；



`活跃度`：nginx比较活跃，更新快，社区模块丰富；haproxy更新比较慢；



`性能`：作为负载均衡器，haproxy要更胜一筹，但nginx不会落后太多；





<br>



# 安装haproxy

## yum方式安装

```bash
$ yum install -y haproxy
```



使用yum安装的haproxy版本可能会比较旧，生产场景建议使用源码安装的方式。



## 源码安装

这里安装的是`2.2.3`版本：

```bash
$ wget http://www.haproxy.org/download/2.2/src/haproxy-2.2.3.tar.gz
$ tar zxf haproxy-2.2.3.tar.gz
$ uname -a
Linux VM-90-115-centos 3.10.107-1-tlinux2_kvm_guest-0049 #1 SMP Tue Jul 30 23:46:29 CST 2019 x86_64 x86_64 x86_64 GNU/Linux

$ make TARGET=linux310107 ARCH=x86_64 PREFIX=/usr/local/haproxy
$ make install PREFIX=/usr/local/haproxy

# 创建保存配置文件的目录
$ mkdir -p /usr/local/haproxy/config/
```

> TARGET：系统内核版本， ARCH：系统架构，PREFIX：安装目录



设置haproxy启动文件：

```bash
cat > /usr/lib/systemd/system/haproxy.service << EOF
[Unit]
Description=HAProxy Load Balancer
After=syslog.target network.target

[Service]
ExecStartPre=/usr/local/haproxy/sbin/haproxy -f /usr/local/haproxy/config/haproxy.cfg -c -q
ExecStart=/usr/local/haproxy/sbin/haproxy -Ws -f /usr/local/haproxy/config/haproxy.cfg -p /var/run/haproxy.pid
ExecReload=/bin/kill -USR2 $MAINPID
KillMode=mixed

[Install]
WantedBy=multi-user.target
EOF
```



<br>



# 配置文件解析

使用yum方式安装的，默认会生成一个配置文件在`/etc/haproxy/haproxy.cfg`，使用源码安装的，则需要自己创建。配置文件包含以下几个重要的块：



## global

**设置进程级别的配置。**

- `log`：定义全局的syslog服务器，最多定义两个；
- `chroot`：修改工作目录至指定的目录并在放弃权限之前执行chroot()操作，可以提升haproxy的安全级别，不过需要注意的是要确保指定的目录为空目录且任何用户均不能有写权限；
- `pidfile`：pid文件位置；
- `maxconn`：每个haproxy进程最大的并发连接数；
- `user/uid`：服务使用的用户；
- `group/gid`：服务使用的用户组；
- `daemon`：放在后台运行；



> 需要在`/etc/rsyslog.conf`下添加`local2.*          /var/log/haproxy.log`才能正常输出日志



## defaults

**配置默认参数，可以应用到frontend、backend、listen组件**

- `mode { tcp|http|health }`：设置实例的运行模式或协议，当实现内容交换时，前后端须一致；
  - tcp：运行纯tcp模式，在客户端和服务器端之间将建立一个全双工的连接，且不会对7层报文做任何类型的检查；通常用于SSL、SSH、SMTP等应用；
  - http：实例运行于HTTP模式，客户端请求在转发至后端服务器之前将被深度分析，所有不与RFC格式兼容的请求都会被拒绝；此为默认模式； 
  - health：实例工作于health模式，其对入站请求仅响应“OK”信息并关闭连接，且不会记录任何日志信息；此模式将用于响应外部组件的健康状态检查请求；目前来讲，此模式已经废弃，因为tcp或http模式中的monitor关键字可完成类似功能；
- `log global`：全局日志；
- `option httplog`：启动记录访问日志；
- `option dontlognull`：日志中不记录空链接；
- `option redispatch`：在使用基于cookie定向时，一旦后端某一server宕机时，会将会话重新定向至某一上游服务器，必须使用的选项；
- `option forwardfor except 127.0.0.0/8`：允许在发往服务器的请求首部中插入“X-Forwarded-For”首部；
- `retries`：重试次数；
- `timeout http-request`：http请求超时时间；
- `maxconn`：设定一个前端的最大并发连接数；

## frontend

**接收请求的前端虚拟节点，frontend可以根据规则直接指定具体的后端backend**

- `frontend <name>`：前端名字；
- `bind`：监听的ip和端口；
- `mode`：运行模式；
- `default_backend <backend name>`：后端的名字；



## backend

**后端服务集群配置，可以覆盖掉全局配置**

- `backend <name>`：后端的名字；
- `balance`：负载均衡算法；
- `server <name> <addr>`：后端节点信息；




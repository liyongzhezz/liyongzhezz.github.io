---
title: Nginx Ingress性能调优
date: 2020-09-09 17:53:36
tags:
- Ingress
- k8s
categories:
- 实践K8s
- 调优
description: 对Nginx Ingress进行高并发场景下性能优化
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1599655866231&di=6dcf53f8ddee33711c9b7d034f9f92de&imgtype=0&src=http%3A%2F%2Fpic4.zhimg.com%2Fv2-b150ca39ea4691b50d3b11c3481f61b3_1440w.jpg%3Fsource%3D172ae18b
---



在高并发场景下，需要对Nginx Ingress进行内核参数调优，才能发挥出性能优势，下面先介绍相关的参数，然后通过`initContainers`方式进行调优。

---



# 系统参数优化



## 连接队列大小调整

进程监听socket的连接队列最大的大小受限于`net.core.somaxconn`参数，在高并发下如果队列过小会导致队列溢出，部分连接将无法建立。建议将这个参数调整为`65535`：

```bash
$ sysctl -w net.core.somaxconn=65535
```



实际上，nginx ingress会读取`somaxconn`作为`backlog`参数写到配置文件中，也就是说nginx ingress的连接队列大小只取决于`somaxconn`的值。



> 在nginx中`backlog`是进程调用listen监听端口时传入的参数，这个参数决定socket队列大小，默认511，所以一般nginx如果不设置这个值，队列大小就只有511。



## 扩大源端口范围

高并发场景下nginx ingress会产生大量的源端口与upstream建立连接，通过`net.ipv4.ip_local_port_range`参数指定。默认是`327868 ~ 60999`：

```bash
$ sysctl -w net.ipv4.ip_local_port_range="1024 65535"
```



## TIME_WAIT复用

短连接并发高的时候会产生大量的`TIME_WAIT`，其默认2MSL才会释放，长期占用端口资源会导致新的请求无法连接，所以需要开启复用：

```bash
$ sysctl -w net.ipv4.tcp_tw_reuse=1
```



## 调整最大文件句柄

作为反向代理的nginx ingress会和client和upstream分别建立一个文件句柄，所以理论上能同时处理的最大连接数是系统最大文件句柄的一半，建议调大：

```bash
$ sysctl -w fs.file-max=1048576
```

<br>



# 通过initContainer配置

在创建nginx ingress的pod的时候添加如下的配置：

```yaml
initContainer:
- name: setsysctl
  image: busybox
  securityContext:
    privileged: true
  command:
  - sh
  - -c
  - |
    sysctl -w net.core.somaxconn=65535
    sysctl -w net.ipv4.ip_local_port_range="1024 65535"
    sysctl -w net.ipv4.tcp_tw_reuse=1
    sysctl -w fs.file-max=1048576
```



<br>



# nginx 优化

## keepalive调整

keepalive连接由`keepalive_requests`参数控制单个keepalive最大请求数，默认100，当单个client的QPS过大，会导致keepalive连接频繁断开，产生大量的`TIME_WAIT`状态。所以应该增大`keep-alive-alive-requests`参数



## 单个worker的最大连接数

`max-worker-connections`控制每个worker进程可以打开的最大连接数，在高并发环境下建议增大，让nginx有处理更多请求的能力。



<br>



# 通过configmap配置

nginx的配置通过configmap进行设置，nginx ingress会watch这并重新加载：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-ingress-controller
data:
  keep-alive-requests: "10000"
  max-worker-connections: "65535"
```


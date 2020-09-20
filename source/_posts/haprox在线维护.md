---
title: haproxy在线维护
date: 2020-09-20 13:38:58
tags:
- HAproxy
categories: 
- 负载均衡和高可用方案
- HAproxy
description: haproxy在线维护不停服
cover: https://ss0.bdstatic.com/70cFvHSh_Q1YnxGkpoWK1HF6hhy/it/u=3429957096,767347968&fm=26&gp=0.jpg
---



haproxy支持在不停止服务的情况下对后端节点进行维护，主要流程是：

1. 通过socket将某个需要维护的后端节点置于维护模式；
2. 升级后端节点；
3. 将后端节点通过socket再加入代理中；



这样做的好处是haproxy可以不停止服务，对用户影响比较小。



首先在配置文件的global下添加一个配置：

```bash
stats socket /var/lib/haproxy/haproxy.sock mode 600 level admin
```

> 这里是设置了haproxy的一个socket地址，并制定了文件权限为600，admin指通过socket可以进行管理操作



然后需要安装一个socket管理的命令`socat`：

```bash
$ yum install -y socat
```



通过下面的命令可以查看socket管理支持的指令：

```bash
$ echo "help" | socat stdio /var/lib/haproxy/haproxy.sock
```



例如将某一个server置于维护模式就可以写成：

```bash
$ echo "disable server backend_www_example_com/node-1" | socat stdio /var/lib/haproxy/haproxy.sock
```

> `backend_www_example_com`为backend名称；`node-1`为该backend下的某一个server名称



**这样，该server就被执行为维护状态，可以进行升级等操作而不会影响用户访问**



升级结束后，可以通过下面的命令将其重新加入backend提供服务：

```bash
$ echo "enable server backend_www_example_com/node-1" | socat stdio /var/lib/haproxy/haproxy.sock
```


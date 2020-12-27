---
title: docker方式部署zookeeper集群
date: 2020-12-16 14:56:37
tags:
- zookeeper
categories:
- zookeeper
description: 使用docker方式部署zookeeper集群
cover:
---


## 规划
部署一个5节点zookeeper集群：

| 主机名 | IP地址 | id   |
| ------ | ------ | ---- |
|        |        |      |
|        |        |      |
|        |        |      |
|        |        |      |
|        |        |      |



zookeeper版本：3.5.5

<br>



## 部署

### 创建目录

在每个节点都创建数据和日志目录：

```bash
$ mkdir -p /data/zookeeper/data
$ mkdir -p /data/zookeeper/logs
```



### 下载镜像

这里使用的是3.5.5版本的zookeeper：

```bash
$ docker pull zookeeper:3.5.5
```



### 创建配置文件

配置文件的内容如下：

```bash
cat > /data/zookeeper/zoo.cfg << EOF
clientPort=2181
dataDir=/data/zookeeper/data
dataLogDir=/data/zookeeper/logs
tickTime=2000
initLimit=5
syncLimit=2
autopurge.snapRetainCount=3
autopurge.purgeInterval=0
maxClientCnxns=60
standaloneEnabled=false
admin.enableServer=true
4lw.commands.whitelist=*
server.2=192.168.12.12:2888:3888
server.3=192.168.12.13:2888:3888
server.4=192.168.12.14:2888:3888
server.5=192.168.12.15:2888:3888
server.6=192.168.12.16:2888:3888
EOF
```



- `clientPort`：服务监听的端口；
- `dataDir`：数据目录；
- `dataLogDir`：顺序日志目录；
- `tickTime`：心跳间隔；
- `autopurge.snapRetainCount`：保留多少个snapshot，之前的会删除；
- `autopurge.purgeInterval`：多久会清理一次数据（0表示不清理）；
- `maxClientCnxns`：客户端连接数限制；
- `standaloneEnabled`：在启动脚本中关闭管理控制台；
- `4lw.commands.whitelist`：白名单；



> 注意修改配置文件中的server配置



**配置文件需要在每个节点都创建**



### 生成myid文件

根据预先设置的myid，在每个节点的`/data/zookeeper/data`下创建一个`myid`文件：

```bash
$ echo 2 > /data/zookeeper/data/myid
```



> 注意每个节点的myid文件内容不一样



### 启动zookeeper

在每个节点分别执行下面的命令启动zookeeper：

```bash
$ docker run \
    --name zookeeper \
    --network=host \
    --restart always \
    -v /data/zookeeper/conf/zoo.cfg:/conf/zoo.cfg \
    -v /data/zookeeper:/data/zookeeper \
    -v /etc/localtime:/etc/localtime:ro \
    -d zookeeper:3.5.5
```



<br>



## 检查


---
title: 部署单点zookeeper
date: 2021-04-06 14:57:30
tags:
- Zookeeper
categories:
- Zookeeper
- 部署
description: 部署一个单点的适合于开发测试场景的zookeeper服务
cover:
---

{% note info 'fas fa-bullhorn' %}

本文主要介绍了部署一个单点的zookeeper3.5.5服务

更新于 2021-04-06

{% endnote %}

<br>



## 下载

下载地址：http://archive.apache.org/dist/zookeeper/

>  需要下载的包必须带`bin`，这个是编译好的，否则运行会报错



```bash
wget http://archive.apache.org/dist/zookeeper/zookeeper-3.5.5/apache-zookeeper-3.5.5-bin.tar.gz
```



<br>



## 安装

下载好的zookeeper压缩包已经经过编译，直接解压就可以用：

```bash
tar zxf apache-zookeeper-3.5.5-bin.tar.gz 
cd apache-zookeeper-3.5.5-bin
```

<br>

## 建立配置文件

默认没有配置文件，可以直接复制配置文件模板：

```bash
mv conf/zoo_sample.cfg conf/zoo.cfg
```

<br>

## 创建数据目录

虽然zookeeper一般数据量不大，但是还是推荐使用单独的数据盘：

```bash
mkdir /data/zookeeper
```



然后修改配置文件的如下字段，使用数据目录：

```bash
# conf/zoo.cfg
dataDir=/data/zookeeper
```

<br>

## 后台方式启动服务

```bash
bin/zkServer.sh start
```

![](./start.png)



## 前台方式启动

```bash
bin/zkServer.sh start-foreground
```

<br>



## 查看服务状态

```bash
bin/zkServer.sh status 
```

![](./status.png)

<br>

## 查看日志

如果是后台方式启动，日志默认存放在安装目录下的logs中，文件名包含主机名：

```bash
tail -f apache-zookeeper-3.5.5-bin/logs/zookeeper-root-server-localhost.out
```





## 停止zookeeper服务

```bash
bin/zkServer.sh stop 
```

![](./stop.png)



<br>
---
title: zookeeper常用操作
date: 2020-12-12 18:06:43
tags:
- zookeeper
categories:
- zookeeper
description: zookeeper日常用到的操作
cover:
---



## 登录zookeeper shell

使用如下命令连接zookeeper：

```bash
$ bin/zkCli.sh
```



<br>



## 节点操作

### 查看节点

使用下面的命令查看都有哪些节点：

```bash
[zk: localhost:2181(CONNECTED) 1] ls /
```

![](./ls.png)

> 默认只有一个zookeeper节点



### 创建一个节点

```bash
[zk: localhost:2181(CONNECTED) 9] create /workers ""
Created /workers
[zk: localhost:2181(CONNECTED) 10] ls /
[workers, zookeeper]
```



> 当创建/workers节点后指定了一个空字符串("")，说明此刻不希望在这个znode中保存数据。然而，该接口中的这个参数可以使我们保存任何字符串到ZooKeeper的节点中。比如，可以替 换""为"workers"。



### 删除节点

```bash
[zk: localhost:2181(CONNECTED) 11] delete /workers
[zk: localhost:2181(CONNECTED) 12] ls /
[zookeeper]
```



<br>



## 退出shell

```bash
[zk: localhost:2181(CONNECTED) 13] quit

WATCHER::

WatchedEvent state:Closed type:None path:null
2020-12-12 15:14:16,594 [myid:] - INFO  [main:ZooKeeper@1422] - Session: 0x100f8067ce90000 closed
2020-12-12 15:14:16,594 [myid:] - INFO  [main-EventThread:ClientCnxn$EventThread@524] - EventThread shut down for session: 0x100f8067ce90000
```


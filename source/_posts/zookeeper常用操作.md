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

使用如下命令连接本地的zookeeper：

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



### 创建带序号的节点

如果此时再次支执行`create /workers ""`，则会报错：`Node already exists`，此时可以使用`-s`参数表示创建一个带序号的节点：

```bash
[zk: localhost:2181(CONNECTED) 9] create -s /workers ""
Created /workers000000002
```



> 带序号的节点会把序号追加到节点名字后且全局唯一



### 创建临时节点

在`create`的时候使用`-e`可以创建临时节点：

```bash
[zk: localhost:2181(CONNECTED) 9] create -e /tmpnode ""
Created /tmnpnode
```



当创建临时节点的会话中断之后，临时节点就会消失；



### 注册到节点

通知客户端zookeeper节点数据变化是zookeeper的一个重要功能，使用如下的命令可以将当前会话注册到zookeeper：

```bash
[zk: localhost:2181(CONNECTED) 9] ls -w / 
```

> `-w`表示`watch`，意思是监控当前的节点，如果节点发生变化则会被zookeepr通知



此时如果`/`节点发生了变化，则这个会话会收到通知：`WatchedEvent state:SyncConnected type:NodeChildrenChanged path:/`





### 查看节点状态

使用下面的命令可以查看节点的状态：

```bash
[zk: localhost:2181(CONNECTED) 9] state /workers
```



其中会输出如下内容：

- `czxid`：创建节点的事物id；
- `ctime`：创建该节点的时间；
- `mZxid`：修改该节点的事物id；
- `pZxid`：最后一次修改子节点的zxid；
- `cversion`：子节点修改次数；
- `dataversion`：当前节点修改次数；
- `aclVersion`：访问控制列表变化的次数；





### 删除节点

```bash
[zk: localhost:2181(CONNECTED) 11] delete /workers
[zk: localhost:2181(CONNECTED) 12] ls /
[zookeeper]
```



`delete`只能删除没有子节点的节点，如果要递归删除则使用`deleteall`：

```bash
[zk: localhost:2181(CONNECTED) 11] deleteall /workers
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


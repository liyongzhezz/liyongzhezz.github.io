---
title: HDFS
date: 2020-10-11 16:39:41
tags:
- 大数据
- HDFS
categories:
- 大数据
- HDFS
description: HDFS的概念原理
cover: https://ss0.bdstatic.com/70cFvHSh_Q1YnxGkpoWK1HF6hhy/it/u=3715612854,779723728&fm=26&gp=0.jpg
---



## 介绍HDFS

### 什么是hdfs

hadoop实现了一个分布式文件系统：hadoop distributed file system，简称HDFS。它是基于谷歌的GFS论文，是GFS的克隆版本。



### hdfs设计目标

- 为大数据存储场景设计；
- 可以运行在普通的廉价的硬件上；
- 易扩展，为用户提供性能不错的文件存储服务；



### 运行环境

hdfs由java开发，所以只要机器运行有java环境即可运行hdfs的相关进程。

<br>



### 优缺点

**优点**：

- 数据冗余、硬件容错；
- 适合一次写入，多次读取；
- 适合存储大文件；
- 可以构建在廉价的服务器上；



**缺点**：

- 不适合低延迟数据访问；
- 不适合小文件存储；



## HDFS架构



### 架构图

![](hdfs.jpg)

hdfs是一个master/slave架构，通常有一个namenode（master简称NN），多个datenode（slave简称DN）：

- datanode：存储文件对应的block，定期向namenode发送心跳信息汇报本身的健康状况和block信息；
- namenode：负责客户端请求的相应、元数据并调度数据存储；



### 副本机制

![](rep.jpg)

文件会被切分为多个block，block为了容错会以多副本的方式存放在不同的节点上，block的大小和副本个数都是可以配置的。



### block存放策略

按照hdfs的策略，第一个副本存放的位置是client提交文件所在的节点上，第二个副本会存放在不同的机架上的节点，第三个副本存放在第二个机架的不同服务器上。

![](rack.png)



### 数据存储和读取

文件会按照`blocksize`的大小拆分成为多个Block进行存储，每个block又会按照多副本的方式存放在不同的节点。

> 例如blocksize大小为128M，那么130M的文件会被拆分为128M和2M的两个block进行存储。



读取的时候，会先到namenode上查找文件的元数据信息（block个数和位置）。



<br>



## HDFS数据读写的流程



### 写流程

1. 向客户端发起写请求，指明block大小及副本数（客户端一般会有默认值）；
2. 客户端将文件被拆分为多个block；
3. 客户端向namenode请求存储多个block并指明副本个数；
4. namenode通过计算返回合适的datenode，客户端开始向第一个datenode中写入数据；
5. 当第一个datenode开始写数据后，这个datenode会将数据转发给后续的datenode进行存储；
6. datenode写完数据后向namenode上报存储结果；
7. namenode返回存储成功结果到客户端，客户端确认后继续存储后续的block或者关闭连接；



### 读流程

1. 客户端向namenode发起请求，请求包括需要读取的文件名；
2. namenode查询后，返回该文件的元数据信息，包括：有哪些block、block所在datenode集合；
3. client向datenode发起请求，下载对应的block信息并组合；
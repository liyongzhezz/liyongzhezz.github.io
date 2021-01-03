---
title: 分布式计算框架MapReduce
date: 2021-01-03 12:27:36
tags:
- 大数据
- MapReduce
categories:
- 大数据
- MapReduce
description: 分布式计算框架MapReduce介绍及使用
cover: https://ss0.bdstatic.com/70cFuHSh_Q1YnxGkpoWK1HF6hhy/it/u=3932792420,2118816984&fm=26&gp=0.jpg
---



## 概述

MapReduce是一个软件框架，源自于google的MapReduce论文，发表于2004年，hadoop MapReduce是google MapReduce的克隆版，MapReduce可以进行海量数据离线处理，非常容易进行开发和运行，但是不适合进行实时流式计算。



整个作业阶段会经历：Map阶段和Reduce阶段，在Map阶段将会执行Map tasks，在Reduce阶段将会执行Reduce tasks：

1.   准备map处理的输入数据；
2. mapper处理；
3. shuffle；
4. reduce处理；
5. 结果输出；



<br>



## 核心概念

- `split`：交由MapReduce作业处理的数据块，是MapReduce中最小的计算单元；默认情况下`split`和HDFS的blocksize是一一对应的；
- `InputFormat`：将输入数据进行分片；
- `OutputFormat`：将结果输出；



![](./core.png)



<br>



## 架构

### MapReduce 1.x

![](./v1.png)

`JobTracker（JT）`：

- 负责作业管理，将作业分解为一堆任务（MapTask和ReduceTask）;
- 将任务分派给`TaskTracker`运行；
- 作业监控、容错处理（当task挂了，重启task的机制）；
- 在一定时间间隔内，没有收到`TaskTracker`心跳信息则重新指派任务到其他`TaskTracker`上执行；



`TaskTracker(TT)`：

- 负责任务执行，执行MapTask和ReduceTask；
- 执行/启动/停止作业，发送心跳信息给`JobTracker`；



`MapTask`：

- 自己开发的map任务交由该task执行；
- 解析每条记录的数据，交给自己的map方法处理；
- 将map的输出结果写到本地磁盘（有些作业仅有map，则将结果写到hdfs）



`ReduceTask`：

- 将`MapTask`的数据读取，按照数据进行分组，传给自己实现的reduce方法进行处理；
- 输出结果到hdfs；





### MapReduce 2.x

![](./v2.png)





<br>



## MapReduce之wordcount

利用MapReduce进行文件的workcount（词频计数）相当于编程中的hello world，它的基本流程如下：

![](./wordcount.png)



1. 将文件进行拆分，例如按行拆分；
2. 启动多个作业对拆分的文件进行并行处理，对每个单词出现的次数设为1；
3. 将相同的单词放到一台机器上；
4. 合并计算出最后的词频结果；
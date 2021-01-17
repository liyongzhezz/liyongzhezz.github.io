---
title: hadoop介绍
date: 2020-09-20 16:09:52
tags:
- 大数据
- Hadoop
categories:
- 大数据
- Hadoop
description: 介绍hadoop的基本概念和生态系统
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1600599516655&di=3397e82eb9e4a48d3319d9d2c8f98e09&imgtype=0&src=http%3A%2F%2Fwww.cbdio.com%2Fimage%2Fattachement%2Fjpg%2Fsite2%2F20141230%2F3417eb9bbd59160c582e52.jpg
---



# hadoop介绍



## 什么是hadoop

hadoop是apache社区的顶级项目，是一个开源的高可靠、可扩展的分布式计算框架。它允许分布式处理大数据集，数据集可以横跨多个集群。



> 分布式存储+分布式计算的底层平台



## hadoop包含的模块

- `hadoop common`：为其他hadoop模块提供工具集；
- `hdfs`：hadoop的分布式文件系统，可以提供高吞吐量；
- `yarn`：作业调度和集群资源管理框架；
- `mapreduce`：基于yarn的大数据集并行处理框架；



## hadoop特性

高可靠性体现：

- 数据存储多副本；
- 任务可以重复调度计算；



高扩展性体现：

- 横向扩展机器增加计算资源；
- 一个集群能够包含上千个节点；



其他方面：

- 存储可以落地在廉价的机器上，减少成本；
- 成熟的生态圈；





## hadoop的用途

- 搭建大型的数据仓库；
- 分析、存储、处理pb级别的数据；
- 搜索引擎、日志分析、数据挖掘、商业智能；





<br>



# 核心组件--分布式文件系统HDFS



hdfs基于谷歌的论文`gfs`，具有很强的扩展性和容错性，并且能够存储海量的数据；



上传到hdfs中的数据会被切分为指定大小的数据块并以多副本的方式存储在多个节点上。如果某个副本所在服务器挂了，由于存在副本，所以数据还是可以访问的。



![](hdfs.png)



<br>



# 核心组件--YARN

YARN是hadoop中负责资源管理和调度的组件。例如一个作业提交后占用的资源大小的分配，都是YARN来负责的。



YARN具有高扩展性和容错性，也可以进行多框架资源统一调度。

![](yarn.jpg)



<br>



# 核心组件-- mapreduce

这个组件源于谷歌的论文`MapReduce`，具有高扩展性和容错性，能够进行海量数据的离线处理。



![](mapreduce.png)



从上图的mapreduce的流程来看，它的执行步骤主要分为两步：

- `mapping`：数据映射；
- `reducing`：数据合并；



<br>



# Hadoop发行版和选型



## 常见发行版

- Apache hadoop：apache官方的hadoop；
- CDH：cloudera公司发布的hadoop发行版；
- HDP：hortonworks发布的hadoop发行版；



## 选型

apache hadoop在使用hadoop生态圈的其他软件结合解决问题的时候，有可能会出现jar包冲突的问题，测试和学习环境可以使用apache hadoop，生产可以使用CDH和HDM版本，搭建和升级方便。



<br>



# Hadoop 3.x新特性



> hadoop3.x需要依赖java 8 及以上版本



## common改进

- shell脚本进行了重写，修复了存在的bug以及新增了新特性，并保持了兼容性；
- 删除了过时的api；



## HDFS改进

- 支持了 eresure的编码方式；
- 支持两个以上的NameNode；
- Intra-datanode balancer，防止在新增或替换磁盘时导致数据不平衡；
- 多个服务的默认端口变化：NameNode、SecondaryNameNode、DataNode、KMS（以前的端口处于Linux的默认端口范围内，有可能启动时会有端口冲突问题）；



## YARN改进

- yarn timeline service v.2；
- support for opportunistic containers and distributed scheduling;



## MapReduce改进

- task级别进行了优化；
- hadoop daemon进程和task堆内存管理优化；




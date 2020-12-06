---
title: 分布式资源调度框架yarn
date: 2020-12-06 14:12:35
tags:
- 大数据
- YARN
categories:
- 大数据
- YARN
description: 分布式资源调度框架YARN的介绍及伪分布式部署 
cover: https://ss0.bdstatic.com/70cFuHSh_Q1YnxGkpoWK1HF6hhy/it/u=203111150,1649122478&fm=11&gp=0.jpg
---

 

## YARN 产生的背景

MapReduce在1.x的架构图如下：

![](mapreduce.png)

其中：

- `JobTracker`负责资源调度和作业调度；
- `TaskTracker`负责定期向`JobTracker`汇报本节点资源状态、作业执行情况和健康状况，并接受来自`JobTracker`的命令（启动或杀死Job）；



从架构图中可以看到：

- `JobTracker`存在单点故障问题；
- `JobTracker`会接受`TaskTracker`和客户端的请求，如果集群规模大会压力大成为瓶颈；
- 仅能运行mapreduce作业，不支持其他类型的框架；



<br>



## YARN概念

### 什么是YARN

针对以上问题，YARN诞生了，yarn可以理解为一个操作系统级别的调度框架，为上层应用提供了统一的任务和资源调度，可以让很多的不同的计算框架运行在一个集群中并共享一个hdfs，按需分配资源，极大的减少了运维成本并提高了资源利用率。

![](yarn.jpg)



> 这就是所谓的 `XXX on YARN`，如：`Spark on YARN`、`Flink on YARN`



### YARN架构

![](yarn-arch.png)



重要的组件：

- `resource manager(RM)`：
  - 整个集群同一时间**提供服务的RM**只有一个；
  - 负责资源统一管理和资源调度；
  - 处理客户端的请求：创建作业、杀死作业；
  - 监控NM，一旦一个NM挂了，则通知AM处理该NM上的任务；
- `node manager(NM)`：
  - 整个集群中有多个NM；
  - 负责单个节点本身的资源管理和使用，定时向RM汇报本节点资源使用情况；
  - 接收并处理来自RM的命令：启动container等；
  - 处理来自AM的命令
- `App Mstr（AM）`：
  - 每个应用程序对应一个AM，运行在container中；
  - 负责应用程序管理，为应用程序向RM申请资源，分配给内部的task；
  - 需要与NM通信：启动/停止task，task运行在container中；
- `container`：
  - 封装了CPU、Memory等资源的一个容器，是一个任务运行环境的抽象；
- `client`：
  - 提交作业、查看作业进度、杀死作业；



### YARN执行流程

![](liucheng.jpg)

1. 用户（client）向yarn提交一个作业；
2. RM与NM通信要求在该NM上启动一个AM container，AM container注册到RM，此时client就可以通过RM查询到作业运行情况；
3. AM container根据作业向RM申请资源，并在NM上启动task container；



<br>



## 单节点伪分布式环境搭建

这里使用的是`hadoop-2.6.0-cdh5.7.0`，下载和hdfs搭建可以参考：{% post_link hdfs伪分布式搭建 hdfs伪分布式搭建 %}



### 设置mapreduce运行在yarn上

```bash
$ cd etc/hadoop/
$ cp mapred-site.xml.template mapred-site.xml
```



编辑`etc/hadoop/mapred-site.xml`，增加如下的内容，表明让mapreduce作业运行在yarn上：

```xml
<property>
  <name>mapreduce.framework.name</name>
  <value>yarn</value>
</property>
```





### 设置yarn

编辑`etc/hadoop/yarn-site.xml`，在`<configuration>`下增加如下的内容：

```xml
<property>
  <name>yarn.nodemanager.aux-services</name>
  <value>mapreduce_shuffle</value>
</property>
```





### 启动RM和NM进程

执行下面的命令启动进程：

```bash
$ sbin/start-yarn.sh
```

![](startyarn.png)



通过`jps`命令查看进程是否启动：

```bash
$ jps
```

![](jps.png)

> 可以看到`ResourceManager`和`NodeManager`启动了



### 访问RM

通过浏览器访问RM，默认端口时8088

![](web.png)



### 停止yarn

```bash
$ sbin/stop-yarn.sh
```



<br>



## 提交一个作业到YARN运行

在`hadoop-2.6.0-cdh5.7.0/share/hadoop/mapreduce`有一个`hadoop-mapreduce-examples-2.6.0-cdh5.7.0.jar`可以作为一个例子来运行。



```bash
$ hadoop jar hadoop-mapreduce-examples-2.6.0-cdh5.7.0.jar
```

![](example.png)



可以看到这个jar包包含了很多用例，这里以`pi`这个mapreduce作业为例，它将使用蒙特卡洛算法计算pi的值

```bash
$ hadoop jar hadoop-mapreduce-examples-2.6.0-cdh5.7.0.jar pi 2 3
```

> 参数分别表示2个map和3个取样数量



此时在web页面上查看任务：

![](task.png)

过了一会儿任务结束：

![](task-finished.png)



最后的结果可以在控制台上查看：

![](result.png)

> 这个结果好像有问题，但是不重要
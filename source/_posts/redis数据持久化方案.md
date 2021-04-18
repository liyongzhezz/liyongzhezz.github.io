---
title: redis数据持久化方案
date: 2021-04-18 15:07:59
tags:
- Redis
categories:
- 数据库
- Redis
- 数据持久化和主从复制
description: Redis的两种数据持久化方案：AOF和RDB
cover: https://gimg2.baidu.com/image_search/src=http%3A%2F%2Fwww.linuxdiyf.com%2Flinux%2Fuploads%2Fallimg%2F160228%2F2-16022P94942238.jpg&refer=http%3A%2F%2Fwww.linuxdiyf.com&app=2002&size=f9999,10000&q=a80&n=0&g=0n&fmt=jpeg?sec=1621321738&t=d0e2350d8170a863dbe58a9953fddfb9
---



{% note info 'fas fa-bullhorn' %}

本文主要介绍redis的两种数据持久化方案：AOF和RDB

更新于 2021-04-18

{% endnote %}

<br>



# RDB方式



>  RDB方式是利用子进程在指定的时间间隔内，将内存中的数据集生成快照写入磁盘中；默认支持，不需要进行配置





## 优势和缺陷

优势：

- redis数据库将只包含一个文件（利于备份）；
- 利于灾难恢复；
- 性能最大化，启动效率最高（原理是fork一个子进程进行数据备份）；

缺陷：

- 无法完全保证数据高可用；
- 数据量大的时候可能在数据持久化的时候导致服务不可用；



> 如果redis使用的内存很大，例如32G等，如果使用RDB方式则建议磁盘选择ssd，否则会由于备份数据文件过大导致磁盘IO不足。



## 配置RDB持久化
在redis.conf配置文件中，有如下的一些关于RDB持久化的配置：

```bash
// 每900秒至少1个key发生变化就持久化一次
save 900 1

// 每300秒至少10个key发生变化就持久化一次
save 300 10

// 每60秒至少10000个key发生变化就持久化一次
save 60 10000

// 持久化数据文件名
dbfilename dump.rdb

// 数据文件路径
dir ./

// 不做持久化则注释上边的选项并打开下边选项的注释
save ""
```



<br>



# AOF方式

## AOF方式

将以日志的方式记录服务器处理的每一次操作。服务器启动之初会读取日志文件来重建数据库。



## 优势和缺陷

优势：

- 更高的数据安全性；
- 不会破坏日志文件的数据完整性；
- redis可以自动启动日志重写机制，对重复的日志进行合并，减小文件大小；
- 包含格式清晰的日志文件。可以基于这个日志文件进行数据重建和迁移；
- AOF将数据库操作顺序写到文件的最后，如果出现误操作，可以在文件中删掉最后一行在进行还原；



劣势：

- 对于相同的数据集来说，AOF往往比RDB文件大；
- 效率低于RDB；



## 配置AOF持久化

在redis.conf配置文件中，有如下的一些关于AOF持久化的配置：

```bash
// AOF持久化功能开关，默认没有打开，改成yes表示打开
appendonly yes

// AOF文件名
appendfilename "appendonly.aof"

// AOF同步策略设置
// 每修改一次同步到磁盘上
appendfsync always

// 每秒同步一次到磁盘上
appendfsync everysec

// 不同步
appendfsync no
```

<br>



# 同时使用RDB和AOF需要注意

在同时使用AOF和RDB持久化数据的时候，重启redis后，优先使用AOF进行数据恢复。


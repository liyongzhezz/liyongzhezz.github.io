---
title: Redis基础知识
date: 2021-04-18 13:05:01
tags:
- Redis
categories:
- 数据库
- Redis
- 基础
description: 介绍redis的基本原理和特性
cover: https://gimg2.baidu.com/image_search/src=http%3A%2F%2F5b0988e595225.cdn.sohucs.com%2Fimages%2F20180327%2F34adc98d775145f0b23c5fa67217af1d.png&refer=http%3A%2F%2F5b0988e595225.cdn.sohucs.com&app=2002&size=f9999,10000&q=a80&n=0&g=0n&fmt=jpeg?sec=1621314652&t=420ccb71557b55ae50c2d87c053e9bb0
---



{% note info 'fas fa-bullhorn' %}

本文主要介绍redis的基本原理和特性

更新于 2021-04-18

{% endnote %}

<br>



# 什么是Redis

Redis是C语言编写的开源的基于键值对的存储服务系统，支持多种数据结构的一种高性能，功能丰富的NoSQL数据库。



Redis很像memcache，整个数据都是放在内存中进行操作的，定期通过异步方式持久化到硬盘上。

> Redis有时候也会用户缓存，相比于memcache，如果是单纯的缓存场景，memcache性能是优于redis的；

<br>



# Redis的特点

- **速度快**：单线程模型，数据存储在内存，最高可达10W OPS；
- **支持持久化**：提供AOF和RDB方式的持久化；
- **支持多种数据结构**：支持字符串、哈希、列表、集合、有序集合等数据结构；
- **支持多种编程语言**：提供TCP接口，支持Python，Java，Lua等；
- **功能丰富**：支持发布订阅、lua脚本、事物、pipeline等功能；
- **代码简单，使用简单**：核心代码量2W行，个性化定制方便；不依赖外部的库；
- **支持主从复制**
- **支持高可用和分布式**：2.8版本后提供Sentinel功能以支持高可用；3.0版本后支持分布式；
- **原子性**：redis所有操作都是原子的(要么成功要么失败)；

> redis支持的数据结构有：字符串(string)、哈希(hash)、字符串列表(list)、字符串集合(set)、有序字符串集合(sorted set)等



<br>



# Redis应用场景

- **缓存系统**：使用redis在server层和存储层中间构建存储层，加速请求的响应，减轻后端压力；
- **计数器**：一条微博的转发数、评论数、点赞数；
- **消息队列系统**
- **排行榜功能**
- **社交网络**：粉丝数、关注数、共同关注等；
- **实时系统**：垃圾邮件系统、过滤器；

<br>



# 持久化方案

Redis提供了两种数据持久化的方式：AOF和RDB，用于防止服务器宕机导致数据丢失。



## RDB

rdb是Redis DataBase缩写，核心函数为`rdbSave`(生成RDB文件)和`rdbLoad`（从文件加载内存）两个函数。



![img](./rdb.svg)‌



## AOF

每当执行服务器任务或者函数时`flushAppendOnlyFile`函数都会被调用， 这个函数执行以下两个工作：

‌

1. WRITE：根据条件，将 aof_buf 中的缓存写入到 AOF 文件；
2. SAVE：根据条件，调用 fsync 或 fdatasync 函数，将 AOF 文件保存到磁盘中；





![img](./aof.svg)



‌

## 两种方式比较

- aof文件比rdb更新频率高，优先使用aof还原数据；
- aof比rdb更安全也更大；
- rdb性能比aof好；
- 如果两个都配了优先加载AOF；



<br>



# Redis特性

## 多数据库特性

一个redis实例可以包含多个数据库，一个客户端可以指定连接redis实例其中的一个数据库。

> 一个redis实例可以提供16个数据库，编号从0 到 15。客户端默认是连接0号数据库。



### 连接指定的数据库

可以通过`select`指令加上数据库编号来连接数据库，例如：

```bash
127.0.0.1:6379> select 1
OK

127.0.0.1:6379[1]> keys *
(empty list or set)

127.0.0.1:6379[1]> select 0
OK

127.0.0.1:6379> keys *
 1) "mya3"
 2) "mya1"
 3) "qq"
 ...
```





### 数据库间移动key

可以将一个key从一个库移动到另一个库，使用`move`指令，例如：

```bash
// 将0库中的myset这个key移动到1库中
127.0.0.1:6379> move myset 1
(integer) 1

127.0.0.1:6379> select 1
OK

127.0.0.1:6379[1]> keys *
1) "myset"
```

<br>



## 事务特性

事务执行将被串行执行，执行期间redis将不会为其他客户端提供服务，从而保证事务中的指令原子化执行。redis中实现事务特性使用multi、exec、discard指令。

- multi：将会创建一个事务，其后的参数将被视为事务中的命令；
- exec：exec将会执行事务；
- discard：事务回滚



如果在exec之前出现网络问题连接不上redis，则事务中的命令不会被执行；如果exec之后出现网络问题连接不上redis，则事务将继续执行。

### multi

multi将会开启一个事务，其后的指令将会存在事务命令队列中，直到执行。

```bash
127.0.0.1:6379> set num 2
OK

127.0.0.1:6379> get num
"2"

127.0.0.1:6379> incr num
QUEUED

127.0.0.1:6379> incr num
QUEUED
```



>  可以看到，事务命令已经被存在队列中。



### exec

exec将会执行事务命令队列中的命令，相当于mysql中的commit。

```bash
127.0.0.1:6379> exec 
1) (integer) 3
2) (integer) 4
```



### discard

discard是回滚操作，相当于mysql中的rollback。

```bash
127.0.0.1:6379> set user tom
OK
127.0.0.1:6379> get user
"tom"
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set user jerry
QUEUED
127.0.0.1:6379> discard
OK
127.0.0.1:6379> get user
"tom"
```


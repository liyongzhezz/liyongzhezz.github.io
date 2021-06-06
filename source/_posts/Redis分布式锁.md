---
title: Redis分布式锁
date: 2021-06-06 18:00:58
tags:
- Redis
categories:
- 数据库
- Redis
- 常用操作
description: redis中分布式锁的原理和使用方法
cover: https://gimg2.baidu.com/image_search/src=http%3A%2F%2Fimg.php.cn%2Fupload%2Farticle%2F000%2F000%2F029%2F5cda85b2be126130.jpg&refer=http%3A%2F%2Fimg.php.cn&app=2002&size=f9999,10000&q=a80&n=0&g=0n&fmt=jpeg?sec=1611641043&t=dc5360df5690a298d45dddbf1299fad2
---



{% note info 'fas fa-bullhorn' %}

本文主要介绍redis中常见的几种数据类型的基本命令行操作

更新于 2021-06-06

{% endnote %}

<br>



## 分布式应用遇到的问题

分布式应用在逻辑处理时常会遇到并发的问题。



![](./fenbushi.png)

一个修改用户状态的操作是从redis中读取数据修改然后写回redis，但这样的操作如果同时又多个操作在进行，就会出现并发的问题，因为读和写不是原子操作。



> 原子操作是指不会被线程调度机制打断的操作，这种操作从开始到结束中间不会有线程切换



这种时候最好是通过redis的分布式锁方式进行解决。



<br>



## 分布式锁的原理

分布式锁的本质类似于一个令牌，当进程想要操作redis的时候会先尝试获取这个令牌，如果发现这个令牌已经被别的进程占有，则会放弃或者稍后再试；



### 占用和释放锁

一般使用`setnx`占用锁，用完后使用`del`释放锁，例如：

```bash
127.0.0.1:6379> setnx lock:hole true
OK
```



这样就占据了锁，然后就进行自己的处理逻辑即可，完成后需要将锁释放：

```bash
127.0.0.1:6379> del lock:hole
(integer) 1
```



### 避免死锁

如果程序占用锁后，自己的处理逻辑出现异常导致所不能释放，那么会出现死锁，其他程序也就无法调用，这时候可以给锁加上一个过期时间，到了时间锁会强制释放：

```bash
127.0.0.1:6379> setnx lock:hole true
OK
127.0.0.1:6379> expire lock:hole 5
```



>  这里是添加了5秒的过期时间



但是这两个操作`setnx`和`expire`不是原子操作，假设`setnx`和`expire`操作之间出现问题，也会出现死锁的现象，所以推荐使用下面的方式，抢占锁的同时设置过期时间：

```bash
127.0.0.1:6379> set lock:hole true ex 5 nx
```



<br>



## 超时问题

使用锁的时候都会设置超时时间，如果当逻辑执行时间太长，已经到了锁释放时间还没执行完，就会导致第二个线程持有了这个锁。所以redis锁不适合那种执行时间长的任务。



一种避免这个问题的方法是在设置锁的时候设置一个随机数：

```python
tag = random.nextint()

if redis.set(key, tag, nx=True, ex=5):
  do_somthing()
  redis.delifequals(key, tag)
```




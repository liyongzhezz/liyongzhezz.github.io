---
title: 部署Redis
date: 2020-07-05 21:12:32
tags:
- Redis
categories: 
- 数据库
- Redis
description: 使用yum和源码两种方式部署Redis
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1593964908993&di=2711fc4cc3a4ac713a65fe86d24bac76&imgtype=0&src=http%3A%2F%2F5b0988e595225.cdn.sohucs.com%2Fimages%2F20180327%2F34adc98d775145f0b23c5fa67217af1d.png
---



## yum方式部署

### 依赖安装

```bash
$ yum install -y epel-release
```



###  安装Redis

```bash
$ yum install -y redis
$ redis-cli --version
```



直接yum安装的redis不是最新的版本，如果要安装最新版本需要执行下面的步骤：

```bash
$ yum install -y http://rpms.famillecollet.com/enterprise/remi-release-7.rpm
$ yum --enablerepo=remi install redis -y
```



###  设置配置文件

修改`/etc/redis.conf`中的`bind`参数为下面的值，监听所有IP地址：

```bash
bind 0.0.0.0
```

<br>



###  设置系统参数

```bash
$ sysctl vm.overcommit_memory=1
$ echo "sysctl vm.overcommit_memory=1" >> /etc/rc.local
```

`vm.overcommit_memory`是控制内存分配策略的参数：

- 1：内核分配所有的物理内存而不管当前内存状态；
- 0：内核检查是否有足够的内存共当前进程使用，没有则会返回错误给进程；
- 2：内核允许分配超过物理内存和交换空间总和的内存；



### 启动Redis

```bash
$ systemctl start redis
$ systemctl enable redis
$ systemctl status redis
```

<br>

------



## 源码方式安装

### 安装依赖

```bash
$ yum install -y gcc gcc-c++
```



### 下载并编译

```bash
$ wget https://github.com/antirez/redis/archive/5.0.3.tar.gz
$ tar zxf 5.0.3.tar.gz
$ cd redis-5.0.3
$ make PREFIX=/usr/local/redis install
```



###  设置环境变量和系统参数

```bash
$ echo "PATH=$PATH:/usr/local/redis/bin" >> /etc/profile
$ source /etc/profile
```



安装后会在`/usr/local/redis`下生成安装目录，安装的命令有如下：

```
# /usr/local/redis/bin
redis-benchmark：性能测试工具
redis-check-aof：文件修复工具
redis-check-dump：rbd文件检查工具
redis-cli：客户端命令行工具
redis-server：服务启动命令
```



设置系统参数：

```bash
$ sysctl vm.overcommit_memory=1
$ echo "sysctl vm.overcommit_memory=1" >> /etc/rc.local
```



`vm.overcommit_memory`是控制内存分配策略的参数：

- 1：内核分配所有的物理内存而不管当前内存状态；
- 0：内核检查是否有足够的内存共当前进程使用，没有则会返回错误给进程；
- 2：内核允许分配超过物理内存和交换空间总和的内存；



###  设置配置文件

编辑配置文件`/usr/local/redis/redis.conf`，修改`daemonize`参数为下面的值，让redis在后台运行：

```
daemonize yes
```

> 如果没有`redis.conf`，可以拷贝源码包中的配置文件过去



###  启动Redis

```bash
$ redis-server /usr/loca/redis/redis.conf

$ ps -ef | grep redis
root      10983      1  0 13:57 ?        00:00:00 redis-server 127.0.0.1:6379

$ netstat -ntlp | grep redis
tcp        0      0 127.0.0.1:6379          0.0.0.0:*               LISTEN      10983/redis-server
```



###  连接Redis

```bash
$ redis-cli
127.0.0.1:6379>
```



###  停止Redis

```bash
$ redis-cli shutdonw
```


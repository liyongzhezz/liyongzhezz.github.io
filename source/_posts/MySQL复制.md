---
title: MySQL复制
date: 2021-01-03 14:47:50
tags:
- MySQL
categories:
- 数据库
- MySQL
description: mysql数据库复制功能
cover: https://gimg2.baidu.com/image_search/src=http%3A%2F%2Fimg.mp.sohu.com%2Fupload%2F20170623%2Ff59b0882997245bca244fa7599797e42_th.png&refer=http%3A%2F%2Fimg.mp.sohu.com&app=2002&size=f9999,10000&q=a80&n=0&g=0n&fmt=jpeg?sec=1612248532&t=eef7be3609bfbf0d8c679daf0498a9a7
---



## MySQL复制介绍



### 什么是复制

mysql 复制（mysql replication）主要用于主服务器和从服务器之间的数据复制操作，主要是将主服务器的DDL和DML操作通过二进制日志传送到从服务器上，然后在从服务器上对这些日志重新进行执行，从而使得主从服务器数据保持一致；



> 复制操作是异步的，从服务器不需要持续保持连接用于接收主服务器的数据



### 常见的主从复制架构

**一主一从**

![](./one-one.png)



**一主多从**

![](./one-many.png)

> 这种可以将从服务器作为只读服务器，常用与提高系统的读性能



**多主一从**

![](./many-one.png)

> 这是从5.7开始支持的方式，可将多个数据库备份到一台性能较好的服务器上



**双主模式**

![](./master-master.png)

> 每台服务器即是master，又是另一台的slave，何一方所做的变更，都会通过复制到另外一方的数据库中



**级联模式**

![](./jilian.png)

> 级联模式下部分从服务器将会作为下级slave的主服务器



### 复制的基本步骤

1. 主服务器将数据变动记录到二进制日志中；
2. 从服务器将主服务器的二进制日志复制到自己的中继日志（relay log）中；
3. 从服务器重新执行中继日志中的事件，将数据改变与从服务器保持同步；



<br>



## 使用mysql_multi启动多实例



### 环境

mysql版本：5.7

操作系统：centos7



> 这里没有那么多资源，所以在一个服务器上启动3个mysql实例来试验复制功能



### 部署mysql

这里使用源码方式部署mysql，可以参考 {% post_link 部署MySQL5-7 %}



### 创建数据目录并初始化

`mysql_multi`是管理多个mysql的服务进程，这些mysql实例监听不同的端口，首先需要停止已经在运行的mysql，然后创建3个mysql实例目录：

```bash
$ mkdir -p /data/mysql/{3306,3307,3308}
```



然后初始化：

```bash
$ mysql_install_db --datadir=/data/mysql/3306 --user=mysql
$ mysql_install_db --datadir=/data/mysql/3307 --user=mysql
$ mysql_install_db --datadir=/data/mysql/3308 --user=mysql
$ cp /usr/local/mysql/support-files/mysqld_multi.server /etc/init.d/mysqld_multi.server
```



### 修改配置文件

修改mysql配置文件，添加三个实例的配置：

```bash
$ cp /etc/my.cnf /etc/my.cnf.bak
$ cat > /etc/my.cnf << EOF
[mysqld_multi]
mysqld     = /usr/local/mysql/bin/mysqld_safe
mysqladmin = /usr/local/mysql/bin/mysqladmin
user       = root

[mysqld1]
port    = 3306
datadir = /data/3306
socket  = /var/run/mysqld1.sock

[mysqld2]
port    = 3307
datadir = /data/3307
socket  = /var/run/mysqld2.sock

[mysqld3]
port    = 3308
datadir = /data/3308
socket  = /var/run/mysqld3.sock

[mysqld]
EOF
```



### 启动多实例

```bash
# 启动
$ mysqld_multi --defaults-extra-file=/etc/my.cnf start

# 查看状态
$ mysqld_multi --defaults-extra-file=/etc/my.cnf report
```



> 如果想要通知，则执行`mysqld_multi --defaults-extra-file=/etc/my.cnf stop`



### 检查状态

```bash
# 查看端口
$ netstat -ntlp | grep 330

# 登录测试
$ mysql -u root -p -P 3306
$ mysql -u root -p -P 3307
$ mysql -u root -p -P 3308
```



<br>



## 单机主从复制

### 前提

这里我们以3306端口的mysql服务为主，其他两个为从；



### 设置复制用户

登录主服务器，设置一个复制专用的账号并授权：

```bash
$ mysql -u root -p -P 3306

mysql> grant replication slave on *.* to 'repl'@'localhost' identified by '123';
mysql> grant replication slave on *.* to 'repl'@'%' identified by '123';
```



### 启动binlong

修改配置文件`my.cnf`，将主数据库开启binlog：

```bash
# my.cnf
[mysqld1]
port    = 3306
datadir = /data/3306
socket  = /var/run/mysqld1.sock
log-bin = /data/3306/mysql-bin
server-id = 1
```



修改完后重启：

```bash
$ mysqld_multi --defaults-extra-file=/etc/my.cnf stop 1-3
$ mysqld_multi --defaults-extra-file=/etc/my.cnf start 1-3
$ mysqld_multi --defaults-extra-file=/etc/my.cnf report
```



在主数据库上设置锁定有效：

```bash
$ mysql -u root -p -P 3306

mysql> flush tables with read lock;
```



> 这个命令杀伤力很大，需要确保数据库目前无操作，命令执行中数据库可能会hang住，主要用于备份工具获取一致性备份(数据与binlog位点匹配)



登录master数据库执行下面的命令：

```mysql
mysql> show master status;
```



> 这个命令将得到当前日志的名字和偏移量，以便后面复制从这个点开始复制



可以对主库做一个备份，可以在服务器停止的情况下直接使用系统复制命令：

```bash
$ tar zcf data.tar.gz data
```



操作完成后，取消锁：

```mysql
mysql> unlock tables;
```



### 设置从库配置文件

修改配置文件`my.cnf`：

```bash
[mysqld_multi]
mysqld     = /usr/local/mysql/bin/mysqld_safe
mysqladmin = /usr/local/mysql/bin/mysqladmin
user       = root

[mysqld1]
[mysqld1]
port      = 3306
datadir   = /data/3306
socket    = /var/run/mysqld1.sock
log-bin   = /data/3306/mysql-bin
server-id = 1

[mysqld2]
port      = 3307
datadir   = /data/3307
socket    = /var/run/mysqld2.sock
log-bin   = /data/3307/mysql-bin
server-id = 2

[mysqld3]
port      = 3308
datadir   = /data/3308
socket    = /var/run/mysqld3.sock
log-bin   = /data/3308/mysql-bin
server-id = 3

[mysqld]
```



> 注意每个实例的server-id是不一样的



重启服务：

```bash
$ mysqld_multi --defaults-extra-file=/etc/my.cnf stop 1-3
$ mysqld_multi --defaults-extra-file=/etc/my.cnf start 1-3
$ mysqld_multi --defaults-extra-file=/etc/my.cnf report
```



### 从库数据库配置

从数据库需要指定复制的用户、主库的地址端口和开始复制的文件位置等：

```bash
$ mysql -u root -p -P 3307

mysq> show variables like '%log_bin%';

mysql> stop slave;

mysql> change master to master_host="127.0.0.1" master_user="repl" master_password="123" master_log_file="mysql-bin.000001" master_log_pos=120;

mysql> start slave;
```

> `master_log_pos`是之前设置master时候显示的



### 检查

在从库执行下面的sql：

```mysql
# 查询从服务器状态
mysql> show slave status\G;

# 查询从服务器进程状态
mysql> show processlist\G;
```



### 验证

现在可以在主服务器上执行一次更新操作，看从服务器是否进行了复制：

```bash
# 登录主服务器
$ mysql -u root -p -P 3306

# 创建一张表
mysql> use test;
mysql> show tables;
mysql> create table repl_test(id int);
mysql> insert into repl_test values(1),(2);
```



登录从服务器，看是否同步过来了：

```bash
# 登录从服务器
$ mysql -u root -p -P 3307

mysql> use test;
mysql> show tables;
mysql> select * from repl_test;
```



> 这里只操作了3307，3308这个实例也是同样的操作
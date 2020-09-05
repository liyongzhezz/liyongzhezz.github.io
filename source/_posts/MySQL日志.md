---
title: MySQL日志
date: 2020-08-18 16:17:20
tags:
- MySQL
categories:
- 数据库
- MySQL
description: MySQL中的一些常见日志
cover: https://ss0.bdstatic.com/70cFvHSh_Q1YnxGkpoWK1HF6hhy/it/u=2049101755,2221976873&fm=26&gp=0.jpg
---



mysql日志记录了用户日常操作和错误信息，从日志中可以查询到mysql的运行情况。mysql日志主要分为4类：

- 错误日志：记录mysql启动、运行、停止中出现的问题；
- 查询日志：记录客户端建立的连接和执行的语句；
- 二进制日志：记录所有更改数据的语句，可用于数据复制；
- 慢查询日志：记录所有执行时间超过 long_query _time 的查询或不使用索引的查询；



------





# 二进制日志

## 什么是二进制日志

二进制日志主要记录mysql中数据库的变化，其中包含所有更新了数据以及潜在更新了数据（例如未匹配到任何行的 DELETE）的语句。

>  使用二进制日志的最大目的是恢复数据库。



## 启动和设置二进制日志

默认情况下，二进制日志是关闭的，可以通过修改mysql配置文件 `my.cnf` 来启动和设置二进制日志，例如

```bash
log-bin 
expire_logs_days = 10 
max_binlog_size = 100M 
server_id=1
```

- `log-bin`：表示开启二进制日志，后面也可以指定日志文件名称，例如：`log-bin=/data/binlog/binlog.log` 表示将二进制日志放在 `/data/binlog`下取名为 `binlog.log`；
- `expire_logs_days`：清理过期日志的时间，默认为0，表示不删除；
- `max_binlog_size`：binlog文件最大大小，默认1G，如果超过当前指定的大小，日志会滚动；
- `server_id`：为mysql提供一个独立的ID，用于使用binlog做主从备份；



设置好上边的参数，重启即可。重启后登录mysql，使用如下命令检查是否成功开启：

```mysql
mysql> show variables like 'log_bin'; 
+---------------+-------+ 
| Variable_name | Value | 
+---------------+-------+ 
| log_bin       | ON    | 
+---------------+-------+
```

>  看到log_bin 已经开启。



## 查看二进制日志

二进制日志中，有两种日志：

- `xxx.index`：保存所有的日志文件名清单；
- `xxx.数字`：实际日志数据文件；

mysql服务器每重新启动一次，`xxx.数字` 格式的日志文件就会增加一个。在mysql中，通过下面命令就可以查看当前所有的binlog以及大小：

```mysql
mysql> show binary logs; 
+---------------+-----------+ 
| Log_name      | File_size | 
+---------------+-----------+ 
| binlog.000001 |       177 | 
| binlog.000002 |       177 | 
| binlog.000003 |       154 | 
+---------------+-----------+
```



二进制日志不能直接查看，需要使用专门的指令查看：

```bash
$ mysqlbinlog /data/binlog/binlog.000001
```





## 删除二进制日志

### 删除所有二进制日志

使用 RESET MASTER 指令可以删除所有的二进制日志并重新创建，编号从000001开始

```mysql
mysql> reset master; 
Query OK, 0 rows affected (0.02 sec) 
mysql> show binary logs; 
+---------------+-----------+ 
| Log_name      | File_size | 
+---------------+-----------+ 
| binlog.000001 |       154 | 
+---------------+-----------+
```





### 删除指定的日志

使用 `PURGE MASTER LOG` 指令可以删除指定的日志，其格式如下：

```bash
# 方式1：指定文件名，删除文件名编号比指定文件名编号小的所有日志 
PURGE {MASTER | BINARY}  LOGS TO 'log_name' 

# 方式2：指定日期，删除指定日期前的日志 
PURGE {MASTER | BINARY} LOGS BEFORE 'date'
```



例如：

```mysql
# 删除 binlog.00003之前的日志 
mysql> purge master logs to 'binlog.00003'; 

# 删除2018年1月30号之前的日志 
mysql> purge master logs before '20180130';
```





## 使用二进制日志恢复数据库

当数据库丢失数据时，可以使用 `mysqlbinlog` 命令恢复数据，其格式如下：

```bash
$ mysqlbinlog [options] filename | mysql -u<user> -p<password>
```

- options：一系列可选参数；

- - --start-data：开始时间；
  - --stop-data：结束时间；
  - --start-position：开始位置；
  - --stop-position：结束位置；

- filename：日志名；



例如：

```bash
# 恢复到 2018年1月30日 15：27：38 
$ mysqlbinlog --stop-date="2018-01-30 15:27:38" binlog.00008 | mysql -uroot -proot123
```





## 停止binlog

如果想永久停止binlog，需要修改配置文件关闭binlog功能，并重启mysql。

如果是想暂停binlog，可以通过 `SET  sql_log_bin` 来设置mysql变量的方式暂停：

```mysql
# 暂停binlog功能 
mysql> set sql_log_bin 0; 

# 恢复binlog功能 
mysql> set sql_log_bin 1;
```



<br>



# 错误日志

## 启动和设置错误日志

可以在mysql配置文件 `my.cnf `中配置错误日志的位置：

```bash
log-error=/var/log/mysqld.log
```



这样的话错误日志就会记录到 `/var/log/mysql.log` 下了。



也可以在mysql中查看设置的错误日志信息：

```mysql
mysql> show variables like 'log_error'; 
+---------------+---------------------+ 
| Variable_name | Value               | 
+---------------+---------------------+ 
| log_error     | /var/log/mysqld.log | 
+---------------+---------------------+
```





## 删除错误日志

错误日志可以直接删除，不会影响运行。在运行状态下删除错误日志后，mysql并不会重新创建日志。所以如果在删除错误日志后，如果需要重新创建，可以执行下面的命令：

```bash
$ mysqladmin -uroot -p flush-logs
```

<br>





# 通用查询日志

## 设置和启动通用查询日志

默认情况下，没有开启查询日志，通过修改 `my.cnf `文件来开启查询日志：

```bash
general_log=1 
general_log_file=/data/querylog/mysql-query.log
```



第一个参数表示将通用查询日志开启，第二个参数指定通用查询日志的位置。修改后重启mysql生效。



## 查看通用查询日志

这是一个文本文件，直接打开查看即可，里边记录了所有的执行指令和请求。

> 一般不打开这个日志，因为会导致文件太大；在调试排查问题的时候可以打开。



## 删除通用查询日志

该日志文件可以直接删除，或者使用下面的命令重新打开日志文件：

```bash
$ mysqladmin -uroot -p flush-logs
```

<br>



# 慢查询日志

## 启动和设置慢查询日志

通过修改配置文件，增加如下配置来开启慢查询日志：

```bash
slow_query_log=1 
slow_query_log_file=/data/slowlog/mysql-slow.log 
log_queries_not_using_indexes=on 
long_query_time=1
```

- `slow_query_log`：开启慢查询日志；
- `slow_query_log_file`：慢查询日志位置；
- `log_queries_not_using_indexes` ：是否记录没有使用索引的查询；
- `long_query_time`：超过这个时间的查询都定义为慢查询，单位秒，默认10秒；



## 查看慢查询日志

慢查询日志也是文本文件，直接打开查询即可。借助慢查询日志，可以对查询语句进行优化。



## 删除慢查询日志

该日志文件可以直接删除，或者使用下面的命令重新打开日志文件：

```bash
$ mysqladmin -uroot -p flush-logs
```




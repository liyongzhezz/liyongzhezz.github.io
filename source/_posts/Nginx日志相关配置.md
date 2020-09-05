---
title: Nginx日志相关配置
date: 2020-07-22 11:07:36
tags:
- Nginx
categories: Nginx
description: 设置nginx中的访问日志和错误日志
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1595397450412&di=ab410df2454377e6bae373a582f699bf&imgtype=0&src=http%3A%2F%2Fpic2.zhimg.com%2Fv2-2607f748a39504d37d8827306c335eb3_1200x500.jpg
---



## 日志分类

nginx中的日志分为两类：

- `access.log`：访问日志，记录了用户请求的一些信息；
- `error.log`：错误日志，记录了请求中以及服务本身发生的错误信息；



<br>



## 访问日志

用户的每一次请求都会记录在`access.log`中，包括：客户端IP，浏览器信息，referer，请求处理时间，请求URL等；



### 开启访问日志

访问日志的开启可以通过`access_log`参数来指定，其格式为：

```nginx
access_log path [format [buffer=size] [gzip[=level]] [flush=time] [if=condition]];
```

- path 指定日志的存放位置。
- format 指定日志的格式。默认使用预定义的combined。
- buffer 用来指定日志写入时的缓存大小。默认是64k。
- gzip 日志写入前先进行压缩。压缩率可以指定，从1到9数值越大压缩比越高，同时压缩的速度也越慢。默认是1。
- flush 设置缓存的有效时间。如果超过flush指定的时间，缓存中的内容将被清空。
- if 条件判断。如果指定的条件计算为0或空字符串，那么该请求不会写入日志。



例如：

```nginx
# 指定日志的写入路径为/var/logs/nginx-access.log，日志格式使用默认的combined。
access_log /var/logs/nginx-access.log

# 指定日志的写入路径为/var/logs/nginx-access.log，日志格式使用默认的combined；
# 指定日志的缓存大小为32k，日志写入前启用gzip进行压缩，压缩比使用默认值1，缓存数据有效时间为1分钟。
access_log /var/logs/nginx-access.log buffer=32k gzip flush=1m
```





### 关闭访问日志

访问日志关闭直接设置如下的参数即可：

```nginx
access_log off;
```



> 在合适的作用域下设置该参数，一般用在http，server，location，limit_except下。



<br>



### 设置日志格式

nginx可以使用`log_format`参数来指定日志的格式，例如：

```nginx
log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
```

> 这里设置了一个格式名为`main`的日志格式



设置好日志格式后，就可以在`access_log`中指定使用该日志格式：

```nginx
access_log  /var/log/nginx/access.log  main;
```



这样输出的日志就是如下的格式：

```nginx
112.195.209.90 - - [20/Feb/2018:12:12:14 +0800] 
"GET / HTTP/1.1" 200 190 "-" "Mozilla/5.0 (Linux; Android 6.0; Nexus 5 Build/MRA58N) 
AppleWebKit/537.36 (KHTML, like Gecko) Chrome/63.0.3239.132 Mobile Safari/537.36" "-"
```





`log_format`支持多种变量，常见的如下：

```nginx
# 发送给客户端的总字节数
$bytes_sent

# 发送给客户端的字节数，不包括响应头的大小
$body_bytes_sent

# 连接序列号
$connection

# 当前通过连接发出的请求数量
$connection_requests

# 日志写入时间，单位为秒，精度是毫秒
$msec

# 如果请求是通过http流水线发送，则其值为"p"，否则为“."
$pipe

# 完整的原始请求行，如 "GET / HTTP/1.1"
$request

# 请求长度（包括请求行，请求头和请求体）
$request_length

# 请求处理时长，单位为秒，精度为毫秒，从读入客户端的第一个字节开始，直到把最后一个字符发送张客户端进行日志写入为止
$request_time

# 完整的请求地址，如 "https://daojia.com/"
$request_uri

# 响应状态码
$status

# 标准格式的本地时间,形如“2017-05-24T18:31:27+08:00”
$time_iso8601

# 通用日志格式下的本地时间，如"24/May/2017:18:31:27 +0800"
$time_local

# 请求地址，即浏览器中输入的地址（IP或域名）
$http_host

# 请求的referer地址。
$http_referer

# 客户端浏览器信息。
$http_user_agent

# 当前端有代理服务器时，设置web节点记录客户端地址的配置，此参数生效的前提是代理服务器也要进行相关的x_forwarded_for设置。
$http_x_forwarded_for

# 客户端IP
$remote_addr

# 客户端用户名称，针对启用了用户认证的请求
$remote_user

# upstream状态
$upstream_status

# 后台upstream的地址，即真实服务器的地址
$upstream_addr

# 请求过程中，upstream响应时间
$upstream_response_time

# SSL协议版本
$ssl_protocol
```



<br>



## 错误日志

错误日志在Nginx中是通过error_log指令实现的。该指令记录服务器和请求处理过程中的错误信息。



### 开启错误日志

错误日志开启格式如下：

```nginx
error_log file [level];
```

- file指定错误日志存放位置；
- level指定错误日志级别；



> level可以是debug, info, notice, warn, error, crit, alert,emerg中的任意值（级别从低到高）。只有日志的错误级别等于或高于level指定的值才会写入错误日志中。默认值是error。



**错误日志不宜设置的太低，否则会产生大量的磁盘IO**



配置实例如下：

```nginx
# 错误日志存放在/var/log/nginx/error.log，级别为warn
error_log  /var/log/nginx/error.log warn;
```



### 关闭错误日志

关闭错误日志，将`error_log`注释即可。




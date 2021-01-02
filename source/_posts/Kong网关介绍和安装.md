---
title: Kong网关介绍和安装
date: 2021-01-02 15:02:43
tags:
- Kong
categories:
- 微服务网关
- Kong
description: Kong网关介绍和安装
cover: https://gimg2.baidu.com/image_search/src=http%3A%2F%2Fpic4.zhimg.com%2Fv2-25687fc3edae51f8c644355301196f3d_1440w.jpg%3Fsource%3D172ae18b&refer=http%3A%2F%2Fpic4.zhimg.com&app=2002&size=f9999,10000&q=a80&n=0&g=0n&fmt=jpeg?sec=1612163007&t=992eeb6bb71ba67a7595161b28268c0f
---



## 介绍

Kong网关是一款基于OpenResty（Nginx+Lua模块）编写的高可用、易扩展的，由Mashape公司开源的API Gateway项目。Kong是基于NGINX和Apache Cassandra或PostgreSQL构建的，能提供易于使用的RESTful API来操作和配置API管理系统，所以它可以水平扩展多个Kong服务器，通过前置的负载均衡配置把请求均匀地分发到各个Server，来应对大批量的网络请求。



Kong本身提供包括HTTP基本认证、密钥认证、CORS、TCP、UDP、文件日志、API请求限流、请求转发及Nginx监控等基本功能。



### 核心组件

- Kong Server：基于Nginx的服务器，用来接收API请求。
- Apache Cassandra/PostgreSQL：用来存储操作数据。
- Kong Dashboard：官方推荐UI管理工具，也可以使用开源的Konga平台。



Kong采用插件机制进行功能定制，当前本身已经具备了类似安全，限流，日志，认证，数据映射等基础插件能力，同时也可以很方便的通过Lua定制自己的插件。插件完全是一种可以动态插拔的模式，通过插件可以方便的实现Kong网关能力的扩展。



### 关键功能

- api注册和服务代理；
- 负载均衡；
- 认证鉴权；
- 安全和访问控制；
- 流量控制；
- 日志；



### kong和其他网关对比

![](./duibi.png)

从上面对比图也可以看到，Kong网关在功能，性能，插件可扩展性各方面都能够更好的满足企业API网关的需求。



<br>



## YUM方式部署kong

### 环境

操作系统：centos7

版本：kong 2.2

PostgreSQL：需要提前准备好数据库



### 添加yum源

```bash
$ wget https://bintray.com/kong/kong-rpm/rpm -O bintray-kong-kong-rpm.repo
$ export major_version=`grep -oE '[0-9]+\.[0-9]+' /etc/redhat-release | cut -d "." -f1`
$ sed -i -e 's/baseurl.*/&\/centos\/'$major_version''/ bintray-kong-kong-rpm.repo
$ mv bintray-kong-kong-rpm.repo /etc/yum.repos.d/
$ yum clean all && yum makecache
```



### 安装kong

```bash
$ yum install -y kong
```



### 数据库创建kong用户

```bash
$ su - postgres
$ psql -U postgres
 CREATE USER kong; 
 CREATE DATABASE kong OWNER kong;
 ALTER USER kong with encrypted password 'kong';  
```



### 修改配置文件

修改配置文件，让kong连接到PGsql：

```bash
$ cp -r /etc/kong/kong.conf.default /etc/kong/kong.conf

$ cat > /etc/kong/kong.conf << EOF
database = postgres
pg_host = 172.28.102.62
pg_port = 5432
pg_timeout = 5000
pg_user = kong
pg_password = kong
pg_database = kong
```



### 初始化数据库

```bash
$ kong migrations bootstrap [-c /etc/kong/kong.conf]
```



注意执行过程如遇到`failed to retrieve PostgreSQL server_version_num: FATAL: Ident authentication failed for user "kong"`，请给该用户设定密码，并修改postgres的授权方式为MD5，操作如下：

```bash
$ alter user kong with password  'kong';
```



### 启动/停止服务

启动kong：

```bash
$ kong start [-c /etc/to/kong.conf]
```



检查服务是否运行：

```bash
$ curl -i http://localhost:8001/
# 或者
$ kong health
```



停止服务：

```bash
$ kong stop
```



### 安装kong dashboard

首先安装nodejs：

```bash
$ wget http://nodejs.org/dist/v8.1.0/node-v8.1.0.tar.gz 
$ tar zxvf node-v8.1.0.tar.gz 
$ cd node-v8.1.0
$ ./configure –prefix=/usr/local/node/*
$ make
$ make install
$ ln -s /usr/local/node/bin/* /usr/sbin/  

#npm 配置
$ npm set prefix /usr/local
$ echo -e '\nexport PATH=/usr/local/lib/node_modules:$PATH' >> ~/.bashrc
$ source ~/.bashrc
```



安装dashboard：

```bash
$ git clone https://github.com/PGBI/kong-dashboard

#安装Kong Dashboard:
$ npm install -g kong-dashboard@v2

#启动时候可以自己指定端口号如9001
$ kong-dashboard start -p 9001
```



启动之后，访问本地的9001端口即可打开kong页面


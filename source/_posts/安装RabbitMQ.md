---
title: 安装RabbitMQ
date: 2020-08-09 10:21:53
tags:
- RabbitMQ
categories:
- 消息中间件
- RabbitMQ
description: 在Linux上部署RabbitMQ服务
cover: https://ss0.bdstatic.com/70cFuHSh_Q1YnxGkpoWK1HF6hhy/it/u=2897651972,890317148&fm=26&gp=0.jpg
---



# 下载

在官网 [rabbitmq官网](https://www.rabbit.com) 找到合适的版本下载，这里使用的是`3.6.5`：

```bash
$ wget http://www.rabbitmq.com/releases/rabbitmq-server/v3.6.5/rabbitmq-server-3.6.5-1.noarch.rpm
```



 RabbitMQ依赖于Erlang，可以在 [Erlang版本对照](https://www.rabbitmq.com/which-erlang.html) 找到对应的Erlang版本，这里我应该下载`18.3`版本：

```bash
$ wget http://www.rabbitmq.com/releases/erlang/erlang-18.3-1.el7.centos.x86_64.rpm
```



<br>



# 安装

安装依赖：

```bash
$ yum install -y openssl openssl-devel make gcc gcc-c++ kernel-devel socat
```



安装rabbitmq：

```bash
# 先安装erlang
$ rpm -ivh erlang-18.3-1.el7.centos.x86_64.rpm

# 再安装rabbitmq
$ rpm -ivh rabbitmq-server-3.6.5-1.noarch.rpm
```



<br>



# 编辑配置文件

安装完成后默认配置文件在：`/usr/lib/rabbitmq/lib/rabbitmq_server-3.6.5/ebin/rabbit.app`，修改这个文件来设置密码，修改配置等：

```bash
# 主要关注env模块的配置

# 监听端口，默认5672
{tcp_listeners, [5672]}

# 只允许本地访问的用户
loopback_users, [guest]}
```



<br>



# 启动服务

```bash
# 启动服务
$ rabbitmq-server start &

# 后台启动
$ rabbitmq-server -detached

# 停止服务
$ rabbitmqctl app_stop
```



会启动如下的几个端口：

- 5672：java程序进行连接的端口；
- 15672：控制台的端口（后面会安装）；
- 25672：集群通信端口；



<br>



# 启动管理控制台

```bash
$ rabbitmq-plugins enable rabbitmq_management
```



插件启动成功后，默认监听15672，通过浏览器访问该端口即可看到默认的登录界面，可以使用`guest/guest`登录：

![](index.png)



<br>



# 常用命令

```bash
# 查看队列
$ rabbitmqctl list_queues

# 查看虚拟主机
$ rabbitmqctl list_vhosts

# 查看节点状态
$ rabbitmqctl status

# 新建用户
$ rabbitmqctl add_user <用户名> <密码>

# 设置用户为管理员
$ rabbitmqctl set_user_tags <用户名> administrator

# 赋予用户所有权限
$ rabbitmqctl set_permissions -p / <用户名> ‘。*’ ‘。*’ ‘。*’
```


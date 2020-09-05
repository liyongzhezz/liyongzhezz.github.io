---
title: Nginx基础概念
date: 2020-07-03 16:33:07
tags:
- Nginx
categories:
- Nginx
description: 介绍nginx基本概念和工作原理
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1596540192624&di=bbd964cce4cea1709707e02e51dc38ce&imgtype=0&src=http%3A%2F%2Fwww.otcms.cn%2Fupfiles%2Finfoimg%2Fcoll%2Fot20190122232426541.jpg
---





# 介绍

nginx是目前最火热的web服务器和负载均衡服务器，它能反向代理HTTP, HTTPS, SMTP, POP3, IMAP的协议链接，同时也可以作为一个缓存服务器。



<br>



# 与apache相比

Nginx：

- IO 多路复用，Epoll（freebsd 上是 kqueue）
- 高性能
- 高并发
- 占用系统资源少


Apache：

- 阻塞+多进程/多线程
- 更稳定，Bug 少
- 模块更丰富



<br>



# nginx工作模型



![](nginx-process.png)



Nginx 服务器，正常运行过程中：

- 多进程：一个 Master 进程、多个 Worker 进程。
- Master 进程：管理 Worker 进程。接收外部的操作（信号）；根据外部的操作的不同，通过信号管理 Worker；监控 Worker 进程的运行状态，Worker 进程异常终止后，自动重启 Worker 进程。
- Worker 进程：所有 Worker 进程都是平等的。网络请求由 Worker 进程处理。Worker 进程数量在 nginx.conf 中配置，一般设置为核心数，充分利用 CPU 资源，同时，避免进程数量过多，避免进程竞争 CPU 资源，增加上下文切换的损耗。

<br>



# 工作细节

master工作细节：

1. master建立listen的socket；
2. master fork出worker进程；
3. 新请求到来时，worker间相互竞争，获胜的获得listenfd可读时间；
4. 获胜的worker accept当前连接，处理请求；



worker工作细节：

1. 新请求到来时，所有worker的listenfd都变为可读的；
2. worker间相互竞争，获胜的获得listenfd可读时间；
3. worker处理并响应请求；




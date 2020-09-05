---
title: 升级Nginx
date: 2020-07-05 15:22:46
tags:
- Nginx
categories: Nginx
description: 两种升级Nginx服务的方式
cover: https://ss1.bdstatic.com/70cFuXSh_Q1YnxGkpoWK1HF6hhy/it/u=1375552705,3642822908&fm=26&gp=0.jpg
---



## 停机升级

停机升级的过程就是直接将新版本的二进制文件进行替换，然后重启nginx进程即可。



<br>



## 平滑升级

平滑升级可以做到不停机升级，升级步骤如下：

1. 发送USR2信号通知旧版本nginx准备升级：`kill -s SIGUSR2 <nginx master pid>`；此时运行中的nginx会将pid重命名；
2. 启动新版本的nginx（任意方式启动），通过命令确认新版启动成功；
3. 向旧版本发送QUIT信号优雅终止旧版本：`kill -s SIGQUIT <nginx master old pid>`；


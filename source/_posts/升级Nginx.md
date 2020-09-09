---
title: 升级Nginx
date: 2020-07-05 15:22:46
tags:
- Nginx
categories: Nginx
description: 两种升级Nginx服务的方式
cover: https://ss1.bdstatic.com/70cFuXSh_Q1YnxGkpoWK1HF6hhy/it/u=1375552705,3642822908&fm=26&gp=0.jpg
---



# 停机升级

停机升级的过程就是直接将新版本的二进制文件进行替换，然后重启nginx进程即可。



<br>



# 平滑升级

## 确认旧版本

首先通过命令确定旧版本nginx进程存在：

```bash
$ ps aux | grep nginx
```



![](1.png)



## 编译安装新版本

首先获取旧版本的编译选项：

```bash
$ nginx -V
```

![](2.png)



找到编译命令后，用这个命令编译新版本：

```bash
$ wget http://nginx.org/download/nginx-1.18.0.tar.gz
$ mv nginx-1.18.0.tar.gz /usr/local
$ cd /usr/local
$ tar xf nginx-1.18.0.tar.gz
$ cd /usr/local/nginx-1.18.0
$ ./configure --prefix=/usr/local/nginx \
              --with-http_ssl_module \
              --with-http_v2_module \
              --with-http_realip_module \
              --with-http_addition_module \
              --with-http_image_filter_module \
              --with-http_geoip_module \
              --with-http_gunzip_module \
              --with-http_stub_status_module \
              --with-http_gzip_static_module \
              --with-pcre --with-stream \
              --with-stream_ssl_module \
              --with-stream_realip_module
$ make 
```

> 注意`--prefix`指向的是旧版本的目录，编译后先不`make install`



## 备份原来的二进制程序

```bash
$ cd /usr/local/nginx/sbin
$ cp nginx nginx.bak
$ ./nginx -V
```

![](3.png)



## 更新二进制程序

在下载的新版本源码包中找到新版本的二进制程序：

```bash
$ cd /usr/local/nginx-1.18.0/objs

# 用新版本覆盖旧版本
$ cp -f nginx /usr/local/nginx/sbin/nginx
```



## 旧版本平滑停止请求

当前还是旧版本的在运行，查看进程信息：

```bash
$ ps aux | grep nginx
```



![](1.png)

> 可以看到，父进程的id为17400



向父进程发送`USER2`信号，让新的子进程接收请求，旧的子进程不再接收新请求：

```bash
$ kill -USER2 17400
```



此时查看进程信息，发现同时存在新旧版本：

```bash
$ ps aux | grep nginx
```

![](4.png)

> 尽管旧进程在监听，但不会处理新的请求。



## 关闭旧进程

```bash
$ kill -WINCH 17440 
```



现在查看进程信息：

```bash
$ ps aux | grep nginx
```

![](5.png)

> 现在只剩下新版本的nginx进程了



查看当前的版本：

```bash
$ nginx -V
```

![](6.png)

<br>


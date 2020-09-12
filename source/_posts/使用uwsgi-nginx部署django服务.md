---
title: 使用uwsgi+nginx部署django服务
date: 2020-09-12 14:39:51
tags:
- Django
- uwsgi
categories:
- python web开发
- Django
- 服务部署
description: 使用uwsgi和nginx结合部署django服务
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1599902999809&di=6306855101fdb4e6f595c38dfce7f8dc&imgtype=0&src=http%3A%2F%2Fblog.demon8.cn%2Fmedia%2Farticle_img%2F2019%2F12%2F12%2Fw1c30rf061.jpeg
---



# uwsgi介绍

## wsgi

WSGI，全称 `Web Server Gateway Interface`，或者` Python Web Server Gateway Interface` ，是为 Python 语言定义的 Web 服务器和 Web 应用程序或框架之间的一种简单而通用的接口，是一个Gateway，在协议之间进行转换。



WSGI 是作为 Web 服务器与 Web 应用程序或应用框架之间的一种低级别的接口，以提升可移植 Web 应用开发的共同点。WSGI 是基于现存的 CGI 标准而设计的。

很多框架都自带了 WSGI server ，比如 Flask，webpy，Django、CherryPy等等。当然性能都不好，自带的 web server 更多的是测试用途，发布时则使用生产环境的 WSGI server或者是联合 nginx 做 uwsgi 。



## uwsgi

uWSGI是一个Web服务器，它实现了WSGI协议、uwsgi、http等协议。Nginx中`HttpUwsgiModule`的作用是与uWSGI服务器进行交换。

要注意 WSGI / uwsgi / uWSGI 这三个概念的区分。

- WSGI看过前面小节的同学很清楚了，是一种通信协议。
- uwsgi同WSGI一样是一种通信协议。
- 而uWSGI是实现了uwsgi和WSGI两种协议的Web服务器。

uwsgi协议是一个uWSGI服务器自有的协议，它用于定义传输信息的类型（type of information），每一个uwsgi packet前4byte为传输信息类型描述，它与WSGI相比是两样东西。

为什么有了uWSGI为什么还需要nginx？因为nginx具备优秀的静态内容处理能力，然后将动态内容转发给uWSGI服务器，这样可以达到很好的客户端响应。





<br>





# 安装uwsgi

在python3环境下安装uwsgi：

```bash
$ yum install -y epel-release uwsgi-plugin-python36
```



<br>



# 通过uwsgi启动服务

## 创建虚拟环境

创建一个python虚拟环境：

```bash
$ pip install virtualenv
$ virtualenv  venv
```



## 安装依赖包

```bash
$ source venv/bin/activate
$ pip install -r requirements.txt
$ pip install uwsgi
```



## 创建uwsgi配置文件

在django项目的根目录下创建一个名为`uwsgi.ini`的配置文件：

```ini
[uwsgi]
# django项目的根目录
chdir=/root/src/

# 模块名，格式为：<django项目名称>.wsgi:application
module=miniproject.wsgi:application

# wsgi文件相对位置
wsgi-file=miniproject/wsgi.py

# wsgi启动后监听的地址
socket=0.0.0.0:7000

# websocket允许的最大字节
websocket-max-size = 2048

# 启动的进程数
workers=8

# 每个worker中的线程数
threads= 8

enable-threads=true

# 启动的用户和组
uid=root
gid=root

# 禁用请求日志记录
disable-logging = false
thunder-lock=true
post-buffering=4096

# 指定的插件
plugins=python36

# 虚拟环境的位置
home=/root/venv
```



## 启动uwsgi

使用下面的命令直接启动uwsgi：

```bash
$ uwsgi --ini uwsgi.ini
```



<br>



# 配置nginx

通过nginx代理uwsgi，配置文件如下：

```nginx
upstream uwsgiapi {
  server 127.0.0.1:7000;
}

server {
      listen 9000;

       location /miniProj/segment/querySegment {
           include /usr/local/nginx/conf/uwsgi_params;
           uwsgi_pass miniapi;
           proxy_buffer_size 128k;
           proxy_buffering on;
           proxy_buffers 4 128k;
           proxy_busy_buffers_size 256k;
           proxy_max_temp_file_size 256k;
           proxy_http_version 1.1;
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection "upgrade";
           uwsgi_read_timeout 86400s;
           uwsgi_send_timeout 86400s;
           client_max_body_size    1000m;
           proxy_set_header X-Real-IP $remote_addr;
       }

       location = /favicon.ico {
           log_not_found off;
           access_log off;
       }

       location ~* /\.(svn|git)/ {
           return 404;
       }
}
```


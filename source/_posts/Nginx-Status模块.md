---
title: Nginx Status模块
date: 2020-07-29 11:03:46
tags:
- Nginx
categories: 
- Nginx
- 常用模块
description: 使用nginx status模块简单监控nginx状态
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1596002098189&di=31ed01979fca9ba000275ac1005240c0&imgtype=0&src=http%3A%2F%2Fdata.useit.com.cn%2Fuseitdata%2Fforum%2F201510%2F20%2F183829k5771z2jqs2ssuuh.jpg
---



`http_stub_status_module`可以监控nginx的状态，其在yum安装时会默认加入，在源码安装时需要手动加入编译。



------



# 作用

这个模块的作用是展示nginx当前处理连接的状态，常用于监控nginx。

<br>



# 配置

格式：

```nginx
location /status {
    stub_status;
}
```

**该模块需要配置在server中的location下。**



例如：

```nginx
server {
    listen       80;
    server_name  localhost;

    location /status {
        stub_status;
    }

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

<br>



# 生效配置

```bash
$ nginx -t 
$ nginx -s reload 
```



通过浏览器访问`location`匹配的模块路径，例如：`http://192.168.221.201/status`，即可看到如下信息：



<img src="status.png" style="zoom:130%;" />



- `Active connections`：当前活跃的连接数；
- `accepts`：nginx接收的总共的握手次数，这里是9；
- `handled`：处理的连接数，这里是9；
- `requests`：处理的请求数，这里是5；
- `Reading`：正在读的连接数；
- `Writing`：正在写的连接数；
- `Waiting`：等待连接个数；



> 正常情况下，握手次数和连接次数应该相等，表示没有丢失请求；




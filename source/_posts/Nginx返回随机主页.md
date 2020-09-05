---
title: Nginx返回随机主页
date: 2020-07-29 11:13:28
tags:
- Nginx
categories: 
- Nginx
- 常用模块
description: 让nginx返回随机的主页内容
cover: https://ss0.bdstatic.com/70cFuHSh_Q1YnxGkpoWK1HF6hhy/it/u=4160398481,520081383&fm=11&gp=0.jpg
---



`--with-http_random_index_module`模块可以返回一个随机主页，yum安装时会默认加入，在源码安装时需要手动加入编译。



------



# 作用

在主目录中随机选择一个文件作为随机主页。

<br>





# 配置

**该模块需要配置在server中的location下。**



格式：

```nginx
location / {
    random_index on;
}
```



> 默认 `random_index`为`off`。



例如：

```nginx
server {
    listen       80;
    server_name  localhost;

    location / {
        root   /usr/share/nginx/html;
        random_index on;
        #index  index.html index.htm;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```



跟新配置后，作为测试，需要在根目录下创建几个静态文件作为测试：

```bash
$ touch /usr/share/nginx/html/{1,2,3}.html
$ echo this is index 1 > /usr/share/nginx/html/1.html
$ echo this is index 2 > /usr/share/nginx/html/2.html
$ echo this is index 3 > /usr/share/nginx/html/3.html
```



<br>



# 生效配置

```nginx
$ nginx -t
$ nginx -s reload
```



通过浏览器访问首页，并不断刷新，可以看到3个页面会随机出现。

![](random.png)





<br>


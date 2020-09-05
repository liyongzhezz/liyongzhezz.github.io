---
title: Nginx替换响应内容
date: 2020-07-29 14:58:54
tags:
- Nginx
categories:
- Nginx
- 常用模块
description: 使用--with-http_sub_module模块对响应内容进行替换
cover: https://ss0.bdstatic.com/70cFvHSh_Q1YnxGkpoWK1HF6hhy/it/u=3968007614,1554103190&fm=26&gp=0.jpg
---



`--with-http_sub_module`可以在服务端响应时进行html内容的替换，在yum安装时会默认加入，在源码安装时需要手动加入编译。



------



# 作用

用于服务端在响应客户端http请求时进行内容替换。

<br>





# 配置

**该模块可以配置在http、server或location下，如果配置在http下，则可以完成对多个server的替换。**



格式：

```nginx
location / {
	sub_filter_last_modified on;
	sub_filter_once on;
	sub_filter <string> <replacement>;
}
```



> `sub_filter_last_modified`用于让客户端判断服务端相应的内容是否发生变化，默认情况下，为`off`；
>
> `sub_filter_once`在进行html替换时，如果为`on`，则只匹配到第一个满足条件的字符串，`off`匹配所有，默认为`on`；



例如：

```nginx
# 将返回的HTML中的jeson替换为JESON
location / {
	root   /usr/share/nginx/html;
	sub_filter_last_modified on;
	sub_filter_once on;
	sub_filter '<a>jeson' '<a>JESON';
}
```



<br>



# 测试

首先创建一个html测试文件：

```bash
cat > /usr/share/nginx/html/submodule.html << EOF
<html>
<head>
  <meta charset="utf-8">
  <title>submodule</title>
</head>
<body>
  <a>jeson</a>
  <a>at</a>
  <a>peter</a>
  <a>jeson</a>
</body>
</html>
EOF
```



然后在nginx配置中首先添加一个替换条件：

```nginx
location / {
	root   /usr/share/nginx/html;
	sub_filter_last_modified on;
	sub_filter '<a>jeson' '<a>JESON';
}
```



重新加载配置后请求，发现第一个匹配到的`jeson`被替换了：

```bash
$ nginx -t
$ nginx -s reload 

# 发起请求
$ curl localhost/submodule.html
<html>
<head>
  <meta charset="utf-8">
  <title>submodule</title>
</head>
<body>
  <a>JESON</a>
  <a>at</a>
  <a>peter</a>
  <a>jeson</a>
</body>
</html>
```



如果需要替换所有匹配到的，则需要添加`sub_filter_once`参数：

```nginx
location / {
	root   /usr/share/nginx/html;
	sub_filter '<a>jeson' '<a>JESON';
	sub_filter_once off;
	sub_filter_last_modified on;
}
```



重新加载配置后，发现所有匹配到的都被替换了：

```bash
$ nginx -t
$ nginx -s reload 

# 发起请求
$ curl localhost/submodule.html
<html>
<head>
  <meta charset="utf-8">
  <title>submodule</title>
</head>
<body>
  <a>JESON</a>
  <a>at</a>
  <a>peter</a>
  <a>JESON</a>
</body>
</html>
```



<br>
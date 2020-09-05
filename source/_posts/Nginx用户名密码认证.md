---
title: Nginx用户名密码认证
date: 2020-07-29 15:05:49
tags: Nginx
categories: 
- Nginx
- 常用模块
description: nginx基于用户名密码的认证
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1596016589978&di=3faa7ad2ca5d23c5bf55d775db6b2dc2&imgtype=0&src=http%3A%2F%2Fwww.nd9p.com%2Fuploads%2Fallimg%2F160529%2F2143353111_0.jpg
---



`http_auth_basic_module`是用来做用户登录认证的模块。



------



# 依赖工具安装

该模块一般配合htpasswd命令使用，htpasswd命令用于生成用户名密码的配置文件，安装方式如下：

```bash
$ yum install -y httpd-tools
```

<br>



# 生成用户名密码文件

使用该命令生成一个包含用户名和密码的配置文件：

![](htpass.png)

> 第一次使用`-c`命令表示创建文件，直接接上用户名即可。

<br>





# 设置nginx配置文件

修改nginx的配置文件如下：

```nginx
location ~ ^/admin.html {
  root /usr/share/nginx/html;
  index index.html index.htm;
  auth_basic "Auth access is need!!!";
  auth_basic_user_file /etc/nginx/auth_conf;
}
```

- `auth_basic`是在验证的时候的提示信息；
- `auth_basic_user_file`保存用户信息的文件；

<br>



# 存在的问题

该方式存在局限性：

- 依赖文件的方式存储用户信息，效率低；
- 无法和其他系统打通用户密码，维护成本高；



解决方案：

- nginx+lua解决
- nginx+ldap，利用nginx-auth-ldap模块实现用户信息打通


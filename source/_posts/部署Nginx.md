---
title: 部署Nginx
date: 2021-04-13 08:38:54
tags:
- Nginx
categories:
- Nginx
- 部署
description: 使用yum和源码两种方式部署nginx服务
cover: https://gimg2.baidu.com/image_search/src=http%3A%2F%2Fwww.nd9p.com%2Fuploads%2Fallimg%2F160529%2F2143353111_0.jpg&refer=http%3A%2F%2Fwww.nd9p.com&app=2002&size=f9999,10000&q=a80&n=0&g=0n&fmt=jpeg?sec=1620866496&t=e367c0adebb6cf217109959821f24a66
---



{% note info 'fas fa-bullhorn' %}

本文主要介绍了使用yum和源码两种方式部署nginx服务

更新于 2021-04-13

{% endnote %}

<br>



# yum方式部署

## 安装依赖

```bash
yum install -y gcc gcc-c++ autoconf pcre pcre-devel make automake yum-utils zlib zlib-devel openssl openssl-devel
```



 有些工具包也可以选择安装：

```bash
yum install -y wget http-tools vim
```

‌

## 添加yum源

```bash
cat > /etc/yum.repos.d/nginx.repo << EOF
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/\$releasever/\$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true

[nginx-mainline]
name=nginx mainline repo
baseurl=http://nginx.org/packages/mainline/centos/\$releasever/\$basearch/
gpgcheck=1
enabled=0
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true
EOF
```

‌

## 安装nginx

```bash
# 查看nginx版本
yum list | grep nginx

# 安装最新版
yum install -y nginx
```



## 查看nginx版本和编译参数

```bash
# 查看nginx版本
nginx -v

# 查看nginx编译参数
nginx -V
```



## 默认安装的目录和文件

使用下面的命令可以查看nginx通过rpm方式安装的目录结构和文件：

```bash
rpm -ql nginx
```



重要的配置文件如下：

- `/etc/logrotate.d/nginx`：nginx配置logrotate日志切割的配置文件；
- `/etc/nginx`：安装目录；
- `/etc/nginx/nginx.conf`：主配置文件；
- `/etc/nginx/conf.d`：其他配置存放的目录；
- `/etc/nginx/conf.d/default.conf`：默认加载的配置；
- `/etc/nginx/fastcgi_params`：fastcgi配置文件；
- `/etc/nginx/scgi_params`：scgi配置文件；
- `/etc/nginx/uwsgi_params`：uwsgi配置文件；
- `/etc/nginx/{koi-win, koi-utf, win-utf}`：编码转换配置文件；
- `/etc/nginx/mime.types`：设置http协议的Content-Type与扩展名对应关系的配置文件；
- `/var/cache/nginx`：用于缓存的目录；
- `/var/log/nginx`：日志目录；



## nginx启动

```bash
# 启动
systemctl start nginx

# 开机自启动
systemctl enable nginx

# 查看nginx状态
systemctl status nginx

# 停止nginx
systemctl stop nginx

# 重启
systemctl restart nginx
```

<br>



# 源码方式部署

## 安装依赖

```bash
yum install -y gcc gcc-c++ autoconf pcre pcre-devel make automake yum-utils zlib zlib-devel openssl openssl-devel
```



## 下载源码包

这里我使用的是1.18.0版本

```bash
wget http://nginx.org/download/nginx-1.18.0.tar.gz
```



## 编译安装

```bash
# 添加用户和组
groupadd nginx
useradd -g nginx nginx 

# 编译安装
tar zxf nginx-1.18.0.tar.gz
cd nginx-1.18.0
./configure \
--user=nginx \
--group=nginx \
--prefix=/usr/local/nginx \
--with-http_ssl_module \
--with-http_stub_status_module \
--with-http_realip_module \
--with-threads

make && make install

# 设置环境变量
echo "export PATH=$PATH:/usr/local/nginx/sbin" >> /etc/profile
source /etc/profile
```

> 根据实际需求增减编译的模块



## 检查

```bash
nginx -v
nginx -V
```



![](./install_source.png)



## 设置启动文件

```bash
cat > /usr/lib/systemd/system/nginx.service << EOF
[Unit]
Description=nginx service
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/nginx/sbin/nginx
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s quit
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF
```



## 启动nginx

```bash
systemctl daemon-reload
systemctl start nginx
systemctl enable nginx
systemctl status nginx
```



## 安装的文件

在编译的时候指定了nginx的安装目录为：`--prefix=/usr/local/nginx`，安装后会有以下文件：

- `conf`：配置文件目录；
  - `fastcgi.conf`：fastcgi配置文件；
  - `mime.types`：设置http协议的Content-Type与扩展名对应关系的配置文件；
  - `uwsgi_params`：uwsgi配置文件；
  - `koi-win, koi-utf, win-utf`：编码转换配置文件；
  - `scgi_params`：scgi配置文件；
  - `fastcgi_params`：fastcgi配置文件；
  - `nginx.conf`：nginx主配置文件；
- `html`：默认的静态文件根目录；
- `sbin`：二进制命令文件目录；


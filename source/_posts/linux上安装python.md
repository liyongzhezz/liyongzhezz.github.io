---
title: linux上安装python
date: 2021-11-21 14:45:46
tags:
- Python
categories:
- 编程
- Python
- 安装部署
description: 在Linux上安装python3
cover: https://ss2.bdstatic.com/70cFvnSh_Q1YnxGkpoWK1HF6hhy/it/u=185678737,1186265065&fm=26&gp=0.jpg
---





>  这里以安装`python3.7.3`版本为例。



安装依赖：

```bash
$ yum install -y zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel gcc  libffi-devel
```



下载并解压python包：

```bash
wget https://www.python.org/ftp/python/3.7.3/Python-3.7.3.tgz 
tar zxf Python-3.7.3.tgz
cd Python-3.7.3
```



编译安装：

```bash
./configure --prefix=/usr/local/python3.7
make && make install
```



备份原来的python：

```bash
mv /usr/bin/python /usr/bin/python.bak
```



创建软连接：

```bash
ln -s /usr/local/python3.7/bin/python3.7 /usr/bin/python
ln -s /usr/local/python3.7/bin/pip3 /usr/local/bin/
```



验证，使用如下命令，如果输出版本为3.7.3则安装成功：

```bash
python -V
```





修改yum文件：

编辑`/usr/bin/yum`和`/usr/libexec/urlgrabber-ext-down`，将文件头的`#!/usr/bin/python`改为`#!/usr/bin/python2`即可。






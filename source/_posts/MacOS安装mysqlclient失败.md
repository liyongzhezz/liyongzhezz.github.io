---
title: MacOS安装mysqlclient失败
date: 2020-07-31 17:52:30
tags:
- Django
categories:
- python web开发
- Django
- 常见问题
description: 在macos系统上安装mysqlclient报错的解决方案
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1596199327501&di=791911d536ed749b8a8e94e3f871c198&imgtype=0&src=http%3A%2F%2Fwww.west.cn%2Finfo%2Fupload%2F20180713%2Fddb20ffkvi2.jpg
---



在mac上的django项目中安装mysqlclient时出现如下的报错：

`ld: library not found for -lssl`



这个问题可能是两个原因导致的：

1、没有安装openssl，可以使用如下的方式安装：

```shell
$ brew install openssl
$ pip install mysqlclient
```



2、pip寻找依赖机制问题，需要指定openssl库：

```shell
$ LDFLAGS=-L/usr/local/opt/openssl/lib pip install mysqlclient
```






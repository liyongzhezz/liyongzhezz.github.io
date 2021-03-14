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



还有一种可能是报这种错误：

```bash
In file included from MySQLdb/_mysql.c:29:
    In file included from /usr/local/opt/mysql-client/include/mysql/mysql.h:45:
    In file included from /Library/Developer/CommandLineTools/usr/lib/clang/12.0.0/include/stdint.h:52:
    In file included from /Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/stdint.h:53:
    In file included from /Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/sys/_types/_intptr_t.h:30:
    /Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/machine/types.h:37:2: error: architecture not supported
    #error architecture not supported

```



原因是xcode 12编译了 ARM64 版本的二进制文档导致了异常的抛出，使用下面方式解决：

```bash
$ ARCHFLAGS="-arch x86_64"   pip install mysqlclient
```


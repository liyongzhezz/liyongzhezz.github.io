---
title: 部署nfs存储
date: 2021-03-23 21:38:44
tags:
- NFS
categories:
- 存储
- NFS
- 部署
description: 在Linux服务器上部署NFS存储
cover: https://ss1.bdstatic.com/70cFvXSh_Q1YnxGkpoWK1HF6hhy/it/u=773960014,3839457367&fm=26&gp=0.jpg
---



{% note info 'fas fa-bullhorn' %}

本文主要介绍了什么是NFS存储并且在linux服务器上部署nfs服务

更新于 2021-03-21

{% endnote %}

<br>



# NFS介绍

NFS是`Network File System`的缩写，即网络文件系统。功能是通过网络让不同的机器、不同的操作系统能够彼此分享文件，让应用程序在客户端通过网络访问位于服务器磁盘中的数据，是在类Unix系统间实现磁盘文件共享的一种方法。



NFS使用RPC协议进行通信，NFS可以看作是一个RPC Server，主要功能是管理需要分享的目录和文件，它不负责通信和信息传输，而是把这部分工作交给RPC协议来完成。即NFS在文件传送或信息传送过程中依赖于RPC协议。所以只要用到NFS的地方都要启动RPC服务，不论是NFS SERVER或者NFS CLIENT。这样SERVER和CLIENT才能通过RPC来实现PROGRAM PORT的对应。



>  可以这么理解RPC和NFS的关系：NFS是一个文件系统，而RPC是负责负责信息的传输。



<br>



# 部署NFS

NFS依赖于`nfs-utils`和`rpcbind`，所以先安装这两个软件包：

```bash
yum install -y nfs-utils rpcbind
```



创建nfs数据目录：

```bash
mkdir /nfs-data
```

> 一般这个目录会挂载一个数据盘



NFS共享存储需要将存储的地址配置在`/etc/exporters`下，例如这里配置为：

```bash
/nfs-data *(rw,no_root_squash)
```

> 星号表示允许所有客户端访问，支持对客户端IP地址和网段进行限制



其中支持的参数为：

- `ro`：只读；
- `rw`：读写；
- `root_squash`：当NFS客户端以root管理员访问时，映射为NFS服务器的匿名用户；
- `no_root_squash`：当NFS客户端以root管理员访问时，映射为NFS服务器的root管理员；
- `all_squash`：无论NFS客户端使用什么账户访问，均映射为NFS服务器的匿名用户；
- `sync`：同时将数据写入到内存与硬盘中，保证不丢失数据；
- `async`：优先将数据写入到内存，然后再写入硬盘；这样效率更高，但可能会丢失数据；



启动服务：

```bash
systemctl start rpcbind
systemctl enable rpcbind
systemctl start nfs-server
systemctl enable nfs-server
```



其他客户端想要使用nfs存储，则首先需要安装`nfs-utils`，然后可以使用下面的命令将nfs的目录挂载到本地：

```bash
yum install -y nfs-utils
mount -t nfs <nfs服务器地址>:/nfs-data /nfsdata
```


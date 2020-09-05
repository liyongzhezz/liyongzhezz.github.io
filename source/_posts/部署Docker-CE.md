---
title: 部署Docker CE
date: 2020-07-11 19:55:44
tags:
- Docker
categories: Docker
description: 使用yum方式部署Docker CE
cover: https://ss0.bdstatic.com/70cFuHSh_Q1YnxGkpoWK1HF6hhy/it/u=4156517757,536988701&fm=26&gp=0.jpg

---



**我选择的docker-ce版本为18.06，docker在拆分为ce和ee之后，大版本例如18表示发布的年份，小版本表示的是月份。**



------



# 卸载旧版docker

```bash
$ yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-selinux docker-engine-selinux docker-engine
```

<br>



# 安装依赖

```bash
$ yum install -y yum-utils device-mapper-persistent-data lvm2
```

<br>





# 添加docker源

```bash
$ yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

<br>



# 【可选】挂载数据目录

一般docker会把数据存储在`/var/lib/docker`下，所以如果是生产环境，建议挂载一个数据盘到该目录下防止爆盘。

<br>





# 安装Docker CE

## 查看可用版本

```bash
$ yum list docker-ce --showduplicates|sort -r
```

‌

## 安装最新版本

```bash
$ yum install -y docker-ce
```

‌

## 安装指定版本

```bash
$ yum install -y docker-ce-18.06.1.ce-1.el7.centos
```



<br>



#  启动docker

```bash
$ systemctl start docker
$ systemctl enable docker
```

<br>



# 验证安装

```bash
# 查看docker server和client的版本
$ docker version
```

<img src="docker-version.png" alt="img" style="zoom:50%;" />



<br>



# 卸载docker

首先卸载docker软件包：

```bash
$ yum remove docker-ce
```



然后需要手动删除docker镜像、容器等相关文件：

```bash
$ rm -rf /var/lib/docker
```


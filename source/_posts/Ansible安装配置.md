---
title: Ansible安装配置
date: 2020-07-19 12:03:53
tags:
- 自动化运维工具
- ansible
categories:
- 自动化运维
- Ansible
description: ansible介绍和安装配置
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1595141641851&di=f7535d2ca33c2d72c7883c7101c85142&imgtype=0&src=http%3A%2F%2Fimg4.imgtn.bdimg.com%2Fit%2Fu%3D722752994%2C888024066%26fm%3D214%26gp%3D0.jpg
---



## ansible介绍

### 什么是ansible

ansible是一个基于python开发的自动化运维工具，其集成了丰富的模块和组件，并且通过一个命令行工具完成对批量服务器的运维操作。

> ansible依靠各种模块完成部署任务



ansible常见的可以完成如下的几类任务：

- 系统配置；
- 基于ansible开发持续集成和跳板机等复杂软件；
- 编排高级的任务（基于playbook）；



ansible具有如下的特点：

- 轻量级：基于一个命令行工具就可以完成大量的工作；
- 简单易用：文档丰富，社区模块多；
- 操作灵活：命令行选项丰富，playbook提供复杂且丰富的功能；



### 模块和框架

Ansible主要由两大模块构成：

- `Paramiko`：一个纯Python实现的ssh协议库，因此，Ansible不需要再远程主机上安装client或agents，因为它是基于ssh来和远程主机通信的；
- `PyYAML`：是YAML的Python实现，可以用于参数化Python对象，用来当做配置文件；



Ansible框架主要包括 ：

- 连接插件connection plugins：负责和被监测端通信。
- host inventory：指定操作的主机，是一个配置文件里面定义监控的主机。
- 各种核心模块、command模块、自定义模块。
- 借助插件完成记录日志和邮件等功能。
- playbook：剧本执行多个任务时，非必需可以让节点一次运行多个任务。



### ansible架构

ansible的架构图如下所示：

<img src="ansible.png" style="zoom:90%;" />



- `Ansible`：Ansible核心程序。 
- `Host Lnventory`：记录由Ansible管理的主机信息，包括端口、密码、ip等。
- `Playbooks`：“剧本”YAML格式文件，多个任务定义在一个文件中，定义主机需要调用哪些模块来完成的功能。 
- `Core Modules`：核心模块，来完成管理任务。先调用此中的模块，再指定`Host Lnventory`中的主机来完成管理任务。 
- `Custom Modules`：自定义模块，完成核心模块无法完成的功能，支持多种语言。
- `Connection Plugins`：连接插件，做通信使用。



### 工作机制

<img src="work.png" style="zoom:90%;" />

Ansible在管理节点（Control Node）将Ansible模块通过SSH协议等推送到管理端执行，执行完成后自动删除。



<br>



## 安装ansible

ansible使用的时候依赖python环境必须大于2.7，否则有些模块会报错。



### 依赖安装

```bash
$ yum install -y epel-release libselinux-python
```



### 【方式一】使用pip安装

ansible本身使用python开发，所以可以直接使用pip进行ansible的安装：

```bash
$ pip install ansible
```





### 【方式二】使用yum安装

直接使用yum方式安装ansible即可：

```bash
$ yum install -y ansible
```



### 安装的相关文件

-  `/etc/ansible/ansible.cfg`：主配置文件；
- `/etc/ansible/hosts`：主机定义文件；
- `/usr/share/ansible/plugins`：插件目录；



<br>



## 配置ansible



### 添加主机列表

需要在`/etc/ansible/hosts` 中定义需要管理的主机列表，例如：

```
[master]
10.8.138.5
10.8.138.6
10.8.138.10
```



其中，`[master]`表示一个分组，在配置文件中可以有多个分组；分组下面的服务器就是待管理的机器列表。



### 设置免秘钥登录

ansible是基于ssh进行管理的，且需要让ansible控制服务器和其他待管理的服务器间实现免秘钥登录：

```bash
# 如果本机没有秘钥（,ssh/目录下为空），则重新生成
$ ssh-keygen

# 发送秘钥到远程主机
$ ssh-copy-id root@10.8.138.5
```



### 联通测试

使用`ping`模块测试是否能够联通：

```bash
$ ansible master -m ping --user=root
```

- `master`指定需要进行操作的分组，也可以是单个服务器；
- `-m`指定使用的模块，这里使用的是ping模块；
- `--user`指定远程连接使用的用户，这里用的是root用户；



<img src="testping.png" style="zoom:40%;" />

> 可以看到都是success状态，说明ansible是正常工作的。



<br>




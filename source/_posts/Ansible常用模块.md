---
title: Ansible常用模块
date: 2020-07-19 13:00:31
tags:
- 自动化运维工具
- ansible
categories:
- 自动化运维
- Ansible
description: 介绍了常用模块的功能和使用方法
cover: https://ss1.bdstatic.com/70cFvXSh_Q1YnxGkpoWK1HF6hhy/it/u=3898177574,237099017&fm=26&gp=0.jpg
---



## 命令相关模块

### 远程命令command模块

command模块用于执行远程命令，但不能使用变量、管道等。

```bash
$ ansible all -m command -a 'uptime'
```

或者

```bash
$ ansible all -a 'uptime'
```



### 远程命令shell模块

shell模块可移植性远程命令，支持管道等复杂命令

```bash
$ ansible all -m shell -a 'echo test | passwd --stdin user1'
```



### 脚本script模块

script模块可以将本地脚本复制到远程执行

```bash
$ ansible all -m script -a '/tmp/a.sh'
```

<br>



## 文件操作相关模块

### 复制文件copy模块

copy模块用于复制文件到远程主机。

```bash
# 将当前目录下的1文件复制到远程主机的/tmp下并改名为1.asb，属主为root权限640
$ ansible all -m copy -a 'src=1 dest=/tmp/1.asb owner=root mode=640'

# 将此处的信息生成为文件内容并发送到远程主机
$ ansible all -m copy -a 'content="hello \ntest" dest=/tmp/2.asb'
```

常用参数：

- src：本地源文件路径；
- dest：远程目录绝对路径；
- owner：属主；
- group：属组；
- mode：权限；
- content：可以取代src，表示用content的内容生成文件；



### 复制文件模块fetch

fetch模块可以将远程服务器上的文件复制到本地

```bash
# 将服务器 10.8.138.5上/root/app.jar复制到本地
$ ansible 10.8.138.5 -m fetch -a "src=/root/app.jar dest=/root"
```

常用参数：

- src：远程服务器上文件路径；
- dest：本地文件路径



文件拷贝到本地，会在`dest`指定的目录下创建一个远程服务器名称的目录，在目录中再创建名为`dest`的子目录，在其下是拷贝的文件：

```bash
$ tree 10.8.138.5/
10.8.138.5/
└── root
    └── app.jar

1 directory, 1 file
```



### 文件属性file模块

file模块用于设置文件属性。

```bash
# 修改属主和权限
$ ansible all -m file -a 'owner=root group=root mode=644 path=/root/test'

# 创建链接
$ ansible all -m file -a 'path=/tmp/test.link src=/root/test state=link'
```



<br>



## 系统用户和组管理相关模块

### 用户管理user模块

user模块用于用户账号管理。

```bash
# 增加一个用户
$ ansible all -m user -a 'name="user1"'

# 删除一个用户
$ ansible all -m user -a 'name="user1" state=absent'
```



### 组管理group模块

group模块用于管理用户组。

```bash
# 新增一个用户组
$ ansible all -m group -a 'name=mysql gid=306 system=yes'

# 新增一个用户并添加到用户组下
$ ansible all -m user -a 'name=mysql uid=306 system=yes group=mysql'
```



常用参数：

- gid：指定组id；
- name：指定组名；
- state：
  - present：新增（默认）
  - absent：移除组；
- system：是否为系统组；



<br>



## 系统服务管理相关模块

### 服务管理service模块

service模块用于管理服务运行状态。

```bash
$ ansible k8snode -m service -a 'enabled=true name=httpd state=started'
```



常用参数：

- enabled：是否开机自启动；
- name：指定服务名；
- state：指定服务状态
  - started：启动服务；
  - stoped：停止服务；
  - restarted：重启服务；
- arguments：服务参数



### yum模块

yum模块可以安装软件包

```bash
# 安装httpd
$ ansible all -m yum -a 'name=httpd'

# 卸载httpd
$ ansible all -m yum -a 'name=httpd state=absent'
```

常用参数：

- name：程序包名，不指定版本则安装最新版；
- state：
  - present、latest：安装；
  - absent：卸载；



<br>





## 计划任务cron模块

cron模块用于指定计划任务，每一个任务必须有一个名字。

```bash
# 每10分钟执行一个echo命令。
$ ansibel k8snode -m cron -a 'minute="*/10" job="/bin/echo hello" name="test job"'

# 查看任务
$ ansible k8snode -a 'crontab -l'

# 移除一个任务 
$ ansibel k8snode -m cron -a 'minute="*/10" job="/bin/echo hello" name="test job" state=absend'
```



常用参数：

- month：指定月份；
- minute：指定分钟；
- job：指定任务内容；
- day：指定天；
- hour：指定小时；
- weekday：指定周几；
- state：
  - present：新增（默认）
  - absent：移除任务；

<br>



## ping模块

ping模块用于测试主机是否能够联通。

```bash
$ ansible all -m ping
```

<br>



## setup模块

setup模块可以让远程服务器上报自己的具体信息

```bash
$ ansible all -m setup
```



<br>



## hostname主机名管理模块

hostname模块可以管理远程服务器的主机名

```bash
# 修改服务器主机名为testserver
$ ansible 10.10.1.2 -m hostname -a "name=testserver"
```


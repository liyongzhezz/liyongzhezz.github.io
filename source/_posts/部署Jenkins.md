---
title: 部署Jenkins
date: 2020-08-18 14:44:58
tags:
- Jenkins
categories:
- CICD
- Jenkins
description: 使用docker和rpm两种方式部署jenkins服务
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1597743430763&di=05bfdf736602b44a207e43f98c5130d2&imgtype=0&src=http%3A%2F%2Fwww.steptoinstall.com%2Fwp-content%2Fuploads%2F2014%2F10%2FHow-to-uninstall-remove-Jenkins-Application.png
---



# Docker方式部署



## 创建数据目录

```bash
$ mkdir -p /data/jenkins-data
```



> 数据目录最好单独挂载一个磁盘或使用分布式存储



## 启动jenkins容器

使用下面的命令直接启动jenkins容器：

```bash
$ docker run -d --rm \
    --name jenkins \
    -u root \
    -p 8080:8080 \
    -p 50000:50000 \
    -v /data/jenkins-data:/var/jenkins_home \
    -v /var/run/docker.sock:/var/run/docker.sock \
    jenkinsci/blueocean
```



- `-v /data/jenkins-data:/var/jenkins_home`：指定jenkins的数据目录持久化到宿主机的/data/jenkins-data下，这样可以持久化；

- `-v /var/run/docker.sock:/var/run/docker.sock`：允许jenkins和docker进程通信，这样jenkins可以启动docker容器（DinD方式）；



## 检查运行情况

```bash
$ docker ps | grep jenkins
```



然后检查` /data/jenkis-data/secrets/initialAdminPassword `这个文件，复制其中的内容，这是初始管理员用户的密码：



```bash
$ cat /data/jenkins-data/secrets/initialAdminPassword
```



## 访问jenkins

直接访问本机的8080端口即可进入jenkins的初始界面，输入上面复制的管理员密码。接着是一些常用插件的安装，这里可以根据需要自行安装。



<br>



# RPM方式部署



## 安装java8

jenkins要求java 8以上版本：

```bash
$ yum install java-1.8.0-openjdk* -y
$ java -version
```



## 关闭防火墙

生产中应该开启防火墙，开启需要的端口即可：

```bash
$ systemctl stop firewalld
$ systemctl disable firewalld
$ systemctl stop iptables
$ systemctl disbale iptables
```



## 安装jenkins

可以在 https://pkg.jenkins.io/redhat-stable/ 找到对应的jenkins rpm包进行下载：

```bash
$ wget https://pkg.jenkins.io/redhat-stable/jenkins-2.204.1-1.1.noarch.rpm 
$ chmod +x jenkins-2.204.1-1.1.noarch.rpm 
$ rpm -ivh jenkins-2.204.1-1.1.noarch.rpm 
$ systemctl start jenkins 
$ systemctl enable jenkins
```



安装jenkins会生成无密码的系统用户jenkins，使用下面命令设置密码：

```bash
$ passwd jenkins
```



安装后默认一些文件和目录如下：

- `/usr/lib/jenkins/jenkins.war`：jenkins执行文件；
- `/var/log/jenkins/jenkins.log`：日志文件；
- `/etc/sysconfig/jenkins`：配置文件；
- `/var/lib/jenkins`：数据目录；



## 设置配置文件

安装后，jenkins默认配置文件在` /etc/sysconfig/jenkins`，修改这个文件重新设置相关目录：

```bash
JENKINS_HOME="/data/jenkins-data" JENKINS_PORT="8080"
```



这样就将jenkins数据放在新指定的目录下

```bash
$ mkdir /data/jenkins-data -p 
$ chown -R jenkins:jenkins /data/jenkins-data 
$ systemctl restart jenkins
```



## 访问jenkins

启动后，查看 `/data/jenkins-data/secrets/initialAdminPassword `文件的内容，这个是初始密码，使用这个密码登录。

```bash
$ cat /data/jenkins-data/secrets/initialAdminPassword
```



浏览器访问本机的8080端口，输入上边的初始密码，然后安装一些插件，即可进入jenkins。






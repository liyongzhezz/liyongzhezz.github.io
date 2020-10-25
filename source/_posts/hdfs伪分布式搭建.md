---
title: hdfs伪分布式搭建
date: 2020-10-25 12:31:48
tags:
- HDFS
categories:
- 大数据
- HDFS
description: 在一个节点上运行hdfs伪分布式
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1603610502780&di=4fe99d9854187293278a62251420c4f0&imgtype=0&src=http%3A%2F%2Fattachbak.dataguru.cn%2Fattachments%2Fportal%2F201312%2F10%2F1247240batq2qo9uqeyyte.jpg
---



## 环境

- hadoop版本：hadoop-2.6.0-cdh5.7.0
- 操作系统：Centos 7.2
- Jdk：1.7以上



可以在 [cdh下载](http://archive.cloudera.com/cdh5/cdh/5/)找到对应的版本，cdh5.7.0对应网站为：[cdh5.7.0](http://archive.cloudera.com/cdh5/cdh/5/hadoop-2.6.0-cdh5.7.0/)



**伪分布式就是在一个节点上运行hdfs环境，不适合于生产环境，只适合于测试和开发。**

<br>



## 安装依赖

```bash
$ yum install java-1.8.0-openjdk*

$ java -version
openjdk version "1.8.0_71"
OpenJDK Runtime Environment (build 1.8.0_71-b15)
OpenJDK 64-Bit Server VM (build 25.71-b15, mixed mode)

# 设置JAVA_HOME
$ echo "export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.71-2.b15.el7_2.x86_64" >> /etc/profile
$ source /etc/profile
$ echo $JAVA_HOME
```

> JAVA_HOME的路径名可能有所区别，要具体根据版本看下



## 设置ssh免密登录

```bash
# 全部回车即可
$ ssh-keygen -t rsa

$ ls /root/.ssh/
authorized_keys  id_rsa  id_rsa.pub

$ cp /root/.ssh/id_rsa.pub /root/.ssh/authorized_keys
```



> 在真正的分布式环境中，这一步很重要。





## 下载

```bash
$ wget http://archive.cloudera.com/cdh5/cdh/5/hadoop-2.6.0-cdh5.7.0.tar.gz
$ tar zxf hadoop-2.6.0-cdh5.7.0.tar.gz -C /usr/local/
```



其中解压后的重要的目录为：

- `bin`：客户端命令脚本             
- `etc`：配置文件
- `sbin`：hadoop及其他服务的启动命令



修改`etc/hadoop-env.sh`，增加JAVA_HOME配置：

```bash
export JAVA_HOME="/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.71-2.b15.el7_2.x86_64"
```



如果ssh的端口不是22，则增加下面的配置：

```bash
export HADOOP_SSH_OPTS="-p 36000"
```



修改`etc/core-site.xml`文件，设置hdfs的地址：

```xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:8020</value>
    </property>
  
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/data/hadoop/tmp</value>
    </property>
</configuration>
```

> hadoop会存放一些临时文件，如果不指定`hadoop.tmp.dir`，那么服务器重启后临时文件就会被清理，可能会影响服务



创建这个临时文件目录：

```bash
$ mkdir /data/hadoop/tmp
```



修改`etc/hdfs-site.xml`，设置副本系数：

```xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
```

> 这里设置副本系数为1，因为当前只有一个节点



## 配置salve节点

在`etc/slaves`文件中，目前是只有一个`localhost`，因为当前是伪分布式（单机版），所以这个配置是可以满足的不需要更改，如果是一个namenode带多个datanode，则需要将所有的datanode节点的主机名写在这个文件中。



## 启动

首先格式化文件系统：

```bash
$ bin/hdfs namenode -format
```

> 注意，仅第一次启动执行，后面再执行相当于进行了格式化



启动namendoe和datanode进程：

```bash
$ sbin/start-dfs.sh
```



查看启动的进程：

```bash
$ jps
20448 SecondaryNameNode
20273 DataNode
20149 NameNode
20703 Jps
```



通过浏览器，输入`<本机ip>：50070`，将看到一个类似于监控的页面：

![](hadoop.png)



## 停止服务

```bash
$ sbin/stop-dfs.sh
```


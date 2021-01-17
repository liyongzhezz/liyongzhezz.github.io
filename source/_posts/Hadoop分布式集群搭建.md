---
title: Hadoop分布式集群搭建
date: 2021-01-17 12:39:24
tags:
- 大数据
- Hadoop
categories:
- 大数据
- Hadoop
description: 搭建hadoop分布式集群
cover: https://gimg2.baidu.com/image_search/src=http%3A%2F%2Fcdn.easyaq.com%2F%40%2F20170604%2F1496571554997058859.jpg&refer=http%3A%2F%2Fcdn.easyaq.com&app=2002&size=f9999,10000&q=a80&n=0&g=0n&fmt=jpeg?sec=1613450430&t=00f6bd27f119c748fcd5c530cc0f806b
---



## 环境

| 主机名    | IP              | 角色                                             |
| --------- | --------------- | ------------------------------------------------ |
| hadoop000 | 192.168.199.102 | NameNode、ResourceManager、DataNode、NodeManager |
| hadoop001 | 192.168.199.247 | DataNode、NodeManager                            |
| Hadoop002 | 192.168.199.138 | DataNode、NodeManager                            |



> 默认副本系数为3，所以最好有三个datanode，实际中datanode最好不要和namenode公用

<br>



## 环境设置

### 设置host

在三个节点上都执行下面的命令：

```bash
$ cat >> /etc/hosts << EOF
192.168.199.102 hadoop000
192.168.199.247 hadoop001
192.168.199.138 hadoop002
EOF
```



### 设置免密登录

在每台机器上设置：

```bash
$ ssh-keygen -t rsa
$ ssh-copy-id -i ~/.ssh/id_rsa.pub hadoop000
$ ssh-copy-id -i ~/.ssh/id_rsa.pub hadoop001
$ ssh-copy-id -i ~/.ssh/id_rsa.pub hadoop002
```



### JDK安装

每个节点都要安装：

```bash
$ yum install java-1.8.0-openjdk*

$ java -version
openjdk version "1.8.0_71"
OpenJDK Runtime Environment (build 1.8.0_71-b15)
OpenJDK 64-Bit Server VM (build 25.71-b15, mixed mode)
```



设置JAVA_HOME：

```bash
# 设置JAVA_HOME
$ echo "export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.71-2.b15.el7_2.x86_64" >> /etc/profile
$ source /etc/profile
$ echo $JAVA_HOME
```



<br>



## 集群搭建



### 下载软件包

在所有节点执行下面的命令：

```bash
$ wget http://archive.cloudera.com/cdh5/cdh/5/hadoop-2.6.0-cdh5.7.0.tar.gz
$ tar zxf hadoop-2.6.0-cdh5.7.0.tar.gz -C /usr/local/
$ echo "export PATH=$PATH:/usr/local/hadoop-2.6.0-cdh5.7.0/bin:/usr/local/hadoop-2.6.0-cdh5.7.0/sbin" >> /etc/profile
$ source /etc/profile
```



### 修改配置

> 在所有节点执行下面的操作



首先修改`etc/hadoop-env.sh`，增加`JAVA_HOME`：

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
        <value>hdfs://hadoop000:8020</value>
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
        <value>3</value>
    </property>
</configuration>
```

​    

修改`etc/yarn-site.yaml`：

```xml
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
      	<values>mapreduce_shuffle</values>
    </property>
  
    <property>
        <name>yarn.resourcemanager.hostname</name>
      	<values>hadoop000</values>
    </property>
</configuration>
```



修改`etc/mapred-site.xml`：

```xml
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
      	<values>yarn</values>
    </property>
</configuration>
```



修改`etc/slaves`，设置从节点：

```bash
hadoop000
hadoop001
hadoop002
```



### NameNode格式化

只在hadoop000上执行即可:

```bash
$ hdfs namenode -format
```



> **仅第一次启动执行，后面再执行相当于进行了格式化**



### 启动集群

在主节点hadoop000上执行即可

```bash
# 这个脚本在sbin目录下
$ start-all.sh
```



> 可以看到他按照分配的角色启动了对应的进程



### 检查

在每个节点上都可以执行下面的命令验证：

```bash
$ jps
```



按照分配的角色：

- 在hadoop000上执行应该启动：SecondaryNameNode、Datanode、NodeManager、NameNode、ResourceManager；
-  在hadoop001和hadoop002上执行应该启动：Datanode、NodeManager；



### webui检查

访问hadoop000的50070端口，可以查看到当前集群的监控，应该可以看到所有的namenode节点；



访问hadoop000的8088端口，可以进入yarn的webui界面，也可以看到所有的节点； 



### 停止集群

在hadoop000上执行下面的命令：

```bash
$ stop-all.sh
```



<br>


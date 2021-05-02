---
title: 二进制方式部署kafka集群
date: 2021-05-02 19:01:37
tags:
- Kafka
categories:
- 消息中间件
- Kafka
- 部署
description: 使用二进制方式部署kafka集群
cover: https://gimg2.baidu.com/image_search/src=http%3A%2F%2Fwww.kailing.pub%2FUploads%2Fimage%2F20190314%2F20190314151109_73755.png&refer=http%3A%2F%2Fwww.kailing.pub&app=2002&size=f9999,10000&q=a80&n=0&g=0n&fmt=jpeg?sec=1622545443&t=c522a6bf51142ae088cea1e60fd9368f
---



{% note info 'fas fa-bullhorn' %}

本文主要介绍使用二进制方式部署kafka集群，版本为kafka_2.12-2.2.1

更新于 2021-05-02

{% endnote %}



<br>



# 服务器规划

| IP地址     | 主机名      | Kafka版本            | ZooKeeper版本           | JDK版本                    |
| ---------- | ----------- | -------------------- | ----------------------- | -------------------------- |
| 172.17.0.2 | kafka_node1 | kafka_2.12-2.2.1.tgz | zookeeper-3.4.14.tar.gz | jdk-8u161-linux-x64.tar.gz |
| 172.17.0.3 | kafka_node2 | kafka_2.12-2.2.1.tgz | zookeeper-3.4.14.tar.gz | jdk-8u161-linux-x64.tar.gz |
| 172.17.0.4 | kafka_node3 | kafka_2.12-2.2.1.tgz | zookeeper-3.4.14.tar.gz | jdk-8u161-linux-x64.tar.gz |



<br>



# 部署JDK

```bash
tar xf jdk-8u161-linux-x64.tar.gz  -C /usr/local/

cat << EOF >> /etc/profile
#################JAVA#################
export JAVA_HOME=/usr/local/jdk1.8.0_161
export JRE_HOME=\$JAVA_HOME/jre
export CLASSPATH=.:\$JAVA_HOME/lib/dt.jar:\$JAVA_HOME/lib/tools.jar:\$JRE_HOME/lib
export PATH=\$JAVA_HOME/bin:\$JRE_HOME/bin:\$PATH
EOF

source /etc/profile
java -version
```



<br>



# 部署zookeeper

## 下载

```bash
# 下载
wget https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/zookeeper-3.4.14/zookeeper-3.4.14.tar.gz
```



## 安装

```bash
# 安装
tar xf zookeeper-3.4.14.tar.gz  -C /usr/local/
cp -rf /usr/local/zookeeper-3.4.14/conf/zoo_sample.cfg /usr/local/zookeeper-3.4.14/conf/zoo.cfg
```



## 设置配置文件

三个节点的配置需要保持一致。



```bash
cat << EOF > /usr/local/zookeeper-3.4.14/conf/zoo.cfg
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/data/zk_data
dataLogDir=/usr/local/zookeeper-3.4.14/logs
clientPort=2181
maxClientCnxns=60
autopurge.snapRetainCount=3
autopurge.purgeInterval=1
server.1=172.17.0.2:2888:3888
server.2=172.17.0.3:2888:3888
server.3=172.17.0.4:2888:3888
EOF
```



## 创建目录

```bash
mkdir -p /data/zk_data
mkdir /usr/local/zookeeper-3.4.14/logs
```



## 创建ServerID标识

在ZooKeeper集群中除配置文件外，还需要配置一个myid文件，这个文件需要存放在配置文件中dataDir配置项所指定的数据位置，要根据集群中的节点创建不同的文件。我们要根据ServerID标示来创建相应的文件

```bash
# 在node-1节点
echo '1' > /data/zk_data/myid

# 在node-2节点 
echo '2' > /data/zk_data/myid

# 在node-3节点
echo '3' > /data/zk_data/myid
```



## 启动

三个节点全部启动：



```bash
/usr/local/zookeeper-3.4.14/bin/zkServer.sh start

ZooKeeper JMX enabled by default
Using config: /usr/local/zookeeper-3.4.14/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED
```



使用`lsof -i:`命令来查看端口信息：

- 如果是`leader`节点：查看到的连接会是与集群内所有的follower的连接；
- 如果是`follower`节点：查看到的连接将只会与ZK集群中的leader连接；



```bash
lsof -i:2888
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
java    15620 root   30u  IPv4  71449      0t0  TCP kafka_node1:42424->172.17.0.3:spcsdlobby (ESTABLISHED)
```



> node1只有一个连接是和172.17.0.3建立的，可以表明此节点为follower节点



```bash
lsof -i:2888
COMMAND PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
java     91 root   30u  IPv4  65243      0t0  TCP kafka_node2:spcsdlobby (LISTEN)
java     91 root   31u  IPv4  69183      0t0  TCP kafka_node2:spcsdlobby->172.17.0.2:42424 (ESTABLISHED)
java     91 root   33u  IPv4  69192      0t0  TCP kafka_node2:spcsdlobby->172.17.0.4:49420 (ESTABLISHED)

```



> node2是与集群内的其它两台机器所连接，可以表明此节点为leader节点



```bash
lsof -i:2888
COMMAND PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
java     91 root   31u  IPv4  64087      0t0  TCP kafka_node3:49420->172.17.0.3:spcsdlobby (ESTABLISHED)
```



> node3与node1一样为follower节点



<br>



# 部署kafka

## 下载安装

```bash
wget http://mirror.bit.edu.cn/apache/kafka/2.2.1/kafka_2.12-2.2.1.tgz
tar xf kafka_2.12-2.2.1.tgz -C /usr/local/
```



## 修改配置文件

先备份原配置文件：

```bash
cp -rf /usr/local/kafka_2.12-2.2.1/config/server.properties /usr/local/kafka_2.12-2.2.1/config/server.properties.default
```



设置新的配置文件：

```bash
cat << EOF > /usr/local/kafka_2.12-2.2.1/config/server.properties
broker.id=1                                                     #Kafka_node2节点修改为2，3修改为3
listeners=PLAINTEXT://172.17.0.2:9092                           #修改Kafka_node的IP地址为各自node本地地址
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
log.dirs=/data/kafka-logs/
num.partitions=1
num.recovery.threads.per.data.dir=1
offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1
log.retention.hours=72
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000
zookeeper.connect=172.17.0.2:2181,172.17.0.3:2181,172.17.0.4:2181
delete.topic.enable=true
zookeeper.connection.timeout.ms=6000
group.initial.rebalance.delay.ms=3000
EOF
```



## 启动

启动三个节点的服务

```bash
/usr/local/kafka_2.12-2.2.1/bin/kafka-server-start.sh -daemon /usr/local/kafka_2.12-2.2.1/config/server.properties
```



## 检查

ZooKeeper监控本地地址TCP端口2181，可以ZooKeeper看到2181后面对应的还有一个端口号为43918
Kafka监控本地地址TCP端口9092，可以看到Kafka9092后面也有对应的一个端口号55568

```bash
netstat -anplt | egrep "(2181|9092)"
tcp        0      0 172.17.0.2:9092         0.0.0.0:*               LISTEN      15945/java          
tcp        0      0 0.0.0.0:2181            0.0.0.0:*               LISTEN      15620/java          
tcp        0      0 172.17.0.2:45674        172.17.0.4:9092         ESTABLISHED 15945/java          
tcp        0      0 172.17.0.2:55568        172.17.0.2:9092         ESTABLISHED 15945/java          
tcp        0      0 172.17.0.2:9092         172.17.0.2:55568        ESTABLISHED 15945/java          
tcp        0      0 172.17.0.2:43094        172.17.0.3:9092         ESTABLISHED 15945/java          
tcp        0      0 172.17.0.2:2181         172.17.0.2:43918        ESTABLISHED 15620/java          
tcp        0      0 172.17.0.2:43918        172.17.0.2:2181         ESTABLISHED 15945/java   
```



## 查看kafka连接情况

查看node1的情况：

```bash
lsof -i:9092
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
java    15945 root  106u  IPv4  64487      0t0  TCP kafka_node1:XmlIpcRegSvc (LISTEN)
java    15945 root  122u  IPv4  72859      0t0  TCP kafka_node1:55568->kafka_node1:XmlIpcRegSvc (ESTABLISHED)
java    15945 root  123u  IPv4  71815      0t0  TCP kafka_node1:XmlIpcRegSvc->kafka_node1:55568 (ESTABLISHED)
java    15945 root  127u  IPv4  72997      0t0  TCP kafka_node1:43094->172.17.0.3:XmlIpcRegSvc (ESTABLISHED)
java    15945 root  131u  IPv4  75769      0t0  TCP kafka_node1:45674->172.17.0.4:XmlIpcRegSvc (ESTABLISHED)
```

> node1节点同时与node2及node3建立了连接，即node1节点为主节点



查看node2的情况：

```bash
lsof -i:9092
COMMAND PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
java    415 root  106u  IPv4  75016      0t0  TCP kafka_node2:XmlIpcRegSvc (LISTEN)
java    415 root  116u  IPv4  72998      0t0  TCP kafka_node2:XmlIpcRegSvc->172.17.0.2:43094 (ESTABLISHED)
```

> node2只与node1建立了连接，即它是follower节点



查看node3的情况：

```bash
lsof -i:9092
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
java    1125 root  106u  IPv4  73394      0t0  TCP kafka_node3:XmlIpcRegSvc (LISTEN)
java    1125 root  116u  IPv4  72253      0t0  TCP kafka_node3:XmlIpcRegSvc->172.17.0.2:45674 (ESTABLISHED)
```

> node3与node1建立了连接，即它是follower节点




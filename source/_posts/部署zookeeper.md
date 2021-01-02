---
title: 部署zookeeper
date: 2020-12-12 17:51:12
tags:
- zookeeper
categories:
- zookeeper
description: 部署zookeeper服务
cover:
---





## 下载

下载地址：http://archive.apache.org/dist/zookeeper/

>  需要下载的包必须带`bin`，这个是编译好的，否则运行会报错



```bash
$ wget http://archive.apache.org/dist/zookeeper/zookeeper-3.5.5/apache-zookeeper-3.5.5-bin.tar.gz
```



<br>



## 单节点环境部署

**单节点不适合生产环境使用**

### 安装

下载好的zookeeper压缩包已经经过编译，直接解压就可以用：

```bash
$ tar zxf apache-zookeeper-3.5.5-bin.tar.gz 
$ cd cd apache-zookeeper-3.5.5-bin
```



### 建立配置文件

默认没有配置文件，可以直接复制配置文件模板：

```bash
$ mv conf/zoo_sample.cfg conf/zoo.cfg
```



### 创建数据目录

虽然zookeeper一般数据量不大，但是还是推荐使用单独的数据盘：

```bash
$ mkdir /data/zookeeper
```



然后修改配置文件的如下字段，使用数据目录：

```bash
# conf/zoo.cfg
dataDir=/data/zookeeper
```



### 后台方式启动服务

```bash
$ bin/zkServer.sh start
```

![](./start.png)



### 前台方式启动

```bash
$ bin/zkServer.sh start-foreground
```



### 查看服务状态

```bash
$ bin/zkServer.sh status 
```

![](./status.png)



### 查看日志

如果是后台方式启动，日志默认存放在安装目录下的logs中，文件名包含主机名：

```bash
$ tail -f apache-zookeeper-3.5.5-bin/logs/zookeeper-root-server-localhost.out
```





### 停止zookeeper服务

```bash
$ bin/zkServer.sh stop 
```

![](./stop.png)



<br>



## zookeeper集群部署

### 安装

下载好的zookeeper压缩包已经经过编译，直接解压就可以用：

```bash
# 在每个节点都下载
$ tar zxf apache-zookeeper-3.5.5-bin.tar.gz 
$ cd cd apache-zookeeper-3.5.5-bin
```



### 建立配置文件

默认没有配置文件，可以直接复制配置文件模板：

```bash
# 在每个节点都执行
$ mv conf/zoo_sample.cfg conf/zoo.cfg
```



### 创建数据目录

虽然zookeeper一般数据量不大，但是还是推荐使用单独的数据盘：

```bash
# 在每个节点都执行
$ mkdir /data/zookeeper
```



然后在每个节点都修改配置文件的如下字段，使用数据目录：

```bash
# conf/zoo.cfg
dataDir=/data/zookeeper
```



### 添加zookeeper节点

在每个节点的配置文件`zoo.cfg`中，添加zookeeper集群每台节点的地址，例如我这里是三节点的集群：

```bash
# conf/zoo.cfg
# 增加如下的内容
server.2=192.168.12.12:2888:3888
server.3=192.168.12.13:2888:3888
server.4=192.168.12.14:2888:3888
```



> `2888`为zookeeper节点间的通信端口号，`3888`为选举的端口号；主从关系通过`3888`自动选举产生，配置中不需要指定；
>
> `server`后面的数字为集群中节点的id，可以随便起但是不能相同；



### 添加id文件

在上一步中添加了三个节点：

```bash
server.2=192.168.12.12:2888:3888
server.3=192.168.12.13:2888:3888
server.4=192.168.12.14:2888:3888
```



现在到`192.168.12.12`这个节点的zookeeper数据目录`/data/zookeeper`下，设置节点id文件：

```bash
$ cd /data/zookeeper
$ echo 2 > myid
```



> `myid`这个文件名是固定的不能变；文件的内容就是这个节点的id号，根据配置文件中指定的数字配置；



其他的节点，需要执行相同的操作，注意id设置为自己的id；



### 设置日志位置

默认日志位置是放在安装目录下的logs下，可以在`bin/zkENV.sh`中，修改下面的代码进行重新设置：

```bash
# bin/zkENV.sh
if [ "x${ZOO_LOG_DIR}" = "x" ]
then
    ZOO_LOG_DIR="/data/zookeeper/logs"
fi
```



然后创建日志目录：

```bash
$ mkdir /data/zookeeper/logs
```



### 设置JAVA_HOME

如果需要设置`JAVA_HOME`变量，则可以在`bin/zkENV.sh`的前面设置一个环境变量即可：

```bash
# bin/zkENV.sh
export JAVA_HOME=/java/home/dir
```



### 启动

在每个节点都执行启动命令：

```bash
$ bin/zkServer.sh start
```



### 查看状态

当集群一半以上的节点都启动之后，整个集群才会正常工作，此时执行下面的命令查看服务状态：

```bash
$ $ bin/zkServer.sh status 
```



如果出现`Mode: follower`则表示该节点为从节点；如果出现`Mode: leader`则表示该节点为主节点



<br>



## docker方式部署zookeeper集群

### 创建目录

在每个节点都创建数据和日志目录：

```bash
$ mkdir -p /data/zookeeper/data
$ mkdir -p /data/zookeeper/logs
```



### 下载镜像

这里使用的是3.5.5版本的zookeeper：

```bash
$ docker pull zookeeper:3.5.5
```



### 创建配置文件

配置文件的内容如下：

```bash
cat > /data/zookeeper/zoo.cfg << EOF
clientPort=2181
dataDir=/data/zookeeper/data
dataLogDir=/data/zookeeper/logs
tickTime=2000
initLimit=5
syncLimit=2
autopurge.snapRetainCount=3
autopurge.purgeInterval=0
maxClientCnxns=60
standaloneEnabled=false
admin.enableServer=true
4lw.commands.whitelist=*
server.2=192.168.12.12:2888:3888
server.3=192.168.12.13:2888:3888
server.4=192.168.12.14:2888:3888
server.5=192.168.12.15:2888:3888
server.6=192.168.12.16:2888:3888
EOF
```



- `clientPort`：服务监听的端口；
- `dataDir`：数据目录；
- `dataLogDir`：顺序日志目录；
- `tickTime`：心跳间隔；
- `autopurge.snapRetainCount`：保留多少个snapshot，之前的会删除；
- `autopurge.purgeInterval`：多久会清理一次数据（0表示不清理）；
- `maxClientCnxns`：客户端连接数限制；
- `standaloneEnabled`：在启动脚本中关闭管理控制台；
- `4lw.commands.whitelist`：白名单；



> 注意修改配置文件中的server配置



**配置文件需要在每个节点都创建**



### 生成myid文件

根据预先设置的myid，在每个节点的`/data/zookeeper/data`下创建一个`myid`文件：

```bash
$ echo 2 > /data/zookeeper/data/myid
```



> 注意每个节点的myid文件内容不一样



### 启动zookeeper

在每个节点分别执行下面的命令启动zookeeper：

```bash
$ docker run \
    --name zookeeper \
    --network=host \
    --restart always \
    -v /data/zookeeper/conf/zoo.cfg:/conf/zoo.cfg \
    -v /data/zookeeper:/data/zookeeper \
    -v /etc/localtime:/etc/localtime:ro \
    -d zookeeper:3.5.5
```



<br>


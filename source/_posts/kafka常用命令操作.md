---
title: kafka常用命令操作
date: 2021-04-17 12:04:37
tags:
- Kafka
categories:
- 消息中间件
- Kafka
- 常用命令
description: kafka的命令行常用操作
cover: https://ss3.bdstatic.com/70cFv8Sh_Q1YnxGkpoWK1HF6hhy/it/u=1860660051,254729202&fm=26&gp=0.jpg
---



{% note info 'fas fa-bullhorn' %}

本文主要介绍了在kafka中常用的命令行常用操作

更新于 2021-04-17

{% endnote %}

<br>



# topic相关命令



## 创建topic

```bash
kafka-topics.sh --create --bootstrap-server 172.17.0.2:9092,172.17.0.3:9092,172.17.0.4:9092 --replication-factor 3 --partitions 3 --topic kafka_data
```



- `--bootstrap-server`：指定要哪台Kafka服务器上创建Topic，主机加端口，指定的主机地址一定要和配置文件中的listeners一致；
- `--replication-factor`：创建Topic中的每个分区(partition)中的复制因子数量，即为Topic的副本数量，建议和Broker节点数量一致，如果复制因子超出Broker节点将无法创建；
- `--partitions`：创建该Topic中的分区(partition)数量；
- `--topic`：指定Topic名称；



> 如果创建的topic包含"_."之类的符号会收到警告，但是不会影响创建。



## 查看topic

```bash
kafka-topics.sh --list --bootstrap-server 172.17.0.2:9092,172.17.0.3:9092,172.17.0.4:9092
```

- `--bootstrap-server`：指定查看哪一台kafka上的topic，这里查看所有节点的（也可以查看某一个节点）；



## 查看topic详情

```bash
kafka-topics.sh --describe --bootstrap-server 172.17.0.2:9092 --topic kafka_data

Topic:kafka_data    PartitionCount:3    ReplicationFactor:3    Configs:segment.bytes=1073741824
    Topic: kafka_data    Partition: 0    Leader: 1    Replicas: 1,2,3    Isr: 1,2,3
    Topic: kafka_data    Partition: 1    Leader: 2    Replicas: 2,3,1    Isr: 2,3,1
    Topic: kafka_data    Partition: 2    Leader: 3    Replicas: 3,1,2    Isr: 3,1,2
```

- `Topic:kafka_data`：topic名称
- `PartitionCount:3`：分片数量
- `ReplicationFactor:3`：Topic副本数量



## 修改topic分区数

**注意：topic分区数只能增大，不能减少**

```bash
kafka-topics.sh --alter --bootstrap-server 9.235.152.125:9092 --topic kafka_data --partitions 30
```

> 修改topic：kafka_data 的分区数为30



## 删除topic

```bash
kafka-topics.sh --delete --bootstrap-server 172.17.0.2:9092 --topic kafka_data
```



> 在node1节点删除了Topic，三台节点会同步更新，所以我们的`kafka_data`在三台node上全部删除



注意，这个操作只会标记topic为删除状态，想要永久删除，需要登录zookeeper进行删除topic数据：

```bash
# 在zookeeper上执行下面的命令登录
bin/zkCli.sh -server <zookeeper-host:port>

# 登录后执行下面的操作
rmr /brokers/topics/<topicname>
```



<br>



# 消息相关命令



## 发送消息

```bash
kafka-console-producer.sh --broker-list 172.17.0.2:9092 --topic kafka_data
>Hello Kafka_data
>I'm the 172.17.0.2 Kafka create
>test
```

- `--broker-list`：指定使用哪台broker来生产消息
- `--topic`：指定要往哪个Topic中生产消息



## 消费消息

```bash
kafka-console-consumer.sh --bootstrap-server 172.17.0.4:9092 --topic kafka_data --from-beginning 

I'm the 172.17.0.2 Kafka create
test
Hello Kafka_data
```


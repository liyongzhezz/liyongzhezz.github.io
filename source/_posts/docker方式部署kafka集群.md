---
title: docker方式部署kafka集群
date: 2021-04-17 12:29:33
tags:
- Kafka
categories:
- 消息中间件
- Kafka
- 部署
description: 使用docker-compose部署一个三节点kafka集群
cover: https://ss1.bdstatic.com/70cFuXSh_Q1YnxGkpoWK1HF6hhy/it/u=2822442550,174028885&fm=26&gp=0.jpg
---





{% note info 'fas fa-bullhorn' %}

本文主要介绍了使用docker-compose部署一个三节点kafka集群

更新于 2021-04-17

{% endnote %}

<br>





>  kafka依赖于zookeeper，所以在部署kafka的时候也需要部署一个zookeeper。



创建数据目录：

```bash
# zookeeper数据目录
mkdir -p /data/zookeeper/{data,datalog,logs}

# kafka数据目录
mkdir -p /data/kafka/node_{0..2}
```





使用下面的Docker-compose文件进行部署：

```yaml
# docker-compose.yaml
version: "3"
services:
  zookeeper:
    image: zookeeper
    container_name: zookeeper
    ports:
      - 2181:2181
    volumes:
      - ./data/zookeeper/data:/data
      - ./data/zookeeper/datalog:/datalog
      - ./data/zookeeper/logs:/logs
    restart: always
  kafka_node_0:
    depends_on:
      - zookeeper
    container_name: kafka-node-0
    image: wurstmeister/kafka
    environment:
      KAFKA_BROKER_ID: 0
      KAFKA_ZOOKEEPER_CONNECT: 192.168.1.100:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://192.168.1.100:9092
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092
      KAFKA_NUM_PARTITIONS: 3
      KAFKA_DEFAULT_REPLICATION_FACTOR: 2
    ports:
      - 9092:9092
    volumes:
      - ./data/kafka/node_0:/kafka
    restart: unless-stopped
  kafka_node_1:
    depends_on:
      - kafka_node_0
    container_name: kafka-node-1
    image: wurstmeister/kafka
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 192.168.1.100:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://192.168.1.100:9093
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9093
      KAFKA_NUM_PARTITIONS: 3
      KAFKA_DEFAULT_REPLICATION_FACTOR: 2
    ports:
      - 9093:9093
    volumes:
      - ./data/kafka/node_1:/kafka
    restart: unless-stopped
  kafka_node_2:
    depends_on:
      - kafka_node_1
    container_name: kafka-node-2
    image: wurstmeister/kafka
    environment:
      KAFKA_BROKER_ID: 2
      KAFKA_ZOOKEEPER_CONNECT: 192.168.1.100:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://192.168.1.100:9094
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9094
      KAFKA_NUM_PARTITIONS: 3
      KAFKA_DEFAULT_REPLICATION_FACTOR: 2
    ports:
      - 9094:9094
    volumes:
      - ./data/kafka/node_2:/kafka
    restart: unless-stopped
```



> 注意将文件中`192.168.1.100`替换为实际的地址



运行：

```bash
docker-compose up -d 
docker ps 
```



![](/Users/ivinli/Desktop/postback/status.png)



> 确保所有容器都正常启动





上面的有个缺点是zookeeper是单点的，如果想让zookeeper也是集群方式，可以使用如下的配置：

```yml
version: '2.1'

services:
  zoo1:
    image: zookeeper:3.4.9
    hostname: zoo1
    ports:
      - "2181:2181"
    environment:
        ZOO_MY_ID: 1
        ZOO_PORT: 2181
        ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888
    volumes:
      - ./zk-multiple-kafka-multiple/zoo1/data:/data
      - ./zk-multiple-kafka-multiple/zoo1/datalog:/datalog

  zoo2:
    image: zookeeper:3.4.9
    hostname: zoo2
    ports:
      - "2182:2182"
    environment:
        ZOO_MY_ID: 2
        ZOO_PORT: 2182
        ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888
    volumes:
      - ./zk-multiple-kafka-multiple/zoo2/data:/data
      - ./zk-multiple-kafka-multiple/zoo2/datalog:/datalog

  zoo3:
    image: zookeeper:3.4.9
    hostname: zoo3
    ports:
      - "2183:2183"
    environment:
        ZOO_MY_ID: 3
        ZOO_PORT: 2183
        ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888
    volumes:
      - ./zk-multiple-kafka-multiple/zoo3/data:/data
      - ./zk-multiple-kafka-multiple/zoo3/datalog:/datalog


  kafka1:
    image: confluentinc/cp-kafka:5.5.0
    hostname: kafka1
    ports:
      - "9092:9092"
    environment:
      KAFKA_ADVERTISED_LISTENERS: LISTENER_DOCKER_INTERNAL://kafka1:19092,LISTENER_DOCKER_EXTERNAL://${DOCKER_HOST_IP:-127.0.0.1}:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: LISTENER_DOCKER_INTERNAL:PLAINTEXT,LISTENER_DOCKER_EXTERNAL:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: LISTENER_DOCKER_INTERNAL
      KAFKA_ZOOKEEPER_CONNECT: "zoo1:2181,zoo2:2182,zoo3:2183"
      KAFKA_BROKER_ID: 1
      KAFKA_LOG4J_LOGGERS: "kafka.controller=INFO,kafka.producer.async.DefaultEventHandler=INFO,state.change.logger=INFO"
    volumes:
      - ./zk-multiple-kafka-multiple/kafka1/data:/var/lib/kafka/data
    depends_on:
      - zoo1
      - zoo2
      - zoo3

  kafka2:
    image: confluentinc/cp-kafka:5.5.0
    hostname: kafka2
    ports:
      - "9093:9093"
    environment:
      KAFKA_ADVERTISED_LISTENERS: LISTENER_DOCKER_INTERNAL://kafka2:19093,LISTENER_DOCKER_EXTERNAL://${DOCKER_HOST_IP:-127.0.0.1}:9093
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: LISTENER_DOCKER_INTERNAL:PLAINTEXT,LISTENER_DOCKER_EXTERNAL:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: LISTENER_DOCKER_INTERNAL
      KAFKA_ZOOKEEPER_CONNECT: "zoo1:2181,zoo2:2182,zoo3:2183"
      KAFKA_BROKER_ID: 2
      KAFKA_LOG4J_LOGGERS: "kafka.controller=INFO,kafka.producer.async.DefaultEventHandler=INFO,state.change.logger=INFO"
    volumes:
      - ./zk-multiple-kafka-multiple/kafka2/data:/var/lib/kafka/data
    depends_on:
      - zoo1
      - zoo2
      - zoo3

  kafka3:
    image: confluentinc/cp-kafka:5.5.0
    hostname: kafka3
    ports:
      - "9094:9094"
    environment:
      KAFKA_ADVERTISED_LISTENERS: LISTENER_DOCKER_INTERNAL://kafka3:19094,LISTENER_DOCKER_EXTERNAL://${DOCKER_HOST_IP:-127.0.0.1}:9094
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: LISTENER_DOCKER_INTERNAL:PLAINTEXT,LISTENER_DOCKER_EXTERNAL:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: LISTENER_DOCKER_INTERNAL
      KAFKA_ZOOKEEPER_CONNECT: "zoo1:2181,zoo2:2182,zoo3:2183"
      KAFKA_BROKER_ID: 3
      KAFKA_LOG4J_LOGGERS: "kafka.controller=INFO,kafka.producer.async.DefaultEventHandler=INFO,state.change.logger=INFO"
    volumes:
      - ./zk-multiple-kafka-multiple/kafka3/data:/var/lib/kafka/data
    depends_on:
      - zoo1
      - zoo2
      - zoo3
```


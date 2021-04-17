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




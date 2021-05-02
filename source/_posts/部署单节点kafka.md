---
title: 部署单节点kafka
date: 2021-04-19 21:49:44
tags:
- Kafka
categories:
- 消息中间件
- Kafka
- 部署
description: 使用docker-compose部署单点kafka服务
cover: https://gimg2.baidu.com/image_search/src=http%3A%2F%2Fwww.kailing.pub%2FUploads%2Fimage%2F20190314%2F20190314151109_73755.png&refer=http%3A%2F%2Fwww.kailing.pub&app=2002&size=f9999,10000&q=a80&n=0&g=0n&fmt=jpeg?sec=1621432273&t=9d4c537f9a720233047a49967f071162
---



{% note info 'fas fa-bullhorn' %}

本文主要介绍使用docker-compose部署一个适合于开发、测试的单点kafka环境

更新于 2021-04-19

{% endnote %}

<br>



> 单机版适合学习和测试，不适合生产



# 创建docker-compose.yaml文件

创建一个名为`zk-single-kafka-single.yml `的文件：

```yaml
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
      ZOO_SERVERS: server.1=zoo1:2888:3888
    volumes:
      -./zk-single-kafka-single/zoo1/data:/data
      -./zk-single-kafka-single/zoo1/datalog:/datalog
 kafka1:
   image: confluentinc/cp-kafka:5.3.1
   hostname: kafka1
   ports:
     - "9092:9092"
   environment:
     KAFKA_ADVERTISED_LISTENERS: LISTENER_DOCKER_INTERNAL//kafka1:19092,LISTENER_DOCKER_EXTERNAL://${DOCKER_HOST_IP:-127.0.0.1}:9092
     KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: LISTENER_DOCKER_INTERNAL:PLAINTEXT,LISTENER_DOCKER_EXTERNAL:PLAINTEXT
     KAFKA_INTER_BROKER_LISTENER_NAME: LISTENER_DOCKER_INTERNAL
     KAFKA_ZOOKEEPER_CONNECT: "zoo1:2181"
     KAFKA_BROKER_ID: 1
     KAFKA_LOG4J_LOGGERS: "kafka.controller=INFO,kafka.producer.async.DefaultEventHandler=INFO,state.change.logger=INFO"
     KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
   volumes:
     -./zk-single-kafka-single/kafka1/data:/var/lib/kafka/data
   depends_on:
   	 - zoo1
```



# 启动服务

```bash
docker-compose -f zk-single-kafka-single.yml up
```



# 停止服务

```bash
docker-compose -f zk-single-kafka-single.yml down
```
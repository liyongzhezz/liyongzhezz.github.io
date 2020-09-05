---
title: '[k8s实践系列]EFK+Kafka日志收集(一)--架构方案和准备'
date: 2020-07-09 18:48:44
tags:
- k8s
- k8s日志收集
categories: 
- 实践K8s
- EFK日志收集
description: 使用filebeat、kafka、logstash、elasticsearch和kibana进行k8s日志收集
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1594302026732&di=01120f85d5d842f277b2ab6c42ef3373&imgtype=0&src=http%3A%2F%2Fwww.ruanyifeng.com%2Fblogimg%2Fasset%2F2017%2Fbg2017081701.jpg
---



## 架构方案

整体收集方案使用如下的组件：

- `filebeat`：采集节点和容器日志，发送到kafka；
- `kafka`：接收filebeat发送的日志消息；
- `logstash`：从kafka中消费日志消息并进行处理；
- `elasticsearch`：进行日志存储；
- `kibana`：日志可视化展示；



<br>

## 版本选择

- `filebeat`：7.6.2；
- `kafka`：2.12-2.5.0；
- `zookeeper`：3.5.7；
- `logstash`：7.8.0；
- `elasticsearch`：7.8.0；
- `kibana`：7.8.0；



<br>

## 方案可能存在的问题

这套日志收集方案可能存在下面的问题：

- elasticsearch可能存在瓶颈（es还需要进一步调优）；
- logstash会根据写入es的速度调整消息消费的速度，所有kafka有时候会出现消息堆积的情况；
- ......



<br>

## 准备工作



### nfs准备

创建nfs相关资源，包括nfs控制器和storageclass的方法都可以在 {% post_link k8s实践系列-使用nfs存储 %} 中找到，这里不再赘述。



### 创建命名空间

使用下面的yaml文件创建一个namespace：

```yaml
# namesapce.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: logging
```



```bash
$ kubectl apply -f namespace.yaml
```



### 设置elasticsearch密码

elasticsearch开启x-pack认证功能，这里将elasticsearch的密码保存在secret资源中：

```bash
$ espassword="elastic"
$ kubectl create secret generic es-logging-password --from-literal=elastic='elastic' -n logging
```



> 这里将密码设置为了`elastic`
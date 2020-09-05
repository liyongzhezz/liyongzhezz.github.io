---
title: 消息队列RabbitMQ
date: 2020-07-26 12:03:04
tags:
- 消息中间件
- RabbitMQ
categories:
- 消息中间件
- RabbitMQ
description: 消息队列RabbitMQ的原理、应用和安装
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1595746341236&di=2f3dc0c98e66a1dfacc78c7f45bc0ec5&imgtype=0&src=http%3A%2F%2Fdoofuu.com%2Fupload%2F2019%2F06%2F21%2F5d0cd08e17474.jpg
---



RabbitMQ是一个开源的消息代理和队列服务器，用来通过普通协议在完全不同的应用实践共享数据。RabbitMQ使用Erlang语言开发，并且RabbitMQ基于AMQP协议。



------



# AMQP高级消息队列协议

AMQPQ全称：Advanceed Message Queuing Protocol，是具有现代特性的二进制协议，是一个提供统一消息服务的应用层标准高级消息队列协议，是应用层协议的一个开放标准，为面向消息的中间件设计。





## 协议模型

<img src="amqp.png" style="zoom:50%;" />



在这个模型中，生产者（Publisher）将生产的消息发送到消息服务端的交换机（Exchange），消息服务端Exchange将消息保存在队列中，消费者（Consumer）监听消息队列（Message Queue）并从中获取消息进行消费。



## 核心概念

- `server/broker`：接收客户端的连接，实现AMQP实体服务；
- `connection`：应用程序和broker的网络连接；
- `channel`：消息读写的网络信道，所有的操作都在channel中进行，客户端可以建立多个cheannel，每个channel代表一个会话任务；
- `message`：消息，有propertites和body组成；
  - propetities：消息修饰信息，如优先级、延迟等；
  - body：消息的具体内容；
- `virtual host`：虚拟主机，用于进行逻辑隔离，最上层的消息路由；
- `exchange`：交换机，接收消息，根据路由键转发消息到绑定的队列；
- `binding`：绑定，exchange和queue之间的虚拟连接，绑定中还有routing key；
- `routing key`：一个路由规则，虚拟主机可以用它来确定然如何路由一个特定的消息；
- `queue`：消息队列，保存消息并将其撞他转发给消费者；



> 消费者从Queue中获取消息并消费。多个消费者可以订阅同一个Queue，这时Queue中的消息会被平均分摊给多个消费者进行处理，而不是每个消费者都收到所有的消息并处理。



`exchange`具有如下的类型：

- `direct`：完全匹配的路由；
- `topic`：模式匹配的路由；
- `fanout`：广播模式；
- `headers`：键值对匹配路由；



`exchange`具有如下的属性：

- `持久化`：如果起用，则rabbitmq服务重启后仍然存在；
- `自动删除`：如果起用，exchange会在其绑定的队列都被删除后删除自身；







<br>



# RabbitMQ整体架构

<img src="arch.png" style="zoom:50%;" />

> 在架构图中，左侧`P`表示消息生产证，`C`表示消费者，中间的部分为RabbitMQ的`broker`



消息会由生产者直接投递到`exchange`中，`exchange`会将消息传递给`queue`中，消费者从`queue`中消费信息。



<img src="message-trans.png" style="zoom:50%;" />

在上边的消息流转图中看到，消息被发送到`exchange`，`exchange`会绑定一个或多个`queue`，它根据路由策略（routing key）路由到指定的队列。



综上，消息发布的流程大致是：

1. 生产者和Broker建立TCP连接。
2. 生产者和Broker建立通道。
3. 生产者通过通道消息发送给Broker，由Exchange将消息进行转发。
4. Exchange将消息转发到指定的Queue（队列）。



消息消费的大致流程是：

1. 消费者和Broker建立TCP连接 。
2. 消费者和Broker建立通道。
3. 消费者监听指定的Queue（队列）
4. 当有消息到达Queue时Broker默认将消息推送给消费者。
5. 消费者接收到消息。



<br>

# RabbitMQ的优点

- 众多互联网公司都在使用；
- 开源，性能优秀，稳定性可靠；
- 提供可靠的消息投递模式、返回模式；
- 与SpringAMQP完美整合，API丰富；
- 集群模式丰富，表达式配置，HA模式，镜像队列模型；
- 保证数据不丢失的前提下做到高可靠、高可用；



<br>



# 高性能原因

Erlang语言最初用于交换机领域的架构模式，使得RabbitMQ在Broker之间进行数据交互的性能非常优秀，与原生的Socket一样优秀的延迟。



<br>



# 消息可靠性

- `Message acknowledgment`：消息确认，在消息确认机制下，收到回执才会删除消息，未收到回执而断开了连接，消息会转发给其他消费者，如果忘记回执，会导致消息堆积，消费者重启后会重复消费这些消息并重复执行业务逻辑。
- `Message durability`：消息持久化，设置消息持久化可以避免绝大部分消息丢失，比如rabbitmq服务重启，但是采用非持久化可以提升队列的处理效率。如果要确保消息的持久化，那么消息对应的Exchange和Queue同样要设置为持久化。
- `Prefetch count`：每次发送给消费者消息的数量，默认为1



>  如果需要可靠性业务，需要设置持久化和ack机制，如果系统高吞吐，可以设置为非持久化、noack、自动删除机制。


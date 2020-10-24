---
title: consul服务发现
date: 2020-10-24 17:25:26
tags:
- Consul
categories:
- 服务发现
description: 服务发现Consul的基本原理和部署
cover: https://ss1.bdstatic.com/70cFuXSh_Q1YnxGkpoWK1HF6hhy/it/u=3013181417,4005872866&fm=26&gp=0.jpg
---



## 为什么需要服务发现

在使用微服务架构后，原则上是希望每个微服务之间解耦，必然不想让微服务出现硬编码的情况，而且考虑到容灾、水平扩展、提高运维效率等，都应该使用服务发现。



<br>



## consul原理



### 内部原理

在单个数据中心中，Consul分为Client和Server两种节点（所有的节点也被称为Agent），Server节点保存数据，Client负责健康检查及转发数据请求到Server；Server节点有一个Leader和多个Follower，Leader节点会将数据同步到Follower，Server的数量推荐是3个或者5个，在Leader挂掉的时候会启动选举机制产生一个新的Leader。



集群内的Consul节点通过gossip协议（流言协议）维护成员关系，也就是说某个节点了解集群内现在还有哪些节点，这些节点是Client还是Server。单个数据中心的流言协议同时使用TCP和UDP通信，并且都使用8301端口。跨数据中心的流言协议也同时使用TCP和UDP通信，端口使用8302。



集群内数据的读写请求既可以直接发到Server，也可以通过Client使用RPC转发到Server，请求最终会到达Leader节点，在允许数据轻微陈旧的情况下，读请求也可以在普通的Server节点完成，集群内数据的读写和复制都是通过TCP的8300端口完成。



![](datacenter.jpg)

Consul支持多数据中心。

> 在上图中有两个DataCenter，他们通过Internet互联，为了提高通信效率只有Server节点才加入跨数据中心的通信。





### 服务发现原理

![](yuanli.jpg)

一个正常的Consul集群，有Server，有client。这里在服务器Server1、Server2、Server3上分别部署了Consul Server，假设他们选举了Server2上的Consul Server节点为Leader。这些服务器上最好只部署Consul程序，以尽量维护Consul Server的稳定。



然后在服务器Server4和Server5上通过Consul Client分别注册Service A、B、C，这里每个Service分别部署在了两个服务器上，这样可以避免Service的单点问题。服务注册到Consul可以通过HTTP API（8500端口）的方式，也可以通过Consul配置文件的方式。Consul Client可以认为是无状态的，它将注册信息通过RPC转发到Consul Server，服务信息保存在Server的各个节点中，并且通过Raft实现了强一致性。



最后在服务器Server6中Program D需要访问Service B，这时候Program D首先访问本机Consul Client提供的HTTP API，本机Client会将请求转发到Consul Server，Consul Server查询到Service B当前的信息返回，最终Program D拿到了Service B的所有部署的IP和端口，然后就可以选择Service B的其中一个部署并向其发起请求了。如果服务发现采用的是DNS方式，则Program D中直接使用Service B的服务发现域名，域名解析请求首先到达本机DNS代理，然后转发到本机Consul Client，本机Client会将请求转发到Consul Server，Consul Server查询到Service B当前的信息返回，最终Program D拿到了Service B的某个部署的IP和端口。



<br>



## docker方式部署consul集群



### 资源配置

部署4个consul 节点，其中3个为server，1个为client。



### 启动

```bash
#启动第1个Server节点，集群要求要有3个Server，将容器8500端口映射到主机8900端口，同时开启管理界面
docker run -d --name=consul1 -p 8900:8500 -e CONSUL_BIND_INTERFACE=eth0 consul agent --server=true --bootstrap-expect=3 --client=0.0.0.0 -ui

#启动第2个Server节点，并加入集群
docker run -d --name=consul2 -e CONSUL_BIND_INTERFACE=eth0 consul agent --server=true --client=0.0.0.0 --join 172.17.0.2

#启动第3个Server节点，并加入集群
docker run -d --name=consul3 -e CONSUL_BIND_INTERFACE=eth0 consul agent --server=true --client=0.0.0.0 --join 172.17.0.2

#启动第4个Client节点，并加入集群
docker run -d --name=consul4 -e CONSUL_BIND_INTERFACE=eth0 consul agent --server=false --client=0.0.0.0 --join 172.17.0.2
```



通过浏览器访问consul页面：

![](web.jpg)



<br>

## 通过配置文件注册服务

这里有一个hello服务，设置一个配置文件`services.json`，放到容器的`/consul/config`：

```json
{
  "services": [
    {
      "id": "hello1",
      "name": "hello",
      "tags": [
        "primary"
      ],
      "address": "172.17.0.5",
      "port": 5000,
      "checks": [
        {
        "http": "http://localhost:5000/",
        "tls_skip_verify": false,
        "method": "Get",
        "interval": "10s",
        "timeout": "1s"
        }
      ]
    }
  ]
}
```

- name：hello，服务名称，需要能够区分不同的业务服务，可以部署多份并使用相同的name注册。
- id：hello1，服务id，在每个节点上需要唯一，如果有重复会被覆盖。
- address：172.17.0.5，服务所在机器的地址。
- port：5000，服务的端口。
- 健康检查地址：http://localhost:5000/，如果返回HTTP状态码为200就代表服务健康，每10秒Consul请求一次，请求超时时间为1秒。



```bash
# 复制文件
$ docker cp services.json consul4:/consul/config

# 重载配置
$ consul reload
```



通过页面观察，发现注册成功：

![](hello.jpg)



<br>



## 服务发现

调用方获取服务地址的过程就是服务发现。



### http api方式

```bash
$ curl http://127.0.0.1:8500/v1/health/service/hello?passing=true
```



返回的信息包括注册的Consul节点信息、服务信息及服务的健康检查信息。这里用了一个参数`passing=true`，会自动过滤掉不健康的服务，包括本身不健康的服务和不健康的Consul节点上的服务，从这个设计上可以看出Consul将服务的状态绑定到了节点的状态。



如果服务有多个部署，会返回服务的多条信息，调用方需要决定使用哪个部署，常见的可以随机或者轮询。为了提高服务吞吐量，以及减轻Consul的压力，还可以缓存获取到的服务节点信息，不过要做好容错的方案，因为缓存服务部署可能会变得不可用。具体是否缓存需要结合自己的访问量及容错规则来确定。



上边的参数passing默认为false，也就是说不健康的节点也会返回，结合获取节点全部服务的方法，这里可以做到获取全部服务的实时健康状态，并对不健康的服务进行报警处理。



### dns方式

hello服务的域名是：`hello.service.dc1.consul`，后边的service代表服务，固定；dc1是数据中心的名字，可以配置；最后的Consul也可以配置。



官方在介绍DNS方式时经常使用dig命令进行测试，但是alpine系统中没有dig命令，也没有相关的包可以安装，但是有人实现了，下载下来解压到bin目录就可以了。

```bash
$ curl -L https://github.com/sequenceiq/docker-alpine-dig/releases/download/v9.10.2/dig.tgz|tar -xzv -C /usr/local/bin
```



然后执行dig命令：

```bash
$ dig @127.0.0.1 -p 8600 hello.service.dc1.consul. ANY
```

> 如果报错：parse of /etc/resolv.conf failed，请将resolv.conf中的search那行删掉。



正常的话可以看到返回了服务部署的IP信息，如果有多个部署会看到多个，如果某个部署不健康了会自动剔除（包括部署所在节点不健康的情况）。需要注意这种方式不会返回服务的端口信息。



使用DNS的方式可以在程序中集成一个DNS解析库，也可以自定义本地的DNS Server。自定义本地DNS Server是指将.consul域的请求全部转发到Consul Agent，Windows上有DNS Agent，Linux上有Dnsmasq；对于非Consul提供的服务则继续请求原DNS；使用DNS Server时Consul会随机返回具体服务的多个部署中的一个，仅能提供简单的负载均衡。



DNS缓存问题：DNS缓存一般存在于应用程序的网络库、本地DNS客户端或者代理，Consul Sever本身可以认为是没有缓存的（为了提高集群DNS吞吐量，可以设置使用普通Server上的陈旧数据，但影响一般不大），DNS缓存可以减轻Consul Server的访问压力，但是也会导致访问到不可用的服务。使用时需要根据实际访问量和容错能力确定DNS缓存方案。




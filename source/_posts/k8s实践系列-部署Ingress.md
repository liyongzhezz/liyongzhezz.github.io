---
title: '[k8s实践系列]部署Ingress'
date: 2020-07-09 14:09:15
tags:
- k8s
- ingress
categories: 实践K8s
description: 在集群中部署Ingress-nginx将集群内服务暴露出去
cover: https://www.kubernetes.org.cn/img/2019/10/fe9890c6467aa42ebd24d536416942d3-768x442.jpg
---



## 什么是Ingress

Ingress实际是集群的流量统一入口，可以根据不同请求的URL将流量转发给集群中的服务（通过Ingress-Controller实现）。目前有很多Ingress-controller的实现方案：

- `Ingress NGINX`: Kubernetes 官方维护的方案，也是本次安装使用的 Controller；
- `F5 BIG-IP Controller`: F5 所开发的 Controller，它能够让管理员通过 CLI 或 API 让 Kubernetes 与 OpenShift 管理 F5 BIG-IP 设备；
- `Ingress Kong`: 著名的开源 API Gateway 方案所维护的 Kubernetes Ingress Controller；
- `Traefik`: 是一套开源的 HTTP 反向代理与负载均衡器，而它也支援了 Ingress；
- `Voyager`: 一套以 HAProxy 为底的 Ingress Controller；



这里使用的是Ingress Nginx，它实质就是一个nginx服务，他可以通过apiserver动态更新nginx的配置，实现基于域名的虚拟主机工功能。



<br>



## 部署Ingress Nginx

官方维护的配置yaml文件在 [官方YAML文件](https://github.com/kubernetes/ingress-nginx/blob/master/deploy/static/provider/cloud/deploy.yaml)，可以直接执行下面的命令部署：

```bash
$ kubectl apply -f https://github.com/kubernetes/ingress-nginx/blob/master/deploy/static/provider/cloud/deploy.yaml
```



这个文件会创建一个名为：`ingress-nginx`的namespace，相关的pod会部署在这个namespace下面。



有一点需要注意的是，最好修改一下这个文件中deployment的这个地方：

```yaml
spec:
      hostNetwork: true
      dnsPolicy: ClusterFirst
      containers:
        - name: controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.32.0
          imagePullPolicy: IfNotPresent
```

> 这里增加了`hostNetwork: true`，让ingress使用host网络，这样才可以通过默认的service配置被外部访问。



<br>




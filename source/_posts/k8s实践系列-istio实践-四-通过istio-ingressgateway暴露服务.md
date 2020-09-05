---
title: '[k8s实践系列]istio实践(四)--通过istio-ingressgateway暴露服务'
date: 2020-07-21 09:00:18
tags:
- k8s
- Istio
categories:
- 实践K8s
- Istio
description: 通过istio-ingressgateway将服务暴露出去。
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1595303468042&di=e7d983b6f90fa7b6b73ca68d00f346b8&imgtype=0&src=http%3A%2F%2Ft8.baidu.com%2Fit%2Fu%3D3446952059%2C3422785273%26fm%3D193
---



## 准备

在istio部署完成后，会在默认的istio-system下生成一个`istio-ingressgateway`服务。这个服务和传统的`ingress-nginx`类似，但`istio-ingressgateway`允许应用一些监控和路由规则特性来管理进入集群的流量。

```bash
$ kubectl get svc -n istio-system | grep istio-ingressgateway
```

<img src="ingressgateway.png" style="zoom:80%;" />



<br>



## 部署测试服务

这里为了简单，就部署一个官方的nginx服务作为测试服务，首先是deployment文件：

```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: istio-app
  labels:
    app: nginx
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
      version: v1
  template:
    metadata:
      labels:
        app: nginx
        version: v1
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```



然后是service文件：

```yaml
# nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: istio-app
  labels:
    app: nginx
    version: v1
spec:
  ports:
  - name: http
    port: 80
    targetPort: 80
  selector:
    app: nginx
    version: v1
  type: ClusterIP
```



使用下面的命令创建相关的资源：

```bash
$ kubectl apply -f nginx-deployment.yaml
$ kubectl apply -f nginx-service.yaml
```



确保所有的服务都正常启动：

```bash
$ kubectl get pod,svc -n istio-app | grep nginx
```

<img src="check.png" style="zoom:80%;" />



<br>

## 创建Gateway对象

使用下面的文件创建Gateway对象：

```yaml
# nginx-gateway.yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: nginx-proxy
  namespace: istio-app
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "nginx.example.com"
```



执行下面的命令创建：

```bash
$ kubectl apply -f nginx-gateway.yaml
```



<br>

## 创建VirtualService对象

使用下面的文件创建VirtualService对象：

```yaml
# nginx-virtualservice.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: nginx-vs
  namespace: istio-app
spec:
  hosts:
  - "nginx.example.com"
  gateways:
  - nginx-proxy
  http:
  - route:
    - destination:
        host: nginx-service
        port:
          number: 80
```



注意这里的`gateways`名称需要和上边定义的gateway资源名称一致；`destination.host`指定的是服务service名称；



执行下面的命令创建：

```bash
$ kubectl apply -f nginx-virtualservice.yaml
```



<br>



## 配置nginx

使用nginx代理istio-ingressgateway，创建如下的配置文件：

```nginx
# nginx-example.conf
upstream istio-80 {
    server 10.8.138.11:30283 max_fails=3 fail_timeout=5s weight=2;
}

server {
    listen 80;
    server_name nginx.example.com;
    rewrite ^(.*) https://$host$1 permanent;
}

server {
    listen 443 ssl;
    server_name nginx.example.com;

    ssl_certificate /etc/nginx/ssl/nginx.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx.key;

    access_log /var/log/nginx/nginx.example.com_access.log main;
    error_log /var/log/nginx/nginx.example.com_error.log;

    location / {
      proxy_http_version 1.1;
      proxy_pass http://istio-80;
      proxy_set_header Host $http_host;
    }

    location = /favicon.ico {
      log_not_found off;
      access_log off;
    }
  
    location ~* /\.(svn|git)/ {
      return 404;
    }
}
```

- `10.8.138.11:30283`是`istio-ingressgateway`服务对外的IP和端口（http-80）；
- `proxy_http_version 1.1;`这个不能少；



如果要公网访问，即将公网IP地址和nginx服务器绑定，或本地做host解析即可。


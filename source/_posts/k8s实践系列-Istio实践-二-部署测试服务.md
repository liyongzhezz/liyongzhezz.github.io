---
title: '[k8s实践系列]Istio实践(二)--部署测试服务'
date: 2020-07-16 14:51:08
tags:
- k8s
- Istio
categories:
- 实践K8s
- Istio
description: 部署官方的bookinfo测试服务
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1594702185957&di=f0f1cc6419b7f1704a202d56ea54eb1d&imgtype=0&src=http%3A%2F%2Ft9.baidu.com%2Fit%2Fu%3D2440958369%2C359184656%26fm%3D193
---



## 准备

Istio工作依赖于在服务pod中注入代理sidecar，这个功能需要依赖一个标签。是这里新建一个namespace部署bookinfo服务并打上标签，允许istio注入sidecar：

```bash
$ kubectl create ns istio-app
$ kubectl label namespace istio-app istio-injection=enabled
```

<br>



## bookinfo服务架构

<img src="bookinfo.svg" style="zoom:80%;" />

从整体架构图上看，总共分为如下的几个服务：

- `productpage`： 这个微服务会调用 `details` 和 `reviews` 两个微服务，用来生成页面。
- `details`：这个微服务中包含了书籍的信息。
- `reviews`：这个微服务中包含了书籍相关的评论。它还会调用 `ratings` 微服务。有三个版本：
  - v1 版本不会调用 `ratings` 服务。
  - v2 版本会调用 `ratings` 服务，并使用 1 到 5 个黑色星形图标来显示评分信息。
  - v3 版本会调用 `ratings` 服务，并使用 1 到 5 个红色星形图标来显示评分信息。
- `ratings`：这个微服务中包含了由书籍评价组成的评级信息。



<br>



## 部署服务

服务的yaml文件在`istio-1.6.5/samples/bookinfo/platform/kube`下名叫`bookinfo.yaml`，直接部署这个文件：

```bash
$ kubectl apply -f istio-1.6.5/samples/bookinfo/platform/kube/bookinfo.yaml -n istio-app
```

> 注意，一定部署到上边新建的并且打了标签的namespace下，否则sidecar无法注入



查看服务部署情况，确保所有服务都启动并且sidecar注入：

<img src="service.png" style="zoom:50%;" />



<br>



## 对外暴露bookinfo服务

首先创建一个ingress：

```yaml
# bookinfo-ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
   name: bookinfo
   namespace: istio-app
spec:
   rules:
   - host: bookinfo.example.com
     http:
       paths:
       - path: /
         backend:
          serviceName: productpage
          servicePort: 9080
```



执行创建命令：

```bash
$ kubectl apply -f bookinfo-ingress.yaml
```



然后新建一个nginx的配置文件：

```nginx
# bookinfo.conf
server {
    listen 80;
    server_name bookinfo.example.com;
    rewrite ^(.*) https://$host$1 permanent;
}

server {
    listen 443 ssl;
    server_name bookinfo.example.com;

    ssl_certificate /etc/nginx/ssl/nginx.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx.key;

    access_log /var/log/nginx/bookinfo.example.com_access.log main;
    error_log /var/log/nginx/bookinfo.example.com_error.log;

    location ^~/ {
      proxy_pass http://ingress-80;
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



检查nginx配置文件并重载：

```bash
$ nginx -t
$ nginx -s reload
```



本地设置好host解析，通过url访问即可看到服务的页面：

<img src="v1.png" style="zoom:50%;" />



多刷新几次，可以看到不同的reviews服务版本：

<img src="v2.png" style="zoom:50%;" />

<img src="v3.png" style="zoom:50%;" />


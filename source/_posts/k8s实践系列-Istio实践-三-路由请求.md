---
title: '[k8s实践系列]Istio实践(三)--路由请求'
date: 2020-07-16 15:37:51
tags:
- k8s
- Istio
categories:
- 实践K8s
- Istio
description: 将流量路由到不同版本的后端服务
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1594702185957&di=f0f1cc6419b7f1704a202d56ea54eb1d&imgtype=0&src=http%3A%2F%2Ft9.baidu.com%2Fit%2Fu%3D2440958369%2C359184656%26fm%3D193
---



在bookinfo这个服务中，`reviews`服务有三个版本，在访问的时候会重定向到其中一个版本的服务。如果是实际生产场景，应该只允许用户访问其中一个版本，这个就可以用istio的路由请求来实现。



------



## 设定目标规则

目标规则用于根据某种规则将服务分为一些子集，这里我们根据版本标签将服务分组，分组之后就可以配置将流量导入某一个子集中。



官方的标准配置在`istio-1.6.5/samples/bookinfo/networking`下叫`destination-rule-all.yaml`，这里以其中`reviews`服务为例：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
```



从配置中可以看到，`subsets`指定了三个子集`v1,v2,v3`对应三个版本的服务，每个子集通过lable进行服务的匹配。



直接运行这个配置即可：

```bash
$ kubectl apply -f destination-rule-all.yaml -n istio-app
```

<br>



## 应用虚拟服务

虚拟服务是一类服务的抽象，一个虚拟服务中可能有一个或多个服务，在虚拟服务中可以进行灵活的流量配置。



标准的虚拟服务配置在`istio-1.6.5/samples/bookinfo/networking`名叫`virtual-service-all-v1.yaml`，其中大致的配置如下：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
```

> 例如在上边的配置中，配置了名为`reviews`的服务，并将流量转到名为`v1`，host为`reviews`的目标规则中



直接应用这个配置：

```bash
$ kubectl apply -f virtual-service-all-v1.yaml -n istio-app
```

<br>



## 测试

按照目前的配置，我们将流量全部导入v1版本的reviews服务中，在页面上不断刷新，应该只显示v1版本的reviews页面（没有星级评分服务）

<img src="v1.png" style="zoom:50%;" />

<br>



## 基于http header的路由控制

`productpage` 服务在所有到 `reviews` 服务的 HTTP 请求中都增加了一个自定义的 `end-user` 请求头，利用这个header信息可以实现对访问用户的限制。



新建一个yaml配置文件，配置基于header的路由控制：

```yaml
# virtual-service-reviews-v2.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
```



这个配置中，通过匹配`header: end-user == jason`，将路由调度到v2版本，其他的调度到v1版本。使用下面的命令生效该配置：

```bash
$ kubectl apply -f virtual-service-reviews-v2.yaml -n istio-app
```



<br>



## 测试

使用jason用户登录，发现无论怎么刷新，都会请求到v2版本的reviews服务（带有黑白色星星）

<img src="jason.png" style="zoom:50%;" />



而使用其他用户登录，则还是在v1版本：

<img src="peter.png" style="zoom:50%;" />


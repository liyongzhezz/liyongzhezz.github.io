---
title: ingress-nginx实现蓝绿、灰度发布
date: 2021-03-20 15:34:11
tags:
- Ingress
categories:
- Kubernetes
- Ingress-nginx
description: 利用ingress特性实现在k8s中的蓝绿、灰度发布
cover: https://ss1.bdstatic.com/70cFvXSh_Q1YnxGkpoWK1HF6hhy/it/u=175965930,2539474621&fm=26&gp=0.jpg
---



{% note info 'fas fa-bullhorn' %}

本文主要介绍了在k8s中利用 Ingress-nginx 的 Canary 功能实现蓝绿、灰度发布

更新于 2021-03-20

{% endnote %}



<br>



# Canary功能介绍

Ingress-Nginx在0.21版本引入了Canary功能，可以为网关入口配置多个版本的应用程序，使用annotation来控制多个后端服务的流量分配。



canary通过下面的一些注释来实现流量分配：

- `nginx.ingress.kubernetes.io/canary: "true"`：启用Canary功能；
- `nginx.ingress.kubernetes.io/canary-weight` ：指定一个0-100的整数，根据设置的值来决定大概有百分之多少的流量会分配Canary Ingress中指定的后端服务；
- `nginx.ingress.kubernetes.io/canary-by-header` ：基于request header 的流量切分，适用于灰度发布或者A/B测试，当设定的hearder值为always是，请求流量会被一直分配到Canary入口，当hearder值被设置为never时，请求流量不会分配到Canary入口，对于其他hearder值，将忽略，并通过优先级将请求流量分配到其他规则；
- `nginx.ingress.kubernetes.io/canary-by-header-value`： 要和`nginx.ingress.kubernetes.io/canary-by-header` 一起使用，当请求中的hearder key和value 与`nginx.ingress.kubernetes.io/canary-by-header` `nginx.ingress.kubernetes.io/canary-by-header-value`匹配时，请求流量会被分配到Canary Ingress入口，对于其他任何hearder值，将忽略，并通过优先级将请求流量分配到其他规则；
- `nginx.ingress.kubernetes.io/canary-by-cookie` ：基于cookie的流量切分，也适用于灰度发布或者A/B测试，当cookie值设置为always时，请求流量将被路由到Canary Ingress入口，当cookie值设置为never时，请求流量将不会路由到Canary入口，对于其他值，将忽略，并通过优先级将请求流量分配到其他规则；



以上几种注释的优先级为：**canary-by-header - > canary-by-cookie - > canary-weight**



<br>



# 基于权重分配流量

## 创建v1版本服务

{% tabs comments %}

<!-- tab Ingress -->

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
  labels:
    app: echoserverv1
  name: echoserverv1
  namespace: echoserver
spec:
  rules:
  - host: echo.example.com
    http:
      paths:
      - backend:
          serviceName: echoserverv1
          servicePort: 8080
        path: /
```



<!-- endtab -->

<!-- tab Service -->

```yaml
kind: Service
apiVersion: v1
metadata:
  name:  echoserverv1
  namespace: echoserver
spec:
  selector:
    name:  echoserverv1
  type:  ClusterIP
  ports:
  - name:  echoserverv1
    port:  8080
    targetPort:  8080
```

<!-- endtab -->

<!-- tab Deployment -->

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name:  echoserverv1
  namespace: echoserver
  labels:
    name:  echoserverv1
spec:
  template:
    metadata:
      labels:
        name:  echoserverv1
    spec:
      containers:
      - image:  mirrorgooglecontainers/echoserver:1.10
        name:  echoserverv1 
        ports:
        - containerPort:  8080
          name:  echoserverv1
```

<!-- endtab -->

{% endtabs %}



## 检查创建情况

```bash
kubectl get pod,service,ingress -n echoserver
```

<img src="./pod.png" style="zoom:70%;" />



## 访问服务

```bash
for i in `seq 10`;do curl -s echo.example.com|grep Hostname;done
```

<img src="./v1.png" style="zoom:70%;" />



>  可以看到，所有的请求都正常返回且版本为v1



## 发布v2版本服务

{% tabs comments %}

<!-- tab Ingress -->

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "50"
  labels:
    app: echoserverv2
  name: echoserverv2
  namespace: echoserver
spec:
  rules:
  - host: echo.example.com
    http:
      paths:
      - backend:
          serviceName: echoserverv2
          servicePort: 8080
        path: /
```

> 注意这里的ingress就开启了canary功能，切设置权重为50%

<!-- endtab -->

<!-- tab Service -->

```yaml
kind: Service
apiVersion: v1
metadata:
  name:  echoserverv2
  namespace: echoserver
spec:
  selector:
    name:  echoserverv2
  type:  ClusterIP
  ports:
  - name:  echoserverv2
    port:  8080
    targetPort:  8080
```

<!-- endtab -->

<!-- tab Deployment -->

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name:  echoserverv2
  namespace: echoserver
  labels:
    name:  echoserverv2
spec:
  template:
    metadata:
      labels:
        name:  echoserverv2
    spec:
      containers:
      - image:  mirrorgooglecontainers/echoserver:1.10
        name:  echoserverv2 
        ports:
        - containerPort:  8080
          name:  echoserverv2
```

<!-- endtab -->

{% endtabs %}



## 检查v2版本服务

```bash
kubectl get pod,service,ingress -n echoserver
```

<img src="./v2.png" style="zoom:70%;" />



## 访问服务

```bash
for i in `seq 10`;do curl -s echo.example.com|grep Hostname;done
```

<img src="./v2-wight.png" style="zoom:70%;" />



> 可以看到，v1和v2两个版本的请求比几乎是50%





<br>



# 基于header的流量分配

## 更新v2版本服务

{% tabs comments %}

<!-- tab Ingress -->

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "50"
    nginx.ingress.kubernetes.io/canary-by-header: "v2"
  labels:
    app: echoserverv2
  name: echoserverv2
  namespace: echoserver
spec:
  rules:
  - host: echo.example.com
    http:
      paths:
      - backend:
          serviceName: echoserverv2
          servicePort: 8080
        path: /
```

> 注意这里设置了`nginx.ingress.kubernetes.io/canary-by-header`

<!-- endtab -->

<!-- tab Service -->

```yaml
kind: Service
apiVersion: v1
metadata:
  name:  echoserverv2
  namespace: echoserver
spec:
  selector:
    name:  echoserverv2
  type:  ClusterIP
  ports:
  - name:  echoserverv2
    port:  8080
    targetPort:  8080
```

<!-- endtab -->

<!-- tab Deployment -->

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name:  echoserverv2
  namespace: echoserver
  labels:
    name:  echoserverv2
spec:
  template:
    metadata:
      labels:
        name:  echoserverv2
    spec:
      containers:
      - image:  mirrorgooglecontainers/echoserver:1.10
        name:  echoserverv2 
        ports:
        - containerPort:  8080
          name:  echoserverv2
```

<!-- endtab -->

{% endtabs %}



## 访问服务

因为设置了`nginx.ingress.kubernetes.io/canary-by-header: v2`，将会出现以下的集中情况：

{% tabs comments %}

<!-- tab 请求header为 v2:always -->

```bash
for i in `seq 10`;do curl -s -H "v2:always" echo.example.com|grep Hostname;done
```

<img src="./v2-ha.png" style="zoom:70%;" />



> 可以看到请求的header-key为v2，value为always，流量将全部调度到v2版本

<!-- endtab -->

<!-- tab 请求header为 v2:never -->

```bash
for i in `seq 10`;do curl -s -H "v2:never" echo.example.com|grep Hostname;done
```

<img src="./v2-never.png" style="zoom:70%;" />



> 可以看到请求的header-key为v2，value为never，流量将全部调度到v1版本

<!-- endtab -->

<!-- tab 请求header为 v2，header-key任意 -->

```bash
for i in `seq 10`;do curl -s -H "v2:true" echo.example.com|grep Hostname;done
```

<img src="./v2-true.png" style="zoom:70%;" />

> 可以看到请求的header-key为v2，value为其他值，流量将根据权重比例进行分配

<!-- endtab -->

{% endtabs %}



## 按照header key/value进行分配

只需要在v2版本的ingress中指定`nginx.ingress.kubernetes.io/canary-by-header-value`即可：

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "50"
    nginx.ingress.kubernetes.io/canary-by-header: "v2"
    nginx.ingress.kubernetes.io/canary-by-header-value: "true"
  labels:
    app: echoserverv2
  name: echoserverv2
  namespace: echoserver
spec:
  rules:
  - host: echo.example.com
    http:
      paths:
      - backend:
          serviceName: echoserverv2
          servicePort: 8080
        path: /
```



此时因为指定了header key和value，则请求的header必须包含：`v2:true`才能进入v2版本，否则会按照比例进行v1，v2版本分配。



<br>



# 基于cookie进行流量分配

## 更新v2版本服务



基于cookie进行分配的时候，不能定义cookie的value

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "50"
    nginx.ingress.kubernetes.io/canary-by-cookie: "user_from_shanghai"
  labels:
    app: echoserverv2
  name: echoserverv2
  namespace: echoserver
spec:
  rules:
  - host: echo.example.com
    http:
      paths:
      - backend:
          serviceName: echoserverv2
          servicePort: 8080
        path: /
```



## 访问服务

{% tabs comments %}

<!-- tab 设置cookie user_from_shanghai -->

```bash
for i in `seq 10`;do curl -s --cookie "user_from_shanghai" echo.example.com|grep Hostname;done
```



> 请求将按照比例分配到两个版本

<!-- endtab -->

<!-- tab 设置cookie user_from_shanghai:always -->

```bash
for i in `seq 10`;do curl -s --cookie "user_from_shanghai:always" echo.example.com|grep Hostname;done
```



> 请求将按照比例分配到两个版本

<!-- endtab -->

<!-- tab 设置coolie user_from_shanghai=always -->

```bash
for i in `seq 10`;do curl -s --cookie "user_from_shanghai=always" echo.example.com|grep Hostname;done
```

> 请求将全部进入v2版本

<!-- endtab -->

{% endtabs %}


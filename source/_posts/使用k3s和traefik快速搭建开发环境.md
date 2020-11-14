---
title: 使用k3d和traefik快速搭建开发环境
date: 2020-11-14 12:20:35
tags:
- k8s
- k3d
categories:
- 实践K8s
- k3d
description: 使用k3s和treafik快速搭建一个k8s开发环境
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1605337876888&di=2f907b4c0591081e1fffb5b59dd6a571&imgtype=0&src=http%3A%2F%2Fpic2.zhimg.com%2Fv2-7029d91276e5fb38b003606c0ebbae2b_1200x500.jpg
---



一套完成的k8s系统比较复杂，需要很多的资源，部署起来也需要较长的时间。如果是开发环境，对于集群要求快速停止和开启、可以有多个集群并行、使用最少的资源，k3d就可以满足。



<br>



## k3d介绍

k3d是一个非常轻量的集群，可以快速构建用于开发的k8s集群，其具有一下的特点：

- 集群默认使用sqlite而不是etcd存储数据；
- 所有的组件都封装在一个二进制程序中；
- 在docker内运行，快速启停；



<br>



## k3d启动集群



### 安装k3d

直接运行下面的命令安装k3s：

```bash
$ curl -s https://raw.githubusercontent.com/rancher/k3d/main/install.sh | bash
```



这个会在系统中安装`k3d`命令程序，使用下面的命令查看帮助信息：

```bash
$ k3d -h
```





### 启动集群

使用下面的命令启动集群：

```bash
$ k3d cluster create devcluster \
--api-port 127.0.0.1:6443 \
-p 80:80@loadbalancer \
-p 443:443@loadbalancer \
--k3s-server-arg "--no-deploy=traefik"
```



> 默认的k3d绑定的是traefik1，这里使用其他控制器，所以默认不安装



在某些云环境下，只能使用docker的host网络模式，那么创建集群则使用下面的命令：

```bash
$ k3d cluster create devcluster \
--network host \
--api-port 127.0.0.1:6443 \
-p 80:80@loadbalancer \
-p 443:443@loadbalancer \
--k3s-server-arg "--no-deploy=traefik" \
--no-hostip
```



> 指定network类型为host，并不将宿主机ip地址替换到程序中



### 查看集群

查看当前创建的集群直接使用下面的命令：

```bash
$ k3d cluster ls
NAME         SERVERS   AGENTS   LOADBALANCER
devcluster   1/1       0/0      false
```



可以看到当前的集群已经创建好了



### 设置kubectl

首先设置yum源，安装kubectl：

```bash
$ cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

$ yum install -y kubectl-1.17.3
```



然后获取集群`devcluster`的认证信息：

```bash
$ k3d kubeconfig get devcluster > /root/.kube/config
```



现在就可以使用kubectl来操作集群了：

```bash
$ kubectl get node 
NAME                      STATUS   ROLES    AGE   VERSION
k3d-devcluster-server-0   Ready    master   26m   v1.19.3+k3s2

$ kubectl get pod --all-namespaces 
NAMESPACE     NAME                                     READY   STATUS    RESTARTS   AGE
kube-system   metrics-server-7b4f8b595-pghbq           1/1     Running   0          26m
kube-system   local-path-provisioner-7ff9579c6-hmg6k   1/1     Running   0          26m
kube-system   coredns-66c464876b-5lpn5                 1/1     Running   0          26m
```



<br>



## 部署traefik 2

### 安装helm3

```bash
$ wget https://get.helm.sh/helm-v3.2.1-linux-amd64.tar.gz
$ tar zxf helm-v3.2.1-linux-amd64.tar.gz
$ cp linux-amd64/helm /usr/local/bin/
$ helm version
version.BuildInfo{Version:"v3.2.1", GitCommit:"fe51cd1e31e6a202cba7dead9552a6d418ded79a", GitTreeState:"clean", GoVersion:"go1.13.10"}
```



### 安装traefik 2

```bash
$ helm repo add traefik https://containous.github.io/traefik-helm-chart
$ helm repo list
$ helm install traefik traefik/traefik
$ kubectl get pod -n default
NAME                       READY   STATUS    RESTARTS   AGE
svclb-traefik-ldtfr        2/2     Running   0          26s
traefik-5fc4947cf9-w695g   1/1     Running   0          26s
```



### 设置traefik的dashboard

```bash
$ kubectl port-forward $(kubectl get pods --selector "app.kubernetes.io/name=traefik" --output=name) --address 0.0.0.0 9000:9000
```



通过浏览器访问`http://<宿主机IP>:9000/dashboard/`即可进入traefik的dashboard

![](dashboard.png)



<br>



## 应用部署

这里部署一个`whoami`的测试服务：

```bash
$ kubectl create deploy whoami --image containous/whoami
$ kubectl expose deploy whoami --port 80
```



然后创建一个ingress暴露服务让外部访问：

```yaml
# whoami-ingress.yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: whoami
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web,websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
spec:
  rules:
  - http:
      paths:
      - path: /
        backend:
          serviceName: whoami
          servicePort: 80
```



使用下面的命令进行创建：

```bash
$ kubectl apply -f whoami-ingress.yaml
$ kubectl get ingress
```



直接在浏览器，访问：`https://<宿主机IP>`即可：

![](whoami.png)


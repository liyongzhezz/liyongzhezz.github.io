---
title: chaos-mesh简单使用
date: 2020-11-14 14:43:07
tags:
- k8s
- Chaos Mesh
categories:
- 实践K8s
- 混沌工程
description: 混沌工程Chaos Mesh的简单使用
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1605346381963&di=7116962ff868e1f583049b78b5b0b964&imgtype=0&src=http%3A%2F%2Fdownload.pingcap.com%2Fimages%2Fblog%2Fchaos-engineering.png
---



## 混沌工程



### 介绍

混沌工程是在分布式系统上进行实验的学科，目的是建立对系统抵御生产环境中失控条件的能力以及信心。即使分布式系统中的所有单个服务都正常运行, 这些服务之间的交互也会导致不可预知的结果。这些不可预知的结果, 由影响生产环境的罕见且破坏性的事件复合而成，令这些分布式系统存在内在的混沌。



我们采用基于经验和系统的方法解决了分布式系统在规模增长时引发的问题，并以此建立对系统抵御这些事件的能力和信心，通过在受控实验中观察分布式系统的行为来了解它的特性，我们称之为混沌工程。



### Chaos Mesh

Chaos Mesh 是一个云原生混沌工程平台，提供了在 Kubernetes 平台上进行混沌测试的能力。它的特点是对Kubernetes 上的复杂系统进行全方位的故障注入方法，涵盖了 Pod、网络、文件系统甚至内核的故障。



### 架构

Chaos Mesh 主要包括下面两个组件：

- **Chaos Operator**：混沌编排的核心组件。
- **Chaos Dashboard**：用于管理、设计、监控混沌实验的 Web UI。



Chaos Mesh 使用 CRD 来定义混沌对象。Chaos Mesh 的整体架构非常简单，组件部署在 Kubernetes 之上，我们可以使用 YAML 文件或者使用 Chaos mesh Dashboard 上的 Form 来指定场景。其中会有一个 Chaos Daemon，以 Daemonset 的形式运行，对特定节点的网络、Cgroup 等具有系统权限。

![](arch.jpeg)



### 支持的故障注入类型

目前实现的 CRD 对象支持6种类型的故障注入，分别是 PodChaos、NetworkChaos、IOChaos、TimeChaos、StressChaos 和 KernelChaos，对应的主要动作如下所示：

- **pod-kill**：模拟 Kubernetes Pod 被 kill。
- **pod-failure**：模拟 Kubernetes Pod 持续不可用，可以用来模拟节点宕机不可用场景。
- **network-delay**：模拟网络延迟。
- **network-loss**：模拟网络丢包。
- **network-duplication**: 模拟网络包重复。
- **network-corrupt**: 模拟网络包损坏。
- **network-partition**：模拟网络分区。
- **I/O delay** : 模拟文件系统 I/O 延迟。
- **I/O errno**：模拟文件系统 I/O 错误 。



<br>



## 部署



直接使用下面的命令完成一健安装：

```bash
$ curl -sSL https://mirrors.chaos-mesh.org/v1.0.1/install.sh | bash
```



安装后会生成一个`chaos-testing`的namespace：

```bash
$ kubectl get pod -n chaos-testing 
NAME                                        READY   STATUS    RESTARTS   AGE
chaos-controller-manager-69f5dbcf94-69l68   1/1     Running   0          11m
chaos-daemon-msjc6                          1/1     Running   0          11m
chaos-dashboard-69c49f8ff8-xfd9w            1/1     Running   0          11m
```



查看CRD是否安装：

```bash
$ kubectl get crd 
NAME                                   CREATED AT
iochaos.chaos-mesh.org                 2020-11-14T06:30:35Z
kernelchaos.chaos-mesh.org             2020-11-14T06:30:35Z
networkchaos.chaos-mesh.org            2020-11-14T06:30:35Z
podchaos.chaos-mesh.org                2020-11-14T06:30:35Z
podiochaos.chaos-mesh.org              2020-11-14T06:30:35Z
podnetworkchaos.chaos-mesh.org         2020-11-14T06:30:36Z
stresschaos.chaos-mesh.org             2020-11-14T06:30:36Z
timechaos.chaos-mesh.org               2020-11-14T06:30:36Z
```



<br>



## 访问dashboard

chaos-mesh有个dashboard，可以查看任务和新建任务，首先获取dashboard容器的端口：

```bash
$ kubectl get deploy chaos-dashboard -n chaos-testing -o=jsonpath="{.spec.template.spec.containers[0].ports[0].containerPort}{'\n'}"
2333
```



然后暴露出来：

```bash
$ kubectl port-forward -n chaos-testing chaos-dashboard-6fdb79c549-vmvtp --address 0.0.0.0 8888:2333
```



现在就可以通过浏览器访问`http://<宿主机IP>:8888`，查看到chaos-mesh的dashboard：

![](dashboard.png)



<br>



## 创建混沌实验

### 创建服务

```yaml
# pod-chaos.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: appns
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
  namespace: appns
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
```



执行下面的命令完成创建：

```bash
$ kubectl apply -f pod-chaos.yaml
```



> 这里我们创建了一个namespace，并在下边创建了一个3副本的nginx的deployment服务



### 创建实验

假设我们想每分钟随机杀死一个pod，那么实验可以这样写：

```yaml
# chaos-test.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-kill-example
  namespace: chaos-testing
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces:
      - appns
  scheduler:
    cron: "@every 1m"
```



> 使用 `podchaos.chaos-mesh.org` CRD 的 PodChaos对象，动作是`pod-kill`模拟pod被杀死，时间是每分钟执行一次。这里没有标签选择器，所以将杀死`appns`下的随机一个pod



执行下面的命令完成创建：

```bash
$ kubectl apply -f chaos-test.yaml
```



### 检查任务

首先在命令行执行下面的命令，动态查看pod变化情况：

```bash
$ kubectl get pod -n appns --watch
```



然后在dashboard上，可以看到这个实验已经创建：

![](test.jpeg)



然后点击详情按钮，可以看到实验的过程，每分钟杀死一个pod：

![](detail.jpeg)



在终端上，也可以看到pod在每隔一分钟被随机杀死：

```bash
nginx-86c57db685-mf2m9   1/1     Terminating   0          6m
nginx-86c57db685-26cs9   0/1     Pending       0          0s
nginx-86c57db685-26cs9   0/1     Pending       0          0s
nginx-86c57db685-mf2m9   1/1     Terminating   0          6m
nginx-86c57db685-26cs9   0/1     ContainerCreating   0          0s
nginx-86c57db685-26cs9   1/1     Running             0   
```


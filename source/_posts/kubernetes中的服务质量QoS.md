---
title: kubernetes中的服务质量QoS
date: 2021-08-09 22:45:06
tags:
- Kubernetes
categories:
- Kubernetes
- 服务质量
description: 详解kubernetes中的服务Qos
cover: https://gimg2.baidu.com/image_search/src=http%3A%2F%2Fdeveloper.ibm.com%2Fdwblog%2Fwp-content%2Fuploads%2Fsites%2F73%2Fdwblog-kubernetes-850x425.png&refer=http%3A%2F%2Fdeveloper.ibm.com&app=2002&size=f9999,10000&q=a80&n=0&g=0n&fmt=jpeg?sec=1631112416&t=ec53fac9e0c6d318c97d8b1d27623a0a
---



{% note info 'fas fa-bullhorn' %}

本文介绍kubernetes中QoS的相关概念

更新于 2021-08-09

{% endnote %}

<br>



# QoS分类



## 三种QoS类型

kubernetes中QoS分为三类：

- `Guaranteed`
- `Burstable`
- `BestEffort`



> 三种QoS类型，是根据CPU和内存进行划分的；实际中会根据pod不同的QoS类型来采取不同的驱逐、调度策略，尤其是在集群资源紧张的时候。



## QoS类型达成的条件



{% tabs comments %}

<!-- tab Guaranteed -->

`Guaranteed`类型的pod需要满足以下条件：

- Pod 中的每个容器必须指定`mem limit`和`mem reuqest`，且两者必须相等；
- Pod 中的每个容器必须指定`cpu limit`和`cpu request`，且两者必须相等；



例如下面的配置：

```yaml
apiVersion: v1
    kind: Pod
    metadata:
      name: guaranteed-pod
      namespace: default
    spec:
      containers:
        - name: container
          image: busybox
          command: [ /bin/sleep, 1000 ]
          resources:
            limits:
              cpu: 100m
              memory: 10Mi
            requests:
              cpu: 100m
              memory: 10Mi
```

> 在上边的例子中，guaranteed-pod 的cpu `request`和`limit`值相同，memory `request`和`limit`值相同，所以这个pod是一个`Guaranteed`类型的pod；



同时，可以通过下面的命令检查pod是什么QoS类型：

```bash
$ kubectl describe pod limits-and-requests

Name:         guaranteed-pod
Namespace:    default
Priority:     0
Status:       Running
QoS Class:    Guaranteed
```

<!-- endtab -->

<!-- tab Burstable -->

`Burstable`类型的Pod达成的条件是：

- Pod 不符合 Guaranteed QoS 类标准；
- Pod 中至少一个有容器具备内存或 CPU 请求；



例如下面的示例

```yaml
apiVersion: v1
    kind: Pod
    metadata:
      name: brustable-pod
      namespace: default
    spec:
      containers:
        - name: container
          image: busybox
          command: [ /bin/sleep, 10000 ]
          resources:
            limits:
              cpu: 100m
              memory: 10Mi
            requests:
              cpu: 100m
              memory: 0
```



> 也就是说，`Brustable`类型的pod，在调度的时候可能不会考虑节点的资源约束；



通过命令查看pod的QoS类型：

```bash
$ kubectl describe pod limits

Name:         burstable-pod
Namespace:    default
Priority:     0
Status:       Running
QoS Class:    Burstable
```

<!-- endtab -->

<!-- tab BestEffort -->

`BestEffort`类型的pod达成条件是：

- Pod 中的容器不得设置任何内存、CPU 限制或请求；



例如下面的示例：

```yaml
apiVersion: v1
    kind: Pod
    metadata:
      name: besteffort-pod
      namespace: default
    spec:
      containers:
        - name: container
          image: busybox
          command: [ /bin/sleep, 1000 ]
          resources:
            limits:
              cpu: 0
              memory: 0
            requests:
              cpu: 0
              memory: 0
```



> 这样的话，pod在调度和运行时不会关注节点和自身的内存、cpu限制，但是pod实际是否能够调度并运行，还得看具体的节点资源；



通过下面的命令检查pod类型：

```bash
$  kubectl describe pod nothing

Name:         besteffort-pod
Namespace:    default
Priority:     0
Status:       Running
QoS Class:    BestEffort
```

<!-- endtab -->

{% endtabs %}



# OOM发生的原理



## kubelet软驱逐

kubelet会监视节点和pod的资源状态：

- 当节点资源不足时，会主动使一个或多个pod发生故障来回收其占用的资源；
- 当pod使用资源超过limit时，kubelet也会强制重启pod；



kubelet的驱逐顺序如下：

1. 当资源不足时，优先驱逐`Besteffort`类型的pod；
2. `Besteffort`驱逐完成后且资源依然紧张，则驱逐：`Brustable`类型的Pod；
3. 最后驱逐`Guaranteed`类型的pod；



## QoS回收策略



### 系统OOM原理

在遇到较高内存使用压力时，Linux 内核会杀掉一些不太重要的进程，腾出空间保障系统正常运行。它会给每个进程（`/proc/$pid / oom_score`）分配一个得分（`oom_score`），分数越高，被 OOM 的概率就越大。



> 这个参数只反映进程的可用资源在系统中所占的百分比，并没有“该进程有多重要”的概念



`oom_score`分值从-1000-1000，并且可以通过 `oom_score_adj` 来实现，它允许在内存不足的情况下杀死指定进程，具体做法是把可调参数 `/proc/$pid/oom_score_adj` 直接添加到 `badness()` 分数中，使某些任务总是会被考虑 OOM，某些任务则永远不会被 OOM。



### kubernetes的QoS回收原理

kubernetes通过cgroup给pod设置QoS级别，也是利用了系统的OOM机制，OOM分值从0-1000，对于`Guaranteed`级别的 Pod，OOM_ADJ参数设置成了**-998**，对于`Best-Effort`级别的 Pod，OOM_ADJ参数设置成了**1000**，对于`Burstable`级别的 Pod，OOM_ADJ参数取值**从2到999**。



对于 kuberntes 保留资源，比如kubelet，docker，OOM_ADJ参数设置成了**-999**，表示不会被OOM kill掉。OOM_ADJ参数设置的越大，计算出来的OOM分数越高，表明该pod优先级就越低，当出现资源竞争时会越早被kill掉，对于OOM_ADJ参数是-999的表示kubernetes永远不会因为OOM将其kill掉。


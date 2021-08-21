---
title: 使用HPA进行服务自动扩缩容
date: 2021-08-21 16:28:52
tags:
- Kubernetes
categories:
- Kubernetes
- HPA
description: 使用HPA，基于cpu、内存进行服务自动扩缩容
cover: https://gimg2.baidu.com/image_search/src=http%3A%2F%2Fdeveloper.ibm.com%2Fdwblog%2Fwp-content%2Fuploads%2Fsites%2F73%2Fdwblog-kubernetes-850x425.png&refer=http%3A%2F%2Fdeveloper.ibm.com&app=2002&size=f9999,10000&q=a80&n=0&g=0n&fmt=jpeg?sec=1631112416&t=ec53fac9e0c6d318c97d8b1d27623a0a
---



{% note info 'fas fa-bullhorn' %}

本文介绍kubernetes中HPA的用法

更新于 2021-08-21

{% endnote %}

<br>



# HPA

kubernetes中提供了`kubectl scale`命令来实现服务的扩容和缩容，但是这是一个手动的操作。如果要实现自动的基于某些指标的自动扩缩容，就要利用kubernetes提供的资源对象`Horizontal Pod Autoscaling（Pod 水平自动伸缩）`，简称`HPA`。



HPA 通过监控分析一些控制器控制的所有 Pod 的负载变化情况来确定是否需要调整 Pod 的副本数量。



可以简单的通过 `kubectl autoscale` 命令来创建一个 HPA 资源对象，`HPA Controller`默认`30s`轮询一次（可通过 `kube-controller-manager` 的`--horizontal-pod-autoscaler-sync-period` 参数进行设置），查询指定的资源中的 Pod 资源使用率，并且与创建时设定的值和指标做对比，从而实现自动伸缩的功能。



<br>



# metric-server

hpa功能需要利用`metric-server`来获取相关指标数据，`metric-server`通过kubernetes的api将指标暴露出来，例如：

```bash
https://<apiserver>/apis/metrics.k8s.io/v1beta1/namespaces/<namespace-name>/pods/<pod-name>
```



<img src="k8s-hpa-ms.png" style="zoom:80%;" />



部署`metric-server`可以参考 {% post_link 部署MetricServer %}



<br>



# 创建测试服务

使用下面的文件创建一个测试服务：

```yaml
# demo.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-demo
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      volumes:
      - name: increase-mem-script
        configMap:
          name: increase-mem-config
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - name: increase-mem-script
          mountPath: /etc/script
        resources:
          requests:
            memory: 50Mi
            cpu: 50m
        securityContext:
          privileged: true
```



**需要注意的是，Pod 资源必须添加 requests 资源声明，否则hpa不生效。其中挂载的脚本为内存压测脚本，用于增加内存**



创建内存压测脚本：

```yaml
# config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: increase-mem-config
data:
  increase-mem.sh: |
    #!/bin/bash  
    mkdir /tmp/memory  
    mount -t tmpfs -o size=40M tmpfs /tmp/memory  
    dd if=/dev/zero of=/tmp/memory/block  
    sleep 60 
    rm /tmp/memory/block  
    umount /tmp/memory  
    rmdir /tmp/memory
```





直接运行下面的命令创建：

```bash
$ kubectl apply -f config.yaml
$ kubectl apply -f demo.yaml
```



确保服务都正常启动：

```bash
$ kubectl get pod 
```

<img src="pod.png" style="zoom:50%;" />



<br>



# 基于CPU的HPA

使用`kubectl autoscale`命令来创建一个hpa对象：

```bash
$ kubectl autoscale deployment hpa-demo --cpu-percent=10 --min=1 --max=5
```

这里创建了一个关联资源 hpa-demo 的 HPA，最小的 Pod 副本数为1，最大为5。HPA 会根据设定的 cpu 使用率（10%）动态的增加或者减少 Pod 数量。



查看hpa对象创建成功：

```bash
$ kubectl get hpa
```

<img src="hpa-cpu.png" style="zoom:60%;" />



这里我们使用busybox创建一个测试的pod，向nginx服务发送请求：

```bash
$ kubectl run -ti --image busybox test-hpa --restart=Never --rm /bin/sh
/# while true; do wget -q -O- http://172.21.133.86; done
```

> 172.21.133.86 为hpa-demo服务pod的IP



稍等一下即可看到hpa开始工作：

```bash
$ kubectl get hpa
NAME       REFERENCE             TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
hpa-demo   Deployment/hpa-demo   338%/10%   1         5        1          5m15s

$ kubectl get pod --watch 
NAME                        READY   STATUS              RESTARTS   AGE
hpa-demo-6c6489f57-twtdp   1/1     Running             0          5m23s
hpa-demo-6c6489f57-w792g   1/1     Running             0          13s
hpa-demo-6c6489f57-zlxkp   1/1     Running             0          27s
hpa-demo-6c6489f57-znp6q   0/1     ContainerCreating   0          7s
hpa-demo-6c6489f57-ztnvx   1/1     Running             0          6s
```



此时，由于pod的cpu使用率大于了阈值，hpa开始工作，自动创建了pod，最终pod个数达到最大值5。



此时，停止busybox的请求，hpa就会恢复，pod也会缩容到1个：

```bash
$ kubectl get hpa
NAME       REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
hpa-demo   Deployment/hpa-demo   0%/10%    1         5        1          14m

$ kubectl get deployment hpa-demo
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
hpa-demo   1/1     1            1           24m
```



> 可以通过设置 `kube-controller-manager` 组件的`--horizontal-pod-autoscaler-downscale-stabilization` 参数来设置一个持续时间，用于指定在当前操作完成后，`HPA` 必须等待多长时间才能执行另一次缩放操作。默认为5分钟，也就是默认需要等待5分钟后才会开始自动缩放。



<br>



# 基于内存的HPA

创建基于内存的hpa对象：

```yaml
# hpa-mem.yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-demo
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: memory
      targetAverageUtilization: 60
```

> 命令行好像只能创建基于cpu的，内存的需要通过yaml创建。



这里设定pod的内存使用率超过60%时进行扩容。使用下面的命令创建：

```bash
$ kubectl apply -f hpa-mem.yaml
```



确保资源创建成功：

```bash
$ kubectl get hpa
```

<img src="hpa-mem.png" style="zoom:60%;" />



然后进入创建的容器，执行内存压测脚本，增加内存使用率：

```bash
$ kubectl exec -it hpa-demo-66944b79bf-tqrn9 /bin/bash

root@hpa-mem-demo-66944b79bf-tqrn9:/# ls /etc/script/
increase-mem.sh

root@hpa-mem-demo-66944b79bf-tqrn9:/# source /etc/script/increase-mem.sh 
dd: writing to '/tmp/memory/block': No space left on device
81921+0 records in
81920+0 records out
41943040 bytes (42 MB, 40 MiB) copied, 0.584029 s, 71.8 MB/s
```



此时可以看到，内存hpa已经超过阈值，发生了扩容：

```bash
$ kubectl get hpa
NAME        REFERENCE                 TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
nginx-hpa   Deployment/hpa-mem-demo   93%/60%   1         5         1          7m13s

$ kubectl get pods 
NAME                            READY   STATUS    RESTARTS   AGE
hpa-demo-66944b79bf-8m4d9   1/1     Running   0          2m51s
hpa-demo-66944b79bf-tqrn9   1/1     Running   0          8m11s
```



同理，停止脚本后，内存恢复到阈值下，服务进行自动缩容。

<br>






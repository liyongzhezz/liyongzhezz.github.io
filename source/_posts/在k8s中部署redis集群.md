---
title: 在k8s中部署redis集群
date: 2021-03-20 19:28:27
tags:
- Redis
categories:
- 数据库
- Redis
- 部署
description: 在k8s中部署一个高可用redis集群
cover: https://gimg2.baidu.com/image_search/src=http%3A%2F%2F5b0988e595225.cdn.sohucs.com%2Fimages%2F20180327%2F34adc98d775145f0b23c5fa67217af1d.png&refer=http%3A%2F%2F5b0988e595225.cdn.sohucs.com&app=2002&size=f9999,10000&q=a80&n=0&g=0n&fmt=jpeg?sec=1618831808&t=b7f7e1ea802b755f7896decb0502c6a3
---



{% note info 'fas fa-bullhorn' %}

本文主要介绍了在k8s中使用有状态服务Statfulset方式部署一个redis 5.0.5 版本的集群

更新于 2021-03-20

{% endnote %}

<br>



# 创建相关文件

{% tabs comments %}

<!-- tab configmap -->

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-cluster
data:
  update-node.sh: |
    #!/bin/sh
    REDIS_NODES="/data/nodes.conf"
    sed -i -e "/myself/ s/[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}/${POD_IP}/" ${REDIS_NODES}
    exec "$@"
  redis.conf: |+
    cluster-enabled yes
    cluster-require-full-coverage no
    cluster-node-timeout 15000
    cluster-config-file /data/nodes.conf
    cluster-migration-barrier 1
    appendonly yes
    protected-mode no
```

<!-- endtab -->

<!-- tab statefulset -->

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster
spec:
  serviceName: redis-cluster
  replicas: 6
  selector:
    matchLabels:
      app: redis-cluster
  template:
    metadata:
      labels:
        app: redis-cluster
    spec:
      containers:
      - name: redis
        image: redis:5.0.5-alpine
        ports:
        - containerPort: 6379
          name: client
        - containerPort: 16379
          name: gossip
        command: ["/conf/update-node.sh", "redis-server", "/conf/redis.conf"]
        env:
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        volumeMounts:
        - name: conf
          mountPath: /conf
          readOnly: false
        - name: data
          mountPath: /data
          readOnly: false
      volumes:
      - name: conf
        configMap:
          name: redis-cluster
          defaultMode: 0755
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 5Gi
      storageClassName: standard
```

<!-- endtab -->

<!-- tab service -->

```yaml
apiVersion: v1  
kind: Service  
metadata:  
  name: redis-cluster  
spec:  
  type: ClusterIP  
  ports:  
  - port: 6379  
    targetPort: 6379  
    name: client  
  - port: 16379  
    targetPort: 16379  
    name: gossip  
  selector:  
    app: redis-cluster  
```

<!-- endtab -->

{% endtabs %}

<br>



# 创建服务

执行下面的命令创建服务：

```bash
kubectl apply -f configmap.yaml
kubectl apply -f statefulset.yaml
kubectl apply -f service.yaml
```

<img src="./create.png" style="zoom:70%;" />



<br>

# 初始化集群

将前三个节点作为主，后面节点作为从：

```bash
kubectl exec -it redis-cluster-0 -- redis-cli --cluster create --cluster-replicas 1 $(kubectl get pods -l app=redis-cluster -o jsonpath='{range.items[*]}{.status.podIP}:6379 ')  
```

<img src="./cluster.png" style="zoom:70%;" />



<br>



# 验证集群

使用下面的命令验证集群是否正常：

```bash
kubectl exec -it redis-cluster-0 -- redis-cli cluster info  

for x in $(seq 0 5); do echo "redis-cluster-$x"; kubectl exec redis-cluster-$x -- redis-cli role; echo; done 
```

<img src="./info.png" style="zoom:70%;" />

<img src="./detail.png" style="zoom:70%;" />


---
title: 在k8s中部署Mysql单实例
date: 2021-03-21 17:42:30
tags:
- MySQL
categories:
- 数据库
- MySQL
- 部署
description:  在k8s中部署一个mysql单实例
cover: https://ss0.bdstatic.com/70cFvHSh_Q1YnxGkpoWK1HF6hhy/it/u=985528186,1328606288&fm=26&gp=0.jpg
---



{% note info 'fas fa-bullhorn' %}

本文主要介绍了在k8s中部署一个单实例mysql，不具备高可用性只适合于测试开发

更新于 2021-03-21

{% endnote %}

<br>



# 创建service对象

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: default
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
  clusterIP: None
```



执行下面的命令创建：

```bash
kubectl create -f mysql-service.yaml
```



<br>



# 创建存储卷

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```



使用下面的命令创建：

```bash
kubectl create -f mysql-storage.yaml
```



<br>



# 创建deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:5.7
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
```



执行下面的命令创建：

```bash
kubectl create -f mysql-deployment.yaml
```


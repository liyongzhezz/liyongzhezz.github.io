---
title: 在k8s中部署mysql数据库
date: 2020-11-14 13:55:15
tags:
- k8s
- mysql
categories:
- 实践K8s
- 数据库
description: 在k8s中部署mysql服务
cover: https://ss2.bdstatic.com/70cFvnSh_Q1YnxGkpoWK1HF6hhy/it/u=2064508564,168625624&fm=26&gp=0.jpg
---



## 单实例deployment方式



### 创建service对象

```yaml
# mysql-service.yaml
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
$ kubectl create -f mysql-service.yaml
```



### 创建存储卷

```yaml
# mysql-storege.yaml
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
$ kubectl create -f mysql-storage.yaml
```





### 创建deployment

```yaml
# mysql-deployment.yaml
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
$ kubectl create -f mysql-deployment.yaml
```


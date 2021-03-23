---
title: 在k8s中使用nfs存储
date: 2021-03-23 21:16:14
tags:
- NFS
categories:
- Kubernetes
- 存储
- NFS
description: 在k8s中使用nfs作为底层存储的三种姿势
cover: https://ss1.bdstatic.com/70cFuXSh_Q1YnxGkpoWK1HF6hhy/it/u=251466455,608736426&fm=26&gp=0.jpg
---



{% note info 'fas fa-bullhorn' %}

本文主要介绍了在k8s中使用nfs作为存储，实现PV、PVC和storageclass

更新于 2021-03-21

{% endnote %}

<br>



# 安装NFS

直接使用yum命令安装nfs：

```bash
yum install -y nfs-utils rpcbind
```



创建nfs目录：

```bash
mkdir -p /opt/nfs/data
echo "/opt/nfs/data *(rw,no_root_squash)" >> /etc/exports
```

> 生产上应该给该目录挂载一个数据盘



启动nfs：

```bash
systemctl start rpcbind
systemctl status rpcbind
systemctl enable rpcbind
systemctl start nfs 
systemctl status nfs
systemctl enable nfs
```



在k8s的node和master节点上，都安装`nfs-utils`：

```bash
yum install -y nfs-utils
```



<br>



# nfs类型的volume

这种方式是直接在yaml中定义数据卷为nfs类型，示例如下：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-nfs
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-nfs
  template:
    metadata:
      labels:
          app: nginx-nfs
    spec:
      containers:
      - name: nginx-nfs
        image: nginx
        volumeMounts:
        - name: wwwroot
          mountPath: /usr/share/nginx/html
        ports:
        - containerPort: 80
      volumes:
      - name: wwwroot
        nfs:
          server: 10.8.138.8
          path: /opt/nfs/data
```



然后直接apply即可创建一个使用nfs类型volume的pod：

```bash
kubectl apply -f pod-nfs-volume.yaml
```

<img src="./pod-nfs.png" style="zoom:70%;" />



然后可以将这个pod作为service暴露出来：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nfs-svc
  labels:
    app: nginx-nfs-svc
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx-nfs
```



```bash
kubectl apply -f pod-nfs-svc.yaml
```

<img src="./pod-nfs-svc.png" style="zoom:70%;" />



验证访问的话，可以向nfs数据目录中放入一个html文件然后访问：

<img src="./pod-nfs-test.png" style="zoom:70%;" />

<br>





# nfs类型的pv和pvc

## 创建PV

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 500Mi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    path: /opt/nfs/data
    server: 10.8.138.8
```



这里就创建了一个名为`nfs-pv`的pv，大小500Mi，其中指定了nfs的数据目录和nfs地址。

- `accessModes`：pv访问模式，支持如下几个：
  - `ReadWriteOnce`：读写挂载在一个节点上；
  - `ReadOnlyMany`：只读挂载多个节点上；
  - `ReadWriteMany`：读写挂载在多个节点上；
- `persistentVolumeReclaimPolicy`：回收策略，支持以下几个：
  - `Retain`：不作任何操作，需要手动删除（默认）
  - `Recycle`：没有pvc使用时清空数据让其他pvc使用；
  - `Delete`：删除；



<img src="./pv.png" style="zoom:50%;" />



> 可以看到pv目前是可用状态。





## 创建PV

pod的直接消费对象为pvc而非pv，所以还需要创建pvc来绑定pv。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc01
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 500Mi
```

> 这里就创建了一个大小为500Mi，名字为`pvc01`的pvc请求。



pvc是自动绑定pv的，有如下两个原则：

- 根据pvc申请的容量，采用最小配原则匹配到合适的pv并绑定；
- 根据访问模式匹配，绑定pv和pvc访问模式一致的；



<img src="./pvc.png" style="zoom:50%;" />

> 可以看到pvc和pv都是绑定状态了。



## 使用pvc

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-nfs
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-nfs
  template:
    metadata:
      labels:
          app: nginx-nfs
    spec:
      containers:
      - name: nginx-nfs
        image: nginx
        volumeMounts:
        - name: wwwroot
          mountPath: /usr/share/nginx/html
        ports:
        - containerPort: 80
      volumes:
      - name: wwwroot
        persistentVolumeClaim:
          claimName: pvc01
```



 使用pvc的时候，只需要在`volumes`定义的时候制定pvc名称即可。



<img src="./pvc-nginx.png" style="zoom:50%;" />



<br>



# nfs类型的storageclass

**nfs默认不支持storageclass，需要安装额外的插件**



## 安装nfs-client插件

{% tabs comments %}

<!-- tab helm方式部署 -->

```bash
helm install nfs-storageclass --set nfs.server=10.8.138.8 --set nfs.path=/opt/nfs/data --namespace default  stable/nfs-client-provisioner
```



![](./nfs-client.png)

> 可以看到nfs-client服务已经创建，storageclass也已经创建了。

<!-- endtab -->

<!-- tab yaml方式部署 -->

使用yaml文件方式安装，需要部署下面的三个yaml文件来安装nfs-client：

```yaml
# nfs-cluster-role.yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    app: nfs-client-provisioner
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]

---

kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    app: nfs-client-provisioner
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner 
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner 
  apiGroup: rbac.authorization.k8s.io 

---

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    app: nfs-client-provisioner
  name: leader-locking-nfs-client-provisioner
  namespace: kube-system
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]

---

kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    app: nfs-client-provisioner
  name: leader-locking-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner 
    namespace: kube-system
roleRef:
  kind: Role 
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
```

```yaml
# nfs-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  namespace: kube-system
  labels:
    app: nfs-client-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      annotations:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: "quay.io/external_storage/nfs-client-provisioner:v3.1.0-k8s1.11"
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: cluster.local/nfs-client-provisioner
            - name: NFS_SERVER
              value: 10.8.138.8
            - name: NFS_PATH
              value: /data/nfs-data
      volumes:
        - name: nfs-client-root
          nfs:
            server: 10.8.138.8 
            path: /data/nfs-data
```

```yaml
# nfs-service.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app: nfs-client-provisioner 
  name: nfs-client-provisioner 
  namespace: kube-system
```



直接运行这三个文件即可：

```bash
kubectl apply -f nfs-cluster-role.yaml
kubectl apply -f nfs-deployment.yaml
kubectl apply -f nfs-service.yaml
```

<!-- endtab -->

{% endtabs %}



## 创建storageclass

使用类似下面的yaml文件可以创建其他的storageclass：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: es-storageclass
  labels:
    app: nfs-client-provisioner
provisioner: cluster.local/nfs-client-provisioner
allowVolumeExpansion: true
reclaimPolicy: Retain
parameters:
  archiveOnDelete: "true"
```



## pvc使用storageclass

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: testclaim
spec:
  storageClassName: "nfs-client"
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Mi
```



这里创建一个名为`testclaim`的100Mi的pvc，使用`nfs-client`这个storageclass。



![](./storageclass-pvc.png)








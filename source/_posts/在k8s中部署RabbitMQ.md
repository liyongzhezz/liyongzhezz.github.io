---
title: 在k8s中部署RabbitMQ
date: 2021-03-20 12:09:11
tags:
- RabbitMQ
categories:
- 消息中间件
- RabbitMQ
- 部署
description: 本文介绍在k8s中使用有状态服务部署RabbitMQ 3.8.3
cover: https://ss0.bdstatic.com/70cFvHSh_Q1YnxGkpoWK1HF6hhy/it/u=1271185307,1858861969&fm=26&gp=0.jpg
---



{% note info 'fas fa-bullhorn' %}

本文主要介绍了在k8s中使用有状态服务Statfulset方式部署一个RabbitMQ 3.8.3 版本的集群，更新于2021-03-20

{% endnote %}

<br>



# 准备

{% tabs comments %}

<!-- tab K8S集群环境 -->

本文使用的 k8s版本为 `1.20.4`版本，将使用`storageclass`作为持久化存储，底层为`nfs`，使用`default` namespace部署。

<!-- endtab -->

<!-- tab RabbitMq -->

`RabbitMQ`在`k8s`集群中通过`rabbitmq_peer_discovery_k8s plugin`与`k8s apiserver`进行交互，获取各个服务的`URL`，且`RabbitMQ`在`k8s`集群中必须用`statefulset`和`headless service`进行匹配；

> rabbitmq_peer_discovery_k8s`是`RabbitMQ`官方基于第三方开源项目`rabbitmq-autocluster`开发，对`3.7.X`及以上版本提供的`Kubernetes`下的对等发现插件，可实现`rabbitmq`集群在`k8s`中的自动化部署，因此低于3.7.X版本请使用`rabbitmq-autocluster



这里部署使用的版本是`3.8.3`，其他版本信息可以参看官方文档：[rabbitmq download](https://www.rabbitmq.com/download.html)

<!-- endtab -->

{% endtabs %}



<br>



# 部署



## 创建相关文件



{% tabs comments %}

<!-- tab 创建configmap -->

创建一个`rabbitmq-configmap.yaml`的文件：

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: rabbitmq-cluster-config
  namespace: default
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
data:
    enabled_plugins: |
      [rabbitmq_management,rabbitmq_peer_discovery_k8s].
    rabbitmq.conf: |
      default_user = admin
      default_pass = 123!@#
      cluster_formation.peer_discovery_backend = rabbit_peer_discovery_k8s
      cluster_formation.k8s.host = kubernetes.default.svc.cluster.local
      cluster_formation.k8s.address_type = hostname
      cluster_formation.node_cleanup.interval = 30
      cluster_formation.node_cleanup.only_log_warning = true
      cluster_partition_handling = autoheal
      queue_master_locator=min-masters
      loopback_users.guest = false
      cluster_formation.randomized_startup_delay_range.min = 0
      cluster_formation.randomized_startup_delay_range.max = 2
      cluster_formation.k8s.hostname_suffix = .rabbitmq-cluster.default.svc.cluster.local
      vm_memory_high_watermark.absolute = 1GB
      disk_free_limit.absolute = 2GB
```



- `enabled_plugins`：声明开启的插件；
- `default_user/default_pass`：指定用户名和密码；
- `cluster_formation.k8s.address_type`：从`k8s`返回的`Pod`容器列表中计算对等节点列表，这里只能使用主机名，官方示例中是`ip`，但是默认情况下在`k8s`中`pod`的`ip`都是不固定的，因此可能导致节点的配置和数据丢失，后面的`yaml`中会通过引用元数据的方式固定`pod`的主机名；

<!-- endtab -->

<!-- tab 创建service -->

创建一个`rabbitmq-service.yaml`的文件：

```yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    app: rabbitmq-cluster
  name: rabbitmq-cluster
  namespace: default
spec:
  clusterIP: None
  ports:
  - name: rmqport
    port: 5672
    targetPort: 5672
  selector:
    app: rabbitmq-cluster

---
kind: Service
apiVersion: v1
metadata:
  labels:
    app: rabbitmq-cluster
  name: rabbitmq-cluster-manage
  namespace: default
spec:
  ports:
  - name: http
    port: 15672
    protocol: TCP
    targetPort: 15672
  selector:
    app: rabbitmq-cluster
  type: NodePort
```



> 这里定义了两个service，一个是`NodePort`用于用户通过web访问的管理页面，另一个是用于rabbitmq服务通信；

<!-- endtab -->

<!-- tab RBAC -->

创建一个`rabbitmq-rbac.yaml`的文件：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rabbitmq-cluster
  namespace: default
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: rabbitmq-cluster
  namespace: default
rules:
- apiGroups: [""]
  resources: ["endpoints"]
  verbs: ["get"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: rabbitmq-cluster
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: rabbitmq-cluster
subjects:
- kind: ServiceAccount
  name: rabbitmq-cluster
  namespace: default
```

<!-- endtab -->



<!-- tab 创建statefulset -->

创建一个`rabbitmq-cluster-sts.yaml`的文件：

```yaml
kind: StatefulSet
apiVersion: apps/v1
metadata:
  labels:
    app: rabbitmq-cluster
  name: rabbitmq-cluster
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: rabbitmq-cluster
  serviceName: rabbitmq-cluster
  template:
    metadata:
      labels:
        app: rabbitmq-cluster
    spec:
      containers:
      - args:
        - -c
        - cp -v /etc/rabbitmq/rabbitmq.conf ${RABBITMQ_CONFIG_FILE}; exec docker-entrypoint.sh
          rabbitmq-server
        command:
        - sh
        env:
        - name: TZ
          value: 'Asia/Shanghai'
        - name: RABBITMQ_ERLANG_COOKIE
          value: 'SWvCP0Hrqv43NG7GybHC95ntCJKoW8UyNFWnBEWG8TY='
        - name: K8S_SERVICE_NAME
          value: rabbitmq-cluster
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: RABBITMQ_USE_LONGNAME
          value: "true"
        - name: RABBITMQ_NODENAME
          value: rabbit@$(POD_NAME).$(K8S_SERVICE_NAME).$(POD_NAMESPACE).svc.cluster.local
        - name: RABBITMQ_CONFIG_FILE
          value: /var/lib/rabbitmq/rabbitmq.conf
        image: rabbitmq:3.8.3-management
        imagePullPolicy: IfNotPresent
        livenessProbe:
          exec:
            command:
            - rabbitmq-diagnostics
            - status
          initialDelaySeconds: 60
          periodSeconds: 60
          timeoutSeconds: 15
        name: rabbitmq
        ports:
        - containerPort: 15672
          name: http
          protocol: TCP
        - containerPort: 5672
          name: amqp
          protocol: TCP
        readinessProbe:
          exec:
            command:
            - rabbitmq-diagnostics
            - status
          initialDelaySeconds: 20
          periodSeconds: 60
          timeoutSeconds: 10
        volumeMounts:
        - mountPath: /etc/rabbitmq
          name: config-volume
          readOnly: false
        - mountPath: /var/lib/rabbitmq
          name: rabbitmq-storage
          readOnly: false
        - name: timezone
          mountPath: /etc/localtime
          readOnly: true
      serviceAccountName: rabbitmq-cluster
      terminationGracePeriodSeconds: 30
      volumes:
      - name: config-volume
        configMap:
          items:
          - key: rabbitmq.conf
            path: rabbitmq.conf
          - key: enabled_plugins
            path: enabled_plugins
          name: rabbitmq-cluster-config
      - name: timezone
        hostPath:
          path: /usr/share/zoneinfo/Asia/Shanghai
  volumeClaimTemplates:
  - metadata:
      name: rabbitmq-storage
    spec:
      accessModes:
      - ReadWriteMany
      storageClassName: "managed-nfs-storage"
      resources:
        requests:
          storage: 2Gi
```

> 这里指定了storageclass的名字为：`managed-nfs-storage`，需要根据实际情况调整

<!-- endtab -->

{% endtabs %}



## 部署服务

执行下面的命令完成部署：

```bash
kubectl apply -f .
```



## 检查

执行下面的命令检查资源创建情况，确保pod都处于运行状态：

```bash
kubectl get pod,sts -n default
```



<img src="./bushu.png" style="zoom:70%;" />



检查pod日志，观察集群建立情况：

<img src="./log.png" style="zoom:70%;" />



进入pod中，通过客户端命令查看集群状态：

```bash
kubectl exec -ti  rabbitmq-cluster-0 -n default -- rabbitmqctl cluster_status
```

<img src="./client_log.png" style="zoom:70%;" />



## 访问管理页面

查看对应service使用的宿主机端口：

```bash
kubectl get svc -n default
```

<img src="./service.png" style="zoom:70%;" />



通过浏览器访问node节点的30888端口即可打开rabbitmq的管理页面：

<img src="./web.png" style="zoom:70%;" />


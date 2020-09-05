---
title: '[k8s实践系列]EFK+Kafka日志收集(三)--部署zookeeper和kafka集群'
date: 2020-07-09 20:03:25
tags:
- k8s
- k8s日志收集
categories: 
- 实践K8s
- EFK日志收集
description: 部署kafka集群，并安装kafka-manager管理平台
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1594370538704&di=ee50dd4647d528d43e4f3f5ded7058f0&imgtype=0&src=http%3A%2F%2Fstatic.open-open.com%2Fnews%2FuploadImg%2F20160525%2F20160525083516_270.png
---



## 部署zookeeper集群

```yaml
# zookeeper-cluster.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: zk-logging
  namespace: logging
spec:
  podManagementPolicy: Parallel
  replicas: 3
  selector:
    matchLabels:
      app: zk-logging
  serviceName: zk-logging-headless
  template:
    metadata:
      labels:
        app: zk-logging
    spec:
    #   affinity:
    #     podAntiAffinity:
    #       requiredDuringSchedulingIgnoredDuringExecution:
    #         - labelSelector:
    #             matchExpressions:
    #               - key: app
    #                 operator: In
    #                 values:
    #                   - zk-logging
    #           topologyKey: kubernetes.io/hostname
      containers:
        - env:
            - name: ZOO_SERVERS
              value: >-
                server.1=zk-logging-0.zk-logging-headless.logging.svc.cluster.local:2888:3888;2181
                server.2=zk-logging-1.zk-logging-headless.logging.svc.cluster.local:2888:3888;2181
                server.3=zk-logging-2.zk-logging-headless.logging.svc.cluster.local:2888:3888;2181
            - name: ZOO_4LW_COMMANDS_WHITELIST
              value: 'ruok,srvr,conf,stat'
            - name: TZ
              value: Asia/Shanghai
          image: 'zookeeper:3.5.7'
          imagePullPolicy: Always
          name: zk-logging
          ports:
            - containerPort: 2181
              name: client
              protocol: TCP
            - containerPort: 2888
              name: server
              protocol: TCP
            - containerPort: 3888
              name: leader-election
              protocol: TCP
          resources:
            requests:
              cpu: 500m
              memory: 500Mi
          volumeMounts:
            - mountPath: /data
              name: datadir
      dnsPolicy: ClusterFirst
      initContainers:
        - command:
            - sh
            - '-c'
            - >-
              echo $(( $(echo ${POD_NAME} | awk -F "-" '{print $NF}') + 1 )) >
              /data/myid
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.name
          image: 'busybox:1.31.0'
          imagePullPolicy: IfNotPresent
          name: init-zk-logging
          resources: {}
          volumeMounts:
            - mountPath: /data
              name: datadir
    #   nodeSelector:
        # node-role.kubernetes.io/compute: 'true'
      restartPolicy: Always
      securityContext:
        fsGroup: 1000
        runAsUser: 1000
    #   serviceAccount: zk-logging-sa
    #   serviceAccountName: zk-logging-sa
      terminationGracePeriodSeconds: 30
  updateStrategy:
    type: RollingUpdate
  volumeClaimTemplates:
    - metadata:
        creationTimestamp: null
        name: datadir
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 5Gi
        storageClassName: logging-storageclass
```



```bash
$ kubectl apply -f zookeeper-cluster.yaml
```



<br>



## 创建zookeeper service对象

```yaml
# zookeeper-service.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: zk-logging
  name: zk-logging-service
  namespace: logging
spec:
  ports:
    - name: client
      port: 2181
      protocol: TCP
      targetPort: 2181
  selector:
    app: zk-logging
  sessionAffinity: None
  type: ClusterIP
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: zk-logging
  name: zk-logging-headless
  namespace: logging
spec:
  clusterIP: None
  ports:
    - name: server
      port: 2888
      protocol: TCP
      targetPort: 2888
    - name: leader-election
      port: 3888
      protocol: TCP
      targetPort: 3888
  selector:
    app: zk-logging
  sessionAffinity: None
  type: ClusterIP
```



```bash
$ kubectl apply -f zookeeper-service.yaml
```



<br>



## 创建kafka集群

```yaml
# kafka-cluster.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka-logging
  namespace: logging
spec:
  podManagementPolicy: Parallel
  replicas: 3
  selector:
    matchLabels:
      app: kafka-logging
  serviceName: kafka-logging-service
  template:
    metadata:
      labels:
        app: kafka-logging
    spec:
    #   affinity:
    #     podAntiAffinity:
    #       requiredDuringSchedulingIgnoredDuringExecution:
    #         - labelSelector:
    #             matchExpressions:
    #               - key: app
    #                 operator: In
    #                 values:
    #                   - kafka-logging
    #           topologyKey: kubernetes.io/hostname
      containers:
        - env:
            - name: KAFKA_REPLICAS
              value: '3'
            - name: KAFKA_ZK_LOCAL
              value: 'false'
            - name: KAFKA_HEAP_OPTS
              value: '-Xmx1024M -Xms1024M'
            - name: SERVER_num_partitions
              value: '3'
            - name: SERVER_delete_topic_enable
              value: 'true'
            - name: SERVER_log_retention_hours
              value: '2147483647'
            - name: KAFKA_ADVERTISED_PORT
              value: '9092'
            - name: SERVER_zookeeper_connection_timeout_ms
              value: '6000'
            - name: SERVER_log_dirs
              value: /opt/kafka/data/logs
            - name: KAFKA_ADVERTISED_HOST_NAME
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.name
            - name: KAFKA_ZOOKEEPER_CONNECT
              value: 'zk-logging-service.logging.svc.cluster.local:2181'
            - name: BROKER_ID_COMMAND
              value: 'hostname | awk -F ''-'' ''{print $NF}'''
            - name: KAFKA_PORT_NUMBER
              value: '9092'
            - name: KAFKA_ADVERTISED_HOST_NAME
              value: >-
                $(KAFKA_ADVERTISED_HOST_NAME).kafka-logging-service.logging.svc.cluster.local
            - name: TZ
              value: Asia/Shanghai
          image: 'wurstmeister/kafka:2.12-2.5.0'
          imagePullPolicy: Always
          name: kafka-logging
          ports:
            - containerPort: 9092
              name: server
              protocol: TCP
          resources:
            requests:
              cpu: 500m
              memory: 500Mi
          volumeMounts:
            - mountPath: /kafka
              name: datadir
      dnsPolicy: ClusterFirst
    #   nodeSelector:
    #     node-role.kubernetes.io/compute: 'true'
      restartPolicy: Always
      securityContext:
        fsGroup: 0
        runAsUser: 0
    #   serviceAccount: kafka-logging-sa
    #   serviceAccountName: kafka-logging-sa
      terminationGracePeriodSeconds: 30
  updateStrategy:
    type: RollingUpdate
  volumeClaimTemplates:
    - metadata:
        creationTimestamp: null
        name: datadir
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 20Gi
        storageClassName: logging-storageclass
```



```bash
$ kubectl apply -f kafka-cluster.yaml
```



<br>



## 创建kafka service对象

```yaml
# kafka-service.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: kafka-logging
  name: kafka-logging-service
  namespace: logging
spec:
  clusterIP: None
  ports:
    - name: server
      port: 9092
      protocol: TCP
      targetPort: 9092
  selector:
    app: kafka-logging
  sessionAffinity: None
  type: ClusterIP
```



```bash
$ kubectl apply -f kafka-service.yaml
```



<br>



## 部署kafka-manager

kafka-manager是kafka的一个web管理平台，这里部署一下方便在有需要时进行管理。

```yaml
# kafka-manager.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kafka-manager
  name: kafka-manager
  namespace: logging
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kafka-manager
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: kafka-manager
    spec:
      containers:
        - env:
            - name: ZK_HOSTS
              value: 'zk-logging-service.logging.svc:2181'
            - name: TZ
              value: Asia/Shanghai
          image: 'kafkamanager/kafka-manager:2.0.0.2'
          imagePullPolicy: IfNotPresent
          name: kafka-manager
          ports:
            - containerPort: 9000
              name: tcp-9000
              protocol: TCP
          resources: {}
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
```



```bash
$ kubectl apply -f kafka-manager.yaml
```



<br>



## 创建kafka-manager service对象

```yaml
# kafka-manager-service.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: kafka-manager
  name: kafka-manager
  namespace: logging
spec:
  ports:
    - name: kafka-manager
      port: 9000
      protocol: TCP
      targetPort: 9000
  selector:
    app: kafka-manager
  sessionAffinity: None
  type: ClusterIP
```



```bash
$ kubectl apply -f kafka-manager-service.yaml
```



<br>



## 通过ingress暴露kafka-manager服务

```yaml
# kafka-manager-ingress.yaml
kind: Ingress
metadata:
   name: kafka-manager
   namespace: logging
spec:
   rules:
   - host: kafka-manager.example.com
     http:
       paths:
       - path: /
         backend:
          serviceName: kafka-manager
          servicePort: 9000
```



```bash
$ kubectl apply -f kafka-manager-ingress.yaml
```



<br>



## 检查服务

检查所有的pod都正常运行：

```bash
$ kubectl get pod,svc,ingress -n logging | grep -E 'kafka|zk'
```

<img src="kafka-pod.png" style="zoom:50%;" />



<br>

## 创建nginx配置

在nginx服务器上增加kafka-manager配置，代理kafka服务：

```nginx
# /etc/nginx/conf.d/kafka-manager.conf
upstream ingress-80 {
    server 10.8.138.9:80 max_fails=3 fail_timeout=5s weight=2;
    server 10.8.138.11:80 max_fails=3 fail_timeout=5s weight=2;
}

server {
    listen 80;
    server_name kafka-manager.example.com;
    rewrite ^(.*) https://$host$1 permanent;
}

server {
    listen 443 ssl;
    server_name kafka-manager.example.com;

    ssl_certificate /etc/nginx/ssl/nginx.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx.key;

    access_log /var/log/nginx/kafka-manager.example.com_access.log main;
    error_log /var/log/nginx/kafka-manager.example.com_error.log;

    location / {
      proxy_pass http://ingress-80;
      proxy_set_header Host $http_host;
    }

    include conf.d/common.ini;
}
```



检查配置并重载nginx：

```bash
$ nginx -t 
$ nginx -s reload 
```



<br>



## 访问并配置kafka-manager

通过浏览器访问kafka-manager的域名`kafka-manager.example.com`即可进入kafka-manager的控制页面，点击上边的`Cluster`，然后选择`Add Cluster`添加kafka集群，需要填入下面几个信息：

<img src="add-cluster-1.png" style="zoom:50%;" />



<img src="add-cluster-2.png" style="zoom:50%;" />



最后点击`save`后集群信息添加完成。
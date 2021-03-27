---
title: 在k8s中使用EFLK进行日志收集
date: 2021-03-27 12:31:50
tags:
- K8S
- 日志收集
categories:
- Kubernetes
- 日志收集
description: 在kubernetes中使用ELFK进行日志收集和展示
cover: https://ss1.bdstatic.com/70cFvXSh_Q1YnxGkpoWK1HF6hhy/it/u=175965930,2539474621&fm=26&gp=0.jpg
---



{% note info 'fas fa-bullhorn' %}

本文主要介绍了在k8s中使用filebeat、kafka、logstash、elasticsearch、kibana进行日志收集

更新于 2021-03-27

{% endnote %}

<br>



# 架构方案

## 架构

整体收集方案使用如下的组件：

- `filebeat`：采集节点和容器日志，发送到kafka；
- `kafka`：接收filebeat发送的日志消息；
- `logstash`：从kafka中消费日志消息并进行处理；
- `elasticsearch`：进行日志存储；
- `kibana`：日志可视化展示；



> 日志文件 --> filebeat --> kafka --> logstash --> elasticsearch --> kibana



## 版本选择

- `filebeat`：7.6.2；
- `kafka`：2.12-2.5.0；
- `zookeeper`：3.5.7；
- `logstash`：7.8.0；
- `elasticsearch`：7.8.0；
- `kibana`：7.8.0；



## 方案可能存在的问题

这套日志收集方案可能存在下面的问题：

- elasticsearch可能存在瓶颈（es还需要进一步调优）；
- logstash会根据写入es的速度调整消息消费的速度，所有kafka有时候会出现消息堆积的情况；
- ......



<br>



# 准备工作

{% tabs comments %}

<!-- tab 准备nfs相关资源 -->

这里使用nfs作为底层存储，相关的创建方法可以参考：{% post_link 在k8s中使用nfs存储 %}，当然也可以使用其他的存储系统。

<!-- endtab -->

<!-- tab 创建namespace -->

使用下面的yaml文件创建一个namespace：

```yaml
# namesapce.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: logging
```



```bash
kubectl apply -f namespace.yaml
```

<!-- endtab -->

<!-- tab 设置elasticsearch密码 -->

elasticsearch开启x-pack认证功能，这里将elasticsearch的密码保存在secret资源中：

```bash
espassword="elastic"

kubectl create secret generic es-logging-password --from-literal=elastic='elastic' -n logging
```



> 这里讲elasticsearch密码设置为：`elastic`

<!-- endtab -->



{% endtabs %}



<br>



# 部署elasticsearch

## 部署相关资源

{% tabs comments %}

<!-- tab 创建elasticsearch配置 -->

```yaml
# elasticsearch-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: es-logging-config
  namespace: logging
data:
  elasticsearch.yml: |
    cluster.name: es-logging-cluster
    node.name: ${NODE_NAME}
    network.host: 0.0.0.0
    http.port: 9200
    transport.port: 9300
    discovery.seed_hosts: ["es-logging-cluster"]
    cluster.initial_master_nodes:
      - es-logging-0
      - es-logging-1
      - es-logging-2
    xpack.monitoring.collection.enabled: true
    xpack.security.enabled: true
    xpack.license.self_generated.type: basic
    indices.lifecycle.history_index_enabled: false
    xpack.ilm.enabled: true
    
    cluster.routing.allocation.disk.threshold_enabled: false

    # xpack.security.transport.ssl.enabled: true
    # xpack.security.transport.ssl.verification_mode: certificate
    # xpack.security.transport.ssl.key: certs/es-logging-service.key
    # xpack.security.transport.ssl.certificate: certs/es-logging-service.crt
    # xpack.security.transport.ssl.certificate_authorities: certs/ca.crt
    # xpack.security.http.ssl.enabled: false
    # xpack.security.authc.realms:
    #   native.realm1:
    #     order: 0
  es_check.sh: >
    #!/bin/bash

    ES_REST_BASEURL=http://localhost:9200

    EXPECTED_RESPONSE_CODE=200

    max_time=${READINESS_PROBE_TIMEOUT:-30}


    function check_if_ready() {
      path="$1"
      err_msg="$2"
      response_code=$(curl -s -k --head \
          -u elastic:${ELASTIC_PASSWORD} \
          --max-time $max_time \
          -o /dev/null \
          -w '%{response_code}' \
          "${ES_REST_BASEURL}${path}")

      if [ "${response_code}" != ${EXPECTED_RESPONSE_CODE} ]; then
        echo "${err_msg} [response code: ${response_code}]"
        exit 1
      fi
      exit 0
    }


    check_if_ready "/" "Elasticsearch node is not ready to accept HTTP requests yet"
```



配置中的相关参数解释：

- `cluster.name`：设置es集群名称，用于唯一标识一个集群；
- `network.host`：监听的地址；
- `discovery.seed_hosts`：节点发现方式；
- `xpack.security.enabled`起用xpack安全组件；
- `cluster.routing.allocation.disk.threshold_enabled`是否启动磁盘分配器；

> `cluster.routing.allocation.disk.threshold_enabled`这个参数这里设置为false，表示关闭磁盘分配器，这样es可以使用全部磁盘空间。默认情况下当磁盘空间大于85%后，es就不能再创建分片了。这个参数在es使用同一个后端存储的时候应该调节一下。[相关文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-cluster.html#disk-based-shard-allocation)



es启用了x-pack密码认证，并且通过脚本检查es是否正常启动。我这里将es https通信方式注释掉了，如果有需要可以生成相关证书并起用https配置。直接运行下面的命令创建es配置：

```bash
kubectl apply -f elasticsearch-config.yaml
```

<!-- endtab -->

<!-- tab 创建elasticsearch集群 -->

使用下面的yaml文件创建一个namespace：

```yaml
# elasticsearch-cluster.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app: es-logging
  name: es-logging
  namespace: logging
spec:
  replicas: 3
  selector:
    matchLabels:
      app: es-logging
  serviceName: es-logging-cluster
  template:
    metadata:
      labels:
        app: es-logging
      name: es-logging
    spec:
    #   affinity:
    #     podAntiAffinity:
    #       requiredDuringSchedulingIgnoredDuringExecution:
    #         - labelSelector:
    #             matchExpressions:
    #               - key: app
    #                 operator: In
    #                 values:
    #                   - es-logging
    #           topologyKey: kubernetes.io/hostname
      containers:
        - env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.name
            - name: POD_IP
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: status.podIP
            - name: ES_JAVA_OPTS
              value: '-Xms2g -Xmx2g'
            - name: TZ
              value: Asia/Shanghai
            - name: ELASTIC_PASSWORD
              valueFrom:
                secretKeyRef:
                  key: elastic
                  name: es-logging-password
          image: 'docker.elastic.co/elasticsearch/elasticsearch:7.8.0'
          imagePullPolicy: IfNotPresent
          livenessProbe:
            exec:
              command:
                - sh
                - '-c'
                - /usr/share/elasticsearch/config/es_check.sh
            failureThreshold: 3
            initialDelaySeconds: 60
            periodSeconds: 20
            successThreshold: 1
            timeoutSeconds: 1
          name: es-logging
          ports:
            - containerPort: 9200
              name: tcp-9200
              protocol: TCP
            - containerPort: 9300
              name: tcp-9300
              protocol: TCP
          readinessProbe:
            exec:
              command:
                - sh
                - '-c'
                - /usr/share/elasticsearch/config/es_check.sh
            failureThreshold: 3
            initialDelaySeconds: 15
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          resources:
            limits:
              cpu: '2'
          volumeMounts:
            - mountPath: /usr/share/elasticsearch/data
              name: es7x-data
            - mountPath: /usr/share/elasticsearch/config/elasticsearch.yml
              name: es7x-config
              subPath: elasticsearch.yml
            - mountPath: /usr/share/elasticsearch/config/es_check.sh
              name: es7x-config
              subPath: es_check.sh
            # - mountPath: /usr/share/elasticsearch/config/certs
            #   name: es-certs
            #   readOnly: true
        - env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.name
            - name: POD_IP
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: status.podIP
            - name: TZ
              value: Asia/Shanghai
            - name: ELASTIC_PASSWORD
              valueFrom:
                secretKeyRef:
                  key: elastic
                  name: es-logging-password
            - name: ES_URI
              value: 'http://elastic:$(ELASTIC_PASSWORD)@localhost:9200'
            - name: ES_INDICES
              value: 'true'
            - name: ES_ALL
              value: 'true'
            - name: ES_INDICES_SETTINGS
              value: 'true'
          image: 'justwatch/elasticsearch_exporter:1.1.0'
          imagePullPolicy: IfNotPresent
          name: elasticsearch-exporter
          ports:
            - containerPort: 9114
              name: tcp-9114
              protocol: TCP
          resources:
            limits:
              cpu: 200m
              memory: 200Mi
      dnsPolicy: ClusterFirst
      initContainers:
        - command:
            - sysctl
            - '-w'
            - vm.max_map_count=262144
          image: 'busybox:1.31.0'
          imagePullPolicy: IfNotPresent
          name: increase-vm-max-map
          securityContext:
            privileged: true
            runAsUser: 0
        - command:
            - sh
            - '-c'
            - ulimit -n 65536
          image: 'busybox:1.31.0'
          imagePullPolicy: IfNotPresent
          name: increase-fd-ulimit
          securityContext:
            privileged: true
            runAsUser: 0
    #   nodeSelector:
    #     node-role.kubernetes.io/compute: 'true'
      restartPolicy: Always
      securityContext:
        fsGroup: 1000
        runAsUser: 1000
        seLinuxOptions:
          level: 's0:c13,c12'
    #   serviceAccount: es-logging-sa
    #   serviceAccountName: es-logging-sa
      terminationGracePeriodSeconds: 10
      volumes:
        - configMap:
            defaultMode: 493
            name: es-logging-config
          name: es7x-config
        # - name: es-certs
        #   secret:
        #     defaultMode: 420
        #     secretName: es-logging-certs
  updateStrategy:
    type: RollingUpdate
  volumeClaimTemplates:
    - metadata:
        labels:
          name: es-logging
        name: es7x-data
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 20Gi
        storageClassName: logging-storageclass
```



es集群使用了statefulset方式部署，其中storageclass`logging-storageclass`需要自己手动创建，执行下面的命令完成部署。

```bash
kubectl apply -f elasticsearch-cluster.yaml
```

<!-- endtab -->

<!-- tab 创建service -->

```yaml
# elasticsearch-service.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: es-logging
  name: es-logging-service
  namespace: logging
spec:
  ports:
    - name: tcp-9200
      port: 9200
      protocol: TCP
      targetPort: 9200
  selector:
    app: es-logging
  sessionAffinity: None
  type: ClusterIP
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: es-logging
    monitor-app: elasticsearch-exporter
  name: es-logging-cluster
  namespace: logging
spec:
  clusterIP: None
  ports:
    - name: tcp-9300
      port: 9300
      protocol: TCP
      targetPort: 9300
    - name: tcp-9114
      port: 9114
      protocol: TCP
      targetPort: 9114
  selector:
    app: es-logging
  sessionAffinity: None
  type: ClusterIP
```



执行下面的命令完成部署：

```bash
kubeclt apply -f elasticsearch-service.yaml
```

<!-- endtab -->

{% endtabs %}



## 检查

首先确保所有的pod都处于running状态：

```bash
kubectl get pod -n logging | grep es
```

<img src="./es-pod.png" style="zoom:50%;" />



然后执行下面的命令：

```bash
kubectl logs -f es-logging-1 -n logging -c es-logging
```



如果输出的日志中有`Cluster health status changed from [YELLOW] to [GREEN]`，说明集群正常了。



<br>



# 部署kafka

## 部署zookeeper

{% tabs comments %}

<!-- tab 部署zookeeper集群 -->

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



使用下面的命令完成部署：

```bash
kubectl apply -f zookeeper-cluster.yaml
```

<!-- endtab -->

<!-- tab 部署service对象 -->

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



执行下面命令完成部署：

```bash
kubectl apply -f zookeeper-service.yaml
```



<!-- endtab -->

{% endtabs %}



## 部署kafka

{% tabs comments %}

<!-- tab 部署kafka集群 -->

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



使用下面的命令完成部署：

```bash
kubectl apply -f kafka-cluster.yaml
```

<!-- endtab -->

<!-- tab 部署service对象 -->

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



执行下面命令完成部署：

```bash
kubectl apply -f kafka-service.yaml
```

<!-- endtab -->

<!-- tab 部署kafka-manager -->

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
---
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



执行下面命令完成部署：

```bash
kubectl apply -f kafka-manager.yaml
```

<!-- endtab -->

<!-- tab 创建ingress -->

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



执行下面的命令完成创建：

```bash
kubectl apply -f kafka-manager-ingress.yaml
```

<!-- endtab -->

{% endtabs %}



## 检查服务

检查所有的pod都正常运行：

```bash
kubectl get pod,svc,ingress -n logging | grep -E 'kafka|zk'
```

<img src="./kafka-pod.png" style="zoom:50%;" />



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
nginx -t 
nginx -s reload 
```



## 访问并配置kafka-manager

通过浏览器访问kafka-manager的域名`kafka-manager.example.com`即可进入kafka-manager的控制页面，点击上边的`Cluster`，然后选择`Add Cluster`添加kafka集群，需要填入下面几个信息：

<img src="./add-cluster-1.png" style="zoom:50%;" />



<img src="./add-cluster-2.png" style="zoom:50%;" />



最后点击`save`后集群信息添加完成。



<br>



# 部署logstash

##  创建logstash配置

logstash相关的配置文件可以在[logstash配置文件](https://github.com/liyongzhezz/yaml/tree/master/EFK-kafka日志收集) 找到，这里需要将`efk-template.json`, `init_efk.sh`, `k8s-log.json`, `logstash.conf`, `logstash.yml`, `systemd-log.json` 放入一个目录下，例如`logstash-conf`下：

- `efk-template.json`：定义的是针对索引的日志策略；
- `init_efk.sh`：操作es ，初始化一些配置；
- `k8s-log.json`：收集k8s日志的配置；
- `logstash.conf`：logstash的流水线配置；
- `logstash.yml`：logstash配置文件；
- `systemd-log.json`：收集系统日志的配置；



然后执行下面的命令：

```bash
kubectl create configmap logstash-logging-config --from-file=./logstash-conf/ -n logging 
```



<img src="./configmap.png" style="zoom:50%;" />



## 部署logstash

logstash不需要很多实例，所以使用`deployment`方式部署，可以根据需要进行扩展：

```yaml
# logstash-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: logstash-logging
  name: logstash-logging
  namespace: logging
spec:
  replicas: 3
  selector:
    matchLabels:
      app: logstash-logging
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: logstash-logging
      name: logstash-logging
    spec:
      containers:
        - env:
            - name: TZ
              value: Asia/Shanghai
            - name: ELASTIC_PASSWORD
              valueFrom:
                secretKeyRef:
                  key: elastic
                  name: es-logging-password
          image: 'logstash:7.8.0'
          imagePullPolicy: IfNotPresent
          name: logstash-logging
          ports:
            - containerPort: 5044
              name: tcp-5044
              protocol: TCP
            - containerPort: 9600
              name: tcp-9600
              protocol: TCP
          resources: {}
          securityContext:
            privileged: true
            runAsUser: 1000
          volumeMounts:
            - mountPath: /usr/share/logstash/config/logstash.yml
              name: logstash-config
              subPath: logstash.yml
            - mountPath: /usr/share/logstash/pipeline/logstash.conf
              name: logstash-config
              subPath: logstash.conf
            - mountPath: /usr/share/logstash/config/efk-template.json
              name: logstash-config
              subPath: efk-template.json
      dnsPolicy: ClusterFirst
      initContainers:
        - command:
            - sh
            - '-c'
            - /bin/init_efk.sh
          env:
            - name: ELASTIC_PASSWORD
              valueFrom:
                secretKeyRef:
                  key: elastic
                  name: es-logging-password
          image: 'curlimages/curl:latest'
          imagePullPolicy: Always
          name: init-efk
          resources: {}
          volumeMounts:
            - mountPath: /bin/init_efk.sh
              name: logstash-config
              subPath: init_efk.sh
            - mountPath: /tmp/k8s-log.json
              name: logstash-config
              subPath: k8s-log.json
            - mountPath: /tmp/systemd-log.json
              name: logstash-config
              subPath: systemd-log.json
    #   nodeSelector:
    #     node-role.kubernetes.io/compute: 'true'
      restartPolicy: Always
      securityContext:
        fsGroup: 1000
        runAsUser: 1000
        seLinuxOptions:
          level: 's0:c13,c12'
      terminationGracePeriodSeconds: 30
      volumes:
        - configMap:
            defaultMode: 493
            name: logstash-logging-config
          name: logstash-config
```



直接运行下面的命令部署logstash：

```bash
kubectl apply -f logstash-deployment.yaml
```



## 检查服务

确保所有的pod都处于Running状态：

```bash
kubectl get pod -n logging | grep logstash
```

<img src="./logstash-pod.png" style="zoom:50%;" />



<br>



# 部署filebeat

## 部署相关资源

{% tabs comments %}

<!-- tab 创建权限 -->

```yaml
# filebeat-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: filebeat-logging-sa
  namespace: logging
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: filebeat-logging-bind
subjects:
- kind: ServiceAccount
  name: filebeat-logging-sa
  namespace: logging
roleRef:
  kind: ClusterRole
  name: filebeat-logging-clusterrole
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: filebeat-logging-clusterrole
rules:
- apiGroups: [""] # "" indicates the core API group
  resources:
  - namespaces
  - pods
  verbs:
  - get
  - watch
  - list
```



执行下面的命令完成创建：

```yaml
kubectl apply -f filebeat-rbac.yaml
```

<!-- endtab -->

<!-- tab 创建相关配置 -->

```yaml
# filebeat-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-logging-config
  namespace: logging
data:
  filebeat.yml: |
    name: "filebeat-k8s"
    filebeat.registry.path: /var/log/filebeat/registry
    logging.level: warning
    filebeat.inputs:
    - type: container
      scan_frequency: 5s
      enable: true
      symlinks: false
      harvester_buffer_size: 16384
      max_bytes: 10485760
      paths:
      - /var/lib/docker/containers/*/*-json.log
      multiline.pattern: '^[[:space:]]+.+\b|^Caused by:|^[[:alpha:]]+.+Exception:\s\w+.+|^[[:alpha:]]+.+Exception$|^Exception\s\w+.+'
      multiline.negate: false
      multiline.match: after
      multiline.max_lines: 500
      multiline.timeout: 5s
      fields_under_root: true
      overwrite_keys: true

    - type: log
      scan_frequency: 5s
      enable: true
      symlinks: false
      harvester_buffer_size: 16384
      max_bytes: 10485760
      paths:
      - /var/log/messages
      fields_under_root: true
      overwrite_keys: true

    processors:
      - add_kubernetes_metadata:
          in_cluster: true
      - add_host_metadata: 
          netinfo.enabled: true
      - add_locale: ~
      - add_fields:
          target: host
          fields:
            name: ${NODE_NAME}
            ip: ${NODE_IP}
            podip: ${POD_IP}
      - drop_fields:
          fields: ["agent.ephemeral_id","agent.id","log.offset","suricata.eve.timestamp","host.os.codename","host.hostname"]
          ignore_missing: false

    output.kafka:
      hosts: ["kafka-logging-service:9092"]
      version: 2.0.0
      worker: 3
      topics: 
        - topic: "k8s-log.%{[kubernetes.namespace]}"
          when.contains: 
            input.type: "container"
        - topic: "systemd-log.%{[host.name]}"
          when.contains: 
            input.type: "log"
      partition.round_robin:
        reachable_only: false
      required_acks: 1
      compression: gzip
      max_message_bytes: 1000000


    xpack.monitoring.enabled: true
    xpack.monitoring.elasticsearch.hosts: [ "es-logging-service:9200" ]
    xpack.monitoring.elasticsearch.protocol: "http"
    xpack.monitoring.elasticsearch.username: "elastic" 
    xpack.monitoring.elasticsearch.password: "${ELASTIC_PASSWORD}"
```



在配置文件中，指定filebeat去读取docker日志和系统日志，并将日志发送给kafka。filebeat本身不对日志进行处理。执行下面的命令完成创建：

```bash
kubectl apply -f filebeat-configmap.yaml
```

<!-- endtab -->

<!-- tab 部署filebeat -->

每一个工作节点都需要收集日志，所以使用daemonset方式进行部署：

```yaml
# fileat-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: filebeat-logging
    version: v1
  name: filebeat-logging
  namespace: logging
spec:
  selector:
    matchLabels:
      app: filebeat-logging
      version: v1
  template:
    metadata:
      labels:
        app: filebeat-logging
        version: v1
    spec:
      containers:
        - args:
            - '-c'
            - /home/filebeat-config/filebeat.yml
            - '-e'
          env:
            - name: TZ
              value: Asia/Shanghai
            - name: POD_IP
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: status.podIP
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: spec.nodeName
            - name: NODE_IP
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: status.hostIP
            - name: ELASTIC_PASSWORD
              valueFrom:
                secretKeyRef:
                  key: elastic
                  name: es-logging-password
          image: 'elastic/filebeat:7.6.8'
          imagePullPolicy: IfNotPresent
          name: filebeat-logging
          securityContext:
            privileged: true
            runAsUser: 0
          volumeMounts:
            - mountPath: /var/log
              name: filebeat-storage
            - mountPath: /var/log/pods
              name: varlogpods
            - mountPath: /var/lib/docker/containers
              name: varlibdockercontainers
            - mountPath: /home/filebeat-config
              name: filebeat-volume
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      serviceAccount: filebeat-logging-sa
      serviceAccountName: filebeat-logging-sa
      terminationGracePeriodSeconds: 30
      volumes:
        - hostPath:
            path: /var/log
            type: ''
          name: filebeat-storage
        - hostPath:
            path: /var/log/pods
            type: ''
          name: varlogpods
        - hostPath:
            path: /var/lib/docker/containers
            type: ''
          name: varlibdockercontainers
        - configMap:
            defaultMode: 420
            name: filebeat-logging-config
          name: filebeat-volume
  updateStrategy:
    rollingUpdate:
      maxUnavailable: 1
    type: RollingUpdate
```



在yaml文件中，将`/var/lib/docker/containers`和`/var/log`挂载到了容器中，方便容器进行日志收集。

```bash
kubectl apply -f filebeat-daemonset.yaml
```

<!-- endtab -->

{% endtabs %}



## 检查服务

确保所有pod都正常运行：

```bash
kubectl get pod -n logging | grep filebeat
```

<img src="./pod.png" style="zoom:50%;" />



<br>



# 部署kibana

## 部署相关资源

{% tabs comments %}

<!-- tab 创建初始化配置 -->

Kibana相关的配置文件可以在[kibana配置文件](https://github.com/liyongzhezz/yaml/tree/master/EFK-kafka日志收集) 找到，这里需要将`init_kibana.sh` 放入一个目录下，例如`kibana-conf`下，然后执行下面的命令：

```bash
kubectl create configmap kibana-logging-init-config --from-file=./kibana-conf/init_kibana.sh -n logging
```

<!-- endtab -->

<!-- tab 创建配置文件 -->

```yaml
# kibana-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-logging-config
  namespace: logging
data:
  kibana.yml: |
    server.port: 5601
    server.host: "0.0.0.0"
    server.name: "kibana-logging"
    elasticsearch.hosts: ["http://es-logging-service:9200"]
    xpack.monitoring.ui.container.elasticsearch.enabled: true
    xpack.security.enabled: true
    elasticsearch.username: "kibana"
    elasticsearch.password: "${KIBANA_PASSWORD}"
    elasticsearch.requestHeadersWhitelist: [ 'es-security-runas-user',
    'authorization', 'X-Proxy-Remote-User', 'x-forwarded-for',
    'x-forwarded-access-token' ]
    elasticsearch.requestTimeout: 300000
    kibana.index: ".kibana"
    logging.quiet: true
  kibana_check.sh: |
    #!/bin/bash
    KIBANA_REST_BASEURL=http://localhost:5601/login
    EXPECTED_RESPONSE_CODE=200
    max_time="${max_time:-4}"

    response_code="$(
        curl --silent                          \
             --request HEAD                    \
             --head                            \
             --output /dev/null                \
             --max-time "${max_time}"          \
             --write-out '%{response_code}'    \
             "${KIBANA_REST_BASEURL}"
    )"

    if [ "${response_code}" == "${EXPECTED_RESPONSE_CODE}" ]; then
        exit 0
    else
        echo "Kibana node is not ready to accept HTTP requests yet [response code: ${response_code}]"
        exit 1
    fi
```



执行下面的命令完成部署：

```bash
kubectl apply -f kibana-config.yaml
```

<!-- endtab -->

<!-- tab 部署kibana -->

```yaml
# kibana-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kibana-logging
  name: kibana-logging
  namespace: logging
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana-logging
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: kibana-logging
      name: kibana-logging
    spec:
      containers:
        - env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.name
            - name: TZ
              value: Asia/Shanghai
            - name: KIBANA_PASSWORD
              valueFrom:
                secretKeyRef:
                  key: elastic
                  name: es-logging-password
          image: 'kibana:7.8.0'
          name: kibana-logging
          imagePullPolicy: IfNotPresent
          livenessProbe:
            exec:
              command:
                - sh
                - '-c'
                - /usr/share/kibana/config/kibana_check.sh
            failureThreshold: 3
            initialDelaySeconds: 60
            periodSeconds: 20
            successThreshold: 1
            timeoutSeconds: 1
          ports:
            - containerPort: 5601
              name: tcp-5601
              protocol: TCP
          readinessProbe:
            exec:
              command:
                - sh
                - '-c'
                - /usr/share/kibana/config/kibana_check.sh
            failureThreshold: 3
            initialDelaySeconds: 15
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          resources:
            limits:
              cpu: '2'
              memory: 2Gi
            requests:
              cpu: 500m
              memory: 500Mi
          volumeMounts:
            - mountPath: /usr/share/kibana/config/kibana.yml
              name: kibana-config
              subPath: kibana.yml
            - mountPath: /usr/share/kibana/config/kibana_check.sh
              name: kibana-config
              subPath: kibana_check.sh
      dnsPolicy: ClusterFirst
      initContainers:
        - command:
            - sh
            - '-c'
            - /bin/init_kibana.sh
          env:
            - name: ELASTIC_PASSWORD
              valueFrom:
                secretKeyRef:
                  key: elastic
                  name: es-logging-password
          image: 'curlimages/curl:latest'
          imagePullPolicy: Always
          name: init-kibana
          resources: {}
          volumeMounts:
            - mountPath: /bin/init_kibana.sh
              name: kibana-init
              subPath: init_kibana.sh
    #   nodeSelector:
    #     node-role.kubernetes.io/compute: 'true'
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      volumes:
        - configMap:
            defaultMode: 493
            name: kibana-logging-config
          name: kibana-config
        - configMap:
            defaultMode: 493
            name: kibana-logging-init-config
          name: kibana-init
```



执行下面的命令完成部署：

```bash
kubectl apply -f kibana-deployment.yaml
```

<!-- endtab -->

<!-- tab 创建service对象 -->

```yaml
# kibana-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: kibana-logging-service
  namespace: logging
spec:
  ports:
    - name: tcp-5601
      port: 5601
      protocol: TCP
      targetPort: 5601
  selector:
    app: kibana-logging
  sessionAffinity: None
  type: ClusterIP
```



执行下面的命令完成创建：

```bash
kubectl apply -f kibana-service.yaml
```

<!-- endtab -->

<!-- tab 创建ingress对象 -->

```yaml
# kibana-ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
   name: kibana
   namespace: logging
spec:
   rules:
   - host: kibana.example.com
     http:
       paths:
       - path: /
         backend:
          serviceName: kibana-logging-service
          servicePort: 5601
```



执行下面的命令完成创建：

```bash
kubectl apply -f kibana-ingress.yaml
```

<!-- endtab -->

{% endtabs %}



## 配置nginx暴露服务

新增下面的nginx配置:

```nginx
# /etc/nginx/conf.d/kibana.conf
upstream ingress-80 {
    server 10.8.138.9:80 max_fails=3 fail_timeout=5s weight=2;
    server 10.8.138.11:80 max_fails=3 fail_timeout=5s weight=2;
}

server {
    listen 80;
    server_name kibana.example.com;
    rewrite ^(.*) https://$host$1 permanent;
}

server {
    listen 443 ssl;
    server_name kibana.example.com;

    ssl_certificate /etc/nginx/ssl/nginx.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx.key;

    access_log /var/log/nginx/kibana.example.com_access.log main;
    error_log /var/log/nginx/kibana.example.com_error.log;

    location / {
      proxy_pass http://ingress-80;
      proxy_set_header Host $http_host;
    }

    include conf.d/common.ini;
}
```



检查并重载nginx：

```bash
nginx -t
nginx -s reload
```



## 检查服务

确保相关pod都处在Running状态：

```bash
kubectl get pod,svc,ingress -n logging | grep kibana
```

<img src="./kibana-pod.png" style="zoom:50%;" />



## 访问页面

通过浏览器访问`kibana.example.com`即可进入kibana的页面，输入在部署elasticsearch中设置的初始账号密码：`elastic/elastic`。然后在kibana中创建`Index patterns`就可以了。

<img src="./kibana.png" style="zoom:50%;" />



访问kafka-manager观察，发现消息topic也创建出来并且有消息在队列中被logstash消费：

<img src="./kafka.png" style="zoom:50%;" />




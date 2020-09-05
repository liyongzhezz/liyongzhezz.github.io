---
title: '[k8s实践系列]EFK+Kafka日志收集(二)--部署elasticsearch'
date: 2020-07-09 19:54:03
tags: 
- k8s
- k8s日志收集
categories: 
- 实践K8s
- EFK日志收集
description: 部署Elasticsearch集群
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1594302026732&di=01120f85d5d842f277b2ab6c42ef3373&imgtype=0&src=http%3A%2F%2Fwww.ruanyifeng.com%2Fblogimg%2Fasset%2F2017%2Fbg2017081701.jpg
---



## 创建es配置

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
$ kubectl apply -f elasticsearch-config.yaml
```





<br>



## 创建es集群

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
$ kubectl apply -f elasticsearch-cluster.yaml
```



<br>



## 创建es service资源

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



```bash
$ kubeclt apply -f elasticsearch-service.yaml
```



<br>



## 检查

首先确保所有的pod都处于running状态：

```bash
$ kubectl get pod -n logging | grep es
```

<img src="es-pod.png" style="zoom:50%;" />



然后执行下面的命令：

```bash
$ kubectl logs -f es-logging-1 -n logging -c es-logging
```



如果输出的日志中有`Cluster health status changed from [YELLOW] to [GREEN]`，说明集群正常了。


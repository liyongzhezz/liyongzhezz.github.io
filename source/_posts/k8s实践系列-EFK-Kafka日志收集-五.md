---
title: '[k8s实践系列]EFK+Kafka日志收集(五)--部署filebeat'
date: 2020-07-10 11:27:47
tags:
- k8s
- k8s日志收集
categories: 
- 实践K8s
- EFK日志收集
description: 部署filebeat
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1594302026732&di=01120f85d5d842f277b2ab6c42ef3373&imgtype=0&src=http%3A%2F%2Fwww.ruanyifeng.com%2Fblogimg%2Fasset%2F2017%2Fbg2017081701.jpg
---



## 创建filebeat相关权限

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



```bash
$ kubectl apply -f filebeat-rbac.yaml
```

<br>



## 创建相关配置

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



在配置文件中，指定filebeat去读取docker日志和系统日志，并将日志发送给kafka。filebeat本身不对日志进行处理。



```bash
$ kubectl apply -f filebeat-configmap.yaml
```



<br>



## 部署filebeat

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
$ kubectl apply -f filebeat-daemonset.yaml
```



<br>



## 检查服务

确保所有pod都正常运行：

```bash
$ kubectl get pod -n logging | grep filebeat
```

<img src="pod.png" style="zoom:50%;" />
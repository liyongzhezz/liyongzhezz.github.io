---
title: '[k8s实践系列]EFK+Kafka日志收集(四)--部署logstash'
date: 2020-07-10 10:47:57
tags:
- k8s
- k8s日志收集
categories: 
- 实践K8s
- EFK日志收集
description: 部署logstash
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1594370632842&di=f01ac8d28b717c7dcd0e297ae0f25e1c&imgtype=0&src=http%3A%2F%2Fattach.dataguru.cn%2Fattachments%2Fforum%2F201605%2F01%2F224314k0sjhntg3je4qj15.png
---



##  创建logstash配置

logstash相关的配置文件可以在[logstash配置文件](https://github.com/liyongzhezz/yaml/tree/master/EFK-kafka日志收集) 找到，这里需要将`efk-template.json`, `init_efk.sh`, `k8s-log.json`, `logstash.conf`, `logstash.yml`, `systemd-log.json` 放入一个目录下，例如`logstash-conf`下：

- `efk-template.json`：定义的是针对索引的日志策略；
- `init_efk.sh`：操作es ，初始化一些配置；
-  `k8s-log.json`：收集k8s日志的配置；
-  `logstash.conf`：logstash的流水线配置；
- `logstash.yml`：logstash配置文件；
-  `systemd-log.json`：收集系统日志的配置；



然后执行下面的命令：

```bash
$ kubectl create configmap logstash-logging-config --from-file=./logstash-conf/ -n logging 
```



<img src="configmap.png" style="zoom:50%;" />

<br>



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
$ kubectl apply -f logstash-deployment.yaml
```



<br>



## 检查服务

确保所有的pod都处于Running状态：

```bash
$ kubectl get pod -n logging | grep logstash
```

<img src="logstash-pod.png" style="zoom:50%;" />


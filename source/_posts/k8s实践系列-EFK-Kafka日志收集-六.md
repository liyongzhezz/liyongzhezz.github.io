---
title: '[k8s实践系列]EFK+Kafka日志收集(六)--部署kibana'
date: 2020-07-10 11:43:40
tags:
- k8s
- k8s日志收集
categories: 
- 实践K8s
- EFK日志收集
description: 部署kibana
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1594370496285&di=e5ead59539c19a8a8aa1568284feae3f&imgtype=0&src=http%3A%2F%2Fp4.ssl.qhimg.com%2Ft0187295c3528fa925f.png
---



## 创建kibana初始化配置

Kibana相关的配置文件可以在[kibana配置文件](https://github.com/liyongzhezz/yaml/tree/master/EFK-kafka日志收集) 找到，这里需要将`init_kibana.sh` 放入一个目录下，例如`kibana-conf`下，然后执行下面的命令：

```bash
$ kubectl create configmap kibana-logging-init-config --from-file=./kibana-conf/init_kibana.sh -n logging
```



<br>



## 创建kibana配置文件

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



```bash
$ kubectl apply -f kibana-config.yaml
```



<br>



## 部署kibana

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



```bash
$ kubectl apply -f kibana-deployment.yaml
```



<br>



## 创建kibana service对象

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



```bash
$ kubectl apply -f kibana-service.yaml
```



<br>



## 通过ingress暴露服务

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



```bash
$ kubectl apply -f kibana-ingress.yaml
```



新增nginx配置：

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
$ nginx -t
$ nginx -s reload
```



<br>



## 检查服务

确保相关pod都处在Running状态：

```bash
$ kubectl get pod,svc,ingress -n logging | grep kibana
```

<img src="pod.png" style="zoom:50%;" />

<br>



## 访问页面

通过浏览器访问`kibana.example.com`即可进入kibana的页面，输入在第一节设置的初始账号密码：`elastic/elastic`。然后在kibana中创建`Index patterns`就可以了。

<img src="kibana.png" style="zoom:50%;" />



访问kafka-manager观察，发现消息topic也创建出来并且有消息在队列中被logstash消费：

<img src="kafka.png" style="zoom:50%;" />


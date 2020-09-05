---
title: '[k8s实践系列]部署集群监控服务'
date: 2020-07-17 10:59:50
tags:
- k8s
- k8s监控
categories:
- 实践K8s
- k8s监控
description: 部署node-exporter+prometheus+grafana监控kubernetes集群
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1594965034057&di=d1f69b33314f89e42ed55a48f909043b&imgtype=0&src=http%3A%2F%2Fimgcdn.sdk.cn%2Farticle%2FC89u2eYpm4ZIY7BxRTxK.png
---



使用node_exporter + prometheus + grafana 构建一套监控平台。



------



## 组件和架构

在这一套监控服务中，使用到了如下的一些组件：

- `node-exporter`：负责采集服务器监控指标；
- `prometheus`：负责从k8s api和node_exporter周期性获取监控指标并存储数据；
- `grafana`：负责进行数据展示；
- `alertmanager`：根据报警规则进行报警发送。



组件的架构图如下图所示：

<img src="prometheus.png" style="zoom:60%;" />

<br>



## 准备

新建一个namespace用于部署监控组件相关的服务：

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitor
```



```bash
$ kubectl apply -f namespace.yaml
```



<br>



## 部署node-exporter



### 部署daemonset服务

node-exporter用于采集集群节点监控资源，所以采用`daemonset`方式部署：

```yaml
# node-exporter-ds.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitor
  labels:
    k8s-app: node-exporter
spec:
  selector:
    matchLabels:
      k8s-app: node-exporter
  template:
    metadata:
      labels:
        k8s-app: node-exporter
    spec:
      containers:
      - image: 'prom/node-exporter:v1.0.1'
        imagePullPolicy: IfNotPresent
        name: node-exporter
        args:
          - '--path.procfs=/host/proc'
          - '--path.sysfs=/host/sys'
          - '--collector.filesystem.ignored-mount-points=^/(dev|proc|sys)($|/)'
          - >-
            --collector.filesystem.ignored-fs-types=^(autofs|binfmt_misc|cgroup|configfs|debugfs|devpts|devtmpfs|fusectl|hugetlbfs|mqueue|overlay|proc|procfs|pstore|rpc_pipefs|securityfs|sysfs|tracefs)$
          - '--no-collector.wifi'
        ports:
        - containerPort: 9100
          protocol: TCP
          name: http-9100
        resources:
          limits:
            cpu: 400m
            memory: 50Mi
          requests:
            cpu: 200m
            memory: 30Mi
        volumeMounts:
          - mountPath: /host/proc
            name: proc
            readOnly: true
          - mountPath: /host/sys
            name: sys
            readOnly: true
      volumes:
        - hostPath:
            path: /proc
            type: ''
          name: proc
        - hostPath:
            path: /sys
            type: ''
          name: sys
```



执行下面的命令部署node-exporter服务：

```bash
kubectl apply -f node-exporter-ds.yaml
```



### 部署service

部署node-exporter service对象：

```yaml
# node-exporter-service.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: node-exporter
  name: node-exporter
  namespace: monitor
spec:
  ports:
  - name: http-9100
    port: 9100
    nodePort: 31672
    protocol: TCP
  type: NodePort
  selector:
    k8s-app: node-exporter
```



执行下面的命令完成创建：

```bash
$ kubectl apply -f node-exporter-service.yaml
```



### 验证

确保服务正常启动：

<img src="node-exporter.png" style="zoom:60%;" />



node-exporter服务对外暴露了31672端口，可以通过访问node节点的31672端口测试是否能获取到metrics数据：

<img src="check-node.png" style="zoom:60%;" />

<br>



## 部署prometheus



### 创建configmap

通过configmap方式管理prometheus配置文件：

```yaml
# prometheus-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitor
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: EnsureExists
data:
  prometheus.yml: |
    scrape_configs:
    - job_name: node_exporter
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - source_labels: [__meta_kubernetes_role]
        action: replace
        target_label: kubernetes_role
      - source_labels: [__address__]
        regex: '(.*):10250'
        replacement: '${1}:31672'
        target_label: __address__

    - job_name: prometheus
      static_configs:
      - targets:
        - localhost:9090
    - job_name: kubernetes-apiservers
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - action: keep
        regex: default;kubernetes;https
        source_labels:
        - __meta_kubernetes_namespace
        - __meta_kubernetes_service_name
        - __meta_kubernetes_endpoint_port_name
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

    - job_name: kubernetes-nodes-kubelet
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

    - job_name: kubernetes-nodes-cadvisor
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __metrics_path__
        replacement: /metrics/cadvisor
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

    - job_name: kubernetes-service-endpoints
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scrape
      - action: replace
        regex: (https?)
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scheme
        target_label: __scheme__
      - action: replace
        regex: (.+)
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_path
        target_label: __metrics_path__
      - action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        source_labels:
        - __address__
        - __meta_kubernetes_service_annotation_prometheus_io_port
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - action: replace
        source_labels:
        - __meta_kubernetes_namespace
        target_label: kubernetes_namespace
      - action: replace
        source_labels:
        - __meta_kubernetes_service_name
        target_label: kubernetes_name

    - job_name: kubernetes-services
      kubernetes_sd_configs:
      - role: service
      metrics_path: /probe
      params:
        module:
        - http_2xx
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_probe
      - source_labels:
        - __address__
        target_label: __param_target
      - replacement: blackbox
        target_label: __address__
      - source_labels:
        - __param_target
        target_label: instance
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels:
        - __meta_kubernetes_namespace
        target_label: kubernetes_namespace
      - source_labels:
        - __meta_kubernetes_service_name
        target_label: kubernetes_name

    - job_name: 'kubernetes-nodes'
      kubernetes_sd_configs:
      - role: node
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics

    - job_name: kubernetes-pods
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scrape
      - action: replace
        regex: (.+)
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_path
        target_label: __metrics_path__
      - action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        source_labels:
        - __address__
        - __meta_kubernetes_pod_annotation_prometheus_io_port
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - action: replace
        source_labels:
        - __meta_kubernetes_namespace
        target_label: kubernetes_namespace
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_name
        target_label: kubernetes_pod_name
    alerting:
      alertmanagers:
      - kubernetes_sd_configs:
          - role: pod
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - source_labels: [__meta_kubernetes_namespace]
          regex: kube-system
          action: keep
        - source_labels: [__meta_kubernetes_pod_label_k8s_app]
          regex: alertmanager
          action: keep
        - source_labels: [__meta_kubernetes_pod_container_port_number]
          regex:
          action: drop
```



在这个config中定义了常用的监控配置，运行下面的命令创建：

```bash
$ kubectl apply -f prometheus-configmap.yaml
```



### 创建pvc

这里使用storageclass创建一个pvc，用于持久化存储prometheus的数据。后端的存储为nfs。（官方不建议使用nfs）：

```yaml
# prometheus-pvc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: monitor-storageclass
  labels:
    app: nfs-client-provisioner
provisioner: cluster.local/nfs-client-provisioner
allowVolumeExpansion: true
reclaimPolicy: Retain
parameters:
  archiveOnDelete: "true"

---

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: prometheus-data
  namespace: monitor
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 20Gi
  storageClassName: monitor-storageclass
```



执行下面的命令完成创建：

```bash
$ kubectl apply -f prometheus-pvc.yaml
```

<img src="pvc.png" style="zoom:60%;" />



### 服务授权

需要对prometheus进行权限控制，因为它要从apiserver进行监控数据采集：

```yaml
# prometheus-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitor
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: prometheus
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
rules:
  - apiGroups:
      - ""
    resources:
      - nodes
      - nodes/metrics
      - services
      - endpoints
      - pods
      - nodes/proxy
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - get
  - nonResourceURLs:
      - "/metrics"
    verbs:
      - get
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: prometheus
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: monitor
```



执行下面的命令创建：

```bash
$ kubectl apply -f prometheus-rbac.yaml
```



### 部署服务

```yaml
# prometheus-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    name: prometheus-deployment
  name: prometheus
  namespace: monitor
spec:
  selector:
    matchLabels:
      app: prometheus
  replicas: 1
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccountName: prometheus
      containers:
      - image: prom/prometheus:v2.19.2
        name: prometheus
        command:
        - "/bin/prometheus"
        args:
        - "--config.file=/etc/prometheus/prometheus.yml"
        - "--storage.tsdb.path=/prometheus"
        - "--web.console.libraries=/etc/prometheus/console_libraries"
        - "--web.console.templates=/etc/prometheus/consoles"
        - "--web.enable-lifecycle"
        ports:
        - containerPort: 9090
          protocol: TCP
        volumeMounts:
        - mountPath: "/prometheus"
          name: data
          subPath: prometheus
        - mountPath: "/etc/prometheus"
          name: config-volume
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
          limits:
            cpu: 200m
            memory: 1Gi
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: prometheus-data
      - configMap:
          name: prometheus-config
        name: config-volume
```



执行下面的命令部署服务：

```bash
$ kubectl apply -f prometheus-deployment.yaml
```



确保服务正常启动：

<img src="check-prome.png" style="zoom:60%;" />



### 创建service

```yaml
# prometheus-service.yaml
kind: Service
apiVersion: v1
metadata:
  name: prometheus
  namespace: monitor
  labels:
    kubernetes.io/name: "Prometheus"
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  type: ClusterIP
  ports:
    - name: http
      port: 9090
      protocol: TCP
      targetPort: 9090
  selector:
    app: prometheus
```



执行下面的命令创建：

```bash
$ kubectl apply -f prometheus-service.yaml
```



### 创建ingress暴露服务

首先在集群中创建ingress对象资源：

```bash
# prometheus-ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: prometheus
  namespace: monitor
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: prometheus.example.com
    http:
      paths:
      - backend:
          serviceName: prometheus
          servicePort: 9090
        path: /
```



执行下面的命令进行创建：

```bash
$ kubectl apply -f prometheus-ingress.yaml
```



然后在nginx服务器上添加一个配置：

```nginx
# prometheus.conf
server {
    listen 80;
    server_name prometheus.example.com;
    rewrite ^(.*) https://$host$1 permanent;
}

server {
    listen 443 ssl;
    server_name prometheus.example.com;

    ssl_certificate /etc/nginx/ssl/nginx.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx.key;

    access_log /var/log/nginx/prometheus.example.com_access.log main;
    error_log /var/log/nginx/peometheus.example.com_error.log;

    location / {
      proxy_pass http://ingress-80;
      proxy_set_header Host $http_host;
    }

    location = /favicon.ico {
    	log_not_found off;
    	access_log off;
		}

		location ~* /\.(svn|git)/ {
    	return 404;
		}
}
```



检查nginx配置并重载：

```bash
$ nginx -t
$ nginx -s reload 
```



设置好host通过浏览器访问`prometheus.example.com`，正产展示prometheus页面即可：

<img src="prometheus-web.png" style="zoom:60%;" />



<br>



## 部署grafana

这里我是创建了一个空白的grafana服务，即不包含dashboard。可以把需要导入的dashboard作为configmap挂载到pod中，即可实现在创建服务的时候同时生成dashboard



### 创建pvc

使用pvc对grafana的数据进行持久化存储：

```yaml
# grafana-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana-data
  namespace: monitor
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 20Gi
  storageClassName: monitor-storageclass
```



使用下面的命令创建：

```bash
$ kubectl apply -f grafana-pvc.yaml
```

<img src="grafana-pvc.png" style="zoom:60%;" />



### 创建grafana服务

```yaml
# grafana-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitor
  labels:
    app: grafana
spec:
  selector:
    matchLabels:
      app: grafana
  replicas: 1
  template:
    metadata:
      labels:
        app: grafana
    spec:
      securityContext:
        runAsGroup: 472
        runAsUser: 472
        fsGroup: 472
      containers:
      - image: grafana/grafana:7.1.0
        name: grafana-core
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3000
          protocol: TCP
        resources:
          limits:
            cpu: 1000m
            memory: 1000Mi
          requests:
            cpu: 200m
            memory: 256Mi
        env:
          - name: GF_AUTH_DISABLE_LOGIN_FORM
            value: "false"
          - name: GF_AUTH_BASIC_ENABLED
            value: "true"
          - name: GF_AUTH_ANONYMOUS_ENABLED
            value: "false"
          - name: GF_AUTH_ANONYMOUS_ORG_NAME
            value: "Main Org."
          - name: GF_AUTH_ANONYMOUS_ORG_ROLE
            value: "Admin"
          - name: GF_DASHBOARDS_JSON_ENABLED
            value: "true"
          # - name: GF_AUTH_ANONYMOUS_ORG_ROLE
          #   value: Admin
          # does not really work, because of template variables in exported dashboards:
          # - name: GF_DASHBOARDS_JSON_ENABLED
          #   value: "true"
        readinessProbe:
          httpGet:
            path: /login
            port: 3000
          # initialDelaySeconds: 30
          # timeoutSeconds: 1
        volumeMounts:
        - name: grafana-persistent-storage
          mountPath: /var/lib/grafana
      volumes:
      - name: grafana-persistent-storage
        persistentVolumeClaim:
          claimName: grafana-data
```



如果想取消grafana的登录功能，则需要将：

- `GF_AUTH_DISABLE_LOGIN_FORM`设置为false来启用登录表单；
- `GF_AUTH_BASIC_ENABLED`设置为true起用认证；
- `GF_AUTH_ANONYMOUS_ENABLED`设置为false禁用匿名用户登录；



运行下面的命令创建：

```bash
$ kubectl apply -f grafana-deployment.yaml
```



### 创建service

```yaml
# grafana-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitor
  labels:
    app: grafana
spec:
	type: ClusterIP
  ports:
    - name: http
      port: 3000
      targetPort: 3000
      protocol: TCP
  selector:
    app: grafana
```



通过下面的命令创建：

```bash
$ kubectl apply -f grafana-service.yaml
```





### 创建ingress暴露服务

```yaml
# grafana-ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
   name: grafana
   namespace: monitor
spec:
   rules:
   - host: grafana.example.com
     http:
       paths:
       - path: /
         backend:
          serviceName: grafana
          servicePort: 3000
```



使用下面的命令创建：

```bash
$ kubectl apply -f grafana-ingress.yaml
```



在nginx上增加一个配置：

```nginx
# grafana.conf
server {
    listen 80;
    server_name grafana.example.com;
    rewrite ^(.*) https://$host$1 permanent;
}

server {
    listen 443 ssl;
    server_name grafana.example.com;

    ssl_certificate /etc/nginx/ssl/nginx.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx.key;

    access_log /var/log/nginx/grafana.example.com_access.log main;
    error_log /var/log/nginx/grafana.example.com_error.log;

    location / {
      proxy_pass http://ingress-80;
      proxy_set_header Host $http_host;
    }

    location = /favicon.ico {
      log_not_found off;
      access_log off;
		}

		location ~* /\.(svn|git)/ {
    	return 404;
		}
}
```



检查配置并重载：

```bash
$ nginx -t
$ nginx -s reload 
```



通过浏览器访问`grafana.example.com`，默认的密码为`admin/admin`，输入后修改密码即可进入grafana页面：

<img src="grafana-index.png" style="zoom:50%;" />



### 添加数据源

我们的场景中，使用的是prometheus作为数据源的，所以添加一个prometheus的数据源：

<img src="datasource.png" style="zoom:50%;" />



剩下的就是添加dashboard了，可以自己根据promsql编写，也可以在 [grafana dasboard库](https://grafana.com/grafana/dashboards)找需要的模板。
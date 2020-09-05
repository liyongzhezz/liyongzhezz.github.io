---
title: '[k8s实践系列]部署MetricServer'
date: 2020-07-05 13:56:14
tags:
- k8s
categories: 实践K8s
description: 在k8s中部署MetricsServer服务
cover: https://ss1.bdstatic.com/70cFvXSh_Q1YnxGkpoWK1HF6hhy/it/u=175965930,2539474621&fm=26&gp=0.jpg
---



## metrics-server介绍

metrics-server是一个集群内的性能监控服务，可以用来在集群范围可以进行资源使用状况信息的收集，它通过Kubelet收集在各个节点上提供的指标信息。很多功能例如自动扩缩容都依赖它。

<br>



## 前提条件

kube-apiserver需要设置如下的参数：

```yaml
--requestheader-client-ca-file=/opt/kubernetes/cert/ca.pem 
--requestheader-username-headers=X-Remote-User 
--requestheader-group-headers=X-Remote-Group 
--requestheader-extra-headers-prefix=X-Remote-Extra- 
--requestheader-allowed-names=""
--proxy-client-key-file=/opt/kubernetes/cert/st01009vm2-key.pem 
--proxy-client-cert-file=/opt/kubernetes/cert/st01009vm2.pem 
--enable-aggregator-routing=true
--runtime-config=api/all
```

-  --requestheader-XXX、--proxy-client-XXX 是 kube-apiserver 的 aggregator layer 相关的配置参数，metrics-server & HPA 需要使用；
- --requestheader-client-ca-file：用于签名 --proxy-client-cert-file 和 --proxy-client-key-file 指定的证书；在启用了 metric aggregator 时使用；
- 如果 --requestheader-allowed-names 不为空，则--proxy-client-cert-file 证书的 CN 必须位于 allowed-names 中，默认为 aggregator，这里让其匹配所有名称；



> 如果是kubeadm部署，默认已经添加了

<br>



## yaml文件准备

可以从官方获取，地址为：[metrics-server](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/metrics-server)，或者使用下面的yaml文件：

```yaml
# metric-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: metrics-server
  namespace: kube-system
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: metrics-server:system:auth-delegator
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: metrics-server-auth-reader
  namespace: kube-system
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: extension-apiserver-authentication-reader
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
--- 
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:metrics-server
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - nodes
  - nodes/stats
  - namespaces
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - "extensions"
  resources:
  - deployments
  verbs:
  - get
  - list
  - update
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:metrics-server
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:metrics-server
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
```



```yaml
# metric-apiservice.yaml
apiVersion: apiregistration.k8s.io/v1beta1
kind: APIService
metadata:
  name: v1beta1.metrics.k8s.io
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  service:
    name: metrics-server
    namespace: kube-system
  group: metrics.k8s.io
  version: v1beta1
  insecureSkipTLSVerify: true
  groupPriorityMinimum: 100
  versionPriority: 100
```



```yaml
# metric-server-deployment.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: metrics-server-config
  namespace: kube-system
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: EnsureExists
data:
  NannyConfiguration: |-
    apiVersion: nannyconfig/v1alpha1
    kind: NannyConfiguration
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metrics-server-v0.3.6
  namespace: kube-system
  labels:
    k8s-app: metrics-server
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    version: v0.3.6
spec:
  selector:
    matchLabels:
      k8s-app: metrics-server
      version: v0.3.6
  template:
    metadata:
      name: metrics-server
      labels:
        k8s-app: metrics-server
        version: v0.3.6
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
        seccomp.security.alpha.kubernetes.io/pod: 'docker/default'
    spec:
      priorityClassName: system-cluster-critical
      serviceAccountName: metrics-server
      containers:
      - name: metrics-server
        image: mirrorgooglecontainers/metrics-server-amd64:v0.3.6
        command:
        - /metrics-server
        - --metric-resolution=30s
        - --kubelet-insecure-tls
        # These are needed for GKE, which doesn't support secure communication yet.
        # Remove these lines for non-GKE clusters, and when GKE supports token-based auth.
        #- --kubelet-port=10255
        - --kubelet-preferred-address-types=InternalIP,Hostname,InternalDNS,ExternalDNS,ExternalIP
        #- --deprecated-kubelet-completely-insecure=true
        ports:
        - containerPort: 443
          name: https
          protocol: TCP
      - name: metrics-server-nanny
        image: mirrorgooglecontainers/addon-resizer:1.8.4
        resources:
          limits:
            cpu: 100m
            memory: 300Mi
          requests:
            cpu: 5m
            memory: 50Mi
        env:
          - name: MY_POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: MY_POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
        volumeMounts:
        - name: metrics-server-config-volume
          mountPath: /etc/config
        command:
          - /pod_nanny
          - --config-dir=/etc/config
          - --cpu=80m
          - --extra-cpu=0.5m
          - --memory=80Mi
          - --extra-memory=8Mi
          - --threshold=5
          - --deployment=metrics-server-v0.3.6
          - --container=metrics-server
          - --poll-period=300000
          - --estimator=exponential
          # Specifies the smallest cluster (defined in number of nodes)
          # resources will be scaled to.
          #- --minClusterSize={{ metrics_server_min_cluster_size }}
      volumes:
        - name: metrics-server-config-volume
          configMap:
            name: metrics-server-config
      tolerations:
        - key: "CriticalAddonsOnly"
          operator: "Exists"
```



```yaml
# metric-server-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: metrics-server
  namespace: kube-system
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "Metrics-server"
spec:
  selector:
    k8s-app: metrics-server
  ports:
  - port: 443
    protocol: TCP
    targetPort: https
```



<br>



## 部署

将上述yaml文件准备好，直接执行下面的命令进行部署：

```bash
$ kubectl apply -f metric-rbac.yaml
$ kubectl apply -f metric-apiservice.yaml
$ kubectl apply -f metric-server-deployment.yaml
$ kubectl apply -f metric-server-service.yaml
```



<br>



## 检查

部署完成后检查并校验是否能够正常工作：

```bash
$ kubectl get all -n kube-system | grep metrics
$ kubectl top node 
```



![](check.png)
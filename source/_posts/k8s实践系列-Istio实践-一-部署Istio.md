---
title: '[k8s实践系列]Istio实践(一)--部署Istio'
date: 2020-07-14 09:57:06
tags:
- k8s
- Istio
categories:
- 实践K8s
- Istio
description: 部署一个适合于生产环境的Istio。
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1594702185957&di=f0f1cc6419b7f1704a202d56ea54eb1d&imgtype=0&src=http%3A%2F%2Ft9.baidu.com%2Fit%2Fu%3D2440958369%2C359184656%26fm%3D193
---



这里使用的是Istio-1.6.5版本。



------



# 下载Istio

可以在官方下载页面下载对应版本的包：[Download Istio](https://github.com/istio/istio/releases)，也可以使用如下的命令下载：

```bash
# 设定要下载的Istio版本
$ export ISTIO_VERSION="1.6.5"

$ curl -L https://istio.io/downloadIstio | sh -
```



在istio的目录中，有如下的几个目录：

- `samples`：示例应用程序；
- `bin`：客户端工具二进制文件；



设置系统环境变量并开启`istioctl`命令自动补全：

```bash
$ cd istio-1.6.5
$ echo "PATH=$PWD/bin:$PATH" >> /etc/profile
$ cp tools/istioctl.bash /root/
$ echo "source ~/istioctl.bash" >> /root/.bashrc
$ source /etc/profile
```



<br>



# 部署Istio

目前的Istio版本，官方建议使用`istioctl`方式部署。istio下载好后默认有几个配置文件模板：

- `default`：包含最主要的istio组件，适合于生产环境部署；
- `demo`：包含的组件最多，包含了很多额外的组件；
- `minimal`：仅包含`istio-pilot`组件，仅适合测试；
- `sds`：类似于`default`，但是启动了sdas功能；

> 这里就使用`default`配置文件部署



执行下面的命令进行部署：

```bash
# 查看都有哪些profile
$ istioctl profile list

# 使用default配置文件部署
$ istioctl manifest apply --set profile=default

# 如果起用grafana daashboard和kial等组件，则使用下面的命令
$ istioctl manifest apply --set profile=default --set addonComponents.grafana.enabled=true --set addonComponents.tracing.enabled=true --set addonComponents.kiali.enabled=true
```



<img src="istio-install.png" style="zoom:50%;" />

<br>



# 检查服务情况

默认会创建一个名为`istio-system`的namespace，istio相关服务都会放在这个namespace下：

```bash
$ kubectl get pod,svc -n istio-system
```

<img src="check.png" style="zoom:50%;" />



<br>



# 服务要求

## namespace

当使用 `kubectl apply` 来部署应用时，如果 pod 启动在标有 `istio-injection=enabled` 的命名空间中，那么，[Istio sidecar 注入器](https://istio.io/zh/docs/setup/additional-setup/sidecar-injection/#automatic-sidecar-injection)将自动注入 Envoy 容器到应用的 pod 中：

```shell
$ kubectl label namespace <namespace> istio-injection=enabled
$ kubectl create -n <namespace> -f <your-app-spec>.yaml
```



在没有 `istio-injection` 标记的命名空间中，在部署前可以使用 [`istioctl kube-inject`](https://istio.io/zh/docs/reference/commands/istioctl/#istioctl-kube-inject) 命令将 Envoy 容器手动注入到应用的 pod 中：

```shell
$ istioctl kube-inject -f <your-app-spec>.yaml | kubectl apply -f -
```



## pod和service

如果服务要使用istio的相关功能，只注入sidecar是不够的，pod和service还需要满足如下的需求：

- **命名的服务端口**: Service 的端口必须命名。端口名键值对必须按以下格式：`name: <protocol>[-<suffix>]`。更多说明请参看[协议选择](https://istio.io/zh/docs/ops/configuration/traffic-management/protocol-selection/)。
- **Service 关联**: 每个 Pod 必须至少属于一个 Kubernetes Service，不管这个 Pod 是否对外暴露端口。如果一个 Pod 同时属于多个 [Kubernetes Service](https://kubernetes.io/docs/concepts/services-networking/service/)， 那么这些 Service 不能同时在一个端口号上使用不同的协议（比如：HTTP 和 TCP）。
- **带有 app 和 version 标签（label） 的 Deployment**: 我们建议显式地给 Deployment 加上 `app` 和 `version` 标签。给使用 Kubernetes `Deployment` 部署的 Pod 部署配置中增加这些标签，可以给 Istio 收集的指标和遥测信息中增加上下文信息。
  - `app` 标签：每个部署配置应该有一个不同的 `app` 标签并且该标签的值应该有一定意义。`app` label 用于在分布式追踪中添加上下文信息。
  - `version` 标签：这个标签用于在特定方式部署的应用中表示版本。
- **应用 UID**: 确保你的 Pod 不会以用户 ID（UID）为 1337 的用户运行应用。
- **`NET_ADMIN` 功能**: 如果你的集群执行 Pod 安全策略，必须给 Pod 配置 `NET_ADMIN` 功能。如果你使用 [Istio CNI 插件](https://istio.io/zh/docs/setup/additional-setup/cni/) 可以不配置。要了解更多 `NET_ADMIN` 功能的知识，请查看[所需的 Pod 功能](https://istio.io/zh/docs/ops/deployment/requirements/#required-pod-capabilities)。



<br>



# 卸载

```bash
$ istioctl manifest generate --set profile=default | kubectl delete -f -
```





<br>

# 创建kiali秘钥

可以看到创建了一个叫kiali的服务，这是一个链路监控的服务，但是要使用这个服务，需要设置一个初始的秘钥文件。



这里我们设置初始的用户名和密码为`admin`，并用base64加密：

```bash
$ echo -n 'admin' | base64
YWRtaW4=
```



创建一个secret资源类型：

```yaml
# kiali-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: kiali
  namespace: istio-system
  labels:
    app: kiali
type: Opaque
data:
  username: YWRtaW4=
  passphrase: YWRtaW4=
```

> kiali服务会挂载名为`kiali`的secret到容器中，设置初始用户和密码



创建并重启kiali服务：

```bash
$ kubectl apply -f kiali-secret.yaml
$ kubectl delete pod $(kubectl get pod -n istio-system | grep kiali | awk '{print $1}')
```


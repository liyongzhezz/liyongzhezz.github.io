---
title: Prometheus监控指标--kubernetes
date: 2020-07-30 14:48:28
tags:
- Prometheus
categories:
- Prometheus
- 监控指标
description: 使用prometheus监控k8s的常用指标
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1596101882876&di=b13bad8c07c9545ff79b1a92c694afaf&imgtype=0&src=http%3A%2F%2Fhbimg.b0.upaiyun.com%2F4be335086b589e4c398a207233f8d6a594804a8e3cf6e-n19Jwt_fw658
---



# CPU

## CPU Limit合理性

```
sum(increase(container_cpu_cfs_throttled_periods_total{container_name!="", }[5m])) by (container, pod, namespace)
          /
sum(increase(container_cpu_cfs_periods_total{}[5m])) by (container_name, pod, namespace)
          > ( 25 / 100 )
```



用途：查出最近5分钟，超过25%的 CPU 执行周期受到限制的容器。



相关指标：

- `container_cpu_cfs_periods_total`：容器生命周期中度过的 cpu 周期总数

- `container_cpu_cfs_throttled_periods_total`：容器生命周期中度过的受限的 cpu 周期总数



## CPU过度使用

```
sum(namespace:kube_pod_container_resource_requests_cpu_cores:sum{})          /sum(kube_node_status_allocatable_cpu_cores)>(count(kube_node_status_allocatable_cpu_cores)-1) / count(kube_node_status_allocatable_cpu_cores)
```



用途：CPU 已经过度使用无法容忍节点故障，节点资源使用的总量超过节点的 CPU 总量，所以如果有节点故障将影响集群资源运行因为所需资源将无法被分配。



相关指标：

- `kube_pod_container_resource_requests_cpu_cores`：资源 CPU 使用的 cores 数量

- `kube_node_status_allocatable_cpu_cores`：节点 CPU cores 数量



## CPU超分

```
sum(kube_pod_container_resource_limits_cpu_cores{job="kube-state-metrics"})
          /
        sum(kube_node_status_allocatable_cpu_cores)
          > 1.1
```



用途：查看 CPU 资源分配的额度是否超过进群总额度



相关指标：

- `kube_pod_container_resource_limits_cpu_cores`：资源分配的 CPU 资源额度

- `kube_node_status_allocatable_cpu_cores`：节点 CPU 总量



## 命名空间级 CPU 资源使用的比例

```
sum (kube_pod_container_resource_requests_cpu_cores{job="kube-state-metrics"} ) by (namespace)/ (sum(kube_pod_container_resource_limits_cpu_cores{job="kube-state-metrics"}) by (namespace)) > 0.8
```



用途：命名空间级 CPU 资源使用的比例，关乎资源配额。当使用 request 和 limit 限制资源时，使用值和最大值还是有一点区别，当有 request 时说明最低分配了这么多资源。需要注意当 request 等于 limit 时那么说明资源已经是100%已经分配使用当监控告警发出的时候需要区分。



相关指标:

- `kube_pod_container_resource_requests_cpu_cores`：CPU 使用量

- `kube_pod_container_resource_limits_cpu_cores`：CPU 限额最大值





<br>



# 内存

## 内存过度使用

```
sum(namespace:kube_pod_container_resource_requests_memory_bytes:sum{})
          /
        sum(kube_node_status_allocatable_memory_bytes)
          >
        (count(kube_node_status_allocatable_memory_bytes)-1)
          /
        count(kube_node_status_allocatable_memory_bytes)
```



用途：内存已经过度使用无法容忍节点故障，节点资源使用的总量超过节点的内存总量，所以如果有节点故障将影响集群资源运行因为所需资源将无法被分配



相关指标：

- `kube_pod_container_resource_requests_memory_bytes`：资源内存使用的量

- `kube_node_status_allocatable_memory_bytes`：节点内存量



## 内存超分

```
sum(kube_pod_container_resource_limits_memory_bytes{job="kube-state-metrics"})
          /
        sum(kube_node_status_allocatable_memory_bytes{job="kube-state-metrics"})
          > 1.1
```



用途：查看内存资源分配的额度是否超过进群总额度



相关指标:

- `kube_pod_container_resource_limits_memory_bytes`：资源配额内存量

- `kube_node_status_allocatable_memory_bytes`：节点内存量



## 命名空间级内存资源使用的比例

```
sum (kube_pod_container_resource_requests_memory_bytes{job="kube-state-metrics"} ) by (namespace)/ (sum(kube_pod_container_resource_limits_memory_bytes{job="kube-state-metrics"}) by (namespace)) > 0.8
```



用途：命名空间级内存资源使用的比例，关乎资源配额。当使用 request 和 limit 限制资源时，使用值和最大值还是有一点区别，当有 request 时说明最低分配了这么多资源。需要注意当 request 等于 limit 时那么说明资源已经是100%已经分配使用当监控告警发出的时候需要区分。



相关指标:

- `kube_pod_container_resource_requests_memory_bytes`：内存资源使用量

- `kube_pod_container_resource_limits_memory_bytes`：内存资源最大值



<br>



# 存储

## PVC容量监控

```
kubelet_volume_stats_available_bytes{job="kubelet", metrics_path="/metrics"}
          /
        kubelet_volume_stats_capacity_bytes{job="kubelet", metrics_path="/metrics"}
          < 0.3
```



用途：监控pvc剩余容量是否小于30%



相关指标：

- `kubelet_volume_stats_available_bytes`：剩余空间

- `kubelet_volume_stats_capacity_bytes`：空间总量



## PVC磁盘空间耗尽预测

```
(kubelet_volume_stats_available_bytes{job="kubelet", metrics_path="/metrics"}
            /
          kubelet_volume_stats_capacity_bytes{job="kubelet", metrics_path="/metrics"}
        ) < 0.4
        and
        predict_linear(kubelet_volume_stats_available_bytes{job="kubelet", metrics_path="/metrics"}[6h], 4 * 24 * 3600) < 0
```



用途：通过PVC资源使用6小时变化率预测 接下来4天的磁盘使用率



相关指标:

- `kubelet_volume_stats_available_bytes`：剩余空间

- `kubelet_volume_stats_capacity_bytes`：空间总量



## PV使用状态

```
kube_persistentvolume_status_phase{phase=~"Failed|Pending",job="kube-state-metrics"}
```



用途：PV 使用状态监控



相关指标：

- `kube_persistentvolume_status_phase`：PV 使用状态



<br>



# APIServer

## APIServer请求错误率

```
sum(rate(apiserver_request_total{job="apiserver",code=~"5.."}[5m])) by (resource,subresource,verb)
          /
        sum(rate(apiserver_request_total{job="apiserver"}[5m])) by (resource,subresource,verb) > 0.05
```



用途：5分钟内 APIServer 请求错误率。



相关指标：

- `apiserver_request_total：APIServer` 请求数



## APIServer状态

```
absent(up{job="apiserver"} == 1)
```



用途：监控 APIServer 服务状态，失联原因可能是服务 down 或者网络出现状况。



<br>



# 证书

## kubelet客户端证书过期

```
apiserver_client_certificate_expiration_seconds_count{job="apiserver"} > 0 and on(job) histogram_quantile(0.01, sum by (job, le) (rate(apiserver_client_certificate_expiration_seconds_bucket{job="apiserver"}[5m]))) < 2592000

apiserver_client_certificate_expiration_seconds_count{job="apiserver"} > 0 and on(job) histogram_quantile(0.01, sum by (job, le) (rate(apiserver_client_certificate_expiration_seconds_bucket{job="apiserver"}[5m]))) < 604800
```



用途：监测证书状态30天告警和7天告警。



相关指标：

- `apiserver_client_certificate_expiration_seconds_count`：证书有效剩余时间



<br>



# 节点

## 节点状态

```
kube_node_status_condition{job="kube-state-metrics",condition="Ready",status="true"} == 0
```



用途：检测节点是否为就绪状态，或者可能是 kubelet 服务down 了。



相关指标：

- `kube_node_status_condition`：节点状态监测



## 节点状态为 Unreachable

```
kube_node_spec_unschedulable{job="kube-state-metrics"} == 1
```



用途：监测状态为 Unreachable的节点。



## 节点运行pod过多

```
max(max(kubelet_running_pod_count{job="kubelet", metrics_path="/metrics"}) by(instance) * on(instance) group_left(node) kubelet_node_name{job="kubelet", metrics_path="/metrics"}) by(node) / max(kube_node_status_capacity_pods{job="kube-state-metrics"} != 1) by(node) > 0.95
```



用途：监测节点上运行的 Pods 数量是否达到最大限额的95%。



相关指标：

- `kubelet_running_pod_count`：节点运行的 Pods 数量

- `kubelet_node_name`：节点名称

- `kube_node_status_capacity_pods`：节点可运行的最大 Pod 数量



<br>



# pod

## pod重启时间

```
rate(kube_pod_container_status_restarts_total{job="kube-state-metrics"}[5m]) * 60 * 3 > 0
```



用途：Pod 重启时间，重启时间超过3m告警。



相关指标:

- kube_pod_container_status_restarts_total：重启状态0为正常



## pod状态

```
sum by (namespace, pod) (max by(namespace, pod) (kube_pod_status_phase{job="kube-state-metrics", phase=~"Pending|Unknown"}) * on(namespace, pod) group_left(owner_kind) max by(namespace, pod, owner_kind) (kube_pod_owner{owner_kind!="Job"})) >0
```



用途：检测 Pod 是否就绪。



相关指标：

- kube_pod_status_phase：Pod 状态



<br>



# Deployment

## Deployment部署失败

```
kube_deployment_status_observed_generation{job="kube-state-metrics"}
          !=
        kube_deployment_metadata_generation{job="kube-state-metrics"}
```



用途：Deployment 生成的资源与定义的资源不匹配。



相关指标：

- kube_deployment_status_observed_generation：Deployment 生成资源数

- kube_deployment_metadata_generation：Deployment 定义资源数



## deployment副本数

```
(
          kube_deployment_spec_replicas{job="kube-state-metrics"}
            !=
          kube_deployment_status_replicas_available{job="kube-state-metrics"}
        ) and (
          changes(kube_deployment_status_replicas_updated{job="kube-state-metrics"}[3m])
            ==
          0
        )
```



用途：查看 Deplyment 副本是否达到预期。



相关指标：

- kube_deployment_spec_replicas           资源定义副本数

- kube_deployment_status_replicas_available     正在运行副本数

- kube_deployment_status_replicas_updated      更新的副本数



<br>



# statefuleset

## statefulset副本数

```
(
          kube_statefulset_status_replicas_ready{job="kube-state-metrics"}
            !=
          kube_statefulset_status_replicas{job="kube-state-metrics"}
        ) and (
          changes(kube_statefulset_status_replicas_updated{job="kube-state-metrics"}[5m])
            ==
          0
        )
```



用途：监测 StatefulSet 副本是否达到预期。



相关指标：

- kube_statefulset_status_replicas_ready：就绪副本数

- kube_statefulset_status_replicas：当前副本数

- kube_statefulset_status_replicas_updated：更新的副本数



<br>



# DaemonSet

## daemonset就绪

```
kube_daemonset_status_number_ready{job="kube-state-metrics"}
          /
        kube_daemonset_status_desired_number_scheduled{job="kube-state-metrics"} < 1.00
```



用途：监测 DaemonSet 是否处于就绪状态。



相关指标：

- kube_daemonset_status_number_ready：就绪的 DaemonSet

- kube_daemonset_status_desired_number_scheduled：应该调度的 DaemonSet 数量
---
title: Prometheus监控指标--ETCD
date: 2020-07-30 15:05:54
tags:
- Prometheus
categories:
- Prometheus
- 监控指标
description: 使用prometheus监控etcd常用指标
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1596102895688&di=18fdbb30dbb810121221eca19e242917&imgtype=0&src=http%3A%2F%2Fmydlq-club.oss-cn-beijing.aliyuncs.com%2Fimages%2Fprometheus-operator-etcd-1001.jpg%3Fx-oss-process%3Dstyle%2Fshuiyin
---



# 集群

## etcd存活监测

```
up{job="etcd"} < 1
```



用途：etcd 存活检测。



## 集群健康检查

```
count(up{job="etcd"} == 0) > ceil(count(up{job="etcd"}) / 2 - 1)
```



用途：etcd 集群健康检查，判断down 数量是否大于集群可允许故障数量。



## leader检查

```
max(etcd_server_has_leader) != 1
```



用途：检查是否有 leader。



<br>



# I/O监测

## io监测--后端提交延时

```
histogram_quantile(0.99, sum(rate(etcd_disk_backend_commit_duration_seconds_bucket[5m])) by (instance, le)) > 100
```



用途：etcd io 监测，后端提交延时。





## io监测--同步磁盘延时

```
histogram_quantile(0.99, sum(rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m])) by (instance, le)) > 100
```



用途：etcd io 监测，文件同步到磁盘延时。



<br>



# 网络

## grpc调用速率

```
sum(rate(grpc_server_handled_total{grpc_type="unary"}[1m])) > 100
```



用途：Grpc 调用速率



<br>



# 数据库

## 数据库大小

```
etcd_debugging_mvcc_db_total_size_in_bytes/1024/1024 > 1024
```



用途：检测数据库大小。




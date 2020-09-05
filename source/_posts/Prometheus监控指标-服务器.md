---
title: Prometheus监控指标--服务器
date: 2020-07-30 15:01:54
tags:
- Prometheus
categories:
- Prometheus
- 监控指标
description: 使用prometheus监控服务器常用指标
cover: https://ss2.bdstatic.com/70cFvnSh_Q1YnxGkpoWK1HF6hhy/it/u=3974948727,3695780061&fm=26&gp=0.jpg
---



# 时间

## 本地时间偏移

```
(node_timex_offset_seconds > 0.05
        and
          deriv(node_timex_offset_seconds[5m]) >= 0
        )
        or
        (
          node_timex_offset_seconds < -0.05
        and
          deriv(node_timex_offset_seconds[5m]) <= 0)
```



用途：本地时间偏移量。



相关指标：

- node_timex_offset_seconds：误差



<br>



# 网络

## 网卡接收的错误

```
increase(node_network_receive_errs_total[2m]) > 10
```



用途：网卡接收错误量。



相关指标：

- node_network_receive_errs_total：接收错误总量



## 网卡发送的错误

```
increase(node_network_transmit_errs_total[2m]) > 10
```



用途：网卡传输错误量。



相关指标：

- node_network_transmit_errs_total：传输错误总量



## 网卡接收字节数增量

```bash
rate(node_network_receive_bytes_total[1m])
```



## 网络传输IO

```bash
# 单位是M 
rate(node_network_transmit_bytes_total[1m]) / 1024 /1024
```





<br>



# 磁盘

## inode空闲率

```
(          node_filesystem_files_free{job="node-exporter",fstype!=""} / node_filesystem_files{job="node-exporter",fstype!=""} * 100 < 5        and          node_filesystem_readonly{job="node-exporter",fstype!=""} == 0        )
```



用途：监测节点inode空闲是否小于5%。



相关指标：

- node_filesystem_files_free：空闲的 inode

- node_filesystem_files：inodes 总量
- node_filesystem_readonly：文件系统是否只读，0表示正常



## inode耗尽预测

```
(node_filesystem_files_free{job="node-exporter",fstype!=""} / node_filesystem_files{job="node-exporter",fstype!=""} * 100 < 20        and          predict_linear(node_filesystem_files_free{job="node-exporter",fstype!=""}[6h], 4*60*60) < 0        and          node_filesystem_readonly{job="node-exporter",fstype!=""} == 0)
```



用途：inode 耗尽预测，以6小时曲线变化预测接下来24小时和4小时可能使用的 inodes。



相关指标：

- node_filesystem_files_free：空闲的 inode

- node_filesystem_files：inodes 总量



## 分区容量使用率

```
(node_filesystem_avail_bytes{job="node-exporter",fstype!=""} / node_filesystem_size_bytes{job="node-exporter",fstype!=""} * 100 < 10        and          node_filesystem_readonly{job="node-exporter",fstype!=""} == 0        )
```



用途：监测分区容量使用率是否超过90%。



相关指标：

- node_filesystem_avail_bytes：空闲容量

- node_filesystem_size_bytes：总容量



## 分区容量耗尽预测

```
(node_filesystem_avail_bytes{job="node-exporter",fstype!=""} / node_filesystem_size_bytes{job="node-exporter",fstype!=""} * 100 < 15        and          predict_linear(node_filesystem_avail_bytes{job="node-exporter",fstype!=""}[6h], 4*60*60) < 0        and          node_filesystem_readonly{job="node-exporter",fstype!=""} == 0)
```



用途：分区容量耗尽预测，以6小时曲线变化预测接下来24小时和4小时可能使用的容量。



相关指标：

- node_filesystem_avail_bytes：空闲容量

- node_filesystem_size_bytes：总容量



## 磁盘剩余百分比

```bash
(node_filesystem_free_bytes{device=~"/dev/sd[a-z]1"}  /  node_filesystem_size_bytes) * 100
```



## 磁盘IO

```bash
(rate(node_disk_read_bytes_total{device=~"sd[a-z]"}[1m]) + rate(node_disk_written_bytes_total[1m])) /1024 /1024
```



## 文件描述符使用率

```bash
(node_filefd_allocated / node_filefd_maximum) * 100
```



<br>



# CPU

## CPU使用率

```bash
(1- (sum(increase(node_cpu_seconds_total{mode="idle"}[1m])) by (instance) / (sum(increase(node_cpu_seconds_total[1m])) by (instance)))) *  100
```





## 用户态使用率

```bash
sum(increase(node_cpu_seconds_total{mode="user"}[1m])) by (instance) / (sum(increase(node_cpu_seconds_total[1m])) by (instance))
```





## 内核态使用率

```bash
sum(increase(node_cpu_seconds_total{mode="system"}[1m])) by (instance) / (sum(increase(node_cpu_seconds_total[1m])) by (instance))
```



## IO等待使用率

```bash
sum(increase(node_cpu_seconds_total{mode="iowait"}[1m])) by (instance) / (sum(increase(node_cpu_seconds_total[1m])) by (instance))
```



<br>



# 内存

## 内存使用率

```bash
(1- ((node_memory_Buffers_bytes + node_memory_Cached_bytes + node_memory_MemFree_bytes) / node_memory_MemTotal_bytes)) * 100
```




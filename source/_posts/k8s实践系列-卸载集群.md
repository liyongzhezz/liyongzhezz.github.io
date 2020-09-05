---
title: '[k8s实践系列]卸载集群'
date: 2020-07-05 10:11:40
tags:
- k8s
categories: 实践K8s
description: 使用kubeadm卸载集群
cover: http://www.soft6.com/uploadfile/2017/1009/20171009035234769.jpg
---



## 先隔离再删除节点

```bash
$ kubectl drain SCA-LUM700007	SCA-LUM700008 SCA-LUM700012 SCA-LUM700013 SCA-LUM700014 --delete-local-data --force --ignore-daemonsets

$ kubectl delete node SCA-LUM700007	SCA-LUM700008 SCA-LUM700012 SCA-LUM700013 SCA-LUM700014
```

<br>



## 重置节点状态

```bash
$ ansible ins -m shell -a 'kubeadm reset --force'

$ ansible ins -m shell -a 'rm -rf /etc/cni /opt/cni/ /var/lib/calico/ ~/.kube/'
```


---
title: '[k8s实践系列]证书管理'
date: 2020-07-05 10:26:34
tags:
- k8s
- k8s证书管理
categories: 实践K8s
description: 使用kubeadm管理k8s证书状态
cover: https://ss1.bdstatic.com/70cFvXSh_Q1YnxGkpoWK1HF6hhy/it/u=175965930,2539474621&fm=26&gp=0.jpg
---



## 查看证书状态

```bash
$ kubeadm alpha certs check-expiration
[check-expiration] Reading configuration from the cluster...
[check-expiration] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'

CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
admin.conf                 Apr 22, 2021 09:03 UTC   364d                                    no
apiserver                  Apr 22, 2021 09:03 UTC   364d            ca                      no
apiserver-etcd-client      Apr 22, 2021 09:03 UTC   364d            etcd-ca                 no
apiserver-kubelet-client   Apr 22, 2021 09:03 UTC   364d            ca                      no
controller-manager.conf    Apr 22, 2021 09:03 UTC   364d                                    no
etcd-healthcheck-client    Apr 22, 2021 09:03 UTC   364d            etcd-ca                 no
etcd-peer                  Apr 22, 2021 09:03 UTC   364d            etcd-ca                 no
etcd-server                Apr 22, 2021 09:03 UTC   364d            etcd-ca                 no
front-proxy-client         Apr 22, 2021 09:03 UTC   364d            front-proxy-ca          no
scheduler.conf             Apr 22, 2021 09:03 UTC   364d                                    no

CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
ca                      Apr 14, 2030 11:18 UTC   9y              no
etcd-ca                 Apr 14, 2030 11:18 UTC   9y              no
front-proxy-ca          Apr 14, 2030 11:18 UTC   9y              no
```

<br>

## 更新所有证书

```bash
$ kubeadm alpha certs renew all
[renew] Reading configuration from the cluster...
[renew] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'

certificate embedded in the kubeconfig file for the admin to use and for kubeadm itself renewed
certificate for serving the Kubernetes API renewed
certificate the apiserver uses to access etcd renewed
certificate for the API server to connect to kubelet renewed
certificate embedded in the kubeconfig file for the controller manager to use renewed
certificate for liveness probes to healthcheck etcd renewed
certificate for etcd nodes to communicate with each other renewed
certificate for serving etcd renewed
certificate for the front proxy client renewed
certificate embedded in the kubeconfig file for the scheduler manager to use renewed
```

<br>



## 更改证书签名



### 创建新的证书配置文件

```bash
# 创建更新配置
$ cat > ca-sign.yaml <<EOF
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
apiServer:
  certSANs:
  - "10.10.34.89"
  - "113.108.71.77"
  #- "kubernetes"
  #- "kubernetes.default"
  #- "kubernetes.default.svc"
  #- "kubernetes.default.svc.cluster"
  #- "kubernetes.default.svc.cluster.local"
controllerManager:
  extraArgs:
    cluster-signing-cert-file: /etc/kubernetes/pki/ca.crt
    cluster-signing-key-file: /etc/kubernetes/pki/ca.key
#etcd:
#  local:
#    dataDir: /var/lib/etcd
#    serverCertSANs:
#    - "localhost"
#    - "127.0.0.1"
#    peerCertSANs:
#    - "localhost"
#    - "127.0.0.1"
EOF
```



### 更新并生效配置

```bash
# 更新 kubernetes 配置
$ kubeadm config upload from-file --config=ca-sign.yaml

# 确认更新配置生效
$ kubeadm config view
apiServer:
  certSANs:
  - 10.10.34.89
  - 113.108.71.77
...
```



### 重新生成apiserver证书

```bash
# 删除原 apiserver 证书
$ rm -rf /etc/kubernetes/pki/apiserver.*

# 重新生成 apiserver 证书
$ kubeadm init phase certs apiserver --config=ca-sign.yaml
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [ukm01 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.10.34.92 113.108.71.77]

# 确认 apiserver 证书更新情况
$ openssl x509 -text -noout -in /etc/kubernetes/pki/apiserver.crt
...
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage:
                TLS Web Server Authentication
            X509v3 Subject Alternative Name:
                DNS:ukm01, DNS:kubernetes, DNS:kubernetes.default, DNS:kubernetes.default.svc, DNS:kubernetes.default.svc.cluster.local, IP Address:10.96.0.1, IP Address:10.10.34.92, IP Address:10.10.34.89, IP Address:113.108.71.77
...
```



### 重新生成所有证书

```bash
# 重新生成所有证书
$ ansible master -m shell -a 'cd /etc/kubernetes/pki/ && rm -rf apiserver* front* sa*'
$ ansible master -m shell -a 'cd /etc/kubernetes/pki/etcd/ && rm -rf healthcheck-client* peer* server*'
$ ansible master -m copy -a 'src=ca-sign.yaml dest=ca-sign.yaml'
$ ansible master -m shell -a 'kubeadm init phase certs all --config=ca-sign.yaml'

# 验证证书
$ ansible master -m shell -a 'openssl x509 -text -noout -in /etc/kubernetes/pki/apiserver.crt | grep DNS'
$ ansible master -m shell -a 'openssl x509 -text -noout -in /etc/kubernetes/pki/etcd/server.crt | grep DNS'
$ ansible master -m shell -a 'openssl x509 -text -noout -in /etc/kubernetes/pki/etcd/peer.crt | grep DNS'
```



### 更新证书

```bash
# 由于 docker container 有缓存，证书并未加载到 pod 中，因此需要删掉 docker 中的 container 并重启 pod 才可以使证书生效。
$ docker ps | awk '/k8s_etcd/{print "docker rm -f "$1}' | bash
$ kubectl delete pod etcd-01 -n kube-system

# 更新所有证书
$ kubeadm alpha certs renew all --config=ca-sign.yaml
```

<img src="update.png" style="zoom:50%;" />



### 通过api更新

可以通过apiserver的api更新证书，这一步需要进行验证

```bash
$ kubeadm alpha certs renew all --use-api --config=ca-sign.yaml &
...
[certs] Certificate request "kubeadm-cert-kubernetes-admin-8pvf8" created
...

# 批准更新
$ kubectl get csr | awk '!/Approved/ && !/NAME/{print "kubectl certificate approve "$1}' | bash
...
certificatesigningrequest.certificates.k8s.io/kubeadm-cert-kubernetes-admin-8pvf8 approved
...
```


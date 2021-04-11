---
title: 二进制方式部署kubernetes 1.20
date: 2021-04-11 11:11:26
tags:
- Kubernetes
categories:
- Kubernetes
- 集群部署
description: 使用二进制方式安装kubernetes 1.20版本集群
cover: https://gimg2.baidu.com/image_search/src=http%3A%2F%2F5b0988e595225.cdn.sohucs.com%2Fimages%2F20180613%2Fa12e8e321ae040e08c0a1edac71aeb2f.jpeg&refer=http%3A%2F%2F5b0988e595225.cdn.sohucs.com&app=2002&size=f9999,10000&q=a80&n=0&g=0n&fmt=jpeg?sec=1620702740&t=d5454a9fbcd5da397ea7ec9d6531d1ab
---



{% note info 'fas fa-bullhorn' %}

本文主要介绍了使用二进制方式安装kubernetes 1.20版本集群

更新于 2021-04-11

{% endnote %}

<br>



# 集群规划

{% tabs comments %}

<!-- tab 部署流程 -->

先部署一个单master节点的k8s集群，然后再扩展集群为多master节点实现高可用；

<!-- endtab -->

<!-- tab 服务器规划 -->

| 角色       | IP            | 组件                                                         |
| ---------- | ------------- | ------------------------------------------------------------ |
| k8s-master | 192.168.31.71 | kube-apiserver，kube-controller-manager，kube-scheduler，etcd |
| k8s-node1  | 192.168.31.72 | kubelet，kube-proxy，docker，etcd                            |
| k8s-node2  | 192.168.31.73 | kubelet，kube-proxy，docker，etcd                            |

<!-- endtab -->

{% endtabs %}



<br>



# 初始化配置

**下面的初始化操作在所有服务器上进行**

{% tabs comments %}

<!-- tab 关闭防火墙 -->

```bash
systemctl stop firewalld 
systemctl disable firewalld 
```



> 生产环境其实建议按需按端口开放

<!-- endtab -->

<!-- tab 关闭selinux和swap -->

```bash
# 关闭selinux 
sed -i 's/enforcing/disabled/' /etc/selinux/config  
setenforce 0  
 
# 关闭swap 
swapoff -a 
sed -ri 's/.*swap.*/#&/' /etc/fstab   

```

<!-- endtab -->

<!-- tab 添加host -->

```bash
cat >> /etc/hosts << EOF 
192.168.31.71 k8s-master1 
192.168.31.72 k8s-node1 
192.168.31.73 k8s-node2 
EOF 
```

<!-- endtab -->

<!-- tab 内核参数设置 -->

```bash
cat > /etc/sysctl.d/k8s.conf << EOF 
net.bridge.bridge-nf-call-ip6tables = 1 
net.bridge.bridge-nf-call-iptables = 1 
EOF 

# 生效配置
sysctl --system  
```

<!-- endtab -->

<!-- tab 时间同步 -->

```bash
yum install -y  ntpdate 
ntpdate time.windows.com
```

<!-- endtab -->

{% endtabs %}



<br>



# 部署etcd

{% note info 'fas fa-bullhorn' %}

根据规划，我们将在三个节点上部署服务，形成一个etcd集群

{% endnote %}



## 创建证书

{% tabs comments %}

<!-- tab 安装cfssl -->

安装cfssl用于生成证书文件，这里我在master节点上进行安装

```bash
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl_linux-amd64 cfssljson_linux-amd64 cfssl-certinfo_linux-amd64
mv cfssl_linux-amd64 /usr/local/bin/cfssl
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
mv cfssl-certinfo_linux-amd64 /usr/bin/cfssl-certinfo 
```

<!-- endtab -->

<!-- tab 创建CA -->

```bash
# 创建证书目录
mkdir -p ~/TLS/{etcd,k8s}
cd ~/TLS/etcd

# 自签CA
cat > ca-config.json << EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "www": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
EOF

cat > ca-csr.json << EOF
{
    "CN": "etcd CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing"
        }
    ]
}
EOF

# 生成证书
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
```



{% note info 'fas fa-bullhorn' %}

执行完成后会生成`ca.pem`和`ca-key.pem`文件

{% endnote %}

<!-- endtab -->

<!-- tab 签发etcd证书 -->

```bash
# 创建申请文件
cat > server-csr.json << EOF
{
    "CN": "etcd",
    "hosts": [
    "192.168.31.71",
    "192.168.31.72",
    "192.168.31.73"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing"
        }
    ]
}
EOF

# 生成证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www server-csr.json | cfssljson -bare server
```

{% note info 'fas fa-bullhorn' %}

执行完成后会生成`server.pem`和`server-key.pem`文件

{% endnote %}

<!-- endtab -->

{% endtabs %}



## 部署etcd集群



{% tabs comments %}

<!-- tab 安装 -->

```bash
wget https://github.com/etcd-io/etcd/releases/download/v3.4.9/etcd-v3.4.9-linux-amd64.tar.gz

# 创建工作目录
mkdir /opt/etcd/{bin,cfg,ssl} -p
tar zxvf etcd-v3.4.9-linux-amd64.tar.gz
mv etcd-v3.4.9-linux-amd64/{etcd,etcdctl} /opt/etcd/bin/
```

<!-- endtab -->



<!-- tab 创建配置文件 -->

```bash
cat > /opt/etcd/cfg/etcd.conf << EOF
#[Member]
ETCD_NAME="etcd-1"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.31.71:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.31.71:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.31.71:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.31.71:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.31.71:2380,etcd-2=https://192.168.31.72:2380,etcd-3=https://192.168.31.73:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF
```

-  ETCD_NAME：节点名称，集群中唯一
- ETCD_DATA_DIR：数据目录
- ETCD_LISTEN_PEER_URLS：集群通信监听地址
- ETCD_LISTEN_CLIENT_URLS：客户端访问监听地址
- ETCD_INITIAL_ADVERTISE_PEERURLS：集群通告地址
- ETCD_ADVERTISE_CLIENT_URLS：客户端通告地址
- ETCD_INITIAL_CLUSTER：集群节点地址
- ETCD_INITIALCLUSTER_TOKEN：集群Token
- ETCD_INITIALCLUSTER_STATE：加入集群的当前状态，new是新集群，existing表示加入已有集群

<!-- endtab -->

<!-- tab 创建服务启动文件 -->

```bash
cat > /usr/lib/systemd/system/etcd.service << EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=/opt/etcd/cfg/etcd.conf
ExecStart=/opt/etcd/bin/etcd \
--cert-file=/opt/etcd/ssl/server.pem \
--key-file=/opt/etcd/ssl/server-key.pem \
--peer-cert-file=/opt/etcd/ssl/server.pem \
--peer-key-file=/opt/etcd/ssl/server-key.pem \
--trusted-ca-file=/opt/etcd/ssl/ca.pem \
--peer-trusted-ca-file=/opt/etcd/ssl/ca.pem \
--logger=zap
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

<!-- endtab -->

<!-- tab 拷贝证书 -->

```bash
cp ~/TLS/etcd/ca*pem ~/TLS/etcd/server*pem /opt/etcd/ssl/
```

<!-- endtab -->

{% endtabs %}



## 启动集群

首先启动第一个节点：

```bash
systemctl daemon-reload
systemctl start etcd
systemctl enable etcd
```



将上一步中的文件都拷贝到其他节点上：

```bash
scp -r /opt/etcd/ root@192.168.31.72:/opt/
scp /usr/lib/systemd/system/etcd.service root@192.168.31.72:/usr/lib/systemd/system/
scp -r /opt/etcd/ root@192.168.31.73:/opt/
scp /usr/lib/systemd/system/etcd.service root@192.168.31.73:/usr/lib/systemd/system/
```



注意，拷贝过去后需要修改一下配置文件的内容，将IP和节点名称修改为当前所在服务器的地址：

```bash
cat /opt/etcd/cfg/etcd.conf
#[Member]
ETCD_NAME="etcd-1"   # 修改此处，节点2改为etcd-2，节点3改为etcd-3
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.31.71:2380"   # 修改此处为当前服务器IP
ETCD_LISTEN_CLIENT_URLS="https://192.168.31.71:2379" # 修改此处为当前服务器IP

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.31.71:2380" # 修改此处为当前服务器IP
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.31.71:2379" # 修改此处为当前服务器IP
ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.31.71:2380,etcd-2=https://192.168.31.72:2380,etcd-3=https://192.168.31.73:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
```



然后在启动剩下的两个节点，步骤同上。



## 查看集群状态

```bash
ETCDCTL_API=3 /opt/etcd/bin/etcdctl --cacert=/opt/etcd/ssl/ca.pem --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --endpoints="https://192.168.31.71:2379,https://192.168.31.72:2379,https://192.168.31.73:2379" endpoint health --write-out=table

+----------------------------+--------+-------------+-------+
|          ENDPOINT    | HEALTH |    TOOK     | ERROR |
+----------------------------+--------+-------------+-------+
| https://192.168.31.71:2379 |   true | 10.301506ms |    |
| https://192.168.31.73:2379 |   true | 12.87467ms |     |
| https://192.168.31.72:2379 |   true | 13.225954ms |    |
+----------------------------+--------+-------------+-------+

```



{% note success 'fas fa-bullhorn' %}

可以看到集群是正常的，部署成功

{% endnote %}



<br>



# 安装docker

{% note info 'fas fa-bullhorn' %}

在所有的节点都安装docker，也可以换成其他的容器引擎如containerd

{% endnote %}



## 下载安装 

```bash
wget https://download.docker.com/linux/static/stable/x86_64/docker-19.03.9.tgz
tar zxvf docker-19.03.9.tgz
mv docker/* /usr/bin
```



##  创建服务启动文件 

```bash
cat > /usr/lib/systemd/system/docker.service << EOF
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target
EOF
```



## 创建配置文件 

```bash
mkdir /etc/docker
cat > /etc/docker/daemon.json << EOF
{
  "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"]
}
EOF
```

{% note info 'fas fa-bullhorn' %}

这里使用了阿里云镜像加速器

{% endnote %}





## 启动服务 

```bash
systemctl daemon-reload
systemctl start docker
systemctl enable docker
```





<br>



# 部署master节点

## 生成kube-apiserver证书

{% tabs comments %}

<!-- tab 自签CA -->

```bash
cd ~/TLS/k8s

cat > ca-config.json << EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
EOF
cat > ca-csr.json << EOF
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```

<!-- endtab -->

<!-- tab 生成证书 -->

```bash
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
```

{% note info 'fas fa-bullhorn' %}

会生成`ca.pem`和`ca-key.pem`文件。

{% endnote %}

<!-- endtab -->

<!-- tab 签发kube-apiserver证书 -->

```bash
cat > server-csr.json << EOF
{
    "CN": "kubernetes",
    "hosts": [
      "10.0.0.1",
      "127.0.0.1",
      "192.168.31.71",
      "192.168.31.72",
      "192.168.31.73",
      "192.168.31.88",
      "kubernetes",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster",
      "kubernetes.default.svc.cluster.local"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```

{% note info 'fas fa-bullhorn' %}

文件hosts字段中IP为所有Master/LB/VIP IP，一个都不能少！为了方便后期扩容可以多写几个预留的IP

{% endnote %}

<!-- endtab -->

<!-- tab 生成证书 -->

```bash
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes server-csr.json | cfssljson -bare server
```

<!-- endtab -->

{% endtabs %}



## 安装kube-apiserver

**https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.20.md，在这里下载1.20版本的安装包，下载server包就够了，包含了Master和Worker Node二进制文件。**

{% tabs comments %}

<!-- tab 安装 -->

```bash
mkdir -p /opt/kubernetes/{bin,cfg,ssl,logs} 
tar zxvf kubernetes-server-linux-amd64.tar.gz
cd kubernetes/server/bin
cp kube-apiserver kube-scheduler kube-controller-manager /opt/kubernetes/bin
cp kubectl /usr/bin/

# 拷贝证书
cp ~/TLS/k8s/ca*pem ~/TLS/k8s/server*pem /opt/kubernetes/ssl/
```

<!-- endtab -->

<!-- tab 创建配置文件 -->

```bash
cat > /opt/kubernetes/cfg/kube-apiserver.conf << EOF
KUBE_APISERVER_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--etcd-servers=https://192.168.31.71:2379,https://192.168.31.72:2379,https://192.168.31.73:2379 \\
--bind-address=192.168.31.71 \\
--secure-port=6443 \\
--advertise-address=192.168.31.71 \\
--allow-privileged=true \\
--service-cluster-ip-range=10.0.0.0/24 \\
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction \\
--authorization-mode=RBAC,Node \\
--enable-bootstrap-token-auth=true \\
--token-auth-file=/opt/kubernetes/cfg/token.csv \\
--service-node-port-range=30000-32767 \\
--kubelet-client-certificate=/opt/kubernetes/ssl/server.pem \\
--kubelet-client-key=/opt/kubernetes/ssl/server-key.pem \\
--tls-cert-file=/opt/kubernetes/ssl/server.pem  \\
--tls-private-key-file=/opt/kubernetes/ssl/server-key.pem \\
--client-ca-file=/opt/kubernetes/ssl/ca.pem \\
--service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--service-account-issuer=api \\
--service-account-signing-key-file=/opt/kubernetes/ssl/server-key.pem \\
--etcd-cafile=/opt/etcd/ssl/ca.pem \\
--etcd-certfile=/opt/etcd/ssl/server.pem \\
--etcd-keyfile=/opt/etcd/ssl/server-key.pem \\
--requestheader-client-ca-file=/opt/kubernetes/ssl/ca.pem \\
--proxy-client-cert-file=/opt/kubernetes/ssl/server.pem \\
--proxy-client-key-file=/opt/kubernetes/ssl/server-key.pem \\
--requestheader-allowed-names=kubernetes \\
--requestheader-extra-headers-prefix=X-Remote-Extra- \\
--requestheader-group-headers=X-Remote-Group \\
--requestheader-username-headers=X-Remote-User \\
--enable-aggregator-routing=true \\
--audit-log-maxage=30 \\
--audit-log-maxbackup=3 \\
--audit-log-maxsize=100 \\
--audit-log-path=/opt/kubernetes/logs/k8s-audit.log"
EOF
```

- --logtostderr：启用日志
- ---v：日志等级
- --log-dir：日志目录
- --etcd-servers：etcd集群地址
- --bind-address：监听地址
- --secure-port：https安全端口
- --advertise-address：集群通告地址
- --allow-privileged：启用授权
- --service-cluster-ip-range：Service虚拟IP地址段
- --enable-admission-plugins：准入控制模块
- --authorization-mode：认证授权，启用RBAC授权和节点自管理
- --enable-bootstrap-token-auth：启用TLS bootstrap机制
- --token-auth-file：bootstrap token文件
- --service-node-port-range：Service nodeport类型默认分配端口范围
- --kubelet-client-xxx：apiserver访问kubelet客户端证书
- --tls-xxx-file：apiserver https证书
- --etcd-xxxfile：连接Etcd集群证书
- --audit-log-xxx：审计日志

1.20版本必须加的参数：

- --service-account-issuer
- --service-account-signing-key-file

启动聚合层相关配置：

- --requestheader-client-ca-file
- --proxy-client-cert-file
- --proxy-client-key-file
- --requestheader-allowed-names
- --requestheader-extra-headers-prefix
- --requestheader-group-headers
- --requestheader-username-headers
- --enable-aggregator-routing

<!-- endtab -->

<!-- tab 创建TLS Token -->

```bash
# 生成一个token
head -c 16 /dev/urandom | od -An -t x | tr -d ' '

# 创建文件
cat > /opt/kubernetes/cfg/token.csv << EOF
c47ffb939f5ca36231d9e3121a252940,kubelet-bootstrap,10001,"system:node-bootstrapper"
EOF
```

<!-- endtab -->

<!-- tab 创建启动文件 -->

```bash
cat > /usr/lib/systemd/system/kube-apiserver.service << EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-apiserver.conf
ExecStart=/opt/kubernetes/bin/kube-apiserver \$KUBE_APISERVER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

<!-- endtab -->

{% endtabs %}



## 启动kube-apiserver

```bash
systemctl daemon-reload
systemctl start kube-apiserver 
systemctl enable kube-apiserver
```





## 部署kube-controller-manager

{% tabs comments %}

<!-- tab 创建配置文件 -->

```bash
cat > /opt/kubernetes/cfg/kube-controller-manager.conf << EOF
KUBE_CONTROLLER_MANAGER_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--leader-elect=true \\
--kubeconfig=/opt/kubernetes/cfg/kube-controller-manager.kubeconfig \\
--bind-address=127.0.0.1 \\
--allocate-node-cidrs=true \\
--cluster-cidr=10.244.0.0/16 \\
--service-cluster-ip-range=10.0.0.0/24 \\
--cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem \\
--cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem  \\
--root-ca-file=/opt/kubernetes/ssl/ca.pem \\
--service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--cluster-signing-duration=87600h0m0s"
EOF
```

- --kubeconfig：连接apiserver配置文件
- --leader-elect：当该组件启动多个时，自动选举（HA）
- --cluster-signing-cert-file/--cluster-signing-key-file：自动为kubelet颁发证书的CA，与apiserver保持一致

<!-- endtab -->

<!-- tab 创建证书 -->

```bash
cd ~/TLS/k8s

# 创建证书请求文件
cat > kube-controller-manager-csr.json << EOF
{
  "CN": "system:kube-controller-manager",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing", 
      "ST": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF

# 生成证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
```

<!-- endtab -->

<!-- tab 创建kubeconfig文件 -->

```bash
KUBE_CONFIG="/opt/kubernetes/cfg/kube-controller-manager.kubeconfig"
KUBE_APISERVER="https://192.168.31.71:6443"

kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}

kubectl config set-credentials kube-controller-manager \
  --client-certificate=./kube-controller-manager.pem \
  --client-key=./kube-controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}

kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-controller-manager \
  --kubeconfig=${KUBE_CONFIG}

kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
```

<!-- endtab -->

<!-- tab 创建启动文件 -->

```bash
cat > /usr/lib/systemd/system/kube-controller-manager.service << EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-controller-manager.conf
ExecStart=/opt/kubernetes/bin/kube-controller-manager \$KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

<!-- endtab -->

{% endtabs %}



## 启动kube-controller-manager

```bash
systemctl daemon-reload
systemctl start kube-controller-manager
systemctl enable kube-controller-manager
```



## 部署kube-scheduler

{% tabs comments %}

<!-- tab 创建配置文件 -->

```bash
cat > /opt/kubernetes/cfg/kube-scheduler.conf << EOF
KUBE_SCHEDULER_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--leader-elect \\
--kubeconfig=/opt/kubernetes/cfg/kube-scheduler.kubeconfig \\
--bind-address=127.0.0.1"
EOF
```

- --kubeconfig：连接apiserver配置文件
- --leader-elect：当该组件启动多个时，自动选举（HA）

<!-- endtab -->

<!-- tab 创建证书 -->

```bash
cd ~/TLS/k8s

# 创建证书请求文件
cat > kube-scheduler-csr.json << EOF
{
  "CN": "system:kube-scheduler",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF

# 生成证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-scheduler
```

<!-- endtab -->

<!-- tab 创建kubeconfig文件 -->

```bash
KUBE_CONFIG="/opt/kubernetes/cfg/kube-scheduler.kubeconfig"
KUBE_APISERVER="https://192.168.31.71:6443"

kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}

kubectl config set-credentials kube-scheduler \
  --client-certificate=./kube-scheduler.pem \
  --client-key=./kube-scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}

kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-scheduler \
  --kubeconfig=${KUBE_CONFIG}

kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
```

<!-- endtab -->

<!-- tab 创建启动文件 -->

```bash
cat > /usr/lib/systemd/system/kube-scheduler.service << EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-scheduler.conf
ExecStart=/opt/kubernetes/bin/kube-scheduler \$KUBE_SCHEDULER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

<!-- endtab -->

{% endtabs %}



## 启动kube-scheduler

```bash
systemctl daemon-reload
systemctl start kube-scheduler
systemctl enable kube-scheduler
```



## 创建kubectl证书文件连接集群

```bash
# 生成证书
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin

# 生成kubeconfig
mkdir /root/.kube

KUBE_CONFIG="/root/.kube/config"
KUBE_APISERVER="https://192.168.31.71:6443"

kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}

kubectl config set-credentials cluster-admin \
  --client-certificate=./admin.pem \
  --client-key=./admin-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}

kubectl config set-context default \
  --cluster=kubernetes \
  --user=cluster-admin \
  --kubeconfig=${KUBE_CONFIG}

kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
```



## 查看集群状态

```bash
kubectl get cs
NAME                STATUS    MESSAGE             ERROR
scheduler             Healthy   ok                  
controller-manager       Healthy   ok                  
etcd-2               Healthy   {"health":"true"}   
etcd-1               Healthy   {"health":"true"}   
etcd-0               Healthy   {"health":"true"}  
```

{% note info 'fas fa-bullhorn' %}

都是`health`表示集群现在是健康状态

{% endnote %}



## 授权kubelet-bootstrap用户允许请求证书

```bash
kubectl create clusterrolebinding kubelet-bootstrap \
--clusterrole=system:node-bootstrapper \
--user=kubelet-bootstrap
```



<br>



# 部署node节点

## 创建工作目录并拷贝二进制文件

```bash
mkdir -p /opt/kubernetes/{bin,cfg,ssl,logs} 
cd kubernetes/server/bin
cp kubelet kube-proxy /opt/kubernetes/bin  
```





## 部署kubelet

{% tabs comments %}

<!-- tab 创建配置文件 -->

```bash
cat > /opt/kubernetes/cfg/kubelet.conf << EOF
KUBELET_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--hostname-override=k8s-master1 \\
--network-plugin=cni \\
--kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \\
--bootstrap-kubeconfig=/opt/kubernetes/cfg/bootstrap.kubeconfig \\
--config=/opt/kubernetes/cfg/kubelet-config.yml \\
--cert-dir=/opt/kubernetes/ssl \\
--pod-infra-container-image=lizhenliang/pause-amd64:3.0"
EOF
```

- --hostname-override：显示名称，集群中唯一
- --network-plugin：启用CNI
- --kubeconfig：空路径，会自动生成，后面用于连接apiserver
- --bootstrap-kubeconfig：首次启动向apiserver申请证书
- --config：配置参数文件
- --cert-dir：kubelet证书生成目录
- --pod-infra-container-image：管理Pod网络容器的镜像

<!-- endtab -->

<!-- tab 创建配置参数文件 -->

```bash
cat > /opt/kubernetes/cfg/kubelet-config.yml << EOF
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: 0.0.0.0
port: 10250
readOnlyPort: 10255
cgroupDriver: cgroupfs
clusterDNS:
- 10.0.0.2
clusterDomain: cluster.local 
failSwapOn: false
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /opt/kubernetes/ssl/ca.pem 
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
maxOpenFiles: 1000000
maxPods: 110
EOF
```

<!-- endtab -->

<!-- tab 创建kubeconfig文件 -->

```bash
KUBE_CONFIG="/opt/kubernetes/cfg/bootstrap.kubeconfig"
KUBE_APISERVER="https://192.168.31.71:6443" # apiserver IP:PORT
TOKEN="c47ffb939f5ca36231d9e3121a252940" # 与token.csv里保持一致

# 生成 kubelet bootstrap kubeconfig 配置文件
kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}

kubectl config set-credentials "kubelet-bootstrap" \
  --token=${TOKEN} \
  --kubeconfig=${KUBE_CONFIG}

kubectl config set-context default \
  --cluster=kubernetes \
  --user="kubelet-bootstrap" \
  --kubeconfig=${KUBE_CONFIG}

kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
```

<!-- endtab -->

<!-- tab 创建启动文件 -->

```bash
cat > /usr/lib/systemd/system/kubelet.service << EOF
[Unit]
Description=Kubernetes Kubelet
After=docker.service

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kubelet.conf
ExecStart=/opt/kubernetes/bin/kubelet \$KUBELET_OPTS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

<!-- endtab -->

{% endtabs %}



## 启动kubelet

```bash
systemctl daemon-reload
systemctl start kubelet
systemctl enable kubelet
```



## 审批kubelet证书申请并加入集群

```bash
# 查看kubelet证书请求
kubectl get csr
NAME                                                   AGE    SIGNERNAME                                    REQUESTOR           CONDITION
node-csr-uCEGPOIiDdlLODKts8J658HrFq9CZ--K6M4G7bjhk8A   6m3s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending

# 批准申请
kubectl certificate approve node-csr-uCEGPOIiDdlLODKts8J658HrFq9CZ--K6M4G7bjhk8A

# 查看节点
kubectl get node
NAME         STATUS     ROLES    AGE   VERSION
k8s-master1   NotReady   <none>   7s    v1.18.3
```

{% note info 'fas fa-bullhorn' %}

由于网络插件还没有部署，节点会没有准备就绪 NotReady

{% endnote %}



## 部署kube-proxy

{% tabs comments %}

<!-- tab 创建配置文件 -->

```bash
cat > /opt/kubernetes/cfg/kube-proxy.conf << EOF
KUBE_PROXY_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--config=/opt/kubernetes/cfg/kube-proxy-config.yml"
EOF
```

<!-- endtab -->

<!-- tab 创建配置参数文件 -->

```bash
cat > /opt/kubernetes/cfg/kube-proxy-config.yml << EOF
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
metricsBindAddress: 0.0.0.0:10249
clientConnection:
  kubeconfig: /opt/kubernetes/cfg/kube-proxy.kubeconfig
hostnameOverride: k8s-master1
clusterCIDR: 10.0.0.0/24
EOF
```

<!-- endtab -->

<!-- tab 创建kube-proxy证书 -->

```bash
cd ~/TLS/k8s

# 创建证书请求文件
cat > kube-proxy-csr.json << EOF
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF

# 生成证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
```

<!-- endtab -->

<!-- tab 创建kubeconfig文件 -->

```bash
KUBE_CONFIG="/opt/kubernetes/cfg/kube-proxy.kubeconfig"
KUBE_APISERVER="https://192.168.31.71:6443"

kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}

kubectl config set-credentials kube-proxy \
  --client-certificate=./kube-proxy.pem \
  --client-key=./kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}

kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=${KUBE_CONFIG}

kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
```

<!-- endtab -->

<!-- tab 创建启动文件 -->

```bash
cat > /usr/lib/systemd/system/kube-proxy.service << EOF
[Unit]
Description=Kubernetes Proxy
After=network.target

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-proxy.conf
ExecStart=/opt/kubernetes/bin/kube-proxy \$KUBE_PROXY_OPTS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

<!-- endtab -->

{% endtabs %}



## 启动kube-proxy

```bash
systemctl daemon-reload
systemctl start kube-proxy
systemctl enable kube-proxy
```



<br>



## 部署网络插件callico

```bash
kubectl apply -f calico.yaml
kubectl get pods -n kube-system
```



## 检查集群pod状态

```bash
kubectl get node
NAME         STATUS   ROLES    AGE   VERSION
k8s-master   Ready    <none>   37m   v1.20.4
```



## 授权apiserver访问kubelet

```bash
cat > apiserver-to-kubelet-rbac.yaml << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
      - pods/log
    verbs:
      - "*"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF

kubectl apply -f apiserver-to-kubelet-rbac.yaml
```



## node节点扩容

```bash
# 拷贝相关文件
scp -r /opt/kubernetes root@192.168.31.72:/opt/

scp -r /usr/lib/systemd/system/{kubelet,kube-proxy}.service root@192.168.31.72:/usr/lib/systemd/system

scp /opt/kubernetes/ssl/ca.pem root@192.168.31.72:/opt/kubernetes/ssl

# 在node节点上删除kubeconfig文件
rm -f /opt/kubernetes/cfg/kubelet.kubeconfig 
rm -f /opt/kubernetes/ssl/kubelet*

# 修改kubelet配置文件
vi /opt/kubernetes/cfg/kubelet.conf
--hostname-override=k8s-node1 ## 修改这一项

vi /opt/kubernetes/cfg/kube-proxy-config.yml
hostnameOverride: k8s-node1  ## 修改这一项


# 启动
systemctl daemon-reload
systemctl start kubelet kube-proxy
systemctl enable kubelet kube-proxy
systemctl start kubelet kubelet
systemctl enable kubelet kubelet

# 在master上批准请求
# 查看证书请求
kubectl get csr
NAME           AGE   SIGNERNAME                    REQUESTOR           CONDITION
node-csr-4zTjsaVSrhuyhIGqsefxzVoZDCNKei-aE2jyTP81Uro   89s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending

# 授权请求
kubectl certificate approve node-csr-4zTjsaVSrhuyhIGqsefxzVoZDCNKei-aE2jyTP81Uro

# 查看node状态
kubectl get node
NAME       STATUS   ROLES    AGE     VERSION
k8s-master1   Ready    <none>   47m     v1.20.4
k8s-node1    Ready    <none>   6m49s   v1.20.4
```



<br>



# 常用插件部署

## 部署dashboard

```bash
kubectl apply -f kubernetes-dashboard.yaml
kubectl get pods,svc -n kubernetes-dashboard

# 创建serviceaccount
kubectl create serviceaccount dashboard-admin -n kube-system
kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
kubectl describe secrets -n kube-system $(kubectl -n kube-system get secret | awk '/dashboard-admin/{print $1}')
```



上一步将输出一个token，访问：https://NodeIP:30001，输入token即可进入页面



## 部署coredns

```bash
kubectl apply -f coredns.yaml 
 
kubectl get pods -n kube-system  
NAME                          READY   STATUS    RESTARTS   AGE 
coredns-5ffbfd976d-j6shb      1/1     Running   0          32s

# 测试解析
kubectl run -it --rm dns-test --image=busybox:1.28.4 sh 
If you don't see a command prompt, try pressing enter. 
 
/ # nslookup kubernetes 
Server:    10.0.0.2 
Address 1: 10.0.0.2 kube-dns.kube-system.svc.cluster.local 
 
Name:      kubernetes 
Address 1: 10.0.0.1 kubernetes.default.svc.cluster.local

```


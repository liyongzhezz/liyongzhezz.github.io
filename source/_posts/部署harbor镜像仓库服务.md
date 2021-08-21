---
title: 在k8s中部署harbor镜像仓库服务
date: 2021-08-21 16:19:37
tags:
- Docker
categories:
- Docker
- 镜像仓库
description: 在k8s中部署harbor仓库服务
cover: https://gimg2.baidu.com/image_search/src=http%3A%2F%2Fimg2018.cnblogs.com%2Fblog%2F381412%2F201907%2F381412-20190712224747490-693611835.png&refer=http%3A%2F%2Fimg2018.cnblogs.com&app=2002&size=f9999,10000&q=a80&n=0&g=0n&fmt=jpeg?sec=1632126138&t=13e5e3f719bc1466b62b68d8f4da4fe8
---



{% note info 'fas fa-bullhorn' %}

本文介绍kubernetes中部署harbor服务

更新于 2021-08-21

{% endnote %}

<br>





# 准备

首先创建一个namespace：

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: harbor
```



然后创建storageclass用于habror数据持久化存储：

```yaml
# storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: harbor-storageclass
  labels:
    app: nfs-client-provisioner
provisioner: cluster.local/nfs-client-provisioner
allowVolumeExpansion: true
reclaimPolicy: Retain
parameters:
  archiveOnDelete: "true"
```



执行下面的命令进行创建：

```bash
$ kubectl apply -f namespace.yaml
$ kubectl apply -f storageclass.yaml
```

<img src="prepare.png" style="zoom:70%;" />



<br>



# 下载harbor helm

Harbor 官方提供了对应的 Helm Chart 包，可以方便地进行安装，首先将其下载：

```bash
$ git clone https://github.com/goharbor/harbor-helm
```



官方说：`The master branch is in heavy development, please use the other stable versions instead`

，所以需要切换到其他稳定分支，例如：

```bash
$ cd harbor-helm
$ git checkout 1.2.0
```



<br>



# 准备配置文件

在helm下的`values.yaml`文件中定义了很多参数并且有详细的解释，这里我们使用一个新的文件来覆盖其中的部分参数实现自定义配置：

```yaml
# config.yaml
expose:
  type: ingress
  tls:
    enabled: true
  ingress:
    hosts:
      core: harbor.example.com
      notary: notary.example.com
    annotations:
      ingress.kubernetes.io/ssl-redirect: "true"
      ingress.kubernetes.io/proxy-body-size: "0"
      nginx.ingress.kubernetes.io/ssl-redirect: "true"
      nginx.ingress.kubernetes.io/proxy-body-size: "0"

externalURL: https://harbor.example.com

persistence:
  enabled: true
  resourcePolicy: "keep"
  persistentVolumeClaim:
    registry:
      storageClass: "harbor-data"
      size: 50Gi
    chartmuseum:
      storageClass: "harbor-data"
      size: 50Gi
    jobservice:
      storageClass: "harbor-data"
      size: 50Gi
    database:
      storageClass: "harbor-data"
      size: 50Gi
    redis:
      storageClass: "harbor-data"
      size: 50Gi
 
harborAdminPassword: "Harbor12345"
```



如果使用的是`traefik ingress`，则`annotations`则改为如下的形式，我这里用的是上边的`ingress-nginx`：

```yaml
annotations:
      ingress.kubernetes.io/ssl-redirect: "true"
      ingress.kubernetes.io/proxy-body-size: "0"
      kubernetes.io/ingress.class: "traefik"
      traefik.ingress.kubernetes.io/router.tls: "true"
      traefik.ingress.kubernetes.io/router.entrypoints: websecure
```



<br>



# 部署harbor

直接使用helm命令进行部署，执行下面的命令：

```bash
$ helm install harbor -f config.yaml harbor-helm/ -n harbor
```



确保都正常启动：

```bash
$ helm ls -n harbor
$ kubectl get pod,service,ingress -n harbor
```

<img src="check.png" style="zoom:70%;" />



# 设置nginx

在nginx中增加harbor的虚拟主机配置：

```nginx
# harbor.conf
upstream ingress-443 {
    server 10.8.138.12:443 max_fails=3 fail_timeout=5s;
}

server {
    listen 80;
    server_name harbor.example.com;
    rewrite ^(.*) https://$host$1 permanent;
}

server {
    listen 443 ssl;
    server_name harbor.example.com;

    ssl_certificate /etc/nginx/ssl/nginx.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx.key;

    access_log /var/log/nginx/harbor.example.com_access.log main;
    error_log /var/log/nginx/harbor.example.com_error.log;

    location / {
      proxy_pass https://ingress-443;
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

<br>



# 测试

本地绑定好host之后访问域名`harbor.example.com`，即可看到harbor的登录页面：

<img src="login.png" style="zoom:50%;" />



默认的用户名为admin，密码在配置文件中设置的，默认为`Harbor12345`，即可登录进去了：

<img src="index.png" style="zoom:50%;" />



<br>



# docker推送镜像

首先使用docker命令登录harbor：

```bash
$ docker login harbor.example.com
```



这是也许会出现这个报错：`Error response from daemon: Get https://harbor.example.com/v2/: x509: certificate signed by unknown authority`，是因为我们使用的证书是harbor自签的不受信任，所以需要修改docker的配置文件，将我们的仓库地址配置为非安全地址。



需要编辑`/etc/docker/daemon.json`，（如果没有就创建），增加下面的参数：

```bash
"insecure-registries": ["harbor.example.com"],
```



然后重启docker即可登录成功：

```bash
$ docker login harbor.example.com
Username: admin
Password:
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```



接下来下载一个镜像并重新打tag上传：

```bash
$ docker pull busybox:latest
$ docker tag busybox:latest harbor.example.com/library/busybox:v1
$ docker push harbor.example.com/library/busybox:v1
```



在页面上看，已经推上来镜像了，说明基本的harbor功能正常：

<img src="push.png" style="zoom:50%;" />


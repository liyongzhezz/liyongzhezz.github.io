---
title: Prometheus基于consul自动发现节点
date: 2020-07-31 09:21:49
tags:
- Prometheus
categories:
- Prometheus
- 基于Consul自动发现
description: prometheus使用consul自动发现监控节点
cover: https://pic.rmb.bdstatic.com/f42c77f750b95df7b34509c03f1f1ed6.png@wm_2,t_55m+5a625Y+3L+S4gOWPquacqOacqOmxvA==,fc_ffffff,ff_U2ltSGVp,sz_9,x_6,y_6
---



# 为什么使用consul

当新增了一个监控节点（例如新部署了一个node_exporter），往往需要修改prometheus的配置文件来增加节点信息。prometheus支持使用多种服务发现类型来自动加入监控节点，其中就包含consul。



所以只需要将新增的节点注册到consul中，prometheus会自动加入到监控。



<br>



# 部署consul

## 二进制部署

```bash
$ wget https://releases.hashicorp.com/consul/1.7.2/consul_1.7.2_linux_amd64.zip
$ unzip consul_1.7.2_linux_amd64.zip
$ mkdir /data1/consul
$ nohup ./consul agent -server -client=10.25.85.104 -bind=10.25.85.104 -data-dir=/data1/consul -bootstrap -ui &
```



相关参数：

- `-server`：启动server模式；
- `-client`：设置客户端地址；
- `-bind`：设置集群通信绑定的地址；
- `-data-dir`：数据目录；
- `-bootstrap`：设定自己为leader而不进行选举；
- `-ui`：启动内置管理界面；



## 容器化部署

直接下载并运行最新版consul镜像。

```bash
$ docker run --name consul -d -p 8500:8500 consul
```



## 访问测试

直接通过浏览器访问consul节点的8500端口即可进入consul的默认界面：

![](consul-index.png)



<br>



# 注册node_exporter到consul

准备一个用于注册的json数据文件`consul.json`：

```json
{
    "ID": "node-exporter-30.23.18.141",
    "Name": "node-exporter",
    "Tags": [
        "node-exporter"
    ],
    "Address": "30.23.18.141",
    "Port": 9100,
    "Meta": {
        "env": "test",
        "project": "eex"
    },
    "EnableTagOverride": false,
    "Checl": {
        "HTTP": "http://30.23.18.141:9100/metrics",
        "Interval": "10s"
    },
    "Weights": {
        "Passing": 10,
        "Warning": 1
    }
}
```



> Tag字段用于prometheus进行分类，meta中的信息可以随便写。



在安装好node_exporter的服务器上，执行下面的命令注册到consul：

```bash
$ curl --request PUT --data @consul.json http://10.25.85.104:8500/v1/agent/service/register?replace-existing-checks=1
```



执行完毕后，刷新页面发现node_exporter已经注册进来了。

![](zhuce.png)



<br>



# prometheus配置服务发现

编辑prometheus配置文件`prometheus.yml` 实现服务发现：

```yaml
- job_name: 'consul_node-exporter'
  consul_sd_configs:
    - server: '10.25.85.104:8500'
      services: []
  relabel_configs:
    - source_labels: [__meta_consul_tags]
      regex: .*node-exporter.*
      action: keep
    - regex: __meta_consul_service_metadata_(.+)
      action: labelmap
```

- `consul_sd_configs`表明使用consul服务发现机制；

- `relable_configs`中的第一段表示只保留`__meta_consul_tags`中包含`node-exporter`的对象；

- `relable_configs`中的第二段表示匹配`__meta_consul_service_metadata`开头的标签并将捕获到的内容作为新的标签名称；

  

执行下面的命令让prometheus检查配置并重新加载：

```bash
# 检查配置是否正确
promtool check config ./prometheus.yml

# 动态加载配置
curl -XPOST http://10.25.85.104:9090/-/reload
```



在prometheus界面上可以看到注册到consul的node_exporter已经被prometheus发现：

![](find.png)



‌<br>



# 通过consul集群自动发现



## consul集群规划

| 角色     | IP           | 端口 |
| -------- | ------------ | ---- |
| leader   | 30.23.105.83 | 8500 |
| follower | 30.23.107.10 | 8500 |
| follower | 30.23.8.76   | 8500 |



## 下载安装consul

在3台节点上都下载consul安装包：

```bash
$ wget https://releases.hashicorp.com/consul/1.7.2/consul_1.7.2_linux_amd64.zip
$ unzip consul_1.7.2_linux_amd64.zip
$ mv consul /usr/local/bin
$ mkdir /data/consul
$ mkdir /usr/local/consul
```



## 设置配置文件

在每一个节点上设置consul的配置文件：

```bash
$ cat > /usr/local/consul/consul.json << EOF
{
  "datacenter": "dc1",
  "data_dir": "/data/consul",
  "log_level": "INFO",
  "server": true,
  "node_name": "node1-30.23.105.83",
  "ui": true,
  "bind_addr": "30.23.105.83",
  "client_addr": "0.0.0.0",
  "advertise_addr": "30.23.105.83",
  "bootstrap_expect": 3,
  "ports":{
    "http": 8500,
    "dns": 8600,
    "server": 8300,
    "serf_lan": 8301,
    "serf_wan": 8302
    }
}
EOF
```



**注意修改每个节点配置文件中的节点名称和ip地址**



相关参数：

- `datacenter`：数据中心名称；
- `data_dir`*：*数据存放本地目录；
- `log_level`*：输出的日志级别；*
- `server`*：*以 server 身份启动实例，不指定默认为 `client` ；
- `node_name`：节点名称，集群中每个 node 名称不能重复，默认情况使用节点hostname
- `ui`：指定是否可以访问 UI 界面； 
- `bind_addr`：Consul 监听的地址，必须能够被集群中所有节点访问，默认为`0.0.0.0 `；
- `client_addr`：客户端监听地址，`0.0.0.0 `表示所有网段都可以访问；
-  `advertise_addr`：集群广播地址 ；
- `bootstrap_expect`：集群要求的最少成员数量 ；
- `ports`：该参数详细配置各个服务端口，如果想指定其他端口，可以修改这里；

‌

## 启动主节点

首先在leader服务器上启动服务：

```bash
$ nohup consul agent -config-dir=/usr/local/consul/consul.json > /usr/local/consul/consul.log 2>&1 &
```



启动后会发现有`failed to sync remote state: No cluster leader` 的报错，是因为在配置文件中指定了`bootstrap_expect`为3，故需要有三个节点集群才能正常启动，继续启动其他节点。



## 启动从节点

使用下面的命令启动其他节点并加入leader：

```bash
$ nohup consul agent -config-dir=/usr/local/consul/consul.json -join 30.23.105.83:8301 > /usr/local/consul/consul.log 2>&1 &
```



## 检查

启动完成后通过访问任意节点8500端口，打卡consul页面，可以看到三个节点都注册进来了：

![](cluster.png)



也可以通过下面的命令查看集群节点的情况：

```bash
# 查看集群状态
$ consul operator raft list-peers

# 查看节点状态
$ consul members
```



## 配置nginx代理consul集群

使用nginx作为consul集群的统一入口，nginx相关配置如下：

```nginx
upstream service_consul {
    server 30.23.105.83:8500;
    server 30.23.107.10:8500;
    server 30.23.8.76:8500;
    ip_hash;
}

server {
    listen       80;
    server_name  30.23.105.83;
    index  index.html index.htm;    

    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        add_header Access-Control-Allow-Origin *;
        proxy_next_upstream http_502 http_504 error timeout invalid_header;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://service_consul;    
    }

    access_log /var/log/consul.access.log;
    error_log /var/log/consul.error.log;    

    error_page  404              /404.html;

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

>  这样通过nginx的80端口就可以访问consul集群了，在prometheus也只需要配置nginx的80端口即可。







‌


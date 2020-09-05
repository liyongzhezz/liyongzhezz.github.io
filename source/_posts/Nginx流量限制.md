---
title: Nginx流量限制
date: 2020-07-29 10:00:20
tags:
- Nginx
categories: Nginx
description: 使用Nginx流量限制功能
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1595998302769&di=0b91df0693ce97ad9e6b87445c49d83f&imgtype=0&src=http%3A%2F%2Fwww.nd9p.com%2Fuploads%2Fallimg%2F161029%2F0UQ31J5_1.png%3Fw%3D480
---



流量限制(rate-limiting)，是Nginx中一个非常实用的功能。可以用来限制用户在给定时间内HTTP请求的数量。请求可以是一个简单网站首页的GET请求，也可以是登录表单的POST请求。



流量限制可以用作安全目的，如减慢暴力密码破解的速率、抵御DDOS攻击，保护上游应用服务器不被同时太多用户请求所压垮。



------



# 基本配置

使用`limit_req_zone`和`limit_req`来定义和使用流量控制，基本的设置如下：

```nginx
limit_req_zone $binary_remote_addr zone=mylimit:10m rate=10r/s;

server {
    location /login/ {
        limit_req zone=mylimit;
        proxy_pass http://my_upstream;
    }
}
```

- `limit_req_zone`定义流量控制相关参数；
- `limit_req`起用流量控制，将流控配置应用到location或server中；
- `$binary_remote_addr`：流量限制的key，以这个key作为流量控制的指标，可以使用其他的变量；
- `zone`：定义存储状态和被限制请求的URL访问频率的共享内存区，分为两个部分：
  - `mylimit`：定义区域的名字；
  - `10m`：区域大小；（1.6W个IP地址状态信息存储大约1M，所以10m可以存储大约16W个信息）
- `rate`：最大请求速率，例子中是每秒10个，也就是100毫秒1个（nginx以毫秒粒度追踪请求）；



> 当Nginx需要添加新条目时存储空间不足，将会删除旧条目。如果释放的空间仍不够容纳新记录，Nginx将会返回 503状态码(Service Temporarily Unavailable)。另外，为了防止内存被耗尽，Nginx每次创建新条目时，最多删除两条60秒内未使用的条目。



在这个实例中，每个IP地址被限制为每秒只能请求10次/login/，更准确地说，在前一个请求的100毫秒内不能请求该URL。



<br>



# 处理突发请求

如果某个时刻出现100ms内超过1次请求，并且确认是正常请求，那么需要配置允许超额的请求队列长度，否则服务器会返回503错误。

```nginx
location /login/ {
    limit_req zone=mylimit burst=20;
    proxy_pass http://my_upstream;
}
```



burst参数定义了超出zone指定速率的情况下客户端还能发起多少请求。上一个请求100毫秒内到达的请求将会被放入队列，这里将队列大小设置为20。



这意味着，如果从一个给定IP地址发送21个请求，Nginx会立即将第一个请求发送到上游服务器群，然后将余下20个请求放在队列中。然后每100毫秒转发一个排队的请求，只有当传入请求使队列中排队的请求数超过20时，Nginx才会向客户端返回503。



<br>



# 无延迟队列

brust的问题是队列中的请求需要排队，客户端的请求可能会出现响应等待时间过长的问题。使用`nodelay`参数可以解决：

```nginx
location /login/ {
    limit_req zone=mylimit burst=20 nodelay;
    proxy_pass http://my_upstream;
}
```



队列中有20个空位，从给定的IP地址发出的21个请求同时到达。Nginx会立即转发这个21个请求，并且标记队列中占据的20个位置，然后每100毫秒释放一个位置。如果是25个请求同时到达，Nginx将会立即转发其中的21个请求，标记队列中占据的20个位置，并且返回503状态码来拒绝剩下的4个请求。



<br>



# 设置返回状态码

默认情况下，如果请求由于流控被限制，则会返回客户端503状态码，可以通过下面的配置返回指定的状态码：

```nginx
location /login/ {
    limit_req zone=mylimit burst=20 nodelay;
    limit_req_status 404;
}
```


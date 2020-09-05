---
title: '[shell脚本系列]网络'
date: 2020-07-16 14:11:07
tags: 
- shell脚本
- 网络
categories:
- shell脚本系列
- 网络
description: 使用shell脚本监控网络相关信息
cover: https://ns-strategy.cdn.bcebos.com/ns-strategy/upload/fc_big_pic/part-00525-3.jpg
---



## 监控网卡流量

```shell
# network.sh
# 监控实时网卡流量
# $1 接收所传第一个参数 即要监控的网卡
NIC=$1
# echo -e "traffic in --- traffic out"
while true;do
        # $0 命令输出结果 ~ 匹配模式
        OLD_IN=`awk '$0~"'$NIC'"{print $2}' /proc/net/dev`
        OLD_OUT=`awk '$0~"'$NIC'"{print $10}' /proc/net/dev`
        sleep 1
        NEW_IN=`awk '$0~"'$NIC'"{print $2}' /proc/net/dev`
        NEW_OUT=`awk '$0~"'$NIC'"{print $10}' /proc/net/dev`
        clear
        # printf不换行 %s占位符
        IN=$(printf "%.1f%s" "$(($NEW_IN-$OLD_IN))" "B/s")
        OUT=$(printf "%.1f%s" "$(($NEW_OUT-$OLD_OUT))" "B/s")
        echo "       traffic in  `date +%k:%M:%S`  traffic out "
        echo "$NIC   $IN              $OUT"

done
```



使用方法：

```bash
$ sh network.sh eth0
```


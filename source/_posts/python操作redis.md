---
title: python操作redis
date: 2021-11-14 18:35:50
tags:
- Python
categories:
- 编程
- Python
- DB操作
description: python中操作redis数据库的方法
cover: https://gimg2.baidu.com/image_search/src=http%3A%2F%2Fpic3.zhimg.com%2Fv2-dbddf93833cd73caf69f480667f5d8ef_1200x500.jpg&refer=http%3A%2F%2Fpic3.zhimg.com&app=2002&size=f9999,10000&q=a80&n=0&g=0n&fmt=jpeg?sec=1639478194&t=62e3ee9e66298b24db779cc13e0024d1
---



{% note info 'fas fa-bullhorn' %}

本文介绍python中操作redis的方法

更新于 2021-11-14

{% endnote %}





# 安装

首先需要安装`redis`包：

```bash
$ pip install redis
```



<br>



# 代码

这里将redis操作都放到了一个类中：

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import redis


class RedisOperator(object):
    '''
        redis相关操作
    '''

    def __init__(self):
        self.redis_host = '1.1.1.1'
        self.redis_port = '6379'
        self.redis_pass = 'redis123'
        self.redis_db = 0
        self.redis_extime = 60	# 过期时间，不设置就永不过期

    def __connect(self):
        '''连接redis'''
        redis_host = self.redis_host
        redis_port = self.redis_port
        redis_pass = self.redis_pass
        redis_db = self.redis_db

        redis_db_url = {
            'host': redis_host,
            'port': redis_port,
            'password': redis_pass,
            'db': redis_db
        }

        return redis.Redis(**redis_db_url)

    def get_redis_data(self, key):
        '''查询key值，如果key不存在则返回None'''
        conn = self.__connect()
        data = conn.get(key)

        return data

    def set_redis_data(self, key, value):
        '''设置键值对，如果键值对已经存在则覆盖原来的值'''
        conn = self.__connect()
        data = value
        conn.set(
            name = key,
            value = data,
            ex = self.redis_extime
        )

        return 0
```


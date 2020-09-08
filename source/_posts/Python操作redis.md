---
title: Python操作redis
date: 2020-09-08 19:21:54
tags:
- Python
- 数据库
categories:
- Python
- Python操作数据库
description: 使用python操作redis数据库
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1599574285430&di=13d7f6b83db000b0a0298ab9c5e62a43&imgtype=0&src=http%3A%2F%2F5b0988e595225.cdn.sohucs.com%2Fimages%2F20180310%2F39fdd0b4d90346918a01b9f89556ecf3.jpeg
---



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


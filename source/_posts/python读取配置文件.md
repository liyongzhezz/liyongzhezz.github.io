---
title: python读取配置文件
date: 2020-07-23 15:51:11
tags:
- Python
categories:
- Python
- 代码
description: python读取ini格式的配置文件内容
cover:
---



首先定义一个配置文件如下的格式：

```ini
# host.conf
[register]
port = port80,port443
title = AlarmPort

[port80]
port = 80
host = 1.1.1.1,2.2.2.2, 3.3.3.3
```



`[register]`表示一个分组，下面的变量使用`key-value`格式



然后需要定义一个类，用来读取配置文件：

```python
# getconfig.py
import os
import ConfigParser


class Getconfig(object):

    def getconfig(self, section, key):
        conf = ConfigParser.ConfigParser()
        path = os.path.split(os.path.realpath(__file__))[0] + '/config/host.conf'
        conf.read(path)
        return conf.get(section, key)
```



这个类接收两个参数：

- `section`为配置文件中的分组的名称，如`register`；
- `key`为配置文件中分组下的key名，如`title`；



使用中，只需要实例化这个类，然后传入需要读取的配置即可，例如：

```python
import getconfig

config = getconfig.Getconfig()
regist_port = config.getconfig('register', 'port')
```


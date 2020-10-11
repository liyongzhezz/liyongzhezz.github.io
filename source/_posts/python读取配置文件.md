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



## ini类型配置文件



### 配置文件格式

首先定义一个配置文件如下的格式：

```ini
# host.ini
[localdb]  
host     = 127.0.0.1  
user     = root  
password = 123456  
port     = 3306  
database = mysql 
```



>  `[localdb]`表示一个分组，下面的变量使用`key-value`格式



### 读取配置文件

读取配置文件是利用python内置的`configparser`标准库，对配置文件进行解析。

```python
>>> from configparser import ConfigParser  
>>> cfg = ConfigParser()  
>>> cfg.read("host.ini")  
['/root/host.ini']  
>>> cfg.items("localdb")  
[('host', '127.0.0.1'), ('user', 'root'), ('password', '123456'), ('port', '3306'), ('database', 'mysql')]  
>>> dict(cfg.items("localdb"))
{'host': '127.0.0.1', 'user': 'root', 'password': '123456', 'port': '3306', 'database': 'mysql'}
```



或者定义一个类，用来读取配置文件：

```python
# getconfig.py
import os
from configparser import ConfigParser 


class Getconfig(object):

    def getconfig(self, section, key):
        conf = ConfigParser()
        path = os.path.split(os.path.realpath(__file__))[0] + '/config/host.ini'
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
regist_port = config.getconfig('localdb', 'port')
```

<br>



## json格式配置文件

### 配置文件格式

```json
# db.json
{  
    "localdb":{  
        "host": "127.0.0.1",  
        "user": "root",  
        "password": "123456",  
        "port": 3306,  
        "database": "mysql"  
    }  
}    
```



### 读取配置文件

python中可以直接用`json`标准库将json解析为字典进行操作：

```python
>>> import json  
>>> from pprint import pprint  
>>>   
>>> with open('/Users/Bobot/db.json') as j:  
...     cfg = json.load(j)['localdb']  
...   
>>> pprint(cfg)  
{'database': 'mysql',  
 'host': '127.0.0.1',  
 'password': '123456',  
 'port': 3306,  
 'user': 'root'} 
```



json格式的配置文件写注释不是很方便，而且如果配置文件嵌套过多容易出现问题。

<br>



## toml类型配置文件



### 配置文件格式

```toml
# config.toml
[mysql]  
host     = "127.0.0.1"  
user     = "root"  
port     = 3306  
database = "test"  
  
  [mysql.parameters]  
  pool_size = 5  
  charset   = "utf8"  
  
  [mysql.fields]  
  pandas_cols = [ "id", "name", "age", "date"]  
```



toml有点类似ini格式的配置文件，但是toml支持更多的数据类型，例如数组、时间戳等。



### 读取配置文件

首先需要安装一个包：

```bash
$ pip install toml
```



然后读取toml配置文件，将其转化为一个字典：

```python
>>> import toml  
>>> import os  
>>> from pprint import pprint  
>>> cfg = toml.load(os.path.expanduser("~/Desktop/config.toml"))  
>>> pprint(cfg)  
{'mysql': {'database': 'test',  
           'fields': {'pandas_cols': ['id', 'name', 'age', 'date']},  
           'host': '127.0.0.1',  
           'parameters': {'charset': 'utf8', 'pool_size': 5},  
           'port': 3306,  
           'user': 'root'}}  
```



<br>



## yaml/yml格式配置文件



### 配置文件格式

```yaml
# config.yaml
mysql:  
  host: "127.0.0.1"  
  port: 3306  
  user: "root"  
  password: "123456"  
  database: "test"  
  
  parameter:  
    pool_size: 5  
    charset: "utf8"  
  
  fields:  
    pandas_cols:   
      - id  
      - name  
      - age  
      - date  
```



### 读取配置文件

首先需要安装yaml包：

```bash
$ pip install yaml
```



`yaml`包包含`load()`和`safe_load()`方法，但是推荐使用`safe_load()`方法，因为`load()`方法会带来安全问题，可能会直接运行植入在yaml文件中的攻击命令。

```python
>>> import os  
>>> from pprint import pprint  
>>>   
>>> with open(os.path.expanduser("~/config.yaml"), "r") as config:  
...     cfg = yaml.safe_load(config)  
...   
>>> pprint(cfg)  
{'mysql': {'database': 'test',  
           'fields': {'pandas_cols': ['id', 'name', 'age', 'date']},  
           'host': '127.0.0.1',  
           'parameter': {'charset': 'utf8', 'pool_size': 5},  
           'password': '123456',  
           'port': 3306,  
           'user': 'root'}}  
```


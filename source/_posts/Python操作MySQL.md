---
title: Python操作MySQL
date: 2020-10-11 18:49:28
tags:
- Python
- 数据库
categories:
- Python
- Python操作数据库
description: python操作mysql数据库
cover: https://ss0.bdstatic.com/70cFuHSh_Q1YnxGkpoWK1HF6hhy/it/u=4233371922,1020712226&fm=26&gp=0.jpg
---



## python操作mysql模块

python操作mysql常用的模块有如下的几种：

- **原生sql**：
  - PyMySQL：支持python2，3；
  - MySQLdb：支持python2；
- **ORM框架**：
  - SQLAchemy



<br>



## pymysql模块

pymysql可以在python中直接去执行sql语句，首先需要进行安装：

```bash
$ pip install PyMySQL
```



### 连接到数据库

使用`connect()`方法，传入mysql相关的配置即可连接到数据库：

```python
import pymysql

# 创建连接
mysql_info = {
  "host": "127.0.0.1",
  "prot": 3306,
  "user": "root",
  "passowrd": "root123",
  "db": "testdb",
  "charset": "utf8mb4"
}
conn = pymysql.connect(**mysql_info)
```



### 创建游标

连接到数据库后，需要获取一个游标，游标的作用是用于进行后续的操作：

```python
# 创建游标(查询数据返回为元组格式)
# cursor = conn.cursor()

# 创建游标(查询数据返回为字典格式)
cursor = conn.cursor(pymysql.cursors.DictCursor)
```



### 执行sql语句

事先写好sql语句，传给游标的`execute`方法即可：

```python
# 执行SQL,返回受影响的行数
effect_row1 = cursor.execute("select * from USER")

# 执行SQL,返回受影响的行数,一次插入多行数据
effect_row2 = cursor.executemany("insert into USER (NAME) values(%s)", [("jack"), ("boom"), ("lucy")])  # 3
```

> execute和executemany方法返回的是受影响的行数。



如果是增删改查的sql，则执行后需要调用`commit()`方法进行提交保存：

```python
conn.commit()
```



### 获取数据

```python
# 获取所有数据
result = cursor.fetchall()

# 获取第一行数据
result = cursor.fetchone()

# 获取前 5 行数据
result = cursor.fetchmany(5)
```



### 获取最新数据的自增ID

```python
new_id = cursor.lastrowid
```





### 关闭连接和游标

最后需要关闭连接和游标，释放资源：

```python
cursor.close()
conn.close()
```





<br>



## 使用连接池

上述方式可能会在一些情况下频繁创建和断开数据库连接，可以使用`DBUtils`来实现一个数据库连接池。连接池有两种连接方式：

- 为每个线程创建一个连接，线程调用了`close()`方法也不会关闭，会把连接重新放到池中；
- 创建一批连接到连接池，供所有线程共享使用（推荐方式）；



### 方式一

示例代码：

```python
#! /usr/bin/env python
# -*- coding: utf-8 -*-

from DBUtils.PersistentDB import PersistentDB
import pymysql

POOL = PersistentDB(
    creator=pymysql,  # 使用链接数据库的模块
    maxusage=None,  # 一个链接最多被重复使用的次数，None表示无限制
    setsession=[],  # 开始会话前执行的命令列表。如：["set datestyle to ...", "set time zone ..."]
    ping=0,
    # ping MySQL服务端，检查是否服务可用。# 如：0 = None = never, 1 = default = whenever it is requested, 2 = when a cursor is created, 4 = when a query is executed, 7 = always
    closeable=False,
    # 如果为False时， conn.close() 实际上被忽略，供下次使用，再线程关闭时，才会自动关闭链接。如果为True时， conn.close()则关闭链接，那么再次调用pool.connection时就会报错，因为已经真的关闭了连接（pool.steady_connection()可以获取一个新的链接）
    threadlocal=None,  # 本线程独享值得对象，用于保存链接对象，如果链接对象被重置
    host='127.0.0.1',
    port=3306,
    user='root',
    password='root123',
    database='testdb',
    charset='utf8',
)


def func():
    conn = POOL.connection(shareable=False)
    cursor = conn.cursor()
    cursor.execute('select * from USER')
    result = cursor.fetchall()
    cursor.close()
    conn.close()
    return result


result = func()
print(result)
```





### 方式二

示例代码：

```python
#! /usr/bin/env python
# -*- coding: utf-8 -*-

import time
import pymysql
import threading
from DBUtils.PooledDB import PooledDB, SharedDBConnection

POOL = PooledDB(
    creator=pymysql,  # 使用链接数据库的模块
    maxconnections=6,  # 连接池允许的最大连接数，0和None表示不限制连接数
    mincached=2,  # 初始化时，链接池中至少创建的空闲的链接，0表示不创建
    maxcached=5,  # 链接池中最多闲置的链接，0和None不限制
    maxshared=3,
    # 链接池中最多共享的链接数量，0和None表示全部共享。PS: 无用，因为pymysql和MySQLdb等模块的 threadsafety都为1，所有值无论设置为多少，_maxcached永远为0，所以永远是所有链接都共享。
    blocking=True,  # 连接池中如果没有可用连接后，是否阻塞等待。True，等待；False，不等待然后报错
    maxusage=None,  # 一个链接最多被重复使用的次数，None表示无限制
    setsession=[],  # 开始会话前执行的命令列表。如：["set datestyle to ...", "set time zone ..."]
    ping=0,
    # ping MySQL服务端，检查是否服务可用。# 如：0 = None = never, 1 = default = whenever it is requested, 2 = when a cursor is created, 4 = when a query is executed, 7 = always
    host='127.0.0.1',
    port=3306,
    user='root',
    password='root123',
    database='testdb',
    charset='utf8',
)


def func():
    # 检测当前正在运行连接数的是否小于最大链接数，如果不小于则：等待或报raise TooManyConnections异常
    # 否则
    # 则优先去初始化时创建的链接中获取链接 SteadyDBConnection。
    # 然后将SteadyDBConnection对象封装到PooledDedicatedDBConnection中并返回。
    # 如果最开始创建的链接没有链接，则去创建一个SteadyDBConnection对象，再封装到PooledDedicatedDBConnection中并返回。
    # 一旦关闭链接后，连接就返回到连接池让后续线程继续使用。
    conn = POOL.connection()

    # print('连接被拿走了', conn._con)
    # print('池子里目前有', POOL._idle_cache, '\r\n')

    cursor = conn.cursor()
    cursor.execute('select * from USER')
    result = cursor.fetchall()
    conn.close()
    return result


result = func()
print(result)
```



<br>



## 加锁

示例代码：

```python
#! /usr/bin/env python
# -*- coding: utf-8 -*-


import pymysql
import threading
from threading import RLock

LOCK = RLock()
CONN = pymysql.connect(host='127.0.0.1',
                       port=3306,
                       user='root',
                       password='root123',
                       database='testdb',
                       charset='utf8')


def task(arg):
    with LOCK:
        cursor = CONN.cursor()
        cursor.execute('select * from USER ')
        result = cursor.fetchall()
        cursor.close()

        print(result)


for i in range(10):
    t = threading.Thread(target=task, args=(i,))
    t.start()
```



<br>



## 实例

```python
import pymysql
import threading
from DBUtils.PooledDB import PooledDB, SharedDBConnection


POOL = PooledDB(
    creator=pymysql,  # 使用链接数据库的模块
    maxconnections=20,  # 连接池允许的最大连接数，0和None表示不限制连接数
    mincached=2,  # 初始化时，链接池中至少创建的空闲的链接，0表示不创建
    maxcached=5,  # 链接池中最多闲置的链接，0和None不限制
    #maxshared=3,  # 链接池中最多共享的链接数量，0和None表示全部共享。PS: 无用，因为pymysql和MySQLdb等模块的 threadsafety都为1，所有值无论设置为多少，_maxcached永远为0，所以永远是所有链接都共享。
    blocking=True,  # 连接池中如果没有可用连接后，是否阻塞等待。True，等待；False，不等待然后报错
    maxusage=None,  # 一个链接最多被重复使用的次数，None表示无限制
    setsession=[],  # 开始会话前执行的命令列表。如：["set datestyle to ...", "set time zone ..."]
    ping=0,
    # ping MySQL服务端，检查是否服务可用。# 如：0 = None = never, 1 = default = whenever it is requested, 2 = when a cursor is created, 4 = when a query is executed, 7 = always
    host='127.0.0.1',
    port=3306,
    user='root',
    password='root123',
    database='testdb',
    charset='utf8',
)


def connect():
    # 创建连接
    conn = POOL.connection()
    # 创建游标
    cursor = conn.cursor(pymysql.cursors.DictCursor)

    return conn,cursor

def close(conn, cursor):
    # 关闭游标
    cursor.close()
    # 关闭连接
    conn.close()

def fetch_one(sql):
    conn,cursor = connect()
    # 执行SQL，并返回收影响行数
    effect_row = cursor.execute(sql)
    result = cursor.fetchone()
    close(conn,cursor)

    return result

def fetch_all(sql):
    conn, cursor = connect()

    # 执行SQL，并返回收影响行数
    cursor.execute(sql)
    result = cursor.fetchall()

    close(conn, cursor)
    return result

def insert(sql):
    """
    创建数据
    :param sql: 含有占位符的SQL
    :return:
    """
    conn, cursor = connect()

    # 执行SQL，并返回收影响行数
    effect_row = cursor.execute(sql)
    conn.commit()

    close(conn, cursor)

def delete(sql):
    """
    创建数据
    :param sql: 含有占位符的SQL
    :return:
    """
    conn, cursor = connect()

    # 执行SQL，并返回收影响行数
    effect_row = cursor.execute(sql)

    conn.commit()

    close(conn, cursor)

    return effect_row

def update(sql):
    conn, cursor = connect()

    # 执行SQL，并返回收影响行数
    effect_row = cursor.execute(sql)

    conn.commit()

    close(conn, cursor)

    return effect_row
```




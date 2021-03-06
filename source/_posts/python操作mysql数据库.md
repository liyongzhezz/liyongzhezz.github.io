---
title: python操作mysql数据库
date: 2021-06-14 13:21:59
tags:
- Python
categories:
- 编程
- Python
- DB操作
description: python操作mysql数据库
cover: https://ss0.bdstatic.com/70cFuHSh_Q1YnxGkpoWK1HF6hhy/it/u=4233371922,1020712226&fm=26&gp=0.jpg
---

{% note info 'fas fa-bullhorn' %}

本文介绍python中操作mysql的方法

更新于 2021-06-14

{% endnote %}

<br>



# python操作mysql的模块

python操作mysql常用的模块有如下的几种：

- **原生sql**：
  - PyMySQL：支持python2，3；
  - MySQLdb：支持python2；
- **ORM框架**：
  - SQLAchemy



<br>



# pymysql操作数据库



## 安装

```bash
pip install PyMySQL
```



## 连接到数据库

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



## 创建游标

连接到数据库后，需要获取一个游标，游标的作用是用于进行后续的操作：

```python
# 创建游标(查询数据返回为元组格式)
cursor = conn.cursor()

# 创建游标(查询数据返回为字典格式)
cursor = conn.cursor(pymysql.cursors.DictCursor)
```



## 执行sql语句

事先写好sql语句，传给游标的`execute`方法即可，返回值为受影响的行数：

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



## 获取数据

```python
# 获取所有数据
result = cursor.fetchall()

# 获取第一行数据
result = cursor.fetchone()

# 获取前 5 行数据
result = cursor.fetchmany(5)

# 获取最新数据的自增ID 
new_id = cursor.lastrowid
```



## 关闭连接和游标

最后需要关闭连接和游标，释放资源：

```python
cursor.close()
conn.close()
```



<br>



## pymysql使用连接池

上述方式可能会在一些情况下频繁创建和断开数据库连接，可以使用`DBUtils`来实现一个数据库连接池。连接池有两种连接方式：

- 为每个线程创建一个连接，线程调用了`close()`方法也不会关闭，会把连接重新放到池中；
- 创建一批连接到连接池，供所有线程共享使用（推荐方式）；



### 方式一、为每个线程创建连接

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



### 方式二、创建连接到连接池

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



## pymysql加锁

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



## 实例代码

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



<br>



# SQLAlchemy ORM操作数据库



## 安装

```bash
pip install SQLAlchemy
```



检查是否安装成功：

```bash
>>> import sqlalchemy
>>> sqlalchemy.__version__
'1.3.20'
```





## 连接到数据库

```python
from sqlalchemy import create_engine

engine = create_engine('mysql://root:root123@172.21.1.100:3306/news?charset=utf8')
```



使用`create_engine`连接数据库，格式为：

```
mysql://<用户名>:<密码>@<数据库地址>:<端口>/库名?charset=<编码>
```



## 定义模型

```python
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy import Column, Integer, String, DateTime, Boolean

engine = create_engine('mysql://root:root123@172.21.1.100:3306/news?charset=utf8')
Base = declarative_base()

class News(Base):
    __tablename__ = 'news'
    id = Column(Integer, primary_key=True)
    title = Column(String(200), nullable=False)
    content = Column(String(2000), nullable=False)
    types = Column(String(10), nullable=False)
    image = Column(String(300))
    author = Column(String(20))
    view_count = Column(Integer)
    created_at = Column(DateTime)
    is_valid = Column(Boolean)
```



## 创建表

在定义好表结构后，可以这样进行创建表结构：

```python
>>> from mysql_orm import News, engine
>>> News.metadata.create_all(engine)
```



## 数据增删改查

```python
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy import Column, Integer, String, DateTime, Boolean
from sqlalchemy.orm import sessionmaker

engine = create_engine('mysql://root:root123@172.21.1.100:3306/news?charset=utf8')
Base = declarative_base()
Session = sessionmaker(bind=engine)

class News(Base):
    ......
  
class ORM(object):
    def __init__(self):
        self.session = Session()
        
    def insert(self):
        '''新增数据'''
        new_obj = News(
            title = "this is title",
            content = "this is content"
            types = "type1"
        )
        self.session.add(new_obj)
        self.session.commit()
        return new_obj
      
    def select_on(self):
        '''查询一条数据'''
        result = self.session.query(News).get(id=1)
        return result
        
    def select_more(self):
        '''查询多条数据'''
        result = self.session.query(News).filter_by(is_valid=True).order_by(id)
        return result
      
    def update(self):
        '''更新数据'''
        obj = self.session.query(News).filter_by(id=10)
        for item in obj：
            item.is_valid = 0
            self.session.add(item)
        self.session.commit()
        return obj
      
    def delete(self):
        '''删除数据'''
        data = self.session.query(News).filter_by(id=11)
        for item in data:
            self.session.delete(data)
        self.session.commit()
```



## 操作已存在的表

首先连接到数据库：

```python
from sqlalchemy import create_engine, MetaData
from sqlalchemy.orm import Session

class Mysql(object):
    def __init__(self):
        self.host = "127.0.0.1"
        self.pwd = "root123"
        self.port = "3306"
        self.user = "root"

    def connect(self, db):
        uri = 'mysql://{}:{}@{}:{}/{}'.format(self.user, self.pwd, self.host, self.port, db)
        engine = create_engine(uri, echo=True)
        metadata = MetaData(engine)
        session = Session(engine)

        return {'session': session, 'metadata': metadata, 'engine': engine}
      
```



查询数据：

```python
from sqlalchemy import Table

table = "t_test_table"

db_meta = Mysql().connect(table)  
table_obj = Table(self.table, db_meta['metadata'], autoload=True, autoload_with=db_meta['engine'])        

rest = db_meta['session'].query(table_obj).filter_by(id=1).value('name')        
print(rest)
```



修改数据：

```python
from sqlalchemy import Table

table = "t_test_table"

db_meta = Mysql().connect(table)  
table_obj = Table(self.table, db_meta['metadata'], autoload=True, autoload_with=db_meta['engine'])        

db_meta['session'].execute(table_obj.update().where(table_obj.c.id==2).values(name="jack"))
db_meta['session'].commit()
rest = db_meta['session'].query(table_obj).filter_by(id=2).value('name')
print(rest)
```





删除数据：

```python
from sqlalchemy import Table

table = "t_test_table"

db_meta = Mysql().connect(table)  
table_obj = Table(self.table, db_meta['metadata'], autoload=True, autoload_with=db_meta['engine'])    

db_meta['session'].execute(table_obj.delete().where(table_obj.c.id==3))
```





新增数据：

```python
from sqlalchemy import Table

table = "t_test_table"

db_meta = Mysql().connect(table)  
table_obj = Table(self.table, db_meta['metadata'], autoload=True, autoload_with=db_meta['engine'])    

db_meta['session'].execute(table_obj.insert(),{"name":"mike"})
```


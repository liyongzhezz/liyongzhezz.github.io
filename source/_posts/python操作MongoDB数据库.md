---
title: python操作MongoDB数据库
date: 2020-11-29 12:47:50
tags:
- Python
- 数据库
categories:
- Python
- Python操作数据库
description: python操作MongoDB数据库
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1606635418977&di=52be82b3a322192e74cf8143a2b81b7e&imgtype=0&src=http%3A%2F%2Fwww.linuxeden.com%2Fwp-content%2Fuploads%2F2018%2F06%2F074311_vbyh_2720166.png
---



## 安装pymongo

mongodb提供了`pymongo`来进行mongodb的操作，执行下面的命令进行安装：

```bash
$ pip install pymongo
```



<br>



## 连接mongodb

```python
from pymongo import MongoClient

# 方式一：指定地址端口方式连接
client = MongoClient('localhost', 27017)

# 方式二：指定URL方式连接
client = MongoClient('mongodb://localhost:27017/')
```



> mongodb默认自己带了线程池，所以连接之后无需主动关闭

<br>



## 查询数据



### 查询数据库信息

```python
from pymongo import MongoClient

client = MongoClient('mongodb://localhost:27017/')

# 查看当前连接的数据库端口
client.PORT

# 查看当前连接的数据库地址
client.HOST

# 查看所有的数据库
client.database_names()

# 连接到数据库
db = client.<数据库名字>
```



### 查询一条数据

```python
from pymongo import MongoClient

client = MongoClient('mongodb://localhost:27017/')
db = client.test

rest = db.blog.posts.find_one()

# 有条件查询
rest = db.blog.posts.find_one({'name': 'xxxx'})
```



### 查询多条数据

```python
from pymongo import MongoClient

client = MongoClient('mongodb://localhost:27017/')
db = client.test

rest = db.blog.posts.find()

# 有条件查询
rest = db.blog.posts.find({'author': 'xxxx'})
```



## 根据ID查询数据

```python
from pymongo import MongoClient
from bson.objectid import ObjectId

client = MongoClient('mongodb://localhost:27017/')
db = client.test

rest = db.blog.posts.find()
id = "xxxxxxxxxxxx"

obj = ObjectId(oid)
rest = db.blog.posts.find_one({'_id': obj})
```

> mongo中的id是一个对象，查询的时候也要将id转换为对象进行查询



<br>



## 新增数据



```python
from pymongo import MongoClient

client = MongoClient('mongodb://localhost:27017/')

# 连接到数据库
db = client.test

# 待插入的文档
post = {
  'title': 'xxxx',
  'content': 'sssss',
  'author': 'qqqq'
}

# 插入数据
rest = db.blog.posts.insert_one(post)

# 插入成功后会返回id
print(rest.inserted_id)
```



<br>



## 修改数据



### 修改一条数据

```python
from pymongo import MongoClient

client = MongoClient('mongodb://localhost:27017/')

# 连接到数据库
db = client.test

rest = db.blog.posts.update_one({'title': 'first'}, {'$inc': {'x': 3}})

# 匹配的行数
print(rest.matched_count)
# 修改的行数
print(rest.modified_count)
```

> 匹配到`title`为`first`的记录，给每个记录的`x`增加3，如果没有则设置为3



### 修改多条数据

```python
from pymongo import MongoClient

client = MongoClient('mongodb://localhost:27017/')

# 连接到数据库
db = client.test

rest = db.blog.posts.update_many({}, {'$inc': {'x': 1}})

# 匹配的行数
print(rest.matched_count)
# 修改的行数
print(rest.modified_count)
```

> 修改所有的数据，将`x`字段增加1，如果没有则设置为1



<br>



## 删除数据

```python	
from pymongo import MongoClient

client = MongoClient('mongodb://localhost:27017/')

# 连接到数据库
db = client.test

# 删除一条数据
rest = db.blog.posts.delete_one({'title': 'xxxxx'})

# 删除多条数据
rest = db.blog.posts.delete_many({'x': 1})

# 打印删除的行数
print(rest.deleted_count)
```




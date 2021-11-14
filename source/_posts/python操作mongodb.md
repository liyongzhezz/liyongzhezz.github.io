---
title: python操作mongodb
date: 2021-11-14 18:22:07
tags:
- Python
categories:
- 编程
- Python
- DB操作
description: python连接并操作mongodb数据库
cover: https://gimg2.baidu.com/image_search/src=http%3A%2F%2F5b0988e595225.cdn.sohucs.com%2Fimages%2F20170911%2Feb0ef74b80c54905b13014823d588aa3.png&refer=http%3A%2F%2F5b0988e595225.cdn.sohucs.com&app=2002&size=f9999,10000&q=a80&n=0&g=0n&fmt=jpeg?sec=1639477392&t=a9e02705207c17c9b506088dc4013f78
---



{% note info 'fas fa-bullhorn' %}

本文介绍python中操作mongodb的方法

更新于 2021-11-14

{% endnote %}



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



<br>



## ORM方式操作mongodb



### 安装

ORM方式操作需要安装`mongoengine`

```bash
$ pip install mongoengine
```



### 连接数据库

这里以连接`students`库为例：

```python
from mongoengine import connect

# 方式一、指定端口和地址
conn = connect('students', host='127.0.0.1', port='27017', username='xxx', password='xxxx')

# 方式二、指定url方式
conn = connect('students'， host='mongodb://username:password@127.0.0.1/database_name')
```



### ODM模型

```python
from datetime import datetime
from mongoengine import connect, DynamicDocument, EmbeddedDocument, StringField, \
		IntField, FloatField, DateTimeField, ListField

conn = connect('students', host='127.0.0.1', port='27017', username='xxx', password='xxxx')

class Grade(EmbeddedDocument):
  name = StringField(required=True)
  score = FloatField(required=True)
  
SEX_CHOICES = (
    ('female', '女'),
    ('male', '男')
)

class Student(DynamicDocument):
  name = StringField(required=True, max_length=32)
  age = IntField(required=True)
  sex = StringField(required=True, choices=SEX_CHOICES)
  grade = FloatField()
  create_at = DateTimeField(default=datetime.now())
  grades = ListField(EmbeddedDocumentField(Grade))
  
  meta = {
    'collection': 'students',
    'ordering': ['-age']
  }
```



> 1、EmbeddedDocumentField 表示引用嵌套文档
>
> 2、DynamicDocument 的好处是可以随时增加字段，而不用修改class中的内容



### ODM增删改查

```python
from datetime import datetime
from mongoengine import connect, DynamicDocument, EmbeddedDocument, StringField, \
		IntField, FloatField, DateTimeField, ListField

conn = connect('students', host='127.0.0.1', port='27017', username='xxx', password='xxxx')
  
class Grade(EmbeddedDocument):
  ......
  
SEX_CHOICES = (
    ('female', '女'),
    ('male', '男')
)

class Student(DynamicDocument):
  ......
  
  
class MongoODM(object):

  def get_one(self):
    '''查询一条数据'''
    return Student.objects.first()

  def get_all(self):
    '''查询所有数据'''
    return Student.objects.all()

  def get_by_id(self, id):
    '''根据id查询'''
    rest = Student.objects.filter(id=id).first()
    
  def update_one(self):
    '''修改一条数据'''
    rest = Student.objects.filter(sex='male').update_one(inc__age=1)
    return rest
  
  def update_many(self):
    '''修改多条数据'''
    rest = Student.objects.filter(sex='male').update(inc__age=1)
    return rest
  
  def delete_one(self):
    '''删除一条数据'''
    rest = Student.objects.filter(sex='male').first().delete()
    return rest
  
  def delete_many(self):
    '''删除多条数据'''
    rest = Student.objects.filter(sex='male').delete()
    return rest
  
  def add_one(self):
    '''新增一条数据'''
    yuwen = Grade(
      name='语文',
      score=95
    )
    
    stu_obj = Student(
      name='tom',
      age=21,
      sex='male',
      grades=[yuwen]
    )
    stu_obj.remark = 'remark'
    stu_obj.save()
    return stu_obj
```


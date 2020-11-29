---
title: python-ODM操作mongodb
date: 2020-11-29 14:36:11
tags:
- Python
- 数据库
categories:
- Python
- Python操作数据库
description: pyton使用ODM方式操作mongodb数据库
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1606641872741&di=cb5c2e44bb70d8e6de9f71b7a1b34aed&imgtype=0&src=http%3A%2F%2Fwww.siyuanblog.com%2Fwp-content%2Fuploads%2F2018%2F09%2Fmongodb_7.png
---



## 安装MongoEngine

执行下面的命令安装：

```bash
$ pip install mongoengine
```



<br>



## 连接数据库

这里以连接students库为例：

```python
from mongoengine import connect

# 方式一、指定端口和地址
conn = connect('students', host='127.0.0.1', port='27017', username='xxx', password='xxxx')

# 方式二、指定url方式
conn = connect('students'， host='mongodb://username:password@127.0.0.1/database_name')
```



<br>



## ODM模型

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



<br>



## ODM增删改查

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


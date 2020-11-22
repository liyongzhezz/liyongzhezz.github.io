---
title: python ORM操作MySQL
date: 2020-11-22 11:00:04
tags:
- Python
- 数据库
categories:
- Python
- Python操作数据库
description: python使用ORM方式操作mysql数据库
cover: https://ss0.bdstatic.com/70cFuHSh_Q1YnxGkpoWK1HF6hhy/it/u=4233371922,1020712226&fm=26&gp=0.jpg
---



## 安装SQLAlchemy

```bash
$ pip install SQLAlchemy
```



检查是否安装成功：

```python
>>> import sqlalchemy
>>> sqlalchemy.__version__
'1.3.20'
```



<br>



## 连接到数据库

```python
from sqlalchemy import create_engine

engine = create_engine('mysql://root:root123@172.21.1.100:3306/news?charset=utf8')
```



使用`create_engine`连接数据库，格式为：

```
mysql://<用户名>:<密码>@<数据库地址>:<端口>/库名?charset=<编码>
```



<br>



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



<br>



## 创建表

在定义好表结构后，可以这样进行创建表结构：

```python
>>> from mysql_orm import News, engine
>>> News.metadata.create_all(engine)
>>> 
```



<br>



## 数据的增删改查

```python
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy import Column, Integer, String, DateTime, Boolean
from sqlalchemy import sessionmaker

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




---
title: Python操作已经存在的表
date: 2020-12-25 16:06:30
tags:
- Python
categories:
- Python
- Python操作数据库
description: python对已经存在的表进行增删改查
cover:
---



## 连接数据库

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



<br>



## 查询数据

```python
from sqlalchemy import Table

table = "t_test_table"

db_meta = Mysql().connect(table)  
table_obj = Table(self.table, db_meta['metadata'], autoload=True, autoload_with=db_meta['engine'])        

rest = db_meta['session'].query(table_obj).filter_by(id=1).value('name')        
print(rest)
```



<br>



## 修改数据

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



<br>



## 删除数据

```python
from sqlalchemy import Table

table = "t_test_table"

db_meta = Mysql().connect(table)  
table_obj = Table(self.table, db_meta['metadata'], autoload=True, autoload_with=db_meta['engine'])    

db_meta['session'].execute(table_obj.delete().where(table_obj.c.id==3))
```



<br>



## 新增数据

```python
from sqlalchemy import Table

table = "t_test_table"

db_meta = Mysql().connect(table)  
table_obj = Table(self.table, db_meta['metadata'], autoload=True, autoload_with=db_meta['engine'])    

db_meta['session'].execute(table_obj.insert(),{"name":"mike"})
```


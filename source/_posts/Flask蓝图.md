---
title: Flask蓝图
date: 2020-08-03 15:33:28
tags:
- Flask
categories:
- python web开发
- Flask
description: Flask蓝图功能的介绍和使用。
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1596450146150&di=93ca29c3c5343511eb59e24e03763483&imgtype=0&src=http%3A%2F%2Fwww.45fan.com%2Fupload%2F2018-12-10%2F19101690242527503074058584.png
---



# 什么是蓝图

蓝图类似于Django中的路由`include`（个人认为），它是一系列相同类型操作的集合，它是一个路由注册的入口。



一般的flask项目就是一个`app.py`文件，其中定义了路由和路由对应的处理逻辑，例如：

```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def index():
  pass

@app.route('/admin')
def admin():
  pass

@app.route('/admin/edit')
def adminEdit():
  pass

@app.route('/home')
def home():
  pass

@app.route('/home/list')
def homeList():
  pass

if __name__ == '__main__':
  app.run()
```



随着项目扩张，`app.py`文件中的路由会越来越多，随即就有如下的痛点：

1. 对于前台界面和后台管理界面如何进行区分；
2. 如何对将`app.py`文件拆分的简洁；
3. 相同的uri前缀如`/admin`如何只定义一次；
4. 多人如何高效协同开发；
5. ......



<br>



# 使用蓝图

对于以上问题，可以用蓝图的方式解决，例如下面的项目：

![](tree.png)

- `app`目录下表示不同的服务：
  - `admin`表示后台管理服务；
  - `home`表示前台服务；
- `app.py`程序启动文件；



我们的目标是将前台的代码都放到`home`目录下管理，后台管理的代码都放到`admin`下管理。于是在`admin`下的`__init__.py`文件中使用蓝图：

```python
from flask import Blueprint


admin = Blueprint("admin", __name__)

import app.admin.views
```



此时，`admin`下的`views.py`就可以写成如下的形式：

```python
from . import admin


@admin.route("/")
def index():
    return "<h1>this is admin </h1>"

@admin.route("/edit")
def edit():
    return "this is edit"
```



同理，`home`下的`__init__.py`文件需要改写成如下的内容：

```python
from flask import Blueprint


home = Blueprint("home", __name__)

import app.home.views
```



`home`下的`views.py`改写成如下内容：

```python
from . import home
from flask import render_template, redirect, url_for


@home.route("/")
def index():
    return render_template("home/index.html")

@home.route("/login/")
def login():
    return render_template("home/login.html")

@home.route("/logout/")
def logout():
    return redirect(url_for("home.login"))
```



蓝图定义好后，还需要注册，在`app`下的`__init__.py`文件中注册：

```python
from flask import Flask
from app.home import home as home_blueprint
from app.admin import admin as admin_blueprint


app = Flask(__name__)
app.debug = True

app.register_blueprint(home_blueprint)
app.register_blueprint(admin_blueprint, url_prefix="/admin")
```



这里使用了`url_prefix`，这样所有后台管理的url都会加上`admin`前缀。



此时的`app.py`文件，就是一个很简单的启动入口：

```python
from app import app


if __name__ == "__main__":
    app.run()
```



<br>



# 总结

使用蓝图后可以看到，项目会根据app进行拆分（类似于Django中app的概念），通过蓝图将app注册到项目中，并且可以通过url的前缀区分app。`app.py`入口文件会变得极为简单。



对于后续协同开发，开发前台的人只需要修改`home`下的文件，后台的人只需要修改`admin`下的文件即可，不会有影响。新的app只需要将app的蓝图注册到`__inint__.py`中，并指定`url_prefix`进行区分即可。






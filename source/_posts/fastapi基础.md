---
title: fastapi基础
date: 2020-10-17 11:29:58
tags:
- FastAPI
categories:
- python web开发
- FastAPI
description: 最快的python api框架 FastAPI
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1602915595596&di=305a7f05aac3fc9d47fb05cf130c2e63&imgtype=0&src=http%3A%2F%2Fimg-blog.csdnimg.cn%2F20200225165843487.png%3Fx-oss-process%3Dimage%2Fwatermark%2Ctype_ZmFuZ3poZW5naGVpdGk%2Cshadow_10%2Ctext_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTM0MjE2Mjk%3D%2Csize_16%2Ccolor_FFFFFF%2Ct_70
---



## 什么是FastAPI

FastAPI是一个用于构建API的现代web框架，基于python3.6，具有以下特性：

- **快速**：可与 **NodeJS** 和 **Go** 比肩的极高性能（归功于 Starlette 和 Pydantic）。
- **高效编码**：提高功能开发速度约 200％ 至 300％。
- **更少 bug**：减少约 40％ 的人为（开发者）导致错误。
- **智能**：极佳的编辑器支持。处处皆可自动补全，减少调试时间。
- **简单**：设计的易于使用和学习，阅读文档的时间更短。
- **简短**：使代码重复最小化。通过不同的参数声明实现丰富功能。bug 更少。
- **健壮**：生产可用级别的代码。还有自动生成的交互式文档。
- **标准化**：基于（并完全兼容）API 的相关开放标准：[OpenAPI](https://github.com/OAI/OpenAPI-Specification) (以前被称为 Swagger) 和 [JSON Schema](http://json-schema.org/)。



FastAPI基于如下的两个主要的模块：

- [Starlette](https://www.starlette.io/) 负责 web 部分。
- [Pydantic](https://pydantic-docs.helpmanual.io/) 负责数据部分。



<br>



## 安装FastAPI

FastAPI依赖于`python3.6及以上`版本，执行如下的命令进行安装：

```bash
$ pip install fastapi
```



启动FastAPI需要一个ASGI服务器，例如 uvicorn：

```bash
$ pip install uvicorn
```



> ASGI是WSGI的升级版，支持异步调用

<br>



## 示例代码

```python
# main.py
#!/usr/bin/env  python
# -*- coding: utf-8 -*-


from fastapi import FastAPI


app = FastAPI()


@app.get("/")
async def main():
    return {"message": "hello world"}


if __name__ == '__main__':
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```



<br>



## 使用模板引擎



### 安装模板引擎

FastAPI默认不带模板引擎，但是它支持任何模板引擎，例如`jinja2`，首先安装：

```bash
$ pip install jinja2 aiofiles
```



> aiofiles是提供静态文件用的



### 代码示例

```python
#!/usr/bin/env  python
# -*- coding: utf-8 -*-

from starlette.requests import Request
from fastapi import FastAPI
from starlette.templating import Jinja2Templates


app = FastAPI()
template = Jinja2Templates(directory="templates")


@app.get("/")
async def main(request: Request):
    return template.TemplateResponse('index.html', {'request': request,'hello': 'hello world'})


@app.get("/{item_id}/")
async def get_item(request: Request, item_id: int):
    return template.TemplateResponse('index.html', {'request': request,'item_id': item_id})


if __name__ == '__main__':
    import uvicorn
    uvicorn.run(app, host="127.0.0.1", port=8000)
```



`Jinja2Templates(directory="templates")`指定了静态模板文件的位置，这里设置为`templates`目录。

同时可以接受url参数传递，将参数传给对应的函数即可。



在视图函数中，返回一个`template.TemplateResponse`对象，其中包含：

- `index.html`：模板目录下的模板文件名；
- `'request': request`：固定的请求参数；
- `'hello': 'hello world'`：key为模板文件中的变量，value对应的是将要渲染的值；



模板文件简单示例 如下：

```html
<body>
    <h1>hello world</h1>
    <h1>{{ hello }}</h1>
    <h1>{{ item_id }}</h1>
</body>
```



> 注意这里的一个写法：`item_id: int`，他表示传入的参数`item_id`必须是int类型，否则会报错：`{"detail":[{"loc":["path","item_id"],"msg":"value is not a valid integer","type":"type_error.integer"}]}`，如果只是传入`item_id`，则是任何类型都可以





<br>



## 文档功能

FastAPI默认启动后会启动两个api文档地址：

- `http://127.0.0.1:8000/docs`：交互式api文档；
- `http://127.0.0.1:8000/rdoc`：redoc生成的文档；



这两个文档地址都是自动生成的，在前后端分离的项目调试的时候非常方便。



![](docs.png)

<br>

![](redoc.png)
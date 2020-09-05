---
title: Django使用Middleware记录日志
date: 2020-08-05 15:39:23
tags:
- Django
categories:
- python web开发
- Django
description: Django中通过自己编写middleware来实现请求日志记录
cover: https://ss2.bdstatic.com/70cFvnSh_Q1YnxGkpoWK1HF6hhy/it/u=316528160,103010658&fm=26&gp=0.jpg
---



# 默认日志

django在运行后默认会打印请求日志到标准输出，格式如下：

```bash
[05/Aug/2020 15:48:44] "POST /api/infrastructer/v1/devicetype/query HTTP/1.1" 200 480
```



但是这样的日志存在如下的问题：

1. 内容太少；
2. 不知道是哪个客户端请求；
3. 不能显示详细的报错内容；
4. 不能快速定位到报错的代码位置；
5. 不知道请求参数；
6. ......



针对这些问题，可以基于`logging`模块来定制日志内容。



<br>



# 基于middleware的日志输出

django的`middleware`是一个强大的功能，所有的请求都要通过中间件才能进行试图函数的处理，同理响应也会通过中间件。因此，可以手动编写一个日志的中间件来捕获请求和响应中的所需参数实现自定义日志输出。



## 日志中间件

首先新建一个`middleware`的目录，其中保存中间件代码。并在其中新建一个`middleware_logging.py`的中间件：

```python
import json
import socket
import threading
import logging

from django.utils.deprecation import MiddlewareMixin


local = threading.local()


class RequestLogFilter(logging.Filter):
    '''将当前请求的request信息保存到日志record上下文'''
    def filter(self, record):
        record.sip = getattr(local, 'sip', 'none')
        record.dip = getattr(local, 'dip', 'none')
        record.body = getattr(local, 'body', 'none')
        record.api = getattr(local, 'api', 'none')
        record.method = getattr(local, 'method', 'none')
        record.status_code = getattr(local, 'status_code', 'none')
        record.reason_phrase = getattr(local, 'reason_phrase', 'none')

        return True


class RequestLogMiddleware(MiddlewareMixin):
    '''将请求日志记录到当前请求线程上'''
    def __init__(self, get_response=None):
        self.get_response = get_response
        self.apiLogger = logging.getLogger('api')

    def __call__(self, request):
        try:
            body = json.loads(request.body)
        except Exception as e:
            body = dict()

        if request.method == 'GET':
            body.update(dict(request.GET))
        else:
            body.update(dict(request.POST))

        # 日志变量
        local.body = body
        local.method = request.method
        local.api = request.path
        local.sip = request.META.get('REMOTE_ADDR', '')
        local.dip = socket.gethostbyname(socket.gethostname())
        response = self.get_response(request)
        local.status_code = response.status_code
        local.reason_phrase = response.reason_phrase

        return response
```



在这段代码中有两个类：

- `RequestLogFilter`：用于日志过滤器（在logging配置时使用）；
- `RequestLogMiddleware`：日志中间件；



## 设置logging配置

在`setting.py`中新增如下的logging配置代码：

```python
LOGGING = {
    "version": 1,
    "disable_existing_loggers": True,
    "formatters": { 
        "stdout": { 
            "format": '{'
                '"level": "%(levelname)s", '
                '"date": "%(asctime)s", '
                '"method": "%(method)s", '
                '"api": "%(api)s", '
                '"body": "%(body)s", '
                '"filename": "%(pathname)s:%(lineno)d", '
                '"func": "%(funcName)s", '
                '"source": "%(sip)s", '
                '"dest": "%(dip)s", '
                '"message": "%(message)s"'
            '}',
            "datefmt": "%Y-%m-%d %H:%M:%S",
        }
    },
    "filters": {
        "request_info": {"()": "middleware.middleware_logging.RequestLogFilter"}

    },
    "handlers": { 
        "console": { 
            "level": "INFO",
            "class": "logging.StreamHandler",
            "formatter": "stdout",
            "filters": ['request_info']
        },
    },
    "root": {"level": "INFO", "handlers": ["console"]},
    "loggers": {
        "django.request": { 
          	"handlers": ["console"],
            "level": "INFO",
            "propagate": True, 
        },
    },
}
```

- `disable_existing_loggers`：是否禁用现有的日志记录器（设置为True的话只会记录当前设置的日志格式）；
- `formaters`：定义日志的样式，在这个示例中定义了一个名为`stdout`的日志格式；
  - `format`：日志格式的具体内容，我这里写成了json格式，也可以是任何格式；
  - `datefmt`：对日期进行格式化；
- `filters`：定义日志过滤器，这里定义了上边定义的日志过滤器，但是如果想使用，需要在下面进行指定；
- `handlers`：定义日志处理方式，支持输出到文件、输出到标准输出等多种方式；
  - `console`：指定将日志输出到标准输出；
  - `formatter`：指定用哪种日志格式输出日志；
  - `filters`：指定日志过滤器；
- `loggers`：定义哪些类型的日志会被记录；
  - `django.request`：表示所有request请求；
  - `handlers`：绑定使用的日志处理方式；



如果是想要输出日志到文件，可以在`handlers`中增加下面的配置：

```python
'file': { # Info级别以上保存到日志文件
		'level': 'INFO',
		'class': 'logging.handlers.RotatingFileHandler',  # 保存到文件，根据文件大小自动切
		'filename': os.path.join(LOG_DIR,"info.log"),  # 日志文件
		'maxBytes': 1024 * 1024 * 10,  # 日志大小 10M
		'backupCount': 2,  # 备份数为 2
		'formatter': 'stdout', # 简单格式
		'encoding': 'utf-8',
},
```



> 注意设置`TIME_ZONE`为`Asia/Shanghai`



`formatters`支持多种内置的变量，可以在 [日志变量](https://docs.python.org/3/library/logging.html#logrecord-attributes)中查看到。



## 日志打印

需要在视图函数中的合适位置新增如下的代码：

```python
import logging as log

# info日志
log.info('ifno log')

# warning日志
log.warning('warning log')

# error日志
log.error('error log')
```



这样在程序出现问题时就可以打印日志了，格式如下：

```bash
{"level": "INFO", "date": "2020-08-05 16:34:32", "method": "POST", "api": "/api/infrastructer/v1/devicetype/query", "body": "{'page': 1, 'limit': 2}", "filename": "/Users/lee/Desktop/ipaas/infrastructer/api/devicetpye/query_devicetype.py:28", "func": "v1_query_devicetype", "source": "127.0.0.1", "dest": "127.0.0.1", "message": "query devicetype success."}
```



<br>



# 日志输出的流程

1. 首先会加载`setting.py`中的日志配置，并加载`RequestLogFilter`日志过滤器；
2. 之后请求来到`RequestLogMiddleware`中间件中，它会获取请求中的一些参数并保存到日志对象中；
3. 当程序执行到`log.info`这样的打印日志语句时，会根据日志配置打印出日志内容。



<br>
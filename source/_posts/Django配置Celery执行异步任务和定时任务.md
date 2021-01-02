---
title: Django配置Celery执行异步任务和定时任务
date: 2021-01-02 12:18:33
tags:
- Django
categories:
- python web开发
- Django
description: Django配置Celery执行异步任务和定时任务
cover: https://gimg2.baidu.com/image_search/src=http%3A%2F%2F5b0988e595225.cdn.sohucs.com%2Fimages%2F20171204%2F9d4fde3a60ee4b0c9d3b3a5ea336df18.jpeg&refer=http%3A%2F%2F5b0988e595225.cdn.sohucs.com&app=2002&size=f9999,10000&q=a80&n=0&g=0n&fmt=jpeg?sec=1612153214&t=e554f5ad19844d6ba41ea1d4f42236af
---



## celery介绍

### 简介

celery是一个基于python开发的简单、灵活且可靠的分布式任务队列框架，支持使用任务队列的方式在分布式的机器/进程/线程上执行任务调度。采用典型的生产者-消费者模型，主要由三部分组成：

- `消息队列broker`：实际上是一个MQ队列服务，可以使用redis、rabbitMQ等作为broker；
- `任务消费者worker`：broker通知worker队列中存在任务，worker去队列中取出任务然后执行，每个worker是一个进程；
- `结果存储backend`：执行结果存储在backend中，默认也会存储在broker使用的MQ队列服务中，也可以单独配置用何种服务做backend；



![](./celery.png)



### celery 还是 django-celery

Django本身不支持异步，要想实现异步的话借助Celery是个不错的选择，但是django集成celery配置起来比较复杂，且不支持动态添加定时任务。



django-celery提供了celery集成，同时支持将结果存储在django的orm或cache中，最重要的是支持从数据库动态读取计划任务并执行，这也就是说我们只需要将需要执行的加护任务插入数据库，django-celery就可以自动发现并执行了；



### 使用场景

如果你的程序后台处理逻辑需要一段时间，或者你需要定时的执行某个方法（如定期发送报告），那么使用celery或者django-celery便非常有必要。



如：前端接收请求后会发送到后端，后端执行需要一段时间（10分钟）；后端在接收到请求后放入队列异步执行，同时立刻返回给前端一个`任务正在执行`的结果。



<br>



## django集成celery -- 异步任务

### 安装rabbitmq

这里使用rabbitmq作为消息队列，也可以使用其他类型的消息队列，rabbitmq采用最简单的安装方式，但是生产环境一般会进行一些配置；



首先添加erlang仓库并安装erlang：

```bash
# 添加erlang仓库
$ wget http://packages.erlang-solutions.com/erlang-solutions-1.0-1.noarch.rpm
$ rpm -Uvh erlang-solutions-1.0-1.noarch.rpm
$ yum -y install epel-release

# 安装erlang
$ yum -y install erlang
```



验证erlang是否安装成功可以执行下面的命令，如果输出erlang的版本号则表示安装成功：

```bash
$ erl -v
```



安装依赖socat：

```bash
# 安装socat
$ yum -y install socat
```



安装rabbitmq：

```bash
# 添加仓库
$ cat > /etc/yum.repos.d/rabbitmq.repo << EOF
[bintray-rabbitmq-server]
name=bintray-rabbitmq-rpm
baseurl=https://dl.bintray.com/rabbitmq/rpm/rabbitmq-server/v3.8.x/el/7/
gpgcheck=0
repo_gpgcheck=0
enabled=1
EOF

# 下载key
$ rpm --import https://www.rabbitmq.com/rabbitmq-release-signing-key.asc
# 或者
$ rpm --import https://github.com/rabbitmq/signing-keys/releases/download/2.0/rabbitmq-release-signing-key.asc

# 安装
$ wget https://github.com/rabbitmq/rabbitmq-server/releases/download/v3.8.3/rabbitmq-server-3.8.3-1.el7.noarch.rpm
$ yum -y install rabbitmq-server-3.8.3-1.el7.noarch.rpm
```



设置并启动rabbitmq：

```bash
$ cat > /etc/rabbitmq/rabbitmq-env.conf << EOF
NODENAME=rabbit@localhost
EOF

$ systemctl start rabbitmq-server
```



验证rabbitmq是否启动成功则可以执行：

```bash
$ rabbitmqctl status
```



### 安装celery

```bash
$ pip install celery
```



### django项目结构

![](./django.png)



### celery.py文件

```python
from __future__ import absolute_import, unicode_literals
import os
from celery import Celery, platforms

# set the default Django settings module for the 'celery' program.
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'website.settings')

# 配置rabbitmq，注意地址
app = Celery('website', backend='amqp', broker='amqp://admin:admin@localhost')

# Using a string here means the worker don't have to serialize
# the configuration object to child processes.
# - namespace='CELERY' means all celery-related configuration keys
#   should have a `CELERY_` prefix.
app.config_from_object('django.conf:settings', namespace='CELERY')

# Load task modules from all registered Django app configs.
app.autodiscover_tasks()

# 允许root 用户运行celery
platforms.C_FORCE_ROOT = True

# 防止长期运行内存泄露
CELERYD_MAX_TASKS_PER_CHILD = 10

@app.task(bind=True)
def debug_task(self):
    print('Request: {0!r}'.format(self.request))
```



### website/__init__.py文件

在`website/__init__.py`文件中增加如下内容，确保django启动的时候这个app能够被加载到：

```python
from __future__ import absolute_import

# This will make sure the app is always imported when
# Django starts so that shared_task will use this app.
from .celery import app as celery_app

__all__ = ['celery_app']
```



### 创建task文件

在各个应用下创建`tasks.py`文件，这里是`deploy/tasks.py`：

```python
from __future__ import absolute_import
from celery import shared_task

@shared_task
def add(x, y):
    return x + y
```



> **注意tasks.py必须建在各app的根目录下，且只能叫tasks.py，不能随意命名**



### 引用任务异步处理

在`views.py`中引用这个任务进行异步处理：

```python
from deploy.tasks import add

def post(request):
    result = add.delay(2, 3)
```



- `<函数名>.deploy()`表示启动异步任务；
- `result.ready()`可以判断任务是否结束执行；
- 如果任务抛出一个异常，使用`result.get(timeout=1)`可以重新抛出异常；
- 如果任务抛出一个异常，使用`result.traceback`可以获取原始的回溯信息；



### 启动celery

```bash
$ celery -A website worker -l info
```



这样在调用`post`这个方法的时候就可以异步执行里边的`add`了。



<br>



## django集成celery -- 定时任务

### celery.py

`website/celery.py`文件添加如下配置以支持定时任务crontab：

```python
from celery.schedules import crontab
from datetime import timedelta

app.conf.update(
    CELERYBEAT_SCHEDULE = {
        'sum-task': {
            'task': 'deploy.tasks.add',
            'schedule':  timedelta(seconds=20),
            'args': (5, 6)
        }
        'send-report': {
            'task': 'deploy.tasks.report',
            'schedule': crontab(hour=4, minute=30, day_of_week=1),
        }
    }
)
```



- 定义了两个定时任务：`sum-task`和`send-report`；
- `sum-task`每隔20分钟执行一次`add`函数，并传入参数5和6；
- `send-report`每周一的4：30分执行一次`report`函数；
- `timedelta`是`datetime`中的一个对象，需要`from datetime import timedelta`引入，有如下几个参数:
  - `days`：天
  - `seconds`：秒
  - `microseconds`：微妙
  - `milliseconds`：毫秒
  - `minutes`：分
  - `hours`：小时
- crontab的参数有：
  - `month_of_year`：月份
  - `day_of_month`：日期
  - `day_of_week`：周
  - `hour`：小时
  - `minute`：分钟



### 启动celery beat

celery启动了一个beat进程一直在不断的判断是否有任务需要执行：

```bash
$ celery -A website beat -l info
```



## django集成celery -- 小技巧

### 同时启动异步任务和定时任务

如果你同时使用了异步任务和计划任务，可以执行下面的命令同时启动worker和beat：

```bash
$ celery -A website worker -b -l info
```



### 使用的非rabbitmq队列

如果使用的非rabbitmq队列，如redis，则需要在`website/celery.py`中配置broker和backend：

```python
# redis做MQ配置
app = Celery('website', backend='redis', broker='redis://localhost')

# rabbitmq做MQ配置
app = Celery('website', backend='amqp', broker='amqp://admin:admin@localhost')
```



### 以root方式运行

默认不能以root方式运行celery，除非在`website/celery.py`中添加下面的配置：

```python
platforms.C_FORCE_ROOT = True
```



### 防止内存泄露

celery在长时间运行后可能出现内存泄漏，需要添加配置`CELERYD_MAX_TASKS_PER_CHILD = 10`，表示每个worker执行了多少个任务就死掉



<br>



## django集成django-celery

### 安装

```bash
$ pip install django-celery
```



安装完成后在`settings.py`中的`INSTALLED_APPS`下加入`django-celery`：

```python
INSTALLED_APPS = [
    ...
    'djcelery',
]
```



### 修改配置文件

在`settings.py`中增加如下的配置：

```python
import djcelery
djcelery.setup_loader()

# BROKER和BACKEND配置，这里用了本地的redis，其中1和2表示分别用redis的第一个和第二个db
BROKER_URL = 'redis://localhost/1'
CELERY_RESULT_BACKEND = 'redis://localhost/2'

# celery 关闭UTC时区
CELERY_ENABLE_UTC = False

# celery 并发数设置，最多可以有20个任务同时运行
CELERYD_CONCURRENCY = 20
CELERYD_MAX_TASKS_PER_CHILD = 4

# celery开启数据库调度器，数据库修改后即时生效
CELERYBEAT_SCHEDULER = 'djcelery.schedulers.DatabaseScheduler'

from celery import platforms

# 允许root 用户运行celery
platforms.C_FORCE_ROOT = True
```



### 启动celery

```bash
$ python manage.py celery worker -l info
```



### 异步任务

项目结构为：

```python
project
    - testapp
        - __init__.py
        - admin.py
        - app.py
        - models.py
        - tasks.py
        - tests.py
        - views.py
    - webapp
        - __init__.py
        - settings.py
        - urls.py
        - wsgi.py
    - manage.py
```



> 在app根目录下添加一个`tasks.py`文件，在文件中编写函数，给函数添加上`shared_task`装饰器即可



`tasks.py`文件如下内容：

```python
from celery import shared_task


@shared_task
def welcome():
    print('Welcome to my site')

    return True
```



然后在其他视图函数中就可以异步调用这个函数了：

```python
from django.http import JsonResponse
from testapp.tasks import welcome


def index(request):
    welcome.delay()

    return JsonResponse({"state": 1, "message": "welcome"})
```



### 定时任务

首先需要执行`migrate`命令在数据库创建表

```bash
$ python manage.py migrate
```



然后修改`settings.py`文件中添加`CELERYBEAT_SCHEDULE`配置

```python
CELERYBEAT_SCHEDULE = {
    'testapp-1': {
        'task': 'testapp.tasks.welcome',
        'schedule': timedelta(seconds=20)
    },
    'testapp-2': {
        'task': 'testapp.tasks.welcome',
        'schedule': crontab(hour=17, minute=30),
    }
}
```



然后启动beat：

```bash
$ python manage.py celery beat -l info
```





### 报错处理

启动beat时可能会有如下报错

```bash
TypeError: can't subtract offset-naive and offset-aware datetimes
```



这主要是因为时区引起的，请修改时区相关的配置

```python
TIME_ZONE = 'Asia/Shanghai'
USE_TZ = False
```



<br>



## django集成django-celery之动态添加

经过上边的操作会有生成下面的几张表：

![](./table.png)



其中：

- `djcelery_crontabschedule`：计划任务时间定义表；
- `djcelery_crontabschedule`：循环任务时间定义表；
- `djcelery_periodictask`：及任务表；



下面是别人的一个例子：

```python
from djcelery.models import CrontabSchedule, PeriodicTask

with transaction.atomic():
    save_id = transaction.savepoint()

    try:
        _c, created = CrontabSchedule.objects.get_or_create(
            minute=str(minute),
            hour=str(hour),
            day_of_week=str(day_of_week),
            day_of_month=str(day_of_month),
            month_of_year=str(month_of_year)
        )

        dt = datetime.now().strftime('%Y%m%d%H%M%S')
        _p = PeriodicTask.objects.create(
            name=dt + '-' + '周期任务A',
            task='testapp.tasks.welcome',
            args=[37],
            enabled=True,
            crontab=_c
        )

        print('计划任务添加成功')
    except Exception as e:
        transaction.savepoint_rollback(save_id)
        print('添加计划任务失败，错误原因：' + str(e))
```



通过`with transaction.atomic()`创建一个事物，保证时间和任务都能同时添加成功，否则就回滚

然后通过`get_or_create`方法去检索循环任务时间定义表`CrontabSchedule`，如果有就获取到实例，没有就创建

最后往任务表`PeriodicTask`里插入任务，`name`为任务名称，具有唯一性，所以这里加了时间前缀防止重复，`task`为celery的task任务，字符串类型，在启动celery的时候就可以看到，`args`传给任务的参数，这里也可以用kwargs的字典形式传，就把`args`字段改成`kwargs`即可，`enabled`定义了这个任务是启动或关闭状态，`crontab`为循环任务时间实例，如果这里要用周期任务，就是每n秒n分循环执行这样的，只需要将`crontab`关联换成`interval`即可，那就需要事先往`IntervalSchedule`表里插入数据

还记得开头settings.py配置文件中我们配置的`CELERYBEAT_SCHEDULER`吗？就因为有这个配置，所以当数据表里的数据变更之后，celery的beat程序就能监听到从而在配置的时间触发worker去执行任务

至此，主要功能我们都已实现，django-celery的计划任务只能支持固定时间吗？其实不然，他支持的语法与linux下的crontab类似，像`hour='*/3,9-18'`这样的复杂语法也是支持的
---
title: Django分页查询
date: 2020-07-31 13:44:44
tags:
- Django
categories:
- python web开发
- Django
description: Django分页查询
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1596184483878&di=c347e6459c1c3d225c06840cd19f59b9&imgtype=0&src=http%3A%2F%2F5b0988e595225.cdn.sohucs.com%2Fimages%2F20171204%2F9d4fde3a60ee4b0c9d3b3a5ea336df18.jpeg
---



在django做查询接口的时候往往需要做分页查询，根据前端传过来的页码和每页行数来进行查询。



在后端使用django 自带的分页器进行分页：

```python
import json
from django.http import JsonResponse
from django.core.paginator import Paginator

from infrastructer import models
from ipaas.utils import DeviceTypeSerializer


def v1_query_devicetype(request):
    if request.method != 'POST':
        err_msg = "Error http method {}".format(request.method)
        return JsonResponse({"status": 1, "msg": err_msg})

    page = int(json.loads(request.body)['page'])
    page_size = int(json.loads(request.body)['limit'])

    try:
        all_data = models.DeviceType.objects.filter(is_delete=0)
        paginator = Paginator(all_data, page_size)
        query_data = paginator.get_page(page)
        devicetype_data = [DeviceTypeSerializer(i).data for i in query_data]
        ok_msg = "query devicetype success."
        return JsonResponse({"status": 0, "msg": ok_msg, "total": len(all_data), "data": devicetype_data})
    except Exception as e:
        err_msg = "query devicetype failed, {}".format(e)
        return JsonResponse({"status": 1, "msg": err_msg, "data": []})
```



1. 首先判断请求是否为`POST`请求；
2. 要求前端传入`page`：页码，`limit`：每页的行数；
3. 获取到全部的数据，通过分页器根据每页行数进行划分；
4. 使用`get_page`方法根据页码获取该页的数据；
5. 使用序列化工具进行数据序列化，返回给前端；


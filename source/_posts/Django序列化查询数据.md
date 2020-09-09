---
title: Django序列化查询数据
date: 2020-09-09 09:48:40
tags:
- Django
categories:
- python web开发
- Django
description: 使用Django rest_framework序列化ORM查询的数据
cover: https://ss3.bdstatic.com/70cFv8Sh_Q1YnxGkpoWK1HF6hhy/it/u=633567869,609217672&fm=26&gp=0.jpg
---



# 需求

通过ORM查询出来的数据类型是`queryset`，一般需要进行序列化操作来将其变成字典类型数据进行进一步处理。



<br>



# 准备

需要安装`rest_framework`：

```bash
$ pip install djangorestframework
```



<br>



# 代码

首先创建一个`serializers.py`的文件，保存序列化相关的规则：

```python
#!/usr/bin/env  python
# -*- coding: utf-8 -*-

from rest_framework import serializers

from queryapi.models import QuerySegment


class QuerySegmentSerializer(serializers.ModelSerializer):
    class Meta:
        model = QuerySegment
        fields = ("ip", "country", "province", "city", "region", "front_isp", "backbone_isp", "asid", "comment")

    def to_representation(self, instance):
        data = super().to_representation(instance)
        # data为序列化后的结果，可以在这里对结果进行进一步处理
        return data
```



> 其中`fields`是表结构中的需要进行序列化的字段



在查询数据的时候使用序列化：

```python
from queryapi.serializers import QuerySegmentSerializer

query_data = models.QuerySegment.objects.filter(
        ip__in=args['ipList'],
        country__in=args['countryList'],
        province__in=args['provinceList'],
        city__in=args['cityList'],
        region__in=args['regionList'],
        front_isp__in=args['frontIspList'],
        backbone_isp__in=args['backboneIspList'],
        asid__in=args['asIdList']
    )

dbdata = [QuerySegmentSerializer(i).data for i in query_data]
```



这样就可以获得一个列表类型的查询结果数据集合，列表中的元素都是字典类型。
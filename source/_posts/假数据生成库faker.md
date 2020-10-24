---
title: 假数据生成库faker
date: 2020-10-24 16:24:40
tags:
- Python
categories:
- Python
- 常用的库
description: 利用faker生成测试用的假数据
cover:
---



在开发测试阶段，经常会用到一些测试数据，这时候可以通过`faker`库进行生成，它能够生成：

- 文本
- 电话号码
- 街道地址
- IP地址
- 名字
- ......



<br>



## 安装

直接使用`pip`进行安装即可：

```bash
$ pip install Faker
```



<br>



## 使用



### 导入并指定语种

```python
>>> from faker import Faker
>>> fake = Faker(locale='zh-CN')
```



`faker`默认的语种为英语`en_US`，指定对应的语种，才能生成对应的测试数据信息，常用的选项为：

- `zh_CN`：中文；
- `en_US`：英文；



### 常用地理信息

```python
# 国家编码
>>> fake.country_code()

# 国家
>>> fake.country()

# 省份
>>> fake.province()

# 市、县
>>> fake.city_suffix()

# 区
>>> fake.district()

# 街、路
>>> fake.street_suffix()

# 街道名
>>> fake.street_name()

# 街道地址
>>> fake.street_address()

# 详细地址
>>> fake.address()

# 邮编
>>> fake.postcode()

# 地理坐标
>>> fake.geo_coordinate()

# 地理坐标（维度）
>>> fake.latitude()

# 地理坐标(经度)
>>> fake.longitude()
```



###  常用个人信息

```python
# 名字
>>> fake.name()

# 男性全名
>>> fake.name_female()

# 女性全名
>>> fake.name_male()

# 随机生成手机号
>>> fake.phone_number()

# 身份证号
>>> fake.ssn()

# 随机公司名（长）
>>> fake.company()

# 随机公司名（短）
>>> fake.company_prefix()

# 随机职位
>>> fake.job()

# 公司邮箱
>>> fake.company_email()
```



### 网络基础信息

```python
# 生成域名
>>> fake.domain_name()

# 随机url地址
>>> fake.url()

# 随机IP4地址
>>> fake.ipv4()

# 随机IP6地址
>>> fake.ipv6()

# 随机MAC地址
>>> fake.mac_address()

# 随机URI地址
>>> fake.uri()

# 随机用户名
>>> fake.user_name()
```





### 数字信息

```python
# 三位随机数字
>>> fake.numerify()

# 0~9随机数
>>> fake.random_digit()

# 1~9的随机数
>>> fake.random_digit_not_null()

# 随机数字，默认0~9999，可以通过设置min,max来设置
>>> fake.random_int()

# 随机数字，参数digits设置生成的数字位数
>>> fake.random_number()

# 随机Float数字
>>> fake.pyfloat()

# 随机Int数字（参考random_int()参数）
>>> fake.pyint()

# 随机Decimal数字（参考pyfloat参数）
>>> fake.pydecimal()
```


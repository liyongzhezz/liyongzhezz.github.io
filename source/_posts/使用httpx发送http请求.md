---
title: 使用httpx发送http请求
date: 2020-11-01 14:05:13
tags:
- Python
categories:
- Python
- 常用的库
description: 使用httpx发送http请求
cover: https://ss0.bdstatic.com/70cFvHSh_Q1YnxGkpoWK1HF6hhy/it/u=3393331488,2909864782&fm=26&gp=0.jpg
---



httpx基于异步框架，性能优于requests。



## 安装

```bash
$ pip install httpx
```



<br>



## 发送get请求

```python
import httpx

params = {"key1": "value2", "key2": ["value2", "value3"]}
headers = {"user-agent": "my-app/0.0.1"}

r = httpx.get("https://www.example.com", params=params, headers=headers)
```



<br>



## 发送post请求

```python
import httpx

headers = {"user-agent": "my-app/0.0.1"}
data={"key": "value"}

r = httpx.post("https://www.example.com", data=data, headers=headers)

# 发送json编码数据
r = httpx.post("https://www.example.com", json=data)
```



<br>



## 处理响应结果

```python
# 打印请求url
r.url

# 响应内容
r.text

# 设置使用的编码
r.encoding = "ISO-8859-1"

# 响应头
r.headers

# 响应http版本
r.http_version

# 响应状态码
r.status_code

# 获取json内容
r.json()
```



<br>



## cookies

```python
# 获取cookies
>>> r = httpx.get("http://httpbin.org/cookies/set?chocolate=chip", allow_redirects=False)
>>> r.cookies["chocolate"]
'chip'

# 请求带cookie
>>> cookies = {"peanut": "butter"}
>>> r = httpx.get("http://httpbin.org/cookies", cookies=cookies)
>>> r.json()
{'cookies': {'peanut': 'butter'}}
```



cookies也可以按域进行访问设置：

```python
>>> cookies = httpx.Cookies()
>>> cookies.set('cookie_on_domain', 'hello, there!', domain='httpbin.org')
>>> cookies.set('cookies_off_domain', 'nope', domain="example.org")
>>> r = httpx.get("http://httpbin.org/cookies", cookies=cookies)
>>> r.json()
{'cookies': {'cookie_on_domain': 'hello, there!'}}
```



<br>



## 用户身份认证

基本认证：

```python
httpx.get("https://example.com", auth=("my_user", "password123"))
```



摘要式身份认证：

```python
auth = httpx.DigestAuth("my_user", "password123")
httpx.get("https://example.com", auth=auth)
```


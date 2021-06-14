---
title: 使用Request库发送http请求
date: 2021-06-14 13:14:06
tags:
- Python
categories:
- 编程
- Python
- 网络
description: pythn request库的使用
cover: https://ss3.bdstatic.com/70cFv8Sh_Q1YnxGkpoWK1HF6hhy/it/u=2100224046,747583905&fm=26&gp=0.jpg
---



{% note info 'fas fa-bullhorn' %}

本文主要介绍了在python中使用request发送http请求

更新于 2021-06-14

{% endnote %}

<br>



# 安装

使用下面的命令安装requests：

```bash
$ pip install requests
```





<br>



# 发送GET请求

```python
import requests

response = requests.get("http://www.baidu.com/")
```



同时支持添加header参数：

```python
import requests

kw = {'wd':'中国'}

headers = {"User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/54.0.2840.99 Safari/537.36"}

# params 接收一个字典或者字符串的查询参数，字典类型自动转换为url编码，不需要urlencode()
response = requests.get("http://www.baidu.com/s", params=kw, headers=headers)
```



获取响应信息可以使用下面的命令：

```python
# 查看响应内容，response.text 返回的是Unicode格式的数据
response.text

# 查看响应内容，response.content返回的字节流数据
response.content

# 查看完整url地址
response.url

# 查看响应头部字符编码
response.encoding

# 查看响应码
response.status_code
```



<br>



# 发送POST请求

```python
import requests

response = requests.post("http://www.baidu.com/",data=data)
```



传入post数据可以使用如下的格式：

```python
import requests

url = "https://www.lagou.com/jobs/positionAjax.json?city=%E6%B7%B1%E5%9C%B3&needAddtionalResult=false&isSchoolJob=0"

headers = {
  'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/62.0.3202.94 Safari/537.36',
  'Referer': 'https://www.lagou.com/jobs/list_python?labelWords=&fromSearch=true&suginput='
 }

 data = {
     'first': 'true',
     'pn': 1,
     'kd': 'python'
 }

 resp = requests.post(url,headers=headers,data=data)
 # 如果是json数据，直接可以调用json方法
 print(resp.json())
```



<br>



# 通过代理

使用requests添加代理也非常简单，只要在请求的方法中（比如get或者post）传递proxies参数就可以了。例如：

```python
import requests

url = "http://httpbin.org/get"

headers = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/62.0.3202.94 Safari/537.36',
}

proxy = {
    'http': '171.14.209.180:27829'
}

resp = requests.get(url, headers=headers, proxies=proxy)
with open('xx.html','w',encoding='utf-8') as fp:
    fp.write(resp.text)
```



<br>



# cookie和session

如果在一个响应中包含了cookie，那么可以利用cookies属性拿到这个返回的cookie值：

```python
import requests

url = "http://www.renren.com/PLogin.do"
data = {"email":"970138074@qq.com",'password':"pythonspider"}
resp = requests.get('http://www.baidu.com/')
print(resp.cookies)
print(resp.cookies.get_dict())
```



使用requests库给我们提供的session对象可以达到共享cookie的目的。这里的session不是web开发中的那个session，这个地方只是一个会话的对象而已。例如：

```python
import requests

url = "http://www.renren.com/PLogin.do"
data = {"email":"111111111@qq.com",'password':"pythonspider"}
headers = {
    'User-Agent': "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/62.0.3202.94 Safari/537.36"
}
# 登录
session = requests.session()
session.post(url,data=data,headers=headers)

# 访问个人中心
resp = session.get('http://www.renren.com/880151247/profile')

print(resp.text)
```



<br>



# 不受信的证书

对于不受信任的https站点，可以使用如下的方式进行访问：

```python
import requests

resp = requests.get('http://www.12306.cn/mormhweb/',verify=False)
print(resp.content.decode('utf-8'))
```






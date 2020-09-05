---
title: 解决if语句中不能设置proxy_set_header的问题
date: 2020-08-06 17:02:10
tags:
- Nginx
categories:
- Nginx
- 常见问题
description: 实现在if语句中设置proxy_set_header的问题
cover: 
---



nginx支持使用`if`语句进行判断，例如：

```nginx
location / {
  if ($http_referer ~ "http[s]?\:\/\/\w*\.?example.*") {
    # 执行语句
    proxy_set_header Host "test.pass";
  }
  proxy_pass http://proxy-server;
}
```



在上边的例子中，判断域名中匹配到example，就增加一个header头`test.pass`，但是这样会报错：

```bash
"proxy_set_header" directive is not allowed here in /etc/nginx/server.conf:10
```



**nginx不支持在if中设置proxy_set_header**



对于这种问题，可以使用一种变通的方法，通过设置中间变量来解决：

```nginx
location / {
  set $tmphost $host;
  if ($http_referer ~ "http[s]?\:\/\/\w*\.?example.*") {
    # 执行语句
    set $tmphost "test.pass";
  }
  
  proxy_set_header Host $tmphost;
  proxy_pass http://proxy-server;
}
```



这里通过设置了一个变量`tmphost`，使用`set`语句在`if`语句中对其重新赋值，并将`tmphost`设置到`proxy_set_header`中即可。






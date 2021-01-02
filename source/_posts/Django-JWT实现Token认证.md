---
title: Django JWT实现Token认证
date: 2021-01-02 13:36:22
tags:
- Django
categories:
- python web开发
- Django
description: Django JWT实现Token认证
cover: https://ss0.bdstatic.com/70cFuHSh_Q1YnxGkpoWK1HF6hhy/it/u=2309570590,744397428&fm=26&gp=0.jpg
---



## token认证方式

### token认证方式介绍

客户端用自己的账号密码进行登录，服务端验证鉴权，验证鉴权通过生成Token返回给客户端，之后客户端每次请求都将Token放在header里一并发送，服务端收到请求时校验Token以确定访问者身份。



Token的主要目的是为了鉴权，同时又不需要考虑CSRF防护以及跨域的问题，所以更多的用在专门给第三方提供API的情况下，客户端请求无论是浏览器发起还是其他的程序发起都能很好的支持。所以目前基于Token的鉴权机制几乎已经成了前后端分离架构或者对外提供API访问的鉴权标准，得到广泛使用；



### 在django中的实现

django中也可以使用JSON Web Token（JWT)，这个是目前Token鉴权机制下最流行的方案；



<br>



## PyJWT使用

PyJWT 是python中对于JWT的实现。



###  安装

```bash
$ pip install pyjwt
```



### 生成token

```python
>>> import jwt
>>> encoded_jwt = jwt.encode({'username':'lee','site':'https://mysite.cn'},'secret_key',algorithm='HS256')
>>> print(encoded_jwt)
b'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VybmFtZSI6Ilx1OGZkMFx1N2VmNFx1NTQ5Nlx1NTU2MVx1NTQyNyIsInNpdGUiOiJodHRwczovL29wcy1jb2ZmZWUuY24ifQ.fIpSXy476r9F9i7GhdYFNkd-2Ndz8uKLgJPcd84BkJ4'
```



生成token的信息有三部分：

- 第一部分是一个Json对象，称为Payload，主要用来存放有效的信息，例如用户名，过期时间等等所有你想要传递的信息；
- 第二部分是一个秘钥字串，这个秘钥主要用在下文Signature签名中，服务端用来校验Token合法性，这个秘钥只有服务端知道，不能泄露；
- 第三部分指定了Signature签名的算法；



token格式通过两个`.`分成了三段，分别对应Header头部、playload负载和signature签名，其中第一部分Header和第二部分Payload只是对原始输入的信息转成了base64编码，第三部分Signature是用header+payload+secret_key进行加密的结果，可以直接用base64对Header和Payload进行解码得到相应的信息：

```python
>>> import base64
>>> base64.b64decode('eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9')
b'{"typ":"JWT","alg":"HS256"}'

>>> base64.b64decode('eyJ1c2VybmFtZSI6Ilx1OGZkMFx1N2VmNFx1NTQ5Nlx1NTU2MVx1NTQyNyIsInNpdGUiOiJodHRwczovL29wcy1jb2ZmZWUuY24ifQ==')
# 这里最后加=的原因是base64解码对传入的参数长度不是2的对象，需要再参数最后加上一个或两个等号=
```

> 因为JWT不会对结果进行加密，所以不要保存敏感信息在Header或者Payload中，服务端也主要依靠最后的Signature来验证Token是否有效以及有无被篡改



### 解密token

```python
>>> jwt.decode(encoded_jwt,'secret_key',algorithms=['HS256'])
{'username':'lee','site':'https://mysite.cn'}
```

> 服务端在有秘钥的情况下可以直接对JWT生成的Token进行解密，解密成功说明Token正确，且数据没有被篡改





JWT并没有对数据进行加密，如果没有`secret_key`也可以直接获取到Payload里边的数据，只是缺少了签名算法无法验证数据是否准确，pyjwt也提供了直接获取Payload数据的方法，如下

```python
>>> jwt.decode(encoded_jwt, verify=False)
{'username':'lee','site':'https://mysite.cn'}
```



<br>



## django实例

我们可以自己实现一个装饰器，检查用户的认证模式，同时认证完成后验证用户是否有权限操作



```python
from django.conf import settings
from django.http import JsonResponse
from django.contrib.auth import get_user_model
from django.core.exceptions import PermissionDenied

UserModel = get_user_model()


def auth_permission_required(perm):
    def decorator(view_func):
        def _wrapped_view(request, *args, **kwargs):
            # 格式化权限
            perms = (perm,) if isinstance(perm, str) else perm

            if request.user.is_authenticated:
                # 正常登录用户判断是否有权限
                if not request.user.has_perms(perms):
                    raise PermissionDenied
            else:
                try:
                    auth = request.META.get('HTTP_AUTHORIZATION').split()
                except AttributeError:
                    return JsonResponse({"code": 401, "message": "No authenticate header"})

                # 用户通过API获取数据验证流程
                if auth[0].lower() == 'token':
                    try:
                        dict = jwt.decode(auth[1], settings.SECRET_KEY, algorithms=['HS256'])
                        username = dict.get('data').get('username')
                    except jwt.ExpiredSignatureError:
                        return JsonResponse({"status_code": 401, "message": "Token expired"})
                    except jwt.InvalidTokenError:
                        return JsonResponse({"status_code": 401, "message": "Invalid token"})
                    except Exception as e:
                        return JsonResponse({"status_code": 401, "message": "Can not get user object"})

                    try:
                        user = UserModel.objects.get(username=username)
                    except UserModel.DoesNotExist:
                        return JsonResponse({"status_code": 401, "message": "User Does not exist"})

                    if not user.is_active:
                        return JsonResponse({"status_code": 401, "message": "User inactive or deleted"})

                    # Token登录的用户判断是否有权限
                    if not user.has_perms(perms):
                        return JsonResponse({"status_code": 403, "message": "PermissionDenied"})
                else:
                    return JsonResponse({"status_code": 401, "message": "Not support auth type"})

            return view_func(request, *args, **kwargs)

        return _wrapped_view

    return decorator
```



在view使用时就可以用这个装饰器来代替原本的`login_required`和`permission_required`装饰器了

```python
@auth_permission_required('account.select_user')
def user(request):
    if request.method == 'GET':
        _jsondata = {
            "user": "lee",
            "site": "https://mysite.cn"
        }

        return JsonResponse({"state": 1, "message": _jsondata})
    else:
        return JsonResponse({"state": 0, "message": "Request method 'POST' not supported"})
```



我们还需要一个生成用户Token的方法，通过给User model添加一个token的静态方法来处理

```python
class User(AbstractBaseUser, PermissionsMixin):
    create_time = models.DateTimeField(auto_now_add=True, verbose_name='创建时间')
    update_time = models.DateTimeField(auto_now=True, verbose_name='更新时间')
    username = models.EmailField(max_length=255, unique=True, verbose_name='用户名')
    fullname = models.CharField(max_length=64, null=True, verbose_name='中文名')
    phonenumber = models.CharField(max_length=16, null=True, unique=True, verbose_name='电话')
    is_active = models.BooleanField(default=True, verbose_name='激活状态')

    objects = UserManager()

    USERNAME_FIELD = 'username'
    REQUIRED_FIELDS = []

    def __str__(self):
        return self.username

    @property
    def token(self):
        return self._generate_jwt_token()

    def _generate_jwt_token(self):
        token = jwt.encode({
            'exp': datetime.utcnow() + timedelta(days=1),
            'iat': datetime.utcnow(),
            'data': {
                'username': self.username
            }
        }, settings.SECRET_KEY, algorithm='HS256')

        return token.decode('utf-8')

    class Meta:
        default_permissions = ()

        permissions = (
            ("select_user", "查看用户"),
            ("change_user", "修改用户"),
            ("delete_user", "删除用户"),
        )
```



可以直接通过用户对象来生成Token：

```python
>>> from accounts.models import User
>>> u = User.objects.get(username='lee@mysite.cn')
>>> u.token
'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE1NDgyMjg3NzksImlhdCI6MTU0ODE0MjM3OSwiZGF0YSI6eyJ1c2VybmFtZSI6ImFkbWluQDE2My5jb20ifX0.akZNU7t_z2kwPxDJjmc-QxtNdICK0yhnwWmKxqqXKLw'
```



生成的Token给到客户端，客户端就可以拿这个Token进行鉴权了

```python
>>> import requests
>>> token = 'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE1NDgyMjg4MzgsImlhdCI6MTU0ODE0MjQzOCwiZGF0YSI6eyJ1c2VybmFtZSI6ImFkbWluQDE2My5jb20ifX0.oKc0SafgksMT9ZIhTACupUlz49Q5kI4oJA-B8-GHqLA'
>>>
>>> r = requests.get('http://localhost/api/user', headers={'Authorization': 'Token '+token})
>>> r.json()
{'username': 'lee@mysite.cn', 'fullname': 'mysite', 'is_active': True}
```


---
title: Jenkins忘记密码如何处理
date: 2020-08-03 17:25:13
tags:
- Jenkins
categories:
- CICD
- Jenkins
description: jenkins忘记管理员密码后无法登陆的处理方式
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1596456872097&di=9fcad7633bbe4b55c2542f5ca482dd04&imgtype=0&src=http%3A%2F%2F5b0988e595225.cdn.sohucs.com%2Fimages%2F20180604%2F6fe777ff8ac84205b320d59eada0f954.jpeg
---



编辑jenkins目录下的`config.xml`文件，注释下面的代码：

```xml
<useSecurity>true</useSecurity>  
<authorizationStrategy class="hudson.security.FullControlOnceLoggedInAuthorizationStrategy">  
  <denyAnonymousReadAccess>true</denyAnonymousReadAccess>  
</authorizationStrategy>  
<securityRealm class="hudson.security.HudsonPrivateSecurityRealm">  
  <disableSignup>true</disableSignup>  
  <enableCaptcha>false</enableCaptcha>  
</securityRealm>
```



然后重启jenkins。



进入首页>“系统管理”>“Configure Global Security”；勾选如下的选项

<img src="jenkins-forgetpass.png" style="zoom:50%;" />



保存后重新点击首页>“系统管理”,发现此时出现“管理用户”，之后修改用户密码就可以登陆了。








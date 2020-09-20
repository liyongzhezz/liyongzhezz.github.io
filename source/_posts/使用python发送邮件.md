---
title: 使用python发送邮件
date: 2020-09-19 13:03:25
tags:
- Python
categories:
- Python
- 代码
description: 使用python发送电子邮件
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1600501993822&di=399d08b2fa4b7f2b2181b1f576224799&imgtype=0&src=http%3A%2F%2Fd.ifengimg.com%2Fq100%2Fimg1.ugc.ifeng.com%2Fnewugc%2F20190520%2F15%2Fwemedia%2Fc1024597eac12eb5d088951616fb2e45491581a0_size43_w1078_h516.jpg
---



# 指定邮件服务器发送

这里以163邮箱为例：

```python
from smtplib import SMTP    


email_account = "youraccount@163.com"
account_password = "VCGXSEWSDGBFM"  #授权密码

s = SMTP("smtp.163.com")
s.login(email_account, account_password)

to_mail = ["12345678@qq.com"]

# 发送消息，包含发送方和收件人
msg = f'''\ 
From: {email_account}
To: {','.join(to_mail)} 
Subject:测试
这是一封测试邮件'''.encode("utf8")

s.sendmail("baojingtongzhi@163.com",to_mail,msg)

s.quit()
```



> 如果msg对象含带中文需要编码`encode("utf8")`



<br>



# 发送邮件并抄送

```python
import smtplib


def sendMail(body):
    smtp_server = 'smtp.163.com'
    email_account = 'youraccount@163.com'
    account_password = 'VCGXTBA3MFA'
    to_mail = ["123456789@qq.com"]
    cc_mail = ["aaccxx@163.com"]
    from_name = 'monitor'
    subject = '监控'

    # 或者写成列表再拼接，相比上面顶头写更美化些
    mail = [
        "From: %s <%s>" % (from_name, account_password),
        "To: %s" % ','.join(to_mail),
        "Subject: %s" % subject,
        "Cc: %s" % ','.join(cc_mail),
        "",
        body
        ]
    msg = '\n'.join(mail).encode("utf8")

    try:
        s = smtplib.SMTP()
        s.connect(smtp_server, '25')
        s.login(email_account, account_password)
        s.sendmail(email_account, to_mail+cc_mail, msg)
        s.quit()
    except smtplib.SMTPException as e:
        print("Error: %s" %e)
        
        
if __name__ == "__main__":
    sendMail("这是一封测试邮件")
```



<br>



# 发送带附件的邮件

由于`SMTP.sendmail()`方法不支持添加附件，所以需要借助email模块来实现。email模块是一个构造邮件和解析邮件的模块。

```python
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from email.header import Header
from email import encoders
from email.mime.base import MIMEBase
from email.utils import parseaddr, formataddr

# 格式化邮件地址
def formatAddr(s):
    name, addr = parseaddr(s)
    return formataddr((Header(name, 'utf-8').encode(), addr))

def sendMail(body, attachment):
    smtp_server = 'smtp.163.com'
    email_account = 'youraccount@163.com'
    account_password = 'ASDAAFHRMSIJO'
    to_mail = ["12345678@qq.com"]

    # 构造一个MIMEMultipart对象代表邮件本身
    msg = MIMEMultipart()
    # Header对中文进行转码
    msg['From'] = formatAddr('管理员 <%s>' % email_account)
    msg['To'] = ','.join(to_mail)
    msg['Subject'] = Header('监控', 'utf-8')

    # plain代表纯文本
    msg.attach(MIMEText(body, 'plain', 'utf-8'))

    # 二进制方式模式文件
    with open(attachment, 'rb') as f:
        # MIMEBase表示附件的对象
        mime = MIMEBase('text', 'txt', filename=attachment) # 使用MIMEBase类构造附件并添加到msg对象
        # filename是显示附件名字
        mime.add_header('Content-Disposition', 'attachment', filename=attachment)
        # 获取附件内容
        mime.set_payload(f.read())
        encoders.encode_base64(mime)
        # 作为附件添加到邮件
        msg.attach(mime)
    try:
        s = smtplib.SMTP()
        s.connect(smtp_server, "25")
        s.login(email_account, account_password)
        s.sendmail(email_account, to_mail, msg.as_string())  # as_string()把MIMEText对象变成str
        s.quit()
    except smtplib.SMTPException as e:
        print("Error: %s" % e)
        
    
if __name__ == "__main__":
    sendMail('这是一封携带附件的测试邮件', 'test.txt')
```



<br>



# 发送HTML格式邮件

```python
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from email.header import Header
from email.utils import parseaddr, formataddr


# 格式化邮件地址
def formatAddr(s):
    name, addr = parseaddr(s)
    return formataddr((Header(name, 'utf-8').encode(), addr))

def sendMail(body):
    smtp_server = 'smtp.163.com'
    email_account = 'youraccount@163.com'
    account_password = 'ASDAAFHRMSIJO'
    to_mail = ["12345678@qq.com"]

    # 构造一个MIMEMultipart对象代表邮件本身
    msg = MIMEMultipart()
    # Header对中文进行转码
    msg['From'] = formatAddr('管理员 <%s>' % email_account)
    msg['To'] = ','.join(to_mail)
    msg['Subject'] = Header('监控', 'utf-8')
    msg.attach(MIMEText(body, 'html', 'utf-8'))

    try:
        s = smtplib.SMTP()
        s.connect(smtp_server, "25")
        s.login(email_account, account_password)
        s.sendmail(email_account, to_mail, msg.as_string())  # as_string()把MIMEText对象变成str
        s.quit()
    except smtplib.SMTPException as e:
        print("Error: %s" % e)

if __name__ == "__main__":
    body = """
    <h1>测试邮件</h1>
    <h2 style="color:red">这是一封HTML测试邮件</h2>
    """
    sendMail(body)
```



<br>



# 发送图片邮件

```python
import smtplib
from email.mime.text import MIMEText
from email.mime.image import MIMEImage
from email.mime.multipart import MIMEMultipart
from email.header import Header
from email.utils import parseaddr, formataddr


# 格式化邮件地址
def formatAddr(s):
    name, addr = parseaddr(s)
    return formataddr((Header(name, 'utf-8').encode(), addr))

def sendMail(body, image):
    smtp_server = 'smtp.163.com'
    email_account = 'youraccount@163.com'
    account_password = 'ASDAAFHRMSIJO'
    to_mail = ["12345678@qq.com"]

    # 构造一个MIMEMultipart对象代表邮件本身
    msg = MIMEMultipart()
    # Header对中文进行转码
    msg['From'] = formatAddr('管理员 <%s>' % email_account)
    msg['To'] = ','.join(to_mail)
    msg['Subject'] = Header('监控', 'utf-8')
    msg.attach(MIMEText(body, 'html', 'utf-8'))

    # 二进制模式读取图片
    with open(image, 'rb') as f:
        msgImage = MIMEImage(f.read())  # 使用MIMEImage类构造图片并添加到msg对象
        # 定义图片ID，根据ID在HTML里获取图片
        msgImage.add_header('Content-ID', '<image1>')
        msg.attach(msgImage)

    try:
        s = smtplib.SMTP()
        s.connect(smtp_server, "25")
        s.login(email_account, account_password)
        s.sendmail(email_account, to_mail, msg.as_string())  # as_string()把MIMEText对象变成str
        s.quit()
    except smtplib.SMTPException as e:
        print("Error: %s" % e)
        
if __name__ == "__main__":
    body = """
    <h1>测试图片</h1>
    <img src="cid:image1">
    """
    sendMail(body, "123.jpg")
```




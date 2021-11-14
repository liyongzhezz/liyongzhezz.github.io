---
title: python文件操作及内存数据读写
date: 2021-11-14 18:14:01
tags:
- Python
categories:
- 编程
- Python
- 文件I/O
description: 最常用的python操作文件的方法，以及向内存中读写数据
cover: https://gimg2.baidu.com/image_search/src=http%3A%2F%2Fpic1.zhimg.com%2Fv2-931b10f17b1ae77a38b46a5ef84034e7_1440w.jpg%3Fsource%3D172ae18b&refer=http%3A%2F%2Fpic1.zhimg.com&app=2002&size=f9999,10000&q=a80&n=0&g=0n&fmt=jpeg?sec=1639476922&t=1628d6ddc3c037739c2b04c3d151b1a7
---



{% note info 'fas fa-bullhorn' %}

介绍python中最常见的文件操作方法，以及如何读写csv文件；最后使用 StringIO 将数据写入内存

更新于 2021-11-14

{% endnote %}

<br>



# 打开文件

打开文件直接使用`open`函数，指定文件路径即可，该函数的底层是调用操作系统接口：

```python
f = open("/tmp/a.txt", mode='r', encoding='utf-8')
content = f.read()
```



更推荐的方式是使用`with`方式：

```python
with open("/tmp/a.txt", mode='r', encoding='utf-8') as f:
  content = f.read()
```



- `f`为文件句柄，对文件句柄的任何操作都是对文件的操作；
- `encoding`可以省略，`mode='r'`也可以直接写成类似`r`的方式，标识打开文件的模式；



使用`with`打开文件可以不需要关闭文件句柄，并且一个语句可以操作多个文件句柄，例如：

```python
with open("/tmp/a.txt", mode='r', encoding='utf-8') as f1, \
    open("/tmp/b.txt", mode='r', encoding='utf-8') as f2:
  print(f1.read())  
  f2.write('aaaaa')
```





<br>



# 关闭文件

关闭文件直接调用文件句柄的`close()`方式即可：

```python
f = open("/tmp/a.txt", mode='r', encoding='utf-8')
f.close()
```



> 打开文件后好的习惯应该是都有一个关闭文件句柄的操作



<br>



# 指定打开文件的模式

不管是直接`open()`方式还是`with`方式，都需要有打开文件的模式即`mode`，有如下的一些模式可选择：

- `r`：只读（文件必须存在，不存在则报错）
- `w`：只写（不可读，文件不存在则创建，存在则会清空原文件内容）
- `a`：追加（不可读，文件不存在则创建，存在只会追加内容到原文件）
- `rb`：以二进制读取（对于非文本类内容）
- `wb`：以二进制写入（对于非文本类内容）
- `ab`：以二进制追加（对于非文本类内容）
- `r+`：读写模式



> `r+`模式中，文件指针开始位于文件头部，如果先写后读，那么文件会从还是覆盖原文件内容，直到写内容完毕





# csv文件读写

## csv文件写入

```python
import csv

# 以写入方式打开一个csv文件
file = open('test.csv','w')

# 调用writer方法，传入csv文件对象，得到的结果是一个CSVWriter对象
writer = csv.writer(file)

# 调用CSVWriter对象的writerow方法，一行行的写入数据
writer.writerow(['name', 'age', 'score'])

# 还可以调用writerows方法，一次性写入多行数据
writer.writerows([['zhangsan', '18', '98'],['lisi', '20', '99'], ['wangwu', '17', '90'], ['jerry', '19', '95']])
file.close()
```



## csv文件读取

```python
import csv

# 以读取方式打开一个csv文件
file = open('test.csv', 'r')

# 调用csv模块的reader方法，得到的结果是一个可迭代对象
reader = csv.reader(file)

# 对结果进行遍历，获取到结果里的每一行数据
for row in reader:
    print(row)

file.close()
```



<br>

# 内存数据读写

## 读写字符串数据

```python
from io import StringIO

# 创建一个StringIO对象
f = StringIO()

# 可以像操作文件一下，将字符串写入到内存中
f.write('hello\r\n')
f.write('good')

# 需要调用getvalue()方法才能获取到写入到内存中的数据
print(f.getvalue())

f.close()
```



## 读写二进制数据

```python
from io import BytesIO

f = BytesIO()
f.write('你好\r\n'.encode('utf-8'))
f.write('中国'.encode('utf-8'))

print(f.getvalue())
f.close()
```


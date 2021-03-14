---
title: python文件操作
date: 2021-03-14 20:31:45
tags:
- Python
categories:
- Python
- 文件操作
description: 使用 fileinput操作文件
cover: https://ss1.bdstatic.com/70cFuXSh_Q1YnxGkpoWK1HF6hhy/it/u=4163938920,38528075&fm=26&gp=0.jpg
---



fileinput 是对 open 函数的再次封装，在仅需读取数据的场景中， fileinput 显然比 open 做得更专业、更人性



## 从标准输入中读取

```python
import fileinput

for line in fileinput.input():
    print(line) 
```



当没有传入任何参数的时候， fileinput会将标准输入作为输入源；



## 从文件读取

```python
import fileinput

with fileinput.input(files=('a.txt',)) as file:
    for line in file:
        print(f'{fileinput.filename()} 第{fileinput.lineno()}行: {line}', end='') 
```



例如`a.txt`的内容为：

```
hello
world
xxx
```



那么实例代码的输出为：

```python
a.txt 第1行: hello
a.txt 第2行: world
a.txt 第3行：xxx
```



fileinput默认使用的模式为`mode='r'`，调整模式可以添加参数，如读取二进制文件：`mode='rb'`，但是fileinput只有这两种模式；



## 打开多个文件

fileinput的参数`files`可以接收包含多个文件的list或者tuple，例如：

```python
import fileinput

with fileinput.input(files=('a.txt', 'b.txt')) as file:
    for line in file:
        print(f'{fileinput.filename()} 第{fileinput.lineno()}行: {line}', end='') 
```



`a.txt` 和 `b.txt` 的内容分别是：

```bash
$ cat a.txt
hello
world

$ cat b.txt
hello
python
```



那么示例程序的输出结果为：

```python
a.txt 第1行: hello
a.txt 第2行: world
b.txt 第3行: hello
b.txt 第4行: python
```



可以看到输出的内容是对的，但是行号不是真正的行号，如果想获取文件的真正行号，可以使用`fileinput.filelineno()`方法：

```python
import fileinput

with fileinput.input(files=('a.txt', 'b.txt')) as file:
    for line in file:
        print(f'{fileinput.filename()} 第{fileinput.filelineno()}行: {line}', end='') 
```



这样输出的就是文件真实行号了：

```python
a.txt 第1行: hello
a.txt 第2行: world
b.txt 第1行: hello
b.txt 第2行: python
```



## 读取的同时备份文件

fileinput的`backup`参数可以指定后缀名：

```python
import fileinput


with fileinput.input(files=("a.txt",), backup=".bak") as file:
    for line in file:
        print(f'{fileinput.filename()} 第{fileinput.lineno()}行: {line}', end='') 
```



运行后，会多出一个备份文件：

```bash
$ ls -l a.txt*
-rw-r--r--  1 MING  staff  12  2 27 10:43 a.txt

$ python demo.py
a.txt 第1行: hello
a.txt 第2行: world

$ ls -l a.txt*
-rw-r--r--  1 MING  staff  12  2 27 10:43 a.txt
-rw-r--r--  1 MING  staff  42  2 27 10:39 a.txt.bak
```



## 标准输出重定向替换

`fileinput.input` 有一个 inplace 参数，表示是否将标准输出的结果写回文件，默认不取代

```python
import fileinput

with fileinput.input(files=("a.txt",), inplace=True) as file:
    print("[INFO] task is started...") 
    for line in file:
        print(f'{fileinput.filename()} 第{fileinput.lineno()}行: {line}', end='') 
    print("[INFO] task is closed...") 
```



运行后，会发现在 for 循环体内的 print 内容会写回到原文件中了。而在 for 循环体外的 print 则没有变化:

```bash
$ cat a.txt
hello
world

$ python demo.py
[INFO] task is started...
[INFO] task is closed...

$ cat a.txt 
a.txt 第1行: hello
a.txt 第2行: world
```



## 常用的方法

- `fileinput.filenam()`
  返回当前被读取的文件名。在第一行被读取之前，返回 `None`。
- `fileinput.fileno()`
  返回以整数表示的当前文件“文件描述符”。当未打开文件时（处在第一行和文件之间），返回 `-1`。
- `fileinput.lineno()`
  返回已被读取的累计行号。在第一行被读取之前，返回 `0`。在最后一个文件的最后一行被读取之后，返回该行的行号。
- `fileinput.filelineno()`
  返回当前文件中的行号。在第一行被读取之前，返回 `0`。在最后一个文件的最后一行被读取之后，返回此文件中该行的行号。

- `fileinput.isfirstline()`
  如果刚读取的行是其所在文件的第一行则返回 `True`，否则返回 `False`。
- `fileinput.isstdin()`
  如果最后读取的行来自 `sys.stdin` 则返回 `True`，否则返回 `False`。
- `fileinput.nextfile()`
  关闭当前文件以使下次迭代将从下一个文件（如果存在）读取第一行；不是从该文件读取的行将不会被计入累计行数。直到下一个文件的第一行被读取之后文件名才会改变。在第一行被读取之前，此函数将不会生效；它不能被用来跳过第一个文件。在最后一个文件的最后一行被读取之后，此函数将不再生效。
- `fileinput.close()`
  关闭序列。



## 实例

### 将CRLF转换为LF

```python
import sys
import fileinput

for line in fileinput.input(files=('a.txt', ), inplace=True):
    #将Windows/DOS格式下的文本文件转为Linux的文件
    if line[-2:] == "\r\n":  
        line = line + "\n"
    sys.stdout.write(line)
```



## 结合re进行日志分析

```bash
#--样本文件--：error.log
aaa
1970-01-01 13:45:30  Error: **** Due to System Disk spacke not enough...
bbb
1970-01-02 10:20:30  Error: **** Due to System Out of Memory...
ccc
```



```python
import re
import fileinput
import sys

pattern = '\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}'

for line in fileinput.input('error.log',backup='.bak',inplace=1):
    if re.search(pattern,line):
        sys.stdout.write("=> ")
        sys.stdout.write(line)
```



```
#---测试结果---
=> 1970-01-01 13:45:30  Error: **** Due to System Disk spacke not enough...
=> 1970-01-02 10:20:30  Error: **** Due to System Out of Memory...
```



## 实现类似grep功能

```python
import sys
import re
import fileinput

pattern= re.compile(sys.argv[1])
for line in fileinput.input(sys.argv[2]):
    if pattern.match(line):
        print(fileinput.filename(), fileinput.filelineno(), line)
```



```bash
$ ./demo.py import.*re *.py
#查找所有py文件中，含import re字样的
addressBook.py  2   import re
addressBook1.py 10  import re
addressBook2.py 18  import re
test.py         238 import re
```


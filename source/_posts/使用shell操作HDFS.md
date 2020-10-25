---
title: 使用shell操作HDFS
date: 2020-10-25 13:10:39
tags:
- HDFS
categories:
- 大数据
- HDFS
description: 使用shell命令操作hdfs的数据
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1603612760848&di=0cea9339eafad636168163c22cf0efd1&imgtype=0&src=http%3A%2F%2F05.imgmini.eastday.com%2Fmobile%2F20170204%2F20170204191726_c6d9284321172acb0baa79aab390f4ff_1.jpeg
---



## 查看文件

```bash
$ hdfs dfs -ls /
20/10/25 13:19:17 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
```



> 可以看到当前没有文件



<br>



## 上传文件到HDFS

首先创建一个测试文件：

```bash
$ cat > hello.txt << EOF
123
hadoop hdfs
hadoop hdfs 2222
EOF
```



上传到hdfs：

```bash
$ hdfs dfs -put hello.txt /
$ hdfs dfs -ls /
```

![](put.png)



<br>



## 查看文件内容

```bash
$ hdfs dfs -text /hello.txt
$ hdfs dfs -cat /hello.txt
```

![](txt.png)



<br>



## 创建目录

```bash
$ hdfs dfs -mkdir /test
$ hdfs dfs -ls /

# 递归创建
$ hdfs dfs -mkdir -p /test/a/b

# 递归展示
$ hdfs dfs -ls -R /
```

![](mkdir.png)



<br>



## 从本地拷贝文件到HDFS

```bash
$ hdfs dfs -copyFromLocal hello.txt /test/a/b/h.txt
$ hdfs dfs -ls -R /
$ hdfs dfs -text /test/a/b/h.txt
```

![](copy.png)



<br>



## 从HDFS拿文件到本地

```bash
$ hdfs dfs -get /test/a/b/h.txt
$ ls
```

![](get.png)



<br>



## 删除文件和目录

```bash
# 删除文件
$ hdfs dfs -rm /hello.txt
$ hdfs dfs -ls /

# 删除目录
$ hdfs dfs -rm -R /test
$ hdfs dfs -ls /
```

![](delete.png)


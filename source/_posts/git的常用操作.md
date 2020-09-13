---
title: git的常用操作
date: 2020-09-13 13:45:57
tags:
- git
categories:
- git
description: git的常用操作
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1599986079915&di=2b0688693e4df05f79337825639058b0&imgtype=0&src=http%3A%2F%2Fimage.mamicode.com%2Finfo%2F201807%2F20180720154704366185.gif
---



# git安装

一般服务器都默认安装了git，如果没有安装可以使用如下的命令进行安装：

```bash
$ yum install -y git
$ git --version
```



安装好后需要进行简单的设置：

```bash
# 设定用户名
$ git config --global user.name <username>

# 设定邮箱
$ git config --global user.email <email>

# 查看所有配置
$ git config --list
```



<br>



# 初始化项目

初始化一个项目可以使用下面的命令进行：

```bash
# 新建一个项目目录
$ mkdir newproject
$ cd newproject

# 初始化
$ git init
```



`git init`会创建一个`.git`的隐藏目录，其中的文件和用途如下：

- `hooks`：存放一些脚本，可设定指定的git命令运行后执行相应的脚本；
- `info`：包含仓库的信息；
- `logs`：保存所有更新的引用记录；
- `objects`：项目中的所有对象；
- `refs`：保存指向对象的索引；
- `COMMIT_EDITMSG`：保存最新的commit信息；
- `config`：git仓库的配置文件；
- `description`：仓库描述信息；
- `index`：二进制格式文件，暂存区；
- `HEAD`：包含了当前分支的引用，通过这个文件git可以得到下一次commit的parent；
- `ORIG_HEAD`：head指针的前一个状态；



<br>



# 创建git仓库

## clone方式

```bash
# 新建一个仓库目录
$ mkdir gitrepo 
$ cd gitrepo

# 从某个gitweb仓库克隆项目，例如克隆jquery项目
$ git clone https://github.com/jquery/jquery.git
```



上边的clone命令会在本地生成一个和远程仓库同名的目录来保存代码，如果想要一个新的名字，则可以如下的方式：

```bash
$ git clone https://github.com/jquery/jquery.git <本地目录名>
```



> 一般`clone`命令除了支持http(s)协议外，还支持git、ssh等协议。



## pull方式

pull命令是从远程仓库取回分支的更新并合并到对应的本地分支，例如：

```bash
# 将远程仓库的next分支与本地master分支合并
$ git pull origin netx:master

# 如果远程分支就是当前分支
$ git pull origin master

# 或者
$ git pull
```





## fetch方式

fetch也是从远程仓库取回新的更新内容，例如：

```bash
# 将远程仓库的更新全部取回本地
$ git fetch <远程主机名>

# 取回远程主机上特定分支的更新到本地，例如取回master
$ git fetch origin master
```



和pull方式不同的是，fetch方式不会立刻合并到本地分支，fetch后再本地需要使用`远程主机名/分支名`的方式访问，例如：`origin/master`



<br>



# 提交代码到远程仓库

## 提交到本地暂存区

当修改完文件后提交到本地暂存区，以免后续发现问题误提交到远程仓库：

```bash
# 添加所有更新的文件（一般在项目根目录下运行）
$ git add .

# 添加某个文件到暂存区，例如README.md
$ git add README.md

# 查看暂存区的文件
$ git ls-files --stage
```



## 提交到本地仓库

提交到远程仓库之前，先提交到本地的仓库：

```bash
$ git commit

# 好的习惯是commit的时候添加一段说明
$ git commit -m '添加了xxx文件'
```



## 提交到远端仓库

使用`push`将本地仓库的更新提交到远端：

```bash
# 提交到远端的master分支，如果分支不存在，则会自动创建
$ git push origin master
```



<br>



# 管理远端服务器



## 查看远端服务器

为了方便管理，git可以添加多个远端服务器，并且要求每个远端服务器都要有一个主机名（例如origin），可以用过下面的命令进行查看：

```bash
$ git remote -v 
```



查看主机的详细信息，可以使用下面的命令：

```bash
$ git remote show <远端主机名>
```



## 添加远端主机

在clone远程仓库的代码时，使用的远程主机地址会被自动保存并命名为`origin`，如果不想用这个名字，可以用如下的方式制定一个名字：

```bash
$ git clone -o jquery https://github.com/jquery/jquery.git
```



手动添加可以使用下面的命令：

```bash
$ git remote add <主机名> <地址>
```



## 删除远端主机

删除远端主机命令如下：

```bash
$ git remote rm <主机名>
```



## 远端主机改名

更改远端主机的名字可以使用如下的方式：

```bash
$ git remote rename <原名字> <新名字>
```



<br>



# 分支管理

## 查看分支

```bash
# 查看所有分支
$ git branch -a

# 查看远程分支
$ git branch -r

# 查看当前所在的分支
$ git branch
```



## 创建分支

```bash
$ git branch <新分支名称>
```



## 切换分支

使用checkout来切换所在的分支：

```bash
$ git checkout <要切换到的分支名称>

# 创建一个新分支并且换过去
$ git checkout -b <新分支名>
```



## 删除分支

```bash
$ git branch -D <分支名称>
```



## 合并分支

例如将abc分支合并到master分支：

```bash
$ git checkout master
$ git merge abc
```


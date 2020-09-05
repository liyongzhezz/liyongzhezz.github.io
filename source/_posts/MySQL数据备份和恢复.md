---
title: MySQL数据备份和恢复
date: 2020-08-18 15:48:37
tags:
- MySQL
categories:
- 数据库
- MySQL
description: mysql数据备份和恢复的一般方式
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1597747062450&di=c7d22dc4a7843068ac49675b4ff3f9f2&imgtype=0&src=http%3A%2F%2Fku.90sjimg.com%2Felement_origin_min_pic%2F00%2F95%2F28%2F6456f2dd94e9c06.jpg
---





# 数据备份

## mysqldump备份命令

`mysqldump`是mysql提供的数据备份工具，它可以将mysql数据备份为一个文本文件，文件中实际包含了多个`CREATE`和`INSERT`语句，可以重新创建表结构和插入数据。



`mysqldump`的命令格式为：

```bash
$ mysqldump -u <user> -h <host> -p<password> dbname [table-name, [table-name, table-name,...]] > filename.sql
```

- user：用户名；
- host：mysql服务器名；
- dbname：要备份的数据库名；
- table-name：dbname中的某个或多个表；
- filename.sql：备份内容保存的文件；



## 备份单个库中所有表

假设备份 booksdb 库中的所有表结构和数据，则可以使用如下的命令：

```bash
$ mysqldump -uroot -p booksdb > booksdb_bak.sql
```



如果只想备份表结构而不备份数据，可以使用 `--opt` 参数：

```bash
$ mysqldump --opt -uroot -p booksdb > booksdb_bak.sql
```



如果只想备份数据而不被分表结构，可以使用 `-t` 参数：

```bash
$ mysqldump -t -uroot -p booksdb > booksdb_bak.sql
```



## 备份数据库中某个表

例如备份 booksdb 中的 books 表，就可以使用如下的命令：

```bash
$ mysqldump -uroot -p booksdb books > books.sql
```



如果只想备份表结构而不备份数据，可以使用 `--opt` 参数：

```bash
$ mysqldump --opt -uroot -p booksdb books > books.sql
```



如果只想备份数据而不被分表结构，可以使用 `-t` 参数：

```bash
$ mysqldump -t -uroot -p booksdb books > books.sql
```



## 备份多个数据库

备份多个数据库的表结构和数据的时候，需要指定 `--databases` 参数，例如备份 booksdb和test两个库：

```bash
$ mysqldump -uroot -p --databases booksdb test > books_test_bak.sql
```



如果使用 `--all-databases`，则可以备份所有的数据库，如：

```bash
$ mysqldump -uroot -p --all-databases > all_db.sql
```





## 复制数据目录备份

mysql以文件的形式保存数据，所以可以以复制文件的方式备份数据。



在linux下，mysql的数据目录一般为：`/var/lib/mysql`，注意，在备份前，需要对相关的表执行` LOCK TABLES` 操作，然后执行` FLUSH TABLES` 操作，以保证备份的一致性以及允许备份期间继续读操作。当然也可以停止mysql服务然后再备份。



>  这种方式在恢复数据时最好恢复到相同版本的mysql中，以防止版本不兼容的问题。对Innodb表不可用。

<br>



# 数据恢复

## 使用mysql命令恢复

mysql命令可以直接将备份的sql文件执行，格式如下：

```bash
$ mysql -uroot -p <dbname> < filename.sql
```

- dbname：如果是恢复mysqldump创建的包含建库语句的sql文件，则不需要指定dbname；
- filename.sql：待执行的sql备份文件；



例如：恢复 booksdb 数据库：

```bash
$ mysql -uroot -p booksdb < booksdb_bak.sql
```

> 执行前，确保数据库中已经存在 booksdb库。



如果是登录了数据库，也可以使用source命令来恢复数据：

```sql
mysql> use booksdb; 
mysql> source /root/booksdb_bak.sql
```





## 直接复制到数据库目录

如果通过复制文件方式备份的数据库，可以复制到mysql数据目录下进行恢复。通过这种方式恢复，请务必确保两者版本一致，**且对于Innodb表不可用。**

> 复制前关闭mysql服务，复制后注意修改文件和目录的权限，通常用户和组都是mysql。

<br>





# 数据迁移

## 相同版本mysql数据库迁移

在主版本号相同的mysql间迁移数据，推荐使用`mysqldump`进行迁移，而不可以使用拷贝文件的方式迁移数据。



例如，将 www.abc.com 上的数据库迁移到 www.def.com 上，可以使用如下命令：

```bash
# 在 www.abc.com 上执行 
$ mysqldump -h www.abc.com -uroot -p <password> dbname | mysql -h www.def.com -uroot -p <password>
```

>  命令使用管道将mysqldump命令直接传递给mysql。如果是迁移所有的库，则可以使用 `--all-databases `参数。



## 不同版本mysql数据库迁移

不同版本的迁移，需要注意版本使用的字符集是否相同，例如mysql4.x默认使用的是 latin1，而mysql5.x 使用的是 utf8，如果数据库中有中文的话，需要进行字符集修改。

一般新版本会对旧版本有一定兼容性，最好使用`mysqldump`命令进行数据导入和导出。




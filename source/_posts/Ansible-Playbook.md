---
title: Ansible Playbook
date: 2020-07-19 14:11:08
tags:
- 自动化运维工具
- ansible
categories:
- 自动化运维
- Ansible
description: Ansible playbook的基本原理和使用方法
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1595149274046&di=36b13106c6195228a30d1ff098039ed7&imgtype=0&src=http%3A%2F%2F5b0988e595225.cdn.sohucs.com%2Fimages%2F20180901%2F70f9cc358e7f4e3f890316630d6fbf3f.jpeg
---



## playbook介绍

### 什么是playbook

ansible playbook可以类比为ansible为基础的编程语言，它具有编程语言所具有的顺序结构、选择结构、循环结构，并且可以定义变量。可以通过playbook将ansible命令行编排在一起，完成一些更高级的工作。



>  playbook编写方法才哟呵你给yaml格式



### 功能

playbook具有如下的功能：

- 编写复杂的ansible任务；
- 控制任务执行；
- 声明配置；
- ......



### 和命令行对比

- playbook是对命令行的编排；
- 命令行适合简单、单一、快速的任务，playbook适合复杂的任务；
- ......



<br>



## playbook语法



### 执行命令

```yaml
# test-playbook.yml
- hosts: test
  remote_user: root
  tasks:
  - name: Hello World
    shell: ls /root
```



上边就是一个简单的playbook示例，其中：

- `- hosts`指定了playbook运行在哪些服务器上；
- `remote_user`指定远程运行的用户；
- `tasks`指定运行的任务，这里是使用`shell`模块运行`ls /root`这个命令。



执行playbook可以使用如下的命令：

```bash
$ ansible-playbook test-playbook.yml
```



### 定义变量

playbook变量定义可以在host文件中定义，例如：

```
test ansible_ssh_host=192.168.1.2 ansible_ssh_port=22 ansible_ssh_user=root

[node]
test
```



其中`test`是主机`192.168.1.2`的别名，也是一个playbook变量。



变量也可以定义在yaml文件中，使用`vars`关键字，例如：

```yaml
# test-playbook.yml
- hosts: test
  remote_user: root
  vars:
    dir: /root
  tasks:
  - name: Hello World
    shell: ls "{{ dir }}"
```



或者是：

```yaml
# test-playbook.yml
- hosts: test
  remote_user: root
  vars:
    command: /root
  tasks:
  - name: Hello World
    shell: "{{ command }}"
```

> 引用变量的时候最好都用双引号将大括号引起来，以免报错



### 系统变量

ansible提供了很多系统变量，可以完成根据系统的个性化操作，系统变量可以通过下面的命令查看：

```bash
$ ansible <hostname> -m setup
```



使用系统变量的时候，直接使用下面的形式即可，例如：

```jinja2
{{ ansible_devices.sda.model }}
```



###  when条件语句

```yaml
......
tasks:
- name: "shutdown system"
  command: /sbin/shutdonw -t now
  when: ansible_os_family == "Debian"
```

上边这个playbook代码段及使用了when条件语句，判断当系统为Debian时关闭系统。



### bool值

和其他编程语言一样，真用`true`表示，假用`false`表示：

```yaml
......
vars:
  change: true
tasks:
- shell: echo "this is true"
  when: change
- shell: echo "this is false"
  when: not change
```

上边这个playbook通过变量`change`的真假来控制具体执行的任务。



### with_items 循环语句

```yaml
......
tasks:
- name: add user
  user: name={{ item }} state=present groups=wheel
  with_items:
    - user1
    - user2
```

上边这个palybook代码段将会循环`with_items`中的内容，然后使用user模块添加对应的用户



### with_nested嵌套循环

```yaml
......
tasks:
- name: user access control
    mysql_user: name={{ item[0] }}
                priv={{ item[1] }}.*:ALL
                append_privs=yes
                password=foo
    with_nested:
      - ['mike', 'admin']
      - ['testdb', 'docdb', 'userdb']
```

上面的playbook代码段通过`with_nested`来给mysql增加用户并授权。



### 有条件循环

```yaml
......
tasks:
- name: looping
  command: echo {{ item }}
  with_items: [0,1,2,3,4,5,6]
  when: item > 5
```

上边playbook代码段通过结合`when`和`with_items`来实现有条件的循环，打印大于5的数值。



<br>



## paybook常用命令

```bash
# 检查语法是否正确
$ ansible-playbook  --syntax-check  /path/to/playbook.yaml

# 测试运行
$ ansible-playbook -C /path/to/playbook.yaml

# 运行playbook
$ ansible-playbook  /path/to/playbook.yaml
```


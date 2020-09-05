---
title: Ansible常用配置
date: 2020-07-19 12:44:41
tags:
- 自动化运维工具
- ansible
categories:
- 自动化运维
- Ansible
description: ansible主配置文件和主机配置文件中的常用参数
cover: https://ss0.bdstatic.com/70cFuHSh_Q1YnxGkpoWK1HF6hhy/it/u=80183688,2043592912&fm=26&gp=0.jpg
---



## 主配置文件常用参数

ansible主配置文件默认在`/etc/ansible/ansible.cfg`，其中常用的参数如下：

- `inventory = /etc/ansible/hosts`：主机定义文件位置；
- `library = /usr/share/my_modules`：ansible默认搜寻模块的位置；
- `remote_tmp = ~/.ansible/tmp`：远程执行的临时目录； 
- `forks = 5`：在与主机通信时的默认并行进程数；
- `poll_interval = 15`：多久回查任务的状态，当具体的poll interval 没有定义时默认是5秒；
- `sudo_user = root`：sudo使用的默认用户，默认是root；
- `ask_sudo_pass = True`：用来控制Ansible playbook在执行sudo时是否询问sudo密码，默认为no；
- `ask_pass = True`：控制Ansible playbook 是否会自动默认弹出密码；
- `transport = smart`：通信机制。默认值为`smart`；
- `remote_port = 22`：远程SSH端口。 默认是22；
- `module_lang = C`：模块和系统之间通信的计算机语言，默认是C语言；
- `host_key_checking = False`：是否检查主机密钥；
- `timeout = 10`：SSH超时时间；
- `log_path = /var/log/ansible.log`：日志文件存放路径；
- `module_name = command`：ansible命令执行默认的模块；
- `private_key_file = /path/to/file`：私钥文件存储位置；



<br>



## 主机配置文件常用参数

主机配置文件默认为`/etc/ansible/hosts`



### 基本格式

```
[dbserver]    // 使用[] 进行主机分类
172.25.254.1    // 指定iP地址
mysql.example.com    // 指定域名
back.mysql.com:5505  //指定ssh的端口
test ansible_ssh_port=5321 ansible_ssh_host=172.25.254.2 //设定ssh端口及别名为test
db[1...5].example.com  //支持通配符

[web]
localhost ansible_connection=local    // 指定连接类型为localhost
web1.example.com ansible_connection=ssh ansible_ssh_user=web    // 执行连接用户为web
```



### 支持的其他配置

```
# 指定主机别名对应的真实 IP
ansible_ssh_host  

# 指定连接到这个主机的 ssh 端口，默认 22   
ansible_ssh_port 

# 指定连接到该主机上的用户
ansible_ssh_user   

# sudo 密码  
ansible_sudo_pass 

# sudo 命令路径
ansible_sudo_exe    

# 连接类型，可以是 local、ssh 或paramiko，ansible1.2 之前默认为 paramiko   
ansible_connection  

# 私钥文件路径
ansible_ssh_private_key_file      

# 目标系统的 shell 类型，默认为sh
ansible_shell_type   
```


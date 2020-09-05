---
title: Playbook使用实例
date: 2020-07-20 15:09:58
tags:
- 自动化运维工具
- ansible
categories:
- 自动化运维
- Ansible
description: 使用playbook完成一些任务的实例
cover: https://pic.rmb.bdstatic.com/609935701daff991f058cfa45a4219d8.jpeg
---



## 创建系统用户和组

创建系统用户组，创建新的系统用户并加入该组



```yaml
# add_user.yaml
- hosts: cluster
  remote_user: root
  tasks:
  - name: add a group
    group: name=asgroup system=true
  - name: add a user
    user: name=asuser group=asgroup system=true
```



检查语法并测试运行：

```bash
# 检查语法
$ ansible-playbook --syntax-check add_user.yaml

# 测试运行
$ ansible-playbook -C add_user.yaml
```

<img src="check-add-user.png" style="zoom:50%;" />



测试没有报错，执行playbook：

```bash
$ ansible-playbook add_user.yaml
```

<img src="add-user.png" style="zoom:50%;" />



验证：

```bash
$ ansible cluster -m shell -a 'tail -1 /etc/passwd'
$ ansible cluster -m shell -a 'getent group asgroup'
```

<img src="check-res.png" style="zoom:50%;" />



<br>



## 部署http服务

部署http服务，修改监听端口并启动服务

```yaml
- hosts: cluster
  remote_user: root
  tasks:
  - name: install httpd service
    yum: name=httpd state=latest
  - name: install conf file
    copy: src=httpd.conf dest=/etc/httpd/conf/httpd.conf
    notify: restart httpd service
    tags: instconf
  - name: start httpd service
    service: name=httpd state=started
  handlers:
  - name: restart httpd service
    service: name=httpd state=restarted
```



> 需要在当前目录下准备好修改过的`httpd.conf`配置文件



这里使用了`handlers`，它可以由特定条件触发（使用`notify`）。这里是当有新的文件复制到远程服务器就触发服务重启的操作。



`install conf file`这个任务还指定了一个名为`instconf`的tag，它的作用是可以在运行playbook的时候只运行指定的tag的任务。例如后期想要更新配置文件，只需要运行tag为`instconf`的任务，即可实现更新远程服务配置文件和重启服务：

```bash
# 查看playbok中有哪些tag
$ ansible-playbook --list-tags httpd.yaml

# 运行指定tag的任务
$ ansible-playbook -t instconf httpd.yaml
```



<br>



## 安装python flask环境

使用playbook安装python flask环境，具有数据库和缓存能力

```yaml
- hosts: cluster
  remote_user: root
  become: true
  tasks:
  - name: install python for centos
    yum: 
      name: "{{ item }}"
      state: installed
    with_items:
      - python-devel
      - python-setuptools
    when: ansible_distribution == 'Centos'
  - name: install python for ubuntu
    apt: 
      name: "{{ item }}"
      state: latest
      update_cache: yes
    with_items:
      - libpython-dev
      - python-setuptools
    when: ansible_distribution == 'Ubuntu'
  - name: install pip
    shell: easy_install pip
  - name: pip install flask and redis
    pip:
      name: "{{ item }}"
    with_items:
      - flask
      - redis
    
```



<br>



## 安装zabbix

使用playbook安装zabbix的server和client端：

```yaml
- hosts: cluster
  become: true
  remote_user: root
  tasks:
  - name: install zabbix repo for centos
    yum:
      name: http://repo.zabbix.com/zabbix/3.4/rhel/7/x86_64/zabbix-release-3.4-2.el7.noarch.rpm
      state: installed
    when: ansible_distribution == 'Centos'
  - name: download zabbix repo for ubuntu
    get_url:
      url: http://repo.zabbix.com/zabbix/3.4/ubuntu/pool/main/z/zabbix-release/zabbix-release-3.4-1+xenial_all.deb
      dest: /tmp/zabbix.deb
    when: ansible_distribution == 'Ubuntu'
  - name: install zabbix repo for ubuntu
    apt:
      deb: /tmp/zabbix.deb
    when: ansible_distribution == 'Ubuntu'
  - name: install zabbix server
    yum:
      name: "{{ item }}"
      state: installed
    with_items:
      - zabbix-server-mysql
      - zabbix-proxy-mysql
      - zabbix-web-mysql
    when: ansible_distribution == 'Centos'
  - name: install zabbix client
    apt:
      name: zabbix-agent
      update_cache: yes
      state: installed
    when: ansible_distribution == 'Ubuntu'
  - name: config zabbix server
    replace:
      path: /etc/zabbix/zabbix_server.conf
      regexp: DBUser=zabbix
      replace: DBUser=root
    when: ansible_distribution == 'Centos'
  - name: init zabbix mysql
    shell: zcat /usr/share/doc/zabbix-server-mysql-3.4.7/create.sql.gz | mysql -uroot zabbix
    when: ansible_distribution == 'Centos'
  - name: disable selinux
    selinux:
      state: disabled
    when: ansible_distribution == 'Centos'
  - name: start zabbix server
    systemd:
      name: zabbix-server
      state: started
    when: ansible_distribution == 'Centos'
  - name: start zabbix agent
    systemd:
      name: zabbix-agent
      state: started
    when: ansible_distribution == 'Ubuntu'
```


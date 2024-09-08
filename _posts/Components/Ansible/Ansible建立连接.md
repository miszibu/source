---
title: Ansible建立连接
date: 2020-07-23 23:29:41
tags: [Ansible]
---

Ansible可以通过账号密码 和 RSA Key的方式来远程控制目标机器。

本文会讲解如何配置Ansible来连接

<!--more-->

## **个人Ansible 常用配置**

```yaml
vi ansible.cfg

[defaults]
inventory=./hosts           // 指定本地host文件目录
log_path=./playbook_output  // 指定playbook(下文简称pb)的log目录
gather_timeout = 40         // Ansible默认会获取目标机器的metric数据，如果目标机器反应较慢，常常超时，设置其超时时间
command_warnings=False      // 关闭一些无用的warning， 新手可以不设置


```

## 使用账号密码登陆

```shell
cd ${your_ansible_home}
vi hosts

[test_group_with_customize_port]
127.0.0.1:22

[test_group_default_port]
127.0.0.2

[all:vars]
ansible_connection=ssh
ansible_user=test
ansible_ssh_pass=testpwd
```

将ssh连接用户的账号密码设置在host文件中，上面将其设置为全局，也可以针对group设置。

### Necessary Dependency

选择合适的包管理器来安装sshpass

```shell
yum install -y http://mirror.centos.org/centos/7/extras/x86_64/Packages/sshpass-1.06-2.el7.x86_64.rpm

apt-get install sshpass

brew install sshpass
```

### Add Fingerprint

**添加本机fingerprint 到目标的机器的known_hosts**

第一次ssh连接远程机器时，需要用户确认是否信任目标机器，而ansible则会抛出一个异常。

解决方法很简单，手动先连接一遍，选择信任，再退出，重新使用ansible即可。



## 使用ssh免密登陆

这里不需要任何配置，你只需要预先设置好RSA Key就可以。

**详情请见RSA免密登陆章节。**



## 连接Docker Container

Docker Container提供了方便多样的测试环境，如何连接container有如下两种方案

1. **SSH连接容器**
2. **使用Ansible的Delegate_to**

### SSH

#### 寻找镜像 创建容器

在Container中设置sshd服务，并绑定端口到外部宿主机其他端口

在这里可以直接去找预先安装了python和sshd服务的镜像，没有必要自己单独再造轮子，如果真要打镜像，没必要写docker file，太慢了，直接找个基础镜像，做完操作再commit即可。

> docker run -itd --name centos-ansible --privileged -p 2222:22 be666a0ce5e7 /bin/bash

#### 在容器中设置RSA密钥

详情见rsa免密登陆一文

#### Ansible 更改机器默认SSH端口

```shell
vi hosts

[my_server]
hostname:2222
```



### Delegate To

Ansible提供了Task的delegate to方法，可以把Task指定给某台机器执行。我们可以预先注册Docker Container为Host，然后将任务指定给Docker Container执行。

```yaml
- name: add container to inventory
  add_host:
    name: centos5-ansible
    ansible_connection: docker
    ansible_user: root
  changed_when: false
  
- name: print hostname
  delegate_to: centos5-ansible
  command: hostname
  register: result
  ignore_errors: yes
```

通过add_host 将container加入到host中，然后后续的所有tasks，都可以通过添加delegate_to: centos5-ansible 来指定container执行。

> Note: 为每一个Module添加delegate_to过于繁复，可以使用include的配置叠加的性质，来拓展到include的所有tasks上
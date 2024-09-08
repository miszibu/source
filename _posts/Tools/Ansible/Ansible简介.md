---
title: Ansible简介
date: 2019-03-17 12:28:55
tags: [Ansible]
---

> 🚧
>
> Ansible , 一个集群管理部署的工具。
>
> 一台机器批量管理机器集群。传统的方式，我们需要编写shell Script，上传到对应服务器集群，然后执行部署，比较麻烦，而且为了做到幂等性，我们需要对Script仔细的审查，测试。
>
> 在Playbook中，我们可以使用Playbook, 使用其自带的众多Module，这些Module都是幂等的。只要对于模块足够的熟悉，毫无疑问，Ansible能起到极大的作用。

<!--more-->

## Tutorial

### SSH 公钥连接

默认情况，Ansible是通过SSH进行连接的，因此需要把Ansible 控制主机的RSA公钥，加入到被控主机的Know_hosts中去。
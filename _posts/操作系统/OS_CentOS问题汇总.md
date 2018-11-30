---
title: CentOS7问题汇总
date: 2018-06-28 17:25:01
tags: [Linux]

---



之前玩的的Ubuntu，包括云服务器等等，但是实际项目的服务器是Centos，因此单独开一个坑来记录Centos遇到的问题。

<!-- more -->

**1.C7默认安装 网关不打开**

```bash
ping baidu.com 
ping:baidu.com :Name or Service not know
```

原因有二：

1. 默认安装没有DNS
2. 默认安装网管未打开

```bash
cd /etc/sysconfig/network-scipts
vi ifcfg-ens33 	#后面的尾缀可能不同
修改onboot属性为yes

vi /etc/resolv.conf
i
nameserver 8.8.8.8
nameserver 8.8.4.4
Esc :wq
新增DNS记录

sudo service network restart 或者 reboot
```


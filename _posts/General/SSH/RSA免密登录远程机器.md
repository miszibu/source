---
title: RSA免密登录远程机器
date: 2020-07-29 23:27:25
tags: [SSH]
---

通过预先配置RSA公钥，可以做到免密登陆远程机器。但是在设置时，往往会遇到很多问题，因此记录步骤和常见问题来快速定位解决。

<!--more-->

## 本地创建RSA 密钥对

```shell
# 生成2048位的RSA公密钥，密钥有多种不同类型和长度，4096安全性更改，2048可被暴力破解
ssh-keygen -t rsa -b 2048
cd ~/.ssh; ls -l
-rw------- 1 root root 1675 Sep  4  2019 id_rsa        私钥
-rw-r--r-- 1 root root  398 Sep  4  2019 id_rsa.pub   公钥
```

默认情况下，rsa密钥会被创建到`~/.ssh`目录下，也可以在命令交互时，指定到其他目录中去。

## 拷贝公钥到目标机器

我们已经创建RSA密钥对，需要拷贝**公钥**到到目标机器指定文件中去上去。个人常用Secure Copy命令

> scp file.txt user@Server:/remote/directory 

放置公钥到以下任意一个文件中，推荐直接放置`/etc/.ssh`目录下，如果最终仍然无法建立连接，则同样创建authorized_keys到`~/.ssh`目录下。

* **file:  /etc/ssh/authorized_keys/root**
* **file:  /root/.ssh/authorized_keys**

### 具体实例：以Root用户为例

#### 方法1：/etc/ssh/authorized_keys/root

```shell
mkdir -p /etc/ssh/authorized_keys/root
cd /etc/ssh
chown -R root:root authorized_keys
chmod -R 600 authorized_keys
cat id_ras.pub >> /etc/ssh/authorized_keys/root
```

我们将本地root账号机器的公钥放置到了目标机器的authorized_keys文件夹中的root文件中，那么我们就具备了在本地使用root账号连接到远端root账号的权限了。

如果我们想以其他帐号登陆目标机器，我们应该创建在authorized_keys下创建别的用户文件，并且设置为该用户的600权限。

#### 方法2：/root/.ssh/authorized_keys

```shell
cd ~/.ssh
touch authorized_keys
chown root:root authorized_keys
chmod 600 authorized_keys
cat id_ras.pub >>  ~/.ssh/authorized_keys
```

我们在用户的.ssh文件夹下，创建了authorized_keys的文件，将公钥加了进去。

## 重启ssh服务

重启SSH服务来激活配置修改

不同服务器使用的管理命令和ssh服务都有所不同，但是基本逃不出以下几种方式。自行选择

```
systemctl restart sshd
systemctl restart ssh

service sshd restart
service ssh restart

/etc/init.d/ssh restart
/etc/init.d/sshd restart
```

## 连接目标机器

> ssh user@server_name  -p 22

## 常见问题 & Debug

> ssh -vvT user@yourhost

使用以上命令，来查看ssh连接过程的具体会话信息。根据报错信息，来直接使用搜索引擎解决。

以下是常见的几个问题

* 公钥目录权限不对，其他用户不应该具有写权限。
* ssh配置问题，如禁止RSA密钥登陆，禁止root用户登陆等，修改对应配置即可。
* authorized_keys目录问题，有一次在Container中使用/etc/.ssh目录连接失败，放置个人用户目录底下即可。考虑是ssh版本问题。
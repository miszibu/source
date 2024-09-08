---
title: Redis-Password
date: 2020-05-17 19:09:25
tags: [Redis] 
---

Redis有以下两个密码设置

* **requirepass**: 用于客户端连接Redis
* **masterauth:** 当 Redis Master 节点设置了 requirepass, 需要在 Slave 节点添加配置以连接。

Redis 密码设置可以通过修改配置文件和命令行的方式。注意：命令行的参数配置会在节点重启后消失。

<!--more-->

### Redis password introduction

**requirepass : Guard redis** 

```
# Require clients to issue AUTH <PASSWORD> before processing any other
# commands.  This might be useful in environments in which you do not trust
# others with access to the host running redis-server.
#
# This should stay commented out for backward compatibility and because most
# people do not need auth (e.g. they run their own servers).
#
# Warning: since Redis is pretty fast an outside user can try up to
# 150k passwords per second against a good box. This means that you should
# use a very strong password otherwise it will be very easy to break.

# requirepass foobared
```

**Master Auth: Replica used to connect master node**

```
# If the master is password protected (using the "requirepass" configuration
# directive below) it is possible to tell the replica to authenticate before
# starting the replication synchronization process, otherwise the master will
# refuse the replica request.
#
# masterauth <master-password>
```



---



### Two ways to set redis password

**Set redis password by commands**

**Note:** If we set the password by command(not all configurations support this way) without make config changes persistence. These configurations will disappear after redis restart.

>  ./redis-cli -p {{redis_port}} -h {{ip_addr}} config set masterauth '{{password}}'

>  ./redis-cli -p {{redis_port}} -h {{ip_addr}} config set requirepass '{{password}}'

**Set redis password via configuration file**

```
# requirepass foobared

# masterauth <master-password>
```
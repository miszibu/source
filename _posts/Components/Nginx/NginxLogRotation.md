---
layout: Nginx
title: 使用CronTab设置Nginx日志滚动
date: 2022-05-12 19:09:25
tags: [Nginx] 

---

Nginx 本身不实现日志滚动策略，服务器上access.log 13G，收到系统部报警，写个 CronTab 来定时滚动服务器日志，并清除。

<!-- more -->

# Log rotation shell 

```shell
#!/bin/bash
cd /usr/local/nginx/logs
# 判断当前access.log的 mb 大小
fileSize=`du -sm access.log | awk '{print $}'`

if[ $fileSize -lg  1024]
then
  rm -rf access.log.*
  mv access.log access.log.0
  ps -fe|grep nginx|grep master|awk '{print $2}'|xargs kill -USR1
  gzip access.log.0
else
  echo "Access log size smaller than 1024MB, Check Pass!"
fi
```

# Crontab Setting

```shell
crontab -l 
crontab -e 

# 编辑
0 01 * * * /your_shell_path
```



# Reference

[Nginx Log Rotation](https://www.nginx.com/resources/wiki/start/topics/examples/logrotation/)


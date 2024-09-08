---
title: DockerIssues-容器内外文件夹无法映射
date: 2021-05-18 16:45:57
tags: [Docker]
---

DockerIssues-容器内外文件夹无法映射，本质上文件映射不一致，文件重建后inode变化，看起来还存在而已。

<!--more-->



**问题描述**: 在线上部署环境时，出现 Docker Container内部文件夹没有与外部文件夹关联起来的问题。Docker mount并没有生效，而通过 docker inspect命令可以看到mounts配置已经存在，且内外文件夹不存在权限问题。

**Root Causes:** 其实是宿主机对应文件夹被脚本删除后，重新创建了，其 *inode* 值已经发生了变化。对于开发人员而言，只关注其文件夹名称，并未注意到这点。解决方案很简单，**重启容器，使其对新建文件夹进行重新映射。**

**什么是 inode:**

> An inode is denoted by the phrase "file serial number", defined as a *per-file system* unique identifier for a file.That file serial number, together with the device ID of the device containing the file, uniquely identify the file within the whole system.Device ID (this identifies the device containing the file; that is, the scope of uniqueness of the serial number).
>
> Inode本质上就是串行的文件id，与所在设备的id一起构成了整个操作系统唯一的文件标志符。

一般通过 `stat`命令来获取文件、文件夹的inode值，包括文件相关的权限，大小，修改日期等一系列参数。

```
➜  Documents stat -x muc_source_record.sql
  File: "muc_source_record.sql"
  Size: 11875        FileType: Regular File
  Mode: (0644/-rw-r--r--)         Uid: (  501/ligaofeng)  Gid: (   20/   staff)
Device: 1,5   Inode: 1716864    Links: 1
Access: Mon Dec 16 23:19:35 2019
Modify: Wed Aug 22 18:04:33 2018
Change: Mon Dec 16 23:19:35 2019
```



# Reference

[inode-Wikipad](https://en.wikipedia.org/wiki/Inode)


---
title: 关于js && java 的随机数问题
date: 2016-10-23 20:58:47
tags: [杂谈]
---

> * 发布日记，杂文，所见所想
> * 编写技术文稿
> * 日后开扩别的区块

## 关于旧的博客

之前的博客 因为主题出错的原因弄得比较乱，尝试修复后失败无果。
由于个人原因吧，更新博文这件事情，我觉得还是很主观的，一但一次不更，就像春节期间，直接导致数个月都没有更新，因此这次搬迁希望，保持好更新的习惯。

<!--more-->

## 更新中遇到的问题

一周前打算重新部署下githubpage，没想到本来以为简单的部署，遇到了本地服务成功，但github上page始终404，找了很多资料，都说是缺少index.html。
什么！我跟着教程走，什么都对的啊，困惑了两天。
偶然突然看到域名二字，方才想起，本机和手机浏览器，访问miszibu.github.io 都是404的原因是，是因为之前的域名miszibu.com访问的都是旧的仓库ip地址，新仓库虽然网址一样，但是Ip地址已经不同了！

```ipconfig /flushdns```

清除本地dns缓存，清除静态映射。重新访问依旧404。
更改域名的@和A命令，等待Dns服务器服务器更新后，重新访问，问题解决。

![book-pic](http://bruce.u.qiniudn.com/2013/11/27/reading/photos-1.jpg)

## Linux 安装Hexo

**1.Centos 安装node环境**

```shell
wget https://nodejs.org/dist/v6.10.1/node-v6.10.1-linux-x64.tar.xz#获取linux版本node  
tar -xf node-v6.10.1-linux-x64.tar #解压缩Node包
mv node-v6.10.1-linux-x64 node# 重命名node包方便使用
ln -s /root/node-v6.10.1/bin/node /usr/local/bin/node  #内联node/npm 命令到服务器命令列表中
ln -s /root/node-v6.10.1/bin/npm /usr/local/bin/npm
```


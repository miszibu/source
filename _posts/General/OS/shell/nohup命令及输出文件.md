---
title: nohup命令及输出文件
date: 2018-12-24 10:21:45
tags: [Linux]
---

> 使用nohup启动程序，在连接断开后，程序不挂断继续运行。
>
> nohup ${Command}  >${fileName} 2>&1 &

<!--more-->

### 操作系统中的三种常用的流

0. 标准输入流 **stdin**
1. 标准输出流 **stdout**
2. 标准错误流 **stderr**

`> console.txt` : 是 `1 > console.txt`的缩写 ，意思是将标准输出流输出到console.txt中去。

`< console.txt` ：是`0 < console.txt`的缩写，意思将console.txt的文件输入到标准输入流中去。

` 2>&1`是什么意思呢，`&1`是指向1标准输出流打开的fd文件地址，所以意思是 将 2标准输出流输出到1标准输出流中去。

错误示例：`> console.txt 2>console.txt`

这样的话stdout/stderr都会打开console.txt,然后竞争覆盖，这不是我们需要的结果。

`/dev/null `：这个是个无底洞，任何输出都可以定向到这里，但是无法打开。但不在乎输出时，可以定向到这里。
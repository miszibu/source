---
title: Linux-命令
date: 2018-03-20 01:09:01
tags: [Linux,mark]
---

总结了一些自己常用到的linux命令，包括文件的增删改查，环境的安装，压缩解压，运维命令。

* Centos: 
* Ubuntu
* Dibian

<!-- more -->

**0.连接服务器**

xshell/mintty ssh root@IP地址

**1.cd命令**

/ 根目录  ~当前用户目录 ..父目录

**2.ls命令**——参数可以组合使用

* -l：列出长数据，包含文件属性和权限数据
* -a：列出全部的文件 包括开头为.的隐藏文件
* -d：列出文件目录
* -h：将文件容量列出
* -R：连同子目录内容一同递归列出

**3.grep命令**

常用于分析**一行**的信息，若当中有我们所需要的信息，就将**该行显示出来**，常与管道命令相结合用于筛选加工命令的输出。

- -a或--text 不要忽略二进制的数据。
- -A<显示列数>或--after-context=<显示列数> 除了显示符合范本样式的那一列之外，并显示该列之后的内容。
- -b或--byte-offset 在显示符合范本样式的那一列之前，标示出该列第一个字符的位编号。
- -B<显示列数>或--before-context=<显示列数> 除了显示符合范本样式的那一列之外，并显示该列之前的内容。
- -c或--count 计算符合范本样式的列数。
- -C<显示列数>或--context=<显示列数>或-<显示列数> 除了显示符合范本样式的那一列之外，并显示该列之前后的内容。
- -d<进行动作>或--directories=<进行动作> 当指定要查找的是目录而非文件时，必须使用这项参数，否则grep指令将回报信息并停止动作。
- -e<范本样式>或--regexp=<范本样式> 指定字符串做为查找文件内容的范本样式。
- -E或--extended-regexp 将范本样式为延伸的普通表示法来使用。
- -f<范本文件>或--file=<范本文件> 指定范本文件，其内容含有一个或多个范本样式，让grep查找符合范本条件的文件内容，格式为每列一个范本样式。
- -F或--fixed-regexp 将范本样式视为固定字符串的列表。
- -G或--basic-regexp 将范本样式视为普通的表示法来使用。
- -h或--no-filename 在显示符合范本样式的那一列之前，不标示该列所属的文件名称。
- -H或--with-filename 在显示符合范本样式的那一列之前，表示该列所属的文件名称。
- -i或--ignore-case 忽略字符大小写的差别。
- -l或--file-with-matches 列出文件内容符合指定的范本样式的文件名称。
- -L或--files-without-match 列出文件内容不符合指定的范本样式的文件名称。
- -n或--line-number 在显示符合范本样式的那一列之前，标示出该列的列数编号。
- -q或--quiet或--silent 不显示任何信息。
- -r或--recursive 此参数的效果和指定"-d recurse"参数相同。
- -s或--no-messages 不显示错误信息。
- -v或--revert-match 反转查找。
- -V或--version 显示版本信息。
- -w或--word-regexp 只显示全字符合的列。
- -x或--line-regexp 只显示全列符合的列。
- -y 此参数的效果和指定"-i"参数相同。
- --help 在线帮助。

```shell
grep [-abcEFGhHilLnqrsvVwxy][-A<显示列数>][-B<显示列数>][-C<显示列数>][-d<进行动作>][-e<范本样式>][-f<范本文件>][--help][范本样式][文件或目录...]
grep test *.md #显示所有*.md的文件中含有test内容的那一行
grep -r test /usr/zibu #递归查找该目录所有文件 包含test内容的行
```

**4.find命令**

[find命令](http://www.runoob.com/linux/linux-comm-find.html)：由于find命令功能强大，这里按需查找吧

```shell
find . -name "*.c" #列出当前目录及其子目录下所有延伸档名是c的文件
find . -type f #列出当前目录及其子目录下所有一般文件
find . -ctime -20 #将目前目录及其子目录下所有最近 20 天内更新过的文件列出
```

**5.cp命令**

```shell
cp -a file1 file2 #连同文件的所有特性把文件file1复制成文件file2  
cp file1 file2 file3 dir #把文件file1、file2、file3复制到目录dir中

-a ：将文件的特性一起复制  
-p ：连同文件的属性一起复制，而非使用默认方式，与-a相似，常用于备份  
-i ：若目标文件已经存在时，在覆盖时会先询问操作的进行  
-r ：递归持续复制，用于目录的复制行为  
-u ：目标文件与源文件有差异时才会复制  
```

**6.mv命令**——用于移动文件，目录或更名

```shell
mv file1 file2 #将file1重命名为file2  
mv file1 file2 file3 dir #把文件file1、file2、file3移动到目录dir中  

-f ：force强制的意思，如果目标文件已经存在，不会询问而直接覆盖  
-i ：若目标文件已经存在，就会询问是否覆盖  
-u ：若目标文件已经存在，且比目标文件新，才会更新  
```

**7.rm命令**——该命令用于删除文件或目录

```shell
rm -i file # 删除文件file，在删除之前会询问是否进行该操作  
rm -fr dir # 强制删除目录dir中的所有文件  

-f ：就是force的意思，忽略不存在的文件，不会出现警告消息  
-i ：互动模式，在删除前会询问用户是否操作  
-r ：递归删除，最常用于目录删除，它是一个非常危险的参数  
```

**8.ps命令**——用于显示当前进程 (process) 的状态。 

```shell
ps --help a #查询ps命令详情
ps -A #显示所有进程信息
ps -u root #显示root进程用户信息
ps -ef #显示所有命令，连带命令行
```

**9.kill命令**——用于删除执行中的程序或工作

```shell
kill 12345 #杀死12345进程
kill -KILL 12345 #强制杀死新城
```

**10.free命令**——用于显示内存状态

- -b 　以Byte为单位显示内存使用情况。
- -k 　以KB为单位显示内存使用情况。
- -m 　以MB为单位显示内存使用情况。
- -o 　不显示缓冲区调节列。
- -s<间隔秒数> 　持续观察内存使用状况。
- -t 　显示内存总和列。
- -V 　显示版本信息。

```shell
free -tm #以MB为单位显示全部内存信息
free -s 10 #每10秒显示内存信息
```

**11.file命令**——用于辨识文件类型

```shell
file zibuHome.md  #显示文件类型
zibuHome.md: UTF-8 Unicode text
file zibuHome.md -i #显示文件MIME类型
zibuHome.md: text/plain; charset=utf-8
file /usr 		  #显示目录信息
/usr: directory
```

**12.tar命令**

**13.cat/head/tail命令**——用于查看文件内容，可以连接管道命令

```shell
cat /etc/profile #查看profile文件内容
cat -b /etc/fstab#查看/etc/目录下的profile内容，并且对非空白行进行编号，行号从1开始； 
cat -n /etc/profile#对/etc目录中的profile的所有的行(包括空白行）进行编号输出显示； 
cat -E /etc/profile#注：查看/etc/下的profile内容，并且在每行的结尾处附加$符号； 
cat /etc/fstab /etc/profile | more#连接管道工具一页页查看多个文件内容

head -n 行数值 文件名
tail -b 行数值 文件名
```

**14.chgrp命令**

```shell

```



**15.chown命令**

```shell

```



**16.chmod命令**

```shell

```



**17.vim命令**

```shel

```



**18.gcc命令**

**19.time命令**

**20.apg-get命令**

apt-get可以运作deb包

```shell
安装：apt-get install <package_name> 
卸载：apt-get remove <package_name> 
更新：apt-get update <package_name>
apt-cache search package  搜索包 
apt-cache show package    获取包的相关信息，如说明、大小、版本等 
sudo apt-get install package  安装包 
sudo apt-get install package -- reinstall 重新安装包 
sudo apt-get -f install     修复安装"-f = --fix-missing" 
sudo apt-get remove package 删除包 
sudo apt-get remove package -- purge 删除包，包括删除配置文件等 
sudo apt-get update  更新源 
sudo apt-get upgrade 更新已安装的包 
sudo apt-get dist-upgrade 升级系统 
sudo apt-get dselect-upgrade 使用 dselect 升级 
apt-cache depends package 了解使用依赖 
apt-cache rdepends package 是查看该包被哪些包依赖 
sudo apt-get build-dep package 安装相关的编译环境 
apt-get source package 下载该包的源代码 
sudo apt-get clean && sudo apt-get autoclean 清理无用的包 
sudo apt-get check 检查是否有损坏的依赖
```

**21.rpm/yum命令**

yum命令可以运作rpm包

```shell
安装：yum install <package_name>
卸载：yum remove <package_name> 
更新：yum update <package_name> 
查找: yum search <keyword>
列出所有可安装的包： yum list
列出所有可更新的包： yum list updates
列出所有已安装的包： yum list installed
列出所指定的软件包： yum list <package_name>
```

**22.CPU负载命令**

**23.硬盘负载命令**

**24.内存负载命令**

**25.网络情况排查**

wget

### 相关引用

[初窥Linux 之 我最常用的20条命令](https://blog.csdn.net/ljianhui/article/details/11100625)
---
[Python](../../../../.Trash/Python)title: shell脚本编程
date: 2018-09-18 10:08:01
tags: shell
---

> 在服务器的运维之中 shell 脚本无处不在，在工作中涉及到服务器脚本时，这方面的能力需要补上。
>
> 虽然现在py方兴未艾，shell脚本由于其复杂性，不适宜用作大型脚本的开发，但是在小型运维脚本方面，shell作为*nix原生的设计，很多地方都值得学习。
>
> shell作为一门与操作系统直接打交道的语言，也支持相当多的特性，本文不与展开，因为过于繁杂实际上并不能记住反而使文章冗余。

<!--more-->

## 开胃菜

```shell
#!/bin/sh          #是shell脚本的注释 #!作为开头 专有注释 标识路径后所指定的程序即使解释此脚本文件shell程序
DIR=`dirname $0`   #'dirname $0'为当前脚本文件的父目录 DIR=即为声明一个变量为DIR，=号左右不可以有空格
cd $DIR			   # 进入父目录下、即为当前目录
nohup java -XX:+UseG1GC -XX:+HeapDumpOnOutOfMemoryError -Xms512M -Xmx4G -jar `ls | grep jar` > /dev/null 2>&1 & 
# java -jar 'ls | grep jar' 启动已经打包好的jar包
# -XX:+UseG1GC 使用G1 GC收集器 
# -XX:+HeapDumpOnOutOfMemoryError  当堆内存空间溢出时输出堆的内存快照。
# -Xms512M 初始堆大小 默认值为物理内存的1/64 当空余堆内存小于40%时，JVM会增大堆
# -Xmx4G 最大堆大小 默认值为物理内存的1/4 当空余对内存大于70%时，JVM会减小堆
# > 标准输出流重定向 /dev/null 重定向到空设备文件 2：标准错误流 1：标准输出流
# 2>&1 stdout/stderr 往一个设备文件输出 但是stderr会沿用sedout的pipe 
# >a 2>a 则会打开两个pipe，每个管道往文件里写入数据时,会flush掉原有的数据
# & 等同 2>&1 2的重定向(>)等于1
```

## 前菜

```shell
cd `dirname $0`
BIN_DIR=`pwd` # 脚本所在目录
cd ..
PROJECT_DIR=`pwd` # 脚本所在父目录 项目目录

mvn clean install -DskipTests # 清除原有Jar包 并 重新Install
# -DskipTests，不执行测试用例，但编译测试用例类生成相应的class文件至target/test-classes下。
# -Dmaven.test.skip=true，不执行测试用例，也不编译测试用例类。

TARGET_JAR="$PROJECT_DIR/trade-web/target/trade.jar" # 选定目标Jar文件

if [ ! -f $TARGET_JAR ]; then	# 如果目标文件不存在 输出错误信息 并 退出
    echo "ERROR: $TARGET_JAR not exist"
    exit 1
fi

rm -rf ./trade.jar # 移除当前目录下的Jar文件
cp $TARGET_JAR ./	# 拷贝新打包的Jar文件

echo "kill prev app.."

ps aux | grep java | grep "trade-web" | awk '{print $2}' | xargs kill -9
# 找出并杀死原有项目进程

DEBUG_OPT="-agentlib:jdwp=transport=dt_socket,address=48091,server=y,suspend=n"
# 设置远程调试的Agent，JVM允许外部的库运行时注入到JVM，这些外部的库就是Agent,Agent能够动态修改Class文件(方法区)
# **-agentlib:jdwp=... **来引入 jdwp 这个 Agent 的。
# jdwp 是一个 JVM 特定的 JDWP（Java Debug Wire Protocol） 可选实现，用来定义调试者与运行JVM之间的通讯，它的是通过 JVM 本地库的 jdwp.so 或者 jdwp.dll 支持实现的。

nohup java $DEBUG_OPT -Dfile.encoding=UTF-8 -DappName=trade-web -jar $PROJECT_DIR/trade.jar --spring.profiles.active=dev > app.log 2>&1 &

tail -f $PROJECT_DIR/app.log
# 查看log日志 
```



## Shell基础

### Shell 环境

Linux 的 Shell 种类众多，常见的有：

- **Bourne Shell**（/usr/bin/sh或/bin/sh）
- **Bourne Again Shell**（/bin/bash）
- C Shell（/usr/bin/csh）
- K Shell（/usr/bin/ksh）
- Shell for Root（/sbin/sh）

 Bourne Again Shell**(Bash)**是Bourne Shell的后续兼容版本  ，所以，像 **#!/bin/sh**，它同样也可以改为 **#!/bin/bash**。

---



### 执行 Shell 脚本：

**1、作为可执行程序**

```shell
chmod +x run.sh chmod 777 test.sh  #使脚本具有执行权限
./test.sh  #执行脚本
```

注意，一定要写成 ./test.sh，而不是 **test.sh**，运行其它二进制的程序也一样，直接写 test.sh，linux 系统会去 PATH 里寻找有没有叫 test.sh 的，而只有 /bin, /sbin, /usr/bin，/usr/sbin 等在 PATH 里,因此需要告诉系统在当前目录需找对应文件。

**2、作为解释器参数**

这种运行方式是，直接运行解释器，其参数就是 shell 脚本的文件名，如：

```
/bin/sh test.sh
/bin/php test.php
```

这种方式运行的脚本，不需要在第一行指定解释器信息，写了也没用。

---

## Shell变量

### 声明变量：局部变量 环境变量

~~~shell
my_name="zibu" #=号两边不能有空格 
#变量只可以由数字英文下划线组成 数字不能开头

echo ${my_name} #${}使用变量

~~~



### 字符串：单引号 双引号

```shell
str='this is a string' 
# 单引号里的任何字符都会原样输出，单引号字符串中的变量是无效的；
# 单引号字串中不能出现单独一个的单引号（对单引号使用转义符后也不行），但可成对出现，作为字符串拼接使用。

your_name='runoob'
str="Hello, I know you are \"$your_name\"! \n"
echo $str
Hello, I know you are "runoob"! 
# 双引号里可以有变量 
# 双引号里可以出现转义字符
```



### 数组：只支持一维数组

```shell
array_name={value0 value1 value2} # shell数组之间使用空格分割 一次性声明数组
array_name[0]=value0
array_name[1]=value1
array_name[2]=value2  			  # 单独定义数组的各个分量

${array_name[n]}				  # 读取数组某个元素
echo ${array_name[@]}			  # 获取数组的全部元素
${#array_name[@]}				  # 获得数组的长度
${#array_name[n]}				  # 获取某个元素某个位置的数据长度
```



---

## Shell运算符：算数，关系，逻辑，布尔，文件测试

### 算数关系运算：原生不支持 利用expr命令

```shell
#!/bin/bash

val=`expr 2 + 2`		# `是反引号 '单引号是字符串 表达符和运算符之间有空格
echo "两数之和为 : $val"

+	加法	`expr $a + $b` 结果为 30。
-	减法	`expr $a - $b` 结果为 -10。
*	乘法	`expr $a \* $b` 结果为  200。 	# *需要转义符
/	除法	`expr $b / $a` 结果为 2。
%	取余	`expr $b % $a` 结果为 0。
=	赋值	a=$b 将把变量 b 的值赋给 a。
==	相等。用于比较两个数字，相同则返回 true。	[ $a == $b ] 返回 false。
!=	不相等。用于比较两个数字，不相同则返回 true。	[ $a != $b ] 返回 true
```



---

## 相关引用

[菜鸟驿站-shell教程](http://www.runoob.com/linux/linux-shell.html)
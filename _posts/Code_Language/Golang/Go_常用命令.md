---
title: Go_常用命令
date: 2018-06-01 14:04:55
tags: [Go]

---

**Go build / install /clean /fmt /doc /get /test**

本文主要介绍了以上几种GO常用命令 

<!--more-->

### go build-编译go文件

1. go build 只会编译main包下的go 文件，不会编译普通包
2. go build 编译main包下文件，会在当前目录下生成一个可执行文件。如果需要在$GOPATH/bin木下生成相应的exe文件，需要执行go install 或者使用 go build -o 路径/a.exe。
3. go build 默认编译当前目录下所有go文件，加上文件名 可单独编译一个文件。
4. go build 会忽略目录下以”_”或者”.”开头的go文件。
5. 如果你的源代码针对不同的操作系统需要不同的处理，那么你可以根据不同的操作系统后缀来命名文件。例如有一个读取数组的程序，它对于不同的操作系统可能有如下几个源文件：

```
array_linux.go 
array_darwin.go 
array_windows.go 
array_freebsd.go1234
```

go build的时候会选择性地编译以系统名结尾的文件（Linux、Darwin、Windows、Freebsd）。例如Linux系统下面编译只会选择array_linux.go文件，其它系统命名后缀文件全部忽略。

**案例**

```shell
#首先运行 go build命令需要进入go path的对应项目src文件中
go build main.go #go build 加上要编译的文件名 就会得到一个可执行文件
go build -o newName main.go #编译文件并且将编译后的文件改名
go build #默认在目录下执行go build 将会得到一个与目录同名的可执行文件
```

### go install-build源码并将相关文件安装到约定的目录下

* go install编译出的可执行文件以其所在目录名(DIR)命名
* go install将可执行文件安装到与src同级别的bin目录下，bin目录由go install自动创建
* go install将可执行文件依赖的各种package编译后，放在与src同级别的pkg目录下
* .exe 文件，带main函数的go文件运行产生，有函数入口可以直接运行
* .a 应用包，不带main函数的go文件运行产生，只能被调用

### go clean-移除当前源码包中的编译文件###



### go fmt-格式化go文件###

开发工具往往默认集成了自动格式化功能，底层实际上也是使用了 go fmt命令。

```shell
gofmt -w testProject/ #格式化整个项目 并写入文件
```

### go get

本质上：clone 目标代码到src目录，然后执行go install。

根据目标代码的地址，使用不同的工具，如果本地不具备该源码管理工具，将不会成功。

* BitBucket (Mercurial Git)
*  GitHub (Git) 
* Google Code Project Hosting (Git, Mercurial, Subversion) 
* Launchpad (Bazaar) 

### go test

自动读取当前目录下*_test.go文件，生成并运行测试用的可执行文件。

相关参数可运行go test testflag命令查看



### 其他命令

```go
go fix #用于修复老版本的代码到新版本
go version #查看go 当前版本
go env #查看当前go的环境变量
go list #列出当前全部安装的PKG
go run #编译并运行go程序
```






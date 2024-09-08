---
title: Go-pprof使用
date: 2018-07-23 14:04:55
tags: [Go]

---



Java已经有了非常成熟性能监控工具，然而Go在这一方面还很欠缺，pprof(Program Profile)是Golang自带的性能分析工具。本文记录了这两天使用pprof包遇到的一些问题和经验。

<!--more-->

### pprof包

golang目前没有成熟的性能分析工具，在社区和配套工具上，golang与Java还是有着不小的差距的。但是golang原生就自带了pprof包，可以对程序进行Cpu，内存，堆栈等信息进行记录分析。

#### **runtime/pprof**  

**runtime/pprof** 与 net/http/pprof 包相比 ，runtime包集成了功能，方便在程序中调用其方法，**功能更为强大**。

而**http/pprof**包则提供了**动态预览**的功能， 能够动态直接查看程序的运行状态。



下面是使用pprof文件记录CPU和Memory信息的代码实现。

```go
需要在运行项目时，在参数中声明 -cpuprofile cpu.prof -memprofile mem.prof
//获取命令行入参
var cpuprofile = flag.String("cpuprofile", "", "write cpu profile `file`")
var memprofile = flag.String("memprofile", "", "write memory profile to `file`")
func main(){
	go cpuProfile()
	//程序的退出函数
    quit()    
}
---------------------------------------------------------------------	
func quit(){
    //结束CPU记时
	log.Info("停止CPU运行记录")
	pprof.StopCPUProfile()
	//记录当前内存状况 以主协程运行 开子协程若主协程执行完 将无法记录下内存状态
	memProfile()
}

func cpuProfile() {
	if *cpuprofile != "" {
		f, err := os.Create(*cpuprofile)
		if err != nil {
			log.Error("创建CPU运行记录文件失败: ", err)
		}
		if err := pprof.StartCPUProfile(f); err != nil {
			log.Error("创建CPU运行状态记录失败: ", err)
		}
	}
}

func memProfile() {
	if *memprofile != "" {
		f, err := os.Create(*memprofile)
		defer f.Close()
		if err != nil {
			log.Error("创建内存使用情况记录文件失败: ", err)
		}
		//runtime.GC() // 调用GC清理内存 可以查看GC后的内存使用情况
		if err := pprof.WriteHeapProfile(f); err != nil {
			log.Error("获取内存使用情况失败: ", err)
		}
	}
}

```

**net/http/pprof**

```go
// net/http/pprof 已经在 init()函数中通过 import 副作用完成默认 Handler 的注册
go func() {
    log.Println(http.ListenAndServe("localhost:6060", nil))
}()
```

>  访问 http://localhost:6060/debug/pprof 即可获取数据

### pprof功能具体实现方法

#### **CPU性能分析**

当 CPU 性能分析启用后，Go runtime 会每 `10ms` 就暂停以下，记录当前运行的 Goroutine 的调用堆栈及相关数据。

#### 内存性能分析

内存性能分析则是在**堆(Heap)分配**的时候，记录一下调用堆栈。默认情况下，是每 `1000` 次分配，取样一次，这个数值可以改变。

**栈(Stack)分配** 由于会随时释放，因此**不会**被内存分析所记录。

由于内存分析是**取样**方式，并且也因为其记录的**是分配内存，而不是使用内存**。因此使用内存性能分析工具来准确判断程序具体的内存使用是比较困难的。

#### 阻塞性能分析

阻塞分析是一个很独特的分析。它有点儿类似于 CPU 性能分析，但是它所记录的是 goroutine 等待资源所花的时间。

阻塞分析对分析程序**并发瓶颈**非常有帮助。阻塞性能分析可以显示出什么时候出现了大批的 goroutine 被阻塞了。阻塞包括：

- 发送、接受无缓冲的 channel；
- 发送给一个满缓冲的 channel，或者从一个空缓冲的 channel 接收；
- 试图获取已被另一个 go routine 锁定的 `sync.Mutex` 的锁；

阻塞性能分析是特殊的分析工具，在排除 CPU 和内存瓶颈前，不应该用它来分析。

### pprof文件分析

在runtime/pprof生成对应的方法后，在命令行中使用go tool pprof工具可以对profile文件进行性能分析。

```shell
go tool pprof XXX.prof #即可分析对应prof文件

go tool pprof localhost:[port]/debug/pprof/profile #cpu性能分析
go tool pprof localhost:[port]/debug/pprof/heap	   #内存性能分析数据
go tool pprof localhost:[port]/debug/pprof/block   #阻塞性能分析

进入命令行后
top [number] #输出对应前N位的数据信息
web #使用grapgviz生成对应svg图 
web > [name].svg #生成svg图后修改名字 存放到同级目录下
list func #显示耗时几个函数

#使用go-torch进行性能分析 svg更好的表现了其中的关系但若想要直观看到所使用资源的占比 falmeGraph会更为直观
go-torch xxx.prof #生成对应火焰图 X轴显示占用资源量 Y轴显示调用栈深度
```





### **grapgviz 和 go-torch的安装**

#### grapgviz

```
windows 从http://www.graphviz.org/download/ 下载对应文件 然后将其bin目录放入系统环境变量
mac brew install grapgviz
```

#### go-torch https://github.com/uber/go-torch

```shell
Mac 
go get github.com/uber/go-torch #下载go-torch
cd $GOPATH/src/github.com/uber/go-torch  #进入go-torch 文件夹
git clone https://github.com/brendangregg/FlameGraph.git #下载其依赖的FlameGrapg.git
Win 
执行与MAC相同命令后
go build #在go-torch文件下build exe文件
#将FlameGraph目录加入 系统变量
./go-torch.exe cpu.prof
 Failed: could not generate flame graph: fork/exec ./FlameGraph/flamegraph.pl: %1 is not a valid Win32 application.
#相关报错 尚未解决
```



### go trace 

go trace 是go1.5以后提供的新的性能分析工具，可以提供更为详细的性能分析数据。
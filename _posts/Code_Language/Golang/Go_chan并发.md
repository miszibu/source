---
title: Go-chan并发
date: 2018-06-03 14:04:55
tags: [Go]
---

Channel是Go中的一个核心类型，可以把它看作一个**管道**，通过它可以完成goroutine之间的通信。Channel是**FIFO的队列**，而且当多个goroutine可以同时读写channel而**不用考虑同步**。

<!--more-->

### 基本操作——箭头表示着数据的方向

```go
ch := make(chan int)//通过make函数创建一个可读可写Int类型的channel-v
ch<-v			   //将V写进channel中
v := <-ch		    //从channel中取出一个数据 写入v中
```

### Channel的类型——只读只写 读写同时###

```go
chan T  //可以接收和发送类型为T的数据
chan<-T //只写channel
<-chan t//只读channel

func main() {
    c := make(chan int)
    go send(c)
    go recv(c)
    time.Sleep(3 * time.Second)
}
//只读只写channel只有在参数传递中能有作用
func send(c chan<- int) {
    for i := 0; i < 10; i++ {
        c <- i
    }
}

func recv(c <-chan int) {
    for i := range c {
        fmt.Println(i)
    }
}
```

### Channel的创建和关闭

```go
ch := make(chan int , 100)//创建一个100容量的channel，当channel空的时候，receiver队列阻塞，当channel满的时候，sender队列阻塞。如果没有设置大小，就默认设置为0
v,ok := <-ch
```

这句语句可以检查channel的开关状态，如果ok为false表示着 channel已经被关闭了

* 如果向一个已经关闭的channel写数据会抛出运行时异常
* 如果读取一个已经关闭的channel，则会将缓冲区的数据读取完，而且可以一直读取该channel数据类型的零值t
* 往一个nil channel中读写数据都会被Block

###Select——适用于管道的Switch

```go
func main() {
	var c1, c2, c3 chan int
	var i1, i2 int
	select {
	case i1 = <-c1:
		fmt.Printf("received ", i1, " from c1\n")
	case c2 <- i2:
		fmt.Printf("sent ", i2, " to c2\n")
	case i3, ok := (<-c3):  // same as: i3, ok := <-c3
		if ok {
			fmt.Printf("received ", i3, " from c3\n")
		} else {
			fmt.Printf("c3 is closed\n")
		}
	default:
		fmt.Printf("no communication\n")
	}
}
```

select是Go中的一个控制结构，类似于用于**通信的switch语句**。每个case必须是一个**通信操作**，要么是发送要么是接收。select随机执行一个可运行的case。如果没有case可运行，它将阻塞，直到有case为止。

* 每个case都必须是一个通信
* 所有channel表达式都会被求值
* 所有被发送的表达式都会被求值
* 如果任意某个通信可以进行，它就执行；其他被忽略。
* 如果有多个case都可以运行，Select会随机公平地选出一个执行。其他不会执行。 如果没有default字句，select将阻塞，直到某个通信可以运行；Go不会重新对channel或值进行求值。

### timeout>

select有很重要的一个应用就是超时处理。因为select语句当没有case可处理时会无限等待。因此需要一个超时操作来处理。

```go
func main() {
    c1 := make(chan string, 1)
    go func() {
        time.Sleep(time.Second * 2)
        c1 <- "result 1"
    }()
    select {
    case res := <-c1:
        fmt.Println(res)
    case <-time.After(time.Second * 1):
        fmt.Println("timeout 1")//超时操作限制1S
    }
}
```

### Timer/Ticker

定义了一个Timer后，会阻塞规定的时间，然后输出一个channel，在将来的那个时间Channel提供了一个当前时间值。

```go
timer := time.NewTimer(time.Second*2)
fmt.Println(<-timer.C)
fmt.Println(time.Now())
```

倘若只是想阻塞线程一定时间，使用time.sleep即可。

可以使用timer.stop来停止计时器

```go
timer2 := time.NewTimer(time.Second)
go func() {
    <-timer2.C
	fmt.Println("Timer 2 expired")
}()
stop2 := timer2.Stop()
if stop2 {
    fmt.Println("Timer 2 stopped")
}
```

Ticker是一个定时器，会不断的以设定的时间为间隔Interval

```go
func main() {
	ticker := time.NewTicker(time.Millisecond*300)//每300MS设置一个ticker
	go func() {
		for t := range ticker.C {//从ticker的channel中轮询访问
			fmt.Println(t)
		}
	}()
	time.Sleep(100*time.Second)//主线程设置时间 否则主进程结束 进程结束
}
```

### 使用channel实现同步###

```go
ch := make(chan bool, 1)
go func() {
	time.Sleep(time.Second)
    ch<-true
		fmt.Println("A 任务完成")
	}()

go func() {
    isFinished := <-ch
	if isFinished {
        fmt.Println("B 在A后 任务完成")
	}else{
		fmt.Println("B 任务失败")
    }
}()
time.Sleep(time.Hour)
```

### 相关引用###

[golang之chan简介](https://leokongwq.github.io/2016/10/15/golang-chan.html)

[Golang 关于通道 Chan 详解](https://blog.csdn.net/netdxy/article/details/54564436)
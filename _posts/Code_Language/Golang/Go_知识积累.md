---
title: Go-知识积累
date: 2018-06-12 11:04:55
tags: [Go]

---



本文主要记录一些Go面试类型的问题，但也为了提高个人对Go的掌握程度。

<!--more-->

### 简述make和new的区别 

make用于**内建类型**（只能用于创建map,slice和channel）的内存分配，并且返回一个**有初始值的T类型**，而不是*T。

new则用于各种类型的**内存分配**，new(T)分配了由零值填充的T类型的内存空间，并且返回其地址，即返回**T类型的指针**。

### 简要描述Go中main和init函数的区别

* main函数只能写在Main包中,是**程序的主入口**

* init可以写在任意的pkg中，用于**初始化操作**，不建议在一个PKG中写多个init，会影响代码的可维护性和可读性，

  main和init函数都**不可以有任何的参数和返回值**



​	//todo 插入asset图片

### 线程，进程，协程

OS将程序分为了进程和线程来提高程序的CPU利用率，一个进程拥有多个线程，多个线程共享同一份内存空间来实现线程间通信。

**协程**，可以看做**用户级线程**（更为轻量级的线程），这个概念是相对于OS **系统级线程**所提出的，协程的切换可以由用户控制，而线程只能由操作系统调度。

Go中的协程又有所不同，Go的协程不由代码控制，而是Go实现了一个**协程调度器**，自动**实现协程的切换**。而且Go的**协程**通信是由**信道channel**实现的

**守护线程daemon**：在主goroutine调用其他goroutine后，当主协程死亡，其他的子协程也直接死亡。这在一定意义上类似于Java的daemon守护线程了。

**协程间通信**：Java中的并发主要是靠锁住临界资源（共享内存）来保证同步的。而**channel**则是goroutines中通信的方法。可以把channel看做一个传送带，实际上是一个**有类型的消息队列**，FIFO。利用Channel来实现协程同步，是Go语言的优势所在，原生支持语言级并发。

### for range

```go
type student struct {
    Name string
    Age int

}

func main() {
    m :=pase_map()
    for k,v :=range m {
        fmt.Printf("key = %s,value =%v\n",k,v)
    }
}

func pase_map()  map[string]*student{
    m :=make(map[string]*student)
    stu :=[]student{{"joy",12},{"lei",14}}
    for _,v :=range stu{
        m[v.Name]=&v
    }
    return m
}
```

**for range** ：**只能被用于读取数据**，而不能用于获取数据指针和修改数据值的操作。这是由for range实现原理决定的。for range中的V，是一个值类型，每下滑一个数据，就把那个数据位的值输入到V中，因此不涉及到数据本身的操作。

### 使用两个协程并发输出1-20

```go
package main

func main() {
    ch := make(chan int)
    exit := make(chan int)
    go func() {
        for i := 1; i <= 20; i++ {
            println("g1:", <-ch)
            i++
            ch <- i
        }
    }()
    go func() {
        defer func() {	// 在子协程中 使用defer 当子协程死亡时 关闭exit
            close(ch)
            close(exit)
        }()
        for i := 0; i < 20; i++ {
            i++
            ch <- i
            println("g2:", <-ch)
        }
    }()
    <-exit	// 这种写法优于time.sleep
}
```




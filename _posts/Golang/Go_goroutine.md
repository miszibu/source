---
title: Go_goroutine
date: 2018-06-04 10:42:55
tags: [Go]

---

### Goroutine ——协程 ###

OS将程序分为了进程和线程来提高程序的CPU利用率，一个进程拥有多个线程，多个线程共享同一份内存空间来实现线程间通信。

**协程**，可以看做**用户级线程**（更为轻量级的线程），这个概念是相对于OS **系统级线程**所提出的，协程的切换可以由用户控制，而线程只能由操作系统调度。

<!--more-->

Go中的协程又有所不同，Go的协程不由代码控制，而是Go实现了一个**协程调度器**，自动**实现协程的切换**。而且Go的**协程**通信是由**信道channel**实现的

**守护线程daemon**：在主goroutine调用其他goroutine后，当主协程死亡，其他的子协程也直接死亡。这在一定意义上类似于Java的daemon守护线程了。

**协程间通信**：Java中的并发主要是靠锁住临界资源（共享内存）来保证同步的。而**channel**则是goroutines中通信的方法。可以把channel看做一个传送带，实际上是一个**有类型的消息队列**，FIFO。利用Channel来实现协程同步，是Go语言的优势所在，原生支持语言级并发。

### Goroutine与调度器###

go中不能自己创建用户级别线程，所有的都是Goroutine协程，Golang的协程调度器来维护协程在系统级线程上的运行。

Go的调度器分为三个重要的结构：M P G

* **M**：代表OS的系统级线程，实际上运行goroutine的地方
* **P**：Processor，主要用于维护goroutine列表并执行goroutine。存储了所有需要P来执行的gotoutine列表。还有一个**全局列表**。
* **G**：Goroutine，G维护了goroutine需要的栈、程序计数器以及它所在的M等信息。 

![go_goroutine](../img/go_goroutine.jpg)

这张图描绘了三者的关系，一个M（OS系统级线程）对应一个P(Processor协程调度器)，并执行一个G(Goroutine)，P维护了一个G的队列。

* **线程阻塞**：当一个线程阻塞时，P会携带G队列转投其他线程来执行。
* **线程空闲**：当一个线程空闲时，它会向全局的P获取G，当无法获取G时，就会从其他的P处偷取G，直接偷取一半的G。
* **线程不足**：当M的对象不足导致P阻塞时，会启动一个新的工作线程以充分利用cpu资源。

### Goroutine主动退出线程占有###

在OS中，一个线程就是一个栈和相关资源。操作系统通过线程的栈来保存线程的运行信息，go通过创建goroutine来降低线程创建和上下文切换的开销。

每个goroutine维护一个**存放在堆中的运行状态栈**，来模拟OS的线程栈。

再创建一个scheduler来运行维护Goroutine，然而Go中的协程**不具备主动退出的能力**，因此调度器需要自己主动来确定协程的切换。主要在一下两个情况下，goroutine会主动退出。

* Go将能够异步并发实现的操作，例如文件访问，网络访问等通过异步的操作进行了封装，当Goroutine运行到了**异步操作**时，便主动退出。
* Goroutine调用了一个无法异步的会导致blocking的System call，这时候Go只能将这个操作挪到一个真正的线程中去执行，自己再切换协程。
* 死循环，当Goroutine运行死循环时，在1.5以后的版本，go若检测到这类循环，会将它设置为可抢占式，以避免占死一个线程的情况。

### 相关资料###

[理解Go的Goroutine和channel](https://www.cnblogs.com/damumu/p/7320195.html)

[goroutine与调度器](http://skoo.me/go/2013/11/29/golang-schedule?hmsr=studygolang.com&utm_medium=studygolang.com&utm_source=studygolang.com)

[Golang 的 goroutine 是如何实现的？](https://www.zhihu.com/question/20862617)


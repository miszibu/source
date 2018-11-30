---
title: Go_异常处理
date: 2018-06-12 17:04:55
tags: [Go]

---

Go语言的设计者认为，将异常和控制语句放在一起会使代码混乱。在Go中，多使用返回值来返回错误，只有在真正遇到错误时（比如除数为0时），才使用Go中引入的Exception处理，即defer panic recover

<!--more-->

```go 
func main() {
	defer func() {
		fmt.Println("zibu")
	}()
	defer func() {				// defer声明异常接受器，并进行处理
		if r:=recover();r!=nil{
			fmt.Println(r)
		}
	}()
	panic("i am died")			// panic 调出异常
}
i am died
zibu
```

**Panic**：用于表示严重不可恢复的错误，它是Go语言的一个内置函数，接受一个interface{}类型的值作为参数（也就是**接受任何值**）。Panic在没有recover的情况下会**导致程序挂掉**，然后**打印出当前的调用栈**。

​	panic执行完后，调用当前func的defer，若defer 栈也运行完，panic再向上传递。

**Recover**：recover也是内置函数，在defer中调用recover，若捕获到当前的panic，被捕获的panic事件就**不会向上冒泡**。注意：当recover之后，不会紧接panic之后的代码继续运行，而是**从recover处继续向下执行**。且使用recover处理panic指令，必须利用defer在panic之前声明，否则当panic时，recover无法捕捉。

**Defer**：defer会在func结束后调用，而且defer的存放是个**defer栈，FILO**。defer在的优点是可以让内存占用的释放可以在创建后显示，更直接。
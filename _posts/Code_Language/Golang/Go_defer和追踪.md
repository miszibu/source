---
title: Go-defer和追踪
date: 2018-06-04 14:04:55
tags: [Go]
---



关键词defer允许我们推迟到函数返回之前（既**函数return语句之后**）一刻才执行某个语句或函数。类似于JAVA中的finally语句块，它一般用于**释放某些资源**。

之所以有defer的语法是为了代码更加的简洁，易读。相当于一个**语法糖**（在创建资源后，就写好释放资源的代码）

defer的特点如下：

* 在函数return之后才会执行
* 当有多个defer执行时，以逆序执行（类似栈，即后进先出）

<!--more-->

```go
func f() {
	for i := 0; i < 5; i++ {
		defer fmt.Printf("%d ", i) //将会逆序执行
	}
}
//上面的代码会输出4 3 2 1 0
```

### defer的实际应用###

**1.关闭文件流**

```go
inputFile, inputError := os.Open("input.dat")
defer inputFile.Close()
```

**2.解锁加锁的资源**

```go
mu.lock()
defer mu.Unlock()
```

**3.关闭 数据库连接**

```go
defer disconnectFromDB()
```

**4.追踪函数**

```go
func trace(s string) string {
	fmt.Println("entering:", s)
	return s
}
func un(s string) {
	fmt.Println("leaving:", s)
}
func a() {
	defer un(trace("a"))//defer只会defer最外层的函数 里面还是会先序执行
	fmt.Println("in a")
}
func b() {
	defer un(trace("b"))
	fmt.Println("in b")
	a()
}
func main() {
	b()
}
/* 
entering: b
in b
entering: a
in a
leaving: a
leaving: b*/
```

**5.使用 defer 语句来记录函数的参数与返回值** 

```go
func func1(s string) (n int, err error) {
	defer func() {
		log.Printf("func1(%q) = %d, %v", s, n, err)
	}()
	return 7, io.EOF
}

func main() {
	func1("Go")
}
```


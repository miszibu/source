---
title: Go-Interface和反射
date: 2018-06-04 14:27:55
tags: [Go,Mark]

---

###Go中的接口###
Go不是传统的面向对象的编程语言，它里面没有类和继承的概念。
但是Go语言中的**接口**概念可以很方便的实现许多面向对象的特性。
```go
type Namer interface {
    Method1(param_list) return_type
    Method2(param_list) return_type
    ...
}
```
<!--more-->
```go
type Shaper interface {
	Area() float32
}

type Square struct {
	side float32
}

func (sq *Square) Area() float32 {
	return sq.side * sq.side
}

type Rectangle struct {
	length, width float32
}

func (r Rectangle) Area() float32 {
	return r.length * r.width
}

func main() {

	r := Rectangle{5, 3} // Area() of Rectangle needs a value
	q := &Square{5}      // Area() of Square needs a pointer
	// shapes := []Shaper{Shaper(r), Shaper(q)}
	// or shorter
	shapes := []Shaper{r, q}
	fmt.Println("Looping through shapes for area ...")
	for n, _ := range shapes {
		fmt.Println("Shape details: ", shapes[n])
		fmt.Println("Area of this shape is: ", shapes[n].Area())
	}
}
```
一个接口可以被多个类同时实现，接口具有多态性。
多态是OOP编程中的概念：同一个类型在不同的实例上表现出不同的行为。

类通过实现接口，来继承接口，注意如果在实现接口时，不加*，实际上是值传递，不能改变类的数据。

###接口嵌套接口###
一个接口可以包含一个或多个其他的接口。
```go
type ReadWrite interface{
	Read(b Buffer) bool
	Write(b Buffer) bool
}
type Lock interface{
	Lock()
	Unlock()
}
type File interface{
	ReadWrite
	Lock
	Close()
}
```
###类型断言###

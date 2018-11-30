---
title: Go-语言基础
date: 2018-5-29 14:38:15
tags: [Go]

---

### linux安装Go环境

```bash
yum install mercurial
yum install git
yum install gcc
wget https://golang.google.cn/dlc/具体包名
tar -zxvf go1.2.linux-amd64.tar.gz 
vim /etc/profile
//配置GO 全局变量 
//Gopath是个人Go目录地址
//Goroot是Go安装地址
export GOROOT=/usr/local/go
export PATH=$GOROOT/bin:$PATH
export GOPATH=/usr/zibu/go
```

<!--more-->

### Go语法格式一览

```go
//当前程序的包名
package main
//导入其他包
import "fmt"
//常量定义
const PI  = 3.14
//全局变量的声明和赋值
var name = "this  a var "
//一般类型声明
type newType int;
//结构的声明
type newStruct struct {}
//接口的申明
type golang interface {}
//main函数作为程序入口点启动
func main(){
	fmt.Println("hello zibu")
}
```



Go 程序是通过 **package** 来组织的。

只有 package 名称为 main 的包可以包含 main 函数。

一个可执行程序有且仅有一个 **main** 包。

通过 **import** 关键字来导入其他非 **main** 包。

```go
//通过 import 关键字单个导入
import "fmt"
import "io"
//同时导入多个:
import {
    "fmt",
    "io"
}
```

使用 <PackageName>.<FunctionName> 调用:

```go
package 别名：
// 为fmt起别名为fmt2
import fmt2 "fmt"
// 调用的时候只需要Println()，而不需要fmt.Println()


//省略调用(不建议使用):
//前面加个点表示省略调用，那么调用该模块里面的函数，可以不用写模块名称了
import . "fmt"
func main (){
    Println("hello,world")
}
```

 **const** :常量

**var**: 函数体外部使用var关键字来进行全局变量的声明和赋值

**type**:使用type来声明结构(struct)和接口(interface)

**func** :声明函数

可见性规则

Go语言中，使用大小写来决定该常量、变量、类型、接口、结构或函数是否可以被外部包所调用。

函数名**首字母小写**即为 **private** :

```
func getId() {}
```

函数名**首字母大写**即为 **public** :

```
func Printf() {}
```

### Go语言数据类型

```go
var isBool bool = true			//bool类型 默认false
var isInt int = 1			    //int类型
var isFloat32 float32 = 1.1		//float32
var isFloat64 float64 = 1.1		//float64
//1.9版本对于数字类型 无需声明 系统自动识别
```

### GO语言变量

```go
var v_name v_type    //默认声明 给与一个默认初始值
var v_name = v_value //自动根据赋值判定数据类型 
v_name ：= v_value	//省略var的语法
```

```go
var x, y int
var (  // 这种因式分解关键字的写法一般用于声明全局变量
    a int
    b bool
)

var c, d int = 1, "zibu"

//这种不带声明格式的只能在函数体中出现
//g, h := 123, "hello"

func main(){
    g, h := 123, "hello"
    println(x, y, a, b, g, h)
}
```

 **值类型**：int,float,bool,string等类型，使用 **=** 赋值时，是**深度copy**

使用**&i** 来获取i变量的内存地址，复杂数据一般使用**引用类型**，引用类型的赋值是地址复制，即浅拷贝。

### Go语言常量

**const**:上文中已经介绍 就不赘述

**iota**：特殊常量，可以认为是一个可以被编译器修改的常量

在每一个const关键字出现时，被重置为0，然后再下一个const出现之前，每出现一次iota，其所代表的数字会自动增加1。

```go
package main

import "fmt"

func main() {
    const (
            a = iota   //0
            b          //1
            c          //2
            d = "ha"   //独立值，iota += 1
            e          //"ha"   iota += 1
            f = 100    //iota +=1
            g          //100  iota +=1
            h = iota   //7,恢复计数
            i          //8
    )
    fmt.Println(a,b,c,d,e,f,g,h,i)
}
//0 1 2 ha ha 100 100 7 8

import "fmt"
const (
    i=1<<iota
    j=3<<iota
    k
    l
)

func main() {
    fmt.Println("i=",i)
    fmt.Println("j=",j)
    fmt.Println("k=",k)
    fmt.Println("l=",l)
}
/*i= 1
j= 6
k= 12
l= 24*/
```

```GO
a = "hello"
unsafe.Sizeof(a)
//输出结果为：16
```

字符串类型在 go 里是个结构, 包含指向底层数组的指针和长度,这两部分每部分都是 8 个字节，所以字符串类型大小为 16 个字节。

### Go语言运算符

* 算数运算符 + - * / % ++ --
* 关系运算符 == != >   <    >=  <=
* 逻辑运算符 && || !
* 位运算符 &(位与) |(位或) ^(位异或) <<(左移运算符)  >>(右移运算符)
* 赋值运算符 = += -= *= /= %= <<= >>= &=  ^= |=
* 其他运算符 &返回变量存储地址  *指针变量

### Go语言条件语句

* if else
* switch 

```go
 switch marks {//marks可以为任何类型
      case 90: grade = "A"//成员需要是相同类型
      case 80: grade = "B"
      case 50,60,70 : grade = "C"
      default: grade = "D"  
   }
//不需要使用break 默认break
```

### Go语言循环语句

* for 
* break 
* continue
* goto :不建议使用

### Go语言函数

Go语言最少有个main()函数

### Go语言变量作用域

* 局部变量  //局部变量与全局变量重名，局部变量覆盖全局变量
* 全局变量  
* 形式参数   //函数的局部变量

### Go语言数组

```go
var varName [size] varType //Go的类型声明都是如此 与Java相反
var balance = [5]float32{1000.0, 2.0, 3.4, 7.0, 50.0}//可以设置数组长度
var balance = []float32{1000.0, 2.0, 3.4, 7.0, 50.0}//也可以不设置数组长度
 var a = [5][2]int{ {0,0}, {1,2}, {2,4}, {3,6},{4,8}}//初始化二维数组
```

### Go语言指针

```go
func main() {
   var a int= 20   /* 声明实际变量 */
   var ip *int        /* 声明指针变量 */
   ip = &a  /* 指针变量的存储地址 */
   fmt.Printf("a 变量的地址是: %x\n", &a  )/* 指针变量的存储地址 */
  
   fmt.Printf("ip 变量储存的指针地址: %x\n", ip ) /* 指针变量的存储地址 */

   fmt.Printf("*ip 变量的值: %d\n", *ip )   /* 使用指针访问值 */
}
/*a 变量的地址是: 20818a220
ip 变量储存的指针地址: 20818a220
*ip 变量的值: 20*/
```

空指针为：**nil**

### Go语言结构体

```go
type struct_variable_type struct {
   member definition;
   member definition;
   ...
   member definition;
}//定义一个结构体
variable_name := structure_variable_type {value1, value2...valuen}//创建结构体变量，并赋值
fmt.Printf(variable_name.value1)//访问子元素
```

结构体指针

```go
var struct_pointer *Books	//定义结构体指针
struct_pointer = &Book1		//指针变量  存储结构体变量地址
struct_pointer.title 		//结构体指针访问结构体成员 使用.操作符
```

### Go语言切片

Go语言切片是对数组的抽象。

Go**数组的长度不可改变**，在特定场景中需要一个变长的集合，既为**切片**。

Go数组的传递是值传递，而非

切片实际的存储是 ***array  length capacity** 指向数组的指针 长度 容量

```go
var identifier []type	//声明一个未制定大小的数组来定义切片
var slice1 []int = make([]int,length)//声明一个制定长度的切片
slice2:=make([]int,length,capacity)  //增加容量声明，容量是容器的大小，长度是数据的长度，容量>=数据

fmt.Printf("len=%d cap=%d slice=%v\n",len(x),cap(x),x)//使用len() cap()函数获取切片长度和容量

/* 创建切片 numbers1 是之前切片的两倍容量*/
numbers1 := make([]int, len(numbers), (cap(numbers))*2)
/* 拷贝 numbers 的内容到 numbers1 */
copy(numbers1,numbers)
```

### Go 语言范围(Range)

Go 语言中 range 关键字用于for循环中迭代数组(array)、切片(slice)、通道(channel)或集合(map)的元素。在数组和切片中它返回元素的索引值，在集合中返回 key-value 对的 key 值。

```Go
package main
import "fmt"
func main() {
    //这是我们使用range去求一个slice的和。使用数组跟这个很类似
    //使用range将传入index和值变量，由于该例子无需序号则施工空白符“_”省略。
    nums := []int{2, 3, 4}
    sum := 0
    for _, num := range nums {
        sum += num
    }
    //range也可以用在map的键值对上。
    kvs := map[string]string{"a": "apple", "b": "banana"}
    for k, v := range kvs {
        fmt.Printf("%s -> %s\n", k, v)
    }
    //range也可以用来枚举Unicode字符串。第一个参数是字符的索引，第二个是字符（Unicode的值）本身。
    for i, c := range "go" {
        fmt.Println(i, c)
    }
}
sum: 9
index: 1
a -> apple
b -> banana
0 103
1 111
```

### Go语言Map（集合）

**Map:无序的键值对的集合**，通过key来快速检索数据。

```Go
/* 声明变量，默认 map 是 nil */
var map_variable map[key_data_type]value_data_type

/* 使用 make 函数 */
map_variable := make(map[key_data_type]value_data_type)

func main() {
    /* 创建map */
    countryCapitalMap := map[string]string{"France": "Paris", "Italy": "Rome", "Japan": "Tokyo", "India": "New delhi"}
    /* 遍历map */
    for country := range countryCapitalMap {
        fmt.Println(country, "首都是", countryCapitalMap [ country ])
    }
    /*删除元素*/ 
    delete(countryCapitalMap, "France")
}
```

### Go语言递归函数

**阶乘**：注意设置退出条件 否则将死循环

```go
func Factorial(n uint64)(result uint64) {
    if (n > 0) {
        result = n * Factorial(n-1)
        return result
    }
    return 1
}

func main() {  
    var i int = 15
    fmt.Printf("%d 的阶乘是 %d\n", i, Factorial(uint64(i)))
}
```

**斐波那契数列**

```go
func fibonacci(n int) int {
  if n < 2 {
   return n
  }
  return fibonacci(n-2) + fibonacci(n-1)
}

func main() {
    var i int
    for i = 0; i < 10; i++ {
       fmt.Printf("%d\t", fibonacci(i))
    }
}
```

### Go语言类型转换

```go
type_name(expression) 	//类型转换格式
var sum int = 17
var count int = 5
var mean float32
   
mean = float32(sum)/float32(count)
```

### Go错误处理

http://www.runoob.com/go/go-error-handling.html



### 相关引用

[Go语言教程——菜鸟教程](http://www.runoob.com/go/go-recursion.html)
---
title: Go-struct
date: 2018-06-04 14:27:55
tags: [Go]

---



* 结构体的三种声明方式，直接声明，new（Struct）,混合字面量
* 结构体的内存布局——Go的结构体数据以连续块的形式存放在内存中
* 递归结构体——可以引用自身来定义，方便树和链表的建立
* 结构体工厂
* 带标签的结构体
* 匿名字段
* 结构方法

<!--more-->

### 结构体三种声明方式——直接声明，new,混合字面量

```go 
type Person struct {
    firstName   string
    lastName    string
}

func upPerson(p *Person) {//接收一个person的引用类型
    p.firstName = strings.ToUpper(p.firstName)
    p.lastName = strings.ToUpper(p.lastName)
}

func main() {
    // 1-struct as a value type:值类型 变量本身
    var pers1 Person 
    pers1.firstName = "Chris"
    pers1.lastName = "Woodward"
    upPerson(&pers1)
    fmt.Printf("The name of the person is %s %s\n", pers1.firstName, pers1.lastName)

    // 2—struct as a pointer:使用new返回的是指向已分配内存的指针
    pers2 := new(Person)
    pers2.firstName = "Chris"
    pers2.lastName = "Woodward"
    (*pers2).lastName = "Woodward"  
    pers2.lastName = "Woodward"	    //指针类型可以不解引用 直接访问子元素
    upPerson(pers2)
    fmt.Printf("The name of the person is %s %s\n", pers2.firstName, pers2.lastName)

    // 3—struct as a literal:结构体字面量 混合字面量语法是一种简写，底层任然会调用new 因此返回的仍然是内存指针
    pers3 := &Person{"Chris","Woodward"}
    upPerson(pers3)
    fmt.Printf("The name of the person is %s %s\n", pers3.firstName, pers3.lastName)
}
```

输出： 

```
The name of the person is CHRIS WOODWARD
The name of the person is CHRIS WOODWARD
The name of the person is CHRIS WOODWARD
```

### 结构体的内存布局

Go语言中，结构体和它所包含的数据在内存中是以连续块的形式存在的，即便结构体中嵌套着其他的结构体，这在性能上带来了很大的优势。

Java中的引用类型，一个对象和它里面包含的对象可能会在不同的内存空间中，与Go的指针类型类似。

### 递归结构体

结构体类型可以通过引用自身来定义。这就方便了链表和树的定义。

```go
type Node struct{//链表定义
    data int 
    next *Node
}
type treeNode struct{//树节点定义
    le *treeNode
    ri *treeNode
    data value
}
```

 ### 结构体工厂

```go
type File struct{//定义一个结构体file
    fd int 
    name string
}
func NewFile(fd int, name string) *File{//结构体创建工厂方法
    if fd<0{
    	return nil        
    }
    return &File(fd, name)//输入参数 返回一个指向结构体实例的指针
}
f := NewFile(100,"./zibu.txt") //调用工厂方法
size := unsafe.Sizeof(T{})

```

### 带标签的结构体

结构体中的字段除了有名字和类型外，还有一个可选的标签tag。标签无法通过.选择符直接访问，只能通过反射机制来获取。可以利用tag存储一些隐藏性描述数据，在代码中反射调用出来。

```go
type TagType struct { // tags
    field1 bool   "An important answer"
    field2 string "The name of the thing"
    field3 int    "How much there are"
}
func main() {
    tt := TagType{true, "Barak Obama", 1}
    for i := 0; i < 3; i++ {
        refTag(tt, i)
    }
}

func refTag(tt TagType, ix int) {
    ttType := reflect.TypeOf(tt)
    ixField := ttType.Field(ix)
    fmt.Printf("%v\n", ixField.Tag)
}
```

### 匿名字段

每个结构体都可以拥有匿名字段，每个数据类型最多可以出现一次。当然结构体可以内嵌。

```
type outerS struct {
    b    int
    c    float32
    int  // anonymous field
    innerS //anonymous field
}
```

当内嵌结构体时，可能会出现命名冲突需要注意。

### 方法

```go
func (structNew *StructNew) method(){
    // 方法名前的就是方法所属的类
}
```




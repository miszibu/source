---
title: Go-Bug记录
date: 2018-06-03 14:04:55
tags: [Go]

---



此处记录日常业务编写中所出现的Bug并加以记录，以作为以后快速解决Bug的查询数据库

<!--more-->

### error 216

```go 
//main文件里面的包一定要是main package main即可
CreateProcess failed with error 216: 

```

***

### 协程调度器导致的输出乱序

```go
func main() {
   ch := make(chan int)
   for i := 1; i <= 20; i++ {
      go func() {
         //lock := sync.Mutex{}
         //lock.Lock()
         fmt.Println(<-ch)
         //lock.Unlock()
      }()
      ch <- i
   }
}
```

在不加锁的情况下，上述代码会出现输出乱序的现象。

 这个问题 是go的**协程调度器**产生的 ：当主协程往通道里写入数据时，此时子协程1（即会输出1的协程）创建但还没有执行输出。然后 主协程继续创建了子协程2（即会输出2的协程）。因此在个别协程上出现了偶尔的乱序。比如2在1前面输出，实际上是子协程1先于子协程2创建。然后子协程2先于子线程1获得运行。

***

### invalid memory address or nil pointer dereference

```go
res, err := http.Get(picUrl)
defer res.Body.Close()
if err != nil {
	log.Error("访问图片失败",err)
}

// 错误的内存地址或空指针引用
panic: runtime error: invalid memory address or nil pointer dereference
[signal 0xc0000005 code=0x0 addr=0x40 pc=0x5e5ae1]
```

> defer 是先访问代码相关的变量参数，但是不执行，当函数执行时执行

```go
// 先判断err 再执行defer就可以解决这个问题
res, err := http.Get(picUrl)
if err != nil {
	log.Error("访问图片失败",err)
}
defer res.Body.Close()
```

### 不要使用fmt.Sprintf做非字符串类型的类型转换

**fmt.Sprintf**可以用于格式化任意参数，非常好用。但是golang可不支持重载功能，而是使用interface{}来接受任意类型的参数。因此与相应的标准库函数相比，fmt.Sprintf 需要更大的开销。

关于golang不支持重载，这各有说法吧，我认为影响不大。

> https://golang.org/doc/faq#overloading 官方对于不支持重载的解释:  

```go
//int to string
var num int64 = 10000
fmt.Sprintf("%d",num)
strconv.FormatInt(num,10)

//hexadecimal to string
var s = "123456"
func makeMd5(data string) []byte{
    md5Hash := md5.new()
    md5Hash.Write(data)
    //io.WriteString(md5Hash,data)
    return md5Hash.Sum(nil)
}
hexByte := makeMd5(s)
fmt.Sprintf("%x",hexBytes)
hex.EncodeToString(hexBytes)
```


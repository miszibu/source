---
title: Go-Bug记录
date: 2018-06-03 14:04:55
tags: [Go]

---



此处记录日常业务编写中所出现的Bug并加以记录，以作为以后快速解决Bug的查询数据库

<!--more-->



```go 
//main文件里面的包一定要是main package main即可
CreateProcess failed with error 216: 

```

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
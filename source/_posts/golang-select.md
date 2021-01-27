---
title: Go语言并发编程笔记（十）：Select 的使用
date: 2021-01-21 15:56:17
tags:
- Go语言
categories:
- coding
---

如果要同时与多个 channel 发生交互，可以借助 select 语句。

<!--more-->

先来看看一个 select 的简单例子：
```go
start := time.Now()
c := make(chan interface{})
go func() {
    time.Sleep(5*time.Second)
    close(c)
}()

fmt.Println("Blocking on read...")
select {
case <-c:
    fmt.Printf("Unblocked %v later.\n", time.Since(start))
}
```
运行这段程序，会输出：
```
Blocking on read...
Unblocked 5.004568624s later.
```

以上这个例子，虽然并没体现出「同时与多个 channel」进行交互，但可以看出使用 select 语句的基本模式。select 语句与网络编程中 I/O 多路复用的 select 技术很像，也是发生一些事件（例如 channel 可读）时进行一些操作，利用 select 的好处是一个 goroutine 可以并发与多个 channel 进行交互。

可能会对使用 select 产生一些疑问，例如：
- 如果 select 中的多个 channel 同时可读，会发生什么事？
- 如果 select 中的各个 channel 都不就绪怎么办？有没有超时机制？
- 在等待 channel 就绪的过程中，能不能干别的事情？

以下通过一些例子来看看这三个问题。

先来看看如果有多个 channel 同时就绪，select 会如何处理：
```go
package main

import (
    "fmt"
)

func main() {
    c1 := make(chan interface{})
    close(c1)
    c2 := make(chan interface{})
    close(c2)

    var c1Count, c2Count int
    for i := 1000; i > 0; i-- {
        select {
        case <-c1:
            c1Count++
        case <-c2:
            c2Count++
        }
    }

    fmt.Printf("c1Count: %d\nc2Count: %d\n", c1Count, c2Count)
}
```
运行以上程序，会输出:
```
c1Count: 508
c2Count: 492
```
以上例子使用可以对已关闭的 channel 多次反复读取的特性来模拟 channel 读就绪的情况。可以看出当多个 channel 同时就绪的时候，select 会较均衡地平均分对各个 channel 的读操作。

如果一直没有可读的 channel 就绪，还可以在 select 中应用上超时机制：
```go
var c <-chan int
select {
case <-c:
case <-time.After(1 * time.Second):
    fmt.Println("Timed out.")
}
```
这样就可以避免一直无限地等待 channel 就绪了。

接下来的例子看看如果所有的 channel 都未就绪，还可以做别的事情：
```go
start := time.Now()
var c1, c2 <-chan int
select {
case <-c1:
case <-c2:
default:
    fmt.Printf("In default after %v\n", time.Since(start))
}
```

select 会配合 for 循环使用，这是一种常用的模式：
```go
for {    // 可能是无限循环，或者是迭代遍历某些东西
    select {

    }
}
```

例如：
```go
for _, s := range []string{"a", "b", "c"} {
    select {
    case <- done:
        return
    case stringStream <- s:
    }
}
```

再例如：
```go
for {
    select {
    case <-done:
        return
    default:
        // 这里可以做些别的事情
    }

    // 这里可以做一些别的事情
}
```

---
参考资料：
- 《Goncurrency in Go》第三、四章，作者 Katherine Cox-Buday

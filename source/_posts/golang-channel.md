---
title: Go语言并发编程笔记（九）：Channel 的使用
date: 2021-01-19 10:05:00
tags:
- Go语言
categories:
- coding
---

本文关于 Go 语言中一个独特的数据结构——Channel，Channel 是 Go 语言并发编程的利器。

<!--more-->

## Channel 的使用场景

Channel 用于在不同 goroutine 之间交流数据，可以说是搭建在不同 goroutine 之间的管道。
使用 Channel 符合 Go 语言的理念：「Don’t communicate by sharing memory, share memory by communicating.」

## 如何使用 Channel

### 声明和初始化

使用 channel 之间首先要进行声明和初始化：
```go
var dataStream chan interface{}
dataStream = make(chan interface{})
```
以这种方式声明的话，可以往管道里面写入任何类型的数据。
也可以限制写入特定类型的数据：
```go
intStream := make(chan int)
stringStream := make(chan string)
```

### 通过 channel 传送数据

利用 channel 进行数据交互的示例：
```go
stringStream := make(chan string)
go func() {
    stringStream <- "Hello channels!"
}()
fmt.Println(<-stringStream)
```
注意一点是，从 channel 中读取数据是阻塞式读取的，所以以上的程序中主 goroutine 不需要使用 WaitGroup 等待别的 goroutine 结束。

### 关闭 channel

可以对 channel 进行关闭操作：
```go
intStream := make(chan int)
close(intStream)
integer, ok := <- intStream
fmt.Printf("(%v): %v", ok, integer)
fmt.Printf("(%v): %v", ok, integer)
```
运行以上程序会输出:
```
(false): 0
(false): 0
```
从这个例子中可以看出（1）从 channel 中读取数据时，有第二个返回值，它是一个布尔类型，表示管道是否被关闭；（2）可以对已关闭的 channel 进行读取，并且可以读取多次。

如果 channel 的生产方对 channel 进行关闭，那么 channel 的消费方可以使用一个 for 循环：
```go
intStream := make(chan int)
go func() {
    defer close(intStream)
    for i := 1; i <= 5; i++ {
        intStream <- i
    }
}()

for val := range intStream {
    fmt.Printf("%v ", $val)
}
```

可以改造一下以上的例子，一个 goroutine 生产数据，两个 goroutine 消费数据：
```go
var wg sync.WaitGroup
intStream := make(chan int)

wg.Add(3)
go func() {
    defer close(intStream)
    defer wg.Done()
    for i := 1; i <= 30; i++ {
        intStream <- i
    }
}()

readFromStream := func() {
    defer wg.Done()
    for val := range intStream {
        fmt.Printf("%v ", val)
    }
}

go readFromStream()
go readFromStream()

wg.Wait()
```

### 指定 channel 的容量

如果不指定 channel 的容量，例如上文那样子：
```go
intStream := make(chan int)
```
此时，初始化的 channel 是一个「unbuffered channel」。数据的生产方要等待消费方把数据从 channel 取走后，写入才会结束，不然会一直被阻塞。

在初始化时，第二个参数指定 channel 的容量：
```go
intStream := make(chan int, 5)
```
如果 channel 的容量未被占满，那么生产方写入时会马上返回；如果 channel 被填满，生产者往 channel 写数据也会阻塞，所以要给 channel 指定一个合适的容量。

### 单向 channel
可以声明一个单向 channel：
```go
// 只读 channel
var readOnlyChan <-chan int
// 只写 channel
var writeOnlyChan chan<- int
```

单向 channel 的意义是防止对 channel 误读或误写。例如将上文出现过的例子改一改，应用上单向 channel：
```go
var wg sync.WaitGroup
intStream := make(chan int)

wg.Add(3)
go func(ch chan<- int) {
    defer close(ch)
    defer wg.Done()
    for i := 1; i <= 30; i++ {
        ch <- i
    }
}(intStream)

readFromStream := func(ch <-chan int) {
    defer wg.Done()
    for val := range ch {
        fmt.Printf("%v ", val)
    }
}

go readFromStream(intStream)
go readFromStream(intStream)

wg.Wait()
```

### 对不同状态的 channel 进行操作会发生什么事

![](/images/golang-channel/01.png)

## 使用 channel 时的注意事项

### 尽量考虑把 channel 封装起来

从上文的图可以发现，对 channel 的误用可能导致死锁或 panic，所以有必要地对 channel 进行一些封装以避免发生意外。

先引入一个概念：ownership，或者称为「channel 的主人」，意思就是找一个单一的 goroutine 做以下的事情：
- 初始化 channel
- 进行写操作，或者传递给别的 goroutine 进行写操作
- 关闭 channel
- 将以上三种行为封装起来，并暴露出「只读 channel」给别的 goroutine 使用

举一个简单的例子:
```go
chanOwner := func() <-chan int {
    // channel 的主人负责初始化 channel
    resultStream := make(chan int, 5)
    go func() {
        // 这里不是由 channel 的主人亲自进行关闭，但由于这里是主人自身的匿名函数，只有一个生产者，问题不大
        defer close(resultStream)
        for i := 0; i <= 5; i++ {
            resultStream <- i
        }
    }() // 将 channel 的写入交给别的 goroutine

    // channel 的主人暴露一个只读的 channel 给外界消费
    return resultStream
}

resultStream := chanOwner()
for result := range resultStream {
    fmt.Printf("Received: %d\n", result)
}
fmt.Println("Done receiving!")
```

这样的做法可以避免很多麻烦：
- 避免有 goroutine 往一个未初始化的 channel 写数据
- 避免关闭一个未初始化的 channel
- 避免往一个已关闭的 channel 写数据
- 避免对一个 goroutine 进行多次关闭
- 防止对 channel 误写数据

### 传入 channel 的数据是复制传递

举个例子:
```go
package main

import (
    "fmt"
    "sync"
)

type person struct {
    name string
    age int
}

func main() {
    var wg sync.WaitGroup
    ch := make(chan person, 5)
    p := person{"Tom", 16}
    p.name = "Tom"
    p.age = 16
    ch <- p

    wg.Add(1)
    go func(c chan person, w *sync.WaitGroup) {
        defer w.Done()

        p := <- c
        p.name = "Jack"
        p.age = 36
    }(ch, &wg)

    wg.Wait()
    fmt.Printf("%v\n", p.name)
    fmt.Printf("%v\n", p.age)
}
```
运行以上程序，输出的是：
```
Tom
16
```
从这个例子就可以看出，数据是以复制的方式写入 channel 的。因为 channel 本身的理念就是要避免在 goroutine 之间共享使用相同的数据内存。

----
参考资料：
- 《Goncurrency in Go》第三章，作者 Katherine Cox-Buday

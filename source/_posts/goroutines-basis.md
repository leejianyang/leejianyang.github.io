---
title: Go语言并发编程笔记（三）：Goroutine 基础
date: 2021-01-13 11:56:11
tags:
- Go语言
categories:
- coding
---

本文是关于 Goroutine 的基础知识。

<!--more-->

## Goroutine 的本质

在 Go 语言中借助 goroutine 编写并发代码，本质上还是多线程编程，因为在操作系统的概念上来说，线程是 CPU 调度的最基本单位了，Go 语言也无法超越操作系统的基础来玩出什么新花样。

但是这并不等同于说一个 goroutine 就是一个线程。粗浅地说，一个 Go 编写的进程会按需创建多个线程，但是这个线程数是不会无限膨胀的，具体最多会开多少个线程视乎运行该程序的机器的 CPU 的实际情况而定。举个例子说，进程开启了两个线程，分别称为 t1 和 t2。该程序创建了 8 个 goroutine，分别称为 g1，g2，g3，g4，g5，g6，g7 和 g8，在代码逻辑中这 8 个 goroutine 是期望并发运行的。那可能发生的情况就是 g1、g2、g3 和 g4 在线程 t1 上运行；而线程 g5、g6、g7 和 g8 在线程 t2 上运行；从操作系统的角度来看，这个程序就是 2 个线程并发运行，从程序员的角度来看，就是 8 个 goroutine 并发运行。那 Go 语言的 runtime 要做的事情就是，利用 t1 交替运行 g1、g2、g3 和 g4；在 t2 交替运行 g5、g6、g7 和 g8。goroutine 是非抢占式的，它决定不了自己什么时候运行，它也无需在运行期间自发让渡出运行权，Go 的 runtime 会根据一个 goroutine 运行的行为自动地决定什么时候让其暂停下来，调度别的 goroutine 来运行。

## Goroutine 的基本使用
```go
package main

import "fmt"

func main() {
    sayHello := func() {
        fmt.Println("hello")
    }

    go sayHello()
}
```

以上这个例子就是创建了一个 goroutine 来运行 sayHello 函数。但这个例子跟传统的多线程编程的情况相似，有一个严重的问题，基本上 hello 并不会被打印出来，因为没等到 sayHello 的 goroutine 被调度执行，主 goroutine 就执行完毕了，整个程序就退出了。

有很多方法可以解决这个问题，其中之一就是借助 WaitGroup：
```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var wg sync.WaitGroup
    sayHello := func() {
        defer wg.Done()
        fmt.Println("hello")
    }

    wg.Add(1)
    go sayHello()
    wg.Wait()
}
```

以上代码的大致意思就是，使用 WaitGroup 来记录当前有多少个未结束的 goroutine，代码执行到 Wait() 的时候，若当前计数器计数不为 0，就一直阻塞等待下去，那么负责执行 sayHello 的 goroutine 就有足够的时间执行完成。

## 外部作用域的可见性

举个例子:
```go
var wg sync.WaitGroup

for _, salutation := range []string{"hello", "greetings", "good day"} {
    wg.Add(1)
    go func() {
        defer wg.Done()
        fmt.Println(salutation)
    }()
}

wg.Wait()
```

运行以上这段程序，输出的结果是:
> good day
good day
good day

由此看来，三个并发的 goroutine 和主 goroutine 是使用同一块内存，所以随着对该 slice 遍历下去，三个 goroutine 输出的 salutation 变量的值都是「good day」。

如果想要每个 goroutine 输出不同的元素，可以这样做：
```go
var wg sync.WaitGroup

for _, salutation := range []string{"hello", "greetings", "good day"} {
    wg.Add(1)
    go func(salutation string) {
        defer wg.Done()
        fmt.Println(salutation)
    }(salutation)
}

wg.Wait()
```

这段程序输出的结果就变成：
> good day
greetings
hello

由以上这两个示例来看，其实就跟 js 的闭包作用域差不多。

## Goroutine 的性能优势

首先从内存方面来考量，每个 goroutine 的内存开销是很小的，Go 官网的相关描述是这样的：
> To make the stacks small, Go's run-time uses resizable, bounded stacks. A newly minted goroutine is given a few kilobytes, which is almost always enough. When it isn't, the run-time grows (and shrinks) the memory for storing the stack automatically, allowing many goroutines to live in a modest amount of memory. The CPU overhead averages about three cheap instructions per function call. It is practical to create hundreds of thousands of goroutines in the same address space. If goroutines were just threads, system resources would run out at a much smaller number.

由此可见，完成相同的任务，使用 goroutine 所要花销的内存比使用单纯的多线程方式要少。

另外一点是，多线程运行时，线程的上下文切换也是有开销的。因为 Go 采取的方式是一个线程对应多个 goroutine，因而多个 goroutine 轮流执行的时候，在操作系统的角度来看并没有发生线程切换，因而大大减少线程上下文切换的开销。

---
参考资料：
- 《Goncurrency in Go》第三章，作者 Katherine Cox-Buday

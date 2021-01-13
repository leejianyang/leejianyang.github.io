---
title: Go语言并发编程笔记（二）：线程间数据交互
date: 2021-01-13 10:04:49
tags:
- Go语言
categories:
- coding
---

本文关于 Go 语言中线程间数据交互的核心理念。

<!--more-->

在 Go 语言中，goroutine 的本质是线程，因而在 Go 语言中使用 goroutine 编写并发程序，本质上就是多线程编程。接下来本文将谈及两种多线程的数据交互理念，以及使用它们的权衡取舍，在下文中将混着使用线程和 goroutine 这两个概念，暂不作严格区分（实质上还是有很大差别的)。

## Go 语言的并发哲学

Go 语言有一条格言：share memory by communicating，don't communicate by sharing memory。意思就是说，如果多个 goroutine 之间要进行数据交流，不要使用直接共享内存的方式，而是要引入一些数据结构让数据在多个 goroutine 之间流转起来。

基于以上这个编程理念，Go 设计了 channel 这个数据结构。顾名思义，channel 就是并发运行的 goroutine 进行数据交流的一个渠道。使用 channel 以及一些其余的相关配套技术，就可以让并发代码的耦合度降低。


## 传统的并发原语

话又说回来，别的编程语言使用共享内存和锁等等这些设计方式和工具，同样可以进行多线程并发编程。在 Go 语言中同样可以让开发者采用这些原始的基础并发原语。Go 语言在官方提供的 sync 包里面就提供了这些基础的并发原语，它的文档是这样说的：
> Package sync provides basic synchronization primitives such as mutual exclusion locks. Other than the Once and WaitGroup types, most are intended for use by low-level library routines. High-level synchronization is better done via channels and communication.

就是说这两大类工具 Go 语言都提供了，留给开发者自己去选择。

## 在 channel 和传统并发原语之间做选择

Go 语言在 sync 包中提供了传统的锁机制。大部分的并发场景既可以选择传统的共享内存加锁，也可以选择使用 channel 及其相关工具，那究竟如何选择呢？《Concurrency in Go》一书中提供了这样一个决策树：

![](/images/golang-csp/decision-tree.jpg)


接下来具体看看每个决策点。

### 是否涉及数据的所有权转移？

什么是数据的所有权转移呢？举个例子，一个 goroutine 产生的数据，想交给别的 goroutine 去使用，类似于一个生产——消费的模式，这就是一个所有权转移的过程。要达到这个目的，使用 channel 就最合适了。

使用 channel 的另一个好处是能创建一个进程内纯内存的通信 buffer，这样的话就很有利于不同的逻辑代码之间达到解耦的目的。这种操作可以类比成微服务架构中经常会引入消息队列。

### 是否打算保护一个结构体中的内部状态？

举一个例子:
```go
type Counter struct {
    mu sync.Mutex
    value int
}

func (c *Counter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value++
}
```
显而易见这是一个可以让多个线程并发使用的计数器。类似于这种多个 goroutine 同时并发使用计数器的场景，channel 就派不上用场了。于是以上代码就采用在结构体中封装排它锁的方式，既达到了线程安全的目的，又可以隐藏具体的实现细节——代码的使用者直接执行 increment 操作即可，不用关心加锁释放锁的细节问题。

还有一个需要留意的经验是，如果发现锁在代码的各处暴露着，就要注意了，最好把锁封装在一个小的作用域范围里面。

### 是否打算协调各个复杂的代码逻辑？

这个在上文中也有涉及到，使用 channel 以及 select 语句等概念，可以更容易地协调各部分的代码逻辑。

### 是否对性能有严苛要求？

如果程序对性能的要求很高，那就只好使用传统的并发原语了，因为直接对内存进行共享，性能肯定更好。但是，没必要过早优化，可以先用 channel 来写，等到确定程序的瓶颈真的出现在 channel 身上，再用传统的并发原语进行重构。

## 总结

以上梳理了在 Go 语言中线程间数据共享的两种理念，线程间数据交互是多线程编程中的核心概念之一。至于这些并发原语是如何使用的，将在往后的文章之中陆续展开。

----
参考资料：
- 《Goncurrency in Go》第二章，作者 Katherine Cox-Buday

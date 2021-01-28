---
title: Go语言并发编程笔记（十一）：避免 goroutine 内存泄漏
date: 2021-01-27 14:45:50
tags:
- Go语言
categories:
- coding
---

虽然每个 goroutine 占用的内存很少，但也要避免 goroutine 的内存泄漏。

<!-- more -->

goroutine 内存泄漏是指，goroutine 在进程运行的生命周期中没有终止，一直运行下去。
通常一个 goroutine 终止运行要么是自身的代码执行完毕，要么是根据别的 goroutine 的状态来判断何时应该结束。如果是后一种的情况的话，往往需要由别的 goroutine （通常是主 goroutine）通知它何时结束。

先来看看一个内存泄漏的例子：
```go
doWork := func(strings <-chan string) <-chan interface{} {
    completed := make(chan interface{})
    go func() {
        defer fmt.Println("doWork exited.")
        defer close(completed)
        for s := range strings {
            fmt.Println(s)
        }
    }()
    return completed
}

doWork(nil)

// main goroutine 继续做别的事情
```
以上代码的第 6 行，由于一直从 channel 中读不到信息，goroutine 会一直处于阻塞状态，那么这个 goroutine 的内存是一直得不到释放的，就会发生内存泄露问题。

改进方案是，goroutine 再接收多一个参数，可以是一个只读的 channel，如果从这个 channel 中接收到数据，就表明别的 goroutine 通知自己该收工了。
改进例子如下：
```go
doWork := func(done <-chan interface{}, strings <-chan string) <-chan interface{} {
    terminated := make(chan interface{})

    go func() {
        defer fmt.Println("doWork exited.")
        defer close(terminated)
        for {
            select {
            case s:= <-strings:
                fmt.Println(s)
            case <-done:
                return
            }
        }
    }()

    return terminated
}

done := make(chan interface{})
terminated := doWork(done, nil)

go func() {
    time.Sleep(1 * time.Second)
    fmt.Println("Canceling doWork goroutine...")
    close(done)
}()

<-terminated
fmt.Println("Done.")
```

虽然以上这段代码的例子看上去有点无厘头，但体现了一种模式：goroutine 通过接收一个只读的 channel 参数来获知外界通知自己声明周期结束的信息，并且配合 select 语句来使用。

要防止 goroutine 的内存泄漏问题，关键是要意识到一点：**谁创建了 goroutine，谁就要负责确保该 goroutine 能在某个时间点正常结束**。

----
参考资料：
- 《Goncurrency in Go》第四章，作者 Katherine Cox-Buday

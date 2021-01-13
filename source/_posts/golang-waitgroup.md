---
title: Go语言并发编程笔记（四）：WaitGroup 的使用
date: 2021-01-13 17:29:54
tags:
- Go语言
categories:
- coding
---

Go 语言的 sync 包中提供的 WaitGroup 并发原语，是编排 goroutine 的有用工具。

<!--more-->

## WaitGroup 的使用场景

WaitGroup 是 sync 包提供的并发原语。它的作用是等待一组 goroutine 执行完毕。如果不关心 goroutine 的执行结果，那么就可以考虑使用 WaitGroup。当然也可以结合使用 channel 和 select 达到类似的效果。

## WaitGroup 的用法

WaitGroup 提供了三个方法:
```go
func (wg *WaitGroup) Add(delta int)
func (wg *WaitGroup) Done()
func (wg *WaitGroup) Wait()
```

以下是一个基本用法示例：
```go
hello := func(wg *sync.WaitGroup, id int) {
    defer wg.Done()
    fmt.Printf("Hello from %v!\n", id)
}

const numGreeters = 5
var wg sync.WaitGroup
wg.Add(numGreeters)
for i := 0; i < numGreeters; i++ {
    go hello(&wg, i+1)
}
wg.Wait()    // 阻塞等待所有的 goroutine 执行完毕
```

## 使用 WaitGroup 时的注意事项

一个是 WaitGroup 的计数不能为负值，就是说 WaitGroup.Done() 调用的次数不能超过使用 WaitGroup.Add() 累加的总数。

另一个是要执行所有的 Add() 操作后再调用 Wait()，不然会导致运行效果不符合预期。

还有一个是，如果一个 goroutine 在 Wait() 等待期间，还有别的 goroutine 调用同一个变量的 Add() 操作会导致发生错误。所以说 WaitGroup 虽然可以重用，但在开始了 Wait() 之后，要等 Wait() 结束后再开始新一轮的使用。

----
参考资料：
- 《Goncurrency in Go》第三章，作者 Katherine Cox-Buday

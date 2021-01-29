---
title: Go语言并发编程笔记（十五）：几种 channel 的使用模式
date: 2021-01-29 14:35:12
tags:
- Go语言
categories:
- coding
---

本文介绍几种实用的 channel 使用模式。

<!--more-->

## 模式一：Or Channel

先直接上代码例子：
```go
package main

import (
    "time"
    "fmt"
)

func main() {
    var or func(channels ...<-chan interface{}) <-chan interface{}

    or = func(channels ...<-chan interface{}) <-chan interface{} {
        switch len(channels) {
        case 0:
            return nil
        case 1:
            return channels[0]
        }

        orDone := make(chan interface{})
        go func() {
            defer close(orDone)

            switch len(channels) {
            case 2:
                select {
                case <-channels[0]:
                case <-channels[1]:
                }
            default:
                select {
                case <-channels[0]:
                case <-channels[1]:
                case <-channels[2]:
                case <-or(append(channels[3:], orDone)...):
                }
            }
        }()
        return orDone
    }

    sig := func(after time.Duration) <-chan interface{}{
        c := make(chan interface{})
        go func() {
            defer close(c)
            time.Sleep(after)
        }()
        return c
    }

    start := time.Now()
    <-or(
        sig(2*time.Hour),
        sig(5*time.Minute),
        sig(3*time.Second),
        sig(1*time.Minute),
    )
    fmt.Printf("done after %v", time.Since(start))
}
```
运行以上代码，输出的是：
```
done after 3.002105925s
```

以上代码演示了一个名为 or 的函数，它可以接受一组数量不限的 channel，or 函数的作用是让主 goroutine 一直阻塞，直到从其中一个 channel 读取到数据为止。
这种模式适合的场景是：有一组的任务同时进行，只要其中一个任务运行完毕，那么这一组任务整组都视为运行结束。

## 模式二：Or-done Channel

还是先上一段代码
```go
orDone := func(done, c <-chan interface{}) <-chan interface{} {
    valStream := make(chan interface{})
    go func() {
        defer close(valStream)
        for {
            select {
            case <-done:
                return
            case v, ok := <-c:
                if ok == false {
                    return
                }
                select {
                case valStream <- v:
                case <-done:
                }
            }
        }
    }()
    return valStream
}
```
以上这段代码中的 orDone 函数，先通过 c channel 从别的地方接收数据，并通过 valStream channel 把数据传递给下游。在这个过程中，orDone 要关注两件事情：
一是上游有没有关闭 c channel，另一个是外部有没有通过 done channel 告知整个任务要完结，因而可以写成这样的模式。

对于 orDone 的使用者来说，只要这样简单地调用:
```go
for val := range orDone(done, myChan) {
    // 对 val 进行使用
}
```

## 模式三：Tee Channel

这种模式顾名思义，就是为了实现类型 Linux tee 命令的功能：将输入的数据复制多分，传给多个下游接收方。

不太复杂，直接上代码：
```go
package main

import (
    "fmt"
)

func main() {
    tee := func(
        done <-chan interface{},
        in <-chan interface{},
    ) (<-chan interface{}, <-chan interface{}) {
        out1 := make(chan interface{})
        out2 := make(chan interface{})
        go func() {
            defer close(out1)
            defer close(out2)

            for val := range in {
                var out1, out2 = out1, out2
                for i := 0; i < 2; i++ {
                    select {
                    case <-done:
                    case out1<-val:
                        out1 = nil
                    case out2<-val:
                        out2 = nil
                    }
                }
            }
        }()

        return out1, out2
    }

    productData := func() <-chan interface{} {
        dataStream := make(chan interface{})
        go func() {
            defer close(dataStream)
            for i := 0; i < 10; i++ {
                dataStream <- i
            }
        }()
        return dataStream
    }

    done := make(chan interface{})
    defer close(done)

    c1, c2 := tee(done, productData())
    for {
        select {
        case v1, ok := <-c1:
            if ok {
                fmt.Printf("get from c1: %v\n", v1)
            } else {
                c1 = nil
            }
        case v2, ok := <-c2:
            if ok {
                fmt.Printf("get from c2: %v\n", v2)
            } else {
                c2 = nil
            }
        }

        if c1 == nil && c2 == nil {
            fmt.Printf("done!\n")
            break
            }
    }
}
```
运行以上代码输出：
```
get from c2: 0
get from c1: 0
get from c2: 1
get from c1: 1
get from c2: 2
get from c1: 2
get from c1: 3
get from c2: 3
get from c2: 4
get from c1: 4
get from c1: 5
get from c2: 5
get from c1: 6
get from c2: 6
get from c1: 7
get from c2: 7
get from c1: 8
get from c2: 8
get from c1: 9
get from c2: 9
done!
```

核心的分发数据的代码是这部分：
```go
tee := func(
        done <-chan interface{},
        in <-chan interface{},
    ) (<-chan interface{}, <-chan interface{}) {
    out1 := make(chan interface{})
    out2 := make(chan interface{})
    go func() {
        defer close(out1)
        defer close(out2)

        for val := range in {
            var out1, out2 = out1, out2
            for i := 0; i < 2; i++ {
                select {
                case <-done:
                case out1<-val:
                    out1 = nil  // 暂时设为 nil，好让另外一个 out2 写入
                case out2<-val:
                    out2 = nil // 暂时设为 nil，好让另外一个 out1 写入
                }
            }
        }
    }()

    return out1, out2
}
```
有点美中不足的是，tee 函数分别通过 out1 和 out2 两个 channel 向下游分发数据，但是如果其中一个下游消费得较慢，会阻塞另一方的数据流。

## 总结

本文中三种 channel 的使用模式都比较实用，但应用这些模式的时候必须灵活，先了解这种模式背后的思路，再根据实际的应用场景来调整实用。

----
参考资料：
- 《Goncurrency in Go》第四章，作者 Katherine Cox-Buday

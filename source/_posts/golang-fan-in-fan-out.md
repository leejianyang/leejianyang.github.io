---
title: Go语言并发编程笔记（十四）：Fan-Out, Fan-In模式
date: 2021-01-28 17:20:03
tags:
- Go语言
categories:
- coding
---

在整个 pipeline 中，可能有某些阶段特别费时，这会造成整个 pipeline 的阻塞，此时需要一些方法提高整个 pipeline 的并发效率。

<!--more-->

先来看一个程序代码，这段代码做的事情是：随机生成 10 个 50,000,000 以内的质数。
```go
package main

import (
    "fmt"
    "math/rand"
    "time"
)

func main() {
    repeatFn := func(
        done <-chan interface{},
        fn func() interface{},
    ) <-chan interface{} {
        valueStream := make(chan interface{})
        go func() {
            defer close(valueStream)
            for {
                select {
                case <-done:
                    return
                case valueStream <- fn():
                }
            }
        }()

        return valueStream
    }

    take := func(
        done <-chan interface{},
        valueStream <-chan interface{},
        num int,
    ) <-chan interface{} {
        takeStream := make(chan interface{})
        go func() {
            defer close(takeStream)
            for i := 0; i < num; i++ {
                select {
                case <-done:
                    return
                case takeStream <- <-valueStream:
                }
            }
        }()

        return takeStream
    }

    toInt := func(
        done <-chan interface{},
        valueStream <-chan interface{},
    ) <-chan int {
        intStream := make(chan int)
        go func() {
            defer close(intStream)
            for v := range valueStream {
                select {
                case <-done:
                    return
                case intStream <- v.(int):
                }
            }
        }()
        return intStream
    }

    primeFinder := func(
        done <-chan interface{},
        intStream <-chan int,
    ) <-chan interface{} {
        primeStream := make(chan interface{})
        go func() {
            defer close(primeStream)
            for integer := range intStream {
                integer -= 1
                prime := true
                for divisor := integer - 1; divisor > 1; divisor-- {
                    if integer % divisor == 0 {
                        prime = false
                        break
                    }
                }

                if prime {
                    select {
                    case <- done:
                        return
                    case primeStream <- integer:
                    }
                }
            }
        }()

        return primeStream
    }

    rand := func() interface{} {
        // 这段代码故意没有在每次运行时重新生成随机种子，便于多次复现，比较不同实现的效率
        return rand.Intn(50000000)
    }

    done := make(chan interface{})
    defer close(done)

    start := time.Now()

    randIntStream := toInt(done, repeatFn(done, rand))
    fmt.Println("Primes: ")
    for prime := range take(done, primeFinder(done, randIntStream), 10) {
        fmt.Printf("\t%d\n", prime)
    }

    fmt.Printf("Search took: %v", time.Since(start))
}
```

以上这段代码应用上 pipeline 模式，但运行起来很费时。其中一组随机种子，要 9 秒多才能运行完毕。

如何改良呢？可以应用「fan-out, fan-in」模式。 Fan-out 是指开启多个 goroutine 来并发处理 pipeline 中上游传过来的输入数据，fan-in 是指将多个 goroutine 处理完成的数据汇入同一个 channel 传递给下游。

如果某个 pipeline stage 符合以下条件，就可以效率使用这种模式：
- 每次数据处理都是独立进行的，也就说不依赖于之前处理过的数据（order-independent）
- 该 stage 运行时间特别长

那显而易见，上文程序中的 primeFinder 函数特别适合改成 fan-out, fan-in 的模式。

那么将上文的程序修改如下：
```go
package main

import (
    "fmt"
    "math/rand"
    "time"
    "sync"
    "runtime"
)

func main() {
    repeatFn := func(done <-chan interface{}, fn func() interface{}) <-chan interface{} {
        valueStream := make(chan interface{})
        go func() {
            defer close(valueStream)
            for {
                select {
                case <-done:
                    return
                case valueStream <-fn():
                }
            }
        }()

        return valueStream
    }

    take := func(
        done <-chan interface{},
        valueStream <-chan interface{},
        num int,
    ) <-chan interface{} {
        takeStream := make(chan interface{})
        go func() {
            defer close(takeStream)
            for i := 0; i < num; i++ {
                select {
                case <-done:
                    return
                case takeStream <- <-valueStream:
                }
            }
        }()

        return takeStream
    }

    toInt := func(
        done <-chan interface{},
        valueStream <-chan interface{},
    ) <-chan int {
        intStream := make(chan int)
        go func() {
            defer close(intStream)
            for v := range valueStream {
                select {
                case <-done:
                    return
                case intStream <- v.(int):
                }
            }
        }()
        return intStream
    }

    primeFinder := func(
        done <-chan interface{},
        intStream <-chan int,
    ) <-chan interface{} {
        primeStream := make(chan interface{})
        go func() {
            defer close(primeStream)
            for integer := range intStream {
                integer -= 1
                prime := true
                for divisor := integer - 1; divisor > 1; divisor-- {
                    if integer % divisor == 0 {
                        prime = false
                        break
                    }
                }

                if prime {
                    select {
                    case <- done:
                        return
                    case primeStream <- integer:
                    }
                }
            }
        }()

        return primeStream
    }

    fanIn := func(
        done <-chan interface{},
        channels ...<-chan interface{},
    ) <-chan interface{} {
        var wg sync.WaitGroup
        multiplexedStream := make(chan interface{})

        multiplex := func(c <-chan interface{}) {
            defer wg.Done()
            for i := range c {
                select {
                case <-done:
                    return
                case multiplexedStream <- i:
                }
            }
        }

        wg.Add(len(channels))
        for _, c := range channels {
            go multiplex(c)
        }

        go func() {
            wg.Wait()
            close(multiplexedStream)
        }()

        return multiplexedStream
    }

    rand := func() interface{} {
        return rand.Intn(50000000)
    }

    done := make(chan interface{})
    defer close(done)

    randIntStream := toInt(done, repeatFn(done, rand))

    start := time.Now()

    numFinders := runtime.NumCPU()
    fmt.Printf("Spinning up %d prime finders.\n" ,numFinders)
    finders := make([]<-chan interface{}, numFinders)

    fmt.Println("Primes: ")
    for i := 0; i < numFinders; i++ {
        finders[i] = primeFinder(done, randIntStream)
    }

    for prime := range take(done, fanIn(done, finders...), 10) {
        fmt.Printf("\t%d\n", prime)
    }

    fmt.Printf("Search took: %v", time.Since(start))
}
```
经过这样的优化后，随机生成 10 个 50,000,000 以内的质数的效率提升了好几倍。

以上 Fan-out，fan-in 是如何实现的呢？首先，可以看到以上这段代码根据 CPU 的数量开启了多个 goroutine 来运行 primeFinder，这是一个 fan-out 的操作：
```go
numFinders := runtime.NumCPU()
finders := make([]<-chan interface{}, numFinders)
for i := 0; i < numFinders; i++ {
    finders[i] = primeFinder(done, randIntStream)
}
```
每一个独立的 finder 都会从上游 randIntStream 中获取数据。

那如何确保各个运行 primeFinder 的 goroutine 都能正常运行，并 fan-in 到下游呢？关键在新引入的 fan-in 函数：
```go
fanIn := func(
    done <-chan interface{},
    channels ...<-chan interface{},
) <-chan interface{} {
    var wg sync.WaitGroup
    multiplexedStream := make(chan interface{})

    multiplex := func(c <-chan interface{}) {
        defer wg.Done()
        for i := range c {
            select {
            case <-done:
                return
            case multiplexedStream <- i:
            }
        }
    }

    wg.Add(len(channels))
    for _, c := range channels {
        go multiplex(c)
    }

    go func() {
        wg.Wait()
        close(multiplexedStream)
    }()

    return multiplexedStream
}
```
可以说 primeFinder 是一个 stage，take 是一个 stage，在没引入「fan-out, fan-in」模式前，primeFinder 生成的数据直接流入 take；但引入了「fan-out, fan-in」模式后，primeFinder 生成的数据先流入 fan-in，由 fan-in 再统一流入 take。Fan-in 这个 stage 会确保各个并发运行的 primeFinder 产生的数据都会集中流入到下游。

重新完整捋一捋。首先源头的 repeatFn 生成随机数，然后经过 toInt 进行类型转换后，数据进入 randIntStream channel 等待下游接收。下游的 primeFinder 由多个（数量因当前机器的 CPU 而定）goroutine 并发运行，每个 goroutine 都会从 randIntStream 尝试提取数据，提取到数据后计算是否质数，若是质数，则把数据传入 primeStream channel 中。要注意的是，不同的 primeFinder 创建了不同的 primeStream channel。然后 fan-in 中也开启了与 primeFinder 并发数量相同的 goroutine，每个独立的 goroutine 从独立的 primeStream channel 中读取数据，把数据集中传入 multiplexedStream channel 中，等待下游的 take 阶段读取。

以上就是一个「fan-out, fan-in」模式的例子。

---

参考资料：
- 《Goncurrency in Go》第四章，作者 Katherine Cox-Buday

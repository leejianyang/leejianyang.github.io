---
title: Go语言并发编程笔记（十三）：Pipeline
date: 2021-01-27 16:37:14
tags:
- Go语言
categories:
- coding
---

Pipeline 是编程中一种有用的模式，利用 Pipeline 模式，可以使程序的结构更加清晰，降低代码间的耦合度，并提高程序的并发效率。

<!--more-->

## 什么是 Pipeline

Pipeline 可以理解成将整个程序分成多个组件，每个组件做三件事情：接收数据、处理数据、返回数据。每个组件可以称为一个「阶段」（stage）。

Pipeline Stage 有以下特性：
- 同一个 pipeline 中的 stage 接收的数据和返回的数据必须是同一种类型
- stage 可以在代码中任意传递

当 Stage 符合以上两点时，就可以将多个 stage 组合起来形成一个完整的 pipeline。而 Go 语言的函数就很适合作为 stage。

## Pipeline 的简单例子

有以下一个函数 multiply：第一个参数接收一个整形切片，另一个参数接收一个整数值 multiplier，该函数做的事情是将切片中的每一个数与第二个参数 multiplier 相乘:
```go
multiply := func(values []int, multiplier int) []int {
    multipliedValues := make([]int, len(values))
    for i, v := range values {
        multipliedValues[i] = v * multiplier
    }
    return multipliedValues
}
```

还有以下一个函数 add，接收的参数和 multipy 类型相同，做的事情不是相乘而是相加：
```go
add := func(values []int, additive int) []int {
    addedValues := make([]int, len(values))
    for i, v := range values {
        addedValues[i] = v + additive
    }
    return addedValues
}
```

将 multipy 和 add 结合起来，就可以形成一个 pipeline：
```go
ints := []int{1, 2, 3, 4}
for _, v := range multiply(add(multiply(ints, 2), 1), 2) {
    fmt.Println(v)
}
```
这段代码输出的结果是：
```
6
10
14
18
```

以上这种 pipeline 的形式称为「batch processing」，它的特点是每个 stage 一次性处理整个数据集。还有一种 pipeline 称为「stream processing」，这种 piepline 每个 stage 每次只处理整个数据集的一个子集。
将以上的 batch processing 改为 stream processing 的形式：
```go
multiply := func(value, multiplier int) int {
    return value * multiplier
}

add := func(value, additive int) int {
    return value + additive
}

ints := []int{1, 2, 3, 4}
for _, v := range ints {
    fmt.Println(multiply(add(multiply(v, 2), 1), 2))
}
```
运行以上这段代码输出的结果也是：
```
6
10
14
18
```

## 构造 Pipeline 的最佳实践

Go 语言很适合来构造 pipeline 模式的代码，使用 channel 来改造以上的代码：
```go
generator := func(done <-chan interface{}, integers ...int) <-chan int {
    intStream := make(chan int)
    go func() {
        defer close(intStream)
        // 将要处理的数据集写入一个 channel，形成一个数据输入流
        for _, i := range integers {
            select {
            case <-done:
                return
            case intStream <- i:
            }
        }
    }()
    return intStream
}

multiply := func(done <-chan interface{}, intStream <-chan int, multiplier int) <-chan int {
    multipliedStream := make(chan int)
    go func() {
        defer close(multipliedStream)
        for i := range intStream {
            select {
            case <-done:
                return
            case multipliedStream <- i*multiplier:
            }
        }
    }()
    return multipliedStream
}

add := func(done <-chan interface{}, intStream <-chan int, additive int) <-chan int {
    addedStream := make(chan int)
    go func() {
        defer close(addedStream)
        for i := range intStream {
            select {
            case <-done:
                return
            case addedStream <- i+additive:
            }
        }
    }()
    return addedStream
}

// 确保不会出现 goroutine 内存泄漏
done := make(chan interface{})
defer close(done)

// 将要处理的数据集生成数据流，供后续的 stage 处理
intStream := generator(done, 1, 2, 3, 4)
// 分解成多个 stage 处理，对数据流中的每个数据，依次进行乘以2，加1，和乘以 2 的操作
pipeline := multiply(done, add(done, multiply(done, intStream, 2), 1), 2)

// pipeline 变量也是一个 channel，从 channel 中逐个取出最后的结果
for v:= range pipeline {
    fmt.Println(v)
}
```
运行一下，结果依然是：
```
6
10
14
18
```

由以上代码可以看出，Go 语言的 channel 是很适合构造 pipeline 的，因为 channel 就是用于流转数据的，而 pipeline 讲究的就是数据在各个 stage 之间流转。再配合使用上 goroutine 技术，整个 pipeline 中的各个 stage 就可以并发地运行了。虽然代码是变复杂了，但运行的效率得到大大提升。

### Generator

再来加深一下对 generator 的认识。上文已经提到，generator 是将整个待处理的数据集转换成数据流的形式；也可以理解为，generator 是整个 pipeline 的数据的制造者，要在 pipeline 处理的数据由 generator 产生，源源不断地流入 pipeline 中直到数据生产完毕。

看下面一个例子：
```go
package main

import (
    "fmt"
    "math/rand"
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
                case valueStream <- fn():
                }
            }
        }()
        return valueStream
    }

    take := func(done <-chan interface{}, valueStream <-chan interface{}, num int) <-chan interface{} {
        takeStream := make(chan interface{})
        go func() {
            defer close(takeStream)
            for i := 0; i < num; i++ {
                select {
                case <-done:
                    return
                case takeStream <- <- valueStream:
                }
            }
        }()
        return takeStream
    }

    done := make(chan interface{})
    defer close(done)

    rand := func() interface{} {
        return rand.Int()
    }

    for num := range take(done, repeatFn(done, rand), 10) {
        fmt.Println(num)
    }
}
```
以上代码的 repeatFn 是一个比较通用的 Generator。数据实际由外部的函数 fn 生成，该 generator 只是把目标数据集以流的形式生成，下游的 stage 消费一个数据，就利用外部的 fn 生成一个新的数据并写入 piepline 的流中，下有的 stage 未处理完毕，就不会有新的数据生成。

还有一个值得留意的细节是，上面很多处类型声明都使用了`interface{}`的类型，这样写是为了通用性，必要时可以使用具体类型，性能会稍稍好一点。

---
参考资料：
- 《Goncurrency in Go》第四章，作者 Katherine Cox-Buday

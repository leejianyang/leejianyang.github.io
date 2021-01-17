---
title: Go语言并发编程笔记（七）：Once 的使用
date: 2021-01-17 15:11:14
tags:
- Go语言
categories:
- coding
---

本文介绍 Go 语言 sync 包中的 Once 的用法。

<!--more-->

## 适用场景

在并发编程中，要确保某些行为在整个进程的生命周期里面只发生一次，就可以使用 sync 包中提供的 Once 结构。

## Once 的用法

Once 只提供了一个接口：
```go
func (o *Once) Do(f func())
```

传递给 Do 的是一个无参数无返回值的函数。

举个使用的例子：
```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var once sync.Once

    for i := 0; i < 10; i++ {
        once.Do(func () {
            fmt.Printf("%d\n", i)
        })
    }
}
```

运行以上代码，只会打印出一行 0。

## 使用 Once 的注意事项

就算每次传给 Do 函数的参数不是同一个函数，也不会运行多次：
```go
var count int
increment := func() { count++ }
decrement := func() { count-- }

var once sync.Once
once.Do(increment)
once.Do(decrement)

fmt.Printf("Count: %d\n", count)
```
运行以上代码，输出 `Count: 1`

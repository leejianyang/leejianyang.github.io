---
title: Go语言并发编程笔记（十二）：错误处理
date: 2021-01-27 15:52:54
tags:
- Go语言
categories:
- coding
---

在并发编程中，进行恰当的错误处理变得更为复杂，错误处理的代码与正常运行流程的代码同样重要。

<!--more-->

在编写错误处理代码的时候，要考虑一个核心问题：谁来负责处理错误？

在并发编程中，通常可以考虑的做法是：**将当前运行部分的错误信息，传递给进程中其余掌握更多全局逻辑的代码来处理**。

举个例子：
```go
type Result struct {
    Error error
    Response *http.Response
}

checkStatus := func(done <-chan interface{}, urls ...string) <-chan Result {
    results := make(chan Result)

    go func() {
        defer close(results)

        for _, url := range urls {
            var result Result
            resp, err := http.Get(url)
            result = Result{Error: err, Response: resp}
            select {
            case <-done:
                return
            case results <- result:
            }
        }
    }()

    return results
}

done := make(chan interface{})
defer close(done)

urls := []string{"https://www.google.com", "https://bacdhost"}
for result := range checkStatus(done, urls...) {
    if result.Error != nil {
        fmt.Printf("error: %v", result.Error)
        continue
    }
    fmt.Printf("Response: %v\n", result.Response.Status)
}
```
以上代码就是一个例子，负责进行 checkStatus 的代码不知道当 http 请求发生异常时该如何处理，那么就把结果通过 channel 传递出去，交由有能力的部分来处理，达到很好的解耦目的。

----
参考资料：
- 《Goncurrency in Go》第四章，作者 Katherine Cox-Buday

---
title: Go语言并发编程笔记（十六）：Context 的使用
date: 2021-01-31 21:17:06
tags:
- Go语言
categories:
- coding
---

Context 是一个在调用栈上下文中传递数据时非常有用的包，它支持在不同调用层级的 goroutine 之间传递取消信号及其它数据。

<!--more-->

## Context 包的简介

Context 包稍微有一点儿复杂，先引用官方文档的介绍：
> Package context defines the Context type, which carries deadlines, cancellation signals, and other request-scoped values across API boundaries and between processes.
>
> Incoming requests to a server should create a Context, and outgoing calls to servers should accept a Context. The chain of function calls between them must propagate the Context, optionally replacing it with a derived Context created using WithCancel, WithDeadline, WithTimeout, or WithValue. When a Context is canceled, all Contexts derived from it are also canceled.
>
> The WithCancel, WithDeadline, and WithTimeout functions take a Context (the parent) and return a derived Context (the child) and a CancelFunc. Calling the CancelFunc cancels the child and its children, removes the parent's reference to the child, and stops any associated timers. Failing to call the CancelFunc leaks the child and its children until the parent is canceled or the timer fires. The go vet tool checks that CancelFuncs are used on all control-flow paths.
>
> Use context Values only for request-scoped data that transits processes and APIs, not for passing optional parameters to functions.
>
> The same Context may be passed to functions running in different goroutines; Contexts are safe for simultaneous use by multiple goroutines.

来看看 Context 类型的接口：
```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)

    Done() <-chan struct{}

    Err() error

    Value(key interface{}) interface{}
}
```

接下来看看 Context 包提供了几个主要的函数。

### WithCancel

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
```

调用 WithCancel 需要把父 goroutine 传过来的 context 作为参数，返回一个新的 context 和 cancel 函数。在两种情况下，这个新的 ctx 的 Done channel 会关闭：情况一是 cancel 方法被调用；情况二是 parent 的 Done channel 关闭了。

### WithDeadline
```go
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)
```

调用 WithDeadline 也与 WithCancel 类似，就是说根据父 context 创建新的 context，同时可以指定该 context 的过期时间。如果该过期时间比 parent 的过期时间迟，那么还是使用 parent 的过期时间作为新 context 的过期时间。新的 context 的 Done channel 在三种情况下关闭：（1）到达过期时间（2）CancelFunc 被调用（3）父 context 的 Done channel 关闭。

### WithTimeout
```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

WithTimeout 的使用几乎与 WithDeadline 一样，不同之处是 WithDeadline 指定的是在某个时刻超时，WithTimeout 指定的是在某个时长后超时。

### CancelFunc

上文多次出现 CancelFunc 类型的函数，官网文档说得很清楚了：
> A CancelFunc tells an operation to abandon its work. A CancelFunc does not wait for the work to stop. A CancelFunc may be called by multiple goroutines simultaneously. After the first call, subsequent calls to a CancelFunc do nothing.

### 产生空的 Context

有两个方法可以产生空的 Context
```go
func Background() Context

func TODO() Context
```

Background 函数返回的 Context 是一个空的 Context，它永远不会自动取消，不包含值，没有过期时间。通常是最顶层的 Context。

TODO 函数返回的 Context 跟 Background 返回的 Context 类似，不过语义上用于该 Context 用途不明的情况。

### 使用 Context 时的注意事项

- 不要将 Context 存储在一个结构体中
- 在函数中显式地传递 Context，通常是函数的第一个参数
- 不要传递 nil Context，如果你不知道使用哪个 Context，那么就使用 TODO 函数返回的 context

## Context 的应用场景

Context 的含义是「上下文」，顾名思义，Context 就是用于在多个 goroutine 调用栈之中传递数据的。例如在 main 函数中开启一个 goroutine 运行 A 函数，A 函数中又开启一个 goroutine 运行 B 函数，那么怎么将信息从 main 函数中传递到 B 函数中呢？main 函数如何控制 B 函数合适运行结束呢？这都可以通过 Context 来传递上下文信息。

概括起来，Context 包主要有两个方面的用途：
- 在并发调用栈中，取消一系列的调用操作
- 在调用栈中传递数据

先说「取消调用」的使用场景，通常有几种情况，一是当前的 goroutine 要通知子 goroutine 提前结束；二是当前的 goroutine 收到父 goroutine 的通知要提前结束；三是运行超时了或者到达一定时限了，goroutine 要结束。具体这几种情况如何应用 Context 包，下文再具体举例子。

还可以利用 Context 在 goroutine 调用栈中传递数据，官方文档强调，Context 传递的数据是「request-scoped data」，但官方文档中并没有说什么样的数据才算是 request-scoped data，《Concurrency in Go》的作者给出了他的理解：
- The data should transit process or API boundaries：就是说这个数据是打算跨进程传输的
- The data should be immutable：数据不可改变
- The data should trend toward simple types：如果数据是需要跨进程的话，那么基本数据类型方便接收方来使用
- The data should be data, not types with methods：只包含数据，不包含方法
- The data should help decorate operations, not drive them：如果接收方的算法行为是受 Context 包含的某些数据影响，那么这些数据不应该放在 Context 中，而是应该作为参数传递

这几条也不是什么金科玉律，仅供参考。

## Context 使用实例：终结运行中的 goroutine

先来看这一段代码：
```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func main() {
    var wg sync.WaitGroup
    done := make(chan interface{})
    defer close(done)

    wg.Add(1)
    go func() {
        defer wg.Done()
        if err := printGreeting(done); err != nil {
            fmt.Printf("%v", err)
            return
        }
    }()

    wg.Add(1)
    go func() {
        defer wg.Done()
        if err := printFarewell(done); err != nil {
            fmt.Printf("%v", err)
            return
        }
    }()

    wg.Wait()
}

func printGreeting(done <-chan interface{}) error {
    greeting, err := genGreeting(done)
    if err != nil {
        return err
    }
    fmt.Printf("%s world!\n", greeting)
    return nil
}

func printFarewell(done <-chan interface{}) error {
    farewell, err := genFarewell(done)
    if err != nil {
        return err
    }
    fmt.Printf("%s world!\n", farewell)
    return nil
}

func genGreeting(done <-chan interface{}) (string, error) {
    switch locale, err := locale(done); {
    case err != nil:
        return "", err
    case locale == "EN/US":
        return "hello", nil
    }
    return "", fmt.Errorf("unsupported locale")
}

func genFarewell(done <-chan interface{}) (string, error) {
    switch locale, err := locale(done); {
    case err != nil:
        return "", err
    case locale == "EN/US":
        return "goodbye", nil
    }
    return "", fmt.Errorf("unsupported locale")
}

func locale(done <-chan interface{}) (string, error) {
    select {
    case <-done:
        return "", fmt.Errorf("canceld")
    case <-time.After(1*time.Minute):
    }
    return "EN/US", nil
}
```

运行以上代码，过了大约一分钟后，会输出以下内容：
```
hello world!
goodbye world!
```

以上代码看起来比较长，但其实就是一个简单的调用逻辑，如下图：

![](/images/golang-context/01.jpg)

现在修改一下以上程序的逻辑：genGreeting 只愿意等待 1 秒钟，如果超过了 1 秒，则终止运行自身，当然也会终止运行 locale；这样一来，printGreeting 的正常逻辑就不会进行下去了，既然 greeting 都不进行了，那么 farewell 的行为也要取消。这一系列的变化，可以通过 Context 来实现。

先看代码：
```go
package main

import (
    "sync"
    "context"
    "time"
    "fmt"
)

func main() {
    var wg sync.WaitGroup
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    wg.Add(1)
    go func() {
        defer wg.Done()

        if err := printGreeting(ctx); err != nil {
            fmt.Printf("cannot print greeing: %v\n", err)
            cancel()
        }
    }()

    wg.Add(1)
    go func() {
        defer wg.Done()
        if err := printFarewell(ctx); err != nil {
            fmt.Printf("cannot print farewell: %v\n", err)
        }
    }()

    wg.Wait()
}

func printGreeting(ctx context.Context) error {
    greeting, err := genGreeting(ctx)
    if err != nil {
        return err
    }
    fmt.Printf("%s world!\n", greeting)
    return nil
}

func printFarewell(ctx context.Context) error {
    farewell, err := genFarewell(ctx)
    if err != nil {
        return err
    }
    fmt.Printf("%s world!\n", farewell)
    return nil
}

func genGreeting(ctx context.Context) (string, error) {
    ctx, cancel := context.WithTimeout(ctx, 1*time.Second)
    defer cancel()

    switch locale, err := locale(ctx); {
    case err != nil:
        return "", err
    case locale == "EN/US":
        return "hello", nil
    }
    return "", fmt.Errorf("unsupported locale")
}

func genFarewell(ctx context.Context) (string, error) {
    switch locale, err := locale(ctx); {
    case err != nil:
        return "", err
    case locale == "EN/US":
        return "goodbye", nil
    }
    return "", fmt.Errorf("unsupported locale")
}

func locale(ctx context.Context) (string, error) {
    select {
    case <-ctx.Done():
        return "", ctx.Err()
    case <-time.After(1 * time.Minute):
    }
    return "EN/US", nil
}
```

以上这段代码引入了 Context。Context 在各个调用层次中又生成了新的 context 往下传递。在 genGreeting 函数中生成了一个 1 秒超时的 context，当这个 context 超时后，locale 函数就会发现 ctx 的 channel 已关闭，则会返回 `ctx.Err()` 的内容，这个内容会逐层向上传递，一直传递到 main 函数。代码的第 19-22 行中，main 函数收到 printGreeting 函数返回的错误信息，就会把 context cancel 掉，这个 cancel 的操作是会一直沿着 printFarewell 那个分支一直传递下去的。Context 的应用大致就是这个样子。

还有一个小优化的地方。locale 函数要运行 1 分钟才能返回，但是上层的 genGreeting 是不可以接受那么久的。上层的 genGreeting 函数的 timeout 设置是通过 context 传递进来的，所以 locale 是知道这个情况的，因而可以进一步修改一下 locale 函数：
```go
func locale(ctx context.Context) (string, error) {
    if deadline, ok := ctx.Deadline(); ok {
        if deadline.Sub(time.Now().Add(1*time.Minute)) <= 0 {
            return "", context.DeadlineExceeded
        }
    }

    select {
    case <-ctx.Done():
        return "", ctx.Err()
    case <-time.After(1 * time.Minute):
    }
    return "EN/US", nil
}
```
locale 函数可以先评估一下当前的运行时间是否满足 context 的 timeout 设置，如果发现满足不了，就不浪费时间执行了，直接返回错误。当然这个做法不是万能的，只能按需使用。

## Context 使用实例：传递 request-scoped 数据

上文已经提到 Context 的另一个用途是作为调用栈中的数据传递。

使用示例如下：
```go
package main

import (
    "context"
    "fmt"
)

func main() {
    ProcessRequest("jane", "abc123")
}

func ProcessRequest(userID, authToken string) {
    ctx := context.WithValue(context.Background(), "userID", userID)
    ctx = context.WithValue(ctx, "authToken", authToken)
    HandleResponse(ctx)
}

func HandleResponse(ctx context.Context) {
    fmt.Printf(
        "handling response for %v(%v)",
        ctx.Value("userID"),
        ctx.Value("authToken"),
    )
}
```

使用起来非常简单，只需注意以下条件：
- 传递给 WithValue 函数的 key 值必须是可比较的，就是说对这个 key 值执行 == 和 != 操作是可以返回正确的结果的
- value 值必须是线程安全的，也就是说支持多个 goroutine 同时取值

---
参考资料：
- 《Goncurrency in Go》第四章，作者 Katherine Cox-Buday

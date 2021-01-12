---
title: Go语言并发编程笔记（一）：并发编程的难点
date: 2021-01-12 10:23:00
tags:
- Go语言
categories:
- coding
---

本文主要关于编写并发代码时会遇到的难点。

<!--more-->

## 并发编程的难点

### Race Conditions

Race conditions 是指多个操作并行地发生，但是它们的先后顺序是不确定的。

举个例子：
```go
var data int

go func() {
    data++    // 代码 a
}()

if data == 0 {    // 代码 b
    fmt.Printf("the value is %v.\n", data)
}
```

以上代码执行的顺序是不确定的。把这段代码运行多次，有时候代码 a 先于代码 b 执行，有时候代码 b 先于代码 a 执行，这意味着每次执行该程序的结果是不确定的。

在编写并发代码时，必须要意识到 race conditions 的情况，不能预先假定代码一定会按某个顺序来执行。

### 原子性

在并发编程中，原子操作很有用，因为一个操作如果符合原子性，那就意味着这个操作在并发场景下是安全的。但很多的操作都不符合原子性，例如一个很常见的自增语句：
```go
i++
```
骤眼看来，这个操作是原子的，但仔细分析，它包含三个步骤：
- 获取 i 的值
- 计算 i+1 的值
- 更新 i 的值

这三个步骤单独执行是原子操作，但三个步骤结合起来并不是原子的。

所以在编写代码时，如果有一系列的操作需要符合原子性，就要使用合适的技术来保证相关操作的原子性。

### 同步访问内存

如果有多个并发的操作要访问同一块内存区域，而且访问操作不符合原子性，那么就可能会造成并发问题。

例如以下这段代码：
```go
var data int

go func() {
    data++
}()

if data == 0 {
    fmt.Println("the value is 0.")
} else {
    fmt.Println("the value is %v.\n", data)
}
```
有两个并行的代码都要对 data 变量进行访问，这就发生了 data race。那就需要一些技术来解决内存的并发访问问题，来个粗暴的解决方案：
```go
var memoryAccess sync.Mutex
var value int

go func() {
    memoryAccess.Lock()
    value++
    memoryAccess.Unlock()
}()

memoryAccess.Lock()
if value == 0 {
    fmt.Printf("the value is %v.\n", value)
} else {
    fmt.Printf("the value is %v.\n", value)
}
memoryAccess.Unlock()
```
以上的代码只是举一个粗糙的例子，来说明解决 data race 的思路之一就是加锁。

但是值得注意的是，以上的方案解决了 data race，但依然没有解决 race condition，因为如上文所说的那样，究竟哪部分代码先被执行是不确定的。

### 死锁，活锁及饥饿问题

加锁可以在某些场景下解决以上的并发编程问题，但又可能带来一系列新的问题。

#### 死锁

举一个死锁的例子:
```go
type value struct {
    mu sync.Mutex
    value int
}

var wg sync.WaitGroup
printSum := func(v1, v2 *value) {
    defer wg.Done()
    v1.mu.Lock()
    defer v1.mu.Unlock()

    time.Sleep(2 * time.Second)
    v2.mu.Lock()
    defer v2.mu.Unlock()

    fmt.Printf("sum=%v\n", v1.value + v2.value)
}

var a, b value
wg.Add(2)
go printSum(&a, &b)
go printSum(&b, &a)
wg.Wait()
```

以上这段代码还是能比较简单地看出死锁问题，但在复杂的项目中，死锁问题不一定能用肉眼就能发现。

#### 活锁

活锁（Livelock）的概念远不如死锁常见，举个活生生的例子：两个人迎面走在一条很窄的路上，两个人都想避让对方，但如果两人都不断地朝着同一侧变换方向避让对方，那么两个人都通不过去。

举一个活锁的代码例子：
```go
package main

import (
    "bytes"
    "fmt"
    "sync"
    "sync/atomic"
    "time"
)

func main() {
    cadence := sync.NewCond(&sync.Mutex{})
    go func() {
    for range time.Tick(1 * time.Millisecond) {
    	cadence.Broadcast()
    	}
    }()

    takeStep := func() {
        cadence.L.Lock()
        cadence.Wait()
        cadence.L.Unlock()
    }

    tryDir := func(dirName string, dir *int32, out *bytes.Buffer) bool {
        fmt.Fprintf(out, " %v", dirName)
        atomic.AddInt32(dir, 1)
        takeStep()              
        if atomic.LoadInt32(dir) == 1 {
        	fmt.Fprint(out, ". Success!")
        	return true
        }
        takeStep()
        atomic.AddInt32(dir, -1)
        return false
    }

    var left, right int32
    tryLeft := func(out *bytes.Buffer) bool { return tryDir("left", &left, out) }
    tryRight := func(out *bytes.Buffer) bool { return tryDir("right", &right, out) }
    walk := func(walking *sync.WaitGroup, name string) {
        var out bytes.Buffer
        defer func() { fmt.Println(out.String()) }()
        defer walking.Done()
        fmt.Fprintf(&out, "%v is trying to scoot:", name)
        for i := 0; i < 5; i++ {
        	if tryLeft(&out) || tryRight(&out) {
        		return
        	}
        }
    	fmt.Fprintf(&out, "\n%v tosses her hands up in exasperation!", name)
    }

    var peopleInHallway sync.WaitGroup
    peopleInHallway.Add(2)
    go walk(&peopleInHallway, "Alice")
    go walk(&peopleInHallway, "Barbara")
    peopleInHallway.Wait()
}
```

运行这段代码，得到的结果是：
>Alice is trying to scoot: left right left right left right left right left right
Alice tosses her hands up in exasperation!
Barbara is trying to scoot: left right left right left right left right left right
Barbara tosses her hands up in exasperation!

在以上这段代码中，双方都尝试主动地避免死锁，但因为双方并没有协调好，导致了双方都没有达成想要的目的。活锁比死锁更难发现。

活锁是饥饿问题（staravtion）的子集，下面再来看看饥饿问题。

#### 饥饿问题

活锁是并发编程中，所有并发的 goroutine 都获取不全自己想要的资源，而饥饿问题则是一部分的 goroutine 受到「不公平对待」。

```go
package main

import (
    "fmt"
    "sync"
    "time"
    )

func main() {
    var wg sync.WaitGroup
    var sharedLock sync.Mutex
    const runtime = 1 * time.Second

    greedyWorker := func() {
        defer wg.Done()

        var count int
        for begin := time.Now(); time.Since(begin) <= runtime; {
            sharedLock.Lock()
            time.Sleep(3 * time.Nanosecond)
            sharedLock.Unlock()
            count++
        }

        fmt.Printf("Greedy worker was able to execute %v work loops\n", count)
    }

    politeWorker := func() {
    	defer wg.Done()

    	var count int
    	for begin := time.Now(); time.Since(begin) <= runtime; {
            sharedLock.Lock()
            time.Sleep(1 * time.Nanosecond)
            sharedLock.Unlock()

            sharedLock.Lock()
            time.Sleep(1 * time.Nanosecond)
            sharedLock.Unlock()

            sharedLock.Lock()
            time.Sleep(1 * time.Nanosecond)
            sharedLock.Unlock()

            count++
    	}

    	fmt.Printf("Polite worker was able to execute %v work loops.\n", count)
    }

    wg.Add(2)
    go greedyWorker()
    go politeWorker()

    wg.Wait()
}
```

greedyWorker 由于更「野蛮」地占据着锁，所以它执行循环的次数会比 politeWorker 更多。这是由于程序自身的写法问题而导致不同代码段抢占到的资源不均衡。

还需要注意的是，除了 Go 代码以外，别的对资源的使用场景：例如 CPU、内存、文件句柄、数据库连接等等，一切可共享的资源都可能发生饥饿问题。

### 实现并发安全的人为因素

人的因素也不能忽视，每一行代码都是由人写出来的。程序员在写并发代码时，有很多需要考虑的地方。例如当你编写了一个库函数之后，如何让调用方意识到要注意并发问题？如果让自己写的并发代码更易用更易修改？到底要在代码结构的哪个层级中处理并发问题？这些都是权衡取舍的艺术。

再例如，要调用一段别人写的代码，怎么处理并发编程问题？例如使用别人提供的函数，其签名如下：
```go
// CalculatePi calculates digits of Pi between the begin and end place.
func CalculatePi(begin, end int64, pi *Pi)
```
那不看函数体的具体实现，可能就会疑惑，究竟这个函数实现时是不是已经利用了并发技术，一大批量的计算需要调用方一次性调用 CalculatePi 函数还是并性地小批次地调用该函数？传了 Pi 指针进去，意味着可能并发地访问该内存空间，谁来对它进行保护？

在编写并发代码时，对程序员的代码编写能力有更高的要求，既需要设计时的权衡取舍，选择合适的数据结构，还需要定义优雅的接口。

## 总结

在编写并发代码的时候，首先要意识到的第一个问题是，并发运行的多个代码片段的执行顺序是不确定的；正是由于执行顺序的不确定性，才需要确保一部分操作的原子性；特别是操作一些临界资源时，需要应用一定的技术来实现并发安全，例如是加锁；但加锁的时候要注意死锁以及活锁问题。另外，由于并发的程序往往是抢占式地执行的，要注意资源分配的均衡问题。

本文只涉及到并发难题的介绍，没有深入展开如何应对这些难题，因为在并发编程中，注意到这些可能出现的问题是最基础的，只有在有情景意识的前提下才能灵活应用具体的解决方案。在后面的文章里面再深入展开在 Go 语言中，实现并发安全的技术和套路。

-----
参考资料：
- 《Goncurrency in Go》第一章，作者 Katherine Cox-Buday

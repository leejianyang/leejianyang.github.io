---
title: Go语言并发编程笔记（八）：Pool 的使用
date: 2021-01-18 17:24:15
tags:
- Go语言
categories:
- coding
---

本文关于 Go 语言标准库 sync 中的 Pool 的使用和注意事项。

<!--more-->

## Pool 的适用场景

笼统地说，Pool 把已初始化的对象存储起来，以供后续复用。Pool 支持多个 goroutine 同时并发访问。

Go 官方的文档是这样说的：
> A Pool is a set of temporary objects that may be individually saved and retrieved.
>
> A Pool is safe for use by multiple goroutines simultaneously.
>
> Pool's purpose is to cache allocated but unused items for later reuse, relieving pressure on the garbage collector. That is, it makes it easy to build efficient, thread-safe free lists. However, it is not suitable for all free lists.
>
> An appropriate use of a Pool is to manage a group of temporary items silently shared among and potentially reused by concurrent independent clients of a package. Pool provides a way to amortize allocation overhead across many clients.

## Pool 的简单用法

```go
myPool := &sync.Pool{
    New: func() interface{} {
        fmt.Println("Creating new instance.")
        return struct{}{}
    },
}

myPool.Get()
instance := myPool.Get()
myPool.Put(instance)
myPool.Get()
```

运行以上的代码，「Creating new instance.」会输出2次。

Pool 的用法很简单，给 Pool 结构提供一个 New 方法用于创建对象，Get 到的对象在使用完毕后使用 Put 方法放回到 Pool 中，后续就可以复用了。

## 使用 Pool 时的注意事项

### Pool 在初始使用后不能进行复制

这一点跟 sync 包中的其余结构是类似的，要注意避免对 Pool 进行复制。

### Pool 中的缓存的对象会被 GC 回收

放在 Pool 里面未被其余变量引用的对象是会被 GC 回收的，所以并不完全适合用来做连接池；而且因为 Pool 里面的对象可能会被回收，也意味着不应该对该对象的内部状态进行任何假设。

### Pool 中最好存储同一种数据类型

放在同一个 Pool 里面的对象最好是同构的，也就是说同一种数据类型，以避免从 Pool 取出来之后还要进行类型转换。

### 注意内存泄漏问题

如果使用 Pool 来进行 buffer 的缓存，可能会有内存泄漏的问题。例如以下例子：
```go
pool := &sync.Pool {
    New: func() interface{} {
        return new(bytes.Buffer)
    }
}

buf := pool.Get().(*bytes.Buffer)
// 这里利用 buf 来做一些事情，buf 可能会不断增大
pool.Put(buf)
```
像这段代码，buf 占用的内存可能变得很大，如果把它放回 Pool，GC 又没有把它回收的话，就会造成内存的泄漏，就算放回 Pool 前调用了 Reset 方法也是无济于事的。

要避免内存泄露的方法之一，就是在放回 Pool 之前检查它的大小，如果过大了就直接丢弃：
```go
if buf.Cap() <= maxSize {
    pool.Put(buf)
}
```

为了规避内存问题，可以使用实现良好的第三方库：
- [valyala/bytebufferpool](https://github.com/valyala/bytebufferpool)
- [oxtoacart/bpool](https://github.com/oxtoacart/bpool)

## 连接池的使用场景

因为 Pool 中的对象会被 GC 回收，因而不完全使用作为连接池来使用。要使用连接池的话，可以考虑使用别的库：
- HTTP 客户端的话可以使用官方 http 包中的 [Transport](https://golang.org/pkg/net/http/#Transport)
- TCP 客户端的连接池可以使用 [faith/pool](https://github.com/fatih/pool)
- 数据库连接池在标准库 [db](https://golang.org/pkg/database/sql/#DB) 中已经有实现了

---
参考资料：
- 极客时间课程：Go并发编程实战课

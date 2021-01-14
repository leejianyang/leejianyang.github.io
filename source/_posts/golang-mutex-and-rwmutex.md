---
title: Go语言并发编程笔记（五）：Mutex 和 RWMutex 的使用
date: 2021-01-14 10:04:26
tags:
- Go语言
categories:
- coding
---

加锁是并发编程中的常规操作，Go 语言提供了互斥锁和读写锁这两种锁。

<!--more-->

## Mutex

sync 包中提供的 Mutex 是一个互斥锁，限制一个临界区域只能允许一个 goroutine 进入。

### Mutex 的用法

Mutex 的接口很简单，就上锁操作和释放锁操作：
```go
// 获取锁
func (m *Mutex) Lock()
 // 释放锁
func (m *Mutex) Unlock()
```

接下来看一段 demo：
```go
var count int
var lock sync.Mutex

increment := func() {
    lock.Lock()
    defer lock.Unlock()
    count++
    fmt.Printf("Incrementing: %d\n", count)
}

decrement := func() {
    lock.Lock()
    defer lock.Unlock()
    count--
    fmt.Printf("Decrementing: %d\n", count)
}

var wg sync.WaitGroup
for i := 0; i <= 5; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        increment()
    }()
}

for i := 0; i <= 5; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        decrement()
    }()
}

wg.Wait()
```

### 使用 Mutex 时的注意事项

首先一个是加锁操作 Lock 和释放锁操作要成双成对地出现，例如 Unlock 配合 defer 使用。

另一个要注意的是不能复制已使用后的同步原语，例如已经对 Mutex 进行加锁操作，就不能将它复制使用了。

再一个是，Mutex 是不可重入的，意思就是说一个 goroutine 已经成功获得该锁，就不能再进行一次 Lock 操作。

最后还需要注意，Unlock 方法可以被任意的 goroutine 调用释放锁，即使是没持有这个互斥锁的 goroutine，也可以进行这个操作。这是因为**Mutex 本身并没有包含持有这把锁的 goroutine 的信息**，所以，Unlock 也不会对此进行检查。所以，在使用 Mutex 的时候，必须要保证 goroutine 尽可能不去释放自己未持有的锁，一定要遵循「谁加锁，谁释放」的原则。

## RWMutex

Mutex 的加锁是比较单一的，一个 goroutine 持有了，这个临界区域就锁死了，别的 goroutine 都进不来。因而 Go 语言的 sync 包还提供了读写锁（RWMutex），读写锁的作用是：支持临界资源的并发读，但如果写锁定，别的线程就无法进行读写。

### RWMutex 的用法
RWMutex 提供了五个接口:
```go
// 获取写锁
func (rw *RWMutex) Lock()
// 获取读锁
func (rw *RWMutex) RLock()
// 释放读锁
func (rw *RWMutex) RUnlock()
// 释放写锁
func (rw *RWMutex) Unlock()
```

接下来看一段使用读写锁的 demo：
```go
package main

import (
    "sync"
    "fmt"
    "time"
)

func Read(idx int, l *sync.RWMutex, wg *sync.WaitGroup) {
    defer wg.Done()
    for i := 0; i < 5; i++ {
        fmt.Printf("只读 goroutine %d 尝试获取读锁...\n", idx)
        l.RLock()
        fmt.Printf("只读 goroutine %d 获取读锁成功\n", idx)
        time.Sleep(3 * time.Second)
        fmt.Printf("只读 goroutine %d 释放读锁\n", idx)
        l.RUnlock()
        time.Sleep(1 * time.Second)
    }
}

func Write(idx int, l *sync.RWMutex, wg *sync.WaitGroup) {
    defer wg.Done()
    for i := 0; i < 5; i++ {
        fmt.Printf("[非只读 goroutine %d 尝试获取写锁...]\n", idx)
        l.Lock()
        fmt.Printf("[非只读 goroutine %d 获取写锁成功]\n", idx)
        time.Sleep(1 * time.Second)
        fmt.Printf("[非只读 goroutine %d 释放写锁]\n", idx)
        l.Unlock()
        time.Sleep(3 * time.Second)
    }
}

func main() {
    var lock sync.RWMutex
    var wg sync.WaitGroup

    const READ_ONLY_NUMS int = 3
    const WRITE_NUMS int = 2

    wg.Add(READ_ONLY_NUMS + WRITE_NUMS)
    for i := 0; i < READ_ONLY_NUMS; i++ {
        go Read(i, &lock, &wg)
    }
    for i := 0; i < WRITE_NUMS; i++ {
        go Write(i, &lock, &wg)
    }
    wg.Wait()
}

```

### 使用 RWMutex 时的注意事项

使用 RWMutex 时的注意事项跟使用 Mutex 差不多，不能复制；注意不要重入；加锁和释放锁的操作要一一对应，既不能多也不能少；最后还要注意避免死锁。

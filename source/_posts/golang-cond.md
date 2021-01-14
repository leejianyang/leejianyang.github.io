---
title: Go语言并发编程笔记（六）：Cond 的使用
date: 2021-01-14 17:23:24
tags:
- Go语言
categories:
- coding
---

本文介绍 sync 包中的一个并发原语：Cond。

<!--more-->

## Cond 的使用场景

在并发编程过程中，有时候 goroutine 要等待某些条件符合时才继续执行下去。那这时候有两种选择，要么它进行轮询，每隔一段时间自行判断一下等待的条件是否符合；要么就把自己挂起来，等待别人给自己发送信号，收到信号后再次判断等待条件是否满足。显而易见，第二种方法更优。后者可以利用 Cond 来完成。

用一个伪代码来看就是这样：
```go
阻塞获取锁

if 某些条件不符合 {
    将锁释放，并阻塞等待信号到来
}

条件符合了，执行后续操作

通知别的等待者，条件可能发生了变化

释放锁
```

Cond 的作用就是配合 Mutex 或者 RWMutex 一起，完成对条件的等待和通知工作。 Cond 维护的等待队列是先进先出的。

## Cond 使用示例

Cond 类型主要有以下几个接口:
```go
// 创建一个 Cond 类型变量，且与某个锁关联起来
func NewCond(l Locker) *Cond
// 向所有等待者发送信号
func (c *Cond) Broadcast()
// 向单个等待者发送信号
func (c *Cond) Signal()
// 释放锁，并等待信号到来
func (c *Cond) Wait()
```

以下的 demo 代码模拟一个例子，顾客在服装店购物，一共只有两间更衣室，如果两间更衣室都被占用了，只能在更衣室外等候，只要有一间空出来了，就马上进去。
```go
package main

import (
    "fmt"
    "sync"
    "time"
    "math/rand"
)

func useRoom(personNum int, inRoomCount *int, cond *sync.Cond, wg *sync.WaitGroup) {
    // 模拟用户在 10 秒内到达
    time.Sleep(time.Duration(rand.Intn(10000)) * time.Millisecond)

    cond.L.Lock()
    if *inRoomCount >= 2 {
        fmt.Printf("客户 %d 在更衣室外排队\n", personNum)
        cond.Wait()
    }
    *inRoomCount++
    cond.L.Unlock()

    fmt.Printf("客户 %d 进入更衣室\n", personNum)
    // 模拟用户使用更衣室 1-5 秒
    time.Sleep(time.Duration(1000+rand.Intn(4000)) * time.Millisecond)

    fmt.Printf("客户 %d 离开更衣室\n", personNum)
    *inRoomCount--
    cond.Signal()

    wg.Done()
}

func main() {
    c := sync.NewCond(&sync.Mutex{})
    inRoomCount := 0

    var wg sync.WaitGroup
    wg.Add(10)
    rand.Seed(time.Now().Unix())
    for i := 0; i < 10; i++ {
        go useRoom(i, &inRoomCount, c, &wg)
    }

    wg.Wait()
}
```

## 使用 Cond 时的注意事项

使用 Cond 时，要注意在 Wait() 之前一定要先获取到锁，在获取到锁的情况下再等待唤醒信号；另一个是，虽然收到了唤醒信号，并不代表条件之前的等待条件就一定符合了，要重新判断一次条件。

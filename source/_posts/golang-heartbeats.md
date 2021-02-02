---
title: Go语言并发编程笔记（十七）：心跳机制
date: 2021-02-02 16:33:50
tags:
- Go语言
categories:
- coding
---

本文关于在 Go 代码中如何实现心跳机制。

<!--more-->

在 Go 语言中，开启了 goroutine 后有时可能需要定期向外界发送心跳信息，告知自身依然在正常工作。可以借助 time 包的 Tick 函数作为一个周期性触发器，定期向外部发送心跳。

示例代码如下：
```go
package main

import (
    "fmt"
    "time"
)

func main() {
    doWork := func(
        done <-chan interface{},
        pulseInterval time.Duration) (<-chan interface{}, <-chan time.Time) {
        heartbeat := make(chan interface{})
        results := make(chan time.Time)

        go func() {
            defer close(heartbeat)
            defer close(results)

            // 周期性地触发心跳发送
            pulse := time.Tick(pulseInterval)
            // 模拟周期性地生成结果
            workGen := time.Tick(2*pulseInterval)

            // 通过 heartbeat channel 发送心跳数据
            sendPulse := func() {
                select {
                case heartbeat <- struct{}{}:
                default:
                }
            }

            // 向 result channel 输送数据
            sendResult := func(r time.Time) {
                for {
                    select {
                    case <-done:
                        return
                    case <-pulse:
                        // 因为 results 可能被阻塞，所以在这里也要发送心跳
                        sendPulse()
                    case results <- r:
                        return
                    }
                }
            }

            for {
                select {
                case <-done:
                    return
                case <-pulse:
                    // 新的心跳周期发送心跳
                    sendPulse()
                case r := <-workGen:
                    // 数据就绪时发送数据
                    sendResult(r)
                }
            }


        }()
        return heartbeat, results
    }

    done := make(chan interface{})
    time.AfterFunc(11*time.Second, func() { close(done) })

    const timeout = 2 * time.Second
    heartbeat, results := doWork(done, timeout/2)
    for {
        select {
        case _, ok := <-heartbeat:
            // 收到心跳
            if ok == false {
                return
            }
            fmt.Println("pulse")
        case r, ok := <-results:
            // 收到数据
            if ok == false {
                return
            }
            fmt.Printf("results %v\n", r.Second())
        case <-time.After(timeout):
            // 如果超过 timeout 时间既收不到心跳，也收不到数据，判定为异常超时，进程结束
            return
        }
    }
}
```

以上代码演示了 goroutine 间如何传递心跳信息。

---
参考资料：
- 《Goncurrency in Go》第五章，作者 Katherine Cox-Buday

---
title:  关于 Linux 的平均负载和进程的不可中断状态
date: 2019-12-19
tags:
- Linux
- 操作系统
- 性能分析
categories:
- coding
---

在 uptime 命令的 man 信息中，对系统平均负载给出的定义是这样的：「System load averages is the average number of processes that are either in a runnable or uninterruptable state」。平均负载当然是一个直观反映系统当前负载程度的指标，有些人会误以为这个数值越高，就表明 CPU 的使用率越高。那究竟什么是进程的 uninterruptable 状态？为什么 load averages 既包括runnable 状态的进程，又要包括 uninterruptable 状态的进程？
<!--more-->

## 进程的不可中断状态
在操作系统教科书上面，通常一个进程的状态分为「就绪」、「运行」和「阻塞」，其中「阻塞」表示进程正在等待某些事件（例如 I/O）的发生，而「就绪」和「运行」状态的进程都可以被 CPU 继续执行。

再顺便看看 Linux 中对进程状态的分类：
- D: uninterruptible sleep (usually IO)
- I: Idle kernel thread
- R: running or runnable (on run queue)
- S: interruptible sleep (waiting for an event to complete)
- T: stopped by job control signal
- t: stopped by debugger during the tracing
- W: paging (not valid since the 2.6.xx kernel)
- X: dead (should never be seen)
- Z: defunct (“zombie”) process, terminated but not reaped by its parent

其中 interruptible sleep 和 uninterruptible sleep 是可以摆在一起对比的；处于「可中断休眠」状态的进程可能在等待终端输入，可能在等待数据写入当前的空管道；而处于「不可中断休眠」状态的进程可能是在等待 I/O 的完成。重要的区别是前者可以立刻对信号做出响应，但**处于不可中断状态的进程不能响应信号**。通常「不可中断休眠」状态不会持续很久，如果进程长时间一直维持在这个状态，就需要排查问题了。

## 为什么系统平均负载要算上不可中断状态的进程
看完 uptime 的 man 手册中关于系统平均负载的定义之后，我最大的疑问是：为什么要算上不可中断状态的进程？难道不是处于「就绪」和「运行」状态的进程越多，系统就越「卡」平均负载越「高」吗？我找到一篇非常好的文章：[Brendan Gregg 写的《Linux Load Averages: Solving the Mystery》](http://www.brendangregg.com/blog/2017-08-08/linux-load-averages.html) 解释这个问题。

在这篇文章中，Brendan Gregg 指出一些很古典的计算机系统，平均负载的定义只考虑单位时间内可运行的进程数，他也很疑惑为什么 Linux 系统的平均负载要扯上不可中断的进程数。经过一番努力，他在一封写于 1993 年的邮件中找到答案：
```
From: Matthias Urlichs <urlichs@smurf.sub.org>
Subject: Load average broken ?
Date: Fri, 29 Oct 1993 11:37:23 +0200

The kernel only counts "runnable" processes when computing the load average.
I don't like that; the problem is that processes which are swapping or
waiting on "fast", i.e. noninterruptible, I/O, also consume resources.

It seems somewhat nonintuitive that the load average goes down when you
replace your fast swap disk with a slow swap disk...

Anyway, the following patch seems to make the load average much more
consistent WRT the subjective speed of the system. And, most important, the load is still zero when nobody is doing anything. ;-)

--- kernel/sched.c.orig Fri Oct 29 10:31:11 1993
+++ kernel/sched.c  Fri Oct 29 10:32:51 1993
@@ -414,7 +414,9 @@
    unsigned long nr = 0;

    for(p = &LAST_TASK; p > &FIRST_TASK; —p)
-       if (*p && (*p)->state == TASK_RUNNING)
+       if (*p && ((*p)->state == TASK_RUNNING) ||
+                  (*p)->state == TASK_UNINTERRUPTIBLE) ||
+                  (*p)->state == TASK_SWAPPING))
            nr += FIXED_1;
    return nr;
 }
—
Matthias Urlichs        \ XLink-POP N|rnberg   | EMail: urlichs@smurf.sub.org
Schleiermacherstra_e 12  \  Unix+Linux+Mac     | Phone: …please use email.
90491 N|rnberg (Germany)  \   Consulting+Networking+Programming+etc’ing      42
```

看来在 1993 年以前，Linux 的平均负载也只是反映 runnable 状态的进程。但一个叫 Matthias Urlichs 的人觉得，那些在等待 swap 完成或等待 IO 完成的进程，它们也在消耗资源，系统也正在为它们服务；所以系统的平均负载也应该考虑这些情况，所以他把进程 state 属性为 TASK_UNINTERRUPTIBLE 和 TASK_SWAPPING 的进程也纳入系统平均负载的计算。

当年 Urlichs 是这样想的，那 Gregg 考古发现这段 email 后，于 2017 年发了一封 email 去问 Urlichs：过了 24 年了，你如何评价自己当年对 load average 的修改？Matthias Urlichs 很快回复说：
>The point of “load average” is to arrive at a number relating how busy the system is from a human point of view. TASK_UNINTERRUPTIBLE means (meant?) that the process is waiting for something like a disk read which contributes to system load. A heavily disk-bound system might be extremely sluggish but only have a TASK_RUNNING average of 0.1, which doesn’t help anybody.

这段话清楚明了，要让 load average 直观全面地反应出系统的负载情况，就要让它**既反应出 CPU 的负载情况，又要反应出磁盘的负载情况**。

## 总结
看了《Linux Load Averages: Solving the Mystery》这篇好文，再结合我之前对系统平均负载的理解，可以得出以下总结：
- load average 能够直观地反映出系统的负载情况，但它是一个相对值
- 究竟 load average 高不高，不能单纯考虑 CPU 核心数，**并不是**说若 CPU 单核， load average 大于 1 就高；若 CPU 有 2 个核，load average 大于 2 就高
- 最好一直监控着 load average 的变化情况，若系统一切正常时的 load average 为 x，某些时刻发现 load average 升到数倍 x，就很可能负载过高了
- 当 load average 真的过大时，究竟是 CPU 负载过高，还是磁盘负载过高，还要结合别的工具进行分析

---
参考资料：
- [《Linux Load Averages: Solving the Mystery》](http://www.brendangregg.com/blog/2017-08-08/linux-load-averages.html)
- 《Linux/UNIX 系统编程手册》22.3
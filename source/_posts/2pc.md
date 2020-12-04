---
title: 分布式事务数据一致性方案：两阶段提交协议
date: 2020-03-06 16:15:01
tags:
- 分布式系统
- 分布式事务
categories:
- coding
---

两阶段提交协议（two phase commit）简称 2PC，是一种解决分布式事务数据一致性问题的方案。

<!--more-->

## 基本流程

在 2PC 中有两类角色，姑且分别称为 manager 和 node。在正常情况下，2PC 的流程如下:
- 第一阶段（voting 阶段），manager 把将要发生的数据变化通知各个 node，各个 node 根据各自的业务逻辑，回复 yes 或者 no
    - 如果 node 回复 yes，意味着该次的事务是可以执行的，而且自该时刻起，node 是不能反悔的
    - 如果 node 回复 no，那么意味着该节点通知 manager 不能执行该次事务
- 第二阶段（commit 阶段），manger 根据第一阶段各个 node 回复的情况，确定事务是否能最终执行
    - 如果第一阶段中，各个 node 都回复 yes，那么本阶段 manager 将进行 commit 操作，通知各个 node 确认执行本次事务
    - 如果第一阶段中，只要有一个 node 回复了 no，那么本阶段 manager 将进行 rollback 操作，通知各个 node 本次事务撤销

![](https://github.com/leejianyang/pic/raw/master/2pc/1.png)

## 异常情况

以上是正常的情况，可能会存在一些异常的场景：
- 在 voting 阶段中，manager 向 node 发起 voting 请求，可能会收不到 node 的回复。若发生这种情况， manager 只能选择重试，或者超时后视为 node 回复了 no
- 在 voting 阶段中，如果 node 向 manager 回复了 yes，但后续迟迟收不到 manager 发来下一阶段的 commit 或 rollback 通知。若发生这种情况，node 是无权擅自取消本次事务的，因为它已经承诺支持本次事务的发生，它只能继续等待，或者主动询问 manager
- 在 commit 阶段中，manager 向 node 发起 commit 或 rollbcak 通知，但收不到  node 的回复。这种情况下，manager 只能不断重新发送通知，直至 node 确认为止
- 在 commit 阶段中，如果 manager 发生崩溃。如果发生这种情况的话，manager 必须做好一定的措施，记录下本次事务相关的信息，并设计好相关的机制，防止 node 回复 yes 后无限期等待下去


## 不足之处

两阶段提交协议存在一些不足，主要是 node 回复了 yes 之后，必要时需要加锁或其它的一系列措施，保证事务的后续执行。但如果 node 迟迟收不到 manager 的最终决策，它只能苦等下去，甚至是会影响到别的事务的执行。其中一种修改的方案是三阶段提交协议（three-phase commit protocol），在这里先不展开。

虽然两阶段提交协议有它的不足之处，但依然是实现分布式事务最终一致性时值得借鉴的思想。

## 总结

总体来说，2PC 比较适合对数据一致性要求较高的场景；但并发性不是很高，而且也要注意 manager 单点宕机对整个系统的影响。

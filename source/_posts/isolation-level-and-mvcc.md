---
title: InnoDB 事务隔离级别与MVCC
date: 2020-11-09
tags:
- MySQL
- InnoDB
categories:
- coding
---

本文总结 InnoDB 事务隔离级别和 MVCC 的相关概念。

<!--more-->

## 事务隔离级别

事务隔离级别是为了解决，当有多个事务同时对数据库进行并发读写时，一个事务对数据的修改是如何影响其余事务的读写结果的问题。因为一个事务对数据进行修改，可能会造成其余并发事务的逻辑混乱（得不到预期的数据操作结果），所以 InnoDB 提供了几种级别的事务隔离机制，分别是：
- SERIALIZABLE 串行化
- REPEATABLE READ 可重复读
- READ COMMITTED 读提交
- READ UNCOMMITTED 读未提交

不同强度级别的事务隔离机制，可以解决不同的事务并发问题。常见的事务并发问题有脏读（dirty read）、不可重复读（non-repeatable read）和幻读（phantom read）：
- 脏读：一个事务读到了其余事务未提交的修改结果
- 不可重复读：一个事务多次读到的数据内容不一致（由别的事务修改造成的不一致，而不是由当前事务自身修改造成的不一致）
- 幻读：一个事务的同一个查询，多次执行读到的数据条数不一致

最严格的事务隔离级别是 SERIALIZABLE，在该隔离级别下，如果一个事务对某些数据进行读操作，其余事务就不能对这些数据就行增删改操作了，SERIALIZABLE 可以解决脏读、不可重复读和幻读的问题，但其对并发性能影响较大，并不常用。而最弱的隔离级别——READ UNCOMMITTED，则是什么问题都避免不了，所以也不常用。因而比较常用的事物隔离级别是 REPEATABLE READ 和 READ COMMITTED，REPEATABLE READ 可以解决脏读和不可重复读的问题，READ COMMITTED 则是可以解决脏读的问题。InnoDB 默认的事务隔离级别是 REPEATABLE READ。

无论选择哪种隔离级别，特别是 REPEATABLE READ 和 READ COMMITTED 之间的取舍，需要了解 InnoDB 背后如何实现事务隔离，以及在这些隔离级别下，一些常见的数据查改场景会发生什么事情。

## MVCC
MVCC 是多版本并发控制（multiversion concurrency control）的缩写。「多版本」顾名思义，就是 InnoDB 会为并发修改的数据保存多个版本，这样做是为了给事务提供一致性读（consistent read）的操作能力，而且这种机制的好处是可以减少对数据进行加锁，不会因过多的锁而影响并发性能。

MVCC 的基本工作原理是这样的，每个数据行保存着多个历史版本，每个历史版本中有一个 trx_id 值，该值表示这个版本的数据是由哪一条事务创建的（包括删除数据也同理）。trx_id 可以称为事务ID，是单调递增的数字。而各个事务需要进行一致性非锁定读的时候，就逐个逐个历史版本进行查看，判断对于自己来说该版本是否可见。至于是如何判定是否可见的，用到一个叫做 ReadView 的东西，这个 ReadView 是每个事务各自独立的，ReadView 包括几个概念：
- m_ids：在生成 ReadView 时当前系统中活跃的事务的 trx_id 列表
- min_trx_id：m_ids 中的最小值
- max_trx_id：生成该 ReadView 时，应该分配给下一个事务的 trx_id 值
- creator_trx_id：生成该 ReadView 的事务的 trx_id

当事务进行一致性非锁定读的时候，由新到旧遍历数据的历史版本，若：
- 该版本的 trx_id 与 creator_trx_id 相同，表示该版本是自己修改的记录，当然可见
- 该版本的 trx_id 小于 min_trx_id，表示该版本在本事务开始前已经提交，所以该版本可被本事务可见
- 该版本的 trx_id 大于或等于 max_trx_id，那么该版本本事务是不可见的
- 该版本的 trx_id 在 min_trx_id 和 max_trx_id 之间的话，就要看看 trx_id 是否在 m_ids 列表里面，如果在，则本事务不可见；如果不在，则本事务可见。

接下来看看一些案例。使用以下的数据表结构说明 MVCC 的基本原理：
```sql
CREATE TABLE `t` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `val` int(11) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 简单示例

先看一个最简单的场景，事务隔离级别是可重复读，autocommit=1（下同）：

时刻|事务A|事务B|事务C
---|---|---|---|
t0|insert into t(1, 1);||		
t1||begin();|
t2||select val from t where id=1;|
t3|||update t set val=2 where id=1;
t4||select val from t where id=1;|

根据可重复读的定义，事务 B 在 t4 时刻读到的 val 值依然为 1，那么显然 InnoDB 既保留了(1, 1) 又保留了 (1, 2) 这两个版本。假设事务 A 的 trx_id 是 3，事务 B 的 trx_id 是 4, 事务 C 的 trx_id 是5。那么事务 B 的 ReadView 的 m_ids 是[4]，min_trx_id 是 4，max_trx_id 是 5，creator_trx_id 是 4。在 t4 时刻，id=1 的数据行的版本分别是 [trx_id=3，val=1]和[trx_id=5, val=2]，显然，事务 B 只能读到 trx_id=3 的数据。

### 事务 ID 的生成时间
以下各种情况，事务隔离级别都是 REPEATABLE READ，在 t0 时刻，id=1 的行 val 值都为 1。
情况一：

时刻|事务A|事务B
--|--|---
t1|begin();|
t2||update t set val=2 where id=1;
t3|select val from t where id=1;|

情况二：

时刻|事务A|事务B
--|--|---
t1|begin();|
t2|select val from t where id=2;|
t3||update t set val=2 where id=1;
t4|select val from t where id=1;|

情况三：

时刻|事务A|事务B
--|--|---
t1|begin();|
t2|update t set val=1000 where id=2;|
t3||update t set val=2 where id=1;
t4|select val from t where id=1;|

对比以上三种情况，在情况一的 t3 时刻，事务 A 读到的 val 值是 1；而情况二、三中，在 t4 时刻，事务 A 读到的 val 值都是2。由此可知，生成事务的版本号的实际时机不是事务开始时，而是事务首次进行一致性读的时刻。

### 修改数据的情况
在 t0 时刻，id=1 的行 val 值为 0，事务隔离级别是 REPEATABLE READ，进行如下操作：

时刻|事务A|事务B|事务C
--|---|---|---
t1|begin;||		
t2||begin|
t3|select val from t where id=2;||		
t4||select val from t where id=2;|
t5|||update t set val=2 where id=1;
t6||update t set val=val+1 where id=1;|
t7||select val from t where id=1;|
t8|select val from t where id=1;||

在 t8 时刻，事务 A 读到 val 的值依然是 1，因为事务 A 的事务编号比事务 C 的要小，所以事务 C 对 id=1 的行的更新，事务A 进行一致性读的时候是看不到的；但是 t7 时刻，事务 B 查询到的 val 值是3，意味着在 t6 时刻事务 B 对 val 的修改是基于事务 C 已提交的结果来修改的，它进行的不是一致性读，而是要对数据进行修改，所以要基于最新的已提交事务的数据版本进行修改。

再来看下一个例子，事务隔离级别还是 REPEATABLE READ

时刻|事务A|事务B
---|---|---
t1|begin;|
t2||begin;
t3|update t set val=val+1 where id=1;|
t4||update t set val=val+1 where id=1;

在 t4 时刻，由于事务 A 对 id=1 的行进行修改却未进行提交，所以事务 B 对 id=1 的行的修改会一直被阻塞，直到事务 A 提交或者回滚为止。这里并不是一致性非锁定读的情况，就超出了 MVCC 的范畴了。

再看一种情况，t0 时刻 id=1 的行的 val 值是 1，事务隔离级别还是 REPEATABLE READ。

时刻|事务A|事务B
----|---|---
t1|begin;|
t2||begin;
t3|update t set val=val+1 where id=1;|
t4||select val from t where id=1

这里 t4 时刻事务 B 读到的 val 值还是 1，而且不会被事务 A 在 t3 时刻的修改操作阻塞，因为事务 B 在这里进行的还是一致性非锁定读，还是利用 MVCC 机制进行查询。

### READ COMMITTED 隔离级别
前文已经提到了，READ COMMITTED 就是可以读到别的事务已经提交的数据。具体到 MVCC 机制里面，就是 REPEATABLE READ 在第一次进行一致性非锁定读的时候生成本事务的 ReadView，而 READ COMMITTED 是每一次进行一致性非锁定读的时候都要重新生成一次 ReadView。

-----
参考资料：
- 《MySQL是怎样运行的：从根儿上理解 MySQL》，作者：小孩子4919

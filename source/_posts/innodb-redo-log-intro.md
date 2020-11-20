---
title: InnoDB Redo Log 基本机制
date: 2020-10-29 11:09:39
tags:
- MySQL
- InnoDB
categories:
- coding
---

本文是关于从应用程序开发者视觉需要了解的 redo log 基本知识。

<!--more-->

## 为什么需要 Redo Log

为什么需要 redo log 的存在呢？因为当 InnoDB 需要修改数据（包括增删改）的时候，会先将数据加载进内存的 buffer pool，然后在 buffer pool 里面修改数据，修改完之后是不会马上写入硬盘的。这样做的好处是增加了增删查改操作的效率。但是万一在数据于内存中修改完毕而又未写入硬盘时，MySQL 服务崩溃了，就会造成数据的丢失；而在客户端的角度看来，数据是已经增删改成功的。因而 InnoDB 引入了 redo log 这个东西，redo log 会把事务对数据的修改操作记录下来，MySQL 崩溃重启后，InnoDB 可以利用 redo log 对数据进行恢复，避免数据的丢失。所以简单直接来说，redo log 的作用就是崩溃恢复（crash recovery）。

## Redo Log 的基本工作流程

从文件的角度来看，默认情况下，redo log 有两个文件，分别是 ib_logfile0 和 ib_logfile1，它们的大小是不会无限增长的（可以通过配置项修改文件的个数和最大容量）。这两个文件是交替使用的，ib_logfile0 写满了就写 ib_logfile1，ib_logfile1 写满了就又回到 ib_logfile0 覆盖写入，所以日志数据不是永久不会消失的。还有一点是，文件是以追加的方式写入的，因为日志文件是连续 IO 写入，所以往硬盘写日志的效率比修改数据存储文件的效率高得多。

从日志的写入时机来看，InnoDB 也应用了 Write-Ahead Log（WAL）机制，即先写日志，再修改数据。但是还需要知道的是，写 Redo Log 也不是直接就往硬盘文件中写入的，而是先写到内存中的 log buffer 中，等待合适的时机再往硬盘文件写入。往 log buffer 中写入完数据，并且在 buffer pool 中修改好数据后，就会告诉客户端本次的操作成功了（当然还有别的 undo log 等操作本文先忽略不提）。那在此看来，数据也并不是百分百不会丢失的，因为在 redo log 写入到 log buffer 与实际写入硬盘之间还有一小个时间间隔，这个短暂的间隔通常问题不是很大。

接下来说说 redo log 从 buffer 中写入文件的时机。在以下几种情况，redo log 会实际写入文件：
- log buffer 写满后被迫要将数据写入文件
- 后台线程每秒刷新一次将 log buffer 的数据写入文件
- 当 buffer pool 的数据写入硬盘时，要保证 log buffer 中小于该 LSN（Log Sequence Number，可以简单理解为日志序号） 的 redo log 数据写入硬盘
- 当事务提交后写入文件，但要根据 innodb_flush_log_at_trx_commit 参数的值来决定写入硬盘的时机

当 MySQL 崩溃后重启时，InnoDB 会自动使用 redo log 将那些崩溃前已经写入 buffer pool 但又未最终落盘的数据写入到数据文件中，完成这个步骤后，InnoDB 就会尽快重新接受客户端链接。

## 与 Redo Log 相关的配置项

以下是一些与 Redo Log 调优相关的配置项：
- innodb_log_file_size：日志文件大小
- innodb_log_files_in_group：日志文件个数
- innodb_log_buffer_size：日志 buffer 的大小
- innodb_flush_log_at_trx_commit
    - 当该值为0时，不写入文件，由后台线程每隔一秒写入文件
    - 当该值为1时，写入并刷新到硬盘文件，这是默认值
    - 当该值为2时，写入到操作系统的文件缓存中，每隔一秒刷盘

## 与 Redo Log 相关的监控数据

执行 `show engine innodb status` 命令后，其中一部分是关于 redo log 的状态信息。举个例子：
```
---
LOG
---
Log sequence number 36171122019
Log flushed up to   36171121788
Pages flushed up to 36170513811
Last checkpoint at  36170512861
1 pending log flushes, 0 pending chkp writes
934999 log i/o's done, 69.39 log i/o's/second
```

- Log sequence number 表示当前 log buffer 中最大的 LSN 值
- Log flushed up to 表示当前已经持久化到硬盘文件中的最大的 LSN 值
- Pages flushed up to 几乎可以认为是下次要从 buffer pool 中刷新页面到硬盘数据文件的 LSN 值
- Last checkpoint at 表示当前已经做过 checkpoint 的数据对应的 LSN 值
- 最后一行可以看到 redo log 刷盘的速度

## 综述

本文只是关于 redo log 的基本工作机制，还有大量的技术细节需要深入到 InnoDB 源码才能弄清楚。从应用程序开发者的视觉来看，对 redo log 的基本原理有一个整体上的认知还是很有意义的。

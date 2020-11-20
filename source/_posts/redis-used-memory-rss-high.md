---
title: 为何 Redis 设置了 maxmemory 后还占用大量内存？
date: 2016-08-18 15:05:09
updated: 2016-08-18 15:05:09
tags:
- LRU
- Redis
categories:
- coding
---

最近发现 Redis 用了一段时间后，占用的内存过高，但明明设置了 maxmemory，为何 Redis 进程占用的内存远比 maxmemory 值要高？mem_fragmentation_ratio 又是什么？如何避免 Redis 内存的碎片化呢？

<!--more-->
## 与内存有关的几个监控数据
由 INFO 命令可以看到几个关于内存的数据：
```bash
# Memory
used_memory:6181955720
used_memory_human:5.76G
used_memory_rss:6869622784
used_memory_peak:6444663704
used_memory_peak_human:6.00G
used_memory_lua:35840
mem_fragmentation_ratio:1.11
mem_allocator:jemalloc-3.6.0
```
主要关注的是三个数据：
- used_memory: 存储的数据占用了多少内存
- used_memory_peak：存储的数据占用内存的峰值
- used_memory_rss: 该Redis进程实际占用了操作系统多少内存（应该跟ps命令看到的是一致的）

## maxmemory配置项
maxmemory 限制的是 Redis 存储的数据所占用的内存的最大值，因而设置了 maxmemory 之后，used_memory_peak 是不应该超过 maxmemory 的。

### maxmemory策略
当 used_memory 达到了 maxmemory 的限制后，Redis 将根据配置的「maxmemory-policy」项对 key 进行驱逐。
根据官方文档，maxmemory-policy 有如下的选择：
- noeviction：不删除现有的key，向客户端返回错误
- allkeys-lru：尝试根据LRU算法清理key
- volatile-lru：在设置了过期时间的key中，根据LRU算法尝试清理key
- allkeys-random: 随机删除key
- volatile-random: 在设置了过期时间的key中，随机删除key
- volatile-ttl: 在设置了过期时间的key中，根据ttl值来删除key

### Redis 采用的 LRU 算法
按Redis 2.8 来说，Redis 采用的 LRU 清理 key 的算法是这样的：随机找 n 个 key，然后清理这 n 个 key 中最久未使用的那个。Redis 官方的说法是这样做省内存，而且效果不比真正的 LRU 差。

至于这个 n 等于多少，是可以通过配置项「maxmemory-samples」来设置的，还可以根据实际的 key 的命中率进行调整。

## 为什么 maxmemory 设置后 Redis 占用内存仍过高？
重点来了，**为什么设置了 maxmemory 后，redis 进程占用的内存仍会过高，甚至引发 OOM 呢？**这是因为 maxmemory 只能限制 used_memory，是限制不了 used_memory_rss 的。
官方给出来的解释是：
> Redis will not always free up (return) memory to the OS when keys are removed. For example if you fill an instance with 5GB worth of data, and then remove the equivalent of 2GB of data, the Resident Set Size (also known as the RSS, which is the number of memory pages consumed by the process) will probably still be around 5GB, even if Redis will claim that the user memory is around 3GB. This happens because the underlying allocator can't easily release the memory. For example often most of the removed keys were allocated in the same pages as the other keys that still exist.

大概意思就是由于 Redis 申请内存的时候是一块块地申请的，即便某些 key 被删除后，该块内存中如果还有别的 key，那么整块内存都无法被回收，这就造成 used_memory_rss 值居高不下。当然 Redis 还会重用那块内存的。

因而 Redis 中还有另一个数据叫做「mem_fragmentation_ratio」，就是用 used_memory / used_memory_rss 计算出来的，用来表示当前 Redis 内存的碎片化程度。

## 如何避免 mem_fragmentation_ratio 过高
按照以上的理论，当 used_memory_peak 比 used_memory 高得多的时候，used_memory_rss 就会偏高。而 mem_fragmentation_ratio 的正常值应该是一点多，Redis 的作者有说过大概是 1.4。

有一个避免碎片率过高的说法是禁用系统的「transparent huge pages」，这个是官网文档的说明：
> Make sure to disable Linux kernel feature transparent huge pages, it will affect greatly both memory usage and latency in a negative way. This is accomplished with the following command: echo never > /sys/kernel/mm/transparent_hugepage/enabled.

禁用了这个选项后，系统给程序分配的内存 page 就不会太大，有利于减少 Redis 的碎片率。还有另一个说法就是减少每个 key 的数据大小有利于减小碎片率；然后数据频繁增删也会导致碎片率偏大。

如果 mem_fragmentation_ratio 实在太高那就没办法了，只能重启 Redis >.<

## Redis 内存分配设置的考虑
- 一定要设置 maxmemory 参数，防止内存过高引起 swap 甚至因 OOM 而被 kill 掉
- 出于持久化备份等因素的考虑，要留出 55% 的内存空间；然后假设碎片率是 1.4，因而 maxmemory 大概只能设置为整台机器最大可用内存的 40%



-----
### 参考资料
- [Redis官网: Using Redis as an LRU cache](http://redis.io/topics/lru-cache)
- [Redis官网：Memory allocation](http://redis.io/topics/memory-optimization)
- [Github Issue: Redis 2.8.13 OOM crash even with maxmemory configured](https://github.com/antirez/redis/issues/2136)

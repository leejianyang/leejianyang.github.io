---
title: Redis-py 源码阅读：使用Redis作为分布式锁
date: 2016-04-04 21:14:19
updated: 2016-04-04 21:14:19
tags:
- 锁
- 分布式系统
- Redis
- 源码阅读
- Python
categories:
- coding
---
本文借助 redis-py 的源码简述利用 Redis 实现分布式锁，以及 Redlock 算法。

<!--more-->

看了下 redis-py 源码的目录，发现有一个 lock.py 的模块，里面提供了两个类：Lock 和 LuaLock。Lock 的功能如下：
> A shared, distributed Lock. Using Redis for locking allows the Lock to be shared across processes and/or machines.

感觉借助 Redis 来实现分布式锁是一个不错的选择，于是便看一下源码。

## Lock的实现
### 基本思路
无非就是提供以下几种操作：
- acquire：获得锁
- release：释放锁
- extend：延长锁的占用时间

### acquire
每个锁都是借助 Redis 的「字符串」的数据结构，字符串的 key 是锁的名称，value 是当前占用该锁的线程的标识符。这个标识符必须尽量做到线程与线程间不重复，如何分配呢？源码中使用了 Python 标准库的 uuid 库：
``` Python
token = b(uuid.uuid1().hex)
```
然后我顺便查了下什么时候用 uuid1 什么时候用 uuid4，先看看 Python 的官方文档：
> uuid.uuid1([node[, clock_seq]])
> Generate a UUID from a host ID, sequence number, and the current time. If node is not given, getnode() is used to obtain the hardware address. If clock_seq is given, it is used as the sequence number; otherwise a random 14-bit sequence number is chosen.
>
> uuid.uuid4()
> Generate a random UUID.

如果涉及多台机器之间的 uuid 重复问题的话，最好选择 uuid1，因为 uuid1 利用了 mac 地址，能更好地避免重复。不过官方文档又说了：
>  Note that uuid1() may compromise privacy since it creates a UUID containing the computer’s network address.

然后正常使用的话，uuid4 出现重复的概率非常非常非常低，已经能够很好地保证不重复了。

回到正题上面去，token 值生成了之后还要存起来：
``` Python
self.local = threading.local() if self.thread_local else dummy()
```
self.thread_local 指定了是否使用 thread local storage 来存放 token 值，默认情况下是使用的。什么时候不用呢？文档给出来的其中一个栗子是如果 A 线程获得了锁，并把这个锁交给了 B 线程，并由 B 线程释放锁，那这样用 thread local storage 就不合适了。那如果这里不用 thread local 的话，就会使用 dummy。dummy 也是 redis-py 库自己 diy 出来的类。然而 dummy 什么都没做，仅仅是一个容器- -！
``` Python
class dummy(object):
    """
    Instances of this class can be used as an attribute container.
    """
    pass
```

获得锁的函数很简单：
``` Python
def do_acquire(self, token):
    if self.redis.setnx(self.name, token):
        if self.timeout:
            # convert to milliseconds
            timeout = int(self.timeout * 1000)
            self.redis.pexpire(self.name, timeout)
        return True
    return False
```
利用了 Redis 字符串的 setnx 命令，这个命令用在这里非常适合！如果被占用了就会返回 False。然后还设置了这个键的过期时间，如果客户端没有主动释放这个锁的话，Redis 会帮忙自动释放这个锁。但是为什么要使用 pexpire 这种「毫秒」级别的设置我还不清楚，毕竟时间值是通过「秒」乘以 1000 得到的，并不精确。

然后这个 do_acquire 函数外面还有一层 while 循环，如果使用者想以阻塞的方式获得锁，就会设置一段时间间隔，每隔一段时间去尝试一下获取锁。

### release
锁的释放操作如下：
``` Python
def do_release(self, expected_token):
    name = self.name

    def execute_release(pipe):
        lock_value = pipe.get(name)
        if lock_value != expected_token:
            raise LockError("Cannot release a lock that's no longer owned")
        pipe.delete(name)

    self.redis.transaction(execute_release, name)
```
需要用到 Redis 的「事务」，因为要防止在释放锁的过程中误释放了由别的线程占用的锁。这里的 transaction 函数是由 redis-py 这个库自己封装的，平时可以利用下，其实就是 watch和 pipeline 的组合。顺便说下 Redis 的事务与传统的关系数据库有点不一样，Redis 只保证执行这一堆命令的时候不被别的客户端打断，不过这个在这里不是重点。

### extend
acquire 的时候说过用来存储锁的键是设了过期时间的，那么这里就提供了方法延长被自动释放的时间。代码如下：
``` Python
def do_extend(self, additional_time):
    pipe = self.redis.pipeline()
    pipe.watch(self.name)
    lock_value = pipe.get(self.name)
    if lock_value != self.local.token:
        raise LockError("Cannot extend a lock that's no longer owned")
    expiration = pipe.pttl(self.name)
    if expiration is None or expiration < 0:
        # Redis evicted the lock key between the previous get() and now
        # we'll handle this when we call pexpire()
        expiration = 0
    pipe.multi()
    pipe.pexpire(self.name, expiration + int(additional_time * 1000))

    try:
        response = pipe.execute()
    except WatchError:
        # someone else acquired the lock
        raise LockError("Cannot extend a lock that's no longer owned")
    if not response[0]:
        # pexpire returns False if the key doesn't exist
        raise LockError("Cannot extend a lock that's no longer owned")
    return True
```
基本是那么一回事，但这里没有用封装好的 transaction 函数。

### what's more？
redis-py 提供的 Lock 类就是这么简单，可以说是基本正确。但有一个问题就是如果 Redis 挂掉了那怎么办？如果加一个 slave 的话，也可能会出现数据同步性不一致的问题。因而，官方推荐了名为「Redlock」的算法。

## Redlock
Redlock 的核心思路就是用多个 Redis 实例来避免单点故障带来的问题，获得锁的时候要从多个 Redis 实例获得锁才算成功。官方推荐了其中一个 Python 的按照 Redlock 算法来实现的代码:[redlock-py](https://github.com/SPSCommerce/redlock-py)，那就参照它来看看 Redlock 是怎样的。

``` python
def lock(self, resource, ttl):
    retry = 0
    val = self.get_unique_id()

    # Add 2 milliseconds to the drift to account for Redis expires
    # precision, which is 1 millisecond, plus 1 millisecond min
    # drift for small TTLs.
    drift = int(ttl * self.clock_drift_factor) + 2

    redis_errors = list()
    while retry < self.retry_count:
        n = 0
        start_time = int(time.time() * 1000)
        del redis_errors[:]
        for server in self.servers:
            try:
                if self.lock_instance(server, resource, val, ttl):
                    n += 1
            except RedisError as e:
                redis_errors.append(e)
        elapsed_time = int(time.time() * 1000) - start_time
        validity = int(ttl - elapsed_time - drift)
        if validity > 0 and n >= self.quorum:
            if redis_errors:
                raise MultipleRedlockException(redis_errors)
            return Lock(validity, resource, val)
        else:
            for server in self.servers:
                try:
                    self.unlock_instance(server, resource, val)
                except:
                    pass
            retry += 1
            time.sleep(self.retry_delay)
    return False
```
基本思路是，尝试获得锁的操作前，先记下当前的时间戳 t1，然后逐个逐个实例（例如有 5 个 Redis 实例）去获得锁。遍历一遍后算算从多少个实例里面获得了锁，这里限定的是超过半数实例「(len(connection_list) // 2) + 1」。除了获得实例的数量外，还要比较一下获得锁过程中的总用时，如果获得锁的时间比锁的有效时间还有大，那就没意义了。如果获得锁失败，那么要把已获得锁的实例释放掉。

释放锁就很简单了，直接把已获得锁的实例都释放掉就好。

Redlock 是 Redis 官方推荐的实现分布式锁的算法，如果对锁的要求很严格，可以尝试使用它。更详细的说明可见官网文章[《Distributed locks with Redis》](http://redis.io/topics/distlock)

## 综述
借助 Redis 来实现分布式锁是一个不错的选择，特别是 Redis 提供的事务特性比传统的关系型数据库弱，如果需要事务性地访问的数据也存储于 Redis中，那么使用 Redis 作为分布式锁顺理成章，而且这种方式的效率要比单纯使用 Watch 的要高。

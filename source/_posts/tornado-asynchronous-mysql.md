---
title: Tornado 中异步非阻塞地使用 MySQL
date: 2015-07-22 11:02:59
updated: 2015-07-22 11:02:59
tags:
- Python
- MySQL
- Tornado
- 异步IO
categories:
- coding
---

Tornado 标榜的是 asynchronous 和 non-blocking，然而，很多时候一不小心一不讲究就会把整个 tornado 阻塞住，特别是做 MySQL 操作时。本文简述两种在 tornado 中异步无阻塞地使用 MySQL 的方法。
<!--more-->

## 常见的阻塞情况
``` python
class Handler(tornado.web.RequestHandler):
    def get(self):
        connection = pymysql.connect()        # 连接数据库的相关参数省略

        with connection.cursor() as cursor:
            sql = '这里应该有一条SQL查询语句'
            cursor.execute(sql)
            result = cursor.fetchone()

        self.write(str(result['id']))
```
以上这种情景，tornado 会在 pymysql.connect() 和cursor.execute(sql) 的地方阻塞住，无法响应其他 request。

## 无阻塞方案1：使用Tornado-MySQL

Tornado-MySQL 是 PyMySQL 的一个分支，它支持 Tornado 的异步特性。详情请戳[git hub](https://github.com/PyMySQL/Tornado-MySQL)

看代码示例：
``` python
import tornado.gen
import tornado.web
from tornado_mysql import pools

MYSQL_POOL = pools.Pool(dict(host='127.0.0.1', port=3306, user='test', db='test'), max_idle_connections=5, max_open_connections=10)


class Handler(tornado.web.RequestHandler):
    @tornado.web.asynchronous
    @tornado.gen.coroutine
    def get(self):

        cur = yield MYSQL_POOL.execute('SELECT * FROM table_1 LIMIT 1')

        self.write('ok')
        self.finish()
```

## 无阻塞方案2：使用Celery
[Celery](http://www.celeryproject.org/)是一个异步的任务队列，它能很方便地支持分布式扩展，因而很适合将 tornado 中的长时间的阻塞工作交由 Celery 来完成。
而要 Celery 配合 Tornado 一起工作，需要借助一个名为 tornado-celery[请戳pypi页面](https://pypi.python.org/pypi/tornado-celery)的包。

看栗子，首先是 tornado handler 的代码：
``` python
import tornado.gen
import tornado.web
import tcelery

import mysql_task

tcelery.setup_nonblocking_producer()

class Handler(tornado.web.RequestHandler):

    @tornado.web.asynchronous
    @tornado.gen.coroutine
    def get(self):
        result = yield tornado.gen.Task(mysql_task.mysql_test.apply_async)

        self.write(result)
        self.finish()
```

而具体操作 MySQL 的部分则摆在 mysql_task.py 文件中。至于 celery worker 的写法这里就省略掉了。

**特别需要注意的是：tornado-celery目前只支持AMQP的backend。**

## 总结
- 使用 tornado 框架的话，MySQL 的操作，以及其它耗时长的语句，一定注意要写成非阻塞的形式
- 如果是 MySQL 的话，借助 Tornado-MySQL 就很好了
- 别的耗时任务可考虑 celery 或者 gearman 等任务队列

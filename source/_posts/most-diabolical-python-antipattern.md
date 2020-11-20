---
title: 最恶劣的 Python 反模式
date: 2015-08-03 21:17:58
updated: 2015-08-03 21:17:58
tags:
- Python
categories:
- coding
---

本文源于 Aaron Maxwell 的[《The Most Diabolical Python Antipattern》](https://realpython.com/blog/python/the-most-diabolical-python-antipattern/)，原文作者指出的最 Diabolical 的 Python 反模式是关于异常捕获方面的一个很糟糕的写法。我大胆猜测，这种写法应该比较常见。

<!--more-->

代码示例如下：
``` python
try:
    do_something()
except:
    pass
    # 也有可能简单地用print输出一些东西
```

像以上的代码那样，不加分类地捕获所有异常，然后什么都不干。当然这样做程序不会崩溃，但可能会埋下重大隐患，出了问题的时候调试起来也非常麻烦。

为什么很多人都会这样做呢？通常情况是图个省事；或者是程序猿也不知这段代码会抛出什么异常；或者某段代码在绝大部分时候都运行良好，但突然崩溃了，开发人员懒或者不够时间仔细 debug，就粗劣地 try...except... 一下。

但从现在起必须意识到以上的做法是罪恶的根源，一时的简单省事可能导致日后调试起来非常非常麻烦。所以一定要用正确的方式捕获异常，并且，一定要详细地写日志！一定要详细地写日志！一定要详细地写日志！重要的事情说三遍。

--------

原文作者建议将以下内容加入自己或者团队的代码规范中：

> If some code path simply must broadly catch all exceptions – for example, the top-level loop for some long-running persistent process – then each such caught exception must write the full stack trace to a log or file, along with a timestamp. Not just the exception type and message, but the full stack trace.
>
> For all other except clauses – which really should be the vast majority – the caught exception type must be as specific as possible. Something like KeyError, or ConnectionTimeout, etc.

具体到代码来说应该怎么写呢？可以这样做：
``` python
import logging

logging.basicConfig(level=logging.DEBUG, format='[%(levelname)s] %(asctime)s: %(message)s')

try:
    1 / 0
except Exception as ex:
    # 这样既记下捕获时间戳，也标记了一些异常的信息，还输出整个stack trace
    logging.exception('catch some exception')
```

如果不使用标准库提供的 logging 模块，也可以这样：
``` python
import sys
import traceback

def log_traceback(ex, ex_traceback):
    tb_lines = traceback.format_exception(ex.__class__, ex, ex_traceback)
    tb_text = ''.join(tb_lines)
    exception_logger.log(tb_text)

try:
    x = get_number()
except Exception as ex:
    # Here, I don't really care about the first two values.
    # I just want the traceback.
    _, _, ex_traceback = sys.exc_info()
    log_traceback(ex, ex_traceback)
```

Python 3 的版本则是这样的:
``` python
# The Python 3 version. It's a little less work.
import traceback

def log_traceback(ex):
    tb_lines = traceback.format_exception(ex.__class__, ex, ex.__traceback__)
    tb_text = ''.join(tb_lines)
    # I'll let you implement the ExceptionLogger class,
    # and the timestamping.
    exception_logger.log(tb_text)

try:
    x = get_number()
except Exception as ex:
    log_traceback(ex)
```


最后总结如下：

- 尽可能精准地根据可能出现的异常类型捕获异常；
- 对于某些情况，例如某些调用层级比较高的循环或函数，可以不区分类型地捕获所有异常；
- 捕获异常的时候一定要打日志，而且一定要记下整个 stack trace 和发生异常的时间。

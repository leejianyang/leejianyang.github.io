---
title: 关于 PDO 的 ATTR_EMULATE_PREPARES 属性
date: 2018-01-25 11:36:50
updated: 2018-01-25 11:36:50
tags:
- SQL注入
- PHP
- MySQL
categories:
- coding
---

看到一些关于防范 SQL 注入的文章推荐把 PDO 的「emulate prepared statements」功能关闭掉，说这样做能够提高安全性，因而研究下 pdo 的 ATTR_EMULATE_PREPARES 属性究竟是什么。
<!--more-->

## 官方文档的定义
先来看看官方文档对该属性的定义：
> Enables or disables emulation of prepared statements. Some drivers do not support native prepared statements or have limited support for them. Use this setting to force PDO to either always emulate prepared statements (if TRUE and emulated prepares are supported by the driver), or to try to use native prepared statements (if FALSE). It will always fall back to emulating the prepared statement if the driver cannot successfully prepare the current query. Requires bool.  

以上这段定义有几个关键点：
- pdo 如何对 prepared statements 进行「emulate」？
- 到底「emulate prepared statement」和「native prepared statement」的区别是什么？
- 哪些 driver 支持 native prepared statement？
- 这个属性的默认值是什么？

## ATTR_EMULATE_PREPARES 设为 true 的情况
顾名思义，当 ATTR_EMULATE_PREPARES 属性设为 true 时，其实并没有真正使用到 MySQL 的 prepared statements 的相关功能。仅仅在 PDO 中对 prepared statements 进行「仿真」。做法貌似是对要进行绑定的参数进行「escape」，然后再拼接成真正的SQL语句发送给 MySQL。

## emulate 和 native 的区别
- 使用 emulate prepared statement 的话，防注入的相关过程在 application 层面进行；使用 native prepared statement 的话，防注入的相关过程在 MySQL Server 层面进行
- 使用 emulate prepared statement 的话，直接往 MySQL 发送 SQL 语句；使用 native prepared statement 的话，分开向 MySQL 发送 prepare statement 和 execute statement

## PDO_MYSQL 支持 native prepared statement
根据 [官方文档](http://php.net/manual/en/ref.pdo-mysql.php) 的说明， PDO_MYSQL 是支持 native prepared statement 的：
> PDO_MYSQL will take advantage of native prepared statement support present in MySQL 4.1 and higher. If you're using an older version of the mysql client libraries, PDO will emulate them for you.  

至于其他的 driver 就没有查了，暂时用不上，有需要的时候再看文档。

## ATTR_EMULATE_PREPARES 的默认值
由 PDO 的 getAttribute 函数可以看到，ATTR_EMULATE_PREPARES 的默认值是为 true 的。

## 注意事项
- stack overflow 上面有人提到，如果要把 ATTR_EMULATE_PREPARES 设为 false，编译 PHP 时 MySQL驱动应该应该使用 mysqlnd

## 结论
暂时觉得应该把 ATTR_EMULATE_PREPARES 属性设置为 false。

----
参考资料
- [PDO MySQL: Use PDO::ATTR_EMULATE_PREPARES or not?](https://stackoverflow.com/questions/10113562/pdo-mysql-use-pdoattr-emulate-prepares-or-not)
- [PDO防注入原理分析以及使用PDO的注意事项](http://zhangxugg-163-com.iteye.com/blog/1835721)

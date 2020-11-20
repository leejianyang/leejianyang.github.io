---
title: PHP 后期静态绑定
date: 2019-12-16 23:00:00
tags:
- PHP
- 面向对象
categories:
- coding

---

后期静态绑定（Late Static Binding，简称静态绑定)是用于有继承情况出现的场景的。在定义方法的时候不确定调用的是继承层级中哪个类的方法，而是要到实际运行的时候才能确定调用的到底是哪个类中的方法。

<!--more-->

先要区分 non-forwarding call 和 forwarding call 的概念:
- 转发调用（forwarding call）:
	- `self::`
	- `parent::`
	- `static::`
	- `forward_static_call()`
- 非转发调用（non-forwarding call）
	- 显式指定类名地调用静态方法，例如`ClassA::function()`
	- 调用非静态方法

关键是怎么确定 static 究竟指代的是哪个类？运行机制是：记下最后一次非转发调用时所属的类，用那个类来替代 static。

**例子1：基本用法**
```php
<?php
class A
{
    public static function whichClass()
    {
        echo 'A' . PHP_EOL;
    }

    public static function do()
    {
        // 在这里还不确定调用的是哪个类的 whichClass 方法
        static::whichClass();
    }
}

class B extends A
{
    public static function whichClass()
    {
        echo 'B' . PHP_EOL;
    }
}

A::do();		// 输出 A
B::do();      // 输出 B
```

**例子2：在非静态方法中也能用**
```php
<?php
class A
{
    const TAG = 'A';

    public function printTag()
    {
        echo static::TAG . PHP_EOL;
    }

}

class B extends A
{
    const TAG = 'B';
}

(new A())->printTag();	// 输出 A
(new B())->printTag();   // 输出 B
```

**例子3：稍微复杂一点的例子**
```php
<?php
class A
{
    protected static function output()
    {
        echo 'A' . PHP_EOL;
    }

    protected static function test()
    {
        static::output();
    }
}

class B extends A
{
    protected static function output()
    {
        echo 'B' . PHP_EOL;
    }
}

class C extends B
{
    protected static function output()
    {
        echo 'C' . PHP_EOL;
    }

    public function f()
    {
        B::test();
        static::test();
    }
}

(new C())->f();
```
输出：
```
B
C
```

----

参考资料:
- [PHP手册 - Late Static Bindings](https://www.php.net/manual/en/language.oop5.late-static-bindings.php)

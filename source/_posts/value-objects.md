---
title: 领域驱动设计（一）：Value Objects
date: 2020-09-25 14:11:31
tags:
- DDD
categories:
- coding
---

本文关于领域驱动设计的其中一个基本概念：值对象。

<!--more-->

值对象的特点是，它是用来表示一个值，而不是一个实体的。例如「张三这个人」是一个实体，而「张三这个名字」是一个值。无论张三身上的特征发生什么变化，张三还是那个实体；但「张三」和「张四」就是完全不同的两个值了。所以在比较值对象的时候，只要两个值对象的所有属性是一模一样的，就可以认为两个值对象是相等的。

例如定义如下的值对象：
```php
class Time
{
    private $hour;
    private $miute;
    private $second;
}

$time1 = new Time();
$time2 = new Time();
```
当两个值对象 $time1 和 $time2 的三个属性都是完全相等的时候，在语义上就可以认为这两个对象是相等的。

但是，如果是别的对象：
```php
class Person
{
    private $name;
    private $gender;
}
```
像以上的 Person 类就不属于值对象了，因为他表示的是实体，意味着就算两个对象（两个人）拥有相同的名字和性别，语义上都不能把它们视为相同的人。所以实体通常会带上身份标识符。

基于值对象这种语义上的特性，需要注意的是，在代码设计上要做到两点（1）值对象的属性在初始化之后就不能变化，换言之，它是一个不可变对象。（2）比较两个值对象时，所有属性都相等，那么两个值对象就可以认为是相等。

举个例子：
```php
class Point
{
    private $x;
    private $y;

    public function __construct($x, $y)
    {
        $this->x = $x;
        $this->y = $y
    }

    // 这个方法表示点发生移动，但这个方法返回的是新的值对象，并不修改原对象的值
    public function move($x, $y)
    {
        return self($this->x+$x, $this->y+$y);
    }

    public function getX()
    {
        return $this->x;
    }

    public function getY()
    {
        return $this->y;
    }
}
```

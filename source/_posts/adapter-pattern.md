---
title: 【设计模式】适配器模式
date: 2020-11-22 15:55:16
tags:
- 设计模式
categories:
- coding
---

本文是设计模式系列文章之一的「适配器模式」。

<!--more-->

## 应用场景

应用场景是这样的，已存在一个类 A，程序的某个地方需要接收实现了某个接口的对象，现在想使用类 A 的实例，但 A 没有完全实现该接口，而且又不能直接修改类 A 的源码，或者不想直接修改类 A 的源码。这个时候可以在类 A 的基础上再加上一层适配器。类比生活中的例子，就好像新款的 iPhone 手机的耳机接口变成了 USB-C，但是很多耳机都是 3.5 毫米接口的，要使用这些耳机听歌怎么办？那就加一层转换器，这就是适配器模式背后的思想。

## 代码示例

有一个这样的接口：
```php
<?php

interface IBook
{
    public function getPage();
    public function open();
    public function turnPage()
};
```

有一个现存的类，不能修改源码，或不想修改源码：
```php
<?php

class Kindle
{
    private $totalPages;
    private $page;

    public function pressNext()
    {
        // 省略具体实现
    }

    public function getPage()
    {
        // 省略具体实现
    }

    public function unlock()
    {
        // 省略具体实现
    }
}
```

某个地方需要接收实现 IBook 接口的对象，而显然 Kindle 类的实例是无法满足的，那么就需要多一个适配器:
```php
<?php

class KindleBookAdapter implements IBook
{
    private $adaptee;

    public function __construct(Kindle $adaptee)
    {
        $this->adaptee = $adaptee;
    }

    public function getPage()
    {
        return $this->adaptee->getPage();
    }

    public function open()
    {
        return $this->adaptee->unlock();
    }

    public function turnPage()
    {
        // 基于 Kindle 类的 pressNext 方法实现该方法所需的功能
    }
}
```

## 总结

适配器模式是结构型设计模式的一种。适配器模式还是比较简单的，它背后的思想就是，如果要改变一个类的接口，可以引入一个中间层（中间类），但这个中间层是不改变原有类实现的功能的。

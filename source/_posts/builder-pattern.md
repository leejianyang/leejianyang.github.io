---
title: 【设计模式】建造者模式
date: 2020-11-21 16:37:32
tags:
- 设计模式
categories:
- coding
---

本文是设计模式系列文章之一的「建造者模式」。

<!--more-->

## 应用场景

有个复杂对象，初始化这个对象的属性包含多个步骤，而不同的场景下这些构建步骤的顺序或者每个步骤的具体执行方式是不一样的。在这种情景下就可以引入建造者模式。

## 代码示例

```php
<?php
class Product
{
    private $partA;
    private $partB;
    private $partC;

    public function setPartA($params)
    {
        // 执行一系列步骤初始化 $partA
    }

    public function setPartB($params)
    {
        // 执行一系列步骤初始化 $partB
    }

    public function setPartC($params)
    {
        // 执行一系列步骤初始化 $partC
    }
}
```

Product 类是一个复杂对象，包含很多个属性，可以通过调用一系列的方法来初始化这些属性。

```php
<?php

abstract class AbstractBuilder
{
    protected $product;

    public function __construct()
    {
    	$this->product = new Product();
    }

    abstract function buildPartA();
    abstract function buildPartB();
    abstract function buildPartC();

    public function getProduct()
    {
        return $this->product;
    }
}

class BuilderOne extends AbstractBuilder
{
    public function buildPartA()
    {
        // 按照该场景的需求构建 PartA
        $this->product->setPartA();
    }

    public function buildPartB()
    {
        // 按照该场景的需求构建 PartB
        $this->product->setPartB();
    }

    public function buildPartC()
    {
        // 按照该场景的需求构建 PartC
        $this->product->setPartC();
    }
}

class BuilderTwo extends AbstractBuilder
{
    public function buildPartA()
    {
        // 按照该场景的需求构建 PartA
        $this->product->setPartA();
    }

    public function buildPartB()
    {
        // 按照该场景的需求构建 PartB
        $this->product->setPartB();
    }

    public function buildPartC()
    {
        // 按照该场景的需求构建 PartC
        $this->product->setPartC();
    }
}
```

Product 类的具体初始化行为由具体的 Builder 子类来实现，所有的 Builder 类都继承抽象类 AbstractBuilder。

```php
<?php

class Director
{
    protected $builder;

    public function __construct($builder)
    {
        $this->builder = $builder;
    }

    public function construct()
    {
        $this->builder->buildPartA();
    	$this->builder->buildPartB();
    	$this->builder->buildPartC();

    	return $this->builder->getProduct();
    }
}
```

引入一个 Director 类，在 Director 类里面按一定的逻辑去调用 Builder 实例的方法去初始化 Product 对象，换言之 Builder 的使用逻辑是在 Director 里面配置决定的。这样的好处是隔离了 Product 的使用和创建过程，这也是工厂模式中应用的设计思想。在不同的场景下，可能各个 Builder 方法的执行顺序不一样，或者有些 Builder 方法是不会执行到的；这样的话引入 Director 类使得整个设计更灵活，不会向使用者暴露细节。在更复杂场景下 Director 类可以进行更多精细化的配置。

使用者这样去获取 Product 类：
```php
<?php

$builder = new BuilderTwo();
$product = (new Director($builder))->getProduct();
```

## 总结

建造者设计模式背后的思想是，当一个对象的初始化过程由多个复杂多样的步骤构成时，可以引入多个不同的 Builder 来具体实现不同场景的初始化过程。而再引入一个 Direcotr 去精细化控制各个 Builder 的执行过程。这种对象的构造形式有点像工厂的车间，各个型号的产品都有个相同的模子，不同的流水行生产处不同型号的产品，而车间工人在流水线上进行操作，决定了产品构建的具体步骤。

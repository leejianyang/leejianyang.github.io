---
title: 【设计模式】工厂模式
date: 2020-11-20 14:51:27
tags:
- 设计模式
categories:
- coding
---

本文是设计模式系列文章之一的「工厂模式」。

<!--more-->

工厂模式是创建型设计模式的一种，它的作用是将对象的创建和使用分离开来，使得对象的类型发生变化时，尽量减少对对象使用方的影响。工厂模式有几个细分的模式：简单工厂、工厂方法和抽象工厂。这三种设计模式背后的思想都是一样的，只是在应对不同复杂度的场景时有不同的实现手段。

## 简单工厂

```php
<?php

abstract class AbstractClass
{
   abstract public function doSomething();
}

class A extends AbstractClass
{
    function public doSomething() {
        // 具体实现
    }
}

class B extends AbstractClass
{
    function public doSomething() {
        // 具体实现
    }
}

class Factory
{
    function public create($type = null) : AbstractClass
    {
        // 有一种情况，$type 不从工厂的调用方传递过来，而是从别的地方读取
        if ($type == null) {
            $type = getTypeFromAnotherComponent();
        }

        if ($type == 'A') {
            return new A();
        } elseif ($type == 'B') {
            return new B();
        }
    }
}
```

以上就是简单工厂的实例，这种设计模式的好处是用户不必关心对象是如何创建的，拿来即用。

以下是调用方的代码：
```php
<?php
$obj = (new Factory())->create();
$obj->doSomething();
```
以后需求发生了变化，多了类 C、D、E、F、G，只要它们是继承 AbstractClass 的，调用方的代码都不需要发生改变。

当然啦，Factory 的 create 方法可以变成 static，思路上都是一样，有些地方会把这称为静态工厂。

## 工厂方法

工厂方法模式由上面的简单工厂进化而来。为什么要进化呢？因为如果 AbstractClass 的子类非常多，或者实例化 AbstractClass 的子类的过程相当复杂，那么上述简单工厂的 Factory 类的 create 方法内部就会过于臃肿复杂，修改起来不容易。

那就进化一下：
```php
<?php

abstract class AbstractClass
{
   abstract public function doSomething();
}

class A extends AbstractClass
{
    public function doSomething() {
        // 具体实现
    }
}

class B extends AbstractClass
{
    public function doSomething() {
        // 具体实现
    }
}

interface IFactory
{
    public function create();
}

class FactoryA implements IFactory
{
    public function create()
    {
        // 假设一下实例化 A 的过程很复杂
        return new A();
    }
}

class FactoryB implements IFactory
{
    public function create()
    {
        // 假设一下实例化 A 的过程很复杂
        return new B();
    }
}

class Factory
{
    public function create($type = null) : AbstractClass
    {
        return $this->createFromFactory($type);
    }

    private function createFromFactory($type = null)
    {
        if ($type == null) {
            $type = getTypeFromAnotherComponent();
        }

        if ($type == 'A') {
            return (new FactoryA())->create();
        } elseif ($type == 'B') {
            return (new FactoryB())->create();
        }
    }
}
```

调用方的代码还是保持不变的：

```php
<?php
$obj = (new Factory())->create();
$obj->doSomething();
```

如果从以上这个如此简陋的示例来看，基本看不出工厂方法模式比简单工厂模式的优势。主要是工厂模式方法背后的思想：把具体类的实例化过程从工厂自身之中分离出来，降低工厂类的代码复杂度，引入新的类时更灵活。其实简单工厂模式和工厂方法模式没有必要作严格的区分；代码简单的时候就是前者，代码复杂的时候自然会将创建具体类的代码分离出来，就成了工厂方法模式。

### 更现实的例子：依赖注入容器

下面来看一个更接地气的例子——依赖注入容器，应用的也是工厂方法的设计思想。

```php
class Container
{
    protected $services = [];

    public function get($name)
    {
        $service = $this->services[$name];
        return new $service();
    }

    public function set($name, $factory)
    {
        $this->services[$name] = $factory;
    }
}
```

以上就是一个最简单版的依赖注入容器。将生成对象的工厂注入到依赖注入容器里面：
```php
<?php

class A
{
    public function doSomething()
    {
        // 具体实现
    }
}

class Factory
{
    public function __invoke()
    {
        return new A();
    }
}

$container = new Container();
$container->set('A', new Factory());
```

使用者直接从依赖注入容器里面取出对象：
```php
<?php

$container = new Container();
$container->get('A');
$obj->doSomething();
```

以上就是一个应用工厂方法的现实例子了。

## 抽象工厂

抽象工厂模式不常用，了解一下即可。抽象工厂是为了解决什么问题呢？上面的简单工厂和工厂方法是一个工厂类生成一种类型的对象实例，而抽象工厂则是一个工厂产生一组对象。

```php
<?php

abstract class CPU
{
    // CPU 抽象类
}

class CpuIntel extends CPU
{
    // CPU 子类
}

class CpuAMD extends CPU
{
    // CPU 子类
}

abstract class Memory
{
    // 内存抽象类
}

class MemoryDDR3 extends Memory
{
    // 内存子类
}

class MemoryDDR4 extends Memory
{
    // 内存子类
}

interface ComputerFactory
{
    abstract public function createCPU;
    abstract public function createMemory;
}

class ComputerModelA implements ComputerFactory
{
    public function createCPU
    {
        return new CpuIntel();
    }

    public function createMemory
    {
        return new MemoryDDR3();
    }
}

class ComputerModelB implements ComputerFactory
{
    public function createCPU
    {
        return new CpuAMD();
    }

    public function createMemory
    {
        return new MemoryDDR4();
    }
}
```

在实际使用中，只要决定好了使用哪个工厂，就可以一下子生成一组相关的对象，减少工厂类的数量。缺点也有，就是按上面的例子来说，如果 CpuAMD 的构造函数发生了变化，那么相应的工厂函数可能就要跟着变化，略微繁琐。


## 总结

工厂模式的使用场景是有一些类，这些类都继承相同的基类或者实现相同的接口，把这些类的创建和实现分离开来，也就是说如果在 A 类里面使用了 B 类，就不要在 A 类中创建 B 类，反之亦然。这样做背后的设计思想遵循了单一职责原则和开闭原则，使得软件行为发生变更时，更有利于扩展或维护代码。

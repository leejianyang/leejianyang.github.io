---
title: 【设计模式】装饰器模式和代理模式
date: 2020-11-26 15:10:10
tags:
- 设计模式
categories:
- coding
---

本文是设计模式系列文章，介绍了装饰器模式和代理模式。

<!--more-->

装饰器模式和代理模式都属于设计模式分类中「结构型模式」的类别。将它们写在同一篇文章中，是因为它们实在太像了。装饰器模式有「被装饰类」（通常称为 Concrete Component），代理模式有「被代理类」（通常称为 Service）；而装饰器模式有装饰器类，代理模式有代理类；无论装饰器类还是代理类，它们都把原始对象采用组合的形式组织起来，装饰器类和代理类都提供与被组合对象相同的接口。所以从代码结构来看两者是非常像的。接下来本文具体介绍什么是装饰器模式，什么是代理模式，以及两者的异同。

## 装饰器模式

装饰器模式通过将原始对象组合进装饰器中的方式，为原始对象添加行为。

### 装饰器模式代码示例

这个将要被装饰的类：
```php
<?php

interface IClient
{
    public function get();
    public function post();
}

class HTTPClient implements IClient
{
    public function get($url, $params)
    {
        // 进行 HTTP Get 请求
    }

    public function post($url, $data)
    {
        // 进行 HTTP Post 请求
    }
}
```

HTTPClient 类负责进行 HTTP 请求。现在需要添加一个打日志的功能，那么没必要把打日志的功能加入 HTTPClient 中，而是添加一个装饰器。

```php
<?php

class HTTPLogDecorator implements IClient
{
    private $httpClient

    public function __construct(IClient $httpClient)
    {
        $this->httpClient = $httpClient;
    }

    public function get($url, $params)
    {
        // 发送请求前打日志
    	$result = $this->httpClient->get($url, $params)
    	// 收到响应数据后打日志
    	return $result;
    }

    public function post($url, $data)
    {
        // 发送请求前打日志
    	$result = $this->httpClient->post($url, $params)
    	// 收到响应数据后打日志
    	return $result;
    }
}
```

通过把原对象组合进装饰器的方式，达到了对原对象添加行为的目的。还需要知道的是，多个装饰器是可以组合起来使用的，例如：
```php
<?php

$obj = new ConcreteComponent();
$decoObjA = new DecoratorA($obj);
$decoObjB = new DecoratorB($decoObjA);
```

以上代码将原对象「装饰」了两次，等于给原对象丰富了两层功能，需要注意的是 ConcreteComponent 类，DecoratorA 类和 DecoratorB 类都应该实现相同的接口。


### 装饰器模式的使用场景

装饰器模式通常用于：
- 在运行时动态地给对象添加额外行为时
- 当无法通过继承的方式为原对象添加行为时（而且组合优于继承嘛）

## 代理模式

代理模式通过将原对象组合进代理类的方式，避免对原对象的直接访问，从而控制对原对象的使用。代理模式并没有在原对象的基础上增加行为。

### 代理模式代码示例

```php
<?php

interface ServiceInterface
{
    public function operation();
}

class Service implements ServiceInterface
{
    public function operation()
    {
        // 实现 operation 方法
    }
}

class Proxy implements ServiceInterface
{
    protected $service;

    // 也可以不通过构造函数的方式组合进来
    public function __construct(ServiceInterface $service)
    {
        $this->service = $service;
    }

    public function operation()
    {
        // 控制对原对象的访问，例如权限校验之类的
        if ($this->checkAuth()) {
            $this->service->operation();
        }
    }
}
```

如前所述，代理模式通过将原对象（Service 类）组合进代理类，达到控制原对象访问的目的。

### 代理模式的使用场景

通常在以下场景使用代理模式：
- 实现被代理对象的懒加载
- 拦截对被代理对象的访问行为（权限校验等）
- 缓存被代理对象行为的结果值

## 装饰器模式和代理模式的区别

由于在代码结构上装饰器模式和代理模式是非常非常接近的，所以人们经常疑惑究竟这两种设计模式有何区别？

首先引用 GoF 的原文：
> Although decorators can have similar implementations as proxies, decorators have a different purpose. A decorator adds one or more responsibilities to an object, whereas a proxy controls access to an object.
>
>Proxies vary in the degree to which they are implemented like a decorator. A protection proxy might be implemented exactly like a decorator. On the other hand, a remote proxy will not contain a direct reference to its real subject but only an indirect reference, such as "host ID and local address on host." A virtual proxy will start off with an indirect reference such as a file name but will eventually obtain and use a direct reference.

我认为这两种设计模式更多是语义上的区别。装饰器模式的作用是**扩展**原对象的行为，而代理模式并不会扩展原对象的行为。

## 总结

装饰器模式和代理模式背后的核心思想都是通过引入中间层来改变对原对象的使用，有时是为原对象增加功能，有时是限制对原对象的功能，也有时是利用的组件（例如缓存）来达到与直接使用原对象相同的目的。正因为这两种设计模式如此之像，可以令人意识到理解和使用设计模式并不需要过于拘泥于名字，而是要去掌握一种套路，要意识到可以通过什么手段（例如组合、提供相同接口）来达到什么目的。

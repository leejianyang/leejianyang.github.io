---
title: 【设计模式】装饰器模式
date: 2020-11-22 16:07:15
tags:
- 设计模式
categories:
- coding
---

本文是设计模式系列文章之一的「装饰器模式」。

<!--more-->

## 应用场景

已经存在一个类 A，在进行类 A 的操作的时候，需要执行额外的行为，而这些行为是与类 A 无关的，不影响类 A 的正常行为。这个时候，可以不直接修改 A 的源码，而是在类 A 的基础上加多一层东西。

常见的场景就是执行某些方法的前后需要打日志。

## 代码示例

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

现在要在请求的前后打日志，但是打日志不属于 HTTPClient 的行为，或者说不能修改 HTTPClient 的源码，或者有些时候不需要打日志，那么就不去修改 HTTPClient 的代码，而是加多一层装饰器：
```php
<?php

class HTTPLogDecorator implements IClient
{
    private $httpClient

    public function __construct($httpClient)
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
以上就是装饰器的简单示例代码。

## 总结

装饰器模式是结构型模式的一种。其背后的思想是，要往已存在的类添加额外的行为，而这种行为是与原类无关的，那么可以引入一个中间层（中间类），但这个中间层是不会改变原被装饰类的接口的，而且代码的调用者可以将装饰前后的类任意替换，不破坏客户端的代码。

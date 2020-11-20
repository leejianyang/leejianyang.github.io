---
title: Slim Framework 源码阅读（三）：Dependency Container
date: 2020-05-25 17:24:55
tags:
- Slim
- 源码阅读
- Web开发
- PHP
- PSR
- 依赖注入
cotegories:
- coding
---


本文是 Slim Framework 源码阅读的第三篇，主要关于 Slim 如何支持依赖注入容器。Slim 源码的版本是 4.5.0。

<!--more-->

Dependency Container 是 Slim 的核心组件之一。Slim 没有自己定义 Dependency Container 的接口，而是直接使用 Psr\Container\ContainerInterface 接口，这个接口很简单，只需要实现两个方法。具体的内容可以看看 PHP-FIG 的文档：[PSR-11: Container interface
](https://www.php-fig.org/psr/psr-11/)。

由此可见，所有符合 PSR-11 标准的依赖注入容器都可以在 Slim 中使用，开发者可以任意替换。在 Slim 以前的版本中，框架引入了 [pimple/pimple](https://github.com/silexphp/Pimple) 作为默认的依赖注入容器，现在的 4.x 版本直接去掉了，看来需要开发者自己选择依赖注入容器。

## Container 如何绑定到 handler

官方给出的依赖注入容器的使用示例代码如下：
```php
<?php
use DI\Container;
use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;
use Slim\Factory\AppFactory;

require __DIR__ . ‘/../vendor/autoload.php’;

// Create Container using PHP-DI
$container = new Container();

// Set container to create App with on AppFactory
AppFactory::setContainer($container);
$app = AppFactory::create();

$container->set(‘myService’, function () {
    $settings = […];
    return new MyService($settings);
});

$app->get(‘/foo’, function (Request $request, Response $response, $args) {
    $myService = $this->get(‘myService’);

    // …do something with $myService…

    return $response;
});
```

以上代码中，handler 是一个匿名函数，那么在匿名函数中的 $this 究竟绑定到哪个对象上呢？答案是直接绑定到「依赖注入容器对象」上。在 handler 中调用 `$this->get()` 等于调用 `$container->get()`。

将匿名函数绑定到对象上的地方发生在 Slim\CallableResolver
```php
private function bindToContainer(callable $callable): callable
{
    if (is_array($callable) && $callable[0] instanceof Closure) {
        $callable = $callable[0];
    }
    if ($this->container && $callable instanceof Closure) {
        $callable = $callable->bindTo($this->container);
    }
    return $callable;
}
```

具体绑定的时机是，等 HTTP 请求到来之后，RouteResolver 决定好使用哪个 route 后再进行绑定，而不是定义 handler 后马上进行绑定。如果构建 Slim\App 对象时，没有指定依赖注入容器，是无法在 handler 中使用 $this 变量的。

## 在自己定义的业务类里面使用 Container

还有一个重要问题是，自己定义的类如何使用依赖注入容器？例如想达到这样的效果：
```php
class Service
{
    public function doSomething()
    {
        // 希望在这里使用 di
    }
}

$app->get('/foo', function (Request $request, Response $response, $args) {
    $service = new Service();
    $service->doSomething();

    return $response;
});

$app->run();
```

由于 Slim 是没有提供相关的支持的，所以参考了一下 Phalcon 的方式，过程中需要自己新建一个 AppFactory，这样可以满足我的需求：
```php
<?php
require ‘vendor/autoload.php’;

use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;
use Slim\Factory\AppFactory;

use DI\Container;

$container = new Container();
$container->set(‘config’, function() {
    $setting = [‘db’ => ‘mysql’];
    return $setting;
});

class Injectable
{
    /**
     * @var \Psr\Container\ContainerInterface
     */
    protected static $container;

    public function get($name)
    {
        return static::$container->get($name);
    }

    public static function setContainer(\Psr\Container\ContainerInterface $container)
    {
        static::$container = $container;
    }
}

class MyAppFactory extends AppFactory
{
    public static function setContainer(\Psr\Container\ContainerInterface $container): void
    {
        parent::setContainer($container);
        Injectable::setContainer($container);
    }

}

MyAppFactory::setContainer($container);
$app = MyAppFactory::create();

class Service extends Injectable
{
    public function doSomething()
    {
        // 在自定义的类里面使用依赖注入容器
        echo $this->get(‘config’)[‘db’];
    }
}

$app->get(‘/foo’, function (Request $request, Response $response, $args) {
    $service = new Service();
    $service->doSomething();

    return $response;
});

$app->run();
```

我认为 Slim 不提供一个方式令开发者可以在自己定义的业务类里面使用依赖注入容器，是一个缺点，不过自己扩展一下也很简单。

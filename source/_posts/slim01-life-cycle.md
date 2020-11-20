---
title: Slim Framework 源码阅读（一）：生命周期基本流程
date: 2020-05-20 17:20:54
tags:
   - Slim
   - 源码阅读
   - Web开发
   - PHP
category:
   - coding
---

本文是 Slim Framework 源码阅读的第一篇，梳理 Slim 生命周期的基本流程。Slim 源码的版本是 4.5.0。

<!--more-->

官网文档描述 Slim 的生命周期是这样的：
- Instantiation
- Route Definitions
- Application Runner
    - Enter Middleware Stack
    - Run Application
    - Exit Middleware Stack
    - Send HTTP Response

看看官网给出来的最简单的示例：
```php
<?php
use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;
use Slim\Factory\AppFactory;

require __DIR__ . '/../vendor/autoload.php';

$app = AppFactory::create();

$app->get('/hello/{name}', function (Request $request, Response $response, array $args) {
    $name = $args['name'];
    $response->getBody()->write("Hello, $name");
    return $response;
});

$app->run();
```

从这个示例程序入手，可以看到 Slim 在一个 HTTP 请求生命周期里面的流程。

## 初始化核心对象 Slim\App

首先 Slim\Factory\AppFactory 会构造一个 Slim\App 对象，这个 App 对象就是整个 Slim 运作的核心，整个流程都是通过 App 对象来调度运作的。

AppFactory 构造 App 对象的方法如下：
```php
public static function create(
    ?ResponseFactoryInterface $responseFactory = null,
    ?ContainerInterface $container = null,
    ?CallableResolverInterface $callableResolver = null,
    ?RouteCollectorInterface $routeCollector = null,
    ?RouteResolverInterface $routeResolver = null,
    ?MiddlewareDispatcherInterface $middlewareDispatcher = null
): App {
    static::$responseFactory = $responseFactory ?? static::$responseFactory;
    return new App(
        self::determineResponseFactory(),
        $container ?? static::$container,
        $callableResolver ?? static::$callableResolver,
        $routeCollector ?? static::$routeCollector,
        $routeResolver ?? static::$routeResolver,
        $middlewareDispatcher ?? static::$middlewareDispatcher
    );
}
```

create 方法一共接收 6 个对象，这  6 个对象分别要实现不同的接口定义，分别是：
- ResponseFactoryInterface
- ContainerInterface
- CallableResolverInterface
- RouteCollectorInterface
- RouteResolverInterface
- MiddlewareDispatcherInterface

由此可见，这六个组件是构成 App 的核心，它们具体的功能后续再展开。当然，create 方法可以一个参数都不传，这样的话就使用框架提供的默认实现；开发者也可以自行实现这些组件，只要实现了相关接口就可以了，非常灵活。

## 定义 Route
示例程序的下一步是注册一个 route。

get 方法的定义：
```php
public function get(string $pattern, $callable): RouteInterface
{
    return $this->map(['GET'], $pattern, $callable);
}
```

具体怎么注册 route 先不展开看。在 Slim\Routing\RouteCollectorProxy 中除了 get 方法外，还定义了 post, put 等等的方法，这些跟 HTTP method 一一对应。Slim\App 对象是直接继承 RouteCollectorProxy 类的，从语义上来说，这个继承关系有点别扭？这样继承可能就是为了注册 route 的方便吧。再看回 get 方法，第三个参数是一个 callback，而这个 callback 就是 HTTP 请求的 handler（也就是有些框架称为的 action）。handler 接收三个参数，Request，Response，和一个 args 数组。见名知意，Request 就是解释 HTTP 请求报文后封装的对象，Response 就是后续用来构造 HTTP 响应报文的对象。

## Slim\App 开始运行
最后看看 run 方法的定义：
```php
/**
 * Run application
 *
 * This method traverses the application middleware stack and then sends the
 * resultant Response object to the HTTP client.
 *
 * @param ServerRequestInterface|null $request
 * @return void
 */
public function run(?ServerRequestInterface $request = null): void
{
    if (!$request) {
        $serverRequestCreator = ServerRequestCreatorFactory::create();
        $request = $serverRequestCreator->createServerRequestFromGlobals();
    }

    $response = $this->handle($request);
    $responseEmitter = new ResponseEmitter();
    $responseEmitter->emit($response);
}
```

三个流程：（1）解释 HTTP 报文，封装 Request 对象（2）进入 middleware stack，最后得到 Response 对象（3）ReponseEmitter 发送 Response 给客户端。

进入 handle 函数里面看一看：
```php
public function handle(ServerRequestInterface $request): ResponseInterface
{
    $response = $this->middlewareDispatcher->handle($request);

    $method = strtoupper($request->getMethod());
    if ($method === ‘HEAD’) {
        $emptyBody = $this->responseFactory->createResponse()->getBody();
        return $response->withBody($emptyBody);
    }

    return $response;
}
```

可以看到在整个请求的生命周期里面，Request 对象生成了之后，就在一层层的 middleware 里面流转。之前定义的 handler，也是 middleware 里面的其中一层。可以想象的是，开发者可以往 MiddlewareDispatcher 里面添加自己定义的 middleware，非常灵活。

----
以上粗略地看了一把生命周期里面的基本流程，整个宏观上的步骤是非常清晰的。可以感受到 Slim 是组件化的，所有参数声明都是接口，这就意味着每一个组件都可以由开发者自行替换。接下来后续篇章的就是看每个组件的实现细节了。

---
title: Slim Framework 源码阅读（五）：Routing
date: 2020-06-02 17:49:52
tags:
- PHP
- Web开发
- 源码阅读
- Slim
categories:
- coding
---

本文是 Slim Framework 源码阅读的第五篇，主要关于 Slim 路由部分的实现。Slim 源码的版本是 4.5.0。

<!--more-->

## 添加 route
前文已经提到过，定义 route 的方式是通过 Slim\App 继承 Slim\Routing\RouteCollectorProxy 的一系列方法，直接来看 RouteCollectorProxy 类中的 get 方法：
```php
public function get(string $pattern, $callable): RouteInterface
{
    return $this->map([‘GET’], $pattern, $callable);
}

public function map(array $methods, string $pattern, $callable): RouteInterface
{
    $pattern = $this->groupPattern . $pattern;

    return $this->routeCollector->map($methods, $pattern, $callable);
}
```

由此可见，实际创建 route 的是 RouteCollector 对象，Slim 提供的默认 collector 是 Slim\Routing\RouteCollector，里面的相关方法如下：
```php
public function map(array $methods, string $pattern, $handler): RouteInterface
{
    $route = $this->createRoute($methods, $pattern, $handler);
    $this->routes[$route->getIdentifier()] = $route;
    $this->routeCounter++;

    return $route;
}

protected function createRoute(array $methods, string $pattern, $callable): RouteInterface
{
    return new Route(
        $methods,
        $pattern,
        $callable,
        $this->responseFactory,
        $this->callableResolver,
        $this->container,
        $this->defaultInvocationStrategy,
        $this->routeGroups,
        $this->routeCounter
    );
}
```

先创建一个 route 对象，然后把 Route 对象加入 RouteCollector 对象的 routes 属性里面，routes 数组记录了所有的 route，可见 RouteCollector 的作用之一就是记录并维护所有的 route。Route 类的构造函数接收的参数不少，现在先不深入来看。下一步看看请求来到之后，路由是怎样找到对应的 route 的。

## 路由解析并执行对应 handler 的流程

前文已知，在中间件调用栈的最内层，是 Slim\Routing\RouteRunner 类，看看它的 handle 方法：
```php
/**
 * This request handler is instantiated automatically in App::__construct()
 * It is at the very tip of the middleware queue meaning it will be executed
 * last and it detects whether or not routing has been performed in the user
 * defined middleware stack. In the event that the user did not perform routing
 * it is done here
 *
 * @param ServerRequestInterface $request
 * @return ResponseInterface
 * @throws HttpNotFoundException
 * @throws HttpMethodNotAllowedException
 */
public function handle(ServerRequestInterface $request): ResponseInterface
{
    // If routing hasn't been done, then do it now so we can dispatch
    if ($request->getAttribute(RouteContext::ROUTING_RESULTS) === null) {
        $routingMiddleware = new RoutingMiddleware($this->routeResolver, $this->routeParser);
        $request = $routingMiddleware->performRouting($request);
    }

    if ($this->routeCollectorProxy !== null) {
        $request = $request->withAttribute(
            RouteContext::BASE_PATH,
            $this->routeCollectorProxy->getBasePath()
        );
    }

    /** @var Route $route */
    $route = $request->getAttribute(RouteContext::ROUTE);
    return $route->run($request);
}
```

走进 RoutingMiddleware 类的 performRouting 方法：
```php
public function performRouting(ServerRequestInterface $request): ServerRequestInterface
{
    $routingResults = $this->routeResolver->computeRoutingResults(
        $request->getUri()->getPath(),
        $request->getMethod()
    );
    $routeStatus = $routingResults->getRouteStatus();

    $request = $request->withAttribute(RouteContext::ROUTING_RESULTS, $routingResults);

    switch ($routeStatus) {
        case RoutingResults::FOUND:
            $routeArguments = $routingResults->getRouteArguments();
            $routeIdentifier = $routingResults->getRouteIdentifier() ?? '';
            $route = $this->routeResolver
                ->resolveRoute($routeIdentifier)
                ->prepare($routeArguments);
            return $request->withAttribute(RouteContext::ROUTE, $route);

        case RoutingResults::NOT_FOUND:
            throw new HttpNotFoundException($request);

        case RoutingResults::METHOD_NOT_ALLOWED:
            $exception = new HttpMethodNotAllowedException($request);
            $exception->setAllowedMethods($routingResults->getAllowedMethods());
            throw $exception;

        default:
            throw new RuntimeException('An unexpected error occurred while performing routing.’);
    }
}
```

这里又引出了 routeResolver 对象，回溯到 Slim\App 的构造函数，可以看到默认的 routeResolver 是 Slim\Routing\RouteResolver。在 RouteResolver 里面，会发现是通过 Slim\Routing\Dispatcher 来计算路由结果。看一下 Dispatcher 类里面创建 dispatcher 的方法：
```php
protected function createDispatcher(): FastRouteDispatcher
{
    if ($this->dispatcher) {
        return $this->dispatcher;
    }

    $routeDefinitionCallback = function (FastRouteCollector $r) {
        $basePath = $this->routeCollector->getBasePath();

        foreach ($this->routeCollector->getRoutes() as $route) {
            $r->addRoute($route->getMethods(), $basePath . $route->getPattern(), $route->getIdentifier());
        }
    };

    $cacheFile = $this->routeCollector->getCacheFile();
    if ($cacheFile) {
        /** @var FastRouteDispatcher $dispatcher */
        $dispatcher = \FastRoute\cachedDispatcher($routeDefinitionCallback, [
            'dispatcher' => FastRouteDispatcher::class,
            'routeParser' => new Std(),
            'cacheFile' => $cacheFile,
        ]);
    } else {
        /** @var FastRouteDispatcher $dispatcher */
        $dispatcher = \FastRoute\simpleDispatcher($routeDefinitionCallback, [
            'dispatcher' => FastRouteDispatcher::class,
            'routeParser' => new Std(),
        ]);
    }

    $this->dispatcher = $dispatcher;
    return $this->dispatcher;
}
```

由此可见，Slim 默认的路由解析是与 FastRoute 深度绑定的。FastRoute 是一个基于正则表达式实现的 router，号称速度非常快，详情见：[https://github.com/nikic/FastRoute](https://github.com/nikic/FastRoute)。如果开发者不想使用 FastRoute 的话，就要使用自己实现的 dispatcher 类，只要这个类实现 Slim\Interfaces\DispatcherInterface 接口即可。

从上面这段代码可以看出，路由解析的结果可以被文件缓存起来。要使用文件缓存，只需调用上文提到的 RouteCollector 里面的 setCacheFile 方法来设置即可。

当请求来到的时候，外层的中间件层都执行完毕后，会来到最内层的 Slim\Routing\RouteRunner 类，如果发现到达该层时，还没完成路由解析操作，那么此时就进行路由解析。它调用 Slim\Routing\RoutingMiddleware 的 performRouting 方法，在该方法内接着调用 Slim\ Routing\RouteResolver 的 computeRoutingResults 获取路由解析的最终结果。RouteResolver 则是借助  Slim\Routing\Dispatcher 中调用真实的 routing 组件来进行路由解析，Slim 默认使用的是 FastRoute。最终获取路由解析的结果出来之后，把结果记录在 Request 对象里面。就是上文提到的 Slim\Routing\RouteRunner 类中 handle 方法的最后两行：
```php
public function handle(ServerRequestInterface $request): ResponseInterface
{
    // 以上内容省略，运行到这里的时候路由解析结果已经存在于 request 对象中，route 对象就是本次请求对应的 route
    $route = $request->getAttribute(RouteContext::ROUTE);
    return $route->run($request);
}
```

接下来看 Route 对象的 run 方法：
```php
public function run(ServerRequestInterface $request): ResponseInterface
{
    if (!$this->groupMiddlewareAppended) {
        $this->appendGroupMiddlewareToRoute();
    }

    return $this->middlewareDispatcher->handle($request);
}
```

奇怪的是，在 Route 内部又维护了一个 MiddlewareDispatcher？在 Route 的构造函数里面有这样一行：
```php
public function __construct(
    array $methods,
    string $pattern,
    $callable,
    ResponseFactoryInterface $responseFactory,
    CallableResolverInterface $callableResolver,
    ?ContainerInterface $container = null,
    ?InvocationStrategyInterface $invocationStrategy = null,
    array $groups = [],
    int $identifier = 0
) {
	  // 该方法内的其余行省略
    $this->middlewareDispatcher = new MiddlewareDispatcher($this, $callableResolver, $container);
}
```

可见这个 MiddlewareDispatcher 不是从外部传进来的，而是由 route 对象自行创建的，这里把 $this 传给 MiddlewareDispatcher 的构造函数，根据以前章节的经验可知，在这个中间件栈里面，最内层的就是 route 自己。至于为什么在 route 对象里面还要维护一个中间件栈，暂时也不十分清楚，可能是为了给特定的一组 url 的请求额外执行一些中间件调用吧，毕竟 Slim\App 里面的中间件是所有请求都要通过执行的。

那 MiddlewareDispatcher 最内层执行的就是 route 对象的 handle 方法：
```php
public function handle(ServerRequestInterface $request): ResponseInterface
{
    if ($this->callableResolver instanceof AdvancedCallableResolverInterface) {
        $callable = $this->callableResolver->resolveRoute($this->callable);
    } else {
        $callable = $this->callableResolver->resolve($this->callable);
    }
    $strategy = $this->invocationStrategy;

    if (
        is_array($callable)
        && $callable[0] instanceof RequestHandlerInterface
        && !in_array(RequestHandlerInvocationStrategyInterface::class, class_implements($strategy))
    ) {
        $strategy = new RequestHandler();
    }

    $response = $this->responseFactory->createResponse();
    return $strategy($callable, $request, $response, $this->arguments);
}
```

这里出现了 strategy 的概念，看官网文档的说明：
> The route callback signature is determined by a route strategy. By default, Slim expects route callbacks to accept the request, response, and an array of route placeholder arguments. This is called the RequestResponse strategy. However, you can change the expected route callback signature by simply using a different strategy. As an example, Slim provides an alternative strategy called RequestResponseArgs that accepts request and response, plus each route placeholder as a separate argument.

Slim 居然连 handler 接收的参数都可以替换 Orz。默认的策略类是 `Slim\Handlers\Strategies\RequestReponse`：
```php
class RequestResponse implements InvocationStrategyInterface
{
    /**
     * Invoke a route callable with request, response, and all route parameters
     * as an array of arguments.
     *
     * @param callable               $callable
     * @param ServerRequestInterface $request
     * @param ResponseInterface      $response
     * @param array<mixed>           $routeArguments
     *
     * @return ResponseInterface
     */
    public function __invoke(
        callable $callable,
        ServerRequestInterface $request,
        ResponseInterface $response,
        array $routeArguments
    ): ResponseInterface {
        foreach ($routeArguments as $k => $v) {
            $request = $request->withAttribute($k, $v);
        }

        return $callable($request, $response, $routeArguments);
    }
}
```

一路下来走到这里，终于能进到 handler，整个路由解析的过程就看完了。

## Routing 的其余功能
再看一些 routing 的功能，加深一下对 routing 的了解。

### 重定向助手

官网文档提供的使用示例：
```php
app->redirect('/books', '/library', 301);
```

redirect 方法的定义位于 Slim\Routing\RouteCollectorProxy 类中：
```php
public function redirect(string $from, $to, int $status = 302): RouteInterface
{
    $responseFactory = $this->responseFactory;

    $handler = function () use ($to, $status, $responseFactory) {
        $response = $responseFactory->createResponse($status);
        return $response->withHeader('Location', (string) $to);
    };

    return $this->get($from, $handler);
}
```

ResponseFactory 等后续篇章说到 Response 对象的时候再说。这个 redirect 方法里面直接创建了一个 handler，在 handler 里面生成 Response 对象，设置了 Response 对象的 header（[具体原理可见 HTTP 协议的重定向机制](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Redirections))，然后按上文提到的方法注册路由。

### 路由分组

官网文档提供的示例：
```php
$app->group('/users/{id:[0-9]+}', function (RouteCollectorProxy $group) {
    $group->map(['GET', 'DELETE', 'PATCH', 'PUT'], '', function ($request, $response, $args) {
        // Find, delete, patch or replace user identified by $args['id']
    })->setName('user');

    $group->get('/reset-password', function ($request, $response, $args) {
        // Route for /users/{id:[0-9]+}/reset-password
        // Reset the password for user identified by $args['id']
    })->setName('user-password-reset');
});
```

这里的 group 方法定义在 Slim\Routing\RouteCollectorProxy 类中：
```php
public function group(string $pattern, $callable): RouteGroupInterface
{
    $pattern = $this->groupPattern . $pattern;

    return $this->routeCollector->group($pattern, $callable);
}
```

当调用 $app 对象的 group 方法时，`$this->groupPattern` 的值是空字符串。接着调用 Slim\Routing\RouteCollector 类的 group 方法：
```php
public function group(string $pattern, $callable): RouteGroupInterface
{
    $routeCollectorProxy = new RouteCollectorProxy(
        $this->responseFactory,
        $this->callableResolver,
        $this->container,
        $this,
        $pattern
    );

    $routeGroup = new RouteGroup($pattern, $callable, $this->callableResolver, $routeCollectorProxy);
    $this->routeGroups[] = $routeGroup;

    $routeGroup->collectRoutes();
    array_pop($this->routeGroups);

    return $routeGroup;
}
```

在这个方法里面又重新新建了一个 RouteCollectorProxy 对象（姑且取名为 Proxy B），从上文已经知道 RouteCollectorProxy 对象的作用是为了往 RouteCollector 里面添加 route 的，创建这个 RouteCollectorProxy 时把 RouteCollector 对象自身传递了进去，达到的效果就是新建的 RouteCollectorProxy 还是调用同一个 RouteCollector 添加 route（全局只有一个  RouteCollector）。

再来看 Slim\Routing\RouteGroup 类的 collectRoutes 方法：
```php
public function collectRoutes(): RouteGroupInterface
{
    if ($this->callableResolver instanceof AdvancedCallableResolverInterface) {
        $callable = $this->callableResolver->resolveRoute($this->callable);
    } else {
        $callable = $this->callableResolver->resolve($this->callable);
    }
    $callable($this->routeCollectorProxy);
    return $this;
}
```

这里的 `$callable` 对应就是前文的匿名函数：
```php
function (RouteCollectorProxy $group) {
    $group->map(['GET', 'DELETE', 'PATCH', 'PUT'], '', function ($request, $response, $args) {
        // Find, delete, patch or replace user identified by $args['id']
    })->setName('user');

    $group->get('/reset-password', function ($request, $response, $args) {
        // Route for /users/{id:[0-9]+}/reset-password
        // Reset the password for user identified by $args['id']
    })->setName('user-password-reset');
}
```

这个匿名函数接收的 RouteCollectorProxy 对象就是前文最新创建的 Proxy B，通过这个 Proxy B 调用全局唯一的 RouteCollector 对象新加入一个 route，注意 Proxy B 的 `groupPattern` 属性的值不再是空字符串而是 `/users/{id:[0-9]+}`，从而达到路由分组的目的，至于加入 RouteCollector 的操作跟上文的「没有分组的路由」是一样的，对于 RouteCollector 来说都是一样的 route 对象。

### 将 route 绑定到 controller 类

别的框架的惯用做法是开发者创建 Controller 类，然后配置路由解析规则，HTTP 请求到来的时候，由框架调用对应的 Controller 类里面的 handler 方法。Slim 也支持这样搞，看官网的示例：
```php
<?php
use Psr\Container\ContainerInterface;

class HomeAction
{
   protected $container;

   public function __construct(ContainerInterface $container) {
       $this->container = $container;
   }

   public function __invoke($request, $response, $args) {
        // your code
        // to access items in the container... $this->container->get('');
        return $response;
   }
}

$app->get('/', \HomeAction::class);
```

与前文不同的是，这次调用 RouteCollectorProxy 的 get 方法时传的不是回调函数了，而是直接传了字符串（Controller 的类名）。

前文提到，在 Slim\Routing\Route 类中的 handle 方法中会调用 `$this->callableResolver->resolveRoute($this->callable)` 函数，这里的 callable 是一个字符串。相关的代码如下：
```php
public function resolveRoute($toResolve): callable
{
    return $this->resolveByPredicate($toResolve, [$this, 'isRoute'], 'handle');
}

private function resolveByPredicate($toResolve, callable $predicate, string $defaultMethod): callable
{
    if (is_callable($toResolve)) {
        return $this->bindToContainer($toResolve);
    }
    $resolved = $toResolve;
    if ($predicate($toResolve)) {
        $resolved = [$toResolve, $defaultMethod];
    }
    if (is_string($toResolve)) {
        [$instance, $method] = $this->resolveSlimNotation($toResolve);
        if ($method === null && $predicate($instance)) {
            $method = $defaultMethod;
        }
        $resolved = [$instance, $method ?? '__invoke'];
    }
    $callable = $this->assertCallable($resolved, $toResolve);
    return $this->bindToContainer($callable);
}

private function resolveSlimNotation(string $toResolve): array
{
    preg_match(CallableResolver::$callablePattern, $toResolve, $matches);
    [$class, $method] = $matches ? [$matches[1], $matches[2]] : [$toResolve, null];

    if ($this->container && $this->container->has($class)) {
        $instance = $this->container->get($class);
    } else {
        if (!class_exists($class)) {
            throw new RuntimeException(sprintf('Callable %s does not exist', $class));
        }
        $instance = new $class($this->container);
    }
    return [$instance, $method];
}
```

以上的代码就会将我们的类名字符串转换成一个 callable。Slim 框架没有提供一个 Controller 基类，开发者可以自己实现一个 Controller 基类，以避免每个 Controller 里面都要实现一个 `__invoke` 方法。

---

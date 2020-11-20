---
title: Slim Framework 源码阅读（四）：Middleware
date: 2020-05-26 10:04:37
tags:
- Slim
- 源码阅读
- Web开发
- PHP
- PSR
- 依赖注入
categories:
- coding
---


本文是 Slim Framework 源码阅读的第四篇，主要关于 Slim 中间件部分的实现。Slim 源码的版本是 4.5.0。

<!--more-->

如 [PSR-15: HTTP Server Request Handlers](https://www.php-fig.org/psr/psr-15/) 所述，中间件（middleware）的作用是：
> HTTP request handlers are a fundamental part of any web application. Server-side code receives a request message, processes it, and produces a response message. HTTP middleware is a way to move common request and response processing away from the application layer.

中间件是 web 框架中一个很重要的部分，将一些对 Request 和 Response 通用化的处理操作从业务逻辑中分离出来。PSR-15 规范了中间件应该提供的接口和相关功能。

每个 web 框架对中间件的具体实现方式是不一样的，Slim Framwork 的实现方式如下：

>When you run the Slim application, the Request object traverses the middleware structure from the outside in. They first enter the outer-most middleware, then the next outer-most middleware, (and so on), until they ultimately arrive at the Slim application itself. After the Slim application dispatches the appropriate route, the resultant Response object exits the Slim application and traverses the middleware structure from the inside out. Ultimately, a final Response object exits the outer-most middleware, is serialized into a raw HTTP response, and is returned to the HTTP client. Here’s a diagram that illustrates the middleware process flow:

![](https://www.slimframework.com/docs/v4/images/middleware.png)

可以同时配置多个中间件，多个中间件之间的关系可以想象成一个同心圆的结构。HTTP 请求进来的时候，最外层（也就是最后添加）的中间件最先触发，依次往内；handler 生成了 Response 后，最里层（也就是最先添加）的中间件最先触发，依次往外传递。

## 中间件调用栈的构造

中间件通过 MiddlewareDispatcher 进行管理，Slim 默认提供的 dispatcher 是 Slim\MiddlewareDispatcher，当然开发者可以自行替换，只要其实现 Slim\Interfaces\MiddlewareDispatcherInterface 接口即可。

### 添加一个 Middleware 对象

首先来看往 Dispatcher 中添加一个中间件的方法，这个方法属于 Slim\MiddlewareDispatcher 类：
```php
public function addMiddleware(MiddlewareInterface $middleware): MiddlewareDispatcherInterface
{
    $next = $this->tip;
    $this->tip = new class ($middleware, $next) implements RequestHandlerInterface {
        /**
         * @var MiddlewareInterface
         */
        private $middleware;

        /**
         * @var RequestHandlerInterface
         */
        private $next;

        public function __construct(MiddlewareInterface $middleware, RequestHandlerInterface $next)
        {
            $this->middleware = $middleware;
            $this->next = $next;
        }

        public function handle(ServerRequestInterface $request): ResponseInterface
        {
            return $this->middleware->process($request, $this->next);
        }
    };

    return $this;
}
```

Dispatcher 有点点像一个栈，最后添加的中间件最先运行，MiddlewareDispatcher 的 $tip 属性就是指向最外层的 middleware。在这个方法里面 Slim 的操作有点 trikcy，直接 new 了一个匿名类，声明了这个类实现了 Psr\Http\Server\RequestHandlerInterface 接口，这个接口同样是 PSR-15 里面规范的内容。再确切点来说，是 MiddlewareDispatcher 维护着一层一层的 RequestHandler，而每一层的 RequestHandler 中通过 Middleware 对象进行具体的处理操作。

### 添加一个 callback

除了往 Dispatcher 中添加实现 Psr\Http\Server\MiddlewareInterface 接口的 Middleware 类外，还有一种方法是添加一个 callback。在 Slim\MiddlewareDispatcher 类中有如下方法：
```php
public function addCallable(callable $middleware): self
{
    $next = $this->tip;

    if ($this->container && $middleware instanceof Closure) {
        $middleware = $middleware->bindTo($this->container);
    }

    $this->tip = new class ($middleware, $next) implements RequestHandlerInterface {
        /**
         * @var callable
         */
        private $middleware;

        /**
         * @var RequestHandlerInterface
         */
        private $next;

        public function __construct(callable $middleware, RequestHandlerInterface $next)
        {
            $this->middleware = $middleware;
            $this->next = $next;
        }

        public function handle(ServerRequestInterface $request): ResponseInterface
        {
            return ($this->middleware)($request, $this->next);
        }
    };

    return $this;
}
```

创建匿名对象类的部分跟上文大同小异，不一样的地方是在 addCallable 中，会将这个 callback 绑定到依赖注入容器中。MiddlewareDispatcher 的构造函数是会接收一个依赖注入容器对象的。

### 用类名字符串来添加

可以以「类名字符串」的方式，把中间件添加到栈中。在 Slim\MiddlewareDispatcher 类中有如下方法：
```php
public function addDeferred(string $middleware): self
{
    $next = $this->tip;
    $this->tip = new class (
        $middleware,
        $next,
        $this->container,
        $this->callableResolver
    ) implements RequestHandlerInterface {
        /**
         * @var string
         */
        private $middleware;

        /**
         * @var RequestHandlerInterface
         */
        private $next;

        /**
         * @var ContainerInterface|null
         */
        private $container;

        /**
         * @var CallableResolverInterface|null
         */
        private $callableResolver;

        public function __construct(
            string $middleware,
            RequestHandlerInterface $next,
            ?ContainerInterface $container = null,
            ?CallableResolverInterface $callableResolver = null
        ) {
            $this->middleware = $middleware;
            $this->next = $next;
            $this->container = $container;
            $this->callableResolver = $callableResolver;
        }

        public function handle(ServerRequestInterface $request): ResponseInterface
        {
            if ($this->callableResolver instanceof AdvancedCallableResolverInterface) {
                $callable = $this->callableResolver->resolveMiddleware($this->middleware);
                return $callable($request, $this->next);
            }

            $callable = null;

            if ($this->callableResolver instanceof CallableResolverInterface) {
                try {
                    $callable = $this->callableResolver->resolve($this->middleware);
                } catch (RuntimeException $e) {
                    // Do Nothing
                }
            }

            if (!$callable) {
                $resolved = $this->middleware;
                $instance = null;
                $method = null;

                // Check for Slim callable as `class:method`
                if (preg_match(CallableResolver::$callablePattern, $resolved, $matches)) {
                    $resolved = $matches[1];
                    $method = $matches[2];
                }

                if ($this->container && $this->container->has($resolved)) {
                    $instance = $this->container->get($resolved);
                    if ($instance instanceof MiddlewareInterface) {
                        return $instance->process($request, $this->next);
                    }
                } elseif (!function_exists($resolved)) {
                    if (!class_exists($resolved)) {
                        throw new RuntimeException(sprintf(‘Middleware %s does not exist’, $resolved));
                    }
                    $instance = new $resolved($this->container);
                }

                if ($instance && $instance instanceof MiddlewareInterface) {
                    return $instance->process($request, $this->next);
                }

                $callable = $instance ?? $resolved;
                if ($instance && $method) {
                    $callable = [$instance, $method];
                }

                if ($this->container && $callable instanceof Closure) {
                    $callable = $callable->bindTo($this->container);
                }
            }

            if (!is_callable($callable)) {
                throw new RuntimeException(
                    sprintf(
                        ‘Middleware %s is not resolvable’,
                        $this->middleware
                    )
                );
            }

            return $callable($request, $this->next);
        }
    };

    return $this;
}
```

我认为与「添加对象」的方式相比，「添加类名」的方式的好处在于：（1）会延迟到中间件触发的时候再对 middleware 进行更多的构造；（2）开发者引入一个符合 Psr\Http\Server\MiddlewareInterface 的类，然后由 Slim 提供的 CallableResolver 来对这个类进行更多的绑定操作，以适应 Slim 的运行机制。例如默认情况下，Slim 会使用 Slim\CallableResolver 类来进行解析封装操作，看 Slim\CallableResolver 类里面的 resolveMiddleware 方法以及其余的方法：
```php
public function resolveMiddleware($toResolve): callable
{
    return $this->resolveByPredicate($toResolve, [$this, ‘isMiddleware’], ‘process’);
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
        $resolved = [$instance, $method ?? ‘__invoke’];
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
            throw new RuntimeException(sprintf(‘Callable %s does not exist’, $class));
        }
        $instance = new $class($this->container);
    }
    return [$instance, $method];
}
```

其实这里也没什么特别的操作，就是返回了一个 callback，这个 callback 绑定了依赖注入容器。当然如果使用自己的 CallableResolver 的话，是可以更灵活地对 Middleware 进行更多的配置操作的。

### 中间件调用栈的最内层

默认情况下，中间件调用栈的最内层是 Slim\Routing\RouteRunner 类，也就是进行路由操作了。因为 Slim\App 的构造函数是这样的：
```php
public function __construct(
    ResponseFactoryInterface $responseFactory,
    ?ContainerInterface $container = null,
    ?CallableResolverInterface $callableResolver = null,
    ?RouteCollectorInterface $routeCollector = null,
    ?RouteResolverInterface $routeResolver = null,
    ?MiddlewareDispatcherInterface $middlewareDispatcher = null
) {
    parent::__construct(
        $responseFactory,
        $callableResolver ?? new CallableResolver($container),
        $container,
        $routeCollector
    );

    $this->routeResolver = $routeResolver ?? new RouteResolver($this->routeCollector);
    $routeRunner = new RouteRunner($this->routeResolver, $this->routeCollector->getRouteParser(), $this);

    if (!$middlewareDispatcher) {
        $middlewareDispatcher = new MiddlewareDispatcher($routeRunner, $this->callableResolver, $container);
    } else {
        $middlewareDispatcher->seedMiddlewareStack($routeRunner);
    }

    $this->middlewareDispatcher = $middlewareDispatcher;
}
```

## 中间件的运行
看 Slim\MiddlewareDispatcher 类的运行入口：
```php
public function handle(ServerRequestInterface $request): ResponseInterface
{
    return $this->tip->handle($request);
}
```

如上文所述，`$this->tip` 指向的是最外层的中间件，然后递归地 handle 下去（调用下一层）。例如 Slim 本身提供的 Slim\Middleware\ContentLengthMiddleware 类的 process 方法：
```php
namespace Slim\Middleware;

class ContentLengthMiddleware implements MiddlewareInterface
{
    public function process(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface
    {
        $response = $handler->handle($request);

        // Add Content-Length header if not already added
        $size = $response->getBody()->getSize();
        if ($size !== null && !$response->hasHeader(‘Content-Length’)) {
            $response = $response->withHeader(‘Content-Length’, (string) $size);
        }

        return $response;
    }
}
```

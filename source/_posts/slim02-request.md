---
title: Slim Framework 源码阅读（二）：Request 对象
date: 2020-05-21 16:52:13
tags:
- Slim
- 源码阅读
- Web开发
- PHP
- PSR
categories:
- coding
---

本文是 Slim Framework 源码阅读的第二篇，主要关于 Slim 如何生成封装 HTTP 请求的 Request 对象。Slim 源码的版本是 4.5.0。

<!--more-->

Request 对象是对 HTTP 请求进行解释后，封装成一个包含 HTTP 请求相关信息的对象。有两个非常重要的 PSR 文档：
- [PSR-7: HTTP message interfaces](https://www.php-fig.org/psr/psr-7/)
- [PSR-17: HTTP Factories](https://www.php-fig.org/psr/psr-17/)

这两个文档分别规范了 HTTP message 应该提供的接口，和生成 HTTP message 的 factory 应该提供的接口，而 Request 对象则是 HTTP message 的其中一种类型。在 Slim 中使用的 Request 对象必须是实现 PSR-7 中的 ServerRequestInterface 接口的。

SlimFramework 默认支持的创建 Request 对象的 factory 分别有：
- Slim\Psr7\Factory\ServerRequestFactory
- Nyholm\Psr7Server\ServerRequestCreator
- Laminas\Diactoros\ServerRequestFactory
- Zend\Diactoros\ServerRequestFactory
- GuzzleHttp\Psr7\ServerRequest

那 Slim\App 到底会使用哪个 factory 呢？在 Slim\Factory\ServerRequestCreatorFactory 里面有相应的代码：
```php
public static function determineServerRequestCreator(): ServerRequestCreatorInterface
{
    if (static::$serverRequestCreator) {
        return static::attemptServerRequestCreatorDecoration(static::$serverRequestCreator);
    }

    $psr17FactoryProvider = static::$psr17FactoryProvider ?? new Psr17FactoryProvider();

    /** @var Psr17Factory $psr17Factory */
    foreach ($psr17FactoryProvider->getFactories() as $psr17Factory) {
        if ($psr17Factory::isServerRequestCreatorAvailable()) {
            $serverRequestCreator = $psr17Factory::getServerRequestCreator();
            return static::attemptServerRequestCreatorDecoration($serverRequestCreator);
        }
    }

    throw new RuntimeException(
        "Could not detect any ServerRequest creator implementations. " .
        "Please install a supported implementation in order to use `App::run()` " .
        "without having to pass in a `ServerRequest` object. " .
        "See https://github.com/slimphp/Slim/blob/4.x/README.md for a list of supported implementations."
    );
}
```


如果开发者不配置 `static::$psr17FactoryProvider` 的话，就是使用默认的 `Slim\Factory\Psr17\Psr17FactoryProvider`。它的代码如下：

```php
class Psr17FactoryProvider implements Psr17FactoryProviderInterface
{
    /**
     * @var string[]
     */
    protected static $factories = [
        SlimPsr17Factory::class,
        NyholmPsr17Factory::class,
        LaminasDiactorosPsr17Factory::class,
        ZendDiactorosPsr17Factory::class,
        GuzzlePsr17Factory::class,
    ];

    /**
     * {@inheritdoc}
     */
    public static function getFactories(): array
    {
        return static::$factories;
    }

    /**
     * {@inheritdoc}
     */
    public static function setFactories(array $factories): void
    {
        static::$factories = $factories;
    }

    /**
     * {@inheritdoc}
     */
    public static function addFactory(string $factory): void
    {
        array_unshift(static::$factories, $factory);
    }
}
```

在 Psr17FactoryProvider::factories 就可以看出究竟 Slim 默认支持哪些上文提到的 factory 了。而这些 factory 是不包含在 Slim 的核心代码里面的，开发者自己想使用哪一个，就要在自己项目的 composer.json 文件里面指定。这里再一次体现出 Slim 的组件化思想。

但是，为什么 Slim 不包含这些 factory 的代码，还要在命名空间 Slim\Factory\Psr17 下面定义了几个 factory 类（例如 GuzzlePsr17Factory、LaminasDiactorosPsr17Factory）呢？是为了定义好具体 factory 的相关信息，例如
```php
namespace Slim\Factory\Psr17;

class SlimPsr17Factory extends Psr17Factory
{
    protected static $responseFactoryClass = ‘Slim\Psr7\Factory\ResponseFactory’;
    protected static $streamFactoryClass = ‘Slim\Psr7\Factory\StreamFactory’;
    protected static $serverRequestCreatorClass = ‘Slim\Psr7\Factory\ServerRequestFactory’;
    protected static $serverRequestCreatorMethod = ‘createFromGlobals’;
}
```

而这个 SlimPsr17Factory 中提到的几个类，则需要开发者自行通过 composer 引入。在 Slim\Factory\Psr17\Psr17Factory 类中有一个 isServerRequestCreatorAvailable 方法，用于判断具体的 factory 类有没有被开发者引入：
```php
public static function isServerRequestCreatorAvailable(): bool
{
    return (
        static::$serverRequestCreatorClass
        && static::$serverRequestCreatorMethod
        && class_exists(static::$serverRequestCreatorClass)
    );
}
```

接下来看具体生成 Request 对象的函数。具体使用了哪个 factory，就要看对应的 $serverRequestCreatorClass 的 $serverRequestCreatorMethod 方法。以 SlimPsr17Factory 为例，在 Slim\Psr7\Factory\ServerRequestFactory 类中有一个 createFromGlobals 方法：
```php
public static function createFromGlobals(): Request
{
    $method = isset($_SERVER[‘REQUEST_METHOD’]) ? $_SERVER[‘REQUEST_METHOD’] : ‘GET’;
    $uri = (new UriFactory())->createFromGlobals($_SERVER);

    $headers = Headers::createFromGlobals();
    $cookies = Cookies::parseHeader($headers->getHeader(‘Cookie’, []));

    // Cache the php://input stream as it cannot be re-read
    $cacheResource = fopen(‘php://temp’, ‘wb+’);
    $cache = $cacheResource ? new Stream($cacheResource) : null;

    $body = (new StreamFactory())->createStreamFromFile(‘php://input’, ‘r’, $cache);
    $uploadedFiles = UploadedFile::createFromGlobals($_SERVER);

    $request = new Request($method, $uri, $headers, $cookies, $_SERVER, $body, $uploadedFiles);
    $contentTypes = $request->getHeader('Content-Type') ?? [];

    $parsedContentType = '';
    foreach ($contentTypes as $contentType) {
        $fragments = explode(';', $contentType);
        $parsedContentType = current($fragments);
    }

    $contentTypesWithParsedBodies = ['application/x-www-form-urlencoded', 'multipart/form-data'];
    if ($method === 'POST' && in_array($parsedContentType, $contentTypesWithParsedBodies)) {
        return $request->withParsedBody($_POST);
    }

    return $request;
}
```

为了最终调用到这个 createFromGlobals 方法，Slim 经过了层层的包装，看 Slim\App 的 run 方法，这里是生成 Request 对象的调用处：
```php
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

将真实的 HTTP factory 封装进 Slim\Factory\Psr17\ServerRequestCreator 里面，由 ServerRequestCreator 来调用具体的 factory；而 ServerRequestCreatorFactory 则用于判断开发者自己选择使用了哪个具体的 Factory。

Slim\Factory\Psr17\ServerRequestCreator 类的 createServerRequestFromGlobals 方法也值得一提：
```php
public function createServerRequestFromGlobals(): ServerRequestInterface
{
    /** @var callable $callable */
    $callable = [$this->serverRequestCreator, $this->serverRequestCreatorMethod];
    return (Closure::fromCallable($callable))();
}
```

这里体现了如同通过类名和方法名来调用一个函数。

----

以上内容主要关于 Slim 中生成 Request 对象的代码，一整套下来是一个很好的示例：当实现同一个功能，可以通过不同的库来实现的时候，如何做到灵活地替换。

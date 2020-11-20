---
title: Slim Framework 源码阅读（六）：Response
date: 2020-06-09 14:54:14
tags:
- PHP
- Web开发
- 源码阅读
- Slim
categories:
- coding
---

本文是 Slim Framework 源码阅读的第六篇，主要关于 Slim HTTP Response 部分的实现。Slim 源码的版本是 4.5.0。

<!--more-->

## 创建 Response 对象

[前面关于 Request 对象的文章](http://blog.leejianyang.com/2020/05/21/slim02-request/)已经提到过 PSR-7 和 PSR-17 这两个规范文档，生成 Response 对象的相关规范也在这两篇文档中。

与生成 Request 对象相类似，生成 Response 前也要先决定使用哪个 Factory 类去创建对象。在 Slim\Factory\AppFactory 类中有一个 determineResponseFactory 方法：
```php
public static function determineResponseFactory(): ResponseFactoryInterface
{
    if (static::$responseFactory) {
        if (static::$streamFactory) {
            return static::attemptResponseFactoryDecoration(static::$responseFactory, static::$streamFactory);
        }
        return static::$responseFactory;
    }

    $psr17FactoryProvider = static::$psr17FactoryProvider ?? new Psr17FactoryProvider();

    /** @var Psr17Factory $psr17factory */
    foreach ($psr17FactoryProvider->getFactories() as $psr17factory) {
        if ($psr17factory::isResponseFactoryAvailable()) {
            $responseFactory = $psr17factory::getResponseFactory();

            if (static::$streamFactory || $psr17factory::isStreamFactoryAvailable()) {
                $streamFactory = static::$streamFactory ?? $psr17factory::getStreamFactory();
                return static::attemptResponseFactoryDecoration($responseFactory, $streamFactory);
            }

            return $responseFactory;
        }
    }

    throw new RuntimeException(
        "Could not detect any PSR-17 ResponseFactory implementations. " .
        "Please install a supported implementation in order to use `AppFactory::create()`. " .
        "See https://github.com/slimphp/Slim/blob/4.x/README.md for a list of supported implementations."
    );
}
```

这个方法里面的逻辑跟前文提到的创建 Request 对象的逻辑大同小异，Slim 框架默认使用的 Factory 是 `Slim\Psr7\Factory\ResponseFactory`。

前文提到，路由解析完毕决定好使用哪个 Route 对象后，会在 Route 对象的 handler 中创建 Response 对象。看 Slim\Routing\Route 类中的 handle 方法：
```php
public function handle(ServerRequestInterface $request): ResponseInterface
{
    // 开头的无关代码省略

    $response = $this->responseFactory->createResponse();
    return $strategy($callable, $request, $response, $this->arguments);
}
```

createResponse 方法的具体定义就在各个库里面了，超出了 Slim 框架的范围，现在先不看了。这样下来，Slim 框架就创建好了 Response 对象，开发者可以在 handler 里面使用这个 Response 对象。

## 返回 HTTP 响应报文到客户端
开发者在 handler 里面设置好 Response 对象的各个值后，Slim 框架会根据这个 Response 对象的值组装 HTTP 响应报文返回给客户端。看 Slim\App 类中的相关方法：
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

public function handle(ServerRequestInterface $request): ResponseInterface
{
    $response = $this->middlewareDispatcher->handle($request);

    /**
     * This is to be in compliance with RFC 2616, Section 9.
     * If the incoming request method is HEAD, we need to ensure that the response body
     * is empty as the request may fall back on a GET route handler due to FastRoute's
     * routing logic which could potentially append content to the response body
     * https://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html#sec9.4
     */
    $method = strtoupper($request->getMethod());
    if ($method === 'HEAD') {
        $emptyBody = $this->responseFactory->createResponse()->getBody();
        return $response->withBody($emptyBody);
    }

    return $response;
}
```

特别之处是，如果 HTTP 请求报文中的方法为 HEAD，那么 Slim 框架帮我们确保返回的 HTTP 报文只包含 header 部分以符合 HTTP 标准的规范。再来看 Slim\ResponseEmitter 类：
```php
public function emit(ResponseInterface $response): void
{
    $isEmpty = $this->isResponseEmpty($response);
    if (headers_sent() === false) {
        $this->emitStatusLine($response);
        $this->emitHeaders($response);
    }

    if (!$isEmpty) {
        $this->emitBody($response);
    }
}
```

逐个逐个部分来组装 HTTP 响应报文，先是状态行，接着是头部，最后是主体部分。

### 组装状态行
```php
private function emitStatusLine(ResponseInterface $response): void
{
    $statusLine = sprintf(
        'HTTP/%s %s %s',
        $response->getProtocolVersion(),
        $response->getStatusCode(),
        $response->getReasonPhrase()
    );
    header($statusLine, true, $response->getStatusCode());
}
```

### 组装 header
```php
private function emitHeaders(ResponseInterface $response): void
{
    foreach ($response->getHeaders() as $name => $values) {
        $first = strtolower($name) !== 'set-cookie';
        foreach ($values as $value) {
            $header = sprintf('%s: %s', $name, $value);
            header($header, $first);
            $first = false;
        }
    }
}
```

当 header 的名称不是 set-cookie 的时候，用新的 header 替换之前相同类型的 header。

### 组装 body
```php
private function emitBody(ResponseInterface $response): void
{
    $body = $response->getBody();
    if ($body->isSeekable()) {
        $body->rewind();
    }

    $amountToRead = (int) $response->getHeaderLine('Content-Length');
    if (!$amountToRead) {
        $amountToRead = $body->getSize();
    }

    if ($amountToRead) {
        while ($amountToRead > 0 && !$body->eof()) {
            $length = min($this->responseChunkSize, $amountToRead);
            $data = $body->read($length);
            echo $data;

            $amountToRead -= strlen($data);

            if (connection_status() !== CONNECTION_NORMAL) {
                break;
            }
        }
    } else {
        while (!$body->eof()) {
            echo $body->read($this->responseChunkSize);
            if (connection_status() !== CONNECTION_NORMAL) {
                break;
            }
        }
    }
}
```

可见每 echo 一段数据都检测一下与客户端的连接状态。以上就是返回 HTTP 响应报文给客户端的流程。

---

这篇的内容比较简单，以后再看看 Slim\Psr7\Factory\ResponseFactory 创建 Response 的源码。

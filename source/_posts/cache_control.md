---
title: 浏览器如何处理 HTTP 的 Cache-Control 首部
date: 2019-12-21 17:06:02
category:
- coding
tags:
- HTTP 协议
---

众所周知，服务端可以通过响应中的一些 header 让客户端对资源进行缓存，减轻对服务端的请求压力，但是浏览器具体是怎样处理 HTTP 缓存的？例如，为什么服务端设置了 Cache-Control 首部，但刷新页面时，浏览器还是会再次请求服务端获取最新的资源？还有很多的实现细节，下文一一试验。

<!--more-->

本文所有实验使用的都是 Chrome (version 79.0.3945.88)

## Case 1：请求最简单的 HTML 文档

服务端使用以下代码：
```php
<?php
header('Cache-Control: max-age=3600');	// 告诉客户端，资源1小时内都不会过期
echo date('Y-m-d H:i:s', time());
```

使用 php 内置的 web server:
```bash
php -S 127.0.0.1:9999
```

使用浏览器访问 127.0.0.1:9999/index.php，会发现服务器发回来的 Response Header 里面包含了 `Cache-Control: max-age=3600`，但是当你使用 F5 进行刷新，或者在当前标签页再次输入这个 URL 按回车，这两种情况下，浏览器都会重新发送请求到服务端，获取最新的数据！但是如果我重开一个新的标签页，输入同样的 URL，这次浏览器却会直接使用缓存，不会请求服务端！

打开 DevTools 可以看到以下的信息，特别值得注意的是 Status Code 那里写着的 from disk cache：
```
General
Request URL: http://127.0.0.1:9999/index.php
Request Method: GET
Status Code: 200 OK (from disk cache)
Remote Address: 127.0.0.1:9999
Referrer Policy: no-referrer-when-downgrade
```

但如果在该标签页按 F5 刷新页面，它又会像之前那样，不使用缓存，再一次请求服务端了。

## Case 2：请求的 HTML 文档中嵌入别的资源

服务端响应 HTML 文档的代码：
```php
<?php
header('Cache-Control: max-age=3600');

echo '<html><body><img src="1.php"></body></html>';
```

服务端响应 1.jpg 图片的代码：
```php
<?php
header('Cache-Control: max-age=3600');	// 告诉客户端，资源1小时内都不会过期
header('Content-Type: image/jpeg');

$fh = fopen('1.jpg', "rb");
echo fread($fh, filesize('1.jpg'));
```

使用 php 内置的 web server:
```bash
php -S 127.0.0.1:9999
```

这次情况又发生了变化！首先浏览器打开标签页1，访问 127.0.0.1:9999，服务端响应 1个 HTML文档，文档中再加载1张图片，服务响应这两个资源的时候都返回了 header：`Cache-Control: max-age=3600`。
这次的不同之处是，F5 刷新页面的时候，仅仅向服务端重新请求 html 文档，而那张内嵌的图片并没有发起请求，而是直接使用本地的缓存！再打开新的标签页2，也访问 127.0.0.1:9999，这一次 HTML 和图片两个资源都没有请求服务端，都是直接使用浏览器中的缓存。

## Case 3：在网页中进行 AJAX 请求

### Case 3.1：HTML 不缓存，AJAX 请求的资源指定缓存

HTML 代码如下：
```html
<html>
<body>

<div id="demo">
  当前时间：<p id="time"></p>
  <button type="button" onclick="loadDoc()">加载当前时间</button>
</div>

<script>
function loadDoc() {
  var xhttp = new XMLHttpRequest();
  xhttp.onreadystatechange = function() {
    if (this.readyState == 4 && this.status == 200) {
     document.getElementById("time").innerHTML = this.responseText;
    }
  };
  xhttp.open("GET", "get_time.php", true);
  xhttp.send();
}
</script>

</body>
</html>
```

服务端代码如下：
```php
<?php
header('Cache-Control: max-age=3600');

echo date('Y-m-d H:i:s', time());
```
实验证明，如果 AJAX 请求的资源指定了 `Cache-Control: max-age=3600`，在当前页面重复多次请求，在缓存有效期内，并不会发送请求到服务端，而是直接使用缓存中的数据。

### Case 3.2：HTML 缓存，AJAX 请求的资源不缓存

还是使用 Case 3.1 的 HTML 文件，但在响应的 header 中加入 Cache-Control：
```php
<?php
header('Cache-Control: max-age=3600');
header('Content-Type: text/html; charset=UTF-8');

$fh = fopen('demo.html', "r");
echo fread($fh, filesize('demo.html'));
```

get_time.php 的代码则把缓存去掉
```php
<?php
echo date('Y-m-d H:i:s', time());
```
这种情况 AJAX 请求的资源就不会进行缓存了。




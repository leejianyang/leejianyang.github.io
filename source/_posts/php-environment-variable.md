---
title: PHP 使用环境变量
date: 2017-06-24 12:13:52
updated: 2017-06-24 12:13:52
tags:
- Nginx
- 环境变量
- PHP
categories:
- coding
---

本文简述在 PHP 中使用环境变量的三种方式。

<!-- more -->

例如测试环境和生产环境中的 MySQL 的密码是不一样的，那么可以把密码写在 MYSQL_PASSWORD 这个环境变量中，然后在程序中再把它读出来使用。

## 在 Nginx 中配置
其中一种方式是在 Nginx 的配置文件中配置，通过 fastcgi_param 来传给 php-fpm：

``` php
location ~ ^(.+\.php)(.*)$ {
    fastcgi_param MYSQL_PASSWORD 'Abc123';
}
```
然后程序中就可以通过以下方式读取到：
``` php
$_SERVER['MYSQL_PASSWORD']
// 或者是
getenv('MYSQL_PASSWORD')
```

## 在 php-fpm 中配置
另一种方式是先把环境变量写在 /etc/profile.d/ 文件夹的文件中
```
export MYSQL_PASSWORD="Abc123"
```
然后再在 php-fpm 的配置文件设置
```
env[MYSQL_PASSWORD] = $MYSQL_PASSWORD
```
重启 php-fpm 后，在程序中同样也可以获取到环境变量:
```
getenv('MYSQL_PASSWORD')
```

## Laravel 中使用环境变量的方式

Laravel 的做法是将类似环境变量的相关配置写在了项目根目录的 .env 文件中，其实 Laravel 是借助了 vlucas/phpdotenv 这个库来实现的。

phpdotenv 的好处有：
> - NO editing virtual hosts in Apache or Nginx
- NO adding php_value flags to .htaccess files
- EASY portability and sharing of required ENV values
- COMPATIBLE with PHP's built-in web server and CLI runner

它的内容原理很简单，环境变量先写在一个名为「.env」的文件里面，例如：
``` php
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=homestead
DB_USERNAME=homestead
DB_PASSWORD=secret
```
然后 phpdotenv 就会逐行读取文件，利用 putenv 函数设置该环境变量。

但是 phpdotenv 的初衷是用于开发和测试环境的，因为使用这种方式的话，每个请求来到的时候要从新加载一次 .env，这样的话有一定的性能损耗。但 Laravel 的做法是通过 phpdotenv 加载完环境变量后，与其他配置项一起保存在 bootstrap/cache/config.php 文件里面的，并不需要每个请求到来的时候重新读取一次 .env 文件。

## 综述
综上所述，最简单的应该是第一种方式，最常用的做法应该是第二种方式，但我个人比较偏好第三种方式。

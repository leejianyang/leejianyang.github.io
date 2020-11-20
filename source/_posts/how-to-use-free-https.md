---
title: 全站 HTTPS 实践
date: 2018-01-20 22:50:01
updated: 2018-01-20 22:50:01
tags:
- HTTPS
- Nginx
- 系统安全
categories:
- coding
---

HTTPS 可以说是网站的标配了，本文简述如何免费获取 HTTPS 证书，使网站获取浏览器的「绿挂锁」。

<!--more-->

## 使用免费证书
通常 SSL 证书是需要收费的，但个人建站的话出于成本的角度考虑，还是使用免费的证书吧。本站就是从 Let’s Encrypt 这个网站上获取免费证书的。

### 从Let’s Encrypt获取免费证书

「[Let’s Encrypt]([About Let’s Encrypt - Let’s Encrypt - Free SSL/TLS Certificates](https://letsencrypt.org/about/)」是一个提供免费证书的 CA（certificate authority）。它在官网上的自我介绍如下：
> We give people the digital certificates they need in order to enable HTTPS (SSL/TLS) for websites, for free, in the most user-friendly way we can. We do this because we want to create a more secure and privacy-respecting Web.  

#### Let’s Encrypt 的优点

- 免费
- 达到 HTTPS 的效果
- 简单易用
- 可以借助工具 Certbot 进行傻瓜式配置
- 受到浏览器信任（获得「绿挂锁」）

#### Let’s Encrypt 的运作机制

Let’s Encrypt 的运作主要分为两个步骤：
- 向CA证明该服务器有权限管理这个域名
- 通过客户端请求、更新或吊销证书

客户端证明该服务器有权限管理某个域名的方式有两种：
- 通过DNS记录
- 将某个HTTP资源放在该域名下

其他的关于运作机制的内容可以看[「Let’s Encrypt」官网上的文档](https://letsencrypt.org/how-it-works/)。

### 使用 Certbot 工具

上文提到，从 Let’s Encrypt 获取证书是需要一个「客户端」的，Certbot 就是它客户端。当然 Certbot 不仅可以作为 Let’s Encrypt 的客户端，还可以作为其他遵循 ACME 协议的 CA 的客户端。

以下为 Certbot 官网上的相关自我介绍：
> Certbot is an easy-to-use automatic client that fetches and deploys SSL/TLS certificates for your webserver. Certbot was developed by EFF and others as a client for Let’s Encrypt and was previously known as “the official Let’s Encrypt client” or “the Let’s Encrypt Python client.” Certbot will also work with any other CAs that support the ACME protocol.  
>   
> While there are many other clients that implement the ACME protocol to fetch certificates, Certbot is the most extensive client and can automatically configure your webserver to start serving over HTTPS immediately.  

#### 使用 Certbot 获取证书

安装的方法很简单，从[官网](https://certbot.eff.org)下载就可以了。

安装完毕后，有多种方式可以获取证书，其中一种是傻瓜式的，既获取证书，同时又修改 Nginx 配置文件；另一种是只获取证书，不修改 Nginx 配置文件。需要注意的是，如果 Nginx 是通过编译安装的话，往往它的配置文件不是存放在默认路径下，这会导致 Certbot 无法对配置文件进行修改，这样会导致证书获取失败。因而我只需要 Certbot 帮忙获取证书，不需要它修改 Nginx 配置，使用的命令如下：
```bash
certbot certonly
```

注意获取证书的过程中，是需要使用 80 端口的，所以 Web 服务可能需要临时关闭。证书获取成功之后，会存放在 /etc/letsencrypt 目录下，注意 Nginx 配置的证书的路径要使用 /etc/letsencrypt/live/$domain 目录下的文件，因为 live 目录下是做了软连接的，及后更新证书之后就不需要更改 Nginx 配置了。

#### 使用 Certbot 更新证书

从 Let’s Encrypt 的证书的有效期比较短，只有三个月。不过使用 Certbot 可以很方便地进行证书的更新。只要往cron计划任务中加入以下命令：
```bash
certbot renew --pre-hook "/usr/local/nginx/sbin/nginx -s stop" --post-hook "/usr/local/nginx/sbin/nginx"
```

有两个需要注意的点：
- 只有当证书还有 30 天内过期时，才会进行更新
- /etc/letsencrypt 目录下的所有证书都会进行更新
- 如前文所述，需要用到80端口，所以借助 hook 来启停 Nginx

### Nginx 配置 HTTPS

关于如何在 Nginx 中配置 HTTPS，可以参考 [Mozilla 提供的配置指南]([Generate Mozilla Security Recommended Web Server Configuration Files](https://mozilla.github.io/server-side-tls/ssl-config-generator/))，我使用的是 「 Modern」配置：
```php
    ssl_certificate /etc/letsencrypt/live/blog.leejianyang.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/blog.leejianyang.com/privkey.pem;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;

    ssl_protocols TLSv1.2;
    ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256';
    ssl_prefer_server_ciphers on;

    add_header Strict-Transport-Security max-age=15768000;

    ssl_stapling on;
    ssl_stapling_verify on;

    ssl_trusted_certificate /etc/letsencrypt/live/blog.leejianyang.com/chain.pem;
```

另外，还推荐把所有 http 请求重定向到 https 请求，进行如下配置即可：
```php
server {
    listen 80;
    server_name blog.leejianyang.com;

    return 301 https://$host$request_uri;
}
```

## 得到浏览器的认证
走完上文提到的步骤后，网站就能够通过 HTTPS 协议进行访问了，但是这还不够。还需要获得浏览器的认证，即获取浏览器地址栏 URL 左手边的绿色挂锁。要得到浏览器的认证，有若干注意事项，例如网站中使用的资源也要是 HTTPS 的，连接也要是 HTTPS，所以往往配置完证书之后，还要进行一轮修改。可以借助一些网站（例如[ Why No Padlock 这个网站](https://www.whynopadlock.com/))进行检测，检测后会告诉你需要修改 HTML 文件中的哪些内容，才能获得浏览器的认证。

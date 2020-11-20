---
title: CSRF必知必会
date: 2015-08-24 10:06:54
updated: 2015-08-24 10:06:54
tags:
- 系统安全
- Web开发
- CSRF
categories:
- coding
---

CSRF 的全称是 cross-site request forgery，中文名是跨站请求伪造。一句话来说就是，恶意网站 A 利用受害者的浏览器，在受害者不知情的情况下，向网站 B 发出请求。在好几年前 CSRF 是一个排名很高的安全问题，近几年已经受到重视，很多 web 框架都有相应的措施，但是依然值得 web 开发人员重视。

<!--more-->

## CSRF常见案例
这里就说说[OWASP](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF))给出的例子。假如有一个银行网站叫做 bank.com，如果 Alice 要通过 bank.com转账 100 元给 Bob，那么她在 bank.com 的网站页面的操作其实就是发送以下的 http 请求给 bank.com的服务器：
``` bash
GET http://bank.com/transfer.do?acct=BOB&amount=100 HTTP/1.1
```
假如我想偷偷地、非法地将 Alice 账户下的 10000 元转给一个叫 Maria 的人，那我只需要假借 Alice 的名义，发如下的 http 请求给 bank.com 的服务器：
``` bash
GET http://bank.com/transfer.do?acct=MARIA&amount=10000 HTTP/1.1
```
总不能拿支枪指着 Alice 的头让她发送这条请求吧，要偷偷地在她不知情的情况下由她本人来发出这个 http 请求，具体要怎么做呢？我只需要随便弄一个网页，引诱 Alice 来访问（引诱方法不在CSRF的范畴），甚至 Alice 不需要在这个网页上输入或者是点击任何内容，都可以完成 CSRF。这个页面只需要包含以下一行代码：
``` html
<img src="http://bank.com/transfer.do?acct=MARIA&amount=10000" width="0" height="0" border="0">
```
这行 html 代码在页面嵌入了一个宽度和高度都是 0 的图片（就是说这图片什么都没有），img 标签是自动加载的，因而 Alice 浏览该页面时，就自动发了上文说的那个 http 请求了。
为什么能够冒充 Alice 发出转账请求？因为 bank.com 的服务器是通过 cookie 的内容来判断这条转账请求是由谁来发起，是通过 cookie 来判断发起该请求的人是否输入正确的密码来登录的（当然现在的银行网站不会那么弱智）。而如果 Alice 最近登录过 bank.com，那么她的浏览器是将 cookie 保存一段时间的。Alice 访问我的网页加载那个 img 标签的资源时，就会发送那个 http 请求，并且浏览器会帮忙带上保存在本地的 cookie。然后 bank.com 查看 cookie 信息后，就会认为这笔转账请求是由 Alice 发起的，最后 Alice 的钱就被转走了。
有人会觉得，既然 GET 请求会被利用，那么如果改用 POST 方法，真可以了吧？too youg，too navie。直接看代码：
``` html
<html>
<body>

  <form name="csrf_form" action="http://VULNERABLE_APP/csrf.php" method="POST">
    <input type="hidden" name="csrf_param" value="POST_ATTACK">
  </form>

  <script type="text/javascript">document.csrf_form.submit();</script>
</body>
</html>
```
由此可见，改成 POST 方法也不能保证安全。

以上简单地展示了 CSRF 的案例，现实中可能有更复杂的手段，但本质都是一样的，就不赘述了。下面来看看 CSRF 的本质。

## CSRF的本质
这里需要引用 Bill Zeller 的一段文字：

> CSRF vulnerabilities occur when a website allows an authenticated user to perform a sensitive action but does not verify that the user herself is invoking that action.The key to understanding CSRF attacks is to recognize that websites typically don't verify that a request came from an authorized user. Instead they verify only that the request came from the browser of an authorized user. Because browsers run code sent by multiple sites, there is a danger that one site will (unbeknownst to the user) send a request to a second site, and the second site will mistakenly think that the user authorized the request.

CSRF 能够发挥作用需要几个条件：

- 服务器端处理某些敏感的请求时，使用 cookie 来验证
- 相关的 cookie 被用户的浏览器保存了下来
- 服务器端没有进一步仔细验证带上这个 cookie 的请求的发起源是哪里

看清 CSRF 的本质后，就可以针对性地进行防范了。

## 如何防御CSRF
CSRF 的防御手段有很多种，有些手段比较有效，有些手段太弱，这里只谈效果甚好应用广泛的一个种。它叫做「double-submitted cookie」。这种方法简单来说就是想办法验证 cookie 是否合法地被利用。
不同编程语言的 web app 框架采用的方法都大同小异，本质上就是想办法在 html 页面添加类似以下的一行代码
``` html
<input id="fkey" name="fkey" type="hidden" value="df8652852f139" />
```

以上代码的作用是添加一个不在页面上显示的 input 标签，使得在表单提交 POST 请求时带上一个随机字符串（这个字符串是由后端 web app 渲染页面时生成的），而在此次 post 请求过程中带上的 cookie 中也包含同一个随机字符串，后端 web app 接收到该 POST 请求时就会对它们进行比对，如果一致则证明该请求不是伪造的。
由于即便在同一个浏览器，不同网站之间是无法跨域读取 cookie 的，所以想发动 CSRF 的恶意网站无法知道 cookie 中的随机字符串值，自然也无法伪造请求了。

这次先简单写到这里，以后再写关于 RESTFUL API 的安全验证机制。

-----------
参考资料：

- [Cross-Site Request Forgery (CSRF)](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF))
- [Popular Websites Vulnerable to Cross-Site Request Forgery Attacks](https://freedom-to-tinker.com/blog/wzeller/popular-websites-vulnerable-cross-site-request-forgery-attacks/)

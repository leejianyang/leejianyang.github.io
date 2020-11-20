---
title: 《Salted Password Hashing - Doing it Right》笔记
date: 2016-03-31 15:40:03
tags:
- hash
- 用户系统
- 密码安全
- categories:
- coding
---

看了一篇名为[《Salted Password Hashing - Doing it Right》](https://crackstation.net/hashing-security.htm)的文章，讲述的是用加盐 hash 方式保存密码方面的内容。包括：什么是 password hashing，如何破解 hash，如何正确地加「盐」，如何正确地进行 hash 等内容。
<!--more-->

## 什么是 Password Hashing
将密码通过哈希函数转换成固定长度的字符串，这个过程是不可逆的。

## 关于哈希冲突
既然哈希出来的字符串的长度是固定，那么字符的组合就不是无限的，而且无论什么哈希函数总会出现冲突的情况。Cryptographic 哈希函数被设计成很难发现冲突。

## 常见的哈希破解方式
包括：
- 字典式破解或暴力破解(Dictionary and Brute Force Attacks)
- 查表法(Lookup Tables)
- 反向查表法(Reverse Lookup Tables)
- 彩虹表(Rainbow Tables)

彩虹表与查表法非常类似，但极大地提高了破解的效率。这些方法中，被类似暴力破解的方法破解出来是无可避免的，这很好理解，能做到的只是拖慢破解的速度。而应对其余的破解方式，最有效的方法就是加「盐」。

## 关于加「盐」
### 什么是「盐」
如果使用相同的哈希函数，那么相同的密码就会生成同样的哈希值。但是如果在密码中加入额外的随机的字符串，那么就可以使相同的密码可以对应不同的哈希值。这个随机的字符串就是所谓的「盐」。

### 加「盐」的错误做法
- 重复使用：不同的用户不能使用相同的盐
- 盐的长度过短：盐的长度最好与生成的哈希字符串的长度相同

### 使用「真」随机数作为盐
很多随机函数生成的随机数都是「伪随机的」，因而应该使用一种称为 Cryptographically Secure Pseudo-Random Number Generator 的生成技术，它生成的随机数足够随机。

不同的编程语言提供了不同的库：
- PHP: mcrypt\_create\_iv, openssl\_random\_pseudo_bytes
- Java: java.security.SecureRandom
- C#: System.Security.Cryptography.RNGCryptoServiceProvider
- Ruby: SecureRandom
- Python: os.urandom
- Perl: Math::Random::Secure
- C/C++(Windows API): CryptGenRandom
- GNU/Linux or Unix: /dev/random or /dev/urandom

## 哈希函数的选择
- 现代的经过验证安全可靠的算法，例如：SHA256, SHA512, RipeMD, WHIRLPOOL, SHA3等
- [Portable PHP password hashing framework](http://www.openwall.com/phpass/)
- 使用 key derivation 函数，例如：PBKDF2, bcrypt或scrypt等
- crypt 的安全版本(例如$2y$, $5$, $6$)

有些例如 MD5、SHA1 那些，已经过时了。

### 「慢」哈希函数
破解者如果用 GPU 或者是一些特殊的硬件可以大大加快破解的速度，对应的方法是有一种叫做「key stretching」的技术。它的原理就是令到哈希计算非常慢，但这种慢对于正常的业务计算来说是可以接受的，但对于破解者来说因为计算量大就无法接受了。

比较有名的算法有[PBKDF2](http://en.wikipedia.org/wiki/PBKDF2)或者[bcrypt](http://en.wikipedia.org/wiki/Bcrypt)。需要注意的是通常这类算法的函数都会接受一个类似「iteration count」的参数，这个值越大越安全，但是越大就越慢，需要权衡取舍。

## 「找回密码」设计策略
使用 email 验证的方式，将一个包含随机 token 的连接发到用户邮箱，然后用户点那个链接跳转页面重置密码。但是这个 token 必须设置过期时间（例如15分钟），并且只能使用一次。而且注意不要在 token 中包含用户信息或者时效信息，必须是不可预计的随机值。更不要采用将随机新密码发到邮箱的方式。

## 「密码强度」设计策略
没必要过于硬性限制用户的密码强度，更好的做法是在用户设置密码的时候实时反馈密码的强度给用户看，让用户自己来选择。

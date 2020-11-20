---
title: 字符编码
date: 2016-08-12 22:16:16
updated: 2016-08-12 22:16:16
tags:
- Python
- 字符编码
categories:
- coding
---

字符编码是编程中遇到的一个很基础但很重要的知识点，虽然很多时候框架或库已经帮我们处理好这方面的问题，但对一些基础的内容有所了解还是很必要的。

<!-- more -->
## 基本概念
- 码位(code point)：可以理解为每个字符的编号，通常用十六进制表示
- 字符集(Coded Character Set)：可以理解为一个集合，集合内的每个元素是 tuple (字符, 码位)
- 码元(code units)：码元是能用于处理或交换编码文本的最小**比特组合**
- 字符编码表(CEF)：码位和码元的双射。例如 UTF-8, UTF-16, UTF-32 等
- 字符编码方案（CES）：可以理解为 CEF+ 字节序列化方案。字节序列化方案涉及到一些问题：
    - 大端模式和小端模式：每个码元究竟是高位字节在前还是低位字节在前
    - 如何标记究竟是使用 big-endian 还是 little-endian
- encoding: 码位 ---> 字节流
- decoding: 字节流 ---> 码位

## Unicode和UTF-8的实例
以中文字「李」为例：
- unicode 码位：674E
- 674E 改成用二进制表示就是：01100111 01001110
- 然后使用 utf-8 进行编码：
    - 由于 utf-8 是「变长」的，要根据具体情况使用 1-4 个码元（每个码元就是8bit）来表示一个字符
    - 有些特殊的规则占位：1110xxxx 10xxxxxx 10xxxxxx
    - 将 674E 的二进制值塞回去以上规则得到：11100110 10011101 10001110
    - 将上面得到的二进制值变回十六进制就是：e6 9d 8e
    - 因而「李」字的 utf8 码元序列用十六进制来表示就是 e6 9d 8e

### UTF-8的编码方式

码位的二进制位数 | 码位最小值 | 码位最大值 | 码元数量 | byte1 | byte2 | byte3 | byte4 | byte5 | byte6
--------------|-----------|-----------|--------|-------|-------|-------|-------|-------|------
7 | U+0000 | U+007F | 1 | 0xxxxxxx | | | | |
11 | U+0080 | U+07FF | 2 | 110xxxxx | 10xxxxxx | | | |
16 | U+0800 | U+FFFF | 3 | 1110XXXX | 10XXXXXX | 10XXXXXX | | |
21 | U+10000 | U+1FFFFF | 4 | 11110xxx | 10XXXXXX | 10XXXXXX | 10XXXXXX | |
26 | U+200000 | U+3FFFFFF | 5 | 1111110XX |  10XXXXXX | 10XXXXXX | 10XXXXXX | 10XXXXXX |
31 | U+4000000 | U+7FFFFFFF | 6 | 11111110X | 10XXXXXX | 10XXXXXX | 10XXXXXX | 10XXXXXX | 10XXXXXX

## 与HTTP有关的内容
### Server Side
服务器通过 HTTP 协议的 Content-Type 首部中的 charset 参数和 Content-Language 首部告知客户端文档的字母表和语言。
例如:
```
Content-Type: text/html; charset=iso-2022-jp
```
如果没有显示地列出字符集，接收方可能就要设法从文档内容中推断出字符集。对于 HTML 内容来说，可以在 <HEAD> 的 <meta> 标签中找到相关信息：
```
<meta charset="utf-8" />
```
### Client Side
客户端发送 Accept-Charset 首部和 Accept-Language 首部，告知服务器它能理解哪些字符集编码算法和语言以及其中的优先顺序。
### URL编码
如果当 URL 中包含中文如何处理呢？根据我看[《RFC3986》](https://tools.ietf.org/html/rfc3986#section-2.4)的理解，应该是先将字符用 UTF-8 进行编码，如果能够直接用合法的非保留字符表示则用非保留字符来表示，否则则用「[百分号编码](https://zh.wikipedia.org/wiki/%E7%99%BE%E5%88%86%E5%8F%B7%E7%BC%96%E7%A0%81)」来表示。例如上文说的「李」字则表示为 %e6%9d%8e。

## 与Python有关的内容
*这部分目前只覆盖Python 2.7*

### str，unicode，encode和decode
Python 2 中有两种数据类型，分别是 str 和 unicode，通常分别叫做「字符串」和「unicode字符串」。将 str 转换成 unicode 的过程是 decode，反之将 unicode 转换成 str 的过程是 encode，这两个过程都是需要指定 encoding 参数（也就是编码方案）的。

其中容易迷惑的地方就是要注意「Python解释器的默认编码方案」和「终端使用的编码方案」。

#### 解释器默认编码方案
Python2 解释器的默认编码方案是 ASCII，因为往往在代码的第一行需要声明：
```Python
# encoding: utf-8
```
或者
```Python
# -*- coding:utf-8 -*-
```

#### 终端使用的编码方案
例如 Linux 使用的是 utf8，windows 使用的 gbk，那就意味着：
如果我在我的 macbook 打开 ipython:
```Python
>>> s = '李'
>>> s
'\xe6\x9d\x8e'
```
但是换成 windows 机器的话以上得到的结果就不是「'\xe6\x9d\x8e'」了

### 怎么知道一段字节序列使用哪种编码方案
使用[chardet](https://pypi.python.org/pypi/chardet)

### 与外部相关的操作
从外部都回来的数据应该直接是 str，确定数据使用的编码方案后进行 decode，便可以对数据进行使用；要写到外部时，使用某个编码方案进行 encode，再写。

----------

## 参考资料
- [知乎：冯若航的回答](https://www.zhihu.com/question/31833164/answer/115069547?from=profile_answer_card)
- [知乎：zhijun liu的回答](https://www.zhihu.com/question/31833164/answer/114694586?from=profile_answer_card)
- [维基百科：UTF-8](https://zh.wikipedia.org/wiki/UTF-8)

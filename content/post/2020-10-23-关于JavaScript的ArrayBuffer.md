---
title: 关于JavaScript的ArrayBuffer
categories:
  - JavaScript
date: 2020-10-23 22:28:42
updated: 2020-10-23 22:28:42
tags:
  - JavaScript
---

今天遇到一个情况，就是服务器端是采用的 WebSocket 进行通信的，但是协议是 `ASCII`，因此，传输给到我客户端的，是字节流，在 JS 看来，就是 ArrayBuffer。现在面临的问题，就是要要将字节流转换成字符串，然后解析其中的 JSON 信息。之前还没有遇到过，因此就来探究了一下。

<!--more-->

# ArrayBuffer

根据 [MDN 上文档的说明](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer)。**`ArrayBuffer`** 对象用来表示通用的、固定长度的原始二进制数据缓冲区。

> 它是一个字节数组，通常在其他语言中称为“byte array”。

也就是说，Buffer 是存储二进制，字节的地方，由其组成的数组；不过，对于我们可以对 `ArrayBuffer` 看成是不同长度的 Buffer 组成的。

因此，ArrayBuffer 可以理解成其实就是一个个字节组成的序列了。

实际上 ，[更详细的描述看这里](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Typed_arrays)

# 关于字符与编码

关于字符，与编码，这里不再多说。更多请看 {% post_link Python2和3中字符串与编码 Python2和3中字符串与编码 %} 一个道理。

我们来明确 JS 两个意思：

- 码点 CodePoint ，指的是一个符号在所采用的字符集中的一个整型值（相当于索引值）
- CharCode，指的是一个编码存储，具体在计算机机内是如何存储的。存储的实现来说，是字节序列。

理论上，我们的 `CodePoint` 是一个整型，直接在程序中有一个整数来表示不就行了，我们的 `CharCode` 和 `CodePoint` 应该相等了是吧？

但事实上不是这样的。为什么呢？因为历史的因素，要考虑到兼容性，内存，传输等方面的原因，所以就做了取舍。不同的编码方式，采用了不同的做法。

## 编码

编码的定义是指，对于一个字符集中的码点，我们按照什么规则来存储的问题。

## 编码的转换

我们想要从一种编码转换到另外一种编码，就必须读出码点值，然后根据码点值重新编码。

## UTF8 编码

UTF8 变长字节的存储码点，其规则是：

1. 单字节字符，最高为设置为 0，后面的 7 位存储码点。
2. 对于多字节字符，有几个字节，第一字节的前面几位就设置为 1，后面每个字节的前两位都设置为 10

最后就是这样对应：

```
Unicode符号范围     |        UTF-8编码方式
(十六进制)        |              （二进制）
----------------------+---------------------------------------------
0000 0000-0000 007F | 0xxxxxxx
0000 0080-0000 07FF | 110xxxxx 10xxxxxx
0000 0800-0000 FFFF | 1110xxxx 10xxxxxx 10xxxxxx
0001 0000-0010 FFFF | 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx
```

比如对于 Unicode 字符集， UTF8 编码，就采用了 1-6 个字节来将码点，存储在计算机中。
而对于 Unicode 字符集， UCS2 编码，就采用了 2 个字节来将码点存储在内存中；而 UTF16 使用 2 个字节来编码常用字符，用 4 个字节来编码不常用的字符；UTF16 是 utf16 的一个超集。

举个例子：

中 字，Unicode 码点是 _4e2d_，我们要存储在计算机的时候，采用 utf8 编码，就是将这个码点，用 utf8 映射到字节去，是 _e4b8ad_ 以二进制来对比：

```
1001110 00101101 Unicode 码点
11100100 10111000 10101101 utf8 编码后
```

将 utf8 编码后的取出有效字节来就是：

```
0100 111000 101101
```

# JS 中的码点获取

有两个函数：

- String.prototype.charCodeAt() 这个返回的码点最多 2 字节，它表示的是 ucs2/utf16 中的码点值（也就是常用的那些汉字数）
- String.prototype.codePointAt() 这个返回的码点最多 4 字节，返回的表示 Unicdoe 中的码点值

在码点值在 2 字节也内，没有区别。但超过两字节编码的字符的就会出问题了撒

比如这个字：

```js
"𠮷".charCodeAt(0);
"𠮷".codePointAt(0);
```

# 回到我们的问题

对于我们的服务端，采用的是 utf8 编码，（大多数服务器现在都用这个编码），也就是说，它返回的实际上是一系列字节流，这些字节点代表的是字符的 utf8 编码值。

而对于 JS 内部，使用的 utf16 ，两字节来将存储一个码点。那么，我们要让我们传输的字符确实能够在 JS 内正常的使用，就必须转成 utf16 编码。

而我们要了 JS 内的数据传输到服务器的时候，也必须将 utf16 编码转换成 utf8 编码的字节序列。

用 ArrayBuffer 来进行存储。

因此，我就找了 utfx 这个库，这才能实现我想要的目的。

# utfx 库

[utfx 开源库](https://github.com/dcodeIO/utfx)

这个函数的实现有点那个啥啊。
他的函数一般都定义成：

```js
utfx.UTF16toUTF8(src, dst);
```

这个形式，其实 src 每调用一次就会返回一个元素，然后转换后，就以转换后的结果为参数调用 dst

这和 LUA 中 ltn12 中的 sink/source/pump 的概念累似

---
title: JS中使用正则表达式的exec与match的问题
categories:
  - JavaScript
date: 2020-03-18 16:16:01
updated: 2020-03-18 16:16:01
tags: 
  - JavaScript
---

问题的起因在于，我在弄一个插件的时候，使用了 `exec()` 来进行匹配，结果总是只能匹配到最后一个。思索良久得不到答案，于是我就抱着试试的想法，换成了`match()` 来进行，结果就OK了。这是为什么呢？

<!--more-->

# 定义

首先，这两者的定义是不同的。

- `exec()` 是定义在 Regex 上的方法。
- `match()` 是定义在 String  上的方法。

# RegExp.prototype.exec()

> 此方法会在一个指定的 字符串 上对一个匹配模式进行搜索。返回一个数组，或者 null

当 RegExp 有了 `global(g)` 和  `sticky(y)` 标志的时候（如 `/foo/g` 或 `/foo/y`），其是有状态的。他们会在从前一个匹配处存储一个 `lastIndex` 索引。内部使用这个索引，可以在一个字符串上进行迭代匹配（会捕捉分组）。而如果我们只是想获得匹配的字符串，我们应该使用 [`String.prototype.match()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/match)

重点来了：

如果匹配成功，`exec()` 会返回一个数组加上一些其他的内容（`index, input` 等）。返回的数组中，匹配的字符会是第一个值，然后是匹配文本的每个括号分组。举个例子：

```js
var someText="web2.0 .net2.0";
var pattern=/(\w+)(\d)\.(\d)/g;
var outCome_exec=pattern.exec(someText);
var outCome_matc=someText.match(pattern);

console.log(outCome_exec, outCome_matc)

```

其输出会是什么呢？

```json
[
  'web2.0',
  'web',
  '2',
  '0',
  index: 0,
  input: 'web2.0 .net2.0',
  groups: undefined
] [ 'web2.0', 'net2.0' ]
```

可以看到，其只返回了第一个整个表达式匹配的字符 *web2.0*，以及 `(\w+), (\d)` 的所有匹配的内容。实在是太惊讶了。

而我们  match 才是我们想要的内容。

原来就 是这么的简单。
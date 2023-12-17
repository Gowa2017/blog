---
title: Promise的使用及API
categories:
  - JavaScript
date: 2018-12-12 00:52:05
updated: 2018-12-12 00:52:05
tags: 
  - JavaScript
  - Promise
---
完全是闲的，工作暂时用不到，但是看书偶尔看到了，不了解一下心里就根猫抓的一样。 *Promise* 对象，代表的是一个**异步**操作的最终完成（或失败）及其返回值。[原文地址](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)

<!--more-->

# 例子
开篇就是一个例子：

```js
var promise1 = new Promise(function(resolve, reject) {
  setTimeout(function() {
    resolve('foo');
  }, 300);
});

promise1.then(function(value) {
  console.log(value);
  // expected output: "foo"
});

console.log(promise1);
// expected output: [object Promise]

```

执行脚本的输出是：

```js
> [object Promise]
> "foo"
```

# 语法

```js
new Promise( /* executor */ function(resolve, reject) { ... } );
```

参数：executor。 一个以 *resolve, reject* 为参数的函数，我们就叫它 executor 函数吧。executor 函数会被 Promise 的实现立即执行（这个函数会在 Promise 构造器返回创建好的对象前执行），参数 *resolve, reject* 都是函数哦。调用 *resolve, reject* 分别表示 解决或拒绝了此次 Promise。executor 函数通常会执行一些异步操作，一旦完成成，就会调用  *resolve* 来解决或在有错误时调用*reject* 来拒绝。如果 executor 抛出一个错误，那么本次 Promise 被拒绝，返回值被忽略。

# 描述

Promise 是一个值的代理，但在这个 promise 创建的时候并不需要知道。这允许我们把异步操作的最终完成或失败与事件处理器相关联。这让异步方法像同步方法一样返回值：asynchronous 方法能立即返回一个将来可能用到的 *promise*，而不是立即返回最终值。

一个 *Promise* 有几种状态：

* **pending** 初始状态
* **fulfilled** 操作已成功
* **rejected** 操作失败

pending 状态的 Promise 可以用一个值来 *fulfilled* ，或用一个原因（错误）来 *rejected*。当前面两个操作中的任意一个发生时，通过 promise 的 `then()` 函数排队的相关事件处理器就会被调用。（如果在事件处理器排队的时候 Promise 已经 *fulfieed* 或 *rejected*，此处理器会被立即调用。所以在异步操作完成与设置处理器间没有竞争条件）

`Promise.prototype.then()，Promise.prototype.catch()` 都返回的是 promise，所以可以链式调用。

![](../res/promises.png)

> *fulfilled, rejected* 状态的 Promise 都可以被叫做 *settled*。

# 属性

* Promise.length 总是为 1。 代表构造器参数。
* Promise.prototype Promise构造器的原型。

# 方法

* **Promise.all(iterable)** 在所有的 *iterable* 参数中的所有 promises 都满足的情况下返回一个 fulfilled 的 promise，或者，一旦 *iterable* 中有一个 Promise 出现错误就返回一个 rejected 的 promise。也就是说，如果返回的 Promise 是 fulfilled 的，那么返回值是一个数组，其中的是值于 *iterable* 中的 promise。如果返回的是 rejected的，那么就是第一个被拒绝的 promise。这个类似于多个文件描述符的 select/epoll 这样。
* **Promise.race(iterable)** *iterable*中有有一个 Promise 完成（fulfill or rejectd），就返回。
* **Promise.reject(reason)** 返回一个被给定的 *reason* 拒绝的 Promise。
* **Promise.resolve(value)** 返回一个被给的值　*resolve* 的Promise。如果这个值是 **可then** 的（也就是有一个 `then()`方法），返回的 Promise 就会执行 then()方法，获取最终状态；不然的话就用 *value* 来 fulfill 返回的 Promise。通常，如果您不知道某个值是否为promise，则Promise.resolve（value）代替它并将返回值作为promise。

# 原型

## 属性

**Promise.prototype.constructor** 返回创建了一个 Promise 实例的原型。默认情况下就是 `Promise()` 函数。

## 方法

* **Promise.prototype.catch(onRejected)** 给 Promise 一个拒绝回调函数。只处理 reject 的情况。其行为和调用 `Promise.prototype.then(undefined, onRejected)`相似，（实际上，调用 Obj.catch(onRejected)内部调用的是 obj.(undefined, onRejected)。这意味着，即使是你想得到一个 undefined 值的情况下，你也要提供这个 *onRejected*函数。
* **Promise.prototype.then(onFulfilled, onRejected)** 两种情况下的回调函数。
* **Promise.prototype.finally(onFinally)** 这个函数会  *settled* 的时候被调用。无论是 fulfilled 或 rejected。
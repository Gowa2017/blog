---
title: ReactNative中WebSocket实现的BUG
categories:
  - [JavaScript]
  - [React]
date: 2020-08-09 13:50:15
updated: 2020-08-09 13:50:15
tags:
  - JavaScript
  - React Native
  - WebSocket
  - React
---

本来，在 JS 或者是 Node 中使用 WebSocket 都是很简单的问题。但是，想用 React Native 进行开发的时候，使用其自带的 WebSocket 库，就出现很多的问题。

<!--more-->

# 一般的使用办法

```js
const WebSocket = require("ws");
let socket = new WebSocket("ws://127.0.0.1:8888", "ascii");
socket.onopen = (e) => {};
socket.onmessage = (e) => {
  console.log(e.data.toString());
  socket.send("json");
};

socket.onclose = (e) => console.log(e.code, e.reason);
socket.onerror = (e) => console.log(e.message);
```

OK，如果这个用 Node 中的 _ws_ 库来跑，没有任何问题。

但是用 React Native 的问题就出现了

# 问题

## onmessage 数据为空

在这个回调中，收到的事件，其返回的数据，当服务器是传输的二进制的数据，而不是字符串的时候，返回的会是一个空的 _ArrayBuffer_，拿不到任何数据。

原因：ReactNative 不支持二进制数据。

解决办法：

```js
let socket = new WebSocket(cfg.HOST, cfg.PROTOCOL);
socket.binaryType = "blob";
socket.onopen = (e) => {
  console.log("open");
};
socket.onmessage = (e) => {
  console.log(e.data);
  var reader = new FileReader();
  reader.readAsText(e.data, "UTF-8");
  reader.onloadend = (ev) => {
    store.dispatch(serverMsgCreator(reader.result));
  };
};

socket.onclose = (e) => console.log(e.code, e.reason);
socket.onerror = (e) => console.error(e.message);
```

采用 blob，格式，然后读成文本。感觉这样效率挺低的。

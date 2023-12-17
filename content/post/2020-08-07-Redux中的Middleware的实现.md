---
title: Redux中的Middleware的实现
categories:
  - JavaScript
date: 2020-08-07 09:08:19
updated: 2020-08-07 09:08:19
tags:
  - JavaScript
  - Redux
---

在 [文档 Redux 的 Middleware](https://www.redux.org.cn/docs/advanced/Middleware.html) 中描述了整个 Middleware 实现的原理和过程，但如果不从整体上来看一下代码的话，还是有点懵逼的，知道怎么用，知其然而知道其所以然还是有点没有底。所以需要来看一下代码的具体实现。

<!--more-->



# Middleware是什么

>Middleware 接收了一个 `next()` 的 dispatch 函数，并返回一个 dispatch 函数，返回的函数会被作为下一个 middleware 的 `next()`，以此类推。由于 store 中类似 `getState()` 的方法依旧非常有用，我们将 `store` 作为顶层的参数，使得它可以在所有 middleware 中被使用。

>Middleware 只是包装了 store 的 [`dispatch`](https://cn.redux.js.org/docs/api/Store.html#dispatch) 方法。技术上讲，任何 middleware 能做的事情，都可能通过手动包装 `dispatch` 调用来实现，但是放在同一个地方统一管理会使整个项目的扩展变的容易得多。

# 怎么使用

一般来说，我们在创建 Store 的时候进行应用 Middleware，注意，一个 Middlerware 只是一个函数而已。

```js
import { createStore, applyMiddleware } from 'redux'
import todos from './reducers'
const logger = store => next => action =>{
      console.log('will dispatch', action)

    // 调用 middleware 链中下一个 middleware 的 dispatch。
    const returnValue = next(action)

    console.log('state after dispatch', getState())

    // 一般会是 action 本身，除非
    // 后面的 middleware 修改了它。
    return returnValue
}
const store = createStore(todos, ['Use Redux'], applyMiddleware(logger))
store.dispatch({
  type: 'ADD_TODO',
  text: 'Understand the middleware'
})
```

`createStore()` 指定了，`reduces, 初始状态,Middlewares`。



# 实现

实现还是需要联系两个地方 `createStore()` 和 `applyMiddleware()`，这两个函数大量的用到了闭包，ES6 的箭头函数，所以看起来还是有点累的。

## createStore

```js
export default function createStore(reducer, preloadedState, enhancer) {}
```

在这个函数中，三个参数代表了：

- reducer 一个总的 reducer 函数
- preloadedState 初始化的 state
- enhancer 增强器

enhancer 用来提高我们建立 store 的能力，它是一个函数，在 Redux 中，我们使用的就是 `applyMiddleware` 来返回一个 enhancer。

指定 enhancer  时候，实际上就将具体 store 交给了 enhancer。

## applyMiddleware



```js
export default function applyMiddleware(...middlewares) {
  return createStore => (...args) => {
    const store = createStore(...args)
    let dispatch = () => {
      throw new Error(
        'Dispatching while constructing your middleware is not allowed. ' +
          'Other middleware would not be applied to this dispatch.'
      )
    }

    const middlewareAPI = {
      getState: store.getState,
      dispatch: (...args) => dispatch(...args)
    }
    const chain = middlewares.map(middleware => middleware(middlewareAPI))
    dispatch = compose(...chain)(store.dispatch)

    return {
      ...store,
      dispatch
    }
  }
}
```



在这里我们看到有几个关键，对于每个 middleware 都传入了 

```js
    const middlewareAPI = {
      getState: store.getState,
      dispatch: (...args) => dispatch(...args)
    }
```

进行调用，最后形成一个 middleware 数组，

再对这个数组，组成一个 dispatch 链。

最后进行返回。
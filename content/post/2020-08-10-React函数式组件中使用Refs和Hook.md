---
title: React函数式组件中使用Refs和Hook
categories:
  - [JavaScript]
  - [React]
date: 2020-08-10 14:45:46
updated: 2020-08-10 14:45:46
tags:
  - JavaScript
  - React
  - React Native
---

事情的缘由，是在于我想要在一个列表的数据变更的时候，自动将列表滑动到最底部；同时对于一个输入框，保持焦点。这就需要直接操作组件了。而对于函数组件，是不能使用生命周期回调函数的。折腾了许久才找到了解决的办法。这个例子中的我是使用了 React Native 和 Redux ，所以就不做什么变更了。

<!--more-->

函数式组件是以前的趋势，所以就这样用了。同时 React 16.8 以后，提供了一些 Hook.

> _Hook_ 是 React 16.8 的新增特性。它可以让你在不编写 class 的情况下使用 state 以及其他的 React 特性。

# 原先的组件

```js
let App = (props) => {
  send = (e) => {
    props.cmd(e.nativeEvent.text);
  };
  return (
    <>
      <Text>{props.msg}</Text>
      <TextInput
        style={styles.input}
        autoCapitalize="none"
        autoCompleteType="off"
        onSubmitEditing={send}
      ></TextInput>
    </>
  );
};
const s2p = (state) => {
  return { msg: state.msg };
};

export default connect(s2p, { cmd })(App);
```

这个例子中只保留了想要重新获取焦点的 TextInput。对于 Class 类型的组件，我们可以通过对 TextInput 直接指定一个字符串的 'ref' 就行了，然后在代码的其他地方用 `this.refs.input` 进行引用就行了。

但是函数式必须使用一些其他特殊的手段才能进行，因为对于一个函数式组件，是没有实例的，无法通过 `this.` 的形式进行引用。这就要用到了 `useEffect(), useRef` 这两个函数了。

# useRef

具体的说明见 [官方文档 useRef](https://zh-hans.reactjs.org/docs/hooks-reference.html#useref)

> useRef 返回一个可变的 ref 对象，其 .current 属性被初始化为传入的参数（initialValue）。返回的 ref 对象在组件的整个生命周期内保持不变。
> 本质上，useRef 就像是可以在其 .current 属性中保存一个可变值的“盒子”。

useRef 创建的其实就是一个普通对象，其内拥有一个 `current` 键，这个与我们自己建立一个对象的区别在于：useRef 每次渲染的时候会返回同一个对象。

之后，我们将 TextInput 的 `ref` 属性设置为我们建立的这个 Ref 对象，那么就会自动将实例绑定过来。我们就可以使用了。

如：

```js
const input = useRef(null);
return (
  <>
    <Text style={styles.container}>{props.msg}</Text>
    <TextInput
      ref={input}
      style={styles.input}
      autoCapitalize="none"
      autoCompleteType="off"
      onSubmitEditing={send}
    ></TextInput>
  </>
);
```

# useEffect

[useEffect 文档在此](https://zh-hans.reactjs.org/docs/hooks-reference.html#useeffect)

签名：

```js
    function useEffect(effect: EffectCallback, deps?: DependencyList): void;
```

> 使用 useEffect 完成副作用操作。赋值给 useEffect 的函数会在组件渲染到屏幕之后执行。你可以把 effect 看作从 React 的纯函数式世界通往命令式世界的逃生通道。

> 默认情况下，effect 将在每轮渲染结束后执行，但你可以选择让它 在只有某些值改变的时候 才执行。这个我们就需要传递第二个参数了，这样就只会在我们传递的第二个参数进行改变的时候才会进行渲染。

如:

```js
useEffect(() => {
  input.current.focus();
});
```

# 最终例子

```js
let App = (props) => {
  send = (e) => {
    input.current.clear();
    props.cmd(e.nativeEvent.text);
  };
  const input = useRef(null);
  useEffect(() => {
    input.current.focus();
  });

  return (
    <>
      <Text style={styles.container}>{props.msg}</Text>
      <TextInput
        ref={input}
        style={styles.input}
        autoCapitalize="none"
        autoCompleteType="off"
        onSubmitEditing={send}
      ></TextInput>
    </>
  );
};
const s2p = (state) => {
  return { msg: state.msg };
};

export default connect(s2p, { cmd })(App);
```

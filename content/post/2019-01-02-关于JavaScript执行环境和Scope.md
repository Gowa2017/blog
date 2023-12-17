---
title: 关于JavaScript执行环境和Scope
categories:
  - JavaScript
date: 2019-01-02 09:46:23
updated: 2019-01-02 09:46:23
tags: 
  - JavaScript
---
此文章来源于 medium 上作者的系列文章，做个翻译和记录，感觉讲得比较直观易解。[javascript-demystified](https://codeburst.io/javascript-demystified-variable-hoisting-c3c4d2e8fd40)

<!--more-->

# 变量提升

其开篇几以一个非常简单的例子来问大家：

```js
console.log('x is', x)

var x

console.log('x is', x)

x = 5

console.log('x is', x)
```

三个 `console.log` 的输出是什么。结果是：

```
x is undefined
x is undefined
x is 5
```
为什么会是这个结果而不是其他的？

因为：


JavaScript 的解释器会在解释（编译） js 脚本的时候，类似于把所有的变量的声明提升到最前，所以，你可以在看起来变量声明之前使用它，不过得到的是一个 undefined 的值。 undefined 是一个正常值，我把它理解为和 null, Nan 差不多这样。

# 函数提升

第一个例子：

```js
sayHello()

function sayHello () {
  function hello () {
    console.log('Hello!')
  }
  
  hello()
  
  function hello () {
    console.log('Hey!')
  }
}
```

输出是什么？

第二个例子：

```js
sayHello()

function sayHello () {
  function hello () {
    console.log('Hello!')
  }
  
  hello()
  
  var hello = function () {
    console.log('Hey!')
  }
}

```

这个的输出又是什么。


第三个例子：

```js
sayHello()

var sayHello = function () {
  function hello () {
    console.log('Hello!')
  }
  
  hello()
  
  function hello () {
    console.log('Hey!')
  }
}
```

结果：

1. 第一个例子输出是 Hey!
2. 第二个例子输出是 Hello!
3. 第三个例子输出是 TypeError。

为什么？

同变量提升一样，对于函数声明，也会进行提升。所以可以在函数定义之前使用它。这也就是第一个例子和第二个例子都可以正常输出的原因。第一个例子输出 Hey，是因为重复定义了，后一个生效。

第二个例子输出是 Hello ，是因为首先定义一个叫 hello 的函数，再次用 var 声明变量的时候已经存在叫 hello 的变量了；只有在运行时，对 hello 重新进行赋值为函数表达式了。

第三个例子一样不用说了，因为其是变量，在执行  sayHello() 的时候，var sayHello  的值是 undefined ，所以肯定会报错了。

对于第三个例子： 

```js
var sayHello = function () {
  function hello () {
    console.log('Hello!')
  }
  
  hello()
  
  function hello () {
    console.log('Hey!')
  }
}
```

叫做函数表达式，是不会进行提升的。

# Scope

```js
var greet = 'Hello!'

function sayHi () {
  console.log('2: ', greet)
  var greet = 'Ciao!'
  console.log('3: ', greet)
}

console.log('1: ', greet)
sayHi()
console.log('4: ', greet)
```

输出的结果是：

```
1: Hello!
2: undefined
3: Ciao!
4: Hello!
```

作者在这里的提醒是： 作用域 和 执行环境 非常的相近，但并不一样。

## 作用域

作用域定义了你所处代码位置，能访问的变量和函数。

```js
var greet = 'Hello!'

function sayHi () {
  console.log('1: ', greet)
}

sayHi()
console.log('2: ', greet)

// 1: Hello!
// 2: Hello!
```


```js
function sayHi () {
  var greet = 'Hello!'
  console.log('1: ', greet)
}

sayHi()
console.log('2: ', greet)

// 1: Hello!
// ReferenceError: greet is not defined
```

为什么第二个例子的输出会是 引用错误？

第一个和第二个例子的不同再于：greet 定义的位置不同，一个位于 sayHi 函数内，一个在 sayHi 函数外。我们可以在函数内访问函数外定义的 greet，相反则不能。这是因为，在函数外第一的变量 greet 具有全局作用域，而在 sayHi 内定义的只有本地作用域。

那么 全局作用域和本地作用域又是什么？

## 全局作用域

默认作用域，引擎在执行代码前就已经定义了。一般情况下我们只有一个全局作用域，我们打开 chrome，直接在 console 里面输入  this 就能查看我们当前的全局作用域。 

全局作用域的变量，也叫全局变量，可以在其他任何地方访问和修改。

## 本地作用域

本地作用域是在全局作用域内建立的作用域。每当声明一个新的函数的时候，即建立了一个本地作用域，在函数内声明的变量就属于这个刚建立的作用域。

在执行环节，本地变量只能在其声明的作用域内被访问和修改。当函数执行完毕的时候，就返回到全局作用域，就会丢失在本地作用域内的变量。

这就是为什么上面的例子2中会报错的原因。

另外，在全局作用域中可能有多个本地作用域，本地作用域间互不影响，相互隔离。

# 执行环境(Execution context- EC)

前面我们说到，执行环境EC与作用域关联非常密切，但并不一样。经常，这两个概念会被不适当的理解同时进行交换使用，这可能会导致一些危险。

## EC != SCOPE

我们现在只需要记住，在 JavaScript 引擎开始阅读我们代码的时候，会发生下面的几件事情：

1. 全局执行环境在任何代码被执行之前建立
2. 当 **执行** 一个函数的时候，会建立一个新的执行环境。
3. 每个执行环境都提供 this 关键字，其指向当前代码在其内执行的一个对象。


现在我们来看一下下面的代码：

```js
var globalThis = this

function myFunc () {  
  console.log('globalThis: ', globalThis)
  console.log('this inside: ', this)
  console.log(globalThis === this)
}

myFunc()

// globalThis: Window {...}
// this inside: Window {...}
// true
```

你可能会有疑问，为什么 globalThis 与 myFunc 内的 this 指向同一个对象呢，虽然，我们是在一个不同的作用域内访问了 this。

这就是我们之前重复的：作用域与执行环境并不一样。但是，作用域在执行环境的定义上，扮演了一个非常关键的角色。

那么，到底什么是执行环境？

## EC定义

在 JavaScript 中，执行环境，执行上下文是一个非常抽象的概念，其维护了当前代码执行的环境信息。

> 注意： JavaScript 在任何代码开始执行之前就建立了全局执行环境。接着，每当执行一个函数的时候就会建立一个新的执行环境。事实上，全局执行环境也没有什么特别的。只不过其是在任何代码执行之前建立罢了。


## 内存建立环节

当一个新的函数被调用的时候， JavaScript 需要花一点点时间来进行配置执行环境。这个环节很重要。

在这个环节会发生下面的事情：

1. 建立作用域
2. 建立作用域链
3. 确定 this 的值。

### 作用域 Scope

每个执行环境需要了解其自身的作用域————换句话说，其需要知道其自己能访问哪些变量和函数。**Hoisting** 就在这个环节发生，JavaScript 会扫描所有的代码，然后把变量和函数声明放到内存中去。

### 作用域链 Scope Chain

除了自身的作用域，每个执行环境还有一个对其外部作用域的引用（可能的话），最终会引用到全局作用域。我们把这一系列的引用就叫做作用域链。

> 一个执行环境的作用域链并不包含任何关于其兄弟作用域的信息（即使是在相同的外部函数内），或者其子作用域。这就是为什么 **a) 你为什么可以从本地作用域访问全局变量，反之则不行； b) 你不能从其他本地作用域访问当前作用域变量。**

下面的例子就显示了上面说的内容：

```js
console.log(one) // undefined
console.log(two)  // ReferenceError: two is not defined
console.log(three) // ReferenceError: three is not defined

var one = 1

function myFunc () {
  console.log(one) // 1
  console.log(two) // undefined
  console.log(three) // ReferenceError: three is not defined
  var two = 2
  console.log(two) // 2
}

function myOtherFunc () {
  console.log(one) // 1
  console.log(two) // ReferenceError: two is not defined
  console.log(three) // undefined
  var three = 3
  console.log(three) // 3
}

myFunc()
myOtherFunc()
```

### this 的值

每个执行环境都有一个特殊的变量 `this`。`this` 指向代表了当前代码在其内执行的那个对象。。

在下面的例子中，全局执行环境中 _this_ 的值是 _window_。而在 _myFunc_ 函数内 _this_ 的值也是 _window_。

```js
var globalThis = this

function myFunc () {  
  console.log('globalThis: ', globalThis)
  console.log('this inside: ', this)
  console.log(globalThis === this)
}

myFunc()

// globalThis: Window {...}
// this inside: Window {...}
// true
```

但是，怎么解释下面这个例子：

```js
var globalThis = this

var myObj = {
  myMethod: function () {    
    console.log('globalThis: ', globalThis)
    console.log('this inside: ', this)
    console.log(globalThis === this)
    console.log(myObj === this)
  }
}

myObj.myMethod()

// globalThis: Window { ... }
// this inside: { myMethod: f }
// false
// true
```

这是因为 _this_ 在有一个执行对象的时候，就指向那个执行对象； 如果没有一个执行对象，那么它就指向全局环境。

### 执行对象

在上面的例子中，当 myMethod() 在第12行执行的时候，其前面有一个对 _myObj_ 的引用————这就是它的执行对象。这样，在 `myMethod()` 的执行环境中，this 就指向了  _myObj_。

那么，你能知道下面例子中的 this 输出是什么么：

```js
var myObj = {
  myMethod: function () {    
    console.log(this)
  }
}

var myFunc = myObj.myMethod
myFunc()

```

输出会是 _Window_，而不是 _myObj_ ， 为什么？因为 **this 只是在执行的时候才会被设置为执行对象**。

那么下一个的输出呢？

```js
var myObj = {
  myMethod: function () {    
    myFunc()
    
    function myFunc () {
      console.log(this)
    }
  }
}

myObj.myMethod()
```

还是 _Window_。

### this 与 self

看起来我们不能对 _this_ 有什么有用的操作，真的是这样么？当在嵌套的函数中的时候，如果 this 指向一个全局对象是很不好的事情，因为调用是在一个对象内进行的。

常规的方法是把 this 赋值给一个变量：

```js
var myObj = {
  myMethod: function () {
    var self = this
    myFunc()
    
    function myFunc () {
      console.log('this: ', this)
      console.log('self: ', self)
    }
  }
}

myObj.myMethod()

// this: Window { ... }
// self: { myMethod: f }
```

### bind(), call(), apply()

另外一种来操作 this 值的方式是调用内建的方法：  `call(), bind(), apply()`。任何一个 JavaScript 函数都可以调用这三个方法。所有他们三个做的事情都一样————以一个对象作为参数，把这个对象作为执行环境的父对象————只有轻微的不同。

- bind() 返回一个函数，不过其 this 也被设置
- call(), apply()：执行调用。

```js
var greet = 'Hello!'

function showGreet () {
  console.log(this.greet)
}

var casualGreet = { greet: 'Hey!' }

showGreet()                    // Hello!
showGreet.bind(casualGreet)()  // Hey!
showGreet.call(casualGreet)    // Hey!
showGreet.apply(casualGreet)   // Hey!
```


## 执行栈

一个简单的规则，调用函数，就会建立执行环境。

那么接下来呢？

假如你的代码中有两个函数。你事实上拥有三个执行环境（包括全局执行环境）。我们需要更深入的了解一下执行栈

每当一个新执行环境建立的时候，其被放在前一个执行环境之上。这就是为什么我们把他们叫做执行栈的意思。

有一个需要了解的地方就是，JavaScript 只能在一个环境执行。

当开始执行代码的时候，是在全局执行环境执行的。

当调用一个函数的时候，将会进入执行环境 _a_，所有在全局环境内的事件都会暂停，直到 JavaScript 引擎退出 _a_。

当在 _a_ 中又进行函数调用的话，那么就会进入一个环境 _b_。此时，会将 _a_ 暂停。

....

上面这个过程就是为什么 JS 被称作是单线程的原因。

# 闭包

下面的例子会打印出什么。


```js
var name = 'John'

function greet (name) {  
  return (function () {
    console.log('Hello ' + name)
  })
}

var sayHello = greet(name)

name = 'Sam'

sayHello()
```

答案是：Hello John。

即使我们在调用  sayHello() 之前改变了 name 的值，输出不会变化。就好像 name 的值，以前在其改变前被捕捉了。

这就是 闭包了。

##　Scope Chain 

在一个执行环境中，包含了一个作用域链，保留了对其外部作用域的引用，直到全局作用域。但事实上我们并不知道作用域链是怎么样建立的。我们需要介绍一个新的角色了————函数的[[scope]]属性。

## [[scope]] Property

每个函数，在**建立**的时候，就被谁知了一个内部的属性 [[scope]]，属性的值就是当前执行环境的作用域链。

当函数被调用的时候，会建立一个新的执行环境，同时新的执行环境的作用域链，此作用域链引用了所有的本地作用域变量。
这个作用域链还会以一个单独的属性来继承 [[scope]] 的值。这就是一个作用域链怎么样引用外部作用域的了。

需要记住的是， [[scope]] 属性是在函数被建立的时候设置的，而不是在其被调用或运行的时候。当函数运行的时候，新的执行环境的作用域链中的变量引用了这个 [[scope]]。


```js
// scope chain: { a: undefined }
var a = 'global'
// scope chain: { a: 'global' }

function outer () {
  // [[scope]]: { a: 'global' }
  
  // scope chain: { b: undefined, outerScope: [[scope]] }  
  var b = 'outer'
  // scope chain: { b: 'outer', outerScope: [[scope]] }
  
  function inner () {
    // [[scope]]: {
    //   b: 'outer',
    //   outerScope: { a: 'global' }
    // }

    // scope chain: { c: undefined, outerScope: [[scope]] }
    var c = 'inner'
    // scope chain: { c: 'inner', outerScope: [[scope]] }
    
    console.log('a:', a)
    console.log('b:', b)
    console.log('c:', c)
  }
  
  inner()
}

outer()

// a: global
// b: outer
// c: inner
```

### 闭包

在上面的例子中，我们来想一下这样的场景：我想在将来的某个时刻执行 inner() 函数，而不是立刻在 outer 内调用？那么在  inner() 的 console.log() 会打印出什么来？

```js
var a = 'global'

function outer () {
  var b = 'outer'
  
  return function inner () {
    var c = 'inner'
    
    console.log('a:', a)
    console.log('b:', b)
    console.log('c:', c)
  }
}

var innerFunc = outer()
innerFunc()

// a: global
// b: outer
// c: inner
```

事实上，对外部变量的引用：

1. 在函数建立时创建
2. 即使在外部执行环境移除后依然可用。

这就叫做闭包。

### 通过引用，而不是值

需要注意，一个闭包捕捉对外部变量的引用，而不是他们的值。其记住的是，在哪里去找到变量，这些变量在不同的时间不同的地方可以代表不同的值。

换句话说，闭包内变量的值可以被修改。


```js
var a = 'global'

function outer () {
  var b = 'outer'
  
  return function inner () {
    var c = 'inner'
    
    console.log('a:', a)
    console.log('b:', b)
    console.log('c:', c)
  }
}

var innerFunc = outer()

a = 'GLOBAL'

innerFunc()

// a: GLOBAL
// b: outer
// c: inner
```

你可能会有疑问，为什么本节开头的那个例子输出的却是 John，而不是 Sam。


这就有所不同了，因为我们的 name 是作为参数传递给 sayHello() 的，其实已经是本地作用域内的变量了，所以你再改变全局变量的值已经不会影响它了。



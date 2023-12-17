---
title: Node中的require与module
categories:
  - JavaScript
date: 2020-03-12 22:31:41
updated: 2020-03-12 22:31:41
tags: 
  - JavaScript
---

我们会经常遇到用 `require` 来引用 JS 中的一个模块，但是却不知道其背后是如何工作的。为什么重复引用的时候会得到一个相同的值呢？

<!--more-->

`require` 是一个函数，而 `module` 是一个对象：

```js
console.log(typeof(module));
console.log(typeof(require));
```

```
object
function
```

这两个都是在全局作用域内可用的，所以任何模块都可以进行使用。

# require

在 [这里](https://github.com/nodejs/node/blob/8828426af45adc29847d0d20717c8fa3eb9bb523/lib/internal/modules/cjs/loader.js#L648-L655) 定义了 `require` 函数：

```js
// Loads a module at the given file path. Returns that module's
// `exports` property.
Module.prototype.require = function(id) {
  validateString(id, 'id');
  if (id === '') {
    throw new ERR_INVALID_ARG_VALUE('id', id,
                                    'must be a non-empty string');
  }
  return Module._load(id, this, /* isMain */ false);
};
```

# Module._load

```js
// Check the cache for the requested file.
// 1. If a module already exists in the cache: return its exports object.
// 2. If the module is native: call `NativeModule.require()` with the
//    filename and return the result.
// 3. Otherwise, create a new module for the file and save it to the cache.
//    Then have it load  the file contents before returning its exports
//    object.
Module._load = function(request, parent, isMain) {
  if (parent) {
    debug('Module._load REQUEST %s parent: %s', request, parent.id);
  }

  var filename = Module._resolveFilename(request, parent, isMain);

  var cachedModule = Module._cache[filename];
  if (cachedModule) {
    updateChildren(parent, cachedModule, true);
    return cachedModule.exports;
  }

  if (NativeModule.nonInternalExists(filename)) {
    debug('load native module %s', request);
    return NativeModule.require(filename);
  }

  // Don't call updateChildren(), Module constructor already does.
  var module = new Module(filename, parent);

  if (isMain) {
    process.mainModule = module;
    module.id = '.';
  }

  Module._cache[filename] = module;

  tryModuleLoad(module, filename);

  return module.exports;
};
```

# Module()

```js
function Module(id, parent) {
  this.id = id;
  this.exports = {};
  this.parent = parent;
  if (parent && parent.children) {
    parent.children.push(this);
  }

  this.filename = null;
  this.loaded = false;
  this.children = [];
}
```

# tryModuleLoad

```js
function tryModuleLoad(module, filename) {
  var threw = true;
  try {
    module.load(filename);
    threw = false;
  } finally {
    if (threw) {
      delete Module._cache[filename];
    }
  }
}
```

# module.load

```js
Module.prototype.load = function(filename) {
  debug('load ' + JSON.stringify(filename) +
        ' for module ' + JSON.stringify(this.id));

  assert(!this.loaded);
  this.filename = filename;
  this.paths = Module._nodeModulePaths(path.dirname(filename));

  var extension = path.extname(filename) || '.js';
  if (!Module._extensions[extension]) extension = '.js';
  Module._extensions[extension](this, filename);
  this.loaded = true;
};
```

# Module._extensions[

```js
Module._extensions['.js'] = function(module, filename) {
  var content = fs.readFileSync(filename, 'utf8');
  module._compile(stripBOM(content), filename);
};
```



# 步骤

实际上，加载一个模块会经历几个步骤：

1. **解析**文件路径。`Module._resolveFilename(request, parent, isMain)`

2. 建立 *Module* 的实例。`var module = new Module(filename, parent);` 并缓存

3. **加载**文件内容。`tryModuleLoad(module, filename);`

   

# 关键在于

首先会建立了一个 Module 实例，然后将用此实例去加载代码，因此在一个模块中，是可以使用  `module`变量的。而为 `module.exports` 设置的内容，就是要导出的内容。

# 需要注意的是

`exports` 实际上是对 `module.exports` 的引用。因此将其换成一个其他的对象，是不会产生导出效果的。

对象是通过引用传递的，而原生类型，则是通过值传递的。



# 例子

我们知道，在 JavaScript 中，如果是原生（值）类型（`Number, Boolean,String, Null, Undefined, Symbol`)，是以值的形式传递的；而对于引用数据类型（`Object, Array, Function`）传递的可以说类似与 C 中的指针（内存地址）。

因此，在我们上面的描述中，对于一个模块，首先会建立一个  *Module* 对象，然后在执行我们定义的模块中的代码的时候，会把这个对象传递过去，由其来修改此 *Module* 对象中的 `module.export` 对象中的值。

因此，如果我们多次引用一个对象，因为缓存的缘故，获取的应该是同一个值。同时，在模块内使用的变量 `exports` 实际上也是对 `module.exports` 的引用。

## 模块会被缓存

假设当前我们有一个模块 *a.js*

```js
// a.js
var v = 1
module.exports.a = v
```

有一个测试文件 *t.js*

```js
//t.js
let t = require('./a')
console.log(t)
t.a = 99
console.log(require('./a').a)
```

输出：

```
{ a: 1 }
99
```

## 模块变量的隐藏

我们知道几个逻辑，对于一个变量，

1. 如果没有使用 `let, var` 声明那他就是一个全局作用域变量，而无论其在何处进行赋值的。如上面我们的 *v* ，如果去掉 `var` 修饰，那么在 *t.js* 就能访问。
2. 如果在函数内使用 `var, let` 声明的，那么就只能在函数内引用。
3. 在函数外使用  `var, let`生命的，在当前模块内可以引用。这是因为，实际上模块在执行的时候，是被放在一个包装函数里面执行的。也相当于是在一个函数内进行定义了。



## 立即执行函数

我们常看到如此定义的模块：

```js
(function(){})()
```

事实上这个语义，可以这样理解：

1. 定义了一个函数  `function(){}`
2. 外面的括号是为了和后面的括号区分开，表示这是一个控制运算符优先级的括号。
3. 最后的那对括号，才表示调用函数的意思。

这样的函数，就将其内部要执行的代码，隐藏起来了，而在函数内部，再通过  `module.exports` 来导出其要暴露的内容。
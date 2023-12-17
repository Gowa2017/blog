---
title: Vue开发使用ESLint-Vuter-Prettier来进行代码格式化与检查
categories:
  - JavaScript
date: 2019-02-22 19:53:38
updated: 2019-02-22 19:53:38
tags: 
  - JavaScript
  - Vue
---
某个项目需要用到。但是代码风格不一致的问题，和 eslint 老报错的问题很是困扰了哦，网络上找的文章一般都是直接贴配置文件，很坑，总是不能达到目的，所以来看一下官方文档。

<!--more-->

# 描述
首先看一下三个东西的描述都是怎么样的：

- [**Vetur**](https://vuejs.github.io/vetur/)：一个专门针对Vue开发的 VS code 插件。
- [**ESlint**](https://eslint.org)： 插件式JavaScript和JSX检查工具
- [**Prettier**](https://prettier.io) 代码格式化工具

我们就先明确我们的目标，使用  Vetur 来进行 Vue 的开发，同时给 Vetur 配置好代码格式化组件，接着，使用 ESLint 来检查代码。

Vetur 包含了针对 vue 的很多功能，比如说代码提示，补全等等，格式化只是其中的一个功能。我们重点关注一下格式化的功能。

# Vuter

VS Code 插件安装很简单，直接搜索安装就行了。接着我们就在工程（或者全局）的 settings.json 内进行配置。

[安装文档](https://vuejs.github.io/vetur/setup.html#extensions)

## 格式化

根据官方文档  [Formatting](https://vuejs.github.io/vetur/formatting.html#formatters)，进行配置。

可用的格式化工具有：

* **prettier**: For css/scss/less/js/ts.
* **prettier-eslint**: For js. Run prettier and eslint --fix.
* **prettyhtml**: For html.
* **stylus-supremacy**: For stylus.
* **vscode-typescript**: For js/ts. The same js/ts formatter for VS Code.

当前默认的 Formatter 如下：

```js
{
  "vetur.format.defaultFormatter.html": "prettyhtml",
  "vetur.format.defaultFormatter.css": "prettier",
  "vetur.format.defaultFormatter.postcss": "prettier",
  "vetur.format.defaultFormatter.scss": "prettier",
  "vetur.format.defaultFormatter.less": "prettier",
  "vetur.format.defaultFormatter.stylus": "stylus-supremacy",
  "vetur.format.defaultFormatter.js": "prettier",
  "vetur.format.defaultFormatter.ts": "prettier"
}
```

> **Vetur bundles all the above formatters. When Vetur observes a local install of the formattesr, it'll prefer to use the local version.**
> Vetur 已经打包了所有的格式化工具。如果本地有安装一个版本的话，会优先调用本地的版本。

这里我们什么都不用改，默认使用  prettier 来格式化就行。

根据官方文档的说明， Vetur 已经打包了所有可用的格式化工具。

# Prettier（可不安装，Vetur 已打包）


可以通过自定义快捷键来触发格式化文档：

- editor.action.formatDocument
- editor.action.formatSelection

**开启保存时格式化**

```json
{
	"editor.formatOnSave": true
}
```

还可以针对特定的语言才开启：

```json
{
	// Set the default
	"editor.formatOnSave": false,
	// Enable per-language
	"[javascript]": {
    "editor.formatOnSave": true
	}
}
```

## 配置
对于 prettier 的配置方式有多种：

1. .prettierrc 文件。
2. 直接在工程中的 settings.json 内配置。
3. 在 Home 目录下写一个 .prettierrc 文件。

.prettierrc 文件的优先级高于 settings.json 的设置。


配置文件会从两个地方读取：

1. 从  prettier 配置文件读取，文件查找顺序查看 [Configuration File](https://prettier.io/docs/en/configuration.html)
2. .editorconfig

如果不存在 prettier 的配置文件，那么就从 settings.json 读取。

- prettier.printWidth (default: 80)
- prettier.tabWidth (default: 2)
- prettier.singleQuote (default: false)
- prettier.trailingComma (default: 'none')
- prettier.bracketSpacing (default: true)
- prettier.jsxBracketSameLine (default: false)
- prettier.parser (default: 'babylon') - JavaScript only
- prettier.semi (default: true)
- prettier.useTabs (default: false)
- prettier.proseWrap (default: 'preserve')
- prettier.arrowParens (default: 'avoid')
- prettier.jsxSingleQuote (default: false)
- prettier.htmlWhitespaceSensitivity (default: 'css')
- prettier.endOfLine (default: 'auto')
- prettier.eslintIntegration (default: false) - JavaScript and TypeScript only 控制是否 prettier-eslint i 来替代 prettier。
- prettier.tslintIntegration (default: false) - JavaScript and TypeScript only
- prettier.stylelintIntegration (default: false) - CSS, SCSS and LESS only
- prettier.requireConfig (default: false)
- prettier.ignorePath (default: .prettierignore)
- prettier.disableLanguages (default: ["vue"])

我觉得我应该开启的选项有：

```js
{
	"prettier.semi": false,
	"prettier.singleQuote": true,
	"prettier.eslintIntegration": true
	"prettier.disableLanguages": []
	
}
```

不知道为什么会 vue 给关掉呢。

# ESLint

ESLint 用来做代码检查，经过上面的步骤，我们代码是按照 prettier 的格式来进行格式化的。但 ESLint 的检查规则与其有不一样的地方。介于我们不想有冲突产生，所以我们有两种办法来将 prettier 与 ESLint 集成。

我们首先开启 ESLint：

```js
{
	"eslint.enable": true,
	"eslint.autoFixOnSave": true,
	"eslint.validate": [ "javascript", "javascriptreact", { "language": "vue", "autoFix": true } ]
}
```
eslint.autoFixOnSave 选项会配置自动修复检查到的问题。

## 问题

ESLint 除了代码检查外，还会做一些代码格式化的功能。我们可以：

1. 关闭 ESLint 的格式化规则，按照 prettier 的来进行格式化。
2. ESLint 按照 prettier 的规则来进行格式化。

## ESLint 运行 Prettier

安装插件 https://github.com/prettier/eslint-plugin-prettier


```bash
yarn add --dev prettier eslint-plugin-prettier
```

配置 .eslintrc.js


```js
{
  "plugins": ["prettier"],
  "rules": {
    "prettier/prettier": "error"
  }
}
```
## 关闭 ESLint 格式化规则

安装  https://github.com/prettier/eslint-config-prettier

```bash
yarn add --dev eslint-config-prettier
```

编辑  .eslintrc.js

```js
{
  "extends": ["prettier"]
}
```

## 同时使用以上两种方法

`eslint-plugin-prettier` 有一个 `recommended ` 配置，其开启了 `eslint-plugin-prettier` 和 `eslint-config-prettier`。

```js
{
  "extends": ["plugin:prettier/recommended"]
}
```

记住要先安装两个插件：

```bash
yarn add --dev eslint-plugin-prettier eslint-config-prettier
```

# 最终解决方案

- 安装 Vetur 插件
- 安装 ESLint 插件

## VS Code settings 配置：

```js
{
  "editor.formatOnSave": true, // 保存时格式化
  "javascript.format.enable": false, // 关闭自带的格式化
  "eslint.enable": true, // 启用eslint
  "eslint.autoFixOnSave": true, // 保存时自动修复
  "eslint.validate": [
    "javascript",
    "javascriptreact",
    "vue"
  ], // eslint 识别格式
  "vetur.format.defaultFormatterOptions": {
    "prettier": {
      "singleQuote": true,
      "semi": false
    }, // vetur 调用 prettier 的配置
  }
}
```

## .eslintrc.js

```js
// https://eslint.org/docs/user-guide/configuring

module.exports = {
  root: true,
  parserOptions: {
    parser: 'babel-eslint'
  },
  env: {
    browser: true
  },
  // https://github.com/vuejs/eslint-plugin-vue#priority-a-essential-error-prevention
  // consider switching to `plugin:vue/strongly-recommended` or `plugin:vue/recommended` for stricter rules.
  extends: ['plugin:vue/essential'],
  // required to lint *.vue files
  plugins: ['vue',],
  // add your custom rules here
  rules: {
    // allow debugger during development
    'no-debugger': process.env.NODE_ENV === 'production' ? 'error' : 'off',
    semi: ['error', 'never'], // 取消分号
    quotes: ['error', 'single'] // 使用单引号
  }
}

```


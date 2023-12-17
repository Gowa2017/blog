---
title: pandoc一个比较经典的latex模板
categories:
  - macOS
date: 2020-10-10 22:41:06
updated: 2020-10-10 22:41:06
tags: 
  - macOS
  - pandoc
---

pandoc 确实是神器啊。加上对 docx 的支持也很棒了，现在来仔细阅读一番 [Eisvogel](https://github.com/Wandmalfarbe/pandoc-latex-template) latex 模板的使用了。比 docx  的操作更加的灵活。关于 pandoc 的基本相关知识见 {% post_link 使用pandoc转换pdf与docx加上书签目录 使用pandoc转换pdf与docx加上书签目录 %}

<!--more-->

# 基本使用

```sh
pandoc example.md -o example.pdf --from markdown --template eisvogel --listings
```

> 注意，这里的 `--template eisvogel` 表示使用 eisvogel 这个模板的意思。这样使用，必须保证 eisvogel 这个模板放在 `data-dir` 下面。
>
> 想知道 data-dir 是在什么位置，可以使用命令
>
> pandoc -v 
>
> 进行查看。
>
> 比如我用 brew 在 macOS 安装的 pandoc
>
> *Default user data directory: /Users/username/.local/share/pandoc or /Users/username/.pandoc* 就是在这里了。

因此我们只需要将 eisvogel.latex 放到这个目录下面就可以保证上面的命令会执行成功了。

# 页眉页脚

如果想要更好看规范的页眉和页脚的话，我们需要提供一些元数据。通过在 *md* 文件中，设置一个 [YAML metadata block](http://pandoc.org/MANUAL.html#extension-yaml_metadata_block) 在开始处。我们的文件可能看起来是这样的：

```
---
title: "The Document Title"
author: [Example Author, Another Author]
date: "2017-02-20"
keywords: [Markdown, Example]
...

Here is the actual document text...
```

## 自定义模板变量

以下内容中，括号里面的代表了默认值。

## 封面设置

- `titlepage`(`false `) 是否启用标题页(封面页)
- `titlepage-color` (`FFFFFF`)封面页的背景颜色。这个值必须设置成 HTML 16进制的颜色值，和 `D8DE2C` 这样，没有开头的 `#`。同时，建议加上双引号（避免颜色被截断，如 `000000` 会被识别成 `0`）。
- `titlepage-text-color` ( `5F5F5F`) 封面页的文本颜色
- `titlepage-rule-color` ( `435488`) 封面页顶部的横条颜色
- `titlepage-rule-height` ( `4`) 封面横条的高度。（单位是 磅）
- `titlepage-background` 封面的背景图片路径。图片会被缩放来覆盖整个页面。
- `logo` 在封面页显示的图片路径。路径总是相对于 pandoc 被执行的位置。选项 `--resource-path` 没有作用。
- `logo-width` ( `100`) logo 的宽度（磅）

## 页眉页脚

- `header-left` (`标题`) 页眉左部的文本
- `header-center` 页眉中间的文本
- `header-right` (`日期`) 页眉右部的文本
- `footer-left` (`作者`) 页脚左部文本
- `footer-center`页脚中部文本
- `footer-right` (`页码`) 页脚右部文本

## 页面设置

- `page-background` 所有页面的背景图片。会被缩放。
- `page-background-opacity` ( `0.2`) 背景图片透明度
- `caption-justification` ( `raggedright`) 图片表格等题注的对齐方式。（使用  [caption](https://ctan.org/pkg/caption?lang=en) 包的 `justification` 参数)
- `toc-own-page` ( `false`) 是否在目录后开始一个新的页面。（传递 `--toc` 选项到 pandoc 命令）
- `listings-disable-line-numbers` ( `false`) 禁用列表的序号
- `listings-no-page-break` ( `false`) 列表内避免分页符
- `disable-header-and-footer` (false`) 禁止页眉页脚
- `footnotes-pretty` (`false`) 脚注美化（需要 `footmisc` 包）
- `footnotes-disable-backlinks` ( `false`) 禁止从页尾的脚注到内容中出现脚注位置的引用跳转。（启用的话需要包 `footnotebackref`）
- `book` ( `false`) typeset as book
- `first-chapter` (  `1`) 如果类型设置是有章节号的 book，设置第一章被赋予的值
- `float-placement-figure` (`H`) 重置图片的默认位置。可用的修饰符如下，前四个修饰符可以联合起来用：
  1. `h`:  *here*, i.e., approximately at the same point it occurs in the source text.
  2. `t`:  *top* of the page.
  3. `b`:  *bottom* of the page.
  4. `p`:  next *page* that will contain only floats like figures and tables.
  5. `H`: Place the float *HERE* (exactly where it occurs in the source text). The `H` specifier is provided by the [float package](https://ctan.org/pkg/float) and may not be used in conjunction with any other placement specifiers.
- `table-use-row-colors` ( `false`) 启用表格行的颜色。默认是 `false`，因为颜色会扩展到表格的边缘，当前没有任何办法来进行改变。
- `code-block-font-size` ( `\small`) 用来改变代码块内字体大小的 LaTeX 命令。可用的值是： `\tiny`, `\scriptsize`, `\footnotesize`, `\small`, `\normalsize`, `\large`, `\Large`, `\LARGE`, `\huge` and `\Huge`. 这个选项会改变使用 verbatim 环境的默认代码块和有列表的代码块的字体大小。

# Latex 基本概念

1. **控制序列(control sequence)**：一组由`\`开始的命令。有两种类型的控
   制序列：`\`之后紧随一个多个单词组成的*控制字(control word)*和`\`之后紧
   随一个非字母组成的*控制符(control symbol)*。

2. **环境(environment)**：LaTeX中的环境主要用于对一块数据进行排版。其格
   式如下：

   ```
   \begin{environment_name}
   The content ...
   \end{environment_name}
   ```

   
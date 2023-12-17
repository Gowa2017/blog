---
title: 使用pandoc转换pdf与docx加上书签目录
categories:
  - macOS
date: 2019-12-31 22:28:56
updated: 2019-12-31 22:28:56
tags: 
  - macOS
  - Pandoc
---
在文章 {% post_link 几种绘图语法的比较 几种绘图语法的比较 %} 中，我说了我想要的一个 markdown 编辑器， typora 实际上非常优秀，但不开源。 Macdown 非常不错，开源。通过将其 mermaid 进行升级，viz.js 进行升级后，感觉非常不错了，唯一还有一点，就是导出 pdf 没有 Toc，而只能在页内设置 Toc，所以来研究一下用 pandoc 来进行转换看看效果如何。

<!--more-->

# pandoc

[pandoc 网站](https://pandoc.org/) 一句话介绍：

> 如果你需要在一个标记文件格式到另外一种标记文件格式间进行转换，那么 pandoc 就是你的瑞士军刀。

其可以在很多种文件间相互转换。

我最在意的是从 md 到 pdf 或者 md 到 word 的转换。

Typora 据说其转换是会将我们的代码转换成一自己专有的中间格式，进行导出。当然，其除了 pdf 和 html 外的导出是通过 pandoc 来实现的，其导出为 pdf 的效果实在是太棒了。

# 抽象语法树

我们可以用命令来生成一个抽象语法树的 JSON 表示：

```
pandoc -t json <input file>
```

如我的测试文件

```markdown
# header
你好
​```mermaid
graph TB;
a --> b;
\`\`\`
```

输出：

```
{
   "pandoc-api-version" : [
      1,
      20
   ],
   "meta" : {},
   "blocks" : [
      {
         "c" : [
            1,
            [
               "header",
               [],
               []
            ],
            [
               {
                  "c" : "header",
                  "t" : "Str"
               }
            ]
         ],
         "t" : "Header"
      },
      {
         "c" : [
            {
               "t" : "Str",
               "c" : "你好"
            }
         ],
         "t" : "Para"
      },
      {
         "c" : [
            [
               "",
               [
                  "mermaid"
               ],
               []
            ],
            "graph TB;\na --> b;"
         ],
         "t" : "CodeBlock"
      }
   ]
}
```

一个 pandoc 的 AST 包含一个 *meta* 块（包含如标题，作者，日期）等的元数据及一个由 *Block* 元素组成的列表。

在我们的例子中，有三个 *Block* 元素:**Header, Str, CodeBlock**。每个元素都有一个内容列表（由 Inline 元素组成）。

简单看一下 CodeBlock 的在 AST 内的组成，其包括两部分：

- Attr 包括三个参数：(identifier, [classes],[(key,value)]) 分别是标识符，类列表，k-v键值对列表
- Text 就是代码本身。事实pandoc 是进行了封装的 Unicode Text 字节

在我们的 md 文件中，将代码的类型，标注成了 classes 。

参考用 python 的一个 filter ，调用外部的 mermaid -cli 来进行渲染：

[pandoc-mermaid-filter](https://github.com/timofurrer/pandoc-mermaid-filter/blob/master/pandoc_mermaid_filter.py)

# 基本语法

```
pandoc -s -f gfm -t pdf -o outputfile
```

- -f FORMAT, -r FORMAT, --from=FORMAT, --read=FORMAT  输入文件格式
- -t FORMAT, -w FORMAT, --to=FORMAT, --write=FORMAT 输出文件格式
- -o 输出文件
- -s, --standalone 增加页眉和页脚。`pdf`, `epub`, `epub3`, `fb2`, `docx`, 格式会自动设置此选项。

# 建立 PDF

最简单的代码就是：

```
pandoc test.txt -o test.pdf
```

pandoc 默认使用 **LaTeX** 来建立 PDF 文件，这就要求我们首先安装 latex 引擎。当然，其也可以使用 **ConText, roff ms, HTML 来作为中间格式**。需要中间格式的时候，我们需要为输出文件设置一个 .pdf 扩展，然后添加 --pdf--engine 选项或者 `-t context, -t html 或者 -t ms`。用来生成中间文件的工具通过 `--pdf-engine` 来进行指定。

```sh
pandoc -V 'CJKmainfont=Songti TC' -V mainfont=Menlo --from gfm --listings --pdf-engine=xelatex 
```



可以通过变量控制 PDF 的风格，这依赖于我们使用的中间文件格式：查看 [variables for LaTeX](https://pandoc.org/MANUAL.html#variables-for-latex), [variables for ConTeXt](https://pandoc.org/MANUAL.html#variables-for-context), [variables for `wkhtmltopdf`](https://pandoc.org/MANUAL.html#variables-for-wkhtmltopdf), [variables for ms](https://pandoc.org/MANUAL.html#variables-for-ms). 。当我们使用  HTML 作为中间格式的时候，其输出可以用 `--css` 来控制风格

如果要调试 PDF 的生成，我们可以通过查看其中间表示：不使用 `-o test.pdf` 我们使用 `-s -o test.tex` 来生成 LaTex。然后用 `pdflatex test.tex`来进行测试。

当使用  LaTex 的时候，下面这些包必须可用（这些基本都包含在活跃的 Tex 版本中）： amsfonts, amsmath, lm, unicode-math, ifxetex, ifluatex, listings (如果使用 *--listings* 选项), fancyvrb, longtable, booktabs, graphicx (如果文档包含图片), hyperref, xcolor, ulem, geometry (geometry 变量已设置), setspace (与 *linestretch* 一起), and babel (with *lang*). 

`xelatex` or `lualatex` 引擎需要 [`fontspec`](https://ctan.org/pkg/fontspec). `xelatex` 使用 [`polyglossia`](https://ctan.org/pkg/polyglossia) (with `lang`), [`xecjk`](https://ctan.org/pkg/xecjk), and [`bidi`](https://ctan.org/pkg/bidi) (with the `dir` variable set). 

如果设置了 *mathspec* 变量，*xelatex* 会使用 *mathspec* 而不是 *unicode-math*。

*upquote* 和 *microtype* 包可用的话就会被使用，当 *csquotes* 被设置为 true 或者元数据字段被设置为 true 时，*csquotes* 会因为 *typography* 而使用。

下面这些包在存在的时候会用来提高输出的质量，但 pandoc 并不要求他们一定要存在： *upquote* (在逐字环境中使用直接引号), *microtype* (更好的间隔控制), *parskip* (更好的段间距控制), *xurl* (为了更好的URLs换行), *bookmark* (为更好的 PDF 书签), and *footnotehyper* or *footnote* (为了允许表中的脚注).

## --pdf-engine

有多个 pdf 引擎：

**pdflatex, lualatex, xelatex, latexmk, tectonic, wkhtmltopdf, weasyprint, prince, context, and pdfroff**

如果引擎不在我们的路径变量中，那么就需要指定完整路径。如果没有指定这个选项， pandoc 会根据输出来决定使用哪一个默认的引擎：

- -t latex or none: pdflatex (other options: xelatex, lualatex, tectonic, latexmk)
- -t context: context
- -t html: wkhtmltopdf (other options: prince, weasyprint)
- -t ms: pdfroff

## --toc, --table-of-contents

包含自动生成的 Toc（或者，`latex`, `context`, `docx`, `odt`, `opendocument`, `rst`, or `ms`, 情况下有指令需要生成）。这个选项必须配合 `-s/--standalone` 使用才有效，其在 `man, docbook4, docbook5, jats` 输出中无效。

如果我们使用  `ms` 来生成 PDF，TOC 会出现在文档标题的前面，我们可以用 `--pdf-engine-opt==--no-toc-relocation` 来让其在文档后面。

## --toc-depth=NUMBER

指定要包含在 TOC 中的节等级。默认是3.

## mactex

用 brew 已经找不到包了。所以我们可以安装 [macTex](https://www.tug.org/mactex/aboutmactex.html)，不过这玩意比较大。所以 pandoc 官方给了一个建议：

> 默认情况下 pandoc 使用LaTeX 来生成 PDF 。 因为完整的 [MacTeX](https://tug.org/mactex/) 会使用 4GB 的磁盘空间，我们建议使用   [BasicTeX](http://www.tug.org/mactex/morepackages.html) or [TinyTeX](https://yihui.name/tinytex/) 同时使用 `tlmgr` 来根据需要安装其他包. 如果我们收到警告说字体不存在，我们可以：
>
> ```
> tlmgr install collection-fontsrecommended
> ```
>
> 

## BasicTex

直接 brew 安装：

```
brew cask install basictex
```

安装后的目录在 

```
/usr/local/texlive/2019basic
```

之后我们很多命令到能用了比如：pdflatex, xelatex, luatex 我们来试试。

先装两个依赖：

```sh
tlmgr  install titling lastpage
```

## 中文字体

使用命令 `fc-list :lang=zh`(fontconfig 包) 来查看有哪些中文字体:

```sh
System/Library/Assets/com_apple_MobileAsset_Font5/b2d7b382c0fbaa5777103242eb048983c40fb807.asset/AssetData/Kaiti.ttc: Kaiti TC,楷體\-繁,楷体\-繁:style=Bold,粗體,粗体
/System/Library/Assets/com_apple_MobileAsset_Font5/1183acef85eb1efe456a14378a2eb985c09768c9.asset/AssetData/Lantinghei.ttc: Lantinghei TC,蘭亭黑\-繁,兰亭黑\-繁:style=Extralight,纖黑,纤黑
/System/Library/Assets/com_apple_MobileAsset_Font5/940db29a0ab220999d9a1dbe3eb0819a718057b5.asset/AssetData/Libian.ttc: Libian SC,隸變\-簡,隶变\-简:style=Regular,標準體,常规体
/System/Library/Fonts/STHeiti Medium.ttc: Heiti SC,黑體\-簡,黒体\-簡,Heiti\-간체,黑体\-简:style=中黑,Medium,Halbfett,Normaali,Moyen,Medio,ミディアム,중간체,Médio,Средний,Normal,中等,Media
/System/Library/Assets/com_apple_MobileAsset_Font5/db09870736c6892b6a56035428f2b1b6d0a954fd.asset/AssetData/WawaTC-Regular.otf: Wawati TC,娃娃體\-繁,娃娃体\-繁:style=Regular,標準體,常规体
/System/Library/Assets/com_apple_MobileAsset_Font5/ce85149bd68e9f8b
```

## 使用示例

```
pandoc  年终总结.md -o srs.pdf --pdf-engine=xelatex -V CJKmainfont='Heiti SC'
```

**我看网上大多的示例都是使用的是 mainfont 结果出错，非得用 CJKmainfont 才行，真是很坑**

这是因为，网上使用的模板，与默认的模板不同，默认的模板位于 pandoc 目录下，比如我用 brew 安装的 pandoc 其模板位于：

```sh
/usr/local/Cellar/pandoc/2.8.1/share/x86_64-osx-ghc-8.8.1/pandoc-2.8.1/data/templates
```

下面，其中使用的就是 CJKmainfont 这个变量来设置字体的。

至此，如何将 md 转换为 pdf 就已经是完成了。但遗留的问题就是：

**对于我 md 里面使用的 graphviz , mermaid 图表，如何才能给我在 PDF 中转换出来呢？**

## 模板

当使用 `-s/--standalone` 选项的时候，pandoc 会在自表示的文档在中，在需要时使用一个模板来**添加页眉和页脚**。如果要查看默认的模板，键入：

```sh
pandoc -D *FORMAT*
```

- FORMAT 输出文档的格式。

例如 

```sh
pandoc -D latex
```

我们可以使用  `--template` 来指定一个自定义的模板，或者，我们可以在系统的目录中对默认模板进行替换（将文件 `templates/default.*FORMAT*`放在用户的数据目录（通过命令 `pandoc --version `来查看）。（关于系统默认模板的目录位置，我使用 brew 安装的话是位于：`/usr/local/Cellar/pandoc/2.8.1/share/x86_64-osx-ghc-8.8.1/pandoc-2.8.1/data/templates` 下面：）

但是有几个例外：

- odt 自定义 default.opendocument 模板
- pdf 自定义 defaut.latex 模板（或 在使用 -t context 时修改 default.context ，使用 ms 的时候自定义 default.ms，或在使用 -t html 的时候定义  -t html ）
- docx pptx 没有模板。他们叫做参考文件。主要是参考 word 文件中的样式来进行设置格式。

模板会包含变量，我们可以通过命令行的 `-V/--variable` 来进行设置。如果一个变量没有设置，那么就会在文档的元数据内进行搜索，文档的元数据可以用 YAML 或者是 `-M/--metadata` 来设置。有些变量会被 pandoc 赋予默认值。我们可以在变量一节进行查看。

## latex 模板语法

### 注释

以 `$--` 开始的行都是注释

### 分隔符

在模板中，我们可以使用  `$...$` 或者  `${...}` 作为分隔符来标识变量和控制结构。两种风格可以混用，但是开始和结尾必须一致。开始的分隔符可能会跟随空白符或 Tab，这些会被忽略。

# PDF幻灯片

在转换 PDF 的时候加上 `-t beamer` 就行了。

# 变量

## 元数据变量

- `title, author, date` 文档的基本标识信息。通过 LaTex 和 ConTeXt 来包含在 PDF 元数据中。可以通过 [pandoc title block ](https://www.pandoc.org/MANUAL.html#extension-pandoc_title_block) 或者一个 YAML 的元数据块来实现。

```yaml
---
author:
- Aristotle
- Peter Abelard
...
```

注意：如果我们只是想设置 PDF 或 HTML 的元数据，我们可以不用在文档中包含这样的块，而只需要设置 `title-meta`, `author-meta`, 和 `date-meta` 变量就行了(默认情况下，这几个变量的默认值是通过 author, title, date 自动设置的)

- `subtitle` HTML, EPUB, LaTeX, ConTeXt, and docx 中会用到的子标题
- `abstract` LaTeX, ConTeXt, AsciiDoc, and docx 文档中的摘要
- `keywords` HTML, PDF, ODT, pptx, docx and AsciiDoc 文档中的关键词
- `subject` ODT, PDF, docx and pptx  中的科目
- `description` ODT, docx and pptx metadata. 中的描述
- `category` docx and pptx  文档分类

## 语言变量

- `lang` 一个BCP 47 的语言标识
- `dir` 文字方向。 rtl 或者是 ltr

## Beamer slides 变量

这些变量改变一个使用 `beamer`  的 PDF 幻灯片的外观。

- `aspectratio` 比例（43 for 4:3 [default], 169 for 16:9, 1610 for 16:10, 149 for 14:9, 141 for 1.41:1, 54 for 5:4, 32 for 3:2）
- `beamerarticle` 从 Beamer slides 产生一个文章
- `beameroption` 通过 `\setbeameroption{}` 来添加额外的 beamer 选项。
- `institute` 作者附属信息；多个作者时可以是一个列表
- `logo`幻灯片的LOGO
- `navigation` 控制导航符号（没有导航符号就是空；其他值是 frame, vertical, horizontal）
- `section-titles` 对新的节启用一个新的页。默认开启。
- `theme, colortheme, fonttheme, innertheme, outertheme`主题
- `themeoptions`  LaTeX beamer themes 选项（一个列表）
- `titlegraphic` 标题幻灯片的图片

## latex 变量

### 布局

- `block-headings` make `\paragraph` and `\subparagraph` (fourth- and fifth-level headings, or fifth- and sixth-level with book classes) free-standing rather than run-in; requires further formatting to distinguish from `\subsubsection` (third- or fourth-level headings). Instead of using this option, [KOMA-Script](https://ctan.org/pkg/koma-script) can adjust headings more extensively:

  ```
  ---
  documentclass: scrartcl
  header-includes: |
    \RedeclareSectionCommand[
      beforeskip=-10pt plus -2pt minus -1pt,
      afterskip=1sp plus -1sp minus 1sp,
      font=\normalfont\itshape]{paragraph}
    \RedeclareSectionCommand[
      beforeskip=-10pt plus -2pt minus -1pt,
      afterskip=1sp plus -1sp minus 1sp,
      font=\normalfont\scshape,
      indent=0pt]{subparagraph}
  ...
  ```

- `classoption` 文档类 class 选项。如：`oneside`，多个选项进行重复就行

  ```
  ---
  classoption:
  - twocolumn
  - landscape
  ...
  ```

- `documentclass` 通常是 `book, article, report` 之一；the [KOMA-Script](https://ctan.org/pkg/koma-script) equivalents, `scrartcl`, `scrbook`, and `scrreprt`, which default to smaller margins; or [`memoir`](https://ctan.org/pkg/memoir)

- `geometry` [geometry](https://ctan.org/pkg/geometry) 包的选项。

  ```
  ---
  geometry:
  - top=30mm
  - left=20mm
  - heightrounded
  ...
  ```

- `hyperrefoptions` [hyperref](https://ctan.org/pkg/hyperref) 包的选项。如:`linktoc=all`

  ```
  ---
  hyperrefoptions:
  - linktoc=all
  - pdfwindowui
  - pdfpagemode=FullScreen
  ...
  ```

- `indent` 使用文档类的缩进设置（默认的 LaTeX 模板会移除缩进，并在段落间添加空白）

- `linestretch` 使用 `[`setspace`](https://ctan.org/pkg/setspace)` 报设置行间距。如：1.25，1.5

- `margin-left,margin-right,marigin-top,margin-bottom` 如果没有使用的 `geometry` 的话，那么就使用这些设置。

- `pagestyle` 控制 `\pagestyle{}`：默认的文章类支持 `plain`（默认），`empty`（没有页眉和页码）和 `headings`（在页眉有节标题）

- `papersize` 如 `letter, a4`

- `secnumdepth` 节的深度（需要传递 `--number-sections/-N`，或者设置 `numbersections` 变量）

### 字体

- `fontenc` 通过 fontenc 包来指定字体的编码（`pdflatex`）；默认是 T1，参考[LaTeX font encodings guide](https://ctan.org/pkg/encguide)

- `fontfamily` pdflatex 中要使用的字体包。[TeX Live](https://www.tug.org/texlive/) 包含了很多的选项, 文档参考 [LaTeX Font Catalogue](https://tug.org/FontCatalogue/). 默认是 Latin Modern](https://ctan.org/pkg/lm).

- `fontfamilyoptions` 用做 `fontfamily` 的包的选项

  ```
  ---
  fontfamily: libertinus
  fontfamilyoptions:
  - osf
  - p
  ...
  ```

- `fontsize` 字体正文大小。标准的值有：`10pt, 11pt, 12pt`。要使用其他尺寸，将 `documentclass` 设置成[KOMA-Script](https://ctan.org/pkg/koma-script) 类，如`scrartcl` 或者 `scrbook`。

- `mainfont,sansfont,monofont,mathfont,CJKmainfont` xelatex 和 lualatex 使用的字体：任何系统字体的名字，通过 `fontspec` 包来实现。`CJKmainfont` 使用  `xecjk` 包。

- `mainfontoptions, sansfontoptions, monofontoptions, mathfontoptions, CJKoptions` 上述字体在的选项。

  ```
  ---
  mainfont: TeX Gyre Pagella
  mainfontoptions:
  - Numbers=Lowercase
  - Numbers=Proportional
  ...
  ```

- `microtypeoptions` 传递给 microtype 包的选项。

### 链接

- `colorlinks` 连接文本加色；如果 ` linkcolor, filecolor, citecolor, urlcolor, or toccolor ` 中任意一个被设置，那么这个变量会自动设置。
- `linkcolor, filecolor, citecolor, urlcolor, toccolor` 也是链接颜色，不过针对的是内部链接、外部链接、引用链接、URLS、到TOC的链接。
- `links-as-notes` 链接打印为脚注

## 扉页Front matter

- `lof, lot` 包含图片和表格的列表
- `thanks` 文档标题下的的一些脚注。
- `toc` 目录。也可通过 `--toc/--table-of-contents`。
- `toc-depth` 需要包含节的深度。

## BibLaTeX Bibliographies

当用 BibLaTeX 来进行文献引用渲染时生效。

- `biblatexoptions` biblatex 的选项列表
- `biblio-style` bibliography 风格, when used with --natbib and --biblatex.
- `biblio-title` bibliography 标题, when used with --natbib and --biblatex.
- `bibliography` bibliography to use for resolving references
- `natbiboptions list of options for natbib

# raw attribute

这是一个扩展：`raw_attribute`

在一些代码块内，我们用特定的形式进行操作的话，那么将会被识别为 raw 内容。

```
​```{=openxml}
<w:p>
  <w:r>
    <w:br w:type="page"/>
  </w:r>
</w:p>
​```
```

上面的代码会在 docx 里面插入一个分页符。

openxml 必须和输出的格式一致。对应的关系如下：

- docx -> openxml
- opendocument -> odt
- html5 -> epub3
- html4 -> epub2
- latex/beamer/ms/html5 -> pdf （这依赖于我们使用的 --pdf-engine）

> RAW 属性不可与常规的属性混合使用



# 过滤器 Filter

Pandoc 提供了一个接口，用户可以用这个接口来编写程序（叫做过滤器）来在 pandoc 上的  AST （抽象语法树）进行操作。

Pandoc 由一系列的 读入器(Reader) 和写出器（Writer）组成。当我们将一个文档从一种格式转换为另外一种格式的时候，首先会由 pandoc 将输入文档转换为解析为 pandoc 的中间格式——abstract syntax tree（抽象语法树），然后由 Writer 来进行输出。AST 定义在 [`Text.Pandoc.Definition` in the `pandoc-types`package](https://hackage.haskell.org/package/pandoc-types/docs/Text-Pandoc-Definition.html). 模块中。

一个 Filter 就是一个修改  AST 的程序：

```
INPUT --reader--> AST --filter--> AST --writer--> OUTPUT
```

Filter 被看成是一个管道，其从标准输入读入，然后输出到标准输出。其会消耗，然后产生一个 pandoc 的 AST  JSON 表示。Filter 可以用任何的程序写成。我们只需要在命令行中指定过滤器就行：

```
pandoc -s input.txt --filter pandoc-citeproc -o output.htl
```

有 一些第三方的过滤器： [list of third party filters on the wiki](https://github.com/jgm/pandoc/wiki/Pandoc-Filters).



````
                         source format
                              ↓
                           (pandoc)
                              ↓
                      JSON-formatted AST
                              ↓
                           (filter)
                              ↓
                      JSON-formatted AST
                              ↓
                           (pandoc)
                              ↓
                        target format
````

如果我们要用 python 来编写 Filter 的话，可以使用  *pandocfilters* 这个包：

```sh
pip install pandocfilters
```

在最开头的例子中，我们可以来写一个过滤器：

```python
#!/usr/bin/env python

"""
Pandoc filter to convert all level 2+ headers to paragraphs with
emphasized text.
"""

from pandocfilters import toJSONFilter, Emph, Para

def behead(key, value, format, meta):
  if key == 'CodeBlock':
    value[1] = 'code'
    return CodeBlock(value[0],value[1])

if __name__ == "__main__":
  toJSONFilter(behead)
```

`toJSONFilter(behead)` 会遍历 AST，然后对每个元素应用 *behead*  action。如果 *behead* 没有返回值，那么这个节点就不会被改变；如果其返回一个对象，那么这个节点就会被替换；如果其返回一个列表，新的列表就会被拼接在一起。

在这个过滤器中 *format, meta* 没有被使用，但 *format* 提供了一个对目标格式进行访问的途径，*meta* 提供了对文档元数据的访问。

在我们的过滤器中，我们将 CodeBlock 中的内容就直接改成了 *code*。需要记住的是：

- **我们的过滤器读取的是 JSON AST 表示，也就是说 key,value 都是常规的 JSON 类型数据**

- **我们的过滤器返回的一定要是一个 pandoc 类型，需要用 JSON 类型的数据来构造相应的对象**

更多的办法，就需要我们去研究 AST 的 JSON 表示了。

##Lua filters

传统的 pandoc filters 操作的是 JSON 表示的 AST，可以用任何语言来编写 filter 。

尽管我们可以使用任何语言来编写 filter ，但其拥有很多不好的地方。首先，读入 JSON 和写出 JSON 都是有开销的（每个 filter 一对，两次）。其次，一个 filter 是否工作依赖用户的环境是否安装了 filter 的依赖。

因此，从2.0 开始，就开始支持用 lua 来编写 filter 了，这样不需要任何额外的依赖。pandoc 内置了一个 5.3 版本的 lua 解释器。pandoc 的所有数据类型都已经注入到这个 Lua 环境中了

下面就是一个简单的例子：

```lua
return {
  {
    Strong = function (elem)
      return pandoc.SmallCaps(elem.c)
    end,
  }
}
```

或者下面这样

```lua
function Strong(elem)
  return pandoc.SmallCaps(elem.c)
end
```

这些 Lua 代码表达的意思是：遍历 AST 抽象语法树，当我们找到一个 Strong 元素，把它用一个 SmallCaps 进行替换。

### Lua 过滤器结构

Lua 过滤器其实是一个 Lua 表，使用元素名作为键，值是要在这些元素上进行执行的动作的 Lua 函数。

过滤器期待是放在不同的文件中，然后逐个通过 `--lua-filter` 传递给 pandoc 命令行。例如：

```sh
pandoc --lua-filter=current-date.lua -f markdown MANUAL.txt
```

Pandoc 期待每个 Lua 文件返回一个过滤器的列表。这个列表中的过滤器被叫做 **序列**，每个都在前一个过滤器的结果上运行。如果过滤器脚本没有返回任何值， pandoc 会通过搜集所有和 pandoc 元素同名的顶级函数来生成单个过滤器。

过滤器函数的返回值必须是以下几个之一：

- `nil`：保持对象不改变
- 一个 pandoc 对象：这必须和输入的对象相同，同时替换原来的对象
- 一个 pandoc 对象列表：替换原始对象；返回一个空列表会删除对象

也就是说过滤器函数的输入和输出类型必须相同，如果是列表，列表的元素类型也必须一致。

如果找不到和元素同名的过滤器函数，那么就会回退到使用通用的函数。	有两个回退函数被支持，`Inline`和 `Word`。

### 处理元素序列的过滤器

对于某些过滤器任务，知道文档中元素出现的顺序是很有必要的。但一次仅检查一个元素是不够的。

有两个特殊的函数名称，可用于在块列表或内联列表上定义过滤器。

- **Inlines (inlines)** 如果在过滤器中出现，这个函数会在所有 inline 元素列表上调用，例如 一个 `Para` 元素的内容，或者 `Image` 的描述。`inlines` 参数是一个由 `Inline` 元素组成的列表
- **Blocks (blocks)** 意思差不多同上。

这两个过滤器函数特殊的地方在于，它们的返回值只能是 `nil` 或者是元素列表，不允许弹不元素出现，且类型必须和输入相同。

### 执行顺序

在一个过滤器集合中，函数的执行有一个固定的顺序，没有出现的就会跳过：

1. `Inline` 元素的函数
2. `Inlines` 过滤器函数
3. `Block` 元素函数
4. `Block()` 过滤器函数
5. `Meta` 过滤器函数
6. `Pandoc` 过滤器函数

当然，强制使用一个不同的顺序也是可以的，我们可以通过显式的返回多个过滤器集合来实现。例如：

```lua
-- ... filter definitions ...

return {
  { Meta = Meta },  -- (1)
  { Str = Str }     -- (2)
}
```

这就会让 `Meta` 的过滤器在 `Str` 的过滤器之前执行。

### 全局变量

有几个全局变量

- **FORMAT** 输出格式，这样使我们可以根据输入的格式来让过滤器执行不同的操作
- **PANDOC_READER_OPTIONS** 传递给解释器的选项组成的表
- **PANDOC_VERSION** 版本号。
- **PANDOC_API_VERSION** API版本号
- **PANDOC_SCRIPT_FILE** 过滤器文件名
- **PANDOC_STATE** 被所有读入器和写出器共享的状态信息

### 导出的 Pandoc 函数

有几个函数已经导出来了

- [`walk_block`](https://pandoc.org/lua-filters.html#pandoc.walk_block) and [`walk_inline`](https://pandoc.org/lua-filters.html#pandoc.walk_inline) allow filters to be applied inside specific block or inline elements;
- [`read`](https://pandoc.org/lua-filters.html#pandoc.read) allows filters to parse strings into pandoc documents;
- [`pipe`](https://pandoc.org/lua-filters.html#pandoc.pipe) runs an external command with input from and output to strings;
- the [`pandoc.mediabag`](https://pandoc.org/lua-filters.html#module-pandoc.mediabag) module allows access to the “mediabag,” which stores binary content such as images that may be included in the final document;
- the [`pandoc.utils`](https://pandoc.org/lua-filters.html#module-pandoc.utils) module contains various utility functions.

### 元素创建

元素创建函数如 `Str, Para, Pandoc` 被设计来能够很容易的创建能够很简单进行使用的元素，同时这些元素又能被读回 Lua 环境。在内部，pandoc 使用这些函数来创建传递给过滤器函数的对象。这就意味着通过这些模块创建的元素和通过参数传递过来的是一样的。

### Lua 初始化

可以通过在 pandoc 的数据目录中添加一个 `init.lua` 来进行初始化 Lua 环境。一个常用的情况就是加载额外的模块，或是修改默认的模块。

下面的代码片段添加了所有在 `text` 模块中定义的 `unicode` 敏感的函数到默认的 `string` 模块中，以 `uc_` 为前缀。

```lua
for name, fn in pairs(require 'text') do
  string['uc_' .. name] = fn
end
```

这使我们可以用冒号的形式来调用这些函数 `mystring:uc_upper()`。

### Lua类型参考

#### 共享属性

**clone()**

所有的类型，除了只读的，都可以通过 `clone()` 方法来复制

```
local emph = pandoc.Emph {pandoc.Str 'important'}
local cloned_emph = emph:clone()  -- note the colon
```

#### Pandoc

Pancod 文档。可以通过 `pandoc.Pandoc` 构造器创建。对象的相等性通过 `pandoc.utils.equals` 来比较。

- `blocks` 文档内容（`Blocks` 元素的列表）
- `meta` 文档元数据信息（`Meta` 对象）

#### Meta

文档元数据信息；字符串索引的 `MetaValues` 集合。

`[`pandoc.Meta`](https://pandoc.org/lua-filters.html#pandoc.meta) `  创建

[`pandoc.utils.equals`](https://pandoc.org/lua-filters.html#pandoc.utils.equals) 判断相等性

#### MetaValue

文档元数据信息项。

[`pandoc.utils.equals`](https://pandoc.org/lua-filters.html#pandoc.utils.equals) 相等性判断

#### MetaBlocks

一个用做 meta 值的块列表（`Blocks` 列表）

字段：

- `tag, t`：`MetaBlocks` 字面值。

#### MetaBool

Lua 布尔值别名。true /false.

#### MetaInlines

一个用做 meta 值的内联列表（`Inlines` 列表）

可通过 [`pandoc.MetaInlines`](https://pandoc.org/lua-filters.html#pandoc.metainlines) 创建

字段：

- `tag, t`：`MetaInlines` 字面值。

#### MetaList

其他 metadata values 列表([List](https://pandoc.org/lua-filters.html#type-list) of [MetaValues](https://pandoc.org/lua-filters.html#type-metavalue)).

 [`pandoc.MetaList`](https://pandoc.org/lua-filters.html#pandoc.metalist) 创建

字段:

- `tag`, `t`

   `MetaList` (string) 字面值

All methods available for [List](https://pandoc.org/lua-filters.html#type-list)s can be used on this type as well.

#### MetaMap

字符串索的 meta-values字典. (table).

[`pandoc.MetaMap`](https://pandoc.org/lua-filters.html#pandoc.metamap) 创建

字段:

- `tag`, `t`

  `MetaMap` (string)字面值

*Note*: The fields will be shadowed if the map contains a field with the same name as those listed.

#### MetaString

Plain Lua string value (string).

#### Block

[`pandoc.utils.equals`](https://pandoc.org/lua-filters.html#pandoc.utils.equals) 相等性判断

#### BlockQuote

块引用元素。

[`pandoc.BlockQuote`](https://pandoc.org/lua-filters.html#pandoc.blockquote) 创建

字段：

- `content`：块内容（`Blocks` 列表）
- `tag, t`：`BlockQuote` 字面值

#### BulletList

项目符号列表。[`pandoc.BulletList`](https://pandoc.org/lua-filters.html#pandoc.bulletlist) 创建

- `content` 列表 (`Blocks` 列表的列表)
- `tag, t`
  BulletList (string) 字面值

#### CodeBlock

代码块。[`pandoc.CodeBlock`](https://pandoc.org/lua-filters.html#pandoc.codeblock) 创建

- `text` 代码
- `attr` 元素属性(`Attr`)
- `identifier` `attr.identifier` 别名（String）
- `classes`  `attr.classes ` 别名 (列表 of strings)
- `attributes` `attr.attributes` 别名（[Attributes](https://pandoc.org/lua-filters.html#type-attributes)）
- `tag, t` CodeBlock 字面值

## 图表

并不是非常好和统一的方式，准备自己研究一个比较好的办法来实现他。

### mermaid

需要通过装过滤器来完成，当前能看到有两个过滤器：

- [pandoc-mermaid-filter](https://github.com/timofurrer/pandoc-mermaid-filter) python 写的，比较贴合常规使用
- [mermaid-filter](https://github.com/raghur/mermaid-filter) nodejs 写就，当时其对于代码块的识别有点怪。所以我用第一个



**安装过滤器**

```sh
pip install pandoc-mermaid-filter
```

**安装 mermaid cli**

```sh
yarn add mermaid.cli
```

或者

```sh
npm install mermaid.cli
```

不推荐全局安装。

**开始转换**

```sh
MERMAID_BIN=./node_modules/.bin/mmdc pandoc  年终总结.md -o srs.pdf --pdf-engine=xelatex -V CJKmainfont='Songti SC' --filter pandoc-mermaid-filter
```

但事实上，用过滤器的形式还是有点麻烦了，所以考虑一下如何用一个比较简单的方式来完成这个工作。

当然，为了概念上的统一，一条路走到最后，还是值得一试的。

### graphviz

有一个预处理器可以让我们完成在 md 内插入 graphviz plantUML 图表。其本身就是为了 pandoc 而设计的。

[Generic preprocessor (with pandoc in mind)](http://cdsoft.fr/pp/)

暂时不细究，考虑还最终是不是要这样做再进行仔细的考虑。

### plantUML

同上

# DOCX

简单来说，我们是可以直接使用一个命令转换成 DOCX 的。

```sh
pandoc -f gfm -t docx -o test.docx test.md
```

但是会很丑陋，解决的办法，是使用参考样式文件。

```
pandoc -f gfm -t docx -c reference.docx -o test.docx test.md
pandoc -f gfm -t docx -S --reference-docx reference.docx -o test.docx test.md
```

注意，这种用法需要将 `reference.docx` 放在数据目录，或者执行命令的当前工作目录中。这个文件可以是 docx ，也可以是 css 文件。

我尝试用 **Typora** 中的 github.css 文件来进行操作。

```
pandoc -f gfm -t docx -c github.css -o test.docx test.md
```

我们使用命令来打印出标准的参考格式：

```
pandoc -o custom.docx --print-default-data-file=reference.docx
```

然后对其中的样式进行修改，大概的格式官方有了一定的说明：

Paragraph styles:

- Normal
- Body Text
- First Paragraph
- Compact
- Title
- Subtitle
- Author
- Date
- Abstract
- Bibliography
- Heading 1
- Heading 2
- Heading 3
- Heading 4
- Heading 5
- Heading 6
- Heading 7
- Heading 8
- Heading 9
- Block Text
- Footnote Text
- Definition Term
- Definition
- Caption
- Table Caption
- Image Caption
- Figure
- Captioned Figure
- TOC Heading

Character styles:

- Default Paragraph Font
- Body Text Char
- Verbatim Char
- Footnote Reference
- Hyperlink
- Section Number

Table style:

- Table

## 关于表格无边框的解决办法

需要用 word 打开我们的 reference.docx ，然后新建一个表格，设计，修改样式中，将 Table 样式修改成有边框的就行了

## 关于标题没有自动编号

对标题设置一下多级编号就行了。

# 我的解决方案

我日常是使用 [macdown](https://github.com/MacDownApp/macdown) 来写文档，也会使用  typora 来进行一些导出工作，因为其导出为 pdf 的效果非常棒。

在 macdown ，其导出的 PDF 不支持 书签式的 TOC，也就是 docx 中的大纲，所以才萌生了用 pandoc 来实现的想法。

其支持 mermaid, graphviz ，其具体的实现是：

- graphviz 使用了 [viz-js](http://viz-js.com) 来实现
- mermaid 使用的是 [mermaidjs](https://mermaidjs.github.io) 来实现。

就此看来，我如果使用 js 来做过滤器，同时将 viz-js 和 meraidjs 引入到过滤器中的话，就能达到我在编辑器中所查看到的效果，和转换出来的效果一致。将 graphviz ，mermaid 的图表都渲染成 svg 格式，这样的效果就非常的OK了。

在 [pandoc github wiki filter 一节中](https://github.com/jgm/pandoc/wiki/Pandoc-Filters)，列出了用 nodejs 来编写过滤器的办法。有一个模块 [pandoc-filter-node](https://github.com/mvhenderson/pandoc-filter-node) 来进行支持 js 的 filter 编写。

## pdf 模板

- [这里有一个模板集合](http://www.github.com/kjhealy/pandoc-templates)
- [这里还有一个模板](https://github.com/Wandmalfarbe/pandoc-latex-template) 我就使用的这个，注意一下说明文档，把需要的包都给安装上。了解一下 cls 与 sty 的区别。

## markdown 解析器

我一般使用的是 Github Flavor markdown ，我们指定的时候加上 ` -f gfm` 就可以了。

## graphviz mermaid filter

- [mermaid-filter](https://github.com/raghur/mermaid-filter) 
- [pandoc-graphviz-filter](https://github.com/Gowa2017/pandoc-graphviz-filter) 自己写了个，用的是 viz.js@1.8.2 版本。

这样实际上是可以达到与 我在  macdown 内进行编辑的时候的效果是一致的。

## 字体

- 中文字体 使用 adobe 开源字体 [adobe-fonts](https://github.com/adobe-fonts)
- 英文字体

字体的区别：

- **Serif 有衬线体**  在字的笔划开始及结束的地方有额外的装饰，而且笔划的粗细会因直横的不同而有不同。相反的，Sans Serif 则没有这些额外的装饰，笔划粗细大致差不多。**通常文章的内文、正文使用的是易读性较佳的 Serif 字体**，这可增加易读性，而且长时间阅读下因为会以 word 为单位来阅读，较不容易疲倦。
- **Sans Serif 无衬线体** **而标题、表格内用字则採用较醒目的 Sans Serif 字体**，它需要显着、醒目，但不必长时间盯着这些字来阅读。
- **Monospace** **所谓的等宽字体**，是指每个字符宽度都一致的字体。一个著名的例子就是 Courier New 字体。因为字符宽度一致，所以特别容易对齐，能快速精确的定位到某行某列，因此经常用来显示代码。


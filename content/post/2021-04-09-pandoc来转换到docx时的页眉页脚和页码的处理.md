---
title: pandoc来转换到docx时的页眉页脚和页码的处理
categories:
  - macOS
date: 2021-04-09 19:31:45
updated: 2021-04-09 19:31:45
tags: 
  - macOS
  - Pandoc
  - Docx
---

我的目的是，当我用 markdown 来写作，当需要转换到 docx 的时候，加上封面（已实现），同时在封面上不应该有页眉页脚和页码，而需要在从目录开始的地方定义页眉页脚页码等。

<!--more-->

要做这个事情，就首先需要从代码层面知道，是如何进行表示的。我 [从这个地方来参考](http://officeopenxml.com/anatomyofOOXML.php)。这首先就要从 WordprocessingML 的 **节** 说起。

# 节 sections

OOXML 并不只是定义了页面————只有段落和文字。它也能够存储特定的对于页面组成重要的信息，如页面大小、页面方向、边框和边距。它就是通过 节来实现的。一个 节 就是一组特定属性的段落，这些特定的属性定义文字所要出现的页面。

一个节的属性存储在一个 `sectPr` 元素中。

除了最后一个节，其他所有节的 `sectPr` 元素都作为节中最后一个段落的子元素而存在。

对于最后一个节，`sectPr` 是 `body` 的子元素。

```xml
<w:body>
<w:p>
. . .
</w:p>
<w:p>
<w:pPr>
<w:sectPr>
<w:pgSz w:w="12240" w:h="15840"/>
</w:sectPr>
</w:pPr>
</w:p>
<w:p>
. . .
</w:p>
<w:sectPr>
<w:pgSz w:w="15840" w:h="12240" w:orient="landscape"/>
</w:sectPr>
</w:body>
```

# 页脚/页眉 footer/Header

节的页脚通过 `<w:footerReference> / <w:headerReference> ` 元素来指定。页脚通过 id 属性来引用。下面是一个例子：

```xml
<w:sectPr>
. . .
<w:footerReference r:id="rId12" w:type="first"/>
<w:footerReference r:id="rId10" w:type="default"/>
<w:footerReference r:id="rId9" w:type="even"/>
. . .
</w:sectPr>
```

这些引用指向 `document.xml.rels` 中的信息：

```xml
<Relationships xmlns="http://schemas.openxmlformats.org/package/2006/relationships">
<Relationship Id="rId9" type="http://purl.oclc.org/ooxml/officeDocument/relationships/footer" target="footer1.xml"/>
<Relationship Id="rId10" type="http://purl.oclc.org/ooxml/officeDocument/relationships/footer" target="footer2.xml"/>
<Relationship Id="rId12" type="http://purl.oclc.org/ooxml/officeDocument/relationships/footer" target="footer3.xml"/>
</Relationships>
```

对于每个节，可能能有三种类型的页脚：首页的，偶数页（even）的，和奇数页（odd）的。通过 `type` 属性来指定。

属性的应用顺序规则如下：

1. 如果首页的页脚不存在，但是在 `sectPr` 中指定了 `titlePg`，那么它将会从前一节继承。如果当前是第一个节，那么就会是一个空的页脚。如果 `titlePg` 没有指定，那就应用 odd （奇数）的页脚。
2. 如果没有指定偶数页（even）的页脚且 `evenAndOddHeaders` 在 `settings` 里面指定，那么就会从前一节继承。如果是第一个节，那就是一个空的页脚。如果 `evenAndOddHeaders` 没有指定，那就会应用 odd 页脚就会应用。
3. 如果没有指定奇数（odd）页脚，那么偶数页的页脚会从前一节继承。如果是第一个节，那么就是一个空的页脚。



听起来有点绕，其实就是：

1. 如果指定了首页页脚，那就直接使用。如果指定了 `titlePg` 就从前一节继承（第一节没有前一节，所以为空），否则就应用 odd 页的页脚（首页也看成奇数页）
2. 如果没有偶数页的页脚但在全局设置里面又需要它，那它就从前一节继承。如果没有全局指定，那就应用 odd 页脚（所以它是默认的）

# 页码 PageNumber

通过节内的 `<w:pgNumType>` 元素来指定：

```xml
<w:sectPr>
<w:pgNumType w:start="25"/>
</w:sectPr>
```



# 我的目的

我的目的是让封面没有页眉页脚页码什么的，那它作为第一节，只需要都不指定就行了。我们在这边设置 `evenAndOddHeaders` 而是单独为后面的正文节指定页眉页脚。

根据继承规则，然后我们指定一个 odd 的页眉页脚，就能达到这个目的了。



最终的实现方法：

1. 在我的 filter 内，在封面的最后插入一个空的 `sectPr` ，这就是第一节，没有指定头、脚，那肯定就是空了。
2. 在默认的模板内设置后页眉、页脚，然后保存，这是默认的一个节，也就是最后一节，会应用到所有的页面，搞定。
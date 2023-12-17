---
title: pandoc从markdown到pdf与docx全过程
categories:
  - macOS
date: 2020-10-13 18:25:49
updated: 2020-10-13 18:25:49
tags: 
  - macOS
  - Pandoc
---

在之前的一篇文章 {% post_link 使用pandoc转换pdf与docx加上书签目录 使用pandoc转换pdf与docx加上书签目录 %} 已经有很多基础的知识，现在是来再次回顾一下这个过程。

<!--more-->

# 起

不要问为什么，就是喜欢 markdown，这个将内容和样式分离的东西，可以让我们专注写作，具体是不是真的专注呢，我觉得是的，只要不是遇到太复杂的东西，都可以解决。

但是，不能要求所有的所以人都像我这样搞，因此，在与工作，或者一些情况下交换文档时，又不得不换成 pdf/docx 格式，所以必须要有一个可以进行转换的工具，所以遇到了 Pandoc。

再有，我经常会画图，现在的 markdown 编辑器，对于 graphviz，mermaid 的支持都非常棒，但是想要形成文件都要以图片或者 svg 的形式了，因此就有了需要对这些绘图代码到图片的转换这么一个过程。Pandoc 的过滤器完美的支持了。

# 环境

操作系统：macOS

编辑器：typora/vscode 关于编辑器的选择，各有各的爱好。比如说 macdown，mweb 这些都还可以，但是总有不如人意的地方。最终还是 typora 胜出。但对于想要内嵌支持 graphviz ，typora 并没有提供支持，所以用 vscode 作为辅助了。

文档存储：git

在线部署：gitbook。这里需要加上两个过滤器来支持 graphviz 和 memraid

格式转换：pandoc

# 遇到的难题

常规的纯文字的 markdown，用 typora 写出来很棒，导出成 pdf 也很OK，但是其实其转换成 PDF 用的也是 Pandoc 来实现的，依然需要我们进行设置。但如果遇到其内嵌不支持的图，那就坑了，因为是不开源的产品，也不知道什么时候官方会进行支持所以只能自己变通一下。

因此，实际上我是进行纯文本的撰写的，不过，有的代码，我需要能够识别出来，给我转换成图片，然后放到 PDF 或者 Word 中，并在线部署。

# 解决的思路

利用 Pandoc 将标签文本，转换成抽象语法树的机制，然后用过滤器来进行修改抽象语法树，交给 Writer 来进行写出。同时，由于 pandoc 内嵌了对于 Lua5.3 的支持，所以能用 lua 过滤器的尽量用 lua 过滤器来写。

## graphviz

实际上就是在过滤器中，调用我们已安装的命令行 `dot` 来进行转换，然后通过管道返回到 pandoc，然后也 base64 图片的形式来嵌到语法树中。

```lua
local function render_dot_code(code, e)
    local img = pipe("base64", {}, pipe(e, {"-T" .. "svg"}, code))
    return pandoc.Para({pandoc.Image("", "data:image/svg+xml;base64," .. img)})
end
```

> graphviz 的安装使用 `brew install graphviz`

## mermaid

这个道理同上，不过利用到了 mermaid 提供的命令行工具来实现。

```sh
yarn @mermaidjs/mermaid.cli
```

```lua
local function render_memrmaid_code(code)
    pipe(os.getenv("HOME") .. "/pandoc/luafilter/node_modules/.bin/mmdc", {"-t default -o svg"}, code)
    local f = io.open("out.svg")
    local img = pipe("base64", {}, (f:read("a")))
    f:close()
    os.remove("out.svg")
    return pandoc.Para({pandoc.Image("", "data:image/svg+xml;base64," .. img)})
```

## 表格内强制换行

对于在 markdown 表格内的东西，我们可以使用  `<BR>` 来进行换行，但这只是在 HTML 的时候有用。 Docx/Pdf 无效，因此我们需要进行对应的转换。

```lua
local function convert_inline_newline(inline)
    if FORMAT == "latex" then
        inline.format = "latex"
        inline.text = "\\\\"
    elseif FORMAT == "docx" then
        inline.format = "openxml"
        inline.text = "<w:r><w:br/></w:r>"
    end
end
function Blocks(el)
    local r = {}
    for k, v in pairs(el) do
        if v.t == "CodeBlock" then
            e = get_dot_engine(v)
            if e then
                r[k] = render_dot_code(v.text, e)
            elseif ismermaid(v) then
                r[k] = render_memrmaid_code(v.text)
            else
                r[k] = v
            end
        elseif v.t == "Table" then
            r[k] = v
            for k, tbody in pairs(v.bodies) do
                for _, row in pairs(tbody.body) do
                    for _, cell in pairs(row[2]) do
                        for _, block in pairs(cell.contents) do
                            for _, inline in pairs(block.content) do
                                if inline.t == "RawInline" then
                                    if string.match(inline.text, "<[BbRr]*.*>") then
                                        convert_inline_newline(inline)
                                    end
                                end
                            end
                        end
                    end
                end
            end
        else
            r[k] = v
        end
    end
    return r
end
```

# 关于左右对齐

有的时候，有些文档。我们需要一部分靠左，一部分靠右，比如合同的签字处。这种情况下我们怎么搞呢？就只能是利用 openxml 的代码了。注意，这个是针对的是 DOCX的，其实更建议还是弄成标签的形式，然后在过滤器里面根据输出进行处理比较友好。

```xml
<w:p>
      <w:pPr>
        <w:pStyle w:val="FirstParagraph" />
        <w:tabs>
          <w:tab w:val="left" w:pos="4366" />
        </w:tabs>
        <w:ind w:firstLineChars="0" w:firstLine="0" />
      </w:pPr>
      <w:proofErr w:type="spellStart" />
      <w:r>
        <w:t>男方：</w:t>
      </w:r>
      <w:proofErr w:type="spellEnd" />
      <w:r>
        <w:t xml:space="preserve"> </w:t>
      </w:r>
      <w:r>
        <w:tab />
      </w:r>
      <w:proofErr w:type="spellStart" />
      <w:r>
        <w:t>女方：</w:t>
      </w:r>
      <w:proofErr w:type="spellEnd" />
      </w:p>
      
      <w:p>
      <w:pPr>
        <w:pStyle w:val="FirstParagraph" />
        <w:tabs>
          <w:tab w:val="left" w:pos="4366" />
        </w:tabs>
        <w:ind w:firstLineChars="0" w:firstLine="0" />
      </w:pPr>
      <w:proofErr w:type="spellStart" />
      <w:r>
        <w:t>日期：    年    月    日</w:t>
      </w:r>
      <w:proofErr w:type="spellEnd" />
      <w:r>
        <w:t xml:space="preserve"> </w:t>
      </w:r>
      <w:r>
        <w:tab />
      </w:r>
      <w:proofErr w:type="spellStart" />
      <w:r>
        <w:t>日期：    年    月    日</w:t>
      </w:r>
      <w:proofErr w:type="spellEnd" />
      </w:p>
```



# 完整项目地址

[https://github.com/Gowa2017/pandoc-dot-mermaid-luafilter](https://github.com/Gowa2017/pandoc-dot-mermaid-luafilter)

这里面也包含我用了 reference.docx 文件。
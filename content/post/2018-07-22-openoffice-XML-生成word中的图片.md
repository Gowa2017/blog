---
title: openoffice-XML-生成word中的图片
categories:
  - Java
date: 2018-07-22 20:49:43
updated: 2018-07-22 20:49:43
tags: 
  - Android
  - Java
  - Docx
---
事情的需求是这样的，用poi来生成文书的，需要在里面插入图片，正常的情况下是没有问题的，但是对于一个是横向的图片（宽>高）的，要 在A4纸上打印出来就非常的丑陋了，要实现将图片进行旋转，并且居中，缩放的操作这个可难到我了。使用的是poi来进行生成，然后用Openoffice来转换为pdf，很明显openOffice对某些支持不不是很好。但依然 还是要了解一下图片，在Word内是怎么样表示的。

# 前言

图片，图标，形状等等，在word中都是以 DrawingML 来表示的一个可绘制的对象。在一个WordprocessingML中，有可能包含几种图形对象：

* 图片
* 已锁定的画布
* 流程图
* 图表

当这些对象要出现在 WordprocessingML 文档时，必须包含指定对象在文档页面中的放置位置信息。如：对象是在行内还是锚定了一个位置。

WordprocessingML Drawing 命令空间实现了这些能力，包含了所有用来锚定和显示 DrawingML 对象的信息。


# anchor
下面的例子显示了一个居中显示的图片的xml代码：

```xml
 <w:r>
    <w:drawing>
<wp:anchor relativeHeight="10" allowOverlap="true"> <wp:positionH relativeFrom="margin">
<wp:align>center</wp:align> </wp:positionH>
<wp:positionV relativeFrom="margin">
<wp:align>center</wp:align> </wp:positionV>
<wp:extent cx="2441542" cy="1828800"/> <wp:wrapSquare wrapText="bothSides"/> <a:graphic>
          ...
        </a:graphic>
      </wp:anchor>
    </w:drawing>
</w:r>
```

`anchor ` 元素，说明，这个对象不会放在文本行内，而`anchor `的子元素则说明，这个对象居会垂直居中和水平居中，且文本可以以方形环绕它。

`anchor` 包含了下面这些子元素

## positionH (Horizontal Positioning)
指定浮动绘制对象在文档中的水平位置。位置有两部分来指定

1. 基准位置 —— 此元素的`relativeFrom`属性指定应该从文档的哪个部分来进行计算。
2. 计算位置 —— 子元素的子元素`align or posOffset`，指定距离基准位置的何处。

此元素的XML代码定义如下：

```xml
<complexType name="CT_PosH">
    <sequence>
       <choice minOccurs="1" maxOccurs="1">
           <element name="align" type="ST_AlignH" minOccurs="1" maxOccurs="1"/>
           <element name="posOffset" type="ST_PositionOffset" minOccurs="1" maxOccurs="1"/>
       </choice>
    </sequence>
    <attribute name="relativeFrom" type="ST_RelFromH" use="required"/>
</complexType>
```
下面的代码指定了一个位于页面中的元素：

```xml
<wp:anchor ... >
<wp:positionH relativeFrom="margin">
<wp:align>center</wp:align> </wp:positionH>
<wp:positionV relativeFrom="margin">
<wp:align>center</wp:align> </wp:positionV>
  </wp:anchor>
```

### 属性
#### relativeFrom(Horizontal Position Relative Base)
这个指定基准位置，可能的值由 `ST_RelFromH`简单类型定义。

#### ST_RelFromH (Horizontal Relative Positioning)

1. character (Character) 
2. column (Column)
3. insideMargin (Inside Margin)
4. leftMargin (Left Margin)
5. margin (Page Margin)
6. outsideMargin (Outside Margin)
7. page (Page Edge)
8. rightMargin (Right Margin)

xml定义如下:

```xml
<simpleType name="ST_RelFromH">
    <restriction base="xsd:token">
       <enumeration value="margin"/>
       <enumeration value="page"/>
       <enumeration value="column"/>
       <enumeration value="character"/>
       <enumeration value="leftMargin"/>
       <enumeration value="rightMargin"/>
       <enumeration value="insideMargin"/>
       <enumeration value="outsideMargin"/>
    </restriction>
</simpleType>
```


### 子元素
#### align (Relative Horizontal Alignment)
指定水平对齐。

下面这个例子，表明这个对象会对齐在页面的左，上。

```xml
<wp:anchor ... >
<wp:positionH relativeFrom="page">
      <wp:align>left</wp:align>
    </wp:positionH>
    ...
  </wp:anchor>
```

可用值：

* left	Left Alignment
* right	Right Alignment
* center	Center Alignment
* inside	Inside
* outside	Outside


#### posOffset (Absolute Position Offset)

## positionV (Vertical Positioning)

和 positionH 一样要指定计算基准位置，偏移参数。


```xml
<complexType name="CT_PosV">
    <sequence>
       <choice minOccurs="1" maxOccurs="1">
           <element name="align" type="ST_AlignV" minOccurs="1" maxOccurs="1"/>
           <element name="posOffset" type="ST_PositionOffset" minOccurs="1" maxOccurs="1"/>
       </choice>
    </sequence>
    <attribute name="relativeFrom" type="ST_RelFromV" use="required"/>
</complexType>
```

### ST_RelFromH

* margin	Page Margin
* page	Page Edge
* paragraph	Paragraph
* line	Line
* topMargin	Top Margin
* bottomMargin	Bottom Margin
* insideMargin	Inside Margin
* outsideMargin	Outside Margin

### align 可用值

* top	Top
* bottom	Bottom
* center	Center Alignment
* inside	Inside
* outside	Outside

---
title: Word-OOXML中的尺寸单位
categories:
  - Java
date: 2018-06-08 09:42:39
updated: 2018-06-08 09:42:39
tags: 
  - Android
  - Java
  - Docx
---

国际纸长标准 ISO 216 A4 （210 * 297mm，8.3 * 11.7in)，在word中档中的表示如下:

原文：[https://startbigthinksmall.wordpress.com/2010/01/04/points-inches-and-emus-measuring-units-in-office-open-xml/](https://startbigthinksmall.wordpress.com/2010/01/04/points-inches-and-emus-measuring-units-in-office-open-xml/)

```xml
// pageSize: with and height in 20th of a point
<w:pgSz w:w="11906" w:h="16838"/>
```

为什么会是这么巨大的值呢？因为其使用的单位是不同的。

对于上面我说到的 A4 纸张标准表示几个单位表示如下：


|宽，高|Twip(1/20Pt)|Point(1/72 Inch)|Inch|Cm（2.54\*Inch）|
|-----|-----|------|-----|----|
||缇|点|英寸|厘米|
|宽度|11906|595.3|8.27|21.0|
|高度|16838|841.9|11.69|29.7|

<!--more-->

# 单位
对于 WordProcessingML 文件，其的DPI是 72。

## halp-points

常用来指定字体大小，*12pt* 的字体大小，等于 *24 half points*。

```xml
// run properties
<w:rPr>
  // size value in half-points
  <w:sz w:val="24"/>
</w:rPr>
```

## 1/50百分点

用来在某些地方进行相对测量。用下面的表格例子来说明：

```xml
<w:tbl>
    <w:tblPr>
      <!-- table width in 50th of a percent -->
      <w:tblW w:w="2500" w:type="pct"/>
    </w:tblPr>
    <w:tblGrid/>
    <w:tr>
        <w:tc>
            <w:p>
                <w:r>
                    <w:t>Hello, World!</w:t>
                </w:r>
            </w:p>
        </w:tc>
    </w:tr>
</w:tbl>
```

上面的表格将会占据可用宽度的 *50%*.如果想用 1/20点，也就是 dxa（缇）来表示，要设置  `w:type=dxa`。

#  EMU(English Metric Unit)

在基于矢量的绘制和图片的时候，EMU常用来进行描述坐标。EMU是 厘米 和 英寸的桥梁。**1Inch = 914400 EMUs**，**1cm = 360000 EMUs**。


加入我们要在一个表格内插入图片：

```xml
<w:tcW w:w="2410" w:type="dxa"/>
```

|宽|Twip(1/20Pt)|Point(1/72 Inch)|Inch|Cm（2.54\*Inch）|EMU|
|-----|-----|------|-----|-----|---|
||缇|点|英寸|厘米|英寸*914400|
|宽度|2410|120.5|1.57361||1530350|

计算方法：


2410 * 914400 / 20 /72  = 1530350

或者 直接：

2140 * 635 = 1530350

就是说 1 dxa = 653 EMU






# A4纸的像素和分辨率

A4纸的尺寸是 210mm * 297 mm， 1 英寸 = 2.54 cm。

根据分辨率来得出常用的尺寸：

当分辨率是72像素/英寸时，A4纸像素长宽分别是842×595；  
当分辨率是120像素/英寸时，A4纸像素长宽分别是2105×1487；  
当分辨率是150像素/英寸时，A4纸像素长宽分别是1754×1240；  
当分辨率是300像素/英寸时，A4纸像素长宽分别是3508×2479；  


现在我们一般 用的是 300dpi的这样。

# POI内的实现
*     public static final int EMU_PER_PIXEL = 9525;
*     public static final int EMU_PER_POINT = 12700;
*     public static final int EMU_PER_CENTIMETER = 360000;
*     public static final int MASTER_DPI = 576;
*     public static final int PIXEL_DPI = 96;
*     public static final int POINT_DPI = 72;
*     public static final float DEFAULT_CHARACTER_WIDTH = 7.0017F;
*     public static final int EMU_PER_CHARACTER = 66691;
---
title: Poi中添加Anchor图片到Docx文档
categories:
  - Java
date: 2018-11-28 13:54:11
updated: 2018-11-28 13:54:11
tags: 
  - Java
  - Poi
  - Docx
---
做文书的时候，需要把印章放在相关的文字上面。事实上这个问题拖了很久了都没有去处理。因为 POI 对于这种要把图片盖在文字上面的做法很蛋疼，其提供的API并不好用。使用了底层的，直接写 xml 的方式来插入进去的。

参考了 [stackoverflow.com/](https://stackoverflow.com/questions/47673133/change-image-layout-or-wrap-in-docx-with-apache-poi) 的做法后完成

<!--more-->

# Inline
这种方式添加图片是非常的简单。只需要一句代码就可以了。

```java
run.addPicture(java.io.InputStream pictureData,
                              int pictureType,
                              java.lang.String filename,
                              int width,
                              int height)
```

指定 输入流，图片类型，文件名，宽，高即可。注意这里的宽高是 EMU 为单位的。

关于这个方法的API解释，地址在这里：[http://poi.apache.org/apidocs/dev/index.html](http://poi.apache.org/apidocs/dev/index.html)

添加之后我们可以查看我们 run 的 xml 代码是什么：

```java
        System.out.println(run.getCTR());
```

```xml
<xml-fragment w:rsidRPr="00924CAC" xmlns:cx="http://schemas.microsoft.com/office/drawing/2014/chartex" xmlns:cx1="http://schemas.microsoft.com/office/drawing/2015/9/8/chartex" xmlns:m="http://schemas.openxmlformats.org/officeDocument/2006/math" xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006" xmlns:o="urn:schemas-microsoft-com:office:office" xmlns:r="http://schemas.openxmlformats.org/officeDocument/2006/relationships" xmlns:v="urn:schemas-microsoft-com:vml" xmlns:w="http://schemas.openxmlformats.org/wordprocessingml/2006/main" xmlns:w10="urn:schemas-microsoft-com:office:word" xmlns:w14="http://schemas.microsoft.com/office/word/2010/wordml" xmlns:w15="http://schemas.microsoft.com/office/word/2012/wordml" xmlns:w16se="http://schemas.microsoft.com/office/word/2015/wordml/symex" xmlns:wne="http://schemas.microsoft.com/office/word/2006/wordml" xmlns:wp="http://schemas.openxmlformats.org/drawingml/2006/wordprocessingDrawing" xmlns:wp14="http://schemas.microsoft.com/office/word/2010/wordprocessingDrawing" xmlns:wpc="http://schemas.microsoft.com/office/word/2010/wordprocessingCanvas" xmlns:wpg="http://schemas.microsoft.com/office/word/2010/wordprocessingGroup" xmlns:wpi="http://schemas.microsoft.com/office/word/2010/wordprocessingInk" xmlns:wps="http://schemas.microsoft.com/office/word/2010/wordprocessingShape">
  <w:rPr>
    <w:rFonts w:ascii="仿宋_GB2312" w:eastAsia="仿宋_GB2312" w:hAnsi="Times New Roman"/>
    <w:kern w:val="2"/>
    <w:sz w:val="32"/>
    <w:szCs w:val="32"/>
  </w:rPr>
  <w:t/>
  <w:drawing>
    <wp:inline distT="0" distR="0" distB="0" distL="0">
      <wp:extent cx="1440000" cy="1440000"/>
      <wp:docPr id="0" name="Drawing 0" descr="stamp"/>
      <a:graphic xmlns:a="http://schemas.openxmlformats.org/drawingml/2006/main">
        <a:graphicData uri="http://schemas.openxmlformats.org/drawingml/2006/picture">
          <pic:pic xmlns:pic="http://schemas.openxmlformats.org/drawingml/2006/picture">
            <pic:nvPicPr>
              <pic:cNvPr id="0" name="Picture 0" descr="stamp"/>
              <pic:cNvPicPr>
                <a:picLocks noChangeAspect="true"/>
              </pic:cNvPicPr>
            </pic:nvPicPr>
            <pic:blipFill>
              <a:blip r:embed="rId6"/>
              <a:stretch>
                <a:fillRect/>
              </a:stretch>
            </pic:blipFill>
            <pic:spPr>
              <a:xfrm>
                <a:off x="0" y="0"/>
                <a:ext cx="1440000" cy="1440000"/>
              </a:xfrm>
              <a:prstGeom prst="rect">
                <a:avLst/>
              </a:prstGeom>
            </pic:spPr>
          </pic:pic>
        </a:graphicData>
      </a:graphic>
    </wp:inline>
  </w:drawing>
</xml-fragment>
```

很一目了然，在 run -> drawing -> inline -> graphic -> ....

由于图片的代码是一样的，其实我们只需要把 inline 元素这里进行变更就好了。

但是，事实上，POI 上并没有提供 Anchor 的 API，所以只能用比较底层的方式来进行了。

# Anchor

我们先定义一个 Anchor 节点。

```java        
String anchorXML =
                "<wp:anchor xmlns:wp=\"http://schemas.openxmlformats.org/drawingml/2006/wordprocessingDrawing\" "
                        + "simplePos=\"0\" relativeHeight=\"0\" behindDoc=\"0\" locked=\"0\" layoutInCell=\"1\" allowOverlap=\"1\">"
                        + "<wp:simplePos x=\"0\" y=\"0\"/>"
                        + "<wp:positionH relativeFrom=\"" + relativeFrom +"\">"
//                        + "<wp:align>" + relativeFrom + "</wp:align>"
                        + "<wp:posOffset>" + hOffset + "</wp:posOffset>"
                        +"</wp:positionH>"
//                        + "<wp:positionH relativeFrom=\"column\"><wp:align>left</wp:align></wp:positionH>"
                        + "<wp:positionV relativeFrom=\"paragraph\"><wp:align>center</wp:align></wp:positionV>"
                        + "<wp:extent cx=\"" + width + "\" cy=\"" + height + "\"/>"
                        + "<wp:effectExtent l=\"0\" t=\"0\" r=\"0\" b=\"0\"/>"
                        + "<wp:wrapNone/>"
                        + "<wp:docPr id=\"1\" name=\"Drawing 0\" descr=\"" + drawingDescr + "\"/><wp:cNvGraphicFramePr/>"
                        + "</wp:anchor>";
```

确定图片在哪里的元素就在 positionH、positionV 上。这两者都有一个属性 relativeFrom 决定相对与哪个地方来计算位置，这个属性的取值可参考文章：[Java/openoffice-XML-生成word中的图片.html#anchor](/Java/openoffice-XML-生成word中的图片.html#anchor)。我一般选用 margin 就行了。主要原因就是，很多取值 我需要用 openoffice 转的时候，不支持。

但是  positionH、positionV  的子元素中  align 与 posOffset 只能有一个，两者共存文档就会损坏。

现在我们用代码来进行操作，根据我们的输入参数来返回 Anchor：

```java
    /**
     *
     * @param graphicalobject 图片数据
     * @param drawingDescr 图片描述
     * @param width 宽
     * @param height 高
     * @param hOffset 水平偏移
     * @param vOffset 垂直偏移
     * @param relativeFrom 相对位置
     * @return
     * @throws Exception
     */
    private static CTAnchor getAnchorWithGraphic(CTGraphicalObject graphicalobject,
                                                 String drawingDescr, int width, int height,
                                                 int hOffset, int vOffset, String relativeFrom) throws Exception {

        String anchorXML =
                "<wp:anchor xmlns:wp=\"http://schemas.openxmlformats.org/drawingml/2006/wordprocessingDrawing\" "
                        + "simplePos=\"0\" relativeHeight=\"0\" behindDoc=\"0\" locked=\"0\" layoutInCell=\"1\" allowOverlap=\"1\">"
                        + "<wp:simplePos x=\"0\" y=\"0\"/>"
                        + "<wp:positionH relativeFrom=\"" + relativeFrom +"\">"
//                        + "<wp:align>" + relativeFrom + "</wp:align>"
                        + "<wp:posOffset>" + hOffset + "</wp:posOffset>"
                        +"</wp:positionH>"
//                        + "<wp:positionH relativeFrom=\"column\"><wp:align>left</wp:align></wp:positionH>"
                        + "<wp:positionV relativeFrom=\"paragraph\"><wp:align>center</wp:align></wp:positionV>"
                        + "<wp:extent cx=\"" + width + "\" cy=\"" + height + "\"/>"
                        + "<wp:effectExtent l=\"0\" t=\"0\" r=\"0\" b=\"0\"/>"
                        + "<wp:wrapNone/>"
                        + "<wp:docPr id=\"1\" name=\"Drawing 0\" descr=\"" + drawingDescr + "\"/><wp:cNvGraphicFramePr/>"
                        + "</wp:anchor>";

        CTDrawing drawing = CTDrawing.Factory.parse(anchorXML);
        CTAnchor anchor = drawing.getAnchorArray(0);
        anchor.setGraphic(graphicalobject);
        return anchor;
    }
```

graphicalobject 从哪里来？我们为了简单，不要手动添加。我们通过  我们刚开始就介绍的添加到 inline 里面的方法来添加。然后获取了以后，我们就把 inline 给删除就OK。

```java
        XWPFDocument document = new XWPFDocument();
        XWPFParagraph paragraph = document.createParagraph();
        XWPFRun run = paragraph.createRun();
        
        // 添加 inline 图片 这里有个问题，就是openOffice识别的时候，必须要图片数据的宽高和后面我们设置的宽高一致
		 run.setText("行内图片: ");
        InputStream in = new FileInputStream("/Users/wodediannao/sample.png");
        run.addPicture(in, Document.PICTURE_TYPE_JPEG, "/Users/wodediannao/sample.png", Units.toEMU(100), Units.toEMU(30));
        in.close();
        
        // 添加浮动图片
        // 1. 先添加一个行内图片
        run = paragraph.createRun();
        in = new FileInputStream("/Users/wodediannao/sample.png");
        run.addPicture(in, Document.PICTURE_TYPE_JPEG, "/Users/wodediannao/sample.png", Units.toEMU(100), Units.toEMU(30));
        in.close();
        
        // 2. 获取到图片数据
        CTDrawing drawing = run.getCTR().getDrawingArray(0);
        CTGraphicalObject graphicalobject = drawing.getInlineArray(0).getGraphic();
        
        // 3. 加入 Anchor 并删除  Inline 的图片
        CTAnchor anchor = getAnchorWithGraphic(graphicalobject, "/Users/wodediannao/sample.png",
                Units.toEMU(100), Units.toEMU(30),
                Units.toEMU(0), Units.toEMU(0),"");
        drawing.setAnchorArray(new CTAnchor[]{anchor});
        drawing.removeInline(0);
        // 4. 写出文件
        document.write(new FileOutputStream("WordInsertPictures.docx"));
        document.close();
```
---
title: 对于TextView或者普通的View应用Selector
categories:
  - Android
date: 2019-03-24 21:53:26
updated: 2019-03-24 21:53:26
tags: 
  - Android
---
说是 Selector ，其实在官方的定义中，有两个概念。 ColorStateList 与 StateListDrawable 两个概念。一个是根据对象的状态来改变颜色，而一个是根据状态的对象来绘制的内容。ColorStateList 位于 res/colors/.xml 中，StateListDrawable 位于 res/drawable/.xml 中

# 起因

TabHost 是很好的，但是用来做标签页的时候感觉麻烦了点。我完全可以根据选中的状态来更新我的　Tab 显示。更换前后背景色，字体颜色。

# ColorStateList

## 语法

```xml
<?xml version="1.0" encoding="utf-8"?>
<selector xmlns:android="http://schemas.android.com/apk/res/android" >
    <item
        android:color="hex_color"
        android:state_pressed=["true" | "false"]
        android:state_focused=["true" | "false"]
        android:state_selected=["true" | "false"]
        android:state_checkable=["true" | "false"]
        android:state_checked=["true" | "false"]
        android:state_enabled=["true" | "false"]
        android:state_window_focused=["true" | "false"] />
</selector>
```

状态根据名字就能明白其意义：

- pressed 按下
- focused 获取焦点
- selected 选中 Tab
- checkable 可选 CheckBox
- checked 已选 CheckBox
- enable 启用
- state_window_focused 应用窗口获取焦点。

我们在 layout 中，对于 TextView 应用 `android:color=@color/tv_state_color` 即可应用我们定义的状态颜色列表了。

当然，由于TextView 是没有状态的，所以我只设置了 selected 的状态色。然后在 Java 代码中手动设置了视图的选中状态。

# StateListDrawable

语法：

```xml
<?xml version="1.0" encoding="utf-8"?>
<selector xmlns:android="http://schemas.android.com/apk/res/android"
    android:constantSize=["true" | "false"]
    android:dither=["true" | "false"]
    android:variablePadding=["true" | "false"] >
    <item
        android:drawable="@[package:]drawable/drawable_resource"
        android:state_pressed=["true" | "false"]
        android:state_focused=["true" | "false"]
        android:state_hovered=["true" | "false"]
        android:state_selected=["true" | "false"]
        android:state_checkable=["true" | "false"]
        android:state_checked=["true" | "false"]
        android:state_enabled=["true" | "false"]
        android:state_activated=["true" | "false"]
        android:state_window_focused=["true" | "false"] />
</selector>
```

唉差不多一个意思了。[详细参考这里](https://developer.android.com/guide/topics/resources/drawable-resource.html#StateList)

## Shape

可以利用形状来绘制各种我们需要的内容。比如圆角啊，光标啊，椭圆等等。

```xml
<?xml version="1.0" encoding="utf-8"?>
<shape
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:shape=["rectangle" | "oval" | "line" | "ring"] >
    <corners
        android:radius="integer"
        android:topLeftRadius="integer"
        android:topRightRadius="integer"
        android:bottomLeftRadius="integer"
        android:bottomRightRadius="integer" />
    <gradient
        android:angle="integer"
        android:centerX="float"
        android:centerY="float"
        android:centerColor="integer"
        android:endColor="color"
        android:gradientRadius="integer"
        android:startColor="color"
        android:type=["linear" | "radial" | "sweep"]
        android:useLevel=["true" | "false"] />
    <padding
        android:left="integer"
        android:top="integer"
        android:right="integer"
        android:bottom="integer" />
    <size
        android:width="integer"
        android:height="integer" />
    <solid
        android:color="color" />
    <stroke
        android:width="integer"
        android:color="color"
        android:dashWidth="integer"
        android:dashGap="integer" />
</shape>
```

这里解释一下几个选项就OK

- corners 圆角半径
- gradient 渐变
- size 大小
- solid 填充色
- stroke 绘制的画笔颜色。
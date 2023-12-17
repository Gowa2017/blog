---
title: FloatingActionButton在安卓6.0上崩溃的问题
categories:
  - Android
date: 2018-09-25 17:56:53
updated: 2018-09-25 17:56:53
tags: 
  - Android
---
这个问题其实不是什么大问题。6.0似乎用的人不多了没怎么在意。空的时候搜索了一下 stackoverflow 果然有人遇到相同问题。

<!--more-->

原问：[https://stackoverflow.com/questions/49291349/floating-action-button-not-working-in-marshmallow-and-lollipop](https://stackoverflow.com/questions/49291349/floating-action-button-not-working-in-marshmallow-and-lollipop)


我原来的 fab 代码如下：

```xml
    <android.support.design.widget.FloatingActionButton
        android:id="@+id/btn_push"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_alignParentBottom="true"
        android:layout_alignParentEnd="true"
        android:layout_alignParentRight="true"
        android:layout_marginBottom="10dp"
        android:layout_marginEnd="10dp"
        android:layout_marginRight="10dp"
        android:clickable="true"
        android:focusable="true"
        android:visibility="gone"
        android:backgroundTint="@color/blue"
        android:src="@drawable/ic_done_white_36dp" />
```

改成：

```xml
    <android.support.design.widget.FloatingActionButton
        android:id="@+id/btn_push"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_alignParentBottom="true"
        android:layout_alignParentEnd="true"
        android:layout_alignParentRight="true"
        android:layout_marginBottom="10dp"
        android:layout_marginEnd="10dp"
        android:layout_marginRight="10dp"
        android:clickable="true"
        android:focusable="true"
        android:visibility="gone"
        app:backgroundTint="@color/blue"
        app:src="@drawable/ic_done_white_36dp" />
```

原因：在低版本使用一个 vector drawables 的时候，需要用这个来指定。

>To use VectorDrawableCompat, you need to set android.defaultConfig.vectorDrawables.useSupportLibrary = true.  To use VectorDrawableCompat, you need to make two modifications to your project. First, set android.defaultConfig.vectorDrawables.useSupportLibrary = true in your build.gradle file, and second, use app:srcCompat instead of android:src to refer to vector drawables.
---
title: Cocos-Creator浏览器正常，打包apk错误的问题
categories:
  - Cocos Creator
date: 2018-09-01 12:02:48
updated: 2018-09-01 12:02:48
tags: 
  - Cocos Creator
---
拷了份代码，浏览器，模拟器都正常预览，使用。而一打包到安卓下就不行了。具体的表现形式是，一直黑屏，用adb 查看 logcat 的话，是js_polyfill 报出了一个错误，调用 `startPhase`失败。搜索良久找不到方案。

# 资源缺失
偶尔发现一个问题就是，当我进入一个Prefab的时候，会提示这个Prefab内使用的资源丢失了。而我看我场景内，有这个Prefab实例的又显示正常，于是进行了一下同步，再进行打包。

居然就OK了。。

实在是坑爹了。

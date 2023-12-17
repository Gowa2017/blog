---
title: React-Native使用Flex来进行布局
categories:
  - [JavaScript]
  - [React]
date: 2020-03-09 23:22:54
updated: 2020-03-09 23:22:54
tags:
  - JavaScript
  - React Native
  - React
---

React Native 官方已经推荐使用  expo 来进行项目的开发了，但是国内使用的话还有几个非常猥琐的问题。一个就是项目的初始化很慢，还有就是要进行模拟器运行的时候，客户端下载非常的慢。不过推荐得用不是。

<!--more-->

至于为什么用 RN，这又是另外一个话题了。场景是需要为一个 MUD 游戏做一个图形界面来，降低上手的难度。这样的话实际上就只是文字和一些文字框的游戏。开始是准备用比较属性的 Cocos Creator，不过感觉用来做 MUD 的客户端还是有点重了。另外就是其对于 UI 控件的实现是自己用 C 在底层实现的，实际上非常的不好用。

另外，还有点比较好的就是，RN 可以像网页那样用 Flex 进行布局，就容易多了。

# 项目建立

实际上，在[官方的文档](https://reactnative.dev/docs/getting-started)中，使用 expo 的步骤非常简单：

```sh
npm install -g expo-cli
expo init AwesomeProject

cd AwesomeProject
npm start # you ca
```

然后就可以在浏览器中进行我们的操作进行启动模拟器等了。

# Flex 布局

官方的定义是，FlexBox 是一个可以使用 Flex 算法来指定子视图的布局的组件。其设计的目的，是为了提供一个在不同屏幕尺寸间的一致性布局。

我们知道，我们的容器一般都是 *View*，所以我们只需要在 *View* 中指定 `flex` 就能使用其布局了。

[官方文档说明在此](https://reactnative.dev/docs/flexbox)。

# 轴
一般来说，定义了两个轴。横轴（row），纵轴（column）。Flex 所定义的行为就在两个轴上作用的。

# 属性列表

- flex：定义了当前视图如何在主轴上填充可用空间的。其是一个数值，定义了主轴上所占据空间的比例。有点类似与安卓中的 `layout_weight` 的意思，就是权重的概念。
- flexDirection：定义主轴的方向
	- `row` 左至右
	- `column` 上至下（**默认值）**。
	- `row-reverse` 从右至左。
	- `column-reverse` 由底而上。
- justifyContent：在**主轴**方向是如何对排列子视图。
	- `flex-start` 从开始对齐
	- `flex-end` 从尾部开始
	- `space-between` 子视图间的间隔距离相等进行排列
	- `center` 居中
	- `space-around` 与 `space-between` ，不过这个在边界间也会计算相等的距离。
	- `space-evenly` 这个和上个选项很相似，但是区别在哪里？
- alignItems：**次轴**如何排列子视图的。
	- `stretch` 拉伸子视图来填满容器的次轴的高度。不能设置子视图的高度为确切的数字
	- `flex-start`
	- `flex-end`
	- `center`
	- `baseline`。对准一个基线。
- alignSelf：改变子视图自身。
- alignContent：定义了**次轴方向**的行分布情况。这**只会在使用了 `flexWrap` 来包装子视图多行的情况。**。意思只有在子视图多行的情况生效。
	- `flex-start`
	- `flex-end`
	- `center`
	- `stretch `
	- `space-between`
	- `space-around`
- flexWrap：作用于 **主轴**。控制当子视图超过了 **主轴** 的尺寸的时候怎么办 。默认情况下，子视图是被强制设置为一行的（这会让有些元素看不到）。如果允许转行的话，那么超过尺寸的那些元素就会放新的行上，包装起来，这个时候 `alignContent ` 就会生效了。
- flexGrow
- flexShrink
- flexBasis
- width，height 宽高
	- `auto` 自动计算
	- `pixels`  绝对值
	- `percentage` 百分比
- position
	- `relative ` 默认值
	- `absolute ` 绝对位置

# 注意事项
---
title: Android-View的绘制过程
categories:
  - Android
date: 2019-04-21 10:22:02
updated: 2019-04-21 10:22:02
tags: 
  - Android
---

当一个 *Activity* 获取焦点的时候，其会被请求绘制其布局。Android 框架会控制绘制的过程，但是 *Activity* 必须提供其布局层级的根节点。

绘制从布局的根节点开始。其会被要求测试和汇制布局树。通过遍历布局树及渲染位于有效区域内的 *View* 来进行绘制。

按照顺序，每个 *ViewGroup* 负责请求其子 View 进行绘制（`draw()`方法），每个 *View* 负责绘制自己。因为布局树是预先定义的，这就意味着 父视图会在子视图之前（之下）进行绘制，兄弟视图则根据其出现的顺序而定。

> Android 框架并不会绘制在有效区域外的 View ，不过其也会关心绘制 View 的背景。
> 可以通过调用  View 的 `invalidate()` 方法来强制绘制。


# 测量

测量阶段通过方法 `measure(int, int)` 实现，其会由上至下遍历布局树。在这个递归遍历中，每个 *View* 都会将其尺寸规范向下推送。在测量阶段的结束，每个 *View* 都保存了其尺寸。

第二个阶段发生在 `layout(int, int, int, int)` 中，其也是由上至下的。在这个阶段中，每个父视图的责任是根据上一阶段测量出的个子视图的尺寸来定位其所有的子视图。

当一个 *View* 对象的 `measure()` 方法返回时，其 `getMeasuredWidth(), getMeasureHeight()` 的值必须被设置，其子视图的这两个方法的值也必须被设置。 *View* 测量后的宽度和高度必须满足其父视图给予的约束。这保证在测量阶段结束时，所有的父视图都会接受其子视图的尺寸。

一个父视图 *View* 可能会多次调用 `measure()` 方法。比如，当父视图以未指定的规格来测量子视图以知道子视图想要显示的尺寸，如果子视图返回的尺寸太大或太小的时候就会再次以实际的规格来调用  `measure()` 。

测量阶段使用两个类来进行沟通规格。*View* 使用 *ViewGroup.LayoutParams* 来告诉其父视图其想要被如何测量及定位。此类仅仅用来描述 *View* 想要其宽和高是什么样的。可以有下面三个值：

- 一个确定的值
- MATCH_PARENT 想要和父视图一样宽。
- WARP_CONTENT 只需要包围视图内容即可

*MeasureSpec* 对象用来从父到子向下推送规格。有三种形式：

- UNSPECIFIED 用来决定子视图的规格。例如，一个 *LinearLayout* 可能会宽为 *EXACTLY* 的 240，高为 *UNSPECIFIED* 来找出子视图希望有多高。
- EXACTLY 确切的值。子视图必须使用这个值，且子视图的后面也必须满足在这个值内。
- AT MOST： 最大值。子视图必须保证其及其子视图不超过这个值


# 布局阶段

为了初始化一个布局，使用 `requestLayout()`。
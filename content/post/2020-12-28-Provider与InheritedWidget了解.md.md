---
title: Provider与InheritedWidget了解.md
categories:
  - Flutter
date: 2020-12-28 14:33:10
updated: 2020-12-28 14:33:10
tags: 
  - Flutter
---

看状态管理，官方推荐的是 Provider，但是看起来有点不明所以，因此，想深入了了解一下这个机制。而不是用起来莫名其妙，出了问题也无法解决这样。

<!--more-->

# InheritedWidget

[Widget](https://flutter.cn/docs/development/ui/widgets-intro) 是 Flutter 的核心，Widget 描述了在当前的配置和状态下视图所应该呈现的样子。当 widget 的状态改变时，它会重新构建其描述（展示的 UI），框架则会对比前后变化的不同，以确定底层渲染树从一个状态转换到下一个状态所需的最小更改。而且， `Widget` 是 `Immutable` ，不可变的。也就是，在渲染过后，我们无法修改，我们只能操纵渲染或者不渲染一个 Widget ，而不能在渲染后进行修改。

耳语 Statefull 的组件，则是是根据状态来进行局部的重新渲染。



[InheritedWidget](https://api.flutter.dev/flutter/widgets/InheritedWidget-class.html) 是一个抽象类，它的定义是：高效地向渲染树向下传递信息的 Widget 的基类。

他的继承如下：

```
InheritedWidget -> ProxyWidget -> Widget
```

如果想要从 `build context` 获取最近的 `InheritedWidget` 的实现类，可以通过 `BuildContext.dependOnInheritedWidgetOfExactType`

当 `Inherited widgets` 用这种方式引用的时候，会在 `Inherited widgets` 自身状态改变的时候让消费者重建。

## 例子

```dart
 class FrogColor extends InheritedWidget {
   const FrogColor({
     Key? key,
     required this.color,
     required Widget child,
   }) : super(key: key, child: child);

   final Color color;

   static FrogColor of(BuildContext context) {
     final FrogColor? result = context.dependOnInheritedWidgetOfExactType<FrogColor>();
     assert(result != null, 'No FrogColor found in context');
     return result!;
   }

   @override
   bool updateShouldNotify(FrogColor old) => color != old.color;
 }
```

`FrogColor` 保留了一个字段 `color`，定义了一个静态方法 `of`，还有重写了 `updateShouldNotify`方法，用来告知框架，是否需要发出通知。





### of

对于 `BuildContext.dependOnInheritedWidgetOfExactType` 的约定是：在 `Inherited Widget` 上实现一个静态方法 `of` 来调用。这就允许我们的实现类定义一个当在范围内没有相应Widget 时的回退逻辑。在上面的例子中，如果确实没有找到  `FrogColor` ,那返回值就是 `null`，但也可以定义为一个其他值。

某些时候，`of` 可能会返回数据而不是 `Inherited Widget`；例如，返回一个 `Color` 而不是 `FrogColor`。

偶然地，`inherited widget ` 是一个其他类的实现细节，因此是私有的(private)。`of` 方法就会放在公有的类里面。如：`Theme` 被实现为一个 `StatelessWidget`，同时构建了一个私有的 `inherited widget;` `Theme.of` 会用 `BuildContext.dependOnInheritedWidgetOfExactType` 来查询这个私有的 `inherited widget` 然后返回  `ThemeData`。

## `of` 的调用

当调用 `of` 的时候，`context` 必须是 `InheritedWidget` 后代，这 就是说必须是树中位置必须在 `InheritedWidget` 下。 

下面这个例子的 `of` 调用，`context `是 来源于 `Builder`，是 `FrogColor` 的后代，因此是有效的。

```dart
 class MyPage extends StatelessWidget {
   @override
   Widget build(BuildContext context) {
     return Scaffold(
       body: FrogColor(
         color: Colors.green,
         child: Builder(
           builder: (BuildContext innerContext) {
             return Text(
               'Hello Frog',
               style: TextStyle(color: FrogColor.of(innerContext).color),
             );
           },
         ),
       ),
     );
   }
 }
```

而下面这个例子是无效了：

```dart
class MyOtherPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: FrogColor(
        color: Colors.green,
        child: Text(
          'Hello Frog',
          style: TextStyle(color: FrogColor.of(context).color),
        ),
      ),
    );
  }
}
```

# Provider

[Provider](https://pub.dev/packages/provider) 是一个对 `InheritWidget` 的包装，让我们更加容易使用和复用。

```
Provider -> InheritedProvider  -> SingleChildStatelessWidget
```

实际上，根据官方定义：

>  InheritedProvider 是 InheritedWidget 的一个通用实现。因此，他拥有和 InheritedWidget 一样的特性，可以向下传递信息。
>
> 这个组件的所有后代都可以通过 `Provider.of` 来获取其中包含的 `value`
>
> 不要直接使用 InheritedProvider

> Provider 管理其提供的 `value` 的生命周期，这是通过一对 `Create, Dispose` 实现的。
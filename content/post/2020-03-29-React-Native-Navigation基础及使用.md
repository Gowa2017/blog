---
title: React-Native-Navigation基础及使用
categories:
  - [JavaScript]
  - [React]
date: 2020-03-29 22:41:44
updated: 2020-03-29 22:41:44
tags:
  - JavaScript
  - React Native
---

React-Redux，React Native Navigation 可以说是用 React Native 进行开发的标配啊。在将 Redux 了解之后，所以就来看一下这一个导航库是如何工作的了。官方现在强推的是 [https://reactnavigation.org](https://reactnavigation.org)

<!--more-->

# 安装

对于[官方文档](https://wix.github.io/react-native-navigation/#/docs/Installing?id=npm)，对于使用的 RN 版本是 0.6 以上的，可以用直接 `link` 的形式来安装。

```sh
npm install --save react-native-navigation
react-native link react-native-navigation
```

但事实上这个是安装不成功的，所以呢，还是老老实实的手动进行安装吧（主要是针对安卓的不好自动安装完成，对于 ios 的，使用 pod 的就很容易了）。

## IOS

使用 CocoaPods 会在 _PodFIle.lock_ 里面增加 RNN 相关的内容，我们执行 `cd ios && pod install` 就行了。

另外，这样是会报错的，我们还需要修改 _AppDelegate.m_：

```objective-c
#import <React/RCTBridge.h>
#import <React/RCTBundleURLProvider.h>
#import <React/RCTRootView.h>

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
#if DEBUG
  InitializeFlipper(application);
#endif

  RCTBridge *bridge = [[RCTBridge alloc] initWithDelegate:self launchOptions:launchOptions];
  RCTRootView *rootView = [[RCTRootView alloc] initWithBridge:bridge
                                                   moduleName:@"GG"
                                            initialProperties:nil];

  rootView.backgroundColor = [[UIColor alloc] initWithRed:1.0f green:1.0f blue:1.0f alpha:1];

  self.window = [[UIWindow alloc] initWithFrame:[UIScreen mainScreen].bounds];
  UIViewController *rootViewController = [UIViewController new];
  rootViewController.view = rootView;
  self.window.rootViewController = rootViewController;
  [self.window makeKeyAndVisible];
  return YES;
}
```

改动最后应该是看起来这样的，我们实际上就修改这个启动方法就行了：

```objective-c
#import <React/RCTBridge.h>
#import <React/RCTBundleURLProvider.h>
#import <React/RCTRootView.h>
#import <ReactNativeNavigation/ReactNativeNavigation.h>

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
#if DEBUG
  InitializeFlipper(application);
#endif

  NSURL *jsCodeLocation = [[RCTBundleURLProvider sharedSettings] jsBundleURLForBundleRoot:@"index" fallbackResource:nil];
  [ReactNativeNavigation bootstrap:jsCodeLocation launchOptions:launchOptions];
  return YES;
}
```

这样的话就能跑起来了。

## Android

安卓照着官方教程全部改就行了：[https://wix.github.io/react-native-navigation/#/docs/Installing?id=android](https://wix.github.io/react-native-navigation/#/docs/Installing?id=android)

# 使用

## 基础

### [registerComponent(screenID, generator)](https://wix.github.io/react-native-navigation/#/docs/Usage?id=registercomponentscreenid-generator)

我们应用中的每个屏幕组件都必须用一个独一无二的名字进行注册。组件本身的话是一个传统的 React 组件，继承自 `React.component` 或者 `React.PureComponent` 。它也可以是一个用来提供上下文的 **HOC** （或者是一个 Redux Store）。这也 RN 的 `AppRegistry.registerComponent()` 相类似：

```js
Navigation.registerComponent(
  `navigation.playground.WelcomeScreen`,
  () => WelcomeScreen
);
```

### registerAppLaunchedListener(callback)

这个事件在 APP 启动后调用。就正是我们用所期待的布局（使用 `setRoot`）来初始化 APP 的地方。这将会建立原生的布局层级，通过名字来将 React 组件加载到 _component_ 中。

在此之后，APP 就已经准备好与用户进行交互了。（注意不要在 `registerAppLaunchedListener` 的回调被执行前就调用 `setRoot()`）

```js
Navigation.events().registerAppLaunchedListener(() => {
  Navigation.setRoot({
    root: {
      component: {
        name: "navigation.playground.WelcomeScreen",
      },
    },
  });
});
```

这里我们使用了一个 `component` 类型的布局，这会渲染一个 React 的组件但是不允许我们导航到其他地方。

如果我们想要进行导航，使用 _stack_ 布局类型：

```javascript
Navigation.events().registerAppLaunchedListener(() => {
  Navigation.setRoot({
    root: {
      stack: {
        children: [
          {
            component: {
              name: "navigation.playground.WelcomeScreen",
            },
          },
        ],
      },
    },
  });
});
```

## 与 Redux 结合

官方文档的原型是：

```js
Navigation.registerComponent(
  "navigation.playground.ReduxScreen",
  () => (props) => (
    <Provider store={reduxStore}>
      <ReduxScreen {...props} />
    </Provider>
  ),
  () => ReduxScreen
);
```

我们将其抽象一下，方便我们进行注入：

```js
import Login from "./components/Login";
import { Provider } from "react-redux";
import { Navigation } from "react-native-navigation";
import React from "react"; // 这里不可少，我就是之前少了这里，所以一直会报错

const ReduxProvider = (Component, store) => {
  return (props) => (
    <Provider store={store}>
      <Component {...props} />
    </Provider>
  );
};

const register = (id, Component, store) => {
  Navigation.registerComponent(
    id,
    () => ReduxProvider(Component, store),
    () => Component
  );
};

export default (store) => {
  register("login", Login, store);
};
```

这样我们就可以在 index.js 里面注入了：

```js
import registerScreens from "./screens";
import store from "./redux/store"; // 我们自己的 Store 逻辑

registerScreens(store);
```

## 屏幕生命周期

`componentDidAppear(), componentDidDisappear` 是属于 RNN 的两个导航生命周期函数，他们会在组件出现和消失的时候被调用（当然，先要通过 `Navigation.events().bindComponent(this)` 进行绑定）。

这和 RN 的 `componentDidMount(), componentWillUnmount()` 相似，不同的是，上面两个生命周期函数代表了用户是否实际看到了组件——而不仅仅是是否已经被挂载。因为 RNN 的这种对性能的优化，一个组件作为布局的一部分将会被尽快的 `mounted`——但并不总是可见（比如其他的屏幕内容 _push_ 到了顶部）

有很多这样的应用场景：比如在组件显示在屏幕上的时候开始或者停止动画。

> They are implemented by iOS's viewDidAppear/viewDidDisappear and Android's ViewTreeObserver visibility detection

要使用他们，只需要在我们的组件中像其他的 React 生命周期函数一样进行实现就行了，同时将此组件绑定到导航事件模块，此导航事件模块将会自动调用这两个函数：

```js
class LifecycleScreenExample extends Component {
  constructor(props) {
    super(props);
    Navigation.events().bindComponent(this);
    this.state = {
      text: "nothing yet",
    };
  }

  componentDidAppear() {
    this.setState({ text: "componentDidAppear" });
  }

  componentDidDisappear() {
    alert("componentDidDisappear");
  }

  navigationButtonPressed({ buttonId }) {
    // a navigation-based button (for example in the topBar) was clicked. See section on buttons.
  }

  componentWillUnmount() {
    alert("componentWillUnmount");
  }

  render() {
    return (
      <View style={styles.root}>
        <Text style={styles.h1}>{`Lifecycle Screen`}</Text>
        <Text style={styles.h1}>{this.state.text}</Text>
      </View>
    );
  }
}
```

# 布局 Layout

RNN 的布局类型主要有几种：

- stacks
- tabs
- drawers

我们可以对齐进行任何组合，这其中会出现一些奇怪的问题，但是并不会报错。这个的话就及时给开发者提 issues 就行了。

## component

只容纳单独的一个 React 组件：

```js
const component = {
  id: "component1", // Optional, Auto generated if empty
  name: "Your registered component name",
  options: {},
  passProps: {
    text: "This text will be available in your component.props",
  },
};
```

## stack

支持任意类型的子布局。一个 stack 可能会被多个 Screen 来进行初始化，最后一个 Screen 会被放在最上面。

类似于 Android 的回退栈。

```js
const stack = {
  children: [
    {
      component: {},
    },
    {
      component: {},
    },
  ],
  options: {},
};
```

## bottomTabs

底部标签

```js
const bottomTabs = {
  children: [
    {
      stack: {
        children: [],
        options: {
          bottomTab: {
            text: "Tab 1",
            icon: require("../images/one.png"),
          },
        },
      },
    },
    {
      component: {
        name: "secondTabScreen",
        options: {
          bottomTab: {
            text: "Tab 2",
            icon: require("../images/two.png"),
          },
        },
      },
    },
  ],
  options: {},
};
```

> 在 Android 中， icon 是必须的

### 代码选择标签

已选中的索引是一个风格属性，它可以被 `mergeOption` 命令进行更新。我们更新 BottomTabs 的属性，传递 BottomTabs 的 _componentId_ 或者其子视图的 _componentID_。

```js
const bottomTabs = {
  id: 'BottomTabsId',
  children: [
    {
      component: {
        name: 'FirstScreen',
        options: { ... }
      }
    },
    {
      component: {
        id: 'SecondScreenId',
        name: 'SecondScreen',
        options: { ... }
      }
    }
  ]
}
```

**通过索引选择**

下面的例子中会选择第二个子组件，我们传递的是 BottomTabs 的 ID，但是我们也可以传递任意子视图的 ID。

```js
Navigation.mergeOptions("BottomTabsId", {
  bottomTabs: {
    currentTabIndex: 1,
  },
});
```

**通过子组件来选择**

```js
Navigation.mergeOptions(this.props.componentId, {
  bottomTabs: {
    currentTabId: this.props.componentId,
  },
});
```

**改变可见性**

_visible_ 用来控制 bottomTabs 的可见性。

在 **Android**，可见性可以动态的使用 _mergeOptions_ 命令来触发。当隐藏 BottomTabs 的时候，应该指定 `drawBehind: true`，这就能让之前为 bottomTabs 分配的区域可以被渲染成其他内容了。

```js
Navigation.mergeOptions(componentId, {
  bottomTabs: {
    visible: false,
    ...Platform.select({ android: { drawBehind: true } }),
  },
});
```

## sideMenu

## splitView(iOS)

## Example

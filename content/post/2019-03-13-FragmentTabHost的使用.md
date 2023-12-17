---
title: FragmentTabHost的使用
categories:
  - Android
date: 2019-03-13 22:56:35
updated: 2019-03-13 22:56:35
tags: 
  - Android
---
为了实现标签页切换。之前用过 TabLayout 但是似乎现在都淘汰了？所以看了一下官方介绍的 FragmentTabHost 看起来很简单的，但是想要定制的话就发现了些问题。

<!--more-->

事实上，FragmentTabHost 就是对 TabHost 类的一个继承，只不过，其是用 Fragment 来显示 Tab 内容。所以在查看相关知识例子的时候，谷歌看到大多是关于 TabHost的。

# 布局

布局很简单，定义一个 FragmentTabHost，然后再用一个装载内容的 FrameLayout 视图就行了。


```xml
    <android.support.v4.app.FragmentTabHost
        android:id="@+id/fth"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_below="@id/ll_status3" />

    <FrameLayout
        android:id="@+id/fly"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />
```

# 代码应用

```java
        fth.setup(this, getSupportFragmentManager(), R.id.fly); // 设置内容容器ID 为 R.id.fly
        // 添加两个 Tab setIndicator()可以用 View 作为参数 
        fth.addTab(fth.newTabSpec("all").setIndicator("全程"), ComRegisterMainFragmentAllElectronic.class, null);
        fth.addTab(fth.newTabSpec("half").setIndicator("半程"), ComRegisterMainFragmentHalfElectronic.class, null);
```

就是这么简单。

但我遇到的问题是，我想要自定义 Tab 显示的颜色，背景，字体的时候没有找到好用的方法。其只提供了 View 相关的方法，关于一些具体实现 如 TextView 的方法无法获取到呢。

# 源码追踪

## FragmentTabHost.setup()

```java
    public void setup(Context context, FragmentManager manager, int containerId) {
        ensureHierarchy(context);  // Ensure views required by super.setup()
        super.setup();
        mContext = context;
        mFragmentManager = manager;
        mContainerId = containerId;
        ensureContent();
        mRealTabContent.setId(containerId);

        // We must have an ID to be able to save/restore our state.  If
        // the owner hasn't set one at this point, we will set it ourselves.
        if (getId() == View.NO_ID) {
            setId(android.R.id.tabhost);
        }
    }
```

## TabHost.setup()

从这个方法中可以看到：

1. TabHost 使用  *com.android.internal.R.id.tabs* 作为 tabs 的容器。
2. 使用  *com.android.internal.R.id.tabcontent* 作为标签内容的容器。而且必须是 *FrameLayout*

但这些内容从哪里来呢？在 FragmentTabHost 中已经默认为我们进行了设置。

```java
    public void setup() {
        mTabWidget = findViewById(com.android.internal.R.id.tabs);
        if (mTabWidget == null) {
            throw new RuntimeException(
                    "Your TabHost must have a TabWidget whose id attribute is 'android.R.id.tabs'");
        }

        // KeyListener to attach to all tabs. Detects non-navigation keys
        // and relays them to the tab content.
        mTabKeyListener = new OnKeyListener() {
            public boolean onKey(View v, int keyCode, KeyEvent event) {
                switch (keyCode) {
                    case KeyEvent.KEYCODE_DPAD_CENTER:
                    case KeyEvent.KEYCODE_DPAD_LEFT:
                    case KeyEvent.KEYCODE_DPAD_RIGHT:
                    case KeyEvent.KEYCODE_DPAD_UP:
                    case KeyEvent.KEYCODE_DPAD_DOWN:
                    case KeyEvent.KEYCODE_ENTER:
                        return false;

                }
                mTabContent.requestFocus(View.FOCUS_FORWARD);
                return mTabContent.dispatchKeyEvent(event);
            }

        };

        mTabWidget.setTabSelectionListener(new TabWidget.OnTabSelectionChanged() {
            public void onTabSelectionChanged(int tabIndex, boolean clicked) {
                setCurrentTab(tabIndex);
                if (clicked) {
                    mTabContent.requestFocus(View.FOCUS_FORWARD);
                }
            }
        });

        mTabContent = findViewById(com.android.internal.R.id.tabcontent);
        if (mTabContent == null) {
            throw new RuntimeException(
                    "Your TabHost must have a FrameLayout whose id attribute is "
                            + "'android.R.id.tabcontent'");
        }
    }
```

## FragmentTabHost.ensureHierarchy()

```java
    private void ensureHierarchy(Context context) {
        // If owner hasn't made its own view hierarchy, then as a convenience
        // we will construct a standard one here.
        if (findViewById(android.R.id.tabs) == null) {
            LinearLayout ll = new LinearLayout(context);
            ll.setOrientation(LinearLayout.VERTICAL);
            addView(ll, new FrameLayout.LayoutParams(
                    ViewGroup.LayoutParams.MATCH_PARENT,
                    ViewGroup.LayoutParams.MATCH_PARENT));

            TabWidget tw = new TabWidget(context);
            tw.setId(android.R.id.tabs);
            tw.setOrientation(TabWidget.HORIZONTAL);
            ll.addView(tw, new LinearLayout.LayoutParams(
                    ViewGroup.LayoutParams.MATCH_PARENT,
                    ViewGroup.LayoutParams.WRAP_CONTENT, 0));

            FrameLayout fl = new FrameLayout(context);
            fl.setId(android.R.id.tabcontent);
            ll.addView(fl, new LinearLayout.LayoutParams(0, 0, 0));

            mRealTabContent = fl = new FrameLayout(context);
            mRealTabContent.setId(mContainerId);
            ll.addView(fl, new LinearLayout.LayoutParams(
                    LinearLayout.LayoutParams.MATCH_PARENT, 0, 1));
        }
    }
```

# 解决办法

使用 newTabSpec("all").setIndicator( View ) 来实现我们自定义的视图。
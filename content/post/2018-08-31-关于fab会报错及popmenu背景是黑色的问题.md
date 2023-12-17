---
title: 关于fab会报错及popmenu背景是黑色的问题
categories:
  - Android
date: 2018-08-31 18:51:45
updated: 2018-08-31 18:51:45
tags: 
  - Android
---
遇到一个问题，就是我以前用的是Activity继承的是 FragmentActivit，然后可以使用 getActivity()函数。而我换成AppCompactActivity后，就不能使用这个了，我就打算用this,或者getBaseContext来做上下文，结果就是popMenu居然背景变成黑色了，换了多个主题还是一样。后面发现了问题。
<!--more-->

# Context
大多数时候，我们的View都需要一个上下文。当我们在建立一个View或者扩张一个布局文件的时候，都需要一个上下文。我们需要的屏幕信息，风格信息，都可以从Context内获取。

上面的问题，应该是我获取的上下文不对，所以会产生，与整体风格不一致的问题。

由于我是在 按钮的回调函数内，建立的菜单。很明显，在内部类里，我们时不能直接使用 `this`的，这个时候，正确的使用方式应该是`MainActivity.this`这样，换成这样的方式后，果然就正常了。

# ApplicationContext
这个上下文在应用存在期间，一直可用，所以对于生命周期长，或无法获取上下文的时候，可以使用这个来作为上下文。

在CodePath上有一篇文章进行了描述上下文：[https://guides.codepath.com/android/Using-Context](https://guides.codepath.com/android/Using-Context)

# Context 用来做什么
下面是一些例子。
## 显式启动一个组件
```java
// Provide context if MyActivity is an internal activity.
Intent intent = new Intent(context, MyActivity.class);
startActivity(intent);
```
当显式启动一个组件时候，我们需要两部分信息：

* 包名，标识了包含这个组件的应用。
* 要启动组件的 完整引用Java类名

如果是启动一个内部组件，context 参数可以传递为当前应用的包名，可通过 `context.getPackageName()`获取。

## 建立视图（View）

```java
TextView textView = new TextView(context);
```

Context 包含了视图需要的如下信息：

* 设备尺寸，已经在 dp,sp 转换为 px的规格
* 风格属性
* OnClick 属性引用的 Activity

## 扩张一个 XML 布局文件

当我们要把一个XML布局文件扩展到内存的时候，我们使用  Context 来获取 `LayoutInflater`。

```java
// A context is required when creating views.
// 建立视图的时候，会需要一个上下文
LayoutInflater inflater = LayoutInflater.from(context);
inflater.inflate(R.layout.my_layout, parent);
```

## 发送本地广播

当我们发送广播或者注册广播接收者的时候，用Context 来获取 `LocalBroadcastManager`。

```java
// The context contains a reference to the main Looper, 
// which manages the queue for the application's main thread.
// context 包含了对 主 Looper 的引用
// Looper 管理应用主线程的队列。
Intent broadcastIntent = new Intent("custom-action");
LocalBroadcastManager.getInstance(context).sendBroadcast(broadcastIntent);
```

## 获取系统服务
要想从一个应用发送通知，需要使用到 `NotificationManager` 系统服务。

```java
// Context objects are able to fetch or start system services.
NotificationManager notificationManager = 
    (NotificationManager) getSystemService(NOTIFICATION_SERVICE);

int notificationId = 1;

// Context is required to construct RemoteViews
Notification.Builder builder = 
    new Notification.Builder(context).setContentTitle("custom title");

notificationManager.notify(notificationId, builder.build());
```

# 应用上下文 VS Activity上下文
主题和风格多数时候都是使用在应用级别，但他们也可以在Activity级别进行使用。在这种方式下，某个Activity就可以和其他的Activity有不同的风格（比如，需要隐藏ActivonBar等等）。`AndroidManifest.xml`中在  Application中有一个属性 `android:theme`，同样，这个属性可以用在 Activity上：

```xml
<application
       android:allowBackup="true"
       android:icon="@mipmap/ic_launcher"
       android:label="@string/app_name"
       android:theme="@style/AppTheme" >
       <activity
           android:name=".MainActivity"
           android:label="@string/app_name"
           android:theme="@style/MyCustomTheme">
```

因为这个原因，所以我们要明白，我们会拥有一个 Application Context 和 Activity Context，上下文存在于他们对应的生命周期中。大多数 Views应该传递一个 Activity Context 来获得需要的 主题，风格，规格等信息。如果 Activity 并没有使用主题，那么默认使用 Application使用的。

大多数情况下，应该使用 Activity Context。正常情况下，Java 中的 `this` 引用了一个类的实例，因此可是在一个Activity中来用在 需要 Context 的地方。

```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);  
        Toast.makeText(this, "hello", Toast.LENGTH_SHORT).show();
    }
}
```

## 匿名函数
当使用匿名函数来实现  监听器时， Java中的 `this` 使用在最近被声明的类上。在这种情况下，外层的类 `MainActivity` 必须指开来表明其引用的是 MainActivity 实例。

```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {

        TextView tvTest = (TextView) findViewById(R.id.abc);
        tvTest.setOnClickListener(new View.OnClickListener() {
              @Override
              public void onClick(View view) {
                  Toast.makeText(MainActivity.this, "hello", Toast.LENGTH_SHORT).show();
              }
          });
        }
    }
```    

## 适配器
### 数组适配器
当为一各ListView构造适配器的时候， 典型的会在 布局扩张过程中调用 `getContext()`。这个方法，经常使用初始化 ArrayAdapter 的Context。

```java
     if (convertView == null) {
        convertView = 
            LayoutInflater
                .from(getContext())
                .inflate(R.layout.item_user, parent, false);
     }       
```

如果你使用  Application Context来初始化 ArrayAdapter，你会发现，主题/风格并没有被应用。因此，确定你传递的是一个 Activity Context。

### RecyclerView Adapter

```java
public class MyRecyclerAdapter extends RecyclerView.Adapter<MyRecyclerAdapter.ViewHolder> {

    @Override 
    public ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        View v = 
            LayoutInflater
                .from(parent.getContext())
                .inflate(R.layout.itemLayout, parent, false);
        
        return new ViewHolder(v);
    }

    @Override
    public void onBindViewHolder(ViewHolder viewHolder, int i) {
        // If a context is needed, it can be retrieved 
        // from the ViewHolder's root view.
        Context context = viewHolder.itemView.getContext();

        // Dynamically add a view using the context provided.
        if(i == 0) {
            TextView tvMessage = new TextView(context);
            tvMessage.setText("Only displayed for the first item.")

            viewHolder.customViewGroup.addView(tvMessage);
        }
    }

   public static class ViewHolder extends RecyclerView.ViewHolder {
       public FrameLayout customViewGroup;

       public ViewHolder(view imageView) {
           // Very important to call the parent constructor
           // as this ensures that the imageView field is populated.
           super(imageView);
           
           // Perform other view lookups.
           customViewGroup = (FrameLayout) imageView.findById(R.id.customViewGroup);
       }
   }
}
```

ArrayAdapter 需要一个上下文传递到其构造器，而`RecyclerView.Adapter`不需要。因为其会在扩张必要的时候，可以从父布局那里推测出来正确的上下文。

相关的 `RecyclerView ` 总是会把其自身作为 `RecyclerView.Adapter.onCreateViewHolder()`的父视图传递。

如果在 `onCreateViewHolder()`外需要使用上下文，而这里又有一个`ViewHolder `实例变量，就可以通过  `viewHolder.itemView.getContext()`来获取上下文。在基本的 ViewHolder类中，`itemView` 是一个 `public, non-null, final`字段。

## 避免内存泄漏
Application Context 典型的用在单例实例需要被创建的时候，比如一个自定义的管理类需要 Context 信息来获取对系统服务的访问，但是其会在多个Activity中使用。因为对Activity Context的引用会导致那个Activity 在其停止后内存不会被释放，这就必须使用  Application Context。

下面的例子中，如果被存储的 *context* 是一个 Activity或Service，其在被安卓系统销毁后，并不会被垃圾回收器回收。因为 `CustomManager ` 持有了对它的一个静态引用。

```java

public class CustomManager {
    private static CustomManager sInstance;

    public static CustomManager getInstance(Context context) {
        if (sInstance == null) {

            // This class will hold a reference to the context
            // until it's unloaded. The context could be an Activity or Service.
            sInstance = new CustomManager(context);
        }

        return sInstance;
    }

    private Context mContext;

    private CustomManager(Context context) {
        mContext = context;
    }
}
```

## 适当的存储一个Context:使用  Application Context

为了避免内存泄漏，不要在其生命周期外持有对它的引用。检查所有你的背景线程，过起的handlers，及内部类，看看他们是不是会持有上下文引用。

正确的方式是在`CustomManager.getInstance()`中存储 Application Context。应用上下文是单例的，且其与应用进程的生命周期相绑定。

当需要一个在组件的生命周期外引用其上下文时，使用 应用上下文。

```java
public static CustomManager getInstance(Context context) {
    if (sInstance == null) {

        // When storing a reference to a context, use the application context.
        // Never store the context itself, which could be a component.
        sInstance = new CustomManager(context.getApplicationContext());
    }

    return sInstance;
}
```
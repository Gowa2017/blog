---
title: 谷歌的todo-mvp-dagger例子探究
categories:
  - Java
date: 2019-01-15 17:22:26
updated: 2019-01-15 17:22:26
tags: 
  - Java
  - Android
  - Dagger
---

根据 Dagger 在官方文档上介绍的在安卓中使用 Dagger 的方式，想看一下谷歌的例子是怎么做的。但是发现，这和其文档上说明的并不一致，所以来深入的看一下。

[官方文档](https://google.github.io/dagger/android)

[中文版](https://gowa.club/Java/Dagger%E4%B8%8E%E5%AE%89%E5%8D%93.html)

<!--more-->

# Activty 注入

按照官方文档的说明，当我们在  Activity 的 onCreate 函数内调用  `AndroidInjection.inject()` 时，我们可以从 `Application` 处获得一个 `DispatchingAndroidInjector<Activity>`，并将调用 Activity 作为参数传递来调用  `inject(Activity)`。 

`DispatchingAndroidInjector` 会查找 我们传递的 Activity 类的 `AndroidInjector.Factory`（实际上就是我们自己定义的 *YourActivitySubcomponent.Builder*），并用它来建立 `AndroidInjector`（实际就是 *YourActivitySubcomponent*），最终调用 `inject(YourActivity)`。

# DaggerAppCompatActivity

这个类，签名为

```java
public abstract class DaggerAppCompatActivity extends AppCompatActivity
    implements HasFragmentInjector, HasSupportFragmentInjector {}
```

我们所有的 Activity 都继承自它。在它的 `onCreate()` 方法中，确实调用了 `AndroidInjection.inject(this);`方法。

按照上面介绍的工作流程。我们来理一理。

# AndroidInjection.inject(this);

这个类，这个方法，都是由 Dagger 已经实现的：

```java
  public static void inject(Activity activity) {
    checkNotNull(activity, "activity");
    Application application = activity.getApplication();
    if (!(application instanceof HasActivityInjector)) {
      throw new RuntimeException(
          String.format(
              "%s does not implement %s",
              application.getClass().getCanonicalName(),
              HasActivityInjector.class.getCanonicalName()));
    }

    AndroidInjector<Activity> activityInjector =
        ((HasActivityInjector) application).activityInjector();
    checkNotNull(activityInjector, "%s.activityInjector() returned null", application.getClass());

    activityInjector.inject(activity);
  }
```

其基本的过程就是：通过 Activity 获取到 应用级别的上下文 Application。然后通过 应用上下文 实现的接口 `HasActivityInjector` 的方法 `activityInjector()` 来获取一个 activityInjector。

```java
  @Override
  public DispatchingAndroidInjector<Activity> activityInjector() {
    return activityInjector;
  }
```
关于 activityInjector 是如何初始化的，我们下一节看。

# injectIfNecessary

DaggerApplication 在应用启动的时候，会尝试是否需要注入的检查，如果需要注入，那么就初始化相应的内容.

```java
  public void onCreate() {
    super.onCreate();
    injectIfNecessary();
  }
  
    private void injecAtIfNecessary() {
    if (needToInject) {
      synchronized (this) {
        if (needToInject) {
          @SuppressWarnings("unchecked")
          AndroidInjector<DaggerApplication> applicationInjector =
              (AndroidInjector<DaggerApplication>) applicationInjector();
          applicationInjector.inject(this);
          if (needToInject) {
            throw new IllegalStateException(
                "The AndroidInjector returned from applicationInjector() did not inject the "
                    + "DaggerApplication");
          }
        }
      }
    }
  }
```

哈，也是通过 Application 来注入自身。`AndroidInjector<DaggerApplication> applicationInjector =
              (AndroidInjector<DaggerApplication>) applicationInjector();` 是一个抽象方法，是我们的 Application 进行实现的：
              
```java
    @Override
    protected AndroidInjector<? extends DaggerApplication> applicationInjector() {
        return DaggerAppComponent.builder().application(this).build();
    }
```

这个就很明了。


```java
  @Inject DispatchingAndroidInjector<Activity> activityInjector;
  @Inject DispatchingAndroidInjector<BroadcastReceiver> broadcastReceiverInjector;
  @Inject DispatchingAndroidInjector<Fragment> fragmentInjector;
  @Inject DispatchingAndroidInjector<Service> serviceInjector;
  @Inject DispatchingAndroidInjector<ContentProvider> contentProviderInjector;
```

在 DaggerAppliction 内是定义了这些几大类型的注入器，这些，都是需要我们自己实现的 应用级别的注入器来实现的。

那么，必然，他会在 AppComponent 内定义这些注入的实现方式。

# AppComponent

```java 
@Singleton
@Component(modules = {TasksRepositoryModule.class,
        ApplicationModule.class,
        ActivityBindingModule.class,
        AndroidSupportInjectionModule.class})
public interface AppComponent extends AndroidInjector<ToDoApplication> {

    TasksRepository getTasksRepository();

    // Gives us syntactic sugar. we can then do DaggerAppComponent.builder().application(this).build().inject(this);
    // never having to instantiate any modules or say which module we are passing the application to.
    // Application will just be provided into our app graph now.
    @Component.Builder
    interface Builder {

        @BindsInstance
        AppComponent.Builder application(Application application);

        AppComponent build();
    }
}
```

其中，用到的模块 `AndroidInjectionModule` 这个是由 Dagger 来实现的。目的就是为了注入 Application 上下文保存的那些 关于 Activty 等五大类型的注入分发器。注入分发器就是为了根据类型来寻找指定的注入器。



```java
@Beta
@Module
public abstract class AndroidInjectionModule {
  @Multibinds
  abstract Map<Class<? extends Activity>, AndroidInjector.Factory<? extends Activity>>
      activityInjectorFactories();

  @Multibinds
  abstract Map<Class<? extends Fragment>, AndroidInjector.Factory<? extends Fragment>>
      fragmentInjectorFactories();

  @Multibinds
  abstract Map<Class<? extends Service>, AndroidInjector.Factory<? extends Service>>
      serviceInjectorFactories();

  @Multibinds
  abstract Map<
          Class<? extends BroadcastReceiver>, AndroidInjector.Factory<? extends BroadcastReceiver>>
      broadcastReceiverInjectorFactories();

  @Multibinds
  abstract Map<
          Class<? extends ContentProvider>, AndroidInjector.Factory<? extends ContentProvider>>
      contentProviderInjectorFactories();

  private AndroidInjectionModule() {}
}
```

此模块是采取的 @MultiBinds 的方式来注解方法的。其实到一步，我们都不必关心了，都是由 Dagger 来实现了。

# 总结

1. 我们只需要定义一个 Component （应用级别的），将我们需要要使用的 Module 传递给它。同时，还要加上 Dagger 自己实现的模块 AndroidSupportInjectionModule。
2. Dagger 会根据我们传递的 Module 自己组织相关需要的注解分发器。而后根据对应的参数就能进行注入了。

# 深入

关于在 AppComponent 中使用的 Module 还有讲究哦。比如我们就以 ActivityBindingModule.class 为例来看看。

```java
/**
我们希望 dagger.android ActivityBindingModule 所在的父组件（在我们的例子中是 AppComponent） 中建立一个子组件。
这样做的美妙之处在于，我们不需要告诉  AppComponent 所有的字组件，也不需要这些子组件 AppComponent 存在。
我们也告诉 Dagger.android 生成的子组件应该包括对应的 Module，同时要注意到注解 @ActivityScoped。
当 Dagger.android 注解处理器工作的时候，其会产生 4 个子组件。
@ContributesAndroidInjector 注解：为方法返回的类型产生一个 AndroidInjector。injector 用 dagger.SubComponent 实现，
其将会是 dagger.Modlue 组件的子组件。此注解必须在一个 Module 内的 抽象方法上，此抽象方法返回一个具体的安卓框架类型。
方法也没有参数。
@ActivityScoped 在 Dagger 中，没有范围的组件不能依赖于有范围的组件。因为 AppComponent 是一个有范围的组件，(Singleton），
所以我们建立一个范围来给所有的 Fragment 使用。
一个有范围的组件，其子组件不能拥有相同的范围。
*/
@Module
public abstract class ActivityBindingModule {
    @ActivityScoped
    @ContributesAndroidInjector(modules = TasksModule.class)
    abstract TasksActivity tasksActivity();

    @ActivityScoped
    @ContributesAndroidInjector(modules = AddEditTaskModule.class)
    abstract AddEditTaskActivity addEditTaskActivity();

    @ActivityScoped
    @ContributesAndroidInjector(modules = StatisticsModule.class)
    abstract StatisticsActivity statisticsActivity();

    @ActivityScoped
    @ContributesAndroidInjector(modules = TaskDetailPresenterModule.class)
    abstract TaskDetailActivity taskDetailActivity();
}
```

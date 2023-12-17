---
title: RxJava使用Schedulers时进行的线程切换
categories:
  - RxJava
date: 2019-05-28 21:54:50
updated: 2019-05-28 21:54:50
tags: 
  - RxJava
  - Java
  - Android
---
默认情况下，RxJava 是单线程的。所谓单线程指的是，Observable 及所有的操作符都会我们调用 Subscribe 方法的线程上完成。我们可以通过 SubscribeOn 操作符来改变这一行为，通过用此操作符来指定 Observable 在不同的 Scheduler 上进行执行工作，而 `ObserveOn()` 用来告诉  Observable 往哪个 Scheduler 发送通知。

<!--more-->

对于 `SubscribeOn()` 来说，其指定 Observable 在哪个线程开始执行，而无论我们在才操作符链中的哪个位置调用 `SubscribeOn()`。

`ObserveOn()` 正好相反，其会影响在其后的操作符执行的线程。因此，我们可能会多次调用 `ObserveOn()`。

在进行研究之前，我们先要了解 Scheduler 是什么东西。

# Scheduler

根据 [JavaDoc](http://reactivex.io/RxJava/javadoc/io/reactivex/Scheduler.html) 的定义：

Scheduler 指定了一个 API 用以调度以 *Runnable* 形式提供的工作单元，这些 *Runnable* 可能是会立刻执行、延迟一段时间或者周期性的重复；Scheduler 也代表在异步界限上的抽象：保证这些工作单元会被一些底层的任务执行方案以统一的属性和保证所执行（如自定义 Thread，event loop，Executor 或 Actor 系统），而不论底层的执行方案具体是怎么样的。

*Scheduler* 中的 *Scheduler.Worker* 可以通过  `createWorker()` 方法来建立，它们运行在相互隔离的情况下调度多个 *Runnable*。每个 *Worker* 中的多个 *Runnable* 保证会非交叉顺序执行。非延迟的 *Runnable* 保证会以一个 FIFO 的形式执行，但这可能会与有延迟的 *Runnable* 相重叠。可以用 `Disposable.dispose()` 方法来取消当前 *Worker* 中的任务，而不会影响其他的 *Worker* 实例。

对于方法 `scheduleDirect(Runnable), Scheduler.Worker.schedule(Runnable)` 的实现中鼓励我们调用 `RxJavaPlugins.onSchedule(Runnable)` 来允许一个 scheduler 钩子在原始的 *Runnable* 提交到底层的执行方案前可以操纵（封装或替换）此 *Runnable*。

在 *Schedule* 抽象类中对于方法 `scheduleDirect()` 的默认实现，是对我们用 `createWorker()` 为每个 *Runnable* 建立出来的 *Scheduler.Worker* 实例中的 `schedule()` 方法的一个代理。事实上 RxJava 鼓励我们在实现中，不要为每个任务都创建一个 *Worker*，因为这是比较耗费时间和内存的。

# Schedulers

Schedulers 是一个工厂类，用来获取一些我们常用的 Schduler。例如：

- computation
- io
- trampoline
- newThread
- single

我们以 io 为例开探究一下。

## Schedulers.io()

本来比较简单的逻辑，但是我是不懂为什么设计来转这么多弯子的。

在 Schedulers 内会初始化 IoScheduler。

```java
    static final Scheduler IO;
    IO = RxJavaPlugins.initIoScheduler(new IOTask());
```

IoTask 是一个 Callable 的实现，简单的返回 IoHolder的默认值。

```java
    static final class IOTask implements Callable<Scheduler> {
        @Override
        public Scheduler call() throws Exception {
            return IoHolder.DEFAULT;
        }
    }
```


IoHolder 是 IoScheduler 的容器：

```java
    static final class IoHolder {
        static final Scheduler DEFAULT = new IoScheduler();
    }
```

IoScheduler 才是具体对 Scheduler 的实现。

OK，现在我们大概知道 IoScheduler 是什么了。

# SubscribeOn

我们知道，每个操作符都会返回一个新的 Observable。例如我们应用 `map()` 会返回一个 *ObservableMap*。同理，应用 `subscribeOn()` 会返回一个 *ObservableSubscribeOn*。

```java
    public ObservableSubscribeOn(ObservableSource<T> source, Scheduler scheduler) {
        super(source);
        this.scheduler = scheduler;
    }
```

*ObservableSubscribeOn* 对象持有了我们指定的调度器 scheduler。

对于一般的 Observable 如 ObservableMap 来说，其在 `subscribeActual()` 中实现的内容是比较简单的，用一个 *MapObserver* 来订阅上游就OK了。

```java
    @Override
    public void subscribeActual(Observer<? super U> t) {
        source.subscribe(new MapObserver<T, U>(t, function));
    }
```

而 *ObservableSubscribeOn* 则比较复杂一些：

```java
    @Override
    public void subscribeActual(final Observer<? super T> observer) {
        final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(observer);

        observer.onSubscribe(parent);

        parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
    }
```

在这里，按照常规的思考，在 *ObservableSubscribeOn* 中应该是使用 *SubscribeOnObserver*(parent) 来订阅上游 source 的。

一般来说，我们以 

```java
    Observable.subscribe(Observer)
```

```java
    void subscribe(@NonNull Observer<? super T> observer);
```
的形式开始的时候，在 Observable 都会调用 Observer 的 onSubscribe 方法。

```java
    void onSubscribe(@NonNull Disposable d);
```



# SubscribeOnObserver.setDisposable()


代码很简单，

```java
        parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
```

只是一 *scheduler* 内调度了一个任务，来执行平时我们执行的代码：

```java
    final class SubscribeTask implements Runnable {
        private final SubscribeOnObserver<T> parent;

        SubscribeTask(SubscribeOnObserver<T> parent) {
            this.parent = parent;
        }

        @Override
        public void run() {
            source.subscribe(parent);
        }
    }
```

可以理解为，是在另外一个 *scheduler* 内对 source 上游进行了订阅。

这就导致了所有的 subscribe 操作在那个新的线程上执行。所以说，subscribeOn 只有距离源最近的那个会生效，及时多次执行，后面的也不会导致线程的切换。

# observeOn

对于 observeOn 会返回一个 *ObservableObserveOn*，之后用 *ObserveOnObserver* 将其与上游联系起来。

需要注意的是，*ObserveOnObserver* 持有一个 worker 实例，对于所有的数据，都会通过 worker 来调度执行。

# 最后

线程的切换都是通过作为中间联系的 Observer 来进行的。


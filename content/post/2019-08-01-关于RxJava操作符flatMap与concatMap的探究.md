---
title: 关于RxJava操作符flatMap与concatMap的探究
categories:
  - RxJava
date: 2019-08-01 23:53:06
updated: 2019-08-01 23:53:06
tags: 
  - RxJava
  - Java
  - Android
---
迷惑的地方在于当 flatMap 和 concatMap 在运作的时候，在配合线程切换的话，其细节到底是怎么样的呢？
<!--more-->

# FlatMap

根据 Reactivex.io 网站上的定义：

>The FlatMap operator transforms an Observable by applying a function that you specify to each item emitted by the source Observable, where that function returns an Observable that itself emits items. FlatMap then merges the emissions of these resulting Observables, emitting these merged results as its own sequence.

>This method is useful, for example, when you have an Observable that emits a series of items that themselves have Observable members or are in other ways transformable into Observables, so that you can create a new Observable that emits the complete collection of items emitted by the sub-Observables of these items.

>**Note** that FlatMap merges the emissions of these Observables, so that they may interleave.

这些说的是：

1. 我们使用一个函数来将 Observable 发射的每个元素都变换为一个 Observable。这个函数，我们称之为 *mapper*
2. 然后 FlatMap 将所有变换后的 Observable 发射的元素进行 merge(合并)，最终，得到一个发射所有这些合并后元素的 Observable。
3. merge(合并)后的元素，其顺序是不一定的。

## 基础

首先，我们知道，{% post_link 关于RxJava操作符链式调用的理解 RxJava 的运作其实分成三个阶段 %}：

- 装配（Assembly Time）
- 订阅（Subscription Time）
- 运行（Runtime）

## 例子

```java
        Observable.just("A","B","C")
                .flatMap(new Function<String, ObservableSource<String>>() {
                    @Override
                    public ObservableSource<String> apply(String it) throws Exception {
                        return Observable.create(emitter -> {
                            for (int i = 0; i < 3; i++) {
                                emitter.onNext(String.format("%s-%d", it, i));
                            }
                            ;
                            emitter.onComplete();
                        });
                    }
                })
                .subscribe(System.out::println);
```
### Assembly
`Observable.just("A","B","C")` 会返回一个 ObservableFromArray，在此我称其为 *source*。

当我们调用 `source.flatMap()`的时候，其实是在装配阶段，这个时候，其结果，会返回一个 ObservableFlatMap，为了称呼，我们称之为 *flatObservable*，此 ObservableFlatMap 的上流（upstream） 就是 *source*：

```java
RxJavaPlugins.onAssembly(new ObservableFlatMap<T, R>(this, mapper, delayErrors, maxConcurrency, bufferSize));
```

好了，这个时候， *source* 与 *flatObservable* 之间的联系，也就是 *flatObservable* 持有 *source* 而已。

### Runtime

当我们对 *flatObservable* 进行订阅的时候，实际上，我们是对其指定了一个 Observer。


```java

    // ObservableFlatMap
    @Override
    public void subscribeActual(Observer<? super U> t) {

        if (ObservableScalarXMap.tryScalarXMapSubscribe(source, t, mapper)) {
            return;
        }

        source.subscribe(new MergeObserver<T, U>(t, mapper, delayErrors, maxConcurrency, bufferSize));
    }
```

是不是很惊喜，*flatObservable* 利用一个 *MergeObserver* 又订阅了 *source*。

我们来看看， 我们的 *source* ObservableFromArray 这个时候干了什么：

```java
//ObservableFromArray
    @Override
    public void subscribeActual(Observer<? super T> observer) {
        FromArrayDisposable<T> d = new FromArrayDisposable<T>(observer, array);

        observer.onSubscribe(d);

        if (d.fusionMode) {
            return;
        }

        d.run();
    }
```

```java
        FromArrayDisposable(Observer<? super T> actual, T[] array) {
            this.downstream = actual;
            this.array = array;
        }
        
        void run() {
            T[] a = array;
            int n = a.length;

            for (int i = 0; i < n && !isDisposed(); i++) {
                T value = a[i];
                if (value == null) {
                    downstream.onError(new NullPointerException("The " + i + "th element is null"));
                    return;
                }
                downstream.onNext(value);
            }
            if (!isDisposed()) {
                downstream.onComplete();
            }
        }
```

很简单，ObservableFromArray 建立了一个 FromArrayDisposable，其利用了 for 循环，给 downstream 传递数据（这里是 MergeObserver ）传递数据。

```java
//MergeObserver
        @Override
        public void onNext(T t) {
            // safeguard against misbehaving sources
            if (done) {
                return;
            }
            ObservableSource<? extends U> p;
            try {
                p = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper returned a null ObservableSource");
            } catch (Throwable e) {
                Exceptions.throwIfFatal(e);
                upstream.dispose();
                onError(e);
                return;
            }

            if (maxConcurrency != Integer.MAX_VALUE) {
                synchronized (this) {
                    if (wip == maxConcurrency) {
                        sources.offer(p);
                        return;
                    }
                    wip++;
                }
            }

            subscribeInner(p);
        }
```
MergeObserver 首先调用我们提供的 mapper 函数来获得一个 Observable。

```java
// MergeObserver
        @SuppressWarnings("unchecked")
        void subscribeInner(ObservableSource<? extends U> p) {
            for (;;) {
                if (p instanceof Callable) {
                    if (tryEmitScalar(((Callable<? extends U>)p)) && maxConcurrency != Integer.MAX_VALUE) {
                        boolean empty = false;
                        synchronized (this) {
                            p = sources.poll();
                            if (p == null) {
                                wip--;
                                empty = true;
                            }
                        }
                        if (empty) {
                            drain();
                            break;
                        }
                    } else {
                        break;
                    }
                } else {
                    InnerObserver<T, U> inner = new InnerObserver<T, U>(this, uniqueId++);
                    if (addInner(inner)) {
                        p.subscribe(inner);
                    }
                    break;
                }
            }
        }
```
接着，用一个 InnerObserver 来订阅了我们 mapper 返回的 Observable。这里，如果是 Callable 来的 Observer ,就直接发射数据了。InnerObserver 有一个 parent 参数，这里是 MergeObserver。

```java
//InnerObserver
        @Override
        public void onSubscribe(Disposable d) {
            if (DisposableHelper.setOnce(this, d)) {
                if (d instanceof QueueDisposable) {
                    @SuppressWarnings("unchecked")
                    QueueDisposable<U> qd = (QueueDisposable<U>) d;

                    int m = qd.requestFusion(QueueDisposable.ANY | QueueDisposable.BOUNDARY);
                    if (m == QueueDisposable.SYNC) {
                        fusionMode = m;
                        queue = qd;
                        done = true;
                        parent.drain();
                        return;
                    }
                    if (m == QueueDisposable.ASYNC) {
                        fusionMode = m;
                        queue = qd;
                    }
                }
            }
        }
        
        @Override
        public void onNext(U t) {
            if (fusionMode == QueueDisposable.NONE) {
                parent.tryEmit(t, this);
            } else {
                parent.drain();
            }
        }
```

当我们 mapper 向 InnerObserver 发射数据的时候，其直接将数据发射给了 MergeObserver。

这里说明一下，如果 InnerObserver 订阅的是一个 QueueDisposable 的话，那么其就会协商一下，发送模式，如果是同步发射，就会要求 MergeObserver 将所有 mapper 返回的 Observable 数据全部发射到 最下游再继续。

当我们的 mapper 返回的 Observable 发射模式是 QueueDisposable.NONE 时，MergeObserver 采用的是 tryEmit 的形式来发射数据：

```java
        void tryEmit(U value, InnerObserver<T, U> inner) {
            if (get() == 0 && compareAndSet(0, 1)) {
                downstream.onNext(value);
                if (decrementAndGet() == 0) {
                    return;
                }
            } else {
                SimpleQueue<U> q = inner.queue;
                if (q == null) {
                    q = new SpscLinkedArrayQueue<U>(bufferSize);
                    inner.queue = q;
                }
                q.offer(value);
                if (getAndIncrement() != 0) {
                    return;
                }
            }
            drainLoop();
        }
```

如果数据无法立即发射，那么就把他放在队列中，在 for 循环内进行遍历发射。

看起来好像不会出乱序的情况？例子中的执行确实也没有出现乱序这是为什么？ 这是因为我们是在同一个线程内进行操作的，同时我们的数据也比较特殊。

可以从这里来看，对于每一个 mapper 返回的 Observable，我们都用 InnerObserver 来进行了订阅，但是其何时发射数据，这个是不一定的。所以说，在 InnerObserver 的 onNext 方法中，随后调用 MergeObserver.tryEmit(value, inner) 的方法时，会有需要发射的值，放到 inner 的队列中，然后再进行发射。

因此这个顺序是不一定的。


# ConcatMap

ConcatMap 的实现其实和 flatMap 有相似的地方。不过其多做了一点事情：

1. 用 SourceObserver 来连接上游的 source。
2. 用 InnerObserver 来订阅每个变换后的 Observable。
3. SourceObserver 的下流是一个 SerializedObserver
4. 由 SerializedObserver 将数据发射至最后。


其工作的过程是：

1. SourceObserver由 onNext 收到数据，放到队列中。
2. 对队列中的数据进行应用 mapper ，然后用一个 InnerObserver 进行订阅。
3. InnerObserver 会将数据发射给 SerializedObserver。
4. SerializedObserver 也在在队列中将数据逐个发射出去。

# SubscribeOn

指定 Observable 在哪个 Scheduler 上进行操作。


# ObserverOn

这个操作符的意义，是指定，我们的 Observer 会在哪个 Scheduler 上观察 Observable，也就是说，Observable 将通知发送到哪个 Scheduler。



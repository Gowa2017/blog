---
title: RxJava之retryWhen操作符
categories:
  - [RxJava]

date: 2019-09-23 14:55:20
updated: 2019-09-23 14:55:20
tags: 
  - RxJava
  - Android
  - Java
---

如何用来进行重复的尝试请求，需要进行探究了一下。

<!--more-->

首先，对于每个 Observable 其都会与一个源 (source)，一个 observer （下游） 想关联。我们的 retryWhen 操作符建立的  **ObservableRetryWhen** 也不例外：

```java
    public ObservableRetryWhen(ObservableSource<T> source, Function<? super Observable<Throwable>, ? extends ObservableSource<?>> handler) {
        super(source);
        this.handler = handler;
    }

```

当我们对 **ObservableRetryWhen** 进行 subscribe 的时候，逻辑就开始进行了：



```java
    @Override
    protected void subscribeActual(Observer<? super T> observer) {
        Subject<Throwable> signaller = PublishSubject.<Throwable>create().toSerialized();

        ObservableSource<?> other;

        try {
            other = ObjectHelper.requireNonNull(handler.apply(signaller), "The handler returned a null ObservableSource");
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            EmptyDisposable.error(ex, observer);
            return;
        }

        RepeatWhenObserver<T> parent = new RepeatWhenObserver<T>(observer, signaller, source);
        observer.onSubscribe(parent);

        other.subscribe(parent.inner);

        parent.subscribeNext();
    }

```

可以看到，在内部其建立了一个即是 Observable 也是 Observer 的 signaller，还用 handler 来建立一个 other 的 Observable。



**RepeatWhenObserver** 是用来将 source, observer及 other 关联起来的关键。



# signaller

```java
        Subject<Throwable> signaller = PublishSubject.<Throwable>create().toSerialized();
```

当我们的 RepeatWhenObserver 对 source 的订阅，收到一个 onError 事件的时候，会将此事件丢给  handler 进行逻辑判断，然后进行返回。

# handler.call()

控制器的应用会返回一个 **ObservableSource<?> other;**，其观察者是 RepeatWhenObserver.InnerRepeatObserver。

这个 handler 的本质，是对 signaller 进行变换，判断，是丢出一个 onNext事件，还是丢出一个 onError 事件。

# RepeatWhenObserver.subscribeNext();

我们的关键再于 RepeatWhenObserver，其首先会对source 进行订阅：

```java
        void subscribeNext() {
            if (wip.getAndIncrement() == 0) {

                do {
                    if (isDisposed()) {
                        return;
                    }

                    if (!active) {
                        active = true;
                        source.subscribe(this);
                    }
                } while (wip.decrementAndGet() != 0);
            }
        }

```



当其收到一个 onError 事件时，就会让 signaller 发射 onNext 事件，进行条件判断。



```java
        @Override
        public void onError(Throwable e) {
            DisposableHelper.replace(upstream, null);
            active = false;
            signaller.onNext(e);
        }

```

最终，其发射的事件由 hanlder 进行变换后，交给 RepeatWhenObserver.InnerRepeatObserver。如果变换后满足条件，那么 RepeatWhenObserver.InnerRepeatObserver 会最到 onNext 事件：

```java
            @Override
            public void onNext(Object t) {
                innerNext();
            }
```

最终会让 RepeatWhenObserver 再次订阅：

```java
        void innerNext() {
            subscribeNext();
        }

```



# 总结

对于 RepeatWhenObserver. 其订阅 source 后，就不再进行任何的操作了。如果一切正常，没有问题，那么就把 onNext 收到的数据丢给 obsever 就行了。



如果其收到 onError 事件，就会调用 signaller ，由 handler 进行判断是否要重新进行订阅，判断结果发送到 RepeatWhenObserver 内部的 InnerRepeatObserver，其在收到 onNext 事件使会重新订阅 source，否则的话就把原来的 error 直接丢给下游了
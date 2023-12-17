---
title: RxJava的blocking操作符
categories:
  - RxJava
date: 2019-05-31 21:20:30
updated: 2019-05-31 21:20:30
tags: 
  - RxJava
  - Java
  - Android
---
之所以有这样的场景，是因为当我想像 Stream API 那样使用 Observable 的时候出现了问题。所以就来了解一下 blocking 操作符到底是怎么样工作的。

<!--more-->

首先，我们知道对于 Observer 与 Observable 的关系是：

```java
Observable.subscribe(Observer)
```

之后， Observable 就会调用  Observer 的方法 `onNext(), onError(), onComplete()` 方法来发送数据和通知。

同时呢，对于在 Observable 上应用的每个操作符，都会返回一个新的 Observable。新成的 Observable 会将之前的 Observable 当作 Upstream。（在代码中可以看到是表示为 *source, upstream* 等）

同时，新返回的 Observable 会用一个内部的 Observer 来和 upstream 进行连接。

# blockingFirst()

对于在 Observable 上应用这个操作符：

```java
    public final T blockingFirst() {
        BlockingFirstObserver<T> observer = new BlockingFirstObserver<T>();
        subscribe(observer);
        T v = observer.blockingGet();
        if (v != null) {
            return v;
        }
        throw new NoSuchElementException();
    }
```

就是用一个 BlockingFirstObserver 来订阅 Observable ，接着就执行了 blockingGet() 方法。

BlockingFirstObserver 继承自 BlockingBaseObserver（继承自 CountDownLatch）。

在 BlockingBaseObserver 中默认以参数 1 初始化了 CountDownLatch。

# CountDownLatch

我们先来看一下 CountDownLatch 这个类到底是对什么的抽象。
CountDownLatch 位于 java.util.concurrent 包下，是 Java 内建的类。

根据 JavaDoc：

> CountDownLatch 是一个同步助手，其允许一个或多个线程在一系列其他线程正在执行的操作完成前进行等待。  
> 其需要一个 *count* 参数来进行初始化。其 `await()` 方法在 当前的 *count* 值为0前等待（可以通过 `countDown()` 方法来减少）。在 *count* 为 0 后，所有等待的线程都被唤醒，后续所有的 `await()` 方法调用都会立刻返回。这个过程是不可逆的———— *count* 值不能被重置。如果我们需要重置 *count*，可以考虑使用 **CyclicBarrier**。  

## Sync 与状态

CountDownLatch 持有一个其内部的类 Sync 的实例，Sync 继承自 AbstractQueuedSynchronizer。其是一个有状态的类，用来作为很多同步操作的基础。

```java
    private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;

        Sync(int count) {
            setState(count);
        }

        int getCount() {
            return getState();
        }

        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }

        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }

    private final Sync sync;
```




## await()

BlockingFirstObserver 会调用 blockingGet 方法：

```java
    public final T blockingGet() {
        if (getCount() != 0) {
            try {
                BlockingHelper.verifyNonBlocking();
                await();
            } catch (InterruptedException ex) {
                dispose();
                throw ExceptionHelper.wrapOrThrow(ex);
            }
        }

        Throwable e = error;
        if (e != null) {
            throw ExceptionHelper.wrapOrThrow(e);
        }
        return value;
    }
```

如前文所述，如果当前的 *count* 值（状态） 不为0，那么就会调用其父类 CountDownLatch 的 `await()` 方法，进入等待：

```java
    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
```


不过根据我观察代码发现一个问题，事实上虽然说的是 blocking 的形式去获取数据。在我们不使用 subscribeOn 操作符的情况下，事实上代码：

```java
    public final T blockingFirst() {
        BlockingFirstObserver<T> observer = new BlockingFirstObserver<T>();
        subscribe(observer);
        T v = observer.blockingGet();
        if (v != null) {
            return v;
        }
        throw new NoSuchElementException();
    }
```

在 subscribe 的时候， Observable 就开始发射数据，等到调用  `blockingGet()` 方法的时候，数据都已经发射过来了，看起来是不会进入阻塞状态的呢。

## countDown()

在源 Observable 发射数据的时候，其实就已经改变了锁的状态(count 值)了，所以呢，不会阻塞了。

```java
public final class BlockingFirstObserver<T> extends BlockingBaseObserver<T> {

    @Override
    public void onNext(T t) {
        if (value == null) {
            value = t;
            upstream.dispose();
            countDown();
        }
    }

    @Override
    public void onError(Throwable t) {
        if (value == null) {
            error = t;
        }
        countDown();
    }
}
```

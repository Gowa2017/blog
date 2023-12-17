---
title: RxJava使用blockingGet造成麻烦问题
categories:
  - RxJava
date: 2019-05-27 22:08:46
updated: 2019-05-27 22:08:46
tags: 
  - RxJava
  - Java
  - Android
---
当使用 RxJava 和 Andorid 一起进行开发的时候，一不小心就中招了。一行不起眼的代码，就让线程永远的阻塞了下去。

<!--remove-->

# 问题代码

```java
List<ComChangePersonInfo> personInfos ;
List<MaterialInfo> lst) ;

        Map<String, Integer> map = Observable.fromIterable(lst)
                .filter(m -> TextUtils.equals(m.getMtype(), AppConstant.ComRegMaterialType.CARD))
                .groupBy(m -> m.getAboutPerson())
                .collect(HashMap::new, (BiConsumer<Map<String, Integer>, GroupedObservable<String, MaterialInfo>>) (m, observable) -> m.put(observable.getKey(), observable.count().blockingGet().intValue()))
                .blockingGet();


```

这段代码的背景是：

- personInfos 保存的是一些人员信息
- lst 保存的是一些证件信息。证件信息通过 MateriInfo.AboutPerson = ComChangePersonInfo.Id 进行关联，我需要的是为 AppConstant.ComRegMaterialType.CARD 这种类型

我现在想做的就是验证一下，每个人是否都传了两张证件信息。

所以我就想着，将 lst 进行一下分组，得出一个 Map，里面保存了每个 personId  的图片数量。

结果就造成了 ANR 。


# 问题

在 `groupBy` 之前是没有什么问题的，正常。

我们知道一个流开始发射数据是在我们调用  subscribe 方法的时候开始的，但事实上对于 blockingGet 也会开始进行订阅。

```java
    public final T blockingGet() {
        BlockingMultiObserver<T> observer = new BlockingMultiObserver<T>();
        subscribe(observer);
        return observer.blockingGet();
    }
```
我们在 `.collect()` 中就开始订阅 GroupObservable 开始发射数据，同时阻塞线程开始等待，直到上游数据`.groupBy()` 发送完毕。

但事实上这个时候是无法获取到数据的，为什么？

GroupObservable 是由 `.groupBy()` 被 `.collect()` 订阅后才会发送的，但因为我们线程阻塞了，我们永远也无法走到最后的那个 `.blockingGet()`，所以就造成了死锁。

ANR 当然就是这样了。

搜索谷歌，大多说对于 blocking 操作，和安卓的使用有点不太好用。


# 解决办法

1. 放弃 RxJava 的方式。
2. 不要在 collect 内进行 blockingGet 操作。

我们换个方式想，对于 `collect()` 操作来说，其不会马上就将数据往下游发射，而是要等到上游数据发射完毕，发出了 `onComplete` 通知，才会往下游发送数据，这就好办了。

```java
List<ComChangePersonInfo> personInfos ;
List<MaterialInfo> lst) ;

        Map<String, Integer> map = Observable.fromIterable(lst)
                .filter(m -> TextUtils.equals(m.getMtype(), AppConstant.ComRegMaterialType.CARD))
                .groupBy(m -> m.getAboutPerson())
                .collect(HashMap<String,Single<Long>::new, (BiConsumer<Map<String, Single<Long>>, GroupedObservable<String, MaterialInfo>>) (m, observable) -> m.put(observable.getKey(), observable.count()))
                .flatMap((Function<HashMap<String, Single<Long>>, SingleSource<HashMap<String, Integer>>>) stringSingleHashMap -> {
                    HashMap<String,Integer> map = new HashMap<>();
                    stringSingleHashMap.forEach((k,v)->map.put(k,v.blockingGet().intValue()));
                    return Single.just(map);
                })
                .blockingGet();
```

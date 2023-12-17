---
title: RxJava的聚合操作函数collect与groupby
categories:
  - RxJava
date: 2019-05-27 23:03:55
updated: 2019-05-27 23:03:55
tags: 
  - RxJava
  - Java
  - Android
---
RxJava 提供了一些类似与 stream 的方法，恰好我们在 API26 以下的安卓是无法使用 stream API的，所以尝试用这种方式来使用，但是会有坑的。

<!--more-->

# groupBy

对一个 Observable 使用  groupBy 后，会返回 ObservableGroupBy。一般情况下，我们只指定 keySelector 就行了，其他的使用默认值，当然，有的时候我们还会指定 valueSelector。

```java
    public final <K, V> Observable<GroupedObservable<K, V>> groupBy(Function<? super T, ? extends K> keySelector,
            Function<? super T, ? extends V> valueSelector,
            boolean delayError, int bufferSize) {
        ObjectHelper.requireNonNull(keySelector, "keySelector is null");
        ObjectHelper.requireNonNull(valueSelector, "valueSelector is null");
        ObjectHelper.verifyPositive(bufferSize, "bufferSize");

        return RxJavaPlugins.onAssembly(new ObservableGroupBy<T, K, V>(this, keySelector, valueSelector, bufferSize, delayError));
    }
```

*keySelector* 用来选择组的键，我们也可以把他当做组的ID。可以理解为：**对于发射的每个值，通过　keySelector 来赋予一个组的ID**。之后我们就可以根据组 ID 来干很多事情了。

```java
        List<Dict> lst = new ArrayList<>();
        lst.add(new Dict("1", "A"));
        lst.add(new Dict("2", "B"));
        lst.add(new Dict("1", "B"));
        lst.add(new Dict("2", "A"));
        lst.add(new Dict("3", "B"));
        lst.add(new Dict("3", "A"));
        Observable.fromIterable(lst)
                .groupBy(new Function<Dict, String>() {
                    @Override
                    public String apply(Dict dict) throws Exception {
                        return dict.getValue()+"group";
                    }
                })
                .subscribe(grp -> grp.subscribe( d -> System.out.println(d)));

```

**简单而言，groupBy 会将每个元素都赋予一个组ID，然后将组ID相同元素放在一个 Observable 内发射出来。**

# collect

这个操作符就比较猥琐了。
根据定义：collect 将会有限个元素的源发射的数据装到一个可变的数据结构内，然后返回一个会发射这个数据结构的 *Single*。

collect 是 reduce 的简化操作，不需像 reduce 那些需要在处理每个元素的时候都返回状态。

要求 upstream 必须要发射 `onComplete` 通知。

```java
    public final <U> Single<U> collect(Callable<? extends U> initialValueSupplier, BiConsumer<? super U, ? super T> collector) {
        ObjectHelper.requireNonNull(initialValueSupplier, "initialValueSupplier is null");
        ObjectHelper.requireNonNull(collector, "collector is null");
        return RxJavaPlugins.onAssembly(new ObservableCollectSingle<T, U>(this, initialValueSupplier, collector));
    }
```

其需要的两个参数：

- initialValueSupplier 数据结构构造方法
- collector 

泛型参数：

- U 表示返回值（数据结构）的类型
- T upstream 发射元素的类型

```java
        List<Dict> lst = new ArrayList<>();
        lst.add(new Dict("1", "A"));
        lst.add(new Dict("2", "B"));
        lst.add(new Dict("1", "B"));
        lst.add(new Dict("2", "A"));
        lst.add(new Dict("3", "B"));
        lst.add(new Dict("3", "A"));
        HashMap<String,Integer> res =
        Observable.fromIterable(lst)
                .groupBy(dict -> dict.getValue() + "group")
                .collect(HashMap<String, Single<Long>>::new, (m, e) -> m.put(e.getKey(), e.count()))
                .flatMap((Function<HashMap<String, Single<Long>>, SingleSource<HashMap<String, Integer>>>) stringSingleHashMap -> {
                    HashMap<String,Integer> map = new HashMap<>();
                    stringSingleHashMap.forEach((k,v)->map.put(k,v.blockingGet().intValue()));
                    return Single.just(map);
                })
                .blockingGet();
        System.out.println(res);
```


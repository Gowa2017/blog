---
title: Java的Lambda-方法引用及Stream
categories:
  - Java
date: 2019-04-09 22:30:31
updated: 2019-04-09 22:30:31
tags: 
  - Java
---
Lambda, Stream, Method Reference 是 Java 迈向函数式编程，操作符式的操作重要的一步。能让我们的代码更精简，更紧凑，手也更轻松些。

<!--more-->

# Lambda

在我们实现一个匿名类的时候，只需要悄悄的 new 一下就行了。但是实际上我不得不写一长串代码，即使这个类只或接口只包含了一个方法。同时了，语法看起来也是不太友好的。

经常，我们会把函数做一个参数传递给另外一个方法，比如在安卓中我们对 View 设置事件监听函数的时候。

## 语法

一个 lambda 表示式由以下几部分组成：

- (a,b,c) 由括号包围起来的，逗号分隔的参数列表。我们可以忽略参数的类型。如果参数只有一个，那么可以忽略括号。也即是说 (a) 与 a 与相同的效果。
- 箭头 -> 
- 函数体。可以是一个表达式，或者是一个语句块。如果只有一个表达式，执行的时候会返回表达式的值。如果是 {} 语句话，就要加上 `return ...` 语句。不过，是一个 void 返回值的方法，也可以同表达式那样使用。

lambda 和 方法声明看起来很相似；我们也可以把他看作是匿名方法。

# Method References

我们用 lambda 在很多时候都只是为了调用一个已经存在的方法。 Method References 可以让我们很用 lambda 来调用一个已经存在的方法。

- 引用静态方法。ContainingClass::staticMethodName
- 引用实例方法。containingObject::instanceMethodName
- 专有类型的实例方法。ContainingType::methodName
- 引用构造器。ClassName::new

# Stream

我们会用 集合 Collections 来干什么？不仅仅是存储数据，我们还经常要从里面获取数据。

比如：

```java
for (Person p : roster) {
    System.out.println(p.getName());
}
```

可以转换为:

```java
roster
    .stream()
    .forEach(e -> System.out.println(e.getName());
```

## 管道与流

一个 **管道** 是有序的聚合操作序列。

```java
roster
    .stream()
    .filter(e -> e.getGender() == Person.Sex.MALE)
    .forEach(e -> System.out.println(e.getName()));
```

用 for-loop 的话就是这种形式：

```java
for (Person p : roster) {
    if (p.getGender() == Person.Sex.MALE) {
        System.out.println(p.getName());
    }
}
```

管道包含：

- 一个源。可以是一个集合，一个数组，一个生成函数，一个 I/O 通道。
- 0 或 多个中间操作。一个中间操作，会产生一个新的流，如 filter。 **流 stream** 是一个序列的元素。和集合不一样的是，它不是一个存储元素的数据结构。一个流从一个源携带值通过管道。
- 一个 中止操作。如 forEach 会产生一个非流的结果，比如一个基础类型，一个集合，也可能什么都不产生。

```java
double average = roster
    .stream()
    .filter(p -> p.getGender() == Person.Sex.MALE)
    .mapToInt(Person::getAge)
    .average()
    .getAsDouble();
```

## 聚合操作与迭代的不同

聚合操作，如 forEach，与迭代器看起来很像。但有一些根本的不同：

- **使用内部迭代** 聚合操作没有像 `next()` 这样的方法来指示处理集合中的下一个元素。
- **从流中处理元素**
- **支持用 lambda 作为参数**


## 流的获取

- Collection.stram(), Collection.parallelStream() 集合
- Arrays.stream(Object[]) 数组
- Stream.of(Object[]), IntStream.range(int, int) or Stream.iterate(Object, UnaryOperator); steam 类的静态工厂方法
- BufferedReader.lines(); 文件行
- Files 内获取文件路径
- Random.ints()
- BitSet.stream(), Pattern.splitAsStream(java.lang.CharSequence), and JarFile.stream().

## API

[https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html)

### 中间操作

- concat
- distinct
- filter
- flatMap
- flatMapToDouble
- flatMapToInt
- flatMapToLong
- generate
- iterate
- limit
- map
- skip
- sorted
 
### 中止操作

- collect
- count
- findAny
- findFirst
- forEach
- forEachOrdered
- of
- peek

### 其他操作

- allMatch
- anyMatch
- builder
- max
- min
- noneMatch
- reduce
- toArray

# 关于 Collector 的一些方法

Collector 是一个接口，你定义用一些用来对 Stream 进行减少操作的方法。

所以的 **减少操作（reduction operation）** 将多个元素整合在一个结果中。

其有三个泛型参数： Collector<T, A, R>

- T 进行减少操作的元素类型
- A 减少操作的可变的方法，可由我们定义
- R 返回的结果类型

## 方法

- BiConsumer<A,T> accumulator() 将一个值当到一个结果容器中的函数
- Set<Collector.Characteristics> characteristics() 返回特征集合
- BinaryOperator< A > combiner() 将两个结果合并
- Function<A,R> finisher()

## Collectors

实现了接口 Collector，提供一系列非常实用的方法。

# 安卓上使用 Stream



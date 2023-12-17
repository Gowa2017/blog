---
title: Java-基本表达式(Primary_Expressions)
categories:
  - Java
date: 2018-03-22 09:49:25
updated: 2018-03-22 09:49:25
tags: 
  - Java
  - The Java Tutorial
---
基本表达式包含了最简单的类型的表达式，其他类似的表达式都由他们来构建：字面表达式，class字面表达式，字段访问，方法调用，数组访问。一个括号表达式语法上也被看成是一个基本表达式。
<!--more-->
网页内容地址 [15.8. Primary Expressions](https://docs.oracle.com/javase/specs/jls/se7/html/jls-15.html#jls-15.8)

```
Primary:
    PrimaryNoNewArray
    ArrayCreationExpression

PrimaryNoNewArray:
    Literal
    Type . class
    void . class
    this
    ClassName . this
    ( Expression )
    ClassInstanceCreationExpression
    FieldAccess
    MethodInvocation
    ArrayAccess
```

# 类字面量
一个类字面量表达式由 类，接口，数组的名字，或基本类型，伪类型 void，后跟上 `.class` 组成。

```
ClassLiteral:
TypeName {[ ]} . class 
NumericType {[ ]} . class 
boolean {[ ]} . class 
void . class
```

* **C.class**，C是一个类，接口，数组的名字，其类型是 `Class <C>`。
* **p.class**，p 是基本类型， 的类型是 `Class <B>`，其中 B 是 p 在经常了黑盒转换后类型的表达式。
* **void.class**，类型是 `Class <void>`。

所以呢，如果我们在需要 `Class <C>` 的地方，我们就可以用 C.class 来指代。

那么 `Class <C>` 又是什么意思呢。

# Class
Class与class并不一致，前者是 `java.lang`中的一个类，后者是关键词。我们常会看到 `Class <T> cls` 这样的声明。

其中，**T** 是被 Class 对象所模仿的 类 的类型。如，**String.class** 的类型是 `Class <String>`。当需要模仿的类是未知的时候，使用 `Class <?>`。

其类声明：

```java
public final class Class<T>
extends Object
implements Serializable, GenericDeclaration, Type, AnnotatedElement
```

Class类的实例代表了一个Java应用中的 类 和 接口。枚举是一种类，注释是一种接口。每个数组都属于一个类，这个类是被 Class 对象反射的，所有的数组共享这个 Class 对象反射的类，因此所有数组具有相同的元素类型。基本的Java类型 (**boolean, byte, char, short, long, int, fload, double**）以及 **void** 都代表一个 Class 对象。

Class没有公开的构造器。Class对象会被 Java 虚拟机以类的形式自动构建，这种类在类加载器中加载且会调用 `defineClass` 方法。

下面的例子使用，**Class** 对象来打印一个对象的类名：

```java
void printClassName(Object obj) {
	System.out.println("The class of " + obj + " is " + obj.getClass().getName());
}
```


# 泛型类
我们经常会看见类似的类声明：

在开源项目 [BaseRecyclerViewAdapterHelper](https://github.com/CymChad/BaseRecyclerViewAdapterHelper
)中就有：   

```java
public abstract class 
BaseQuickAdapter<T, K extends BaseViewHolder> 
extends RecyclerView.Adapter<K> {

}
```
这表示，我们在这个类中使用的关于 T ，K 类型可以是任意我们传入的。

当然，仔细理解了才会明白这是什么意思。


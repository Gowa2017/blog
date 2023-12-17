---
title: Java的接口与抽象类(Interface与Abstarct-Class)
categories:
  - Java
date: 2018-06-01 09:10:15
updated: 2018-06-01 09:10:15
tags: 
  - Java
---
在Java中感觉接口与抽象类非常的相似，但其实应该是有区别的，不然的话何必要设计出这两个东西来呢，所以呢仔细看了一下官方的文档，了解一下细节。

# 接口Interface

在Java编程语言中，接口是一个引用类型，类似于一个类，它**只能包含常量，方法签名，缺省方法，静态方法和嵌套类型**。 方法体只存在于默认方法和静态方法中。 接口不能被实例化 - 它们只能由类实现或由其他接口扩展。 

定义一个**接口**就和定义一个类相似：

```java
public interface OperateCar {

   // constant declarations, if any

   // method signatures
   
   // An enum with values RIGHT, LEFT
   int turn(Direction direction,
            double radius,
            double startSpeed,
            double endSpeed);
   int changeLanes(Direction direction,
                   double startSpeed,
                   double endSpeed);
   int signalTurn(Direction direction,
                  boolean signalOn);
   int getRadarFront(double distanceToCar,
                     double speedOfCar);
   int getRadarRear(double distanceToCar,
                    double speedOfCar);
         ......
   // more method signatures
}
```
> 接口内的方法，只有签名，而没有方法体。

接口，只是方法的集合。任何实现了接口所定义方法的类或对象，都课被当做那个接口使用。还一个可实例化的类实现了一个接口，那么其应该为接口内的所有方法提供一个方法体。

例如，下面这个类就实现了上述的接口：

```java
public class OperateBMW760i implements OperateCar {

    // the OperateCar method signatures, with implementation --
    // for example:
    int signalTurn(Direction direction, boolean signalOn) {
       // code to turn BMW's LEFT turn indicator lights on
       // code to turn BMW's LEFT turn indicator lights off
       // code to turn BMW's RIGHT turn indicator lights on
       // code to turn BMW's RIGHT turn indicator lights off
    }

    // other members, as needed -- for example, helper classes not 
    // visible to clients of the interface
}
```

在上面的机器人汽车示例中，实施接口的是汽车制造商。 当然，雪佛兰的实施将与丰田的实施大不相同，但两家制造商将坚持相同的接口。 作为接口客户的指导制造商将构建使用汽车位置GPS数据，数字街道地图和交通数据来驱动汽车的系统。 这样做时，指导系统将调用接口方法：转弯，改变车道，制动，加速等等。

# 抽象类Abstract Class

抽象类的定义很简单，其实就是一个声明为 **abstract**的类，其可能，也可能不包含抽象方法。抽象类不能被实例化，但他们可以作为其他类的超类。

抽象方法就是由关键字 **abstract** 修饰 的方法。其也不会有方法体：

```java
abstract void moveTo(double deltaX, double deltaY);
```

如果一个类包含了抽象方法，那么它自己必须是 **abstract**的。

```java
public abstract class GraphicObject {
   // declare fields
   // declare nonabstract methods
   abstract void draw();
}
```

当一个抽象类被继承的时候，子类通常会提供所有抽象方法的实现。如果没有的话，子类也必须声明为 **abstract**。

> 接口中没有被声明为 `defalut, static`的方法是 **隐式** 抽象的，所以 `abstract` 关键字没有被用来修饰接口方法。


# 比较
抽象类和接口非常的相似。我们不能实例化他们，他们可能包含了有实现或没有实现的方法。

然而，对于抽象类，可以定义 非 `statis, final` 的字段，可以定义 `public protected private` 约束的方法。

对于接口，所有的字段都自动是 `public,statis,final`的，所有声明的方法都是 `public`的。此外，我们只能扩展一个（抽象）类，不论其是否是抽象的，而我们可以实现任意数量的接口。

怎么选择抽象类还是接口？

**以下情况考虑使用抽象类**：    
* 在几个非常相关的类间共享代码
* 希望扩展抽象类的类有很多公共的方法和字段，或者需要不止是 `public` 的修饰符。
* 想声明非`static, final`的字段。这允许我们定义用来访问和修改这个对象的状态。

**以下情况考虑使用接口**：    

* 期望不相关的类实现我们的接口。例如，接口 *Comparable, Cloneable*
* 想要对一个具体的数据类型指定行为，但是并不在乎谁实现了它的行为。
* 想要利用类型的多重继承优势。

JDK中一个抽象类的例子是 *AbstractMap*，其是 *集合框架* 的一部分。其子类（**HashMap, TreeMap, ConcurrentHashMap**）共享很多方法（如 *get, put, containsKey, containsValue）*。

JDK中一个实现了多个接口的例子就是 **HashMap**，其实现了接口 *Serializable, Cloneable, Map<K, V>*。通过阅读这些接口列表，我们可以推端一个 *HaspMap*的实例可以被克隆，序列化（可以被转换为一个字节流），并且拥有 map 的功能。此外，*Map<K,V>*接口已经被增强了，提供了很多默认方法如 *merge, forEach*。

> 很多库使用了 抽象类和接口。*HaspMap* 实现了几个接口，但也扩展了抽象类*AbstractMap*。



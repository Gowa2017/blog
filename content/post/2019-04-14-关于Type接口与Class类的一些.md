---
title: 关于Type接口与Class类的一些
categories:
  - Java
date: 2019-04-14 22:01:21
updated: 2019-04-14 22:01:21
tags: 
  - Java
---
主要的问题是在于 Gson 在解析泛型化的类型的时候，因为泛型擦除的问题，无法解析出我想要的类型。所以得想个办法解决不是。？

<!--more-->

# 类型擦除
根据官方文档 [泛型擦除](https://docs.oracle.com/javase/tutorial/java/generics/erasure.html)。

泛型的引入是为了提供更严格的**编译时类型检测**及支持泛型编程。为了实现泛型，Java 的编译器会对下面的情况进行类型擦除：

- 将所有泛型的类型参数替换为他们的**界限(bounds)**或**Object（泛型类型参数没有界限）**。因此产生的字节码，只有普通的类，接口和方法。
- 为了保证类型安全会在必要的时候插入类型转换(cast)
- 在扩展的泛型类型中，会生成桥接方法来保证多态。

类型擦除保证参数化的类型不会产生新的类，因此泛型不会导致运行时的负载问题。

也就是说，如果我们传递一个泛型化的类型给 Gson 在运行的时候，其是不知道具体的类型是什么的。比如我们的服务端返回的结构是这样的：

```java
public class SrvRes<T> {
private int state;
private T data;
}
```

在 Gson 解析的时候实际上是按:

```java
public class SrvRes<Object>{
private int state;
private Object data;
}
```

其结果就是会报错，得出一个 LinkTreeMap 无法转换为 Object 的错误。

我们要做的就是在运行时获取这个类的类型。

**也就是说，接受泛型参数的类实例，是不包括类型参数信息的**。

只用通过泛型化参数继承而来的类，才能从其父类获得泛型信息。




# Type 接口

接口位于 *java.lang.reflect* 包内，只定义了一个方法 `getTypeName()`

```java
public interface Type {
    default String getTypeName() {
        return toString();
    }
}
```

其有四个子接口。
## ParameterizedType 接口

对于这个接口的解释，应该说这是是一个以参数类型与父类结合后形成的类。比如一个泛型类 SrvRes<T> ，这个不是一个 ParameterizedType 的实现。必须通过 T 实际的值后形成的类才是，比如：

```java
public class Res extends SrvRes<Res>{}
```

我们以 SrvRes<String> t = new SrvRes<String>() 的形式，实际上不是一个参数化类型，t 在这里只是一个对象，运行时，怎么也无法获取到泛型信息。

而对于 Res 在构造的时候就会保存其泛型信息。

此接口继承自 Type。其代表了一个参数化的类型，如 *Collection<String>*。当一个反射方法调用时如果需要这个类型，那么其会被创建。当一个参数化的类型 *p* 建立后，p 初始化的泛型声明被解析，所有 p 需要的类型参数都会被递归创建。

```java
public interface ParameterizedType extends Type {
    Type[] getActualTypeArguments();
    Type getRawType(); //　获取用来表示声明了此接口或Class的对象
    Type getOwnerType();// 获取包含此类型属于哪个外部类。
}
```

当我看到这的时候我很困惑，我如何能通过这个接口获得我构造的 SrvRes 类的类型参数。实际上我是无法获取到的，因为这个类我们没有实现 ParameterizedType 接口；也没有指定任何父类，所以说其直接的父类是 Object，Object 并没有实现 ParameterizedType 接口。

可是查明就网络上的很多文章，都能通过  getClass().getGenericSuperclass() 来获取泛型父类，以此来获取  ParameterizedType 接口。这是为何？

仔细想想也就明白了。因为 **类型擦除**的原因， SrvRes<T> 实际上运行时是 SrvRes<Object> 当然无法获取其泛型信息。

而对于 Res extends SrvRes<String> 其并没有泛型参数，当然可以获取其类型了。

```java
        public class Res extends SrvRes<Res> {
        }

        Res t = new Res();
        ParameterizedType pt = (ParameterizedType) t.getClass().getGenericSuperclass();
        System.out.println(pt.getActualTypeArguments()[0]);
        System.out.println(pt.getOwnerType());
        System.out.println(pt.getRawType());
        
```

输出：

```java
class Res
null
class SrvRes
```

## TypeVariable

```java
public interface TypeVariable<D extends GenericDeclaration> extends Type, AnnotatedElement {
    Type[] getBounds();
    D getGenericDeclaration();
    String getName();
     AnnotatedType[] getAnnotatedBounds();
}
```

## WildcardType

```java
public interface WildcardType extends Type {
    Type[] getUpperBounds();
    Type[] getLowerBounds();
}
```

##  GenericArrayType

```java
public interface GenericArrayType extends Type {
    Type getGenericComponentType();
}
```
# GenericDeclaration 接口

位于 *java.lang.reflect;* 包中。所有定义了类型变量的类都会实现这个接口。

```java
public interface GenericDeclaration extends AnnotatedElement {
    /**
     * Returns an array of {@code TypeVariable} objects that
     * represent the type variables declared by the generic
     * declaration represented by this {@code GenericDeclaration}
     * object, in declaration order.  Returns an array of length 0 if
     * the underlying generic declaration declares no type variables.
     *
     * @return an array of {@code TypeVariable} objects that represent
     *     the type variables declared by this generic declaration
     * @throws GenericSignatureFormatError if the generic
     *     signature of this generic declaration does not conform to
     *     the format specified in
     *     <cite>The Java&trade; Virtual Machine Specification</cite>
     */
    public TypeVariable<?>[] getTypeParameters();
}
```
# Class 类

Class 类位于 *java.lang* 包中，其实现了 Type 接口:

```java
public final class Class<T> implements java.io.Serializable,
                              GenericDeclaration,
                              Type,
                              AnnotatedElement {
    public String getTypeName() {
        if (isArray()) {
            try {
                Class<?> cl = this;
                int dimensions = 0;
                while (cl.isArray()) {
                    dimensions++;
                    cl = cl.getComponentType();
                }
                StringBuilder sb = new StringBuilder();
                sb.append(cl.getName());
                for (int i = 0; i < dimensions; i++) {
                    sb.append("[]");
                }
                return sb.toString();
            } catch (Throwable e) { /*FALLTHRU*/ }
        }
        return getName();
    }
}
```

演示一下方法的使用：

```java
        System.out.println(Gson.class.getTypeName());
        List<String> ls = new ArrayList<>();
        System.out.println(ls.getClass().getTypeName());
        Map<String,String> ms = new HashMap<>();
        System.out.println(ms.getClass().getTypeName());
        Map<String,List<String>> mls = new HashMap<>();
        System.out.println(mls.getClass().getTypeName());
```

输出是：

```
com.google.gson.Gson
java.util.ArrayList
java.util.HashMap
java.util.HashMap
```
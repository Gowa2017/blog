---
title: Spring中的IoC与DI
categories:
  - Java
date: 2019-06-16 21:29:37
updated: 2019-06-16 21:29:37
tags: 
  - Java
  - Spring
---
通过 Spring 的实现来了解一下 IoC （控制反转） 与 DI （依赖注入）。当然，之前看过的 Dagger 2 也是一个依赖注入的框架。

<!--more-->

# IoC

IoC (Inversion of Control) 控制反转，指的是对象的控制或程序的一部分被转移到了容器或者框架。经常这个概念是在面向对象的编程领域内讨论。

与传统编程相比，我们自己的代码会调用一些库情况下，IoC使框架能够控制程序的流程并调用我们的自定义代码。为了达到这个目的，框架使用了一些内建额外行为的抽象。**如果我们要自己的行为，我们就必须继承框架的类或以插件的形式提供。**

这样做的优点是：

- 将任务的执行和其实现解耦。
- 在不同实现间的切换更容易
- 模块化程序
- 通过隔离或模拟组件依赖让测试更容易，同时组件间通过约束来进行通信。

控制反转有很多方式能达到，我们主要看一下依赖注入（DI）。

# DI

DI 是我们用来实现 IoC 的一种模式，在这里将控制进行反转的控制是：**设置对象的依赖**。
这就是说，将对象与其他对象相连接，或将对象“注入”到其他对象，这是由一个组装器而不是由对象本身来完成的。

比如有个例子，*Store* 依赖于一个接口 *Item* 。我们传统的形式是下面实现：

```java
public class Store {
    private Item item;
  
    public Store() {
        item = new ItemImpl1();    
    }
}
```

上面的例子中，我们需要手动初始化和实现 *Item* 接口。

如果使用 DI 的话，我们就可以如下这样例子，而不用指定 *Item* 的实现。

```java
public class Store {
    private Item item;
    public Store(Item item) {
        this.item = item;
    }
}
```

在下面的章节中，我们会展示怎么样通过元数据来指定 *Item* 的实现。

# Spring IoC 容器

IoC 容器指的是框架中一个实现了 IoC 的普通角色。

在 Spring 框架中，IoC 容器由接口 *ApplicationContext* 表示。Spring 容器的责任是实例化，配置，组装对象（Beans）及管理他们的生命周期。

Spring 框架内部提供了这个接口的几种实现：*ClassPathXmlApplicationContext，FileSystemXmlApplicationContext* 针对本地应用，*WebApplicationContext* 针对 Web 应用。

为了组装 Beans，容器使用了配置元数据（XML形式或者注解形式）。

我们可以这样手动实例化一个容器：

```java
ApplicationContext context
  = new ClassPathXmlApplicationContext("applicationContext.xml");
```

在最开始的例子中，如果我们要设置 *item* 属性，我们也可以使用元数据。然后，容器就会读取这个元数据，在运行时用它们来组装 Beans。

**Spring 中的依赖注入（DI） 可以通过 构造器、setters、字段来实现**

# 基于构造器的 DI

如果使用基于构造器的 DI，容器会将我们要实例化的对象的依赖以参数的形式来调用其构造器。

Spring 主要是根据类型来解析参数，后面跟着属性的名称和没有歧义的索引。下面我们来看看一个 Bean 的配置和其使用了注解的依赖：

```java
@Configuration
public class AppConfig {
 
    @Bean
    public Item item1() {
        return new ItemImpl1();
    }
 
    @Bean
    public Store store() {
        return new Store(item1());
    }
}
```

*@Configuration* 注解表明这个类是一个 Bean 定义源。我们可以将其用到多个类上。

*@Bean* 注解用在方法上来定义了一个 Bean。如果我们不指定一个自定义的名字，那么 Bean 的 Id 默认就和方法名称相同。

对于一个默认是 **单例范围** 的 Bean，Spring 要先检查一下这个 Bean 是不是已经有一个缓存的实例存在，如果没有才会创建一个新的。如果我们使用 **原型作用域**，那么每次都会建立一个新的 Bean 实例。

与上面同样方式，不过是用 XML 来定义 Bean 如下：

```xml
<bean id="item1" class="org.baeldung.store.ItemImpl1" /> 
<bean id="store" class="org.baeldung.store.Store"> 
    <constructor-arg type="ItemImpl1" index="0" name="item" ref="item1" /> 
</bean>
```

# Setter DI

对于基于 Setter 的 DI，容器会在在调用了一个无参数的构造器后，调用我们类的 setter 方法。

```java
@Bean
public Store store() {
    Store store = new Store();
    store.setItem(item1());
    return store;
}
```
```xml
<bean id="store" class="org.baeldung.store.Store">
    <property name="item" ref="item1" />
</bean>
```

对于同样的 Bean，可以结合使用基于构造器的和 Setter 的 DI。Spring 文档推荐使用基于构造器的注入来处理强制的依赖，使用基于 Setter 来处理可选的依赖。

# 基于字段的 DI

可以通过 *@Autowired* 来进行字段注入。

```java
public class Store {
    @Autowired
    private Item item; 
}
```

在构建 *Store* 的时候，如果没有构造器或者setter 方法来注入 *Item* Bean，容器会使用反射来将 *Item* 注入到 *Store*。

当然我们也可以用 XML 来实现：

```xml

```

这个方式可能看起来更简单和清楚，但并不是推荐的用法：

- 使用反射来注入依赖，会更耗性能一些。
- 我们太过容易添加很多依赖。当我们使用构造器的时候，我们可能就会想如果我们添加了过多的依赖，我们的这个类是不是已经违反来**单一责任原则**。

# 自动装配依赖关系

**装配** 操作允许 Spring 容器在相互协做的 Bean 间通过检查已定义的 Bean 来解析依赖关系。

在 XML 配置中有四种模式可以用来自动装配一个 Bean。

- **no**：默认值。意味这不需要自动装配，我们必须显式的命名依赖。
- **byName**：根据属性的名称去解析。因此，Spring 会用需要设置的属性名称去依赖查找 Bean。
- **byType**：与 **byName** 类似，只是根据属性的类型去。如果有多个 Bean 有同样的类型就会抛出异常。
- **constructor**：根据构造器的参数来，这意味着，Spring 要查找的是和构造器参数有相同类型的 Bean。


现在，我们在上面的例子中，来自动将 *item1* 注入到 *Store* Bean 内：

```java
@Bean(autowire = Autowire.BY_TYPE)
public class Store {
     
    private Item item;
 
    public setItem(Item item){
        this.item = item;    
    }
}
```

或者直接在属性上指定自动注入：

```java
public class Store {
     
    @Autowired
    private Item item;
}
```

如果有多个 Bean 有相同的类型，那么我们需要加上 *@Qualifier* 注解来通过名字引用 bean：

```java
    @Autowired
    @Qualifier("item1")
    private Item item;
```

通过 XML 文件来实现：

```xml
<bean id="store" class="org.baeldung.store.Store" autowire="byType"> </bean>

```

```xml
<bean id="item" class="org.baeldung.store.ItemImpl1" />
 
<bean id="store" class="org.baeldung.store.Store" autowire="byName">
</bean>
```

# @Autowired, @Resource, @Inject

这几个注解可以让类以声明的形式来解析依赖：

```java
@Autowired
ArbitraryClass arbObject;
```

而不用直接的实例化它：

```java
ArbitraryClass arbObject = new ArbitraryClass();
```

三个注解中有两个是属于包 *javax.annotation.Resource, javax.inject.Inject*。而 *@Autowired* 是属于包 *org.springframework.beans.factory.annotation*。

这三个注解都可以通过字段注入或者 Setter 注入来解析依赖。下面会有区分一下这三者的不同。

## 区别

@Autowired和@Inject基本是一样的，因为两者都是使用AutowiredAnnotationBeanPostProcessor来处理依赖注入。但是@Resource是个例外，它使用的是CommonAnnotationBeanPostProcessor来处理依赖注入。当然，两者都是BeanPostProcessor。

@Autowired和@Inject
默认 autowired by type
可以 通过@Qualifier 显式指定 autowired by qualifier name。

@Resource
默认 autowired by field name
如果 autowired by field name失败，会退化为 autowired by type
可以 通过@Qualifier 显式指定 autowired by qualifier name
如果 autowired by qualifier name失败，会退化为 autowired by field name。但是这时候如果 autowired by field name失败，就不会再退化为autowired by type了。

建议多用 @Inject，这是 JSR 330 的规范。@Autowired 是 Spring 的实现，比如 Dagger 就用不了。@Resource 是 JSR 250 是比较老的规范。
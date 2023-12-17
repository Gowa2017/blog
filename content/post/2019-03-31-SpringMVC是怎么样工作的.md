---
title: SpringMVC是怎么样工作的
categories:
  - Java
date: 2019-03-31 10:10:17
updated: 2019-03-31 10:10:17
tags: 
  - Java
  - Spring
  - SpringMVC
---

偶尔看到一篇对 SpringMVC 讲得很好的文章。原文地址在 [https://stackify.com/spring-mvc/](https://stackify.com/spring-mvc/)

<!--more-->

# Servlet

Tomcat 是一个 Servlet 容器，很自然地，每个发送到 Tomcat 网页服务器的请求都会一个 Java Servlet 处理。所以呢 Spring 网页应用的入口就是 Servlet。

简单来说，一个 Servlet 就是 一个 Java 网页应用中的一个核心组件；其是一个底层概念，不会在特定的编程模式上涉及太多。

一个 HTTP Servlet 只能接收一个 HTTP 请求，处理，然后发回一个响应。

# DispatcherServlet

DispatcherServlet 是 Spring MVC 的核心。

对于一个网页应用程序开发者来说最想做的事情就是把下面这些非常无聊而重复的任务和有用的业务逻辑分开：

- 将一个 HTTP 请求映射到一个特定的处理方法
- 解析 HTTP 的请求数据和请求头到 DTOs（数据传输对象） 或者 domain 对象。
- MVC 间的交互
- 对 DTOs, domain 对象生成响应。


DispatcherServlet 干的就是上面这些事情，   其就是 Spring MVC 的核心，接收所有到我们应用请求的核心组件。

DispatcherServlet 是可扩展的，其允许我们对很多任务插入已存在的或者新的适配器。

- 将请求映射到一个处理此请求的类或方法（实现 HandlerMapping 接口）。
- 使用一个特定的模式来处理请求，比如一个常规 servlet，一个复杂点的 MVC　工作流，或者只是一个　POJO bean 中的方法。（实现 HandlerAdapter 接口）
- 通过名称来解析 Views，允许我们使用不同的模板引擎，XML,XSLT或者其他的 View 技术（实现 ViewResolver 接口）
- 使用默认的 Apache Common File uploading 实现来解析多部分的请求，或者，编写我们自己的 *MultipartResolve*。
- 使用 LocaleResolver 来解析 locale。包括 cookie, session, header等。

# HTTP 请求处理过程
首先，让我们来追踪一下到我们的控制层中的并返回到浏览器（前端）的简单 HTTP 请求，

DispatcherServlet 的继承层级很深，但很值得深入探究一下。

```
GenericServlet 
    <- HttpServlet 
        <- HttpServletBean 
            <- FrameworkServlet 
                <-DispatcherServlet
```


## GenericServlet

GenericServlet 是 Servlet 规范的一部分，并不直接关注 HTTP。你定义了一个接收所有的请求并产生响应的 `service()` 方法。


```java
public abstract void service(ServletRequest req, ServletResponse res) 
  throws ServletException, IOException;
```

*ServletRequest，ServletResponse* 方法参数与 HTTP 协议没有什么关系。

每个请求到达服务器的时候都会调用这个方法。


## HttpServlet

HttpServlet 关注 HTTP 的实现，也是规范的一部分。

更实际点来说，HttpServlet 是一个实现了 `service()` 方法的抽象类。这个方法，通过 HTTP 的方法类型来分隔请求，看起来似乎是这样：

```java
protected void service(HttpServletRequest req, HttpServletResponse resp)
    throws ServletException, IOException {

    String method = req.getMethod();
    if (method.equals(METHOD_GET)) {
        // ...
        doGet(req, resp);
    } else if (method.equals(METHOD_HEAD)) {
        // ...
        doHead(req, resp);
    } else if (method.equals(METHOD_POST)) {
        doPost(req, resp);
        // ...
    }
```
## HttpServletBean

HttpServletBean 是在层级中第一个 Spring 关注的类。 Spring 使用 web.xml 或 WebApplicationInitializer 中的 *init-param* 值来注入 bean 的属性。

对于到达应用的请求，`doGet(), doPost()` 会根据请求类型来被调用。

## FrameworkServlet

FrameworkServlet 用一个 应用程序上下文来整合了 Servlet 功能，实现了 *ApplicationContextAware* 接口。当然，创建一个自己的 应用程序上下文也是可以的。

上面说到，HttpServletBean 会用 init-params 来注入 bean 属性。所以，如果在此 servlet 中一个上下文类名在 *contextClass* init-params 参数中提供，那么指定的类就会创建一个实例来作为应用的上下文。否则的话，使用一个默认的 XmlWebApplicationContext 实例。

一个 XML 配置已经过时了， Spring Boot 默认 通过  AnnotationConfigWebApplicationContext 来配置 DispatcherServlet 。当然，我们也可以进行修改。


## DispatcherServlet 统一请求处理

`HttpServlet.service()`方法通过 HTTP 的动词类型（get,post）来路由请求，在底层 servlet 上环境中工作得很好。然而，在 Spring MVC 这个级别的抽象，方法类型只是可以用来进行映射请求到处理器的一个参数。

因此，FrameworkServlet 类的另外一个主要功能是将处理逻辑放到单一的 `processRequest()` 方法中，其会按序调用 `doService()`方法。


```
@Override
protected final void doGet(HttpServletRequest request, 
  HttpServletResponse response) throws ServletException, IOException {
    processRequest(request, response);
}

@Override
protected final void doPost(HttpServletRequest request, 
  HttpServletResponse response) throws ServletException, IOException {
    processRequest(request, response);
}

// …

```

## DispatcherServlet: 丰富请求

最后，`DispatcherServlet()` 实现了 `doService()` 方法。在这里，其会给请求添加上一些有用的对象，在接下来的处理过程中有利于我们的处理过程：应用上下文，区域解析器，主题解析器，主题源等等。

```java
request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, 
  getWebApplicationContext());
request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());
```

同样，`doService()`方法会这样好输入和输出快速映射(Flash Map)。 Flash Map 基本上是一个用来在请求间立刻传递参数的模式。这在有的时候会只有用，比如重定向。

```java
FlashMap inputFlashMap = this.flashMapManager
  .retrieveAndUpdate(request, response);
if (inputFlashMap != null) {
    request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, 
      Collections.unmodifiableMap(inputFlashMap));
}
request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
```

接着 `doService()` 会调用 `doDispatch()` 方法进行请求分发。

## DispatcherServlet：请求分发

`dispatch()` 的主要目的是：找出请求的处理器，将请求/响应参数传递过去。 handler 可以是任意类型的对象，而不局限于一个特定的接口。这就意味着，Spring 需要找到一个知道如何与 handler 对话的 adapter。

为了找到与请求匹配的 handler， Spring 会遍历所有已注册的  *HandlerMapping* 接口的实现。实现的类型有很多种。

SimpleUrlHandlerMapping 允许将一个请求的URL 映射到一个特定的处理 bean。例如，可以通过将一个 *java.util.Properties*实例注入到 *mappings* 属性中：

```java
/welcome.html=ticketController
/show.html=ticketController
```

可能使用最广泛的 handler 映射是 *RequestMappingHandlerMapping*了，其将一个请求映射到一个*@Controller* 注解的类中，一个 *@RequestMapping* 注解的方法上。

注意，Spring 关注的方法是用 *@GetMapping, @PostMapping* 对应注解的。这些注解，通过 *@RequestMapping* 元注解来标记。

`doDispatch()` 方法也会关心一些其他 HTTP　相关的任务。

## 请求处理

现在，Spring 决定了请求的handler，业绩 handler 的 adapter，是时候最终处理这个请求了。下面是 `HandlerAdapter.handle()` 方法的签名。注意到 handler 可以选择怎么处理请求很重要：

- 向响应写入数据或者返回 null
- 返回一个 ModleAndView 对象，由 DispatcherServlet 渲染

```java
@Nullable
ModelAndView handle(HttpServletRequest request, 
                    HttpServletResponse response, 
                    Object handler) throws Exception;
```

这里有几种类型的 handler。

下面演示的是 SimpleControllerHandlerAdapter 如何处理一个 Spring MVC 中控制器实例的（不要与 @Controller 注解的 POJO 混淆）。

控制器中的处理器返回的 *ModelAndView* 对象不会被自身渲染，而是由 DispatchServlet 渲染：

```java
public ModelAndView handle(HttpServletRequest request, 
  HttpServletResponse response, Object handler) throws Exception {
    return ((Controller) handler).handleRequest(request, response);
}
```


第二个是 SimpleServletHandlerAdapter，它会将一个常规的 Servlets 改为一个请求 handler。

一个 *Servlet* 对于 ModelAndView 是一点也不清楚的，其只是简单的处理一下请求，将结果渲染到响应对象中。所以，这个 adapter 返回 null 而不是 ModelAndView。

```java
public ModelAndView handle(HttpServletRequest request, 
  HttpServletResponse response, Object handler) throws Exception {
    ((Servlet) handler).service(request, response);
    return null;
}
```

在我们的情况中，控制器是一个 POJO，然后方法上使用了 *@RequestMapping* 注解，所以每个 handler 都是控制器类的一个方法。对于这样的 handler 类型， Spring 使用 *RequestMappingHandlerAdapter* 类。
## 处理参数及Handler方法的返回值

通常，控制器中的 handler 方法，并不会使用  *HttpServletRequest, HttpServletResponse* 作为参数，而是会使用很多其他的类型，比如 domain 对象，路径参数等等。

同时，在控制器方法中，我们并不一定需要返回一个 ModleAndView 实例。我们可能会返回一个 View 名称，一个 *ResponseEntity* 或者一个将会被转换为 JSON 响应的 POJO。

*RequestMappingHandlerAdapter* 保证 handler 方法的参数从 HttpServletRequest 内进行解析，同时，根据 handler 方法的返回值来创建 ModelAndView。

下面这些位于 RequestMappingHandlerAdapter 中的代码保证了上述的实现：

```java
ServletInvocableHandlerMethod invocableMethod 
  = createInvocableHandlerMethod(handlerMethod);
if (this.argumentResolvers != null) {
    invocableMethod.setHandlerMethodArgumentResolvers(
      this.argumentResolvers);
}
if (this.returnValueHandlers != null) {
    invocableMethod.setHandlerMethodReturnValueHandlers(
      this.returnValueHandlers);
}
```

*argumentResolvers* 对象是一个不同 *HandlerMethodArgumentResolver* 实例的复合。

这里有超过 30 种不同的参数解析器实现。他们允许从请求中解压出任何形式的信息，同时将其提供给 handler 方法。参数包括： URL 路径变量，请求体参数，请求头，cookies, session 等。

*returnValueHandlers* 对象是 *HandlerMethodReturnValueHandler* 对象的复合。同样也有很多种不同的 value handlers，他们能处理方法的返回值，然后创建 adapter 需要的 ModelAndView 对象。

具体而言，但我们从 `hello()` 方法中返回字符串，使用的是 *ViewNameMethodReturnValueHandler* 来处理返回值。但是当我们从 `login()` 返回一个 *ModelAndView* 的时候，Spring 使用的是 *ModelAndViewMethodReturnValueHandler* 来进行处理。

## 渲染 View

到现在，Spring 已经处理完 HTTP 请求，同时有了一个 ModelAndView 对象，接下来就必须渲染出 HTML 网页到浏览器给用户。这个基于 ModelAndView 对象中的 model 及封装好的视图。

同时要注意到，我们可能会渲染一个 JSON 对象，或者  XML，或者其他任何能通过 HTTP 协议传输的类型数据。我们会在接下来的REST-一节中进行介绍。

回到 *DispatcherServlet* ，`render()` 方法会首先通过 *LocaleResolver* 实例来设置区域。我们现在来假设我们使用的浏览器正确的设置了 *Accpet* 头，那么默认情况下会使用 *AcceptHeaderLocaleResolver*。

在渲染期间，*ModelAndView* 对象可能已经包含了一个对选定 View 的引用，或者只是一个 View 名称，也可能什么都没有（此时依赖于一个默认的 view）。

因为 `hello(), login()` 方法都将期望的视图通过字符串名称来指定，必须通过这个名称来进行查找。所以，这就是 *ViewResolver* 列表开始表演的地方。

```java
for (ViewResolver viewResolver : this.viewResolvers) {
    View view = viewResolver.resolveViewName(viewName, locale);
    if (view != null) {
        return view;
    }
}
```

这是一个 *ViewResolver* 实例的列表，包括了 *ThymeleafViewResoleve*（thymeleaf-spring5集成库提供）。这个 resolver 知道去哪里查找 View，然后返回对应的 View 实例。

在调用了 View 的 `render()` 方法后，Spring 通过将 HTML 页面发送到用户的浏览器来完结请求。

## REST 支持

在典型的 MVC 场景之外，我们一样可以用这个框架来建立 REST 网页服务。

简单来说，你可以接受一个资源作为输入，指定一个 POJO 作为方法参数，用 *@RequestBody* 进行注解。也可以将此方法加上 *@ResponseBody* 注解来表示其结果必须直接通过转换为一个 HTTP 响应。

```java
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.ResponseBody;

@ResponseBody
@PostMapping("/message")
public MyOutputResource sendMessage(
  @RequestBody MyInputResource inputResource) {
    
    return new MyOutputResource("Received: "
      + inputResource.getRequestMessage());
}
```

为了将内部的 DTOs 转换为 REST 表示，框架使用了 *HttpMessageConverter* 架构。例如，实现中的一个例子是 *MappingJackson2HttpMessageConverter*，其能将 model 对象和 JSON 对象间通过 Jackson 库进行转换。

为了简化 REST API 的创建，Spring 介绍了 *@RestController* 注解。 这对默认会使用 *@ResponseBody* 语义来说非常的方便，避免了在每个 REST 控制器上进行显式的设置。

```java
import org.springframework.web.bind.annotation.RestController;

@RestController
public class RestfulWebServiceController {

    @GetMapping("/message")
    public MyOutputResource getMessage() {
        return new MyOutputResource("Hello!");
    }
}
```

---
title: Retrofit下载文件及ResponseBody解析
categories:
  - Android
date: 2019-06-11 23:29:34
updated: 2019-06-11 23:29:34
tags: 
  - Android
  - Retrofit
  - Java
---
看多数文章都说明用 Retrofit 来下载大文件的时候需要用 *@Streaming* 注解来避免 OOM，当下载的文件的时候，返回的值是一个 ResponseBody。事实上，很少有文章说明，这背后到底是怎么工作的。

<!--more-->

# 从 call 看起

从 Retrofit 的 call 接口来看，我们通常使用的方式是这样的：

```java
    @GET
    @Streaming
    Call<ResponseBody> download(@Url String url);
```
上面的代码中我们定义了一个 Retrofit 方法 `download()`，对每个 Retrofit 方法的调用都会发送一个请求到 web 服务器，然后得到一个响应。

我们的调用，会产生其对应的 HTTP 请求和响应。

```java
public interface Call<T> extends Cloneable {
  /**
   * Synchronously send the request and return its response.
   *
   * @throws IOException if a problem occurred talking to the server.
   * @throws RuntimeException (and subclasses) if an unexpected error occurs creating the request
   * or decoding the response.
   */
  Response<T> execute() throws IOException;

  /**
   * Asynchronously send the request and notify {@code callback} of its response or if an error
   * occurred talking to the server, creating the request, or processing the response.
   */
  void enqueue(Callback<T> callback);

  /**
   * Returns true if this call has been either {@linkplain #execute() executed} or {@linkplain
   * #enqueue(Callback) enqueued}. It is an error to execute or enqueue a call more than once.
   */
  boolean isExecuted();

  /**
   * Cancel this call. An attempt will be made to cancel in-flight calls, and if the call has not
   * yet been executed it never will be.
   */
  void cancel();

  /** True if {@link #cancel()} was called. */
  boolean isCanceled();

  /**
   * Create a new, identical call to this one which can be enqueued or executed even if this call
   * has already been.
   */
  Call<T> clone();

  /** The original HTTP request. */
  Request request();
}
```

对于每个 Retrofit 方法来讲，泛型 `T` 代表了要返回的响应体类型。(Response body type。)

# Response

需要先注意的是， okhttp 与 Retrofit 都有 Response 这个类，但实际上 Retrofit.Response 只是对 okhttp.Response 的封装，提供了一些更简单好用的方法而已。

okhttp.Response 是一个对 HTTP　协议响应的一个完整 Java 表示。其内部持有相应码，请求信息等等内容：

```java
public final class Response implements Closeable {
  final Request request;
  final Protocol protocol;
  final int code;
  final String message;
  final @Nullable Handshake handshake;
  final Headers headers;
  final @Nullable ResponseBody body;
  final @Nullable Response networkResponse;
  final @Nullable Response cacheResponse;
  final @Nullable Response priorResponse;
  final long sentRequestAtMillis;
  final long receivedResponseAtMillis;
}
```
# ResponseBody

这是 OkHttp 定义的类。其是一个字节流，代表了从服务器到客户端的一个一次性的流。其背后都一个活跃的到 web 服务器的连接在支持。

ResponseBody 所代表的流只能消费一次。

**ResponseBody 必须被关闭**。因为每个 ResponseBody 都是有受限的资源来支持的（如套接字数量，打开文件数量），如果不关闭，就会造成资源泄漏，造成性能的下降。

这个可以用来流式操作很大的响应。不论是超过了内存大小的响应，甚至还是超过了本地存储大小的响应（如视频播放）这都是可以的。**因为这个类不会将响应完全缓存到内存**，所以是无法再次阅读这个响应里字节内容。

可以用 `bytes(),string()` 方法来将整个响应读到内存；或者用 `source(), byteStream(), charStream()` 来将响应变成一个流。

通常，就 HTTP1.1 来说，如果服务端返回的数据长度是已知的，在接收完数据后，就由客户端主动关闭连接结束通信。这有一个潜在的意思就是，在数据没有接收完之前，客户端与服务端的 TCP 连接是不会断开的。

有一种情况，客户端将永远也不会主动断开连接。也即：客户端返回的内容 contentLength 字段未知，而且也没有 chunked 标识。但一般来说都不会有这样的情况存在。

# 思考

其实应该是我想得太多，Retrofit/OkHTTP 所言，ResponseBody 不会将数据全部缓存到内存，说的是系统的用户态。

在内核态，确切的来说，是在 TCP 协议栈的实现来说，是完全会将服务端发送的数据放到协议栈内，然后分发到应用程序的。我们从 ResponseBody 读取数据，事实上也就是向内核协议栈拿数据，将内核态中内存的内容取到用户区去。

OK，那么就算了解了这背后的道理了，想太多也不是好事。

# Streaming

对于 Streaming 而言，就是说要将返回的值流式化，而不读进内存，由用户来决定如何从流读取，最终关闭流。

在我们 build  Retrofit 实例的时候，添加了默认的 Converts:

```java
      // Add the built-in converter factory first. This prevents overriding its behavior but also
      // ensures correct behavior when using converters that consume all types.
      converterFactories.add(new BuiltInConverters());
      converterFactories.addAll(this.converterFactories);
      converterFactories.addAll(platform.defaultConverterFactories());
```

对于 BuiltInConverters 而言，其会根据注解来决定，是使用哪一种类型的 Converter：

```java
  @Override public @Nullable Converter<ResponseBody, ?> responseBodyConverter(
      Type type, Annotation[] annotations, Retrofit retrofit) {
    if (type == ResponseBody.class) {
      return Utils.isAnnotationPresent(annotations, Streaming.class)
          ? StreamingResponseBodyConverter.INSTANCE
          : BufferingResponseBodyConverter.INSTANCE;
    }
    if (type == Void.class) {
      return VoidResponseBodyConverter.INSTANCE;
    }
    if (checkForKotlinUnit) {
      try {
        if (type == Unit.class) {
          return UnitResponseBodyConverter.INSTANCE;
        }
      } catch (NoClassDefFoundError ignored) {
        checkForKotlinUnit = false;
      }
    }
    return null;
  }
```

当我们的返回类型是 ResponseBody ，同时加上了 `@Streaming` 注解的时候，就会返回 StreamingResponseBodyConverter。

对比  StreamingResponseBodyConverter 与 BufferingResponseBodyConverter 的 convert() 方法即可知晓其中的道理：

```java

    @Override public ResponseBody convert(ResponseBody value) {
      
      
    @Override public ResponseBody convert(ResponseBody value) throws IOException {
      try {
        // Buffer the entire body to avoid future I/O.
        return Utils.buffer(value);
      } finally {
        value.close();
      }
    }
```
前者会直接返回 ResponseBody 这个一次性的流，而后者，则会缓存到内存区；两者之后都会关闭 ResponseBody。

区别主要就是在这里了。

# 文件的下载

那么这个时候就很简单了，将 ResponseBody 调用其中一些方法，变成一个输入流，然后写到输出流就OK了。

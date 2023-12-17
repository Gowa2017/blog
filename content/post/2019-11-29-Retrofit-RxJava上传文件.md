---
title: Retrofit-RxJava上传文件
categories:
  - Android
date: 2019-11-29 16:53:03
updated: 2019-11-29 16:53:03
tags: 
  - Android
  - RxJava
  - Retrofit
---

在 {% post_link 利用RxJava-Retrofit来下载文件遇到的坑 利用RxJava-Retrofit来下载文件遇到的坑 %} 我们已经可以在下载文件，将整个流程弄成链式的了，但是在上传文件就不会这么容易，有不少地方需要注意。

<!--more-->

这和我们的 Retrofit 与 OkHttp 的接驳有关。在 {% post_link Retrofit-OkHttp请求过程-源码阅读md Retrofit-OkHttp请求过程-源码阅读md %} 中我们看到，Retrofit 会

1. 根据  API 接口生成一个代理实现，这个代理实现的所有接口中的方法都是 HttpServiceMethod（更确切一点，是HttpServiceMethod 的子类 CallAdapted 其持有一个建立的 CallAdapter ）。
2. 调用这个代理实现的方法，就会通过反射调用 HttpServiceMethod（CallAdapted），将一个 OkHttpCall 调用 callAdapter.adapt(call) 进行适配后返回。

# RxJava2CallAdapter

而对于我们的 RxJava2CallAdapter，其 adapt 做的事情就是将我们的 Retrofit 的 OkHttpCall 适配成一个 Observable。最终返回的是一个 ResultObservable。
这个可以我们可以看一下这个调用链调：

OkHttpCall -> CallEnqueueObservable(CallExecuteObservable) -> BodyObservable。

我们的 subscribe 就是对 BodyObservable 进行订阅的。

```java
  @Override public Object adapt(Call<R> call) {
    Observable<Response<R>> responseObservable = isAsync
        ? new CallEnqueueObservable<>(call)
        : new CallExecuteObservable<>(call);

    Observable<?> observable;
    if (isResult) {
      observable = new ResultObservable<>(responseObservable);
    } else if (isBody) {
      observable = new BodyObservable<>(responseObservable);
    } else {
      observable = responseObservable;
    }

    if (scheduler != null) {
      observable = observable.subscribeOn(scheduler);
    }

    if (isFlowable) {
      return observable.toFlowable(BackpressureStrategy.LATEST);
    }
    if (isSingle) {
      return observable.singleOrError();
    }
    if (isMaybe) {
      return observable.singleElement();
    }
    if (isCompletable) {
      return observable.ignoreElements();
    }
    return RxJavaPlugins.onAssembly(observable);
  }

```

# OkHttpCall

当我们对这个 call 调用 `execute(), enqueue()` 方法的时候，最终，它都会建立一个 OkHttp3 的 RealCall，然后开发发送请求。

在 RealCall 中会添加多个拦截器，最先返回的那个是 CallServerInterceptor 在这里面，其会将请求的头，请求的 Body 写到套接字上面去（从 Java 这来看是写到 Http1CodeC 建立的 RequestBody 去）

这个读取，是从我们最上层建立的 RequestBody 处进行读取，然后写入。网络上大多数的思路，都是通过将 RequestBody 的 writeTo 方法进行重写。

# RequestBody

对于一个文件，其构建的 RequestBody 是这样的：

```java
  /** Returns a new request body that transmits the content of {@code file}. */
  public static RequestBody create(final @Nullable MediaType contentType, final File file) {
    if (file == null) throw new NullPointerException("file == null");

    return new RequestBody() {
      @Override public @Nullable MediaType contentType() {
        return contentType;
      }

      @Override public long contentLength() {
        return file.length();
      }

      @Override public void writeTo(BufferedSink sink) throws IOException {
        Source source = null;
        try {
          source = Okio.source(file);
          sink.writeAll(source);
        } finally {
          Util.closeQuietly(source);
        }
      }
    };
  }
```

如果我们重写 WriteTo 方法，每次按一定的量进行读取，然后嘟嘟后就把消息给一个回调函数，这是可以的，但是有没有更好的办法呢？

想了很久感觉是没有的，因为我们是无法侵入到 CallServerInterceptor 里面去进行操作的。

对于订阅的 BodyObservable 来说，我们是想知道的这次调用网络接口是成功还是失败。而不是想知道其上传了多少，毕竟这不是单独为了我们的上传文件而设计的东西。

简而言之，只有CallServerInterceptor 处理后返回到 OkHttpCall 的数据，BodyObservable 才能将其转换为事件往下发射。而我们需要的是在 CallServerInterceptor 处理的过程中就发射事件，而且不可以发射到 OkHttpCall 所以常规的思路肯定就是实现不了。

而且对于 RequestBody 读肯定是在 IO 线程的，其也要涉及到线程的切换来通知进度。

# 实现
在国外网站上找到了一个比较链式一点的实现 [https://medium.com/@PaulinaSadowska/display-progress-of-multipart-request-with-retrofit-and-rxjava-23a4a779e6ba](https://medium.com/@PaulinaSadowska/display-progress-of-multipart-request-with-retrofit-and-rxjava-23a4a779e6ba)

其实现的原理也是在 RequestBody 中重写方法，不过是将事件封装到下一个 Flowable 中，不再使用异步调用发起请求，而是采用  blockingGet 拿到请求的返回结果，然后再抛回来。



# HTTP 协议 multipart

我们可以 Post 多种类型的内容，这些内容会放在 Body 里面，但是这些内容是什么类型，则由 header 里面的 content-type 来进行指定。

> 协议规定：如果发送者发送的消息里面有 body，其必须在 Header 中生成一个 Content-Type 字段，除非它不知道。如果 Content-Type 头字段没有设置，那么接收方就会将其当做是 application/octet-stream 来处理或通过测试来决定其类型。

关于所有类型的完整定义在 [rfc2046 ](https://tools.ietf.org/html/rfc2046)。

对于 content-type  是 multipart 其会拥有多个子类型。[rfc2046 5.1 节](https://tools.ietf.org/html/rfc2046#section-5.1) 对其进行了描述。

对于类型是 multipart 的，在 content-type 字段还需要设置一个参数 **boundary**，然后会定义一个定界分隔符：完全由两个 `-` 符号组成的线，加上 **boundary**。对于里面的每个part 我们都可以指定其类型，及相关参数如：

```
POST /foo HTTP/1.1
Content-Length: 68137
Content-Type: multipart/form-data; boundary=---------------------------974767299852498929531610575

-----------------------------974767299852498929531610575
Content-Disposition: form-data; name="description" 

some text
-----------------------------974767299852498929531610575
Content-Disposition: form-data; name="myFile"; filename="foo.txt" 
Content-Type: text/plain 

(content of the uploaded file foo.txt)
-----------------------------974767299852498929531610575--
```

实际上有两种类型的 multipart ， formdata 和 byterange 不过后一种我们用得少，所以先不管它。



# MultipartBody

MultipartBody 是 RequestBody 的子类，因为其有多个 Part 所以其还有一个内部类 Part，每个 Part 都是一个 RequestBody 加上此 Part 对应的信息如：

```
Content-Disposition: form-data; name="myFile"; filename="foo.txt"  这是 PartHeader 
Content-Type: text/plain  

(content of the uploaded file foo.txt)  Part 中的 RequestBody
```

我们现在要做的就是在一个 RequestBody 上封装一个 CountingRequestBody ，重写且在写出数据的方法，回调进行通知。事实本质上还是进行了回调。



这样做，依然还有 个问题，就是当我们从文件内读取数据，然后写入到 套接字的时候，如果IO快的话，会很快就返回到100%，但这个时候，网络的传输并没有完成，就会出现进度100%，然后却等几秒才有反应的问题。
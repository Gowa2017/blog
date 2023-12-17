---
title: 利用RxJava-Retrofit来下载文件遇到的坑
categories:
  - [RxJava]
  - [Android]
date: 2019-11-27 14:03:06
updated: 2019-11-27 14:03:06
tags: 
  - RxJava
  - Retrofit
  - Android
---
在文章 {% post_link 使用Retrofit-Rxjava来上传下载文件 使用Retrofit-Rxjava来上传下载文件 %} 大概对如何用 RxJava 配合下载文件做了一个介绍，但是最近想着，我们完全是可以将显示的进度，当作是事件发射出来的，但其中遇到了几个坑。

<!--more-->

我们实现的原理是这么一个事实：**对于OkHttpClient** 返回的数据 ResponseBody 实际上并不是包含真正的数据，从一个比较高的角度来看，实际上我们可以把它看成是一个封装了 套接字 的对象，我们可以从 ResponseBody 封装的套接字上来读取数据，读取后就写入文件，显示进度。



# RxServiceGenerator

首先我们建立一个 Service 的生产类，用它来根据接口，构造 ServiceMethod。

```java
public class RxServiceGenerator {

    private static Retrofit.Builder sBuilder = new Retrofit.Builder()
            .baseUrl("https://storage.qoo-app.com/")
            .addConverterFactory(GsonConverterFactory.create())
            .addCallAdapterFactory(RxJava2CallAdapterFactory.createAsync());
    private static Retrofit mRetrofit = sBuilder.build();
    private static OkHttpClient.Builder httpClient = new OkHttpClient.Builder()
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS);
    //添加一个日志拦截器，这个是给客户端 OkHttpClient 添加的
    private static HttpLoggingInterceptor logging = new HttpLoggingInterceptor()
            .setLevel(HttpLoggingInterceptor.Level.BODY);

    public static <S> S createService(Class<S> serviceClass) {
        if (!httpClient.interceptors().contains(logging)){
            httpClient.addInterceptor(logging);
            sBuilder.client(httpClient.build());
            mRetrofit = sBuilder.build();
        }
        return mRetrofit.create(serviceClass);
    }

}

```

其中，我们在 OkHttpClient 上挂了一个 [Logging Interceptor](https://github.com/square/okhttp/tree/master/okhttp-logging-interceptor) 这个是由 Squareup 官方出的一个拦截器，用来显示请求和响应的日志信息。



# DownloadUtils

接下来构造我们的下载类。这里用到了  [https://github.com/Blankj/AndroidUtilCode](https://github.com/Blankj/AndroidUtilCode) 这个 Android 的 Utils 库。

```java
public class DownloadUtils {
    private static final String TAG = "DownloadUtils";
    private static final DownInterface api = RxServiceGenerator.createService(DownInterface.class);

    public static Observable<String> download(String url) {
        return api
                .down(url)
                .flatMap( r -> writeToFile(r,url));

    }

    public static Observable<String> writeToFile(ResponseBody res, String url) {
        return Observable.create(emitter -> {
            Source source = res.source();
            BufferedSink sink = Okio.buffer(Okio.sink(getFile(url)));
            Buffer buffer = new Buffer();
            long total = res.contentLength();
            long downloaded = 0, tosave = 0;
            while ((tosave = source.read(buffer, 8192)) != -1) {
                buffer.readAll(sink);
                downloaded += tosave;
                Long progress = downloaded * 100 / total;
                emitter.onNext(String.format("%d", progress.intValue()));
            }
            emitter.onComplete();
        });
    }

    public static File getFile(@NonNull String url) throws IOException, SecurityException {
        return getSaveFile(getToSavePath(getFileNameByUrl(url)));

    }

    public static String getFileNameByUrl(@NonNull String url) {
        if (TextUtils.isEmpty(url)) {
            return "";
        }
        int pos = url.lastIndexOf("/");
        return url.substring(pos);
    }

    public static String getToSavePath(String fileName) {
         return PathUtils.getExternalAppDownloadPath() + File.separator + fileName;
    }

    public static File getSaveFile(String savePath) throws IOException, SecurityException {
        File f = new File(savePath);
        if (f.exists()) {
            f.delete();
        }
        f.createNewFile();
        return f;
    }

    interface DownInterface {
        @Streaming
        @GET
        Observable<ResponseBody> down(@Url String url);
    }
}

```

在这里我们在 writeToFile 过程中，会将进度信息当做事件发射出去，一旦完成，就发射一个 OnComplete 事件。



# 使用

这样我们就在 Activity 里面进行使用了，我就随便以一个地址来进行测试：

```java
        String REQ_URL = "https://storage.qoo-app.com/apk/com.qooapp.qoohelper/com.qooapp.qoohelper-20191119170001.apk";
        DownloadUtils.download(REQ_URL)
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(nt -> {
                            mProgressBar.setProgress(Integer.parseInt(nt));
                            mTvProgress.setText(String.format("%s%%", nt));
                        },
                        e -> ToastUtils.showShort(e.getMessage()),
                        () -> mTvProgress.setText("下载完成")
                );

```

但是出乎意料，下载的时候不会显示进度，而会在最终将进度一并显示出来。查看日志发现了问题：

```java
2019-11-27 14:13:14.264 8075-26788/com.example.mytext D/OkHttp: <-- 200  https://storage.qoo-app.com/apk/com.qooapp.qoohelper/com.qooapp.qoohelper-20191119170001.apk (760ms)
2019-11-27 14:13:14.265 8075-26788/com.example.mytext D/OkHttp: content-type: application/vnd.android.package-archive
2019-11-27 14:13:14.265 8075-26788/com.example.mytext D/OkHttp: content-length: 21136988
2019-11-27 14:13:14.265 8075-26788/com.example.mytext D/OkHttp: last-modified: Tue, 19 Nov 2019 09:00:03 GMT
2019-11-27 14:13:14.265 8075-26788/com.example.mytext D/OkHttp: accept-ranges: bytes
2019-11-27 14:13:14.266 8075-26788/com.example.mytext D/OkHttp: server: AmazonS3
2019-11-27 14:13:14.266 8075-26788/com.example.mytext D/OkHttp: date: Tue, 26 Nov 2019 09:12:27 GMT
2019-11-27 14:13:14.266 8075-26788/com.example.mytext D/OkHttp: etag: "837724425faaa6c21e331e6b3d25e057"
2019-11-27 14:13:14.266 8075-26788/com.example.mytext D/OkHttp: x-cache: Hit from cloudfront
2019-11-27 14:13:14.267 8075-26788/com.example.mytext D/OkHttp: via: 1.1 df5e8a506b27f692fa07efb955acfd9c.cloudfront.net (CloudFront)
2019-11-27 14:13:14.267 8075-26788/com.example.mytext D/OkHttp: x-amz-cf-pop: HKG62-C1
2019-11-27 14:13:14.267 8075-26788/com.example.mytext D/OkHttp: x-amz-cf-id: AZUqvSCTG2AJBPmuvaL3BDbQovNsxqwDOtKl3toZCu8Q4eif3EpW8Q==
2019-11-27 14:13:14.267 8075-26788/com.example.mytext D/OkHttp: age: 75647
2019-11-27 14:13:17.838 8075-26788/com.example.mytext D/OkHttp: <-- END HTTP (binary 21136988-byte body omitted
```

出现这种情况，我猜想是因为在某些地方，先将所有的 ResponseBody 内的数据读取完了，再给到了我们的 DownloadUtils 内。

我对 getResponseWithInterceptorChain 方法中添加的拦截器都进行 Debug 都没有看到有读取 ResponseBody 的地方。百思不得其解。最终多次尝试，才在  HttpLoggingInterceptor 找到了问题所在。

```java
  Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
  }

```

# HttpLoggingInterceptor

原因在于 HttpLoggingInterceptor 会记录请求和响应的 Body ，所以，其会将 Body 读取出来，然后显示，最终往下，或者往上传递。



```
ResponseBody responseBody = response.body();
long contentLength = responseBody.contentLength();
String bodySize = contentLength != -1 ? contentLength + "-byte" : "unknown-length";
logger.log("<-- " + response.code() + ' ' + response.message() + ' '
    + response.request().url() + " (" + tookMs + "ms" + (!logHeaders ? ", "
    + bodySize + " body" : "") + ')');

if (logHeaders) {
  Headers headers = response.headers();
  for (int i = 0, count = headers.size(); i < count; i++) {
    logger.log(headers.name(i) + ": " + headers.value(i));
  }

  if (!logBody || !HttpHeaders.hasBody(response)) {
    logger.log("<-- END HTTP");
  } else if (bodyEncoded(response.headers())) {
    logger.log("<-- END HTTP (encoded body omitted)");
  } else {
    BufferedSource source = responseBody.source();
    source.request(Long.MAX_VALUE); // Buffer the entire body.
    Buffer buffer = source.buffer();

    Charset charset = UTF8;
    MediaType contentType = responseBody.contentType();
    if (contentType != null) {
      try {
        charset = contentType.charset(UTF8);
      } catch (UnsupportedCharsetException e) {
        logger.log("");
        logger.log("Couldn't decode the response body; charset is likely malformed.");
        logger.log("<-- END HTTP");

        return response;
      }
    }

    if (!isPlaintext(buffer)) {
      logger.log("");
      logger.log("<-- END HTTP (binary " + buffer.size() + "-byte body omitted)");
      return response;
    }

    if (contentLength != 0) {
      logger.log("");
      logger.log(buffer.clone().readString(charset));
    }

    logger.log("<-- END HTTP (" + buffer.size() + "-byte body)");
  }
}
```

 

关于在于 

```java
        BufferedSource source = responseBody.source();
        source.request(Long.MAX_VALUE); // Buffer the entire body.
        Buffer buffer = source.buffer();
```

这个地方将 Body 读取了，在这进行 IO 肯定是要花时间的，所以我们的下载才会先等待，然后突然进度条就满了。

这就和我们设置的拦截器的级别有关了，我们想要查看 POST 和响应的数据，就必须设置为 Level.Body ，但我们下载的时候不能这么干，我们把级别改一下。



```java
    private static HttpLoggingInterceptor logging = new HttpLoggingInterceptor()
            .setLevel(HttpLoggingInterceptor.Level.HEADERS);

```



# InterruptedIOException

在我们上面的代码中，如果在下载的过程中，在 writeToFile 方法中，安卓退出 Activity 退出了，或者是其他的原因导致了下游的生命周期到了，进行了 dispose() 操作，就会出现 InterruptedIOException。

在 [RxJava 的官方文档中说明](https://github.com/ReactiveX/RxJava/wiki/What's-different-in-2.0#error-handling)



>One important design requirement for 2.x is that no `Throwable` errors should be swallowed. This means errors that can't be emitted because the downstream's lifecycle already reached its terminal state or the downstream cancelled a sequence which was about to emit an error.
>
>2.x 中一个非常重要的设计就是，任何 Throwable 都不应该被抛弃。这就意味着，如果下游的生命周期到了终止状态，或者下游取消了一个能发射错误的序列，那么，错误就不能被发射出去。

> Such errors are routed to the `RxJavaPlugins.onError` handler. This handler can be overridden with the method `RxJavaPlugins.setErrorHandler(Consumer<Throwable>)`. Without a specific handler, RxJava defaults to printing the `Throwable`'s stacktrace to the console and calls the current thread's uncaught exception handler.
>
> 这些错误就会被路由到 RxJavaPlugins.onError 控制器。这个 Handler 可以被 `RxJavaPlugins.setErrorHandler(Consumer<Throwable>)` 方法重写。如果没有指定一个 Handler ，RxJava 默认会将 Throwable 的堆栈信息给打印到控制台，然后调用当前线程的 uncaught exception handler

> On desktop Java, this latter handler does nothing on an `ExecutorService` backed `Scheduler` and the application can keep running. However, Android is more strict and terminates the application in such uncaught exception cases.
>
> 在桌面 Java 上，uncaught exception handler 不会做任何事情。然而，安卓上的限制比较严格，其会让应用中止。

> If this behavior is desirable can be debated, but in any case, if you want to avoid such calls to the uncaught exception handler, the **final application** that uses RxJava 2 (directly or transitively) should set a no-op handler:
>
> 如果不想出现这种行为，那么最终使用 RxJava 的应用，生设置一个无操作的 Handler

```java
// If Java 8 lambdas are supported
RxJavaPlugins.setErrorHandler(e -> { });

// If no Retrolambda or Jack 
RxJavaPlugins.setErrorHandler(Functions.<Throwable>emptyConsumer());
```

> 不建议在中间库内改变 error handler。

> 不幸的是，RxJava 不能告知这种超出生命周期，不能发射的 exceptions 应不应该让我们的 APP 崩溃。排查这种原因很烦人，特别是很底层的 source 路由到 RxJavaPlugins.onError。

其也给出了例子的解决方法：

```java
RxJavaPlugins.setErrorHandler(e -> {
    if (e instanceof UndeliverableException) {
        e = e.getCause();
    }
    if ((e instanceof IOException) || (e instanceof SocketException)) {
        // fine, irrelevant network problem or API that throws on cancellation
        return;
    }
    if (e instanceof InterruptedException) {
        // fine, some blocking code was interrupted by a dispose call
        return;
    }
    if ((e instanceof NullPointerException) || (e instanceof IllegalArgumentException)) {
        // that's likely a bug in the application
        Thread.currentThread().getUncaughtExceptionHandler()
            .handleException(Thread.currentThread(), e);
        return;
    }
    if (e instanceof IllegalStateException) {
        // that's a bug in RxJava or in a custom operator
        Thread.currentThread().getUncaughtExceptionHandler()
            .handleException(Thread.currentThread(), e);
        return;
    }
    Log.warning("Undeliverable exception received, not sure what to do", e);
});
```


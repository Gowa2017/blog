---
title: Retrofit对象的建立及与RxJava的使用
categories:
  - Android
date: 2019-04-07 23:36:17
updated: 2019-04-07 23:36:17
tags: 
  - Android
  - Java
  - Retrofit
---

对于 Retrofit 的使用，我们一般会利用 Retrofit.Builder 来填写参数后，建立一个 Retrofit 对象，接着就以此对象来根据我们编写的接口生成 所以的 Call，当配合 RxJava 使用的时候，我很好奇这其中的过程是怎么样的。

<!--more-->

# 前言

在开始用这个东西之前，我们有必要了解一下， Retrofit 到底是什么。根据官方文档上的介绍：[Retrofit官方网站](https://square.github.io/retrofit/)

1. 使用 Retrofit ，我们可以将 HTTP 接口调用，转换为对一个 Java 接口的调用。
2. Retrofit 类会自动生成我们定义的 Java 接口的实现。
3. 调用这个生成的接口实现，就会对远程服务器进行调用。

Retrofit 所做的事情主要是：

1. 生成我们定义的接口实现。
2. 调用底层的 OkHttp 来进行远程调用。
3. 管理往返数据。


# Retrofit 对象的建立

一般来说，我们会这样来建立一个 Retrofit 对象：

```java
        Retrofit retrofit = new Retrofit.Builder()
                .baseUrl("https://api.github.com/")
                .addConverterFactory(GsonConverterFactory.create())
                .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
                .build();

```

接着，让 Retrofit 实例，根据接口来生成一个请求对象：

```java
        GitHubService gitHubService = retrofit.create(GitHubService.class);
```

在这里， GitHubService.class 是我们编写的接口。

每个 retrofit 实例都有 ConverterFactory 与 CallAdapterFactory 列表，其存的都是工厂类，当我们需要 Converter 或 CallAdapter 的时候，调用工厂类的 create() 方法来建立对象。

```java
    private final List<Converter.Factory> converterFactories = new ArrayList<>();
    private final List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>();
```

对于*callFactory* 我们一般都用默认的。  

```java
    private @Nullable okhttp3.Call.Factory callFactory;
```

# retrofit.create()

根据接口，生成接口的实现。

默认情况下， `Retrofit.create()` 方法会返回一个代表 HTTP 请求的  *Call* 对象。 *Call* 的泛型参数就表示表示了响应内容的类型，会被 *Convert.Factory* 实例中的某一个进行转换。

```java
    public <T> T create(final Class<T> service) {
        // 验证我们的传递的 service。 1. 是一个接口； 2. 接口不能继承其他接口。
        Utils.validateServiceInterface(service);
        // 是否立即加载接口定义的所有方法[到 Retrofit 缓存]。
        if (this.validateEagerly) {
            this.eagerlyValidateMethods(service);
        }

        // 建立一个代理对象。调用此对象的方法会调用到代理类的对应方法。
        return Proxy.newProxyInstance(service.getClassLoader(), new Class[]{service}, new InvocationHandler() {
            private final Platform platform = Platform.get();
            private final Object[] emptyArgs = new Object[0];

            public Object invoke(Object proxy, Method method, @Nullable Object[] args) throws Throwable {
                if (method.getDeclaringClass() == Object.class) {
                    return method.invoke(this, args);
                } else {
                    return this.platform.isDefaultMethod(method) ? this.platform.invokeDefaultMethod(method, service, proxy, args) : Retrofit.this.loadServiceMethod(method).invoke(args != null ? args : this.emptyArgs);
                }
            }
        });
    }
```

在 invoke() 方法中，会检查我们传递的 service 是否是一个对象（我们传递的接口，当然不会），或者是一个默认方法（默认方法指的是：public non-abstarct 的实例方法，我们这里也不是）。

所以，当我们以 `        Observable<List<Repo>> service = gitHubService.listRepos("octocat");` 的形式调用 retrofit 对象的时候，实则调用的是:

```java
loadServiceMethod(method).invoke(args != null ? args : emptyArgs);
```

# ServiceMethod

对于接口中的每个方法调用，都是建立在 ServiceMethod 的调用上的。对于接口中定义的每个方法，都会用 `loadServiceMethod()` 来生成方法的实现。

## Retrofit.loadServiceMethod()

```java
  ServiceMethod<?> loadServiceMethod(Method method) {
    ServiceMethod<?> result = serviceMethodCache.get(method);
    if (result != null) return result;

    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        result = ServiceMethod.parseAnnotations(this, method);
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }
```

这首先会从 *serviceMethodCache* 缓存中获取一下看看有没有已经建立好的，如果没有的话，就会根据注解来建立一个新的 ServiceMethod。

## ServiceMethod.parseAnnotations()

因为我们会在方法上，参数上添加各种注解，如：`@GET,@POST,@QUERY,@BODY` 等，所以就要根据这些注解，来分析用哪个 RequestFactory 来处理。

也就说，根据我们的注解，来决定要建立何种类型的请求。

所以此方法做了两个工作：

1. 根据注解确定我们要用的 RequestFactory 是什么。
2. 方法的返回类型不能是不可识别的或空类型。

```java
  static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
    RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);

    Type returnType = method.getGenericReturnType();
    if (Utils.hasUnresolvableType(returnType)) {
      throw methodError(method,
          "Method return type must not include a type variable or wildcard: %s", returnType);
    }
    if (returnType == void.class) {
      throw methodError(method, "Service methods cannot return void.");
    }

    return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
  }
```


## HttpServiceMethod.parseAnnotations()

HttpServiceMethod 是 ServiceMethod 的继承与实现。其会根据我们传递的 retrofit, method, RequestFactor 来建立一个 HttpServiceMethod。

HtppServiceMethod 是一个非常核心的东西，其实现了很多重要的逻辑。

1. 找到 CallAdapter。
2. 找到 responseConverter
3. 使用默认的 callFactory。
4. 最终返回一个 HttpServiceMethod。

```java
  static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
      Retrofit retrofit, Method method, RequestFactory requestFactory) {
    CallAdapter<ResponseT, ReturnT> callAdapter = createCallAdapter(retrofit, method);
    Type responseType = callAdapter.responseType();
    if (responseType == Response.class || responseType == okhttp3.Response.class) {
      throw methodError(method, "'"
          + Utils.getRawType(responseType).getName()
          + "' is not a valid response body type. Did you mean ResponseBody?");
    }
    if (requestFactory.httpMethod.equals("HEAD") && !Void.class.equals(responseType)) {
      throw methodError(method, "HEAD method must use Void as response type.");
    }

    Converter<ResponseBody, ResponseT> responseConverter =
        createResponseConverter(retrofit, method, responseType);

    okhttp3.Call.Factory callFactory = retrofit.callFactory;
    return new HttpServiceMethod<>(requestFactory, callFactory, callAdapter, responseConverter);
  }
```

### HttpServiceMethod.createCallAdapter()

在 HttpServiceMethod 中，会根据 retrofit 对象和 接口方法来查找 CallAdapter。

```java
  private static <ResponseT, ReturnT> CallAdapter<ResponseT, ReturnT> createCallAdapter(
      Retrofit retrofit, Method method) {
    Type returnType = method.getGenericReturnType();
    Annotation[] annotations = method.getAnnotations();
    try {
      //noinspection unchecked
      return (CallAdapter<ResponseT, ReturnT>) retrofit.callAdapter(returnType, annotations);
    } catch (RuntimeException e) { // Wide exception range because factories are user code.
      throw methodError(method, e, "Unable to create call adapter for %s", returnType);
    }
  }
```

具体的逻辑就是在 Retrofit 实里的 CallAdapterFactory 中根据 method 返回值的泛型类型及 注解来查找。

```java
  public CallAdapter<?, ?> callAdapter(Type returnType, Annotation[] annotations) {
    return nextCallAdapter(null, returnType, annotations);
  }
  
  public CallAdapter<?, ?> nextCallAdapter(@Nullable CallAdapter.Factory skipPast, Type returnType,
      Annotation[] annotations) {
    checkNotNull(returnType, "returnType == null");
    checkNotNull(annotations, "annotations == null");

    int start = callAdapterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = callAdapterFactories.size(); i < count; i++) {
      CallAdapter<?, ?> adapter = callAdapterFactories.get(i).get(returnType, annotations, this);
      if (adapter != null) {
        return adapter;
      }
```

在上述遍历 CallAdapterFactory 列表的过程中，当然会遍历到我们添加的 RxJava2CallAdapterFactory。  

对于 RxJava2CallAdapterFactory 的 get 方法，我们来看一下具体的逻辑：

```java
    @Nullable
    public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
        Class<?> rawType = getRawType(returnType);
        if (rawType == Completable.class) {
            return new RxJava2CallAdapter(Void.class, this.scheduler, this.isAsync, false, true, false, false, false, true);
        } else {
            boolean isFlowable = rawType == Flowable.class;
            boolean isSingle = rawType == Single.class;
            boolean isMaybe = rawType == Maybe.class;
            
            // 返回类型的泛型 raw 类型必须是 RxJava 支持的类型
            if (rawType != Observable.class && !isFlowable && !isSingle && !isMaybe) {
                return null;
            } else {
                boolean isResult = false;
                boolean isBody = false;
                // 返回的类型，必须是泛型类型（必须泛型化）
                if (!(returnType instanceof ParameterizedType)) {
                    String name = isFlowable ? "Flowable" : (isSingle ? "Single" : (isMaybe ? "Maybe" : "Observable"));
                    throw new IllegalStateException(name + " return type must be parameterized as " + name + "<Foo> or " + name + "<? extends Foo>");
                } else {
                    // 因为RxJava 这些泛型容器的都只有一个泛型参数，所以说需要拿到泛型参数的上界类型。如  getParameterUpperBound(1, Map<String, ? extends Runnable> ) 会返回 Runnable。
                    Type observableType = getParameterUpperBound(0, (ParameterizedType)returnType);
                    // 拿到上界类型还不够，对于是一个泛型类型，还需要拿到声明此泛型的对象类型。如 List<String> 会返回 List。
                    Class<?> rawObservableType = getRawType(observableType);
                    Type responseType;
                    // 这里表明，我们的返回类型不能是 Single<Response>，而必须是泛型化的，如 Single<Response<String>>。
                    if (rawObservableType == Response.class) {
                        if (!(observableType instanceof ParameterizedType)) {
                            throw new IllegalStateException("Response must be parameterized as Response<Foo> or Response<? extends Foo>");
                        }

                        responseType = getParameterUpperBound(0, (ParameterizedType)observableType);
                    } else if (rawObservableType == Result.class) {
                        if (!(observableType instanceof ParameterizedType)) {
                            throw new IllegalStateException("Result must be parameterized as Result<Foo> or Result<? extends Foo>");
                        }

                        responseType = getParameterUpperBound(0, (ParameterizedType)observableType);
                        isResult = true;
                    // 通常情况下，我们都没有使用 Result, Response ，所以都用到下面这个类型。如 Single<DictInfo>。isBody 说明，我们的返回类型，就直接是响应回来的数据了。
                    } else {
                        responseType = observableType;
                        isBody = true;
                    }

                    return new RxJava2CallAdapter(responseType, this.scheduler, this.isAsync, isResult, isBody, isFlowable, isSingle, isMaybe, false);
                }
            }
        }
    }
```

其运作的逻辑，就是看我们的返回类型是不是 RxJava 支持的那些类型。

1. 检查返回值是否是 Completable。
2. 检查是否是泛型返回类型
3. 获取返回泛型的第一个泛型参数类型。
    1. 泛型参数的如果是 Response.class 其必需泛型化，如 Observable<Response<User>>。返回类型，responseType = User。
    2. 泛型参数如果是 Result.class，其也必须泛型化。如 Observable<Result<User>>。回类型，responseType = User。isResult = true。
    3. 否则，返回的类型就如同 Observable<User>。responseType = User。 isBody = true。
    4. 这里确定的了响应内容需要进行如何转换的问题。后面会用到。

# HttpServiceMethod.invoke

当我们调用 代理对象的 方法的时候，实际上调用到的是我们通过接口来建立的 HttpServiceMethod 的 invoke 方法，其结果，是会返回一个经过 callAdapter 适配过的 对象。

```java
  @Override ReturnT invoke(Object[] args) {
    return callAdapter.adapt(
        new OkHttpCall<>(requestFactory, args, callFactory, responseConverter));
  }
```


而如果我们的是 RxJava 支持的数据类型的话，那么就肯定会用 RxJava2CallAdapter 来进行适配。

在这里终于看到了我们将会用 OkHttp 打交道的地方。 `OkHttpCall`。

在这里不禁要理解一下，，callAdapter.adapt() 到底什么用？其实不要想太多，call 适配器，实际就是是对我们要返回的 OkHttpCall 进行一些适配，设置，将其转换一个 Observable。


 CallAdapter<R, Object>.adapt 进行了从 OkHttpCall 到 Observable 的转换。

```java
  @Override public Object adapt(Call<R> call) {
  // 根据同步还是异步 call 来建立 Observable。
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

1. 根据 OkHttpCall 及同步或者异步来建立一个最底层的 responseObservable。
2. 其根据是否是 Result 类型 或 Body 类型来建立对应的 ResultObservable， BodyObservable。这个建立需要注意的时，其 upstream 也是一个 Observable（responseObservable）。

- **responseObservable** 用来观察 HTTP 请求的响应
- **BodyObservable** 用来观察响应的内容。


当我们以 Single<User> 的形式来定义方法的返回类型的时候，建立的对象类似这样：

```java
Observable<Response<R>>  responseObservable = new CallEnqueueObservable<>(call);
Observable<?> observable = new BodyObservable<>(responseObservable);
Single<User> single = observable.singleOrError();
```

在最后最后我们调用接口的时候，定义了 订阅者，那么这订阅这个动作就会一直往上传递。

当调用  observable.subscribe() 的时候代码执行链：

1. Single<User> 订阅到我们定义的 observer。
2. BodyObservable 订阅到 Single 订阅到 Single<User>  内部的 SingleElementObserver
3. CallEnqueueObservable 订阅到 BodyObservable 内部的 BodyObserver
4. CallEnqueueObservable 驱使 OkHttpCall 执行请求。



```java
    // BodyObservable
  @Override protected void subscribeActual(Observer<? super T> observer) {
    upstream.subscribe(new BodyObserver<T>(observer));
  }
```

```java
    // CallEnqueueObservable
  @Override protected void subscribeActual(Observer<? super Response<T>> observer) {
    // Since Call is a one-shot type, clone it for each new observer.
    Call<T> call = originalCall.clone();
    CallCallback<T> callback = new CallCallback<>(call, observer);
    observer.onSubscribe(callback);
    if (!callback.isDisposed()) {
      call.enqueue(callback);
    }
  }
```

在 CallEnqueueObservable 中进行了网络请求的入队，同时其将一个回调 CallCallback 给到  OkHttpCall 作为回调方法，CallEnqueueObservable 私有的类 CallCallback 会持有 BodyObserver 。

当请求完成后，就会调用 `CallCallback<>(call, observer)`的 `onResponse()` 方法。


```java
    //CallEnqueueObservable 将响应原封不动的传给 BodyObserver
    @Override public void onResponse(Call<T> call, Response<T> response) {
      if (disposed) return;

      try {
        observer.onNext(response);

        if (!disposed) {
          terminated = true;
          observer.onComplete();
        }
      } catch (Throwable t) {
        if (terminated) {
          RxJavaPlugins.onError(t);
        } else if (!disposed) {
          try {
            observer.onError(t);
          } catch (Throwable inner) {
            Exceptions.throwIfFatal(inner);
            RxJavaPlugins.onError(new CompositeException(t, inner));
          }
        }
      }
    }
```

```java

    //BodyObserver 会将返回的 Body 传递给 下游 Single
    
    @Override public void onNext(Response<R> response) {
      if (response.isSuccessful()) {
        observer.onNext(response.body());
      } else {
        terminated = true;
        Throwable t = new HttpException(response);
        try {
          observer.onError(t);
        } catch (Throwable inner) {
          Exceptions.throwIfFatal(inner);
          RxJavaPlugins.onError(new CompositeException(t, inner));
        }
      }
    }
```

# Converter

我们使用的是 OkHttpCall.java 这个类来进行最终的调用。在其中进项了从 Http 响应到 Java 对象的转换。

CallEnqueueObservable 中会驱使网络请求进行队列，同时将 CallCallback 作为回调。


```java
    call.enqueue(new okhttp3.Callback() {
      @Override public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse) {
        Response<T> response;
        try {
          response = parseResponse(rawResponse);
        } catch (Throwable e) {
          throwIfFatal(e);
          callFailure(e);
          return;
        }

        try {
          callback.onResponse(OkHttpCall.this, response);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }
```

```java
  Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
    ResponseBody rawBody = rawResponse.body();

    // Remove the body's source (the only stateful object) so we can pass the response along.
    rawResponse = rawResponse.newBuilder()
        .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
        .build();

    int code = rawResponse.code();
    if (code < 200 || code >= 300) {
      try {
        // Buffer the entire body to avoid future I/O.
        ResponseBody bufferedBody = Utils.buffer(rawBody);
        return Response.error(bufferedBody, rawResponse);
      } finally {
        rawBody.close();
      }
    }

    if (code == 204 || code == 205) {
      rawBody.close();
      return Response.success(null, rawResponse);
    }

    ExceptionCatchingResponseBody catchingBody = new ExceptionCatchingResponseBody(rawBody);
    try {
      T body = responseConverter.convert(catchingBody);
      return Response.success(body, rawResponse);
    } catch (RuntimeException e) {
      // If the underlying source threw an exception, propagate that rather than indicating it was
      // a runtime exception.
      catchingBody.throwIfCaught();
      throw e;
    }
  }
```

关于在于 `T body = responseConverter.convert(catchingBody);`

# 总结

1. Retrofit 根据接口方法来生成实现的 HttpServiceMethod 方法。HttpServiceMethod 是 *requestFactory（建立ServiceMethod 的时候根据注解生成）, callFactory（指定 client 的时候，由 client 处获得）,  callAdapter（定义）, responseConverter（定义）* 的组合。
2. 调用接口方法时会调用  HttpServiceMethod.invoke()。此时会利用 上面提到的四个元素来，建立一个 OkHttpCall。
3. RxJava2CallAdapter 会对 OkHttpCall 进行适配。
  1. 建立 CallEnqueueObservable。驱使 OkHttpCall 请求入列，同时提供回调 CallEnqueueObservable.CallCallback 给 OkHttpCall。
  2. OkHttpCall 使用 okhttp3.Call 来进行网络请求。然后根据 responseConverter 来进行响应内容转换到 Java 对象。
  3. 将转换后的 Body 传递给 BodyObserver。
  4. 将 Body 传递到 SingleObserver。
  5. 到底我们的终止处。

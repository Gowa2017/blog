---
title: 使用Retrofit-Rxjava来上传下载文件
categories:
  - [RxJava]
date: 2019-07-31 11:43:28
updated: 2019-07-31 11:43:28
tags: 
  - RxJava
  - Java
  - Android
---

使用 Http 协议进行文件传输的时候，需要了解一些必要的知识，然后才能配合使用 Retrofit 和 RxJava 来进行操作。

<!--more-->

# HTTP

首先，HTTP 是一个超文本传输协议，以就是说，这个协议，是用来交互文本的，而不是二进制流的。我们必须把我们的文件转换成文本形式，才能进行传输。

RFC7230 定义了此协议。

## 术语
### URI

URI （uniform resource identifier） 是统一资源标识符的意思。可以用来唯一的标识一个资源。

一般由三部分组成：

URI一般由三部组成：

- ①访问资源的命名机制
- ②存放资源的主机名
- ③资源自身的名称，由路径表示，着重强调于资源。
### URL

URL 是（Uniform Resource locator） 是统一资源定位器的意思，其也是一种 URI 的实现，其不仅指定了资源的标识，还指明了如何定位资源。

一般也是由三部分组成：

- ①协议(或称为服务方式)
- ②存有该资源的主机IP地址(有时也包括端口号)
- ③主机资源的具体地址。如目录和文件名等


### URN

URN，uniform resource name，统一资源命名，是通过名字来标识资源，比如mailto:java-net@java.sun.com。

## 消息格式


```
 start-line(method SP request-target SP HTTP-version CRLF/HTTP-version SP status-code SP reason-phrase CRLF)
 *( header-field CRLF )
 CRLF
 [ message-body ](message-body = *OCTET)
```

什么时候允许出现 message-body 在请求和响应消息中是不同的。

在请求中，如果 Header 有一个 Content-Length 或 Transfer-Encoding  字段，就表示会有一个 Body。请求消息的结构和方法的语义是相互独立的，因此及时方法没有定义会使用  message-body 其也可能会出现。

响应消息中是否会出现 message-body 与 其对应的请求和响应的状态码有关。

## multipart/form-data

RFC 7578 定义了 multipart/form-data 媒体类型。对于  message-body 中是一个 multipart/form-data 的数据时，其是由很多分隔符分开的部分组成的。

- Boundary 类似 "CRLF -- Boundary"
- Content-Disposition 每个部分都会有这个字段，其类型是 form-data，同时包含一个额外的参数 "name"，表示从表单中初始的字段名。` Content-Disposition: form-data; name="user"` 对于文件的话，可以用 filename 来替代 name，但有的时候这个参数没有意义，可能不会识别它，看服务器来决定。
- Content-Type 可能会有，默认是 text/plain。如果发送的是文件的时候，那么默认的会是 application/octet-stream。

例子：

```
       --AaB03x
       content-disposition: form-data; name="field1"
       content-type: text/plain;charset=UTF-8
       content-transfer-encoding: quoted-printable

       Joe owes =E2=82=AC100.
       --AaB03x
```

# Retrofit 实现上传

2.0 需要用 OkHttp 的 RequestBody 或者 MultipartBody.Part 来封装我们的文件进行传输。

- RequestBody 包含数据流，contentType, contentLength
- MultipartBody.Part 对 RequestBody 进行封装。事实上就是加上 `content-disposition: form-data; name="field1"; filename="field"`


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

    public static Part createFormData(String name, @Nullable String filename, RequestBody body) {
      if (name == null) {
        throw new NullPointerException("name == null");
      }
      StringBuilder disposition = new StringBuilder("form-data; name=");
      appendQuotedString(disposition, name);

      if (filename != null) {
        disposition.append("; filename=");
        appendQuotedString(disposition, filename);
      }

      return create(Headers.of("Content-Disposition", disposition.toString()), body);
    }

    public static Part createFormData(String name, @Nullable String filename, RequestBody body) {
      if (name == null) {
        throw new NullPointerException("name == null");
      }
      StringBuilder disposition = new StringBuilder("form-data; name=");
      appendQuotedString(disposition, name);

      if (filename != null) {
        disposition.append("; filename=");
        appendQuotedString(disposition, filename);
      }

      return create(Headers.of("Content-Disposition", disposition.toString()), body);
    }

```

可以看到  formdata 实际上就是将多个 RequestBody 封装成 Part 然后进行传输了。

一般来说，2.0会以下面的形式进行上传文件：

```java
public interface FileUploadService {  
    @Multipart
    @POST("upload")
    Call<ResponseBody> upload(
        @Part("description") RequestBody description,
        @Part MultipartBody.Part file
    );
}

## 服务端代码

​```js
method: 'POST',  
path: '/upload',  
config: {  
    payload: {
        maxBytes: 209715200,
        output: 'stream',
        parse: false
    },
    handler: function(request, reply) {
        var multiparty = require('multiparty');
        var form = new multiparty.Form();
        form.parse(request.payload, function(err, fields, files) {
            console.log(err);
            console.log(fields);
            console.log(files);

            return reply(util.inspect({fields: fields, files: files}));
        });
    }
}
```


其将会打印日志：

```
null  
{ description: [ 'hello, this is description speaking' ] }
{ picture:
   [ { fieldName: 'picture',
       originalFilename: '20160312_095248.jpg',
       path: '/var/folders/rq/q_m4_21j3lqf1lw48fqttx_80000gn/T/X_sxX6LDUMBcuUcUGDMBKc2T.jpg',
       headers: [Object],
       size: 39369 } ] }
```
## 完整代码

```java
private void uploadFile(Uri fileUri) {  
    // create upload service client
    FileUploadService service =
            ServiceGenerator.createService(FileUploadService.class);

    // https://github.com/iPaulPro/aFileChooser/blob/master/aFileChooser/src/com/ipaulpro/afilechooser/utils/FileUtils.java
    // use the FileUtils to get the actual file by uri
    File file = FileUtils.getFile(this, fileUri);

    // create RequestBody instance from file
    RequestBody requestFile =
            RequestBody.create(
                         MediaType.parse(getContentResolver().getType(fileUri)),
                         file
             );

    // MultipartBody.Part is used to send also the actual file name
    MultipartBody.Part body =
            MultipartBody.Part.createFormData("picture", file.getName(), requestFile);

    // add another part within the multipart request
    String descriptionString = "hello, this is description speaking";
    RequestBody description =
            RequestBody.create(
                    okhttp3.MultipartBody.FORM, descriptionString);

    // finally, execute the request
    Call<ResponseBody> call = service.upload(description, body);
    call.enqueue(new Callback<ResponseBody>() {
        @Override
        public void onResponse(Call<ResponseBody> call,
                               Response<ResponseBody> response) {
            Log.v("Upload", "success");
        }

        @Override
        public void onFailure(Call<ResponseBody> call, Throwable t) {
            Log.e("Upload error:", t.getMessage());
        }
    });
}
```


# OkHttp 的 Source与Sink

从 RequestBody 的建立代码中，我们看到，实际上是将我们的 File 构造了一个 Source ， 然后写到了一个 Sink 中。

```java
// Okio
    public static Source source(File file) throws FileNotFoundException {
        if (file == null) {
            throw new IllegalArgumentException("file == null");
        } else {
            return source((InputStream)(new FileInputStream(file)));
        }
    }

    public static Sink sink(File file) throws FileNotFoundException {
        if (file == null) {
            throw new IllegalArgumentException("file == null");
        } else {
            return sink((OutputStream)(new FileOutputStream(file)));
        }
    }

```

我们可以简单的将 Source, Sink 看成是输入流或者输入流。虽然其提供了一些很有用很方便有效率的方法，但这不影响我们的讨论。

我们来看一下 OkHttp 是怎么样发送请求的，在 CallServerInterceptor 中，最终发生网络请求，发送数据：

```java
  @Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    HttpCodec httpCodec = realChain.httpStream();
    StreamAllocation streamAllocation = realChain.streamAllocation();
    RealConnection connection = (RealConnection) realChain.connection();
    Request request = realChain.request();

    long sentRequestMillis = System.currentTimeMillis();

    realChain.eventListener().requestHeadersStart(realChain.call());
    httpCodec.writeRequestHeaders(request);
    realChain.eventListener().requestHeadersEnd(realChain.call(), request);

    Response.Builder responseBuilder = null;
    // 如果 Post 方法，且含有一个 Body
    if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
      // If there's a "Expect: 100-continue" header on the request, wait for a "HTTP/1.1 100
      // Continue" response before transmitting the request body. If we don't get that, return
      // what we did get (such as a 4xx response) without ever transmitting the request body.
      if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
        httpCodec.flushRequest();
        realChain.eventListener().responseHeadersStart(realChain.call());
        responseBuilder = httpCodec.readResponseHeaders(true);
      }

      if (responseBuilder == null) {
        // Write the request body if the "Expect: 100-continue" expectation was met.
        realChain.eventListener().requestBodyStart(realChain.call());
        long contentLength = request.body().contentLength();
        CountingSink requestBodyOut =
            new CountingSink(httpCodec.createRequestBody(request, contentLength));
        BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);

        request.body().writeTo(bufferedRequestBody);
        bufferedRequestBody.close();
        realChain.eventListener()
            .requestBodyEnd(realChain.call(), requestBodyOut.successfulCount);
      } else if (!connection.isMultiplexed()) {
        // If the "Expect: 100-continue" expectation wasn't met, prevent the HTTP/1 connection
        // from being reused. Otherwise we're still obligated to transmit the request body to
        // leave the connection in a consistent state.
        streamAllocation.noNewStreams();
      }
    }

    httpCodec.finishRequest();

    if (responseBuilder == null) {
      realChain.eventListener().responseHeadersStart(realChain.call());
      responseBuilder = httpCodec.readResponseHeaders(false);
    }

    Response response = responseBuilder
        .request(request)
        .handshake(streamAllocation.connection().handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();

    int code = response.code();
    if (code == 100) {
      // server sent a 100-continue even though we did not request one.
      // try again to read the actual response
      responseBuilder = httpCodec.readResponseHeaders(false);

      response = responseBuilder
              .request(request)
              .handshake(streamAllocation.connection().handshake())
              .sentRequestAtMillis(sentRequestMillis)
              .receivedResponseAtMillis(System.currentTimeMillis())
              .build();

      code = response.code();
    }

    realChain.eventListener()
            .responseHeadersEnd(realChain.call(), response);

    if (forWebSocket && code == 101) {
      // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
      response = response.newBuilder()
          .body(Util.EMPTY_RESPONSE)
          .build();
    } else {
      response = response.newBuilder()
          .body(httpCodec.openResponseBody(response))
          .build();
    }

    if ("close".equalsIgnoreCase(response.request().header("Connection"))
        || "close".equalsIgnoreCase(response.header("Connection"))) {
      streamAllocation.noNewStreams();
    }

    if ((code == 204 || code == 205) && response.body().contentLength() > 0) {
      throw new ProtocolException(
          "HTTP " + code + " had non-zero Content-Length: " + response.body().contentLength());
    }

    return response;
  }

```


关于代码在于：

```java
long contentLength = request.body().contentLength();
        CountingSink requestBodyOut =
            new CountingSink(httpCodec.createRequestBody(request, contentLength));
        BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);

        request.body().writeTo(bufferedRequestBody);
        bufferedRequestBody.close();

```

此段代码将我们的 RequestBody 写到一个与连接相关联的 Sink 上，发送数据。

```java
// RealConnection.java
      sink = Okio.buffer(Okio.sink(rawSocket));

```
这个 sink 会在 Http1Code 构建 CountingSink 的时候用到，实际上数据就是写到这里面的。

对于构建的 BufferedSink，其是以 8192 字节每次写入套接字：

```java
// RealBufferedSink.java
    public long writeAll(Source source) throws IOException {
        if (source == null) {
            throw new IllegalArgumentException("source == null");
        } else {
            long totalBytesRead = 0L;

            long readCount;
            while((readCount = source.read(this.buffer, 8192L)) != -1L) {
                totalBytesRead += readCount;
                this.emitCompleteSegments();
            }

            return totalBytesRead;
        }
    }

```


# 对于上传文件进度回调的思路

看了一下谷歌，很多都是在我们将 RequestBody 读出并写到 sink 的时候进行回调：

比如这个地方：[okhttp recipes](https://github.com/square/okhttp/blob/master/samples/guide/src/main/java/okhttp3/recipes/Progress.java) 有类似的思路

上面这个是针对下载的。

而这个就是针对上传来显示进度的：[OkHttp 上传显示进度](https://medium.com/@PaulinaSadowska/display-progress-of-multipart-request-with-retrofit-and-rxjava-23a4a779e6ba)

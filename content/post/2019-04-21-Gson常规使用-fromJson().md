---
title: Gson常规使用-fromJson()
categories:
  - [Java]
date: 2019-04-21 21:26:33
updated: 2019-04-21 21:26:33
tags: 
  - Android
  - Java
  - Gson
---
Gson 是一个非常棒的用来解析 Json 对象到 Java 对象的库，或者将 Java 对象转换成为 Json 的库。不过有些细节还值得探讨一下。

<!--more-->

# TypeToken

**TypeToken** 代表了一个泛型。对于 Java 而言，因为泛型擦除的原因，在运行时是无法知道泛型信息的，但是 **TypeToken** 可以。

例如我们可以这样来获取 `List<String>` 的泛型信息：

```java
TypeToken<List<String>> list = new TypeToken<List<String>>(){};
```
在这里要注意的是，因为 TypeToken 构造函数是 `protected` 的，所以只能用匿名内部类的形式进行构造。


但对于类似  `Class <?>` 这样有通配符的泛型类是不行的。

## TypeToken()

```java
  protected TypeToken() {
    this.type = getSuperclassTypeParameter(getClass());
    this.rawType = (Class<? super T>) $Gson$Types.getRawType(type);
    this.hashCode = type.hashCode();
  }
```

在这里我们可以看到， RawType 表示的是接受了泛型参数而形成的泛型类。 Type 则是衍生后的那个类。例如，对于 List<String>。

- Type = List<String>
- RawType = List


# fromJson

通常，我们的做法是将 JSON 字符串转换为 Java 对象，其需要一个 Type 作为参数的。


## 根据 Class 来进行转换

实际上，也是强制将 Class 转换为一个 Type 后调用。
```java
  public <T> T fromJson(String json, Class<T> classOfT) throws JsonSyntaxException {
    Object object = fromJson(json, (Type) classOfT);
    return Primitives.wrap(classOfT).cast(object);
  }
```

## 根据 Type 转换

构造一个 StringReader 来进行转换。

对于 StringReader 可以调用  `read()` 读取一个字符；或者调用  `read(char cbuf[], int off, int len)` 来从 *off* 便宜处读取 *len* 个字符到 *cbuff* 中。

```java
  public <T> T fromJson(String json, Type typeOfT) throws JsonSyntaxException {
    if (json == null) {
      return null;
    }
    StringReader reader = new StringReader(json);
    T target = (T) fromJson(reader, typeOfT);
    return target;
  }
```

## 构造 JsonReader 

更进一步，以 StringReader 为基础，构造一个 JsonReader。 JsonReader 会从 StrinReader 中读取 Json 元素。
```java
  public <T> T fromJson(Reader json, Type typeOfT) throws JsonIOException, JsonSyntaxException {
    JsonReader jsonReader = newJsonReader(json);
    T object = (T) fromJson(jsonReader, typeOfT);
    assertFullConsumption(object, jsonReader);
    return object;
  }
```

## 最终转换


```java
  public <T> T fromJson(JsonReader reader, Type typeOfT) throws JsonIOException, JsonSyntaxException {
    boolean isEmpty = true;
    boolean oldLenient = reader.isLenient();
    reader.setLenient(true);
    try {
      reader.peek();
      isEmpty = false;
      TypeToken<T> typeToken = (TypeToken<T>) TypeToken.get(typeOfT);
      TypeAdapter<T> typeAdapter = getAdapter(typeToken);
      T object = typeAdapter.read(reader);
      return object;
    } catch (EOFException e) {
      /*
       * For compatibility with JSON 1.5 and earlier, we return null for empty
       * documents instead of throwing.
       */
      if (isEmpty) {
        return null;
      }
      throw new JsonSyntaxException(e);
    } catch (IllegalStateException e) {
      throw new JsonSyntaxException(e);
    } catch (IOException e) {
      // TODO(inder): Figure out whether it is indeed right to rethrow this as JsonSyntaxException
      throw new JsonSyntaxException(e);
    } finally {
      reader.setLenient(oldLenient);
    }
  }
```

# Map 的解析

在我们安卓进行解析的时候，有一个比较蛋疼的事情就是，我们的解析器无法解析属于 Map 类型的返回值。

因此我准备用 Gson 来解决这个问题，用 Gson 就很简单了。

```java
new Gson().fromJson(strBody, new TypeToken<Map<String,List<SrvRes>>>(){}.getType());
```

这样也就 OK  了。

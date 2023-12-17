---
title: Java中使用JsonHelp来简化操作
categories:
  - Android
date: 2018-05-06 15:35:32
updated: 2018-05-06 15:35:32
tags: 
  - Android
  - Java
---

Json为什么使用这个广我是不知道的，不过现在阶段公司在开发中大批量的都是使用Json来从服务端获取数据，也只有学习一下了。另外，值得一提的是以前，通过在Http地址中带上方法到接口去获取对应的信息，和在数据包内包含协议，究竟哪一个好，还真不知道
<!--more-->

# 概述
JsonHelp提供了很多方法来简化从Json对象到Java对象的转换，及其相逆的转换。我们来看一下。

# JSONHelper

## toJSONObject
这个方法，会被Java 对象，转换为一个JSON对象。那JSON对象又是怎么样的？在安卓内部的JSONObject类中，我们可以看到，其实其只是一个Map。

```java
    private final LinkedHashMap<String, Object> nameValuePairs;
```

有点类似反射，获取当前类中的所有字段，找到注解的字段，获取字段的值，然后放到 JSON对象的 LinkedHashmap 中。

```java
    public static JSONObject toJSONObject(@NonNull Object object) {
        JSONObject jsonObject = new JSONObject();
        Class<?> currentClass = object.getClass();
        while (currentClass != Object.class) {
            Field[] fields = currentClass.getDeclaredFields();
            for (Field field : fields) {
                if (!field.isAccessible())
                    field.setAccessible(true);

                if (field.isAnnotationPresent(JsonTransparent.class)) {
                    // 忽略掉JsonTransparent注解的部分
                    continue;
                }

                int modifiers = field.getModifiers();
                if (Modifier.isFinal(modifiers) || Modifier.isStatic(modifiers)) {
                    // 忽略掉static 和 final 修饰的变量
                    continue;
                }

                String fieldName;
                if (field.isAnnotationPresent(JsonField.class)) {
                    fieldName = field.getAnnotation(JsonField.class).value();
                } else {
                    fieldName = field.getName();
                }

                try {
                    Object value = getJsonObject(field.get(object));
                    if (value == null) {
                        continue;
                    } else {
                        jsonObject.put(fieldName, value);
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
            currentClass = currentClass.getSuperclass();
        }
        return jsonObject;
    }
```
## toJSONArray
JSONArray是一个 List：

```java
    private final List<Object> values;
```

`toJSONArray`就是把一个对象列表逐个的放进去。

```java
    public static JSONArray toJSONArray(@NonNull List<?> datas) {
        JSONArray jsonArray = new JSONArray();
        for (Object object : datas) {
            jsonArray.put(getJsonObject(object));
        }
        return jsonArray;
    }
```

## getJsonObject
这个方法将Java对象转换为JSON支持的类型。
## toList
将JSONArray转换为一个List。对于JSONArray中的每个元素，获取其值，转换为Java对象后放到列表内。

```java
 public static <V> List<V> toList(@NonNull JSONArray jsonArray, @NonNull Class<V> entityType) {
        List<V> list = new ArrayList<>();

        for (int i = 0; i < jsonArray.length(); i++) {
            try {
                V v;

                String json = jsonArray.getString(i);
                if (json.startsWith("{")) {
                    JSONObject jsonObject = new JSONObject(json);
                    v = toObject(jsonObject, entityType);
                } else {
                    v = (V) getJavaObject(json, entityType, null);
                }

                if (v != null) {
                    list.add(v);
                } else {
                    Log.d("json", "json序列化失败：" + json);
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

        return list;
    }
```
## initObject
## toObject
把JSONObject转换为Java对象。

```java
    @Nullable
    public static <V> V toObject(@NonNull JSONObject jsonObject, @NonNull Class<V> entityType) {
        V v;
        try {
            v = entityType.newInstance();
            initObject(jsonObject, v);
        } catch (Exception e1) {
            e1.printStackTrace();
            v = null;
        }
        return v;
    }
```
## getJavaObject
## getCollectionClass

# JSONObject
当我们获取数据，比如从网络获取的时候，其实获得的是JSON的字符串。查看了一下HttpRequester中的代码，在`OnSuccess`方法中，会把反馈回来的字符串建立一个JSON对象。

```java
        public void onSuccess(int statusCode, Header[] headers, String responseString) {
            Logger.log(Logger.HTTP, TAG + "->onSuccess()->code = " + statusCode + ", content = " + responseString);
            try {
                JSONObject jsonObject = new JSONObject(responseString);
                int state = jsonObject.optInt("state", 0);
                String msg = CodeConfig.getCodeTip(state);
                HttpRequester.this.onFinish(state, msg, jsonObject);
            } catch (JSONException e) {
                e.printStackTrace();
                HttpRequester.this.onFinish(CodeConfig.CODE_FAIL, e.getMessage(), null);
            }
        }
```

当建立成JSON对象后，我们就可以做我们想做的操作了。对象本身就提供了很多方法来操作数据。

我们知道，JSON对象其实放数据的就是一个`LinkedHashMap`。我们所做的事情，其实都是对这个Map进行获取而且了。

我们最常用的，其实就是获取字符串我们来看一下`optString`这个方法。

```java
    public String optString(String name, String fallback) {
        Object object = opt(name);
        String result = JSON.toString(object);
        return result != null ? result : fallback;
    }
    
    public Object opt(String name) {
        return nameValuePairs.get(name);
    }
```

好简单不是，其实就是从Map内获取出获取键的值，然后把这个值转换成字符串返回。

但是我们从服务器返回的数据，有可能是一个列表，一个对象，或者一个复杂的对象结构。这个时候，就不能简单的从对象内获取字符串了。

# 总结
总结以上流程，我们简单来归纳一下。当我们调用HttpRequester的时候，其返回的数据会被封装成JSONObject对象返回给调用进程。

接着，我们就可以从这个返回对象内获取我们想要的部分，然后根据对应的类，来转换成Java对象了。

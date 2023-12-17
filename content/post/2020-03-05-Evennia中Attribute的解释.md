---
title: Evennia中Attribute的解释
categories:
  - Mud
date: 2020-03-05 15:32:23
updated: 2020-03-05 15:32:23
tags: 
  - Mud
  - Python
  - Evennia
---

Evennia 游戏中的所有东西都用 Object 来表示，某个对象有什么特性，完全通过 Attribute 来存储各类数据。而我们实际上可以在 Attribute 内存储任何类型的数据，只需要我们进行合适的序列化与反序列化就行了。事情的起因是，当我查看一个账号登录的时候，拉取角色的时候，没有看到角色构建的过程，很迷惑这个问题。所以深入来了解了一下。

<!--more-->

# 文前

对于所有的 TypeClass 对象，都继承自 TypedObject，然后其内定义了一个 attribute 属性：

```python
    @lazy_property
    def attributes(self):
        return AttributeHandler(self)

```

懒加载形式，只在使用的时候才会用到。



# Attribute

Attribute 是一个 模型，其同样使用了 `SharedMemoryModel` 进行缓存。他只定义了几个字段：

- db_key。这个没什么说的。
- db_value。这个定义为类型 PickledObjectField ，可以接收任何形式的 Python 对象。只是在存储和取除的时候要做一下转换操作。
- db_strvalue。存储字符串，主要来表征 db_value，方便快速搜索。
- db_category。可选，分类。
- db_lock_storage。锁。
- db_model。属性是附加到哪个模型上的。
- db_attrtype。属性的子类。（Attribute 的子类）
- db_date_created。创建时间

对于属性的操作，增删改查，实际上都是通过一个 AttributeHandler 来管理的。



# AttributeHandler

每个对象都会有一个 AttributeHandler。简单的公布了三个方法 `get(), set(), has()`。当我们访问某个属性的时候，就会触发 `get()` 方法。这个方法首先会从此 Handlder 的缓存里面取，如果没有，那么就会去查数据库。

获取属性后，实际上返回的就是这个属性的值。而对于这个值，是什么类型就有很多说法了。

```python
    def get(
        self,
        key=None,
        default=None,
        category=None,
        return_obj=False,
        strattr=False,
        raise_exception=False,
        accessing_obj=None,
        default_access=True,
        return_list=False,
    ):
        ret = []
        for keystr in make_iter(key):
            # it's okay to send a None key
            attr_objs = self._getcache(keystr, category)
            print(attr_objs)
            if attr_objs:
                ret.extend(attr_objs)
            elif raise_exception:
                raise AttributeError
            elif return_obj:
                ret.append(None)

        if accessing_obj:
            # check 'attrread' locks
            ret = [
                attr
                for attr in ret
                if attr.access(accessing_obj, self._attrread, default=default_access)
            ]
        if strattr:
            ret = ret if return_obj else [attr.strvalue for attr in ret if attr]
        else:
            ret = ret if return_obj else [attr.value for attr in ret if attr]

        if return_list:
            return ret if ret else [default] if default is not None else []
        return ret[0] if ret and len(ret) == 1 else ret or default

```



获取属性，后，返回值，触发 Attribute.value 方法：

```python
    def __value_get(self):
        """
        Getter. Allows for `value = self.value`.
        We cannot cache here since it makes certain cases (such
        as storing a dbobj which is then deleted elsewhere) out-of-sync.
        The overhead of unpickling seems hard to avoid.
        """
        return from_pickle(self.db_value, db_obj=self)

```

这就需要将值转换一下。这个方法就不贴了，太长。

但是最重要的一点就是，对于是数据库对象的值，其会构建成相关的对象。

```python
        elif _IS_PACKED_DBOBJ(item):
            # this must be checked before tuple
            return unpack_dbobj(item)

```

# unpack_dbobj

```python
def unpack_dbobj(item):
    _init_globals()
    try:
        obj = item[3] and _TO_MODEL_MAP[item[1]].objects.get(id=item[3])
    except ObjectDoesNotExist:
        return None
    except TypeError:
        if hasattr(item, "pk"):
            # this happens if item is already an obj
            return item
        return None
    if item[1] in _IGNORE_DATETIME_MODELS:
        # if we are replacing models we ignore the datatime
        return obj
    else:
        # even if we got back a match, check the sanity of the date (some
        # databases may 're-use' the id)
        return _TO_DATESTRING(obj) == item[2] and obj or None


```

实际上是拿到相关的 model,和 ID，然后来获取。

# pack_dbobj

```python
def pack_dbobj(item):
    """
    Check and convert django database objects to an internal representation.

    Args:
        item (any): A database entity to pack

    Returns:
        packed (any or tuple): Either returns the original input item
            or the packing tuple `("__packed_dbobj__", key, creation_time, id)`.

    """
    _init_globals()
    obj = item
    natural_key = _FROM_MODEL_MAP[
        hasattr(obj, "id")
        and hasattr(obj, "db_date_created")
        and hasattr(obj, "__dbclass__")
        and obj.__dbclass__.__name__.lower()
    ]
    # build the internal representation as a tuple
    #  ("__packed_dbobj__", key, creation_time, id)
    return (
        natural_key
        and ("__packed_dbobj__", natural_key, _TO_DATESTRING(obj), _GA(obj, "id"))
        or item
    )

```

打包的时候，数据库对象存储的实际上是一个元组：

```python
('__packed_dbobj__','model",'date','id')
```



最后再进行一下序列化。



# 最后

这个数据库查询也很诡异，查询出结果后绑定到模型实例上，这个过程是 Django 做的，没空看代码了。
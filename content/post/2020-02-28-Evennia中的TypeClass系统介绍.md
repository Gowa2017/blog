---
title: Evennia中的TypeClass系统介绍
categories:
  - Mud
date: 2020-02-28 00:45:36
updated: 2020-02-28 00:45:36
tags: 
  - Mud
  - Python
---

Evennia 的灵活性来源于其所构建的 TypeClass 体系。我们在之前的文章中已经大概介绍了 TypeClass 是个什么东西。

<!--more-->

# TypeClass

根据 [官方文档定义](https://github.com/evennia/evennia/wiki/Typeclasses)。**TypeClass** 是 Evennia 中的核心存储构成。其允许我们在 Evennia 中用来将任意不同的游戏实体表示为 Python 类，而不需要改造数据库的结构。

在 Evennia 中，最重要的几个游戏实体，`Accounts, Objects, Scripts, Channels` 都是最终从 `evennia.typeclasses.models.TypedObject` 继承的 Python 类。

```
                  TypedObject
      _________________|_________________________________
     |                 |                 |               |
1: AccountDB        ObjectDB           ScriptDB         ChannelDB
     |                 |                 |               |
2: DefaultAccount   DefaultObject      DefaultScript    DefaultChannel
     |              DefaultCharacter     |               |
     |              DefaultRoom          |               |
     |              DefaultExit          |               |
     |                 |                 |               |
3: Account          Object              Script           Channel
                   Character
                   Room
                   Exit
```

所以我们有必要来探究一下 TypeObject 是个什么东西。



# TypeObject

TypeObject 定义在 ``evennia.typeclasses.models.TypedObject`` 中。其是一个 [抽象 Django 模型 Abstract Django model](https://docs.djangoproject.com/en/3.0/topics/db/models/#abstract-base-classes)。

> 抽象的Django基类在我们想要将一些公用的信息放到很多其他模型的时候非常的有用。我们只需要在我们的基类的 Meta 类中加上字段 `abstract=True` 中就可以了。这个 Model 就不会被用来在数据库中建立任何表。只有在其他其继承类中的模型才会建立表，而其定义的信息也会一起定义进去。

这就是为什么我们在数据库中看不到 TypeObject 表的原因。其定义了几个基本的字段：

```python
      key - main name
      name - alias for key
      typeclass_path - the path to the decorating typeclass
      typeclass - auto-linked typeclass
      date_created - time stamp of object creation
      permissions - perm strings
      dbref - #id of object
      db - persistent attribute storage
      ndb - non-persistent attribute storage

```

其定义了两个比较重要的字段，对应了相关的模型：

```python
    # many2many relationships
    db_attributes = models.ManyToManyField(
        Attribute,
        help_text="attributes on this object. An attribute can hold any pickle-able python object (see docs for special cases).",
    )
    db_tags = models.ManyToManyField(
        Tag,
        help_text="tags on this object. Tags are simple string markers to identify, group and alias objects.",
    )

```

也即是说，所有继承自 TypeObject 的模型，都会与 `Attribute, Tag` 这两个模型产生关联关系，我想如果数据大了，这两个表，特别是 `Attribute` 将会很大，瓶颈就可以出现在这里。假设我们的 Object ，游戏内的对象，有 100万个，每个对象有 10 个属性，那么就有 1000 万条记录在这里面。当然，我们可以将某些对象序列化为一条记录来完成，不知道这样是否可行。

而对于 TypeObject 中在数据库的属性，`Attribute`，也叫 `db` 实际上是通过一个 `AttributeHandler` 来完成的，而此 Handler 则会负责对属性进行存储，读写，缓存的。

# 实现
对于 Evennia 抽象的几个类型 `Account, Object, Channel, Script`  其实都是 TypeClass。在此，我们又不能首先再介绍一个关键的类了：

# 关键类

- `Model` Django 的模型
- `SharedMemoryModel(Model, metaclass=SharedMemoryModelBase)` 所有通过 Evennia 的 idmapper 进行缓存对象的基类。这个实际上只定义缓存相关的操作。
- `TypedObject(SharedMemoryModel)` **抽象的 Django 代理类**，实际上是定义了一些 TypeClass 对象都会拥有的公共属性。（包括数据库字段的定义，及一些通过 Handler 来控制的属性，如 Attribute。需要注意的是，在 Python 方法也是属性，只不过此属性名称所对应的是一个函数方法对象）
- `ObjectDB(TypedObject)`继承自 *TypedObject* ，定义游戏内对象的所拥有的属性和行为（方法）
- `AccountDB(TypedObject, AbstractUser)` 同样。
- `ChannelDB(TypedObject)` 同样
- `ScriptBase(ScriptDB, metaclass=TypeclassBase)` 同样。

我们可以看到，在这几个类中都用到了  `metaclass` 的概念。关于 Python 的 `metaclass` 有什么用，在文章 {% post_link Python中类的构建过程 Python中类的构建过程 %} 进行了描述。简而言之，`metaclass` 控制我们的 **类对象** 如何建立的，确定了建立后的 **类对象** 是如何的。

在我们实际的场景下，实际上就是控制我们的 `TypeClass` 是如何建立的，建立后的结果是如何的。那么 `TypeClass` 的类对象建立后，用此类构建的对象拥有哪些表现也就决定了。


# Django 题外话

**Django 的抽象代理类**，只需要在一个模型的内增加一个 `Meta` 类，同时设置 abstract 即可。这样就可以将当前模型中定义的属性让其子类模型继承，而抽象代理类本身不会往数据库有什么关联。

 **代理模型**：代理模型让我们不需要修改原来的模型，而只在代理模型上进行修改，删除等操作，而数据最终会由原始的模型进行存储。不同的是，我们可以改变原始模型的排序，默认的管理器等，而不需要改变原始的模型。这种代理模型只怎样在一个类的 `Meta` 内部类中设置 `proxy=True` 即可。

 我们如果想要改变默认的模型的管理器的时候，只需要在代理模型中替换 `objects`  字段就行了。

 ```python
 
class NewManager(models.Manager):
    # ...
    pass

class MyPerson(Person):
    objects = NewManager()

    class Meta:
        proxy = True
 ```

# MetaClass

有两个关键的 `MetaClass`：

- `SharedMemoryModelBase(ModelBase)` 建立一个 模型，但是此模型会使用 idmapper，
- `TypeclassBase(SharedMemoryModelBase)` 这个是针对 TypeClass 的 `MetaClass` ，控制 typecalss 类对象的建立，我们可以看到，其是继承自 `SharedMemoryModelBase `，因此，所有的 TypeClass 都使用了 idmapper 来保证我们内存中的对象不会因为从 Django 处拿到的数据库映射产生变化。

## SharedMemoryModelBase
这个用来控制一个 `typeclass`，其作为  `metaclass` 控制一个模型在建立的时候，使其能在类对象上缓存数据。

### `__call__`
定义了这个方法，那么所有使用  idmapper 的 Model ，都能用 `Model()` 的形式调用。也就是说，使用这个 `metaclss` 建立好的 **类对象**，在构建实例的时候，必然经过这个 `__call__` 方法。我们来看一下其用途：

```python
    def __call__(cls, *args, **kwargs):
        def new_instance():
            return super(SharedMemoryModelBase, cls).__call__(*args, **kwargs)

        instance_key = cls._get_cache_key(args, kwargs)
        # depending on the arguments, we might not be able to infer the PK, so in that case we create a new instance
        if instance_key is None:
            return new_instance()
        cached_instance = cls.get_cached_instance(instance_key)
        if cached_instance is None:
            cached_instance = new_instance()
            cls.cache_instance(cached_instance, new=True)
        return cached_instance
```
可以看到，事实上并不复杂，就是在某个模型的时候，查一下类对象上的缓存是否存在，如果存在就返回已缓存的数据，否则的话就从数据库去拿，然后同时缓存到类对象上。

类对象的缓存，是在构建类对象的时候就确定的。见下一个方法。另外，在进行缓存对象的时候，还会触发对于对象的 `at_init()` 方法



```python
  # SharedMemoryModel
  	@classmethod
    def cache_instance(cls, instance, new=False):
        pk = instance._get_pk_val()
        if pk is not None:
            cls.__dbclass__.__instance_cache__[pk] = instance
            if new:
                try:
                    # trigger the at_init hook only
                    # at first initialization
                    instance.at_init()
                except AttributeError:
                    # The at_init hook is not assigned to all entities
                    pass

```

同时，在 `Model` 更新了数据的时候，也会对缓存进行刷新，这就会重新触发 `at_init` 方法。



### `__new__`

这个方法，用来构建一个类对象，具体而言，事实上就是在类对象上挂上缓存，然后封装一些 `wrapper` ，然后调用 Django 的 模型去建立类对象

```python
    def __new__(cls, name, bases, attrs):

        attrs["typename"] = cls.__name__
        attrs["path"] = "%s.%s" % (attrs["__module__"], name)
        attrs["_is_deleted"] = False

        # set up the typeclass handling only if a variable _is_typeclass is set on the class
        def create_wrapper(cls, fieldname, wrappername, editable=True, foreignkey=False):
            "Helper method to create property wrappers with unique names (must be in separate call)"
		.....
        for fieldname, field in (
            (fname, field)
            for fname, field in list(attrs.items())
            if fname.startswith("db_") and type(field).__name__ != "ManyToManyField"
        ):
            foreignkey = type(field).__name__ == "ForeignKey"
            wrappername = "dbid" if fieldname == "id" else fieldname.replace("db_", "", 1)
            if wrappername not in attrs:
                # makes sure not to overload manually created wrappers on the model
                create_wrapper(
                    cls, fieldname, wrappername, editable=field.editable, foreignkey=foreignkey
                )

        return super().__new__(cls, name, bases, attrs)
```

可能你会疑惑，那么缓存池呢？看下一个方法

### _prepare

这个方法，是交由 Django 模型建立的 `__new__` 方法进行调用的，目的就是为了在类对象上进行一些自定义的操作：

```python
    def _prepare(cls):
        """
        Prepare the cache, making sure that proxies of the same db base
        share the same cache.

        """
        # the dbmodel is either the proxy base or ourselves
        dbmodel = cls._meta.concrete_model if cls._meta.proxy else cls
        cls.__dbclass__ = dbmodel
        if not hasattr(dbmodel, "__instance_cache__"):
            # we store __instance_cache__ only on the dbmodel base
            dbmodel.__instance_cache__ = {}
        super()._prepare()
```

这就能看到我们的缓存是在类对象上的建立了。当我们的要构建的类对象的 `Meta` 属性设置了 `proxy=True` 的时候，那么就是交由 `SharedMemoryModelBase` 进行管理的。

## TypeclassBase

其继承自上述的 `ShareMemoryModelBase` 类，然后自己定义了一个 `__new__` 方法，也就说，实际上 typeclass 有自己的一套构建机制。

### __new__

```python
    def __new__(cls, name, bases, attrs):
        """
        We must define our Typeclasses as proxies. We also store the
        path directly on the class, this is required by managers.
        """

        # storage of stats
        attrs["typename"] = name
        attrs["path"] = "%s.%s" % (attrs["__module__"], name)

        # typeclass proxy setup
        if "Meta" not in attrs:

            class Meta(object):
                proxy = True
                app_label = attrs.get("__applabel__", "typeclasses")

            attrs["Meta"] = Meta
        attrs["Meta"].proxy = True

        new_class = ModelBase.__new__(cls, name, bases, attrs)

        # attach signals
        signals.post_save.connect(call_at_first_save, sender=new_class)
        signals.pre_delete.connect(remove_attributes_on_delete, sender=new_class)
        return new_class
```
其具体的作用就是，将所有的 TypeClass 都设置为代理模型，然后就将缓存都交给了 `ShareMemoryModelBase` 来进行管理。

同时我们可以看到，还做了两个回调操作：

```python
        signals.post_save.connect(call_at_first_save, sender=new_class)
        signals.pre_delete.connect(remove_attributes_on_delete, sender=new_class)
```
当 Django 的模型存储和准备删除数据的时候，然后告知其子类进行相关的操作：

```python
def call_at_first_save(sender, instance, created, **kwargs):
    """
    Receives a signal just after the object is saved.
    """
    if created:
        instance.at_first_save()


def remove_attributes_on_delete(sender, instance, **kwargs):
    """
    Wipe object's Attributes when it's deleted
    """
    instance.db_attributes.all().delete()
```

可以为什么不删除 Tags 呢？

因此 TypeClass 还有一个对于

# Manger 

前文提到，我们的代理模型，可以设置自己的 Manager，这里确实是这样做的。其实所有的 Manager 都有追溯到 `SharedMemoryManager` ，这个就是在获取数据的会走一下缓存。首先从缓存获取。

- `SharedMemoryManager` 只定义了一个 `get()` 方法，就在获取数据的时候查一下缓存是否存在，如果不存在，就调用 Django 模型去获取数据回来。
- `TypedObjectManager` 定义了所有数据库对象的管理器。主要定义了很多方便的获取属性啊，nick，等的方法。
- `TypeclassManager` ，继承自 `TypedObjectManager`。这个存在的意思是限制给定 typeclass 对数据库的查询，尽管技术上来说，所有的 typeclass 都被定义为同样的核心数据库模型。
- `ObjectDBManager` 继承自 `TypedObjectManager`，定义一些针对具体模型的搜索方法。
- `ObjectManager`继承自 `TypedObjectManager, TypeclassManager` 主要是将查询限制在当前的 TypeClass 上。
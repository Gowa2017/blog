---
title: Evennia中的Object使用的介绍
categories:
  - Mud
date: 2020-02-29 22:30:13
updated: 2020-02-29 22:30:13
tags: 
  - Mud
  - Python
  - Evennia
---

在 Evennia 中，Object 是最广为使用的一个 TypeClass，基本上所有游戏内的内容都是一个 Object，其拥有的 Attribute 和 Tags 让其变得非常的灵活，但如何使用也不是很容易掌握的。其定义的许多回调，也必须了解一下，才不会出现意外的错误。

<!--more-->

# Objects 上的属性和函数

在[文档-Objects 上的属性和函数中](https://github.com/evennia/evennia/wiki/Objects#properties-and-functions-on-objects)，有详细的说明。其拥有所有 `TypeObject` 定义的所有属性，然后其自己又定义了些额外的属性：

## 属性

- `aliases`：一个 Handler ，允许我们对 Object *添加* * 和 *移除* 别名。使用 `aliases.add(), aliases.remove()` 来添加或移除。这有点类似于 Django 中的使用方式。
- `location`：当前容纳此对象的那个对象。在 Evennia 中，任何对象都可以放在其他对象中。
- `home`：一个备份位置。这个属性的主要目的是在其 `location` 引用的对象销毁的时候，此对象能安全的移动到 `home`。所有的对象都应该让 `home` 有一个具体的值。
- `destination`：引用另外一个对象。其主要的使用还是在 `Exits` 上，否则的话一般都不会设置这个值。
- `nicks`：与 `aliases` 相反，一个 `Nick` 引用的是一个仅在此对象上有效的 *真实名字，词或序列* 的替代。这主要在 Object 作为 Character 的时候有用，这可以让我们快速的引用游戏命令或者其他 Character。使用 `nicks.add(alias, realname)` 来添加 Nick。也就是说，aliases 只是一个名称，而 Nick 是拥有特定对象和意义的。
- `account`：这引用一个 **已连接的** ，控制这个对象的 Account 对象。注意，这个属性及时 Account 不在线也会设置。如果要测试一个 Accout 是否在线，使用 `has_account` 属性 来进行判断。
- `sessions`：如果 `account `字段已设置，同时 `account` 在线，这是一个所有活跃的 ServerSession，用此字段来与这些 ServerSession 进行联系。
- `has_account`：用来判断是否有一个已在线的 Account 连接到了这个对象。
- `contents`：返回所有在此对象内的对象列表（这些对象的 `location` 字段引用此对象）
- `exits`：返回在此对象内的所有  *Exits* 对象列表，也就是，这些对象`destination` 已设置。

有两个特殊的属性：

- `cmdset`：一个 Handler ，存储所有定义在此对象的 **command sets**。
- `scripts`：一个 Handler，管理所有附加到此对象上的 **Scripts**。

## 函数

Object 定义了很多有用的工具函数。可以查看源代码文件 `src/objects/objects.py` 来查看。

- `msg()`：用来从服务器发送消息至连接到此对象的 Accout。
- `msg_contents()`：对此对象内的所有对象调用 `msg()` 方法。
- `search()`：搜索特定的对象，在指定的位置，或者全局搜索。这在定义命令的时候很有用（执行命令的对象一般被被命名为 `caller`，然后我们就可以调用 `caller.search()`来在当前房间内查找需要操作的对象。
- `execute_cmd()`：让此对象执行所给予的字符串命令，就跟在命令行上指定的一样。
- `move_to()`：将此对象 **完整的** 移动到一个新的位置。这是主要的移动方法，其会调用所有相关的 hooks，所有的检查等。
- `clear_exits()`：删除此对象上 *From*, *to*，Exits。
- `clear_contents()`：不会删除任何数据，不过会将其内的所有对象（除了 Exits ）都移动到对象的 `home` 位置。
- `delete()`：删除此对象，会先调用 `clear_exits()`，和`clear_contents()`。
- `return_appearance(self, looker, **kwargs)`：格式化一个字符串。`look` 命令应该调用此方法。

### return_appearance

默认实现：

```python
    def return_appearance(self, looker, **kwargs):
        """
        This formats a description. It is the hook a 'look' command
        should call.

        Args:
            looker (Object): Object doing the looking.
            **kwargs (dict): Arbitrary, optional arguments for users
                overriding the call (unused by default).
        """
        if not looker:
            return ""
        # get and identify all objects
        visible = (con for con in self.contents if con != looker and con.access(looker, "view"))
        exits, users, things = [], [], defaultdict(list)
        for con in visible:
            key = con.get_display_name(looker)
            if con.destination:
                exits.append(key)
            elif con.has_account:
                users.append("|c%s|n" % key)
            else:
                # things can be pluralized
                things[key].append(con)
        # get description, build string
        string = "|c%s|n\n" % self.get_display_name(looker)
        desc = self.db.desc
        if desc:
            string += "%s" % desc
        if exits:
            string += "\n|wExits:|n " + list_to_string(exits)
        if users or things:
            # handle pluralization of things (never pluralize users)
            thing_strings = []
            for key, itemlist in sorted(things.items()):
                nitem = len(itemlist)
                if nitem == 1:
                    key, _ = itemlist[0].get_numbered_name(nitem, looker, key=key)
                else:
                    key = [item.get_numbered_name(nitem, looker, key=key)[1] for item in itemlist][
                        0
                    ]
                thing_strings.append(key)

            string += "\n|wYou see:|n " + list_to_string(users + thing_strings)

        return string

```

1. 获取所有在此对象内的对象列表
2. 根据其内的对象是否拥有 `destination`，`has_account` 来判断是 Exits ，还是 Character，Thins。
3. 然后显示对象的`name`
4. 显示对象的 `desc`
5. 将 Character 和 Thing 都显示出来。



## 更多

Object TypeClass 定义了很多的 hook，不仅仅是 `at_object_creation()`。Evennia 会在不同的时间点调用这些函数。当实现我们自己的对象的时候，我们会重载这些 hook。可以在 `evennia.objects.objects` 或者是 [API for DefaultObject here](http://localhost:4000/evennia.objects.objects#defaultobject) 查看这些 Hook。



## Object 的子类

默认有三个特殊的子类——*Characters, Rooms, Exits*。

### Characters

**角色**，是被  Accout 控制的。当一个 账号 **第一次** 登录 Evennia 的时候，会创建一个新的 **角色**，同时将此 **账号** 设置为 **角色** 的 account 属性。一个 **角色**  **必须** 有 **默认命令集**。

### Rooms

**房间**，是所有其他对象的根容器。房间 与其他对象的不同是其不拥有 `location` 属性，并且默认的命令如 `dig` 会创建此类的对象。

### Exits

**出口**，是用来连接其他对象的对象（通常是连接 Rooms ）。一个叫做 *North* 或者 *in* 可能会是一个 Exits，或者是 *door, portal, jump out the window* 这样的名字。Exits 与其他对象的不同在于。

首先，`destination` 会被设置，同时指向一个有效的对象。这个事实可以让我们很快的在数据库中定位 Exits。

然后， Exits 在其创建的时候定义了一个特殊的[Transit Command](https://github.com/evennia/evennia/wiki/Commands) 命令。这个命令与 Exit 对象名称一致，当调用这个命令的时候，负责将对象移动到 Exits 的 `destination`。这就允许我们只输入 Exits 的名字，就能进行移动了。

Exits 的功能定义在 Exit TypeClass 上，所以我们可以通过在其内进行修改来适配我们自己的游戏。（并不推荐这样做，除非你真的知道你在干什么）。

Exits 被一个叫做 *traverse* 的 access_type 所锁定，同时也会调用一些 hooks 来在失败的时候进行返回信息。可以查看 `evennia.DefaultExit`。

通过一个 Exit 的过程如下：

1. `obj` 会发送一个与 Exit 对象一致的 Exit 命令名称。**cmdhandler** 会检查这一点，然后触发定义在 Exit 上的命令。**通过** 总是会涉及 **源**（当前位置）和 `destination`（存储在 Exits 上）。
2. Exit 命令会检查Exit 对象上的 `traverse` 锁。
3. Exit 命令会在 Exit 对象上触发 `at_traverse(obj, destination)`。
4. 在 `at_traverse` 中，会触发 `object.move_to(destination)`。而其会继续触发以下的 hooks：
   1. `obj.at_before_move(destination)`，如果返回  False，那么移动中止。
   2. `source.at_before_leave(obj, destination)`
   3. `obj.announce_move_from(destination)`
   4. 通过改变 `obj` 的 `obj.location` 值为 `destination`。
   5. `obj.announce_move_to(source)`
   6. `destination.at_object_receive(obj, source)`
   7. `obj.at_after_move(source)`
5. 在 Exit 对象上，触发 `at_after_traverse(obj, source)`。

如果移动失败了，Exit 会查找其身上的一个 `err_traverse` 的属性，并将其显示为一个错误信息。如果这个没有找到，那么就会调用  `at_failed_traverse(obj)`。

# Hooks

- `at_object_post_copy()`：被 `DefaultObject.copy()` 调用。意味着会被重载。当有额外数据没有被 `.copy` 方法处理的时候有用。
- `at_first_save(self)`：这个会在类的实例第一次被存储时被 TypeClass 系统调用。这是一个通用的 hook，用来对游戏的不同实体调用 启动hooks 。当重载的时候，一般不需要重载这个方法，只需要重载被此函数调用的方法即可。
- `at_object_creation(self)`：在对象被建立时，只调用一次。多数都会重载这个方法。
- `at_object_delete(self)`：只有在调用 `delete()` 方法，永久的删除了数据库之前调用。当此方法返回 False，那么 删除操作会中断。
- `at_init(self)`：对象被初始化的时候总是会被调用——也就是其 typelcass 从内存缓存时。根据需求，此方法会在 **对象建立后第一次使用或激活**，**或者服务器重启或reload** 时调用。
- `at_cmdset_get(self)`：仅当此对象的 *cmdsets* 被 `cmdhandler` 请求前。如果需要在将 *cmdsets* 在传递给 `cmdhandler` 前进行一些改变，那么就该在这个地方处理。即使当前的对象没有 *cmdsets* 也会被调用。
- `at_pre_puppet(self, account, session=None, **kwargs)`：在 Account 连接到此对象前调用。
- `at_post_puppet(self, **kwargs)`：在 puppet 操作已经完成，并且所有 **Account < -> Object** 相关的连接已经建立起来后调用。此时，我们已经能用 `self.account, self.sessions.get()` 来获取 账号 和 会话。所有会话中的最后一个就是最后 puppeting 此对象的会话。
- `at_pre_unpuppet(self, **kwargs)`：**Account** 准备 与一个 puppet 断开连接前调用。
- `at_post_unpuppet(self, account, session=None, **kwargs)`：Account 成功的与此对象断开连接，切断所有的联系后调用。
- `at_server_reload(self)`：当服务器在关闭时被调用。如果我们想要在服务器重启间保存非持久化的数据，应该是在这个地方。
- `at_server_shutdown(self)`：服务器完整的关闭后（不是重启）调用。
- `at_access(self, result, accessing_obj, access_type, **kwargs)`：当对对象`access()` 调用的时候会执行。默认没有实现。这个的返回不会影响 **lock** 检查的结果。这个可以用来，比如说，在特定的位置自定义错误消息或根据 `access()` 的结果进行一下其他的效果。
- `at_before_move(self, destination, **kwargs)`：在移动对象到新的位置之前调用。
- `at_after_move(self, source_location, **kwargs)`：移动完成后调用。在这允许改变对象，因为此时其已经完全的移动了目标位置。
- `at_object_leave(self, moved_obj, target_location, **kwargs)`：一个对象离开此对象前调用。
- `at_object_receive(self, moved_obj, source_location, **kwargs)`：其他对象被移动进此对象后调用。
- ``at_traverse(self, traversing_object, target_location, **kwargs)`：负责处理实际的通过，通常通过调用 `traversing_object.move_to(target_location)` 。这通常只会被 Exits 实现。如果此方法返回 False（通常是因为 `move_to` 返回了 False），那么就不应该调用 `at_after_traverse()`，而是应该调用 `at_failed_traverse()`。
- `at_after_traverse(self, traversing_object, source_location, **kwargs)`：当一个对象此对象成功的到达另外一个对象后调用。
- `at_failed_traverse(self, traversing_object, **kwargs)`：对象通过失败时调用。**如果使用默认的 Exits，如果调用 err_traverse 属性，那么这个方法不会被调用**。
- `at_msg_receive(self, text=None, from_obj=None, **kwargs)`：当其他对象调用此对象的 `msg()` 方法时被调用。*from_obj* 有可能会是 None，如果发送者没有将其自身包含在参数内进行传递过来的话，我们需要检测这一点。可以把此方法视为将消息传递到 Sessions 的一个预处理器。如果此方法返回  False，消息将不会继续传递。
- `at_msg_send(self, text=None, to_obj=None, **kwargs)`：当对象调用其他对象的`obj.msg(text, to_obj=obj)` 方法时被调用。此方法是被 `from_obj` 调用，如果 `from_obj` 为 None，此方法不会被调用。
- `at_look(self, target, **kwargs)`：当此对象执行一个 `look`时。这个允许来自定义此方法的意义。此方法是不会发送任何数据的。
- `at_desc(self, looker=None, **kwargs)`：当其他对象 `look` 此对象时被调用。
- `at_before_get(self, getter, **kwargs)`：`get` 命令在对象被捡起前调用。
- `at_get(self, getter, **kwargs)`：`get` 命令在捡起对象后调用。
- `at_before_give(self, giver, getter, **kwargs)`：`give` 命令在给出前被调用。
- `at_give(self, giver, getter, **kwargs)`：`give` 命令给出了某个对象后调用。
- `at_before_drop(self, dropper, **kwargs)`： `drop` 命令在对象被丢弃前调用。
- `at_drop(self, dropper, **kwargs)`：`drop` 命令丢弃对象后调用。
- `at_before_say(self, message, **kwargs)`：对象 *say* 之前。这个一般会被命令 `say`，`whisper` 调用。会在命令将 文本发出前调用，可以用来调用自定义发出的文字。返回 `None` 会中断命令。
- `at_say( self, message, msg_self=None, msg_location=None, receivers=None, msg_receivers=None, **kwargs, )`：显示 `say, whisper` 的实际内容。这个方法应该在所处的位置显示相关内容。其应该告知此对象及其位置某些文本被说出来了。The overriding of messages or `mapping` allows for simple customization of the hook without   re-writing it completely.

# 使用示例

## 创建新的 Object

```python
 # end of file mygame/typeclasses/objects.py
 from evennia import DefaultObject

 class Heavy(DefaultObject):
    "Heavy object"
    def at_object_creation(self):
        "Called whenever a new object is created"
        # lock the object down by default
        self.locks.add("get:false()")
        # the default "get" command looks for this Attribute in order
        # to return a customized error message (we just happen to know
        # this, you'd have to look at the code of the 'get' command to
        # find out).
        self.db.get_err_msg = "This is too heavy to pick up."

```



## 初始化存储数据

`at_object_creation` 只会在对象首次建立的时候调用一次。这对于数据相关的，如 Attribute 这样的操作很理想。但是某些时候我们想要存储一些临时的属性（就是不会存在数据库，但是每次此对象被建立的时候都要有）。这些属性可以在 `at_init()` 内进行处理，`at_init()` 会在对象每次被加载到内存的时候调用。

> 不要在 at_init 内进行任何数据库相关的操作。

使用 `ndb`(non-database Attributes) 来存储这些非持久化的属性是很有用的。

```python
def at_init(self):
    self.ndb.counter = 0
    self.ndb.mylist = []

```

> 只使用 `at_init` 而不要使用  `__init()` 方法。

## 更新已有的对象

如果我们改变了类型的定义，那么已经存在的对象可能不会拥有某些新的属性。	

如果已经存在的对象数量比较少，我们可以使用 `@typeclass/force/reload objectname` 来强制执行一次 `at_object_creation()`。命令 `@update objectname` 可以达到同样的效果。

如果对象太多，我们可以 `py` 来执行：

```python
@py from typeclasses.objects import Heavy; [obj.at_object_creation() for obj in Heavy.objects.all()]
```


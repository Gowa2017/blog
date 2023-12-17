---
title: Evennia中的Spawner与Prototypes
categories:
  - Mud
date: 2020-03-07 00:59:50
updated: 2020-03-07 00:59:50
tags: 
  - Mud
  - Evennia
  - Python
---

Evennia 中 *Spawner* 是一个用来定义从一个基本的模板来建立不同的对象的系统，这 个模板我们叫它为 *Prototype*。这个设计来只是与 **游戏内对象** 使用，而不是其他类型的实体。

<!--more-->

Evennia 常规的创建自定义对象的方式是使用一个 TypeClass。现在，你想建立一个 *Goblin* 敌人。一个常规的方式是首先建立 `Mobile`  `TypeClass`来保存所有游戏内可运动物体的所有内容，如 AI，战斗代码和很多的移动方式等。然后我们使用一个 `Goblin` 子类来继承它，同时加上一些特有的内容，如基于分组的AI等（因为 Goblin 在一起的时候会更聪明）等。

但是现在我们想创造很多 Goblin 然后放到世界中。我们想要每个 Goblin 看起来都不一样怎么办？比如我们要灰色的，或者绿色的，还有的拿着不同的武器？我们可能会继承 Goblin ，然后创建更多的子类，如 `Greengoblin,`。这看起来太麻烦，这也会产生很多的 Python 代码。而当我们想要将这些都结合在一起的时候，是不现实的。

这就是原型的作用了。它是一个Python 字典，描述每个对象上的不同。原型有个优点就是允许建造者不用访问 Python 后端就能自定义对象。Evennia 也允许保存和搜索原型，这样其他建造者就能看到和使用他们了。对于建造者来说，有一原型库是很好的资源。OLC 系统允许用一个菜单系统我们建立，保存，加载和操作原型。

*Spawner* 生成器使用 原型来建立新的，自定义的对象。



# OLC

可以用命令 `ocl` 或者 `@spawn/ocl` 来进入原型向导。这是一个用来建立，加载，保存和操作原型的菜单系统。其主要是提供给游戏内的建造者使用，而且会给出一个比较易于理解的原型说明。使用 `help` 在菜单中的每个节点上看一下。下面是更多的细节。

# Prototype

原型可以通过 OLC 系统来建立，或者写成 Python 模块（然后被 `@spwan` 命令或者 OLC 引用），或者在随手编写，然后加载到生成器函数（或 `@spawn`命令）。

原型字典定义所有可能的关于某个对象的数据库属性。其有一个固定的允许的键集合。当准备将原型存储在数据库时（或使用 OLC），有些键是强制性的。当只是一次性的将原型字典传递给生成器系统的话，就比较宽松，很多没指定的键会使用默认值。

一个原型可能看起来像下面这样：

```json
{ 
   "prototype_key": "house"
   "key": "Large house"
   "typeclass": "typeclasses.rooms.house.House"
 }

```

如果想要将它加载到游戏中我们只需要：

```
@spawn {"prototype_key="house", "key": "Large house", ...}
```

> 命令行上给出的原型字典必须是有效的 Python 结构——所以我们必须在字符串加上引号。为了安全，这个字典不能包含一些高级的 Python 函数，比如说可执行的代码，`lambda`。如果建造者想要使用这些特性，就必须通过 `$protfuncs`来提供，嵌入式可运行函数，在运行之前，您可以完全控制这些函数并进行检查。

## 原型键

所以以 `prototype_` 开头的键是为了记账

- `prototype_key`：原型的名字。有些时候这可以被略过（例如定义一个在模块内的原型或者只是手动提供一个字典给生成器函数时），但包含它是一个好的实践。它用于簿记和存储原型，以便你以后可以找到它。
- `prototype_parent`：如果给定，这将会是存储在系统中或在一个模块中可用的另外一个原型的 `prototype_key`。这让此原型 *继承* 父原型的键，同时只会覆盖必要的字段。给定一个元组 `(partent1,partent2,...)`来进行多个自左至右的继承。如果没有给定，那么一个 `typeclass` 应该被指定。
- `prototype_desc`：可选。在游戏内进行列出原型时用到。
- `protototype_tags`：可选。用来标记原型，方便更容易的找到。
- `prototype_locks`：支持两种类型的锁：`edit, spawn`。第一个决定在 OLC 使用的时候是否可以复制和编辑。第二个决定谁能使用这个原型来建立对象。

下面的这些键决定了对象的实际表现：

- `key`	对象的主要标识。默认是 *Spawned Object X*。
- `typeclass`		一个完整的 Python（从游戏目录开始）。如果没有定义，那么 `prototype_parent ` 应该被定义，且在父原型中有地方定义了 `typeclass`。在使用一次性的命令行时，如果不提供这个，那么会默认使用 `settings.BASE_OBJECT_TYPECLASS`。
- `location`	位置，这应该是一个 `#dbref`
- `home`	有效的 `#dbref`。如果未定义，那么就和 `location` 或者 `settings.DEFAULT_HOME` 一致。
- `destination`  有效的`#dbref`，只会被 *Exit* 使用。
- `permissions `	权限字符串列表，如：`["Accounts", "may_use_red_door"]`
- `lock`	`edit:all();control:perm(Builder)` 这样
- `aliases` 别名
- `tags` 标签，是元组 `(tag,category,data)`
- `attrs`	属性列表。这应该是以元组给定的 `attrname, value, categroy,lockstring)`
- 任何其他的关键子都被解释为未分类的属性和他们的值。这对于简单的属性是非常有效的-使用 `attrs` 来完整的控制属性。


## 原型值
支持几种不同类型的值。
硬编码：

```json
    {"key": "An ugly goblin", ...}
```
也可以是可调用的。这个调用会以无参的形式调用。

```json
    {"key": _get_a_random_goblin_name, ...}
```
使用 Python 的 `lambda`，我们可以将这个调用包装起来，来立即设置原型中的设置：

```js
    {"key": lambda: random.choice(("Urfgar", "Rick the smelly", "Blargh the foul", ...)), ...}
```

### Protfuncs

最后，这个值可以是 *prototype function*。这看起来就是简单的函数，不过我们以 `$` 开头来嵌入到字符串中：

```js
    {"key": "$choice(Urfgar, Rick the smelly, Blargh the foul)",
     "attrs": {"desc": "This is a large $red(and very red) demon. "
                       "He has $randint(2,5) skulls in a chain around his neck."}
```

在执行阶段，原型函数的位置将会被替换为函数执行后的结果（永远都应该是一个字符串）。一个原型函数和 `InlineFunc` 的工作方式很像，他们事实上使用的是一个 `parser`——不同的是，原型函数是在被生成器用来生成新对象的时候就会被调用。（InlineFunc 用来在当一个文本被发送到用户的时候使用）。

下面就是一个原型函数的定义：

```python
# this is a silly example, you can just color the text red with |r directly!
def red(*args, **kwargs):
   """
   Usage: $red(<text>)
   Returns the same text you entered, but red.
   """
   if not args or len(args) > 1:
      raise ValueError("Must have one argument, the text to color red!")
   return "|r{}|n".format(args[0])
```

> 注意，我们必须验证输入的有效性，并在失败的时候抛出 ValueError。同时，无法在对原型函数的调用中使用关键字（`echo(text,align=left)` 是无效的。`kwargs` 只是为了让 evennia 内部使用，而不是全给原型函数（只有 inlinefunc 使用）。

为了让这个原型函数在游戏内可用，将它添加一个模块，然后将模块添加到全局变量中：

```python
# in mygame/server/conf/settings.py

PROT_FUNC_MODULES += ["world.myprotfuncs"]
```

一个全局可调用的被添加到我们的模块中就会被认为是一个新的原型函数。为了避免这个，让我们的函数以 `_` 开头。默认可用的原型函数位于 *evennia/prototypes/profuncs.py*。想要覆盖这些函数的话，只需要覆盖一个同名的函数就行了。

| Protfunc               | Description                                                  |
| ---------------------- | ------------------------------------------------------------ |
| `$random()`            | Returns random value in range [0, 1)                         |
| `$randint(start, end)` | Returns random value in range [start, end]                   |
| `$left_justify()`      | Left-justify text                                            |
| `$right_justify()`     | Right-justify text to screen width                           |
| `$center_justify()`    | Center-justify text to screen width                          |
| `$full_justify()`      | Spread text across screen width by adding spaces             |
| `$protkey()`           | Returns value of another key in this prototype (self-reference) |
| `$add(, )`             | Returns value1 + value2. Can also be lists, dicts etc        |
| `$sub(, )`             | Returns value1 - value2                                      |
| `$mult(, )`            | Returns value1 * value2                                      |
| `$div(, )`             | Returns value2 / value1                                      |
| `$toint()`             | Returns value converted to integer (or value if not possible) |
| `$eval()`              | Returns result of [literal-eval](https://docs.python.org/2/library/ast.html#ast.literal_eval) of code string. Only simple python expressions. |
| `$obj()`               | Returns object #dbref searched globally by key, tag or #dbref. Error if more than one found." |
| `$objlist()`           | Like `$obj`, except always returns a list of zero, one or more results. |
| `$dbref(dbref)`        | Returns argument if it is formed as a #dbref (e.g. #1234), otherwise error. |

对于可以访问 Python 的开发者来说，在原型中使用原型函数是不实用的。传递实际 的 Python 函数会更加的灵活和强大。他们的主要用途还是给游戏内的建造者能够有限的对他们的原型进行编码/脚本，而不需要给他们 Python 访问权限。

# 存储原型

一个原型可以用两种方式定义和存储，在数据库或者在模块中的字典。

## 数据库原型

在数据库中以 **脚本** 存储。某些时候这会被叫做 *数据库原型*。这是游戏内的建造者唯一一种可修改和添加原型的方式。他们的优点是容易修改，和在建造者共享，但是就是必须用游戏内的命令来使用。

## 基于模块的原型

这些原型被定义成字典，同时被赋给在 `settings.PROTOTYPE_MODULES` 中定义的模块中定义的全局变量。他们只能通过在游戏外进行修改，容易他们在游戏内是只读的（但是复制他们可以成为数据库原型）。这在0.8 以前是唯一的原型模式。

默认情况下，`mygame/world/prototypes.py`  已经设置了，可以在这添加我们自定义的原型。所以在这个模块内的 *全局字典*都会被认为是原型。你也可以让 Evennia 在其他模块去寻找原型：

```python
# in mygame/server/conf.py

PROTOTYPE_MODULES = += ["world.myownprototypes", "combat.prototypes"]
```

这是一个定义了原型的模块的例子：

```python

# in a module Evennia looks at for prototypes,
# (like mygame/world/prototypes.py)

ORC_SHAMAN = {"key":"Orc shaman",
	  "typeclass": "typeclasses.monsters.Orc",
	  "weapon": "wooden staff",
	  "health": 20}
```

> 上面的例子中，ORC_SHAMAN 将会成为这个原型的 `prototype_key`。然而，我们总是建议定义 `prototype_key`

# Using @spawn

假设 `goblin`  typeclass 已经可用，我们可以如下使用：

```
@spawn goblin
```

也可以直接指定字典：

```
@spawn {"prototype_key": "shaman", \
    "key":"Orc shaman", \
        "prototype_parent": "goblin", \
        "weapon": "wooden staff", \
        "health": 20}

```



#  Using evennia.prototypes.spawner()

我们可以在代码中直接使用如下来访问生成器：

```python
    new_objects = evennia.prototypes.spawner.spawn(*prototypes)

```

所有的参数都是原型的字典。这个函数会返回一个匹配的建立后的对象列表：

```python
    obj1, obj2 = evennia.prototypes.spawner.spawn({"key": "Obj1", "desc": "A test"},
                                                  {"key": "Obj2", "desc": "Another test"})

```

注意，当我们使用 `evennia.prototypes.spawner.spawn` 的时候，`location` 会被自动设置，我们必须显式的指定它。

如果我们提供的原型定义了 `prototype_parent` 关键词，生成器将会从 `settings.PROTOTYPE_MODULES` 找这个模型，也会从数据库去找。其还有很多的用法，[参考API](https://github.com/evennia/evennia/wiki/evennia.prototypes.spawner#spawn)
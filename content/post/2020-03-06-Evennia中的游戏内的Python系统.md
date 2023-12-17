---
title: Evennia中的游戏内的Python系统
categories:
  - Mud
date: 2020-03-06 22:22:17
updated: 2020-03-06 22:22:17
tags: 
  - Mud
  - Evennia
  - Python
---

Evennia 游戏中的 Python 系统，允许执行任意的 Python 代码，而只有很少的限制。这样一个系统非常的强力，但也会有潜在的危险，如果我们要使用这样一个系统，那么就要记住其可能出现的危险。但是，这系统的强大又让人需要使用它来实现一些东西。

<!--more-->

# 警告

1. 在这个系统中，不受信任玩家的可能会执行 Python 代码。所以要小心的限制谁可以使用这个系统。
2. 我们完全可以在游戏外的 Python 中完成这些事情。in-game Python 系统并不是为了完全的替代我们的游戏特性。

# 最终逻辑
通过创建一个全局脚本，然后将所有的 （对象，事件，回调）都存储在 脚本的属性上：
根据对象的 ID，查事件，然后过滤回调，来执行代码。很给力啊。
```json
{
	npc: {
		'say': [{
			'created_on': datetime.datetime(2020, 3, 6, 16, 15, 20, 346957),
			'author': g,
			'valid': True,
			'code': 'character.location.msg_contents("{character} shrugs and says: \'well, yes, hello to you!\'", mapping=dict(character=character))',
			'parameters': 'hello',
			'updated_on': datetime.datetime(2020, 3, 6, 16, 15, 37, 899715),
			'updated_by': g
		}, {
			'created_on': datetime.datetime(2020, 3, 6, 16, 17, 22, 634880),
			'author': g,
			'valid': True,
			'code': 'character.location.msg_contents("{character} says: \'Bandits he?\'", mapping=dict(character=character))\ncharacter.location.msg_contents("{character} scratches his head, considering.", mapping=dict(character=character))\ncharacter.location.msg_contents("{character} whispers: \'Aye, saw some of them, north from here.  No trouble o\' mine, but...\'", mapping=dict(character=character))\nspeaker.msg("{character} looks at you more closely.".format(character=character.get_display_name(speaker)))\nspeaker.msg("{character} continues in a low voice: \'Ain\'t my place to say, but if you need to find \'em, they\'re encamped some distance away from the road, I guess near a cave or something.\'.".format(character=character.get_display_name(speaker)))',
			'parameters': 'bandit, bandits',
			'updated_on': datetime.datetime(2020, 3, 6, 16, 18, 23, 497149),
			'updated_by': g
		}]
	}
}
```
# 基本结构和概念

- in-game Pyhon 的基础是 **events（事件）**。一个**事件** 定义了我们想要调用一些 Python 代码上下文。例如，一个定义在 **Exits** 上的事件在每次有角色通过的时候触发。事件会在 `typeclass` 上进行描述。所有继承自此 `typeclass`  的对象都可以访问这些事件。
- **Callbacks（回调）** 可以设置在不同的对象上，或者在代码定义的事件上。这些回调可能包含任意的代码和描述一个对象特定的行为。如果事件被触发，所有连接到此对象事件上的回调都会被执行。

我们在特定场景下观察这个系统，当一个对象被捡起的时候（使用 `get` 命令），一个特定的事件被触发：
1. 事件 *get* 设置在对象上。（`Object` tyleclass）
2. 当使用 `get` 命令来拾取对象的时候，对象的 `at_get` hook 被调用。
3. 一个修改过的 hook 被 *事件系统* 设置在 *DefaultObject* 上。这个 hook 会执行（或调用）这个对象上的 `get` 事件。
4. 所有绑定到此`get`事件上的回调都会被按序执行。这些回调就像包含 Python 代码的函数一样工作，这些代码你可以在游戏内编写（使用在编辑回调本身时将列出的特定变量。）
5. 在不同的回调中，可以在添加多行的 Python 代码。在这个例子中，*character* 变量将会包含捡起此对象的角色，而 *obj* 包含的是被捡起的对象。

这样当我们在此对象上建立一个 `get` 回调，并把他放进来：

```python
character.msg("You have picked up {} and have completed this quest!".format(obj.get_display_name(character)))
```
那当我们捡起此对象的时候，就会收到消息：
```
You pick up a sword.
You have picked up a sword and have completed this quest!
```

# 安装
这个系统是默认没有安装的。我们需要手动进行安装。步骤如下：

1. 启动主脚本 `@py evennia.create_script("evennia.contrib.ingame_python.scripts.EventHandler")`
2. 设置权限（可选）：
	1. `EVENTS_WITH_VALIDATION`：一个用户组可以编辑回调，但是需要审批（默认是 None）
	2. `EVENTS_WITHOUT_VALIDATION`：一个组拥有权限来编辑回调，而不需要验证。
	3. `EVENTS_VALIDATING`：一个用户组可以验证回调。
	4. `EVENTS_CALENDAR`：日历类型（`Node, standard, custom`，默认是 Node）
3. 添加 `call` 命令
4. 从 in-game Python 继承我们自定义的 `typeclass` ：
	-  evennia.contrib.ingame_python.typeclasses.EventCharacter
	-  evennia.contrib.ingame_python.typeclasses.EventExit
	-  evennia.contrib.ingame_python.typeclasses.EventObject
	-  evennia.contrib.ingame_python.typeclasses.EventRoom

下面的各节详细的描述了每个步骤。
> 注意，如果我们启动的时候没有启动主脚本（如重置了数据库），我们可以会在登录的时候收到错误，告知我们 `callback` 属性没有定义。这个时候我们重新执行一下主脚本就行了。

## 启动事件脚本
只需要一个命令就能启动事件脚本：

```
@py evennia.create_script("evennia.contrib.ingame_python.scripts.EventHandler")
```

这个命令会建立一个全局脚本（也就是说，独立于任何对象的脚本）。这个脚本将会保留基本的设置，不同的回调等。我们可以直接访问它，不过我们更应该使用 callback handler（回调管理器）。建立这个脚本，将会在所有的对象上建立一个  `callback` 管理器。

## 编辑权限
这个系统添加了三个权限。我们可以在我们的配置文件中直接进行修改。在事件系统定义的配置如下：

- `EVENTS_WITH_VALIDATION` 定义一个可以编辑回调的权限，但是需要审批。如果我们把这个值设置为 `wizards`，那么拥有 `wizards` 权限的用户就能编辑回调了。这些回调不会被连接，需要被管理员进行检查和审批。这个配置可以包含 `None`，意味着没有用户可以经过验证来编辑回调。
- `EVENTS_WITHOUT_VALIDATION` 定义一个允许编辑回调而不需要验证的权限。默认情况下，这个设置是 `immortals`，意味着只有这个权限的用户才能进行编辑回调，同时不需要管理员验证就能连接到对象。
- `EVENTS_VALIDATING`：定义谁能验证回调。默认情况下，这被设置为 `immortals`，只有他们才能看到需要验证的回调，能接受或者拒绝这些验证。

我们可以在我们的配置文件中覆盖这些设置：

```python
# Event settings
EVENTS_WITH_VALIDATION = "wizards"
EVENTS_WITHOUT_VALIDATION = "immortals"
EVENTS_VALIDATING = "immortals"
```

另外，如果我们想使用 **时间相关的** 事件的话，那么还有一个配置我们必须得设置。需要制定我们使用的日历类型。默认情况下，时间相关的事件是被禁用的。我们可以改变 `EVENTS_CALENDAR` 的设置来启用：

- `standard ` 标准日历，有标准的天，年，月等。
- `custom ` 一个自定义的日历将会使用 [自定义游戏时间](https://github.com/evennia/evennia/blob/master/evennia/contrib/custom_gametime.py) 来调度时间

同时也定义了可以在独立的用户上设置的权限：

- `events_without_validation ` 这将会给予用户编辑回调，且不经过验证的权限。
- `events_validating ` 这个允许用户执行需要验证的回调的有效性检查。

具体而言，如果我们想给某个用户编辑回调而不经过任何审批的权限，（如用户 kaldara），我们可能会像下面这样做：

```
@perm *kaldara = events_without_validation
```
移除权限使用 `del`：

```
@perm/del *kaldara = events_without_validation
```

使用 `@call` 权限与这些设置相关：默认情况下，只有拥有`events_without_validation ` 权限和在组 `EVENTS_WITH_VALIDATION` 内，或此组上的用户才能执行。

## 添加 @call 命令

```python
from evennia import default_cmds
from evennia.contrib.ingame_python.commands import CmdCallback

class CharacterCmdSet(default_cmds.CharacterCmdSet):
    """
    The `CharacterCmdSet` contains general in-game commands like `look`,
    `get`, etc available on in-game Character objects. It is merged with
    the `PlayerCmdSet` when a Player puppets a Character.
    """
    key = "DefaultCharacter"

    def at_cmdset_creation(self):
        """
        Populates the cmdset
        """
        super(CharacterCmdSet, self).at_cmdset_creation()
        self.add(CmdCallback())
```

## 改变 typeclass 的父类
如：

```python
from evennia.contrib.ingame_python.typeclasses import EventCharacter

class Character(EventCharacter):

    # ...
```

# 使用 @call 命令    
`in-Game` Python 系统依赖于其 `@call` 命令。谁可以执行这个命令，谁可以对此命令做些什么，依赖于我们设置的权限。

`@call` 命令允许 添加，编辑和删除特定对象事件的回调。事件系统可以被使用在大多数 Evennia 的对象，基本上所有的 `typeclass` 对象（除了玩家）。`@call` 命令的第一个参数是我们想要编辑的对象的名字。其也可以用来知道特定对象上哪些事件是可用的。

## 测试回调和事件

为了查看某个对象上已连接的事件，可以使用  `@call` 命令，以其名字或者 ID 作为参数。如 `@call here` 来测试当前位置事件，或`call self` 来自己身上的事件。

这个命令会显示一个表，包含：

- 第一列是事件名字
- 第二列，此事件上挂了的回调的数量，及回调的总行数。
- 一个简单的帮助说明回调的用途。

```
+------------------+---------+-----------------------------------------------+
| Event name       |  Number | Description                                   |
+~~~~~~~~~~~~~~~~~~+~~~~~~~~~+~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~+
| can_delete       |   0 (0) | Can the character be deleted?                 |
| can_move         |   0 (0) | Can the character move?                       |
| can_part         |   0 (0) | Can the departing character leave this room?  |
| delete           |   0 (0) | Before deleting the character.                |
| greet            |   0 (0) | A new character arrives in the location of    |
|                  |         | this character.                               |
| move             |   0 (0) | After the character has moved into its new    |
|                  |         | room.                                         |
| puppeted         |   0 (0) | When the character has been puppeted by a     |
|                  |         | player.                                       |
| time             |   0 (0) | A repeated event to be called regularly.      |
| unpuppeted       |   0 (0) | When the character is about to be un-         |
|                  |         | puppeted.                                     |
+------------------+---------+-----------------------------------------------+
```

## 建立新的回调
 `/add` 开关用来添加一个回调。其需要除对象的 **名称/dbref** 外的 两个参数：

 - 在 = 号标志后，指定需要编辑的事件（如果没有指定，将会显示可能的事件列表，如上）
 - 参数（可选）

 我们将在后面看到使用参数的回调。但是现在，让我们来尝试组织一个角色通过当前房间的 *north* 出口。

```
@call north
+------------------+---------+-----------------------------------------------+
| Event name       |  Number | Description                                   |
+~~~~~~~~~~~~~~~~~~+~~~~~~~~~+~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~+
| can_traverse     |   0 (0) | Can the character traverse through this exit? |
| msg_arrive       |   0 (0) | Customize the message when a character        |
|                  |         | arrives through this exit.                    |
| msg_leave        |   0 (0) | Customize the message when a character leaves |
|                  |         | through this exit.                            |
| time             |   0 (0) | A repeated event to be called regularly.      |
| traverse         |   0 (0) | After the character has traversed through     |
|                  |         | this exit.                                    |
+------------------+---------+-----------------------------------------------+
```

如果我们想要阻止一个角色通过这个出口，最好的事件应该是 `can_traverse`。
>为什么不是 `traverse`，根据文档说明，`traverse` 是在角色已经通过了出口后调用的。这已经太晚了。

现在我们来编辑这个事件：

```
@call/add north = can_traverse
```
当一个角色想要通过这个出口的时候就会调用此事件。我们可以用 `deny()` 事件函数来拒绝角色通过。
这个事件中我们可以使用的 变量：

```
- character: the character that wants to traverse this exit.
- exit: the exit to be traversed.
- room: the room in which stands the character before moving.
```
后面的章节会详细的解释 `eventfuncs`。不过现在，我们只需要这样做就行了。在打开的编辑器中我们可以输入如下的代码：

```python
if character.id == 1:
    character.msg("You're the superuser, 'course I'll let you pass.")
else:
    character.msg("Hold on, what do you think you're doing?")
    deny()
```

现在我们使用 `:wq` 来保存这个回调。
如果我们调用 `@call north` 就能看到有活跃的回调了。如果我们使用 `@call north=can_traverse` 可以看更详细的信息。

```python
call north
+------------------+---------+-----------------------------------------------+
| Callback name    |  Number | Description                                   |
+~~~~~~~~~~~~~~~~~~+~~~~~~~~~+~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~+
| can_traverse     |   1 (5) | Can the character traverse through this exit? |
| msg_arrive       |   0 (0) | Customize the message when a character        |
|                  |         | arrives through this exit.                    |
| msg_leave        |   0 (0) | Customize the message when a character        |
|                  |         | leaves through this exit.                     |
| time             |   0 (0) | A repeated event to be called regularly.      |
| traverse         |   0 (0) | After the characer has traversed through      |
|                  |         | this exit.                                    |
+------------------+---------+-----------------------------------------------+
```
```python
call north=can_traverse
+--------------+--------------+-----------------+--------------+-------------+
|       Number | Author       | Updated         | Param        | Valid       |
+~~~~~~~~~~~~~~+~~~~~~~~~~~~~~+~~~~~~~~~~~~~~~~~+~~~~~~~~~~~~~~+~~~~~~~~~~~~~+
|            1 | g            | 18 seconds ago  |              | Yes         |
+--------------+--------------+-----------------+--------------+-------------+
```
```python
call north=can_traverse 1
Callback can_traverse 1 of north:
Created by g on 2020-03-06 15:31:40.
Updated by g on 2020-03-06 15:32:43
This callback is connected and active.
Callback code:
if character.id == 1:
    character.msg("You're the super, pass")
    else
    character.msg('you can'to pass this')
    deny()
```

## 编辑和删除回调

`@call/edit`     `@call/del` 用来编辑和删除回调。

# 使用事件
下面的章节描述了如何用事件来完成很多任务，从最简单的到最复杂的。

## eventfuncs
为了使开发变得更容易， in-game Python 系统提供了 `eventfuncs` 来在回调中使用。我们可以不使用他们，他们只是简化而已。一个 eventfunc 只是一个我们可以在回调代码中使用的函数而已。

| Function   | Argument                 | Description                       | Example                           |
| ---------- | ------------------------ | --------------------------------- | --------------------------------- |
| deny       | `()`                     | Prevent an action from happening. | `deny()`                          |
| get        | `(**kwargs)`             | Get a single object.              | `char = get(id=1)`                |
| call_event | `(obj, name, seconds=0)` | Call another event.               | `call_event(char, "chain_1", 20)` |

### deny

用来中断回调的执行及相应的动作。在 `can_*` 事件中，这用来阻止动作的发生。

### get

根据一个特定的标识来获取一个对象。这通常用来根据一个ID 获取对象。

### call_event

某些回调会调用其他事件。这是链式事件的一部分。用来立即调用另外一事件，或者在定义的时间调用。

第一个参数是对象，第二个参数是事件的名称，第三个参数是调用事件的延时秒数（默认是0，立即调用）

## 回调变量

## 带参调用回调

## 时间相关的事件

## 链式事件

# 代码中使用事件

此节描述了从代码中进行回调和使用事件，如何创建新的事件，如何在命令中调用他们，及如何处理特定的情况（如参数）。

本节中，我们看到如何实现以下例子：我们将会创建一个 `push` 命令来压入对象。对象会对此命令做出反应，同时触发特定的事件。

## 添加新的事件

新的事件应该在 `typeclass` 里面完成。事件被包含在 `_events` 类变量中，这是一个字典，以事件名称为键，一个描述事件的元组作为值。同时我们需要注册此类来告诉 in-game Python 系统其包含事件。

这里我们在对象上添加 `push` 事件。在我们的 `typeclasses/objects.py` 文件中，如此写下：

```python
from evennia.contrib.ingame_python.utils import register_events
from evennia.contrib.ingame_python.typeclasses import EventObject

EVENT_PUSH = """
A character push the object.
This event is called when a character uses the "push" command on
an object in the same room.

Variables you can use in this event:
    character: the character that pushes this object.
    obj: the object connected to this event.
"""

@register_events
class Object(EventObject):
    """
    Class representing objects.
    """

    _events = {
        "push": (["character", "obj"], EVENT_PUSH),
    }

```

 重启游戏，然后我们新建一个对象观察：

```
create box
call box
+------------------+---------+-----------------------------------------------+
| Callback name    |  Number | Description                                   |
+~~~~~~~~~~~~~~~~~~+~~~~~~~~~+~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~+
| drop             |   0 (0) | When a character drops this object.           |
| get              |   0 (0) | When a character gets this object.            |
| push             |   0 (0) | A character push the object.                  |
| time             |   0 (0) | A repeated event to be called regularly.      |
+------------------+---------+-----------------------------------------------+
```

## 代码中调用事件

in-game Python 系统可以所有对象上都有的 handler 来访问。这个 hanlder 被命名为 `callbacks`，且能从任何 `typeclass` 对象访问（角色，房间，出口等）。这个 handler 提供了几个方法来测试和调用此对象上的一个事件或回调。

想要调用一个事件，使用 `callbacks.call`。它需要参数：

- 需要调用的事件名称
- 事件定义的位置参数。这需要按照事件定义的顺序传递过去。

现在，我们已经为所有的对象添加了事件 `push`。这个事件当前永远不会被触发。我们应该添加一个 `push` 命令，使用对象的名称作为参数。如果此对象有效，他就能调用 `push` 事件。

```python
from commands.command import Command

class CmdPush(Command):

    """
    Push something.

    Usage:
        push <something>

    Push something where you are, like an elevator button.

    """

    key = "push"

    def func(self):
        """Called when pushing something."""
        if not self.args.strip():
            self.msg("Usage: push <something>")
            return

        # Search for this object
        obj = self.caller.search(self.args)
        if not obj:
            return

        self.msg("You push {}.".format(obj.get_display_name(self.caller)))

        # Call the "push" event of this object
        obj.callbacks.call("push", self.caller, obj)

```

这里，我们用以下参数调用 `callbacks.call`：

- `"push"`：调用事件名称
- `self.caller`：使用命令的对象
- `obj`：被 `push` 的对象。

在 **push** 回调中，我们可以使用 *character* 变量，和 *obj* 变量了。

## 观察效果

```
@create/drop rock
@desc rock = It's a single rock, apparently pretty heavy.  Perhaps you can try to push it though.
@call/add rock = push

```

我们执行 `push rock` 就会看到相关的输出了：

>You push box(#7).
>You push a rock... is... it... going... to... move?

## 添加新的 eventfuncs

事件函数，如 `deny()`，定义在 *contrib/events/eventfuncs.py*。我们可以自己新建一个文件，命名叫 *eventfuncs.py*在我们的 world 目录中。这其中的函数将会被添加为辅助函数。

我们同样可以在其他地方进行保存这个文件。不过这样的话我们就需要在 *server/conf/settings.py* 中设置一个全局变量，只定要搜索的路径：

```python
EVENTFUNCS_LOCATIONS = [
        "world.events.functions",
]

```

## 创建有参数的事件

当我们想要建立有参数的事件时，我们可以在 `_events` 中定义事件的时候，在代表值的元组中添加更多的参数。第三个参数必须包含一个在事件触发时会被调用来在回调中进行过滤的回调。经常使用的是两种参数：

- keyword参数：这个事件的回调将根据特定的 keyword 来过滤。
- Phrase 参数：事件的回调将会使用整个 phrase 来过滤，别使用其中的词来检查。`say` 命令使用的是 phrase 参数。

```python
from evennia.contrib.ingame_python.utils import register_events, phrase_event
# ...
@register_events
class SomeTypeclass:
    _events = {
        "say": (["speaker", "character", "message"], CHARACTER_SAY, phrase_event),
    }

```

当我们调用某个事件时，`obj.callbacks.call`，我们必须同时提供相应的参数：

```python
obj.callbacks.call(..., parameters="<put parameters here>")
```

不提供参数的话，系统俱乐部知道怎么样过滤回调列表。

# 禁止所有事件

```python
obj.callbacks.call(..., parameters="<put parameters here>")
```


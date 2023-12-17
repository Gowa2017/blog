---
title: Evennia中的Script
categories:
  - Mud
date: 2020-02-16 23:47:38
updated: 2020-02-16 23:47:38
tags: 
  - Python
  - Evennia
  - Mud
---

在 Evennia 中，[Script](https://github.com/evennia/evennia/wiki/Glossary#script) 也是一个很重要的对象。很多额外的系统都是用这个来实现的，比如一些战斗逻辑，或者是其他的事件逻辑，都是这样来实现的。

<!--more-->

# Script

[Script](https://github.com/evennia/evennia/wiki/Glossary#script) 是这样被简单的介绍的：
>当我们提到 *Scripts* 的时候，实际上我们说的是 **Script** TypeClass。Scripts 是 Evennia 中一个比较特别的东西——他们和 Objects 类似，但不存在于游戏内。它们既可以用作存储数据的自定义位置，也可以用作持久游戏系统的构建块。由于可以使用计时功能进行初始化，因此它们也可以用于长期持续计时（对于快速更新，其他类型的计时器可能会更好）。

我们可以看到有几个关键词 **存储数据的自定义位置，构建块，计时**。

# 详细介绍

[这里是详细的介绍](https://github.com/evennia/evennia/wiki/Scripts)。

Scripts 是角色内的 Objects 和角色外的一对兄弟。（一个在角色内，一个在角色外）。Sctips 很灵活所以也有些限制。

- 我们必须选择名字来定义它们。基于我们想用他来干什么，可能会出现的名字是：**OOBObjects, StorageContainers or TimerObjects.**。

Evennia 中， Scripts 可以用来做很多的事情：

- 可以附加到 Objects 身上，以不同的方式影响 Objects ——— 或者不依附于游戏内的任何实体而存在（ **Global Scripts**）。
- 可以用做 **定时器** 和 **计时器** ——任何会随着时间变化的东西。但是他们也可以完全不依赖于时间。注意，如果你想要的是只一个重复的对象方法调用，那么你应该考虑使用  [TickHandler](https://github.com/evennia/evennia/wiki/TickerHandler)。它的限制更多，但是它是专门做这项任务的。
- 他们可以用来描述状态变更。Script 是承载持久但独特的系统处理程序的绝佳平台。例如，一个 Script 可以用来作为跟踪回合战斗系统跟踪状态的基础。由于脚本还可以在计时器上运行，因此它们还可以定期更新自身以执行各种操作。
- 它们可以充当数据存储库，用于将游戏数据持久存储在数据库中。（感谢你拥有的 Attributes 能力）

Script 也是一个 TypeClass ，和其他 Evennia 中的实体一样进行操作：

```py
# create a new script 
new_script = evennia.create_script(key="myscript", typeclass=...)  

# search (this is always a list, also if there is only one match)
list_of_myscript = evennia.search_script("myscript")
```

## 新的 Script

Scripts 也被定义一个类，和其他的 TypeClass 一样。这个类有几个用来处理 时间相关的属性。不过他们都是 **可选的**——也可以没有任何时间组件（如充当数据库存储或持有持久性游戏系统）。

**evennia/typeclasses/scripts.py** 中能看到我们能做的事情。下面是一个例子：

```py
from evennia import DefaultScript

class MyScript(DefaultScript):

    def at_script_creation(self):
        self.key = "myscript"
        self.interval = 60  # 1 min repeat

    def at_repeat(self):
        # do stuff every minute 
```

当我们初始化我们自己的游戏在，**mygame/typeclasses/scripts.py** 已经继承了 **DefaultScript**。我们可以将它作为我们想要使用的 Scripts 的基类：我们可以调整 **Script** 的默认行为。例如：

```py
    # for example in mygame/typeclasses/scripts.py  
    # Script class is defined at the top of this module

    import random

    class Weather(Script): 
        """
        A timer script that displays weather info. Meant to 
        be attached to a room. 
          
        """
        def at_script_creation(self):
            self.key = "weather_script"
            self.desc = "Gives random weather messages."
            self.interval = 60 * 5  # every 5 minutes
            self.persistent = True  # will survive reload

        def at_repeat(self):
            "called every self.interval seconds."        
            rand = random.random()
            if rand < 0.5:
                weather = "A faint breeze is felt."
            elif rand < 0.7:
                weather = "Clouds sweep across the sky." 
            else:
                weather = "There is a light drizzle of rain."
            # send this message to everyone inside the object this
            # script is attached to (likely a room)
            self.obj.msg_contents(weather)
```

如果我们将此脚本，放到一个 room 上，那么它将会每隔 5 分钟就对房间内的所有人报告一些随机的天气信息。

想要激活它，只需要将它挂到 Room 的 script 处理器上。

```py
myroom.scripts.add(scripts.Weather)
```

> 由于变量 **TYPECLASS_PATHS** 指定了我们 typeclasses 的位置，所以我们不需要指定 脚本的 完整路径 *typeclasses.scripts.Weather*，而只需要指定 *scripts.Weather*。

同样，我们可以使用  `evennia.create_script` 函数来建立：

```py
    from evennia import create_script
    create_script('typeclasses.weather.Weather', obj=myroom)
```
> 如果我们给了 `create_script()` 一个 **关键词** 参数，那么它将会覆盖 typeclass 类中的 key 。例如，下面就是一个每 10分钟运行一次的 天气脚本的 实例：

```sh
    create_script('typeclasses.weather.Weather', obj=myroom, 
                   persistent=False, interval=10*60)
```

甚至，还可以使用游戏内的命令来构建：

```
@script here = typeclasses.scripts.Weather 
```

## Scripts上的属性和函数

一个脚本拥有所有 typeclass 的属性，如 `db, ndb`。设置 `key` 将会是非常有用的事情，方便管理（例如通过名字删除）。这些参数通常是在 typeclass 中设置，但也可能通过在 `create_script()` 中指定来设置。

- desc 描述
- interval 执行间隔。**interval == 0** 的话将不会有时间组件，不会重复，但会永远存在。这在进行存储或者很多非时间相关的游戏系统的时候很常用。
- start_delay(bool) 在到达第一个 interval 的时候是否等待。
- repeats 重复次数。`repeats <= 0`，会无限重复。注意，每次时间间隔到达都会增加这个值。所以 `start_delay=False, repeats=1` 会执行后立刻就退出。
- persistent 在服务器重启和关闭期间这个脚本是否存在。

特殊的属性：

- obj 表示此脚本附加到的对象。我们不需要手动设置这个属性。我们用 `myobj.scripts.add(myscriptpath)` 或 `myobj`  作为`utils.create.create_script` 的参数时，会自动进行设置。

了解钩子函数也很重要，通常我们的自定义就在与重写这些自定义函数。`src/scripts/scripts.py` 有更长更详细的描述。

- at_script_creation() 通常是脚本设置属性的地方，用来控制脚本如何运行。只执行一次。
- is_valid() 判断脚本是否应该继续执行与否。`obj.scripts.validate()` 调用的时候会被执行，我们可以手动运行。在 Evennia 的一些场景下也会调用，如 `reload`。这在用做状态管理器的时候很有用。如果方法返回 `False`，这个脚本会停止并立即清除。
- at_start() 当脚本开始运行或离开暂停时。对于持久化的脚本，服务器每次启动最少执行一次。这个方法总是被离开调用，及时`start_delay` 是 `False`。
- at_repeat() 每 `interval` 执行一次，或者永不执行。其在启动的时候会立刻执行，除非 `start_delay == True`。
- at_stop() 脚本停止时执行。用来进行自定义清理的时候用。
- at_server_reload() 热启动的时候执行。此时可用来进行存储非**持久化**但我们又想保留的数据。
- at_server_shutdown() 系统 reset 或者 shutdown 的时候执行。

运行方法（通常是引擎自动调用，但我们也可以手动调用）：

- start() 启动脚本。我们将脚本添加到对象的时候就会被执行。`at_start()` 随后。
- stop() 停止，并删除。从一个处理器移除脚本会自动停止。`at_stop()`  随后。
- pause() 暂停脚本。所有的属性会被保留，定时器可以恢复。这在服务器 reload 和不会导致 `at_stop()` 钩子被调用时执行。这是挂起脚本，而不会改变状态。
- unpasue() 重启脚本。`at_start()` 会被调用来回收其内部的状态。定时器恢复。在服务器 reload 会重启所有的暂停脚本。
- force_repeat() 强制定时器执行一步。定时器会重置，`at_repeat()` 被调用。这也会增加计数器。
- `time_until_next_repeat`() 到下一次定时器执行的剩余时间。None if repeats == 0.
- remaining_repeats() 剩余执行次数。
- reset_callcount(value=0) 重置已执行完的次数。repeats > 0 才会有用。
- restart(interval=None, repeats=None, start_delay=None) 以不同的设置重新启动脚本。会调用 `at_stop()` 函数，然后执行 `at_start()`，没有改变的参数则保持状态。


## 全局脚本

一个脚本不需要一定连接到一个游戏内对象。我们不要给脚本添加附加对象参数就行了。

```py
     # adding a global script
     from evennia import create_script
     create_script("typeclasses.globals.MyGlobalEconomy", 
                    key="economy", persistent=True, obj=None)
```

然后，我们可以通过 key 或者 标识符来搜索 `evennia.search_script()`获得这个脚本。游戏中的 scripts 命令会显示所有的脚本。

Evennia 提供了一个方便的容器 **GLOBAL_SCRIPTS** 来访问全局脚本。如果我们知道脚本的名字：

```py
from evennia import GLOBAL_SCRIPTS

my_script = GLOBAL_SCRIPTS.my_script
# needed if there are spaces in name or name determined on the fly
another_script = GLOBAL_SCRIPTS.get("another script")
# get all global scripts (this returns a Queryset)
all_scripts = GLOBAL_SCRIPTS.all()
# you can operate directly on the script
GLOBAL_SCRIPTS.weather.db.current_weather = "Cloudy"
```

> 注意：GLOBAL_SCRIPTS 使用 `key` 来索引全局脚本。如果我们两个脚本有同样的 key ，返回的结果是不确定的，依赖于数据库。最好的办法就是不要用同名的全局脚本，或者是使用 evennia.search_script 来搜索。

有两个方法来让脚本成为 GLOBAL_SCRIPTS 的属性。一个是手动使用  `create_script()` 来手动建立一个全局脚本。通常我们需要是在服务器启动的时候自动执行这个事情，那么我们可以这样设置：

```py
GLOBAL_SCRIPTS = {
    "my_script": {               
        "typeclass": "scripts.Weather",
        "repeats": -1,
        "interval": 50,
        "desc": "Weather script"
        "persistent": True
    },
    "storagescript": {
        "typeclass": "scripts.Storage",
        "persistent": True
    }
}
```

除了其中的 `my_script, storagescript` 参数，其他所有参数都是可选的。如果没指定 typeclass，那么就会用 `settings.BASE_SCRIPT_TYPECLASS` 来建立。脚本与时间相关才需要时间参数。

Evennia 会自动的使用  `settings.GLOBAL_SCRIPTS` 来建立和启动脚本（服务器启动时，除非 `key` 键的脚本已经存在）。在配置阅读和新脚本可用前我们必须 reload 服务器。

> 不要让脚本出现严重错误，不然你会看到错误日志，同时脚本会回退到 DefaultScript

一个脚本如下定义的时候，会保证在访问的时候是已存在的：

```py
from evennia import GLOBAL_SCRIPTS
# first stop the script 
GLOBAL_SCRIPTS.storagescript.stop()
# running the `scripts` command now will show no storagescript
# but below now it's recreated again! 
storage = GLOBAL_SCRIPTS.storagescript
```
也就是说，如果脚本被删除了，我们下一次使用 `settings. GLOBAL_SCRIPTS ` 访问的时候，会使用配置文件的信息来重建脚本。

> 如果我们的目的是为了存储持久化数据。，我们应该将 persistent=True。在 `settings.GLOBAL_SCRIPTS` 或者 在 typeclass 中设置。

# Muddery 中的战斗系统

[抓了一下源码有点奇怪，muddery 是对每个角色，在使用攻击命令的时候，都挂上一个战斗处理器](https://github.com/muddery/muddery/blob/master/muddery/typeclasses/character.py#L693)：

```py
        # create a new combat handler
        chandler = create_script(settings.NORMAL_COMBAT_HANDLER)
                        
        # set combat team and desc
        chandler.set_combat({1: [target], 2: [self]}, desc, 0)
```

为什么不把逻辑单独放呢？




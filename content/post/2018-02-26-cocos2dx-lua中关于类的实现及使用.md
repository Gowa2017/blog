---
title: Cocos2d-X-lua中关于类的实现及使用
categories:
  - Cocos2d-X
date: 2018-02-26 14:53:59
updated: 2018-02-26 14:53:59
tags: 
  - Cocos2d-X
  - Lua
---
Lua中，没有什么其他的数据对象，只有表。但是其提供的元表和元方法，让我们的程序有了更多的可能。另外，我们必须明白一点的就是，cocos2dx框架中，开启了一个Lua State，我们所有的脚本，逻辑都是在这里面执行，然后这个里面会调用一些 cocos2dx 导出给 lua 使用的接口，最终还是通过 c 代码来完成工作的。

<!--more-->

# main.lua中解析

在框架启动完毕后，会执行 `    engine->executeScriptFile("src/main.lua");` lua入口文件。根据版本的不同，可能是 **src/main.lua**，也有可能直接是 **main.lua** 但这没有什么影响。

具体的代码请查看 `framework/runtime-src/Classes/AppDelegate.cpp`。

一开始，cocos2dx就已经把其提供的接口函数注册到了Lua State中，我们已经可以直接使用一些了，当然，我们也可以把它提供的接口，再次用Lua进行封装，Quick干的就是这样的事情。

我们关注的其实，只是**main.lua**中的一行：

```lua
require("app.MyApp").new():run()
```

其只是加载了 **app.MyApp** 文件，然后执行其中的 `new()` 方法后返回一个新 **AppBase** 对象，再执行对象中的 `run()` 方法。

**MyApp.lua** 是 **AppBase.lua**的一个实例，从其文件中我们可以看得出来：

```lua
-- app/MyApp.lua
local AppBase = require("framework.AppBase")
local MyApp = class("MyApp", AppBase)
```

其以 AppBase作为基类（父类），建立了一个对象**MyApp**。我们可以以如下代码来打印出 **MyApp**的内容及其元表的内容。

```lua
local m = require("app.Myapp")
for k, v in pairs(m) do
        print(k, v)
end

print("-------------")
for k, v in pairs(getmetatable(m)) do
        print(k, v)
end
```

输出结果是：

```
[LUA-print] run	function: 0x07c45220
[LUA-print] __ctype	2
[LUA-print] new	function: 0x07c44360
[LUA-print] __cname	MyApp
[LUA-print] super	table: 0x07c44cf8
[LUA-print] ctor	function: 0x07c451e0
[LUA-print] __index	table: 0x07c44650
[LUA-print] -------------
[LUA-print] __index	table: 0x07c44cf8
```

其中  **spuer** 表明其父类是一个表，**__cname** 表示其类名 *MyApp* ，**__index**  是一个表，这里暂时不讨论这个，放在这里，其表示它也可以作为其他类的元表（父类）而已。

最后一行是其 元表的 输出，其元表中只有一个 **__index** 方法，那么所有 *MyApp* 内不存在的方法都会从 **__index** 对应的表中获取。

这里，我们加载了 **app.MyApp.lua** 模块，返回了类 **AppBase.lua** 的一个子类 **MyApp**。

# new()及 class的实现
在 **app.MyApp.lua**中，我们发现，其实并没有定义 **new, super, ctype, __index** 这几个字段。其实由 *class()* 函数进行设置的。

在 文件 `src/framework/functions.lua`，定义了 `class()` 函数：

```lua
function class(classname, super)
    local superType = type(super)
    local cls

	-- 父类 只能是一个函数 或者 一个表
    if superType ~= "function" and superType ~= "table" then
        superType = nil
        super = nil
    end

    if superType == "function" or (super and super.__ctype == 1) then
        -- inherited from native C++ Object
        cls = {}

        if superType == "table" then
            -- copy fields from super
            for k,v in pairs(super) do cls[k] = v end
            cls.__create = super.__create
            cls.super    = super
        else
            cls.__create = super
            cls.ctor = function() end
        end

        cls.__cname = classname
        cls.__ctype = 1

        function cls.new(...)
            local instance = cls.__create(...)
            -- copy fields from class to native object
            for k,v in pairs(cls) do instance[k] = v end
            instance.class = cls
            instance:ctor(...)
            return instance
        end

    else
        -- inherited from Lua Object
        if super then
            cls = {}
            setmetatable(cls, {__index = super})
            cls.super = super
        else
            cls = {ctor = function() end}
        end

        cls.__cname = classname
        cls.__ctype = 2 -- lua
        cls.__index = cls

        function cls.new(...)
            local instance = setmetatable({}, cls)
            instance.class = cls
            instance:ctor(...)
            return instance
        end
    end
    return cls
end
```

其通过一个 *类名，父类* 作为参数，然后返回一个新的类。  

这里，我们首先假设都已经知道，对于元表中的**__index**，其值（元方法）可以是一个函数（当找不到对应的键值是会以 *表，键* 为参数进行调用后返回），或一个表（找不到对应键值时以 *t[k]* 进行返回）。

父类可能有两种情况，我们先来看简单的一种。


## super是lua表

这是比较简单的一种情况，代码体现在：

```lua
        -- inherited from Lua Object
        if super then
            cls = {}
            setmetatable(cls, {__index = super})
            cls.super = super
        else
            cls = {ctor = function() end}
        end

        cls.__cname = classname
        cls.__ctype = 2 -- lua
        cls.__index = cls

        function cls.new(...)
            local instance = setmetatable({}, cls)
            instance.class = cls
            instance:ctor(...)
            return instance
        end
    end
```

我们需要关注的是，`class()` 自动为每个类建立了一个 `new()` 方法：其会返回以接收消息类的实例，并将 对应参数传递给  `ctor()` 方法。

## super是一个函数

当 *spuer* 是一个函数，或者 super类型是 C 类的时候，会麻烦一些：

```lua
    if superType == "function" or (super and super.__ctype == 1) then
        -- inherited from native C++ Object
        cls = {}

        if superType == "table" then
            -- copy fields from super
            for k,v in pairs(super) do cls[k] = v end
            cls.__create = super.__create
            cls.super    = super
        else
            cls.__create = super
            cls.ctor = function() end
        end

        cls.__cname = classname
        cls.__ctype = 1

        function cls.new(...)
            local instance = cls.__create(...)
            -- copy fields from class to native object
            for k,v in pairs(cls) do instance[k] = v end
            instance.class = cls
            instance:ctor(...)
            return instance
        end
```

如果 super是一个lua 函数，其会设置类的 *__create*字段为 super函数，构造的 `new()`方法就会调用这个函数。而当 super是一个是C类时，会调用这个C类构造函数来返回实例。

## lua中的C类
前面提到，当一个类的父类是一个表，但表中的**__ctype** 是 1时，这个类是一个C类。其本质，也是Lua中的一张表。

关于在Lua进行模块的注册流程，请关注一下另外一篇文章[cocos2dx-lua的启动流程.lua](https://gowa2017.github.io/cocos/cocos2dx-lua%E7%9A%84%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B.lua.html#lua-module-register-L)。

现在，我们来关注一下类的导出吧，以Scene为例：

```c
// cocos/scripting/lua-bindings/auto/lua_cocos2dx_auto.cpp
int lua_register_cocos2dx_Scene(lua_State* tolua_S)
{	 // 注册一个用户数据类型 cc.Scene 到State中
    tolua_usertype(tolua_S,"cc.Scene");
    
    // 映射C类 cc.Scene 到Lua 类 Scene ，父类为 cc.NodeSceneLuatolua_cclass(tolua_S,"Scene","cc.Scene","cc.Node",nullptr);
	// 注册模块 Scene
    tolua_beginmodule(tolua_S,"Scene");
    // 注册模块（类）的函数    tolua_function(tolua_S,"render",lua_cocos2dx_Scene_render);
        tolua_function(tolua_S,"createWithSize", lua_cocos2dx_Scene_createWithSize);
        tolua_function(tolua_S,"create", lua_cocos2dx_Scene_create);
    tolua_endmodule(tolua_S);
    std::string typeName = typeid(cocos2d::Scene).name();
    g_luaType[typeName] = "cc.Scene";
    g_typeCast["Scene"] = "cc.Scene";
    return 1;
}
```

我们来简要的说一下这个过程：

*  `tolua_usertype(L, "cc.Scene")` 这个调用，会在Lua State的*LUA_REGISTRYINDEX* 索引处的注册表  *registry* 中建立两项：

```lua
 registry["cc.Scene"] = { "__name" = "cc.Scene"}
 registry["const cc.Scene"] = { "__name" == "const cc.Scene"}
```
* `tolua_beginmodule(tolua_S,"Scene");` 注册模块 *Scene*
* 注册*Scene*模块的方法。

因此，调用 `.new()` 方法会创建一个 MyApp 类（AppBase类的子类）的实例，之后，再调用 实例 的`run()`方法，进入了主场景。

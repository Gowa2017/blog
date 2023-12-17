---
title: Cocos2d-X-lua的启动流程.lua
categories:
  - Cocos2d-X
date: 2018-02-23 20:35:48
updated: 2018-02-23 20:35:48
tags: 
  - Cocos2d-X
  - Lua
---
可能以前用的项目就是Lua，所以比较喜欢Lua。服务端也用skynet框架，都用Lua，能统一的话是最好的了。完全是个人爱好。但是有必要看一下，我对客户端是最不熟的了，图形这一块。

# 启动
新建一个Lua项目后:

	cocos new -l lua -p com.example.me -d game

进入目录：

	cd game/MyLuaGame/frameworks/rutime-src/Classes

**AppDelegate.cpp**是我们需要关注的文件。他干了一系列的事情：

```cpp
bool AppDelegate::applicationDidFinishLaunching()
{
    // set default FPS
    Director::getInstance()->setAnimationInterval(1.0 / 60.0f);

    // register lua module
    // 注册 lua 模块
    auto engine = LuaEngine::getInstance();
    ScriptEngineManager::getInstance()->setScriptEngine(engine);
    lua_State* L = engine->getLuaStack()->getLuaState();
    lua_module_register(L);

    register_all_packages();

    LuaStack* stack = engine->getLuaStack();
    stack->setXXTEAKeyAndSign("2dxLua", strlen("2dxLua"), "XXTEA", strlen("XXTEA"));

    //register custom function
    //LuaStack* stack = engine->getLuaStack();
    //register_custom_function(stack->getLuaState());

	// 添加的这两个路径，我们其实都不用手动在项目内添加了
#if CC_64BITS
    FileUtils::getInstance()->addSearchPath("src/64bit");
#endif
    FileUtils::getInstance()->addSearchPath("src");
    FileUtils::getInstance()->addSearchPath("res");
    if (engine->executeScriptFile("main.lua"))
    {
        return false;
    }

    return true;
}
```

这个类其实就干了两个事情：
* 开个Lua State 把模块都注册进去
* 引擎执行 **main.lua**

另外，其实以前那种写法:

```lua
cc.FileUtils:getInstance():addSearchPath("src/")
cc.FileUtils:getInstance():addSearchPath("res/")
```
这样的代码已经不需要了。程序已经自动注册了这两个路径了。程序会自动在这两个地方寻找资源。

# 模块及函数的注册到State

## lua\_module_register(L);

这玩意会在我们的Lua State内注册一系列的模块。我们取个例子来看一下，比如第一个模块。

```cpp
// frameworks/cocos2d-x/cocos/scripting/lua-bindings/manual/lua_module_register.cpp

int lua_module_register(lua_State* L)
{
    // Don't change the module register order unless you know what your are doing
    register_cocosdenshion_module(L);
    register_network_module(L);
    register_cocosbuilder_module(L);
    register_cocostudio_module(L);
    register_ui_module(L);
    register_extension_module(L);
    register_spine_module(L);
    register_cocos3d_module(L);
    register_audioengine_module(L);
#if CC_USE_3D_PHYSICS && CC_ENABLE_BULLET_INTEGRATION
    register_physics3d_module(L);
#endif
#if CC_USE_NAVMESH
    register_navmesh_module(L);
#endif
    return 1;
}
```

## register\_cocosdenshion_module

这个函数干的活比较简单，就是获取一下Lua的全局环境，然后把所有的模块内容注册进去。


```cpp
//scripting/lua-bindings/manual/cocosdenshion/lua_cocos2dx_cocosdenshion_manual.cpp

int  register_cocosdenshion_module(lua_State* L)
{
    lua_getglobal(L, "_G");
    if (lua_istable(L,-1))//stack:...,_G,
    {
        register_all_cocos2dx_cocosdenshion(L);
    }
    lua_pop(L, 1);
    return 1;
}
```
真正干活的，还是下面一个函数。
## register\_all\_cocos2dx\_cocosdenshion

tolua 没有用过，但是一下他们的代码就知道了。很明显，是用了`open, module, beginmodule, endmodule`来干活的。
```cpp
TOLUA_API int register_all_cocos2dx_cocosdenshion(lua_State* tolua_S)
{
        tolua_open(tolua_S);

        tolua_module(tolua_S,"cc",0);
        tolua_beginmodule(tolua_S,"cc");

        lua_register_cocos2dx_cocosdenshion_SimpleAudioEngine(tolua_S);

        tolua_endmodule(tolua_S);
        return 1;
}
```

### 小插曲
我们先来看一下，Lua关于注册表的概念。
> Lua提供了应该注册表，这个表可以被任何C代码用来存储任何的Lua值。注册表总是位于伪索引 **LUA_REISTRYINDEX**上（并不在Lua State的真正的栈上）。任何C库都可以使用这个表来存储数据，但必须谨慎选择键，以避免出现冲突。典型滴，用一包含你库名的字符串作为键，或者是一个light userdata（C对象的指针），或者任何被自己代码创建的Lua对象。和变量名字一样，以一个下划线后跟大写字母的字符串是保留的。
> 
> 整型的键用来做索引算法（**luaL_ref**）和一些预定义的值。因此，整型键不应该用作其他目录。
> 
> 当建一个新的Lua State时，这个注册您就有一些预定义的值。这些预定义的值以**lua.h**定义的常量整数作为键，下面的两个是定义好的：
> 
> **LUA\_RIDX_MAINTHREAD** 注册内保存的State的主线程（与State一起建立的那个线程）  
> **LUA\_RIDX_GLOBALS** 注册表内的这个索引保存了全局环境。

我们可以假设这个注册表刚开始的时候是这样的：

```lua
t_reg = {  [LUA_RIDX_MAINTHREAD] = value,
			 [LUA_RIDX_GLOBALS] = _G,
			 }
```
### lua\_settable lua\_rawset

```c
luasettable(L, index)
luarawset(L, index)
```

这两个函数和代码 `t[k] = v`是一样的，*t* 就是位于 *index* 处的值，*v* 是栈的顶部的值， *k* 是在栈顶部下的值。

这两个函数会将 键 和 值都弹出。

不同的是，`luarawset` 并不会触发事件 **__newindex** 的元方法。

### lua\_gettable lua\_rawget

```c
lua_gettable(L, index)
lua_rawget(L, index)
```

把 *t[k]*的值压入栈，*t* 是 *index* 指定的值，*k* 是栈顶部的值。

这两个函数都会把键弹出，然后把结果值压入那个位置。`lua_rawget`不会触发 **__index** 事件的元方法。

返回值是是结果值的类型。

## tolua

### tolua_open

首先调用的是 `tolua_open` 函数：

```cpp
// frameworks/cocos2d-x/external/lua/tolua/tolua_map.c

TOLUA_API void tolua_open (lua_State* L)
{
    int top = lua_gettop(L);
    // 检查 tolua 是否打开。用 t["tolua_opened"] = true | false 来判断
    lua_pushstring(L,"tolua_opened");
    lua_rawget(L,LUA_REGISTRYINDEX);
    if (!lua_isboolean(L,-1))
    {
    	// 如果没打开就打开它
    	// t_reg["tolua_opened"] = 1
        lua_pushstring(L,"tolua_opened");
        lua_pushboolean(L,1);
        lua_rawset(L,LUA_REGISTRYINDEX);

        // create value root table
        // 建立根表
        // t_reg["tolua_value_root"] = {}
        lua_pushstring(L, TOLUA_VALUE_ROOT);
        lua_newtable(L);
        lua_rawset(L, LUA_REGISTRYINDEX);

#ifndef LUA_VERSION_NUM /* only prior to lua 5.1 */
        /* create peer object table */
        lua_pushstring(L, "tolua_peers");
        lua_newtable(L);
        /* make weak key metatable for peers indexed by userdata object */
        lua_newtable(L);
        lua_pushliteral(L, "__mode");
        lua_pushliteral(L, "k");
        lua_rawset(L, -3);                /* stack: string peers mt */
        lua_setmetatable(L, -2);   /* stack: string peers */
        lua_rawset(L,LUA_REGISTRYINDEX);
#endif

        /* create object ptr -> udata mapping table */
        lua_pushstring(L,"tolua_ubox");
        lua_newtable(L);
        /* make weak value metatable for ubox table to allow userdata to be
           garbage-collected */
        lua_newtable(L);
        lua_pushliteral(L, "__mode");
        lua_pushliteral(L, "v");
        lua_rawset(L, -3);               /* stack: string ubox mt */
        lua_setmetatable(L, -2);  /* stack: string ubox */
        lua_rawset(L,LUA_REGISTRYINDEX);

//        /* create object ptr -> class type mapping table */
//        lua_pushstring(L, "tolua_ptr2type");
//        lua_newtable(L);
//        lua_rawset(L, LUA_REGISTRYINDEX);

        lua_pushstring(L,"tolua_super");
        lua_newtable(L);
        lua_rawset(L,LUA_REGISTRYINDEX);
        lua_pushstring(L,"tolua_gc");
        lua_newtable(L);
        lua_rawset(L,LUA_REGISTRYINDEX);

        /* create gc_event closure */
        lua_pushstring(L, "tolua_gc_event");
        lua_pushstring(L, "tolua_gc");
        lua_rawget(L, LUA_REGISTRYINDEX);
        lua_pushstring(L, "tolua_super");
        lua_rawget(L, LUA_REGISTRYINDEX);
        lua_pushcclosure(L, class_gc_event, 2);
        lua_rawset(L, LUA_REGISTRYINDEX);

        tolua_newmetatable(L,"tolua_commonclass");

        tolua_module(L,NULL,0);
        tolua_beginmodule(L,NULL);
        tolua_module(L,"tolua",0);
        tolua_beginmodule(L,"tolua");
        tolua_function(L,"type",tolua_bnd_type);
        tolua_function(L,"takeownership",tolua_bnd_takeownership);
        tolua_function(L,"releaseownership",tolua_bnd_releaseownership);
        tolua_function(L,"cast",tolua_bnd_cast);
        tolua_function(L,"isnull",tolua_bnd_isnulluserdata);
        tolua_function(L,"inherit", tolua_bnd_inherit);
#ifdef LUA_VERSION_NUM /* lua 5.1 */
        tolua_function(L, "setpeer", tolua_bnd_setpeer);
        tolua_function(L, "getpeer", tolua_bnd_getpeer);
#endif
        tolua_function(L,"getcfunction", tolua_bnd_getcfunction);
        tolua_function(L,"iskindof", tolua_bnd_iskindof);

        tolua_endmodule(L);
        tolua_endmodule(L);
    }
    lua_settop(L,top);
}                
```

这些都不用多说了，反正就是在 注册表内，添加两很多元素。

### tolua_module
这个函数会创建一个模块。

```cpp
// frameworks/cocos2d-x/external/lua/tolua/tolua_map.c


TOLUA_API void tolua_module (lua_State* L, const char* name, int hasvar)
{
    if (name)
    {
        /* tolua module */
        lua_pushstring(L,name);
        lua_rawget(L,-2);
        if (!lua_istable(L,-1))  /* check if module already exists */
        {
            lua_pop(L,1);
            lua_newtable(L);
            lua_pushstring(L,name);
            lua_pushvalue(L,-2);
            lua_rawset(L,-4);       /* assing module into module */
        }
    }
    else
    {
        /* global table */
        lua_pushvalue(L,LUA_GLOBALSINDEX);
    }
    if (hasvar)
    {
        if (!tolua_ismodulemetatable(L))  /* check if it already has a module metatable */
        {
            /* create metatable to get/set C/C++ variable */
            lua_newtable(L);
            tolua_moduleevents(L);
            if (lua_getmetatable(L,-2))
                lua_setmetatable(L,-2);  /* set old metatable as metatable of metatable */
            lua_setmetatable(L,-2);
        }
    }
    lua_pop(L,1);               /* pop module */
}
```
这个函数么，就是在注册表内，建立一个新模块。*havar* 表示是不是这个模块有元表。

## 注册完成
在执行了函数 `register_all_cocos2dx_cocosdenshion`后，我们的注册表可能看起来是这样的：

```lua
t_reg = {
...  -- 预定义的值
["cc"] = {
			["getInstance"] = lua_cocos2dx_cocosdenshion_SimpleAudioEngi, -- 这就是导出来给我们在Lua中用的
			.....
			}
}
```
# main.lua的执行
`engine->executeScriptFile("main.lua")`就开始执行 *main.lua*了。

之后，其实就跟普通的Lua脚本执行没有什么差别了，通过在State中调用C函数，来操作整个引擎了。

而对于，导出的函数，继续以 Lua 进行封装后以模块的方式调用，其实也没有什么特别的。

这就是 C -> Lua State -> lua script -> Cfunction -> Cobj 流程类似了。


不信你看一下，项目中 src 目录下的 cocos中。全是这样干的。

在我们所有的项目中，都有：

```lua
require "config"
require "cocos.init"
```

其实就是加载我们的配置文件，然后再加载 cocos Lua封装的初始化文件。打开 *cocos/init.lua* 就可以看到，其一一个个个 require 语句，用来加载封装成Lua的各个模块。

我们关注有一句：

```lua
-- src/cocos/init.lua
if CC_USE_FRAMEWORK then
    require "cocos.framework.init"
end
```

其实这一段，是用了更高层的封装，更加方便使用，其实应该就是 Quick 干的事情。

如果我们在我们的 *config.lua* 中定义了这个变量，那么，就可以使用那些封装了。




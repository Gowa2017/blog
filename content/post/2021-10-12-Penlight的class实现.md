---
title: Penlight的class实现
categories:
  - Lua
date: 2021-10-12 20:20:39
updated: 2021-10-12 20:20:39
tags: 
  - Lua
---

我想要一个比较好的 class 实现，但自己实现起来确实有点拙劣，但我却看到了 [Penlight](https://github.com/lunarmodules/Penlight) 这么一个项目，所以我需要看一下他的实现是怎么样的。



<!--more-->

# 使用方式

一般来说，它有两种方式：



```lua
local class  = require("pl.class")

B = class(A) -- 这种方式必须传入一个表，分则会报错。
class.B(A) --这种方式在当前环境建立内先设置一个名字 B，然后将构造后的类进行命名
```



# 类上的函数

- _init (...)	初始化函数
- instance:is_a (some_class)	判断实例是否衍生自 *some_class* 。
- some_class:class_of (some_instance)	判断实例是否衍生自 *some_class* 。
- some_class:cast (some_instance)	强制转换成其他类 
- class (base, c_arg, c)	建立一个新类，从给定的基类衍生。



# class

class 是一个表，不过在它的元表上设置了 `__index, __call` 方法，所以可以用函数的形式调用。

```lua
local class
class = setmetatable({},{
    __call = function(fun,...)
        return _class(...) -- 第一种方式建立类
    end,
    __index = function(tbl,key) -- 第二种方式
        if key == 'class' then
            io.stderr:write('require("pl.class").class is deprecated. Use require("pl.class")\n')
            return class
        end
        compat = compat or require 'pl.compat'
        local env = compat.getfenv(2)
        return function(...)
            local c = _class(...)
            c._name = key
            rawset(env,key,c) -- 当前的环境内设置一个 key = c
            return c
        end
    end
})
```

# 类的建立

上面，我们看到，实际上是 `__class` 来给我们构造类。我们来逐行查看

```lua
---@param base? table|nil 可选的基类
---@param c_arg 给类构造器的可选参数
---@param c table 如果传递，那么就将此表变成一个类
local function _class(base,c_arg,c)
    -- 类 `c` 将会是所有它的对象的元表,
    -- 这些对象会在元表内查询方法。
    local mt = {}   -- 一个用来让这个类支持 __call 和 __handler 的元表 
    -- 也可以传递一个只有方法的表 来构造类 没有元表，所以叫 plain table
    local plain = type(base) == 'table' and not getmetatable(base)
    if plain then -- 基类是一个 Plain 表
        c = base  -- 这个表就作为类
        base = c._base -- 传递的这个表，可以在 _base 指定基类
    else -- 基类不是一个 plain 表，构造新类
        c = c or {}
    end

    if type(base) == 'table' then -- 无论是在 plain 表指定的 _base 基类，还是直接传递的基类
        -- 我们的新类是基类的浅拷贝!
        -- 但是要小心，不要把新类中的方法给覆盖掉了
        tupdate(c,base,plain)
        c._base = base
        -- 如果存在，那就继承  'not found' handler
        if rawget(c,'_handler') then mt.__index = c._handler end
    elseif base ~= nil then -- 基类只能是一个表或者是 nil
        error("must derive from a table type",3)
    end

    c.__index = c
    setmetatable(c,mt)
    if not plain then -- 如果不是 plain 表作为基类
        if base and rawget(base,'_init') then c._parent_with_init = base end -- 基类存在，且基类有 _init 方法，那么就继承下来  ，For superFor super and inherited init
        c._init = nil
    end

    if base and rawget(base,'_class_init') then --基类有初始化方法，则调用
        base._class_init(c,c_arg)
    end

    -- 导出一个 ctor 方法，可以通过 <classname>(<args>) 形式调用。
    mt.__call = function(class_tbl,...)
        local obj
        if rawget(c,'_create') then obj = c._create(...) end -- 有 _create 方法就调用来构造对象
        if not obj then obj = {} end -- 默认构造一个空表
        setmetatable(obj,c)

        if rawget(c,'_init') or rawget(c,'_parent_with_init') then -- 存在初始化函数
            local res = call_ctor(c,obj,...) -- 调用初始化函数
            if res then -- 如果 ctor 函数返回了值，这个值就是作为返回对象
                obj = res
                setmetatable(obj,c)
            end
        end

        if base and rawget(base,'_post_init') then -- 初始化后调用
            base._post_init(obj)
        end

        return obj
    end
    -- Call Class.catch to set a handler for methods/properties not found in the class!
    c.catch = function(self, handler)
        if type(self) == "function" then
            -- called using . instead of :
            handler = self
        end
        c._handler = handler
        mt.__index = handler
    end
    c.is_a = is_a
    c.class_of = class_of
    c.cast = cast
    c._class = c

    if not rawget(c,'__tostring') then
        c.__tostring = _class_tostring
    end

    return c
end
```

# 对象的构造过程

对于一个类，

1. _create 先调用，构造对象
2. _init 方法 或父类有 _init 方法，那么进行调用，若这步返回的了值，则会替换第一步返回的对象。这一步，会先设置好 super 方法，然后才执行 _init，执行完毕，即删掉 super 方法。
3. 调用基类的 _post_init 

第二步是最为复杂的：

```lua
local function call_ctor (c,obj,...)
    local init = rawget(c,'_init')
    local parent_with_init = rawget(c,'_parent_with_init')

    if parent_with_init then -- 父类有初始化函数
        if not init then -- 子类没有
            init = rawget(parent_with_init, '_init') -- 继承父类
            parent_with_init = rawget(parent_with_init, '_parent_with_init') -- 父类的父类还有没有初始化函数
        end
        if parent_with_init then -- super() 函数指向 _init 的所属类的上一级。
            rawset(obj,'super',function(obj,...)
                call_ctor(parent_with_init,obj,...)
            end)
        end
    else
        -- 没有这句，调用不存在的 super() 有的时候会死循环和栈溢出 
        rawset(obj,'super',nil)
    end

    local res = init(obj,...)
    if parent_with_init then -- 调用完毕，就把 super 干掉 
        rawset(obj,'super',nil)
    end

    return res
end

```



# 推荐调用父类的形式

在初始化之后，我们如果想要调用父类的相同方法，推荐的形式是这样的：



```lua
local A = class.A()
function A:test() end
local  B = class.B(A)
fuction B:test() 
A.test(B)
end
```




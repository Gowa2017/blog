---
title: Skynet中服务间的数据共享方案-Sharetable
categories:
  - Skynet
date: 2021-10-23 21:44:21
updated: 2021-10-23 21:44:21
tags:
  - Skynet
  - Lua
---

Skynet 的历史上，有多种共享数据的方案，但最终都被云风一一否决，只留下了 sharetable 一个，但我们从其历史，就可以看出抉择与权限。

<!--more-->

历史上，有 ShareData, DataSheet， Stm, ShareMap(Stm的简单封装），这些方案都是基于共享内存工作，具体见云风的博客： [不同虚拟机间共享不变的 Table](https://blog.codingnow.com/2019/04/share_table.html)

>过去的方案用的思路都是把数据表放在 C 对象中 。Lua 中建立一个 proxy 对象去访问它。C 对象可以跨虚拟机共享，proxy 对象则在不同的虚拟机中各创建一份。

>这种方式比较符合 Lua 正统，但缺点有二：

>1.如果数据表结构比较复杂，那么每一层的子表都需要创建 proxy 对象。如果访问的数据较多，proxy 对象的总量还是很大的，依旧有很大的内存开销。

>2.通过 C function 访问 C 对象，比直接访问 table ，开销要大的多。而且字符串在 Lua 虚拟机和 C 对象间传递，也有不小的开销。（这也是用 DataSheet 取代 ShareData 的主要动机）


而 Sharetable 则是通过修改了 Lua的虚拟间，直接在 Lua 虚拟间共享一个 Lua 的表结构。

这里我们主要来探究， Sharetable 是一个什么东西，承担什么职责，然后，在 Skynet 的 Lua 层面的使用又是怎么样的。

# Sharetable C 层

它的代码位于 [https://github.com/cloudwu/skynet/blob/c008476417b20fd06202cc3d196dc0e076cb1c3a/lualib-src/lua-sharetable.c](https://github.com/cloudwu/skynet/blob/c008476417b20fd06202cc3d196dc0e076cb1c3a/lualib-src/lua-sharetable.c)

我们看到，这个库实际上并没有太多复杂的 API

1. `matrix` 开一个虚拟机，加载我们指定的文件（返回一个表）或者是直接是一些代码也可以的。然后在新开的虚拟机里面，会停止GC。然后在 当前虚拟机内建立一个 Userdata对象，这个对象引用到我们新开的这个虚拟机。新开的这个虚拟机内的对象，被叫哦 matrix。然后返回这个 Userdata，这个对象权且就叫 state
2. `is_sharedtable` 检查一个给定的表是不是 shared
3. `stackvalues` 获取一个线程栈上的值
3. `clone` 将一个表，复制到当前虚拟机，利用了修改API `lua_clonetable`

返回的 state，附加了元表以进行操作：

- __index 索引 Userdata 自身
- close 关闭 state 引用的虚拟机
- getptr 获取到 matrix 表的指针
- size 获取 matrix 所处虚拟机的的大小，约等于matrix对象的大小。

光是这样可能还不够方便使用，所以云风在 Lua 层内又做了封装。



## matrix

matrix 对应的 C 函数是 `matrix_from_file`:

```c
matrix_from_file(lua_State *L) {	
	lua_State *mL = luaL_newstate();
	if (mL == NULL) {
		return luaL_error(L, "luaL_newstate failed");
	}
	const char * source = luaL_checkstring(L, 1);
	int top = lua_gettop(L);
	lua_pushcfunction(mL, load_matrixfile);
	lua_pushlightuserdata(mL, (void *)source);
	if (top > 1) {
		if (!lua_checkstack(mL, top + 1)) {
			return luaL_error(L, "Too many argument %d", top);
		}
		int i;
		for (i=2;i<=top;i++) {
			switch(lua_type(L, i)) {
			case LUA_TBOOLEAN:
				lua_pushboolean(mL, lua_toboolean(L, i));
				break;
			case LUA_TNUMBER:
				if (lua_isinteger(L, i)) {
					lua_pushinteger(mL, lua_tointeger(L, i));
				} else {
					lua_pushnumber(mL, lua_tonumber(L, i));
				}
				break;
			case LUA_TLIGHTUSERDATA:
				lua_pushlightuserdata(mL, lua_touserdata(L, i));
				break;
			case LUA_TFUNCTION:
				if (lua_iscfunction(L, i) && lua_getupvalue(L, i, 1) == NULL) {
					lua_pushcfunction(mL, lua_tocfunction(L, i));
					break;
				}
				return luaL_argerror(L, i, "Only support light C function");
			default:
				return luaL_argerror(L, i, "Type invalid");
			}
		}
	}
	int ok = lua_pcall(mL, top, 1, 0);
	if (ok != LUA_OK) {
		lua_pushstring(L, lua_tostring(mL, -1));
		lua_close(mL);
		lua_error(L);
	}
	return box_state(L, mL);
}
```

我们看到，只需要传递一个文件名（字符串，@ 开头是文件，没有@的就是字符串），还可以传递参数，不过参数只支持：boolean, number, lightuserdata, light c function。

然后调用 load_matrixfile，返回一个表回来。

## load_matrixfile

```c
static int
load_matrixfile(lua_State *L) {
	luaL_openlibs(L);
	const char * source = (const char *)lua_touserdata(L, 1);
	if (source[0] == '@') {
		if (luaL_loadfilex_(L, source+1, NULL) != LUA_OK)
			lua_error(L);
	} else {
		if (luaL_loadstring(L, source) != LUA_OK)
			lua_error(L);
	}
	lua_replace(L, 1);
	if (lua_pcall(L, lua_gettop(L) - 1, 1, 0) != LUA_OK)
		lua_error(L);
	lua_gc(L, LUA_GCCOLLECT, 0);
	lua_pushcfunction(L, make_matrix);
	lua_insert(L, -2);
	lua_call(L, 1, 1);
	return 1;
}
```

我们看到，我们传递的文件，加载后是一个函数，然后这个函数会收到我们传递的参数。

代码不大，实际上感谢于云风所打的几个补丁，可以直接将指针克隆到一个虚拟机中去。

共享的表，实际上是在一个单独的虚拟机中，不过这个虚拟将这个表的指针共享了出来，同时关闭了 GC。

我们可以在共享表中存在的数据类型有：

- table
- number
- boolean
- lightuserdata
- 没有上值的Lua函数
- 字符串。



# Sharetable Lua 库。

这个库，与一个服务 Sharetable 相关，都是由 sharetable 唯一服务去完成的。sharetable 服务内，保持了对所有文件名到 matrix（state）对象的映射。

- **sharetable.loadfile(filename, ...)** 从一个源文件读取一个共享表，这个文件需要返回一个 table ，这个 table 可以被多个不同的服务读取。... 是传给这个文件的参数。
- **sharetable.loadstring(filename, source, ...)** 和 loadfile 类似，但是是从一个字符串读取。
- **sharetable.loadtable(filename, tbl)** 直接将一个 table 共享。
- **sharetable.query(filename)** 以 filename 为 key 查找一个被共享的 table 。
- **sharetable.update(filenames)** 更新一个或多个 key 。

我们来看如何实现的。服务内部，有一些状态信息

```lua
  local matrix = {}	-- all the matrix pointer to refs map 记录每个 matrix 指针被引用的服务列表
	local files = {}	-- filename : matrix
	local clients = {} -- 记录每个服务引用的 matrix 指针列表
```

因此，实际上我我们调用这个库的 API的时候，都是向 sharetable 进行请求，当 请求新建的时候，就由服务建立一个  matrix 对象，但并不返回。必须由使用方向 sharetable 服务进行查询，然后才会返回这个指针。返回这个指针后，就会将这个指针，直接设置为一个 table 类型的值，可以使用了。

```lua
local RECORD = {} --缓存，文件名到复制表
function sharetable.query(filename)
	local newptr = skynet.call(sharetable.address, "lua", "query", filename)
	if newptr then
		local t = core.clone(newptr)
		local map = RECORD[filename]
		if not map then
			map = {}
			RECORD[filename] = map
		end
		map[t] = true
		return t
	end
end
```

每次我们查询，都会返回的都是由 sharetable 引用的那个指针，因此可以保证，都是最新的。我们可以采用这种方式来保证顶层表永远是最新的。

每个文件名，在本地的缓存，可能有多份，体现在 `RECORED[filename][t] = true` ，当有更新过的表的时候，将会是在缓存的对应键名下有多个数据。

# sharetable.update

这个部分是最消耗性能，和复杂的地方。当我们要更新一个或多个键的时候，会查找出本服务内的缓存，和 sharetable 内的的内容，来进行更新。相当于是做一个深层次更新，然后把老的一份表取消引用。

```lua
function sharetable.update(...)
	local names = {...}
	local replace_map = {}
	for _, name in ipairs(names) do
		local map = RECORD[name]
		if map then
			local new_t = sharetable.query(name)
			for old_t,_ in pairs(map) do
				if old_t ~= new_t then
					insert_replace(old_t, new_t, replace_map)
                    map[old_t] = nil
				end
			end
		end
	end

    if next(replace_map) then
        resolve_replace(replace_map)
    end
end
```

replace_map 里面记录的将会是所有老值到新值的映射。

resolve_replace 做的工作将会非常的繁重。他将会从虚拟机的注册表开始，来向下扫描，进行变更：

```lua
    local root = debug.getregistry()
    assert(replace_map[root] == nil)
    match_table(root)
```

关于为什么会需要扫描整个虚拟机， 是因为，因为有些地方，不是引用的顶层表，而有可能是直接用 local 的形式进行了引用，需要完全的扫描才知道到底是不是被使用了。而更新后的表已经是完全一张新表了。

我决定来啃一下是如何更新的。

## resolve_replace

```lua
local function resolve_replace(replace_map)
    local match = {} --记录各类型的匹配函数 table, function, userdata, thread
    local record_map = {} --记录
		-- 这个主要解决，当一个需要替换的值是 nil 的时候，表会索引不到的情况，因此用 NILOBJ 来替换，需要使用时进行替换
    local function getnv(v)
        local nv = replace_map[v]
        if nv then
            if nv == NILOBJ then
                return nil
            end
            return nv
        end
        assert(false)
    end
		-- 更新一个值
    local function match_value(v)
        -- nil 或者是已经记录了新值的，或是一个共享表，就不用遍历了
        if v == nil or record_map[v] or is_sharedtable(v) then
            return
        end

        local tv = type(v)
        local f = match[tv] --值 只能是 userdate, table, thread, function 类型
        if f then
            record_map[v] = true
            return f(v)
        end
    end
	 -- 遍历元表，这个比较简单
    local function match_mt(v)
        local mt = debug.getmetatable(v)
        if mt then
            local nv = replace_map[mt]
            if nv then
                nv = getnv(mt)
                debug.setmetatable(v, nv)
            else
                return match_value(mt)
            end
        end
    end

    local function match_internmt()
        local internal_types = {
            pointer = debug.upvalueid(getnv, 1),
            boolean = false,
            str = "",
            number = 42,
            thread = coroutine.running(),
            func = getnv,
        }
        for _,v in pairs(internal_types) do
            match_mt(v)
        end
        return match_mt(nil)
    end

		-- 遍历替换一个表的键和值进行更新
    local function match_table(t)
        local keys = false
        for k,v in next, t do
            local tk = type(k)
            if match[tk] then -- 若是以  table, function, userdata, thread 作为一个键，那么需要进行更新，记录下来
                keys = keys or {}
                keys[#keys+1] = k
            end

            local nv = replace_map[v] 
            if nv then -- 发现值是被更新过了
                nv = getnv(v)
                rawset(t, k, nv)
            else  -- 没有更新的值，那么遍历
                match_value(v)
            end
        end

        if keys then -- 需要遍历的键
            for _, old_k in ipairs(keys) do
                local new_k = replace_map[old_k]
                if new_k then
                    local value = rawget(t, old_k)
                    new_k = getnv(old_k)
                    rawset(t, old_k, nil)
                    if new_k then
                        rawset(t, new_k, value)
                    end
                else  -- 找不到，继续递归遍历
                    match_value(old_k)
                end
            end
        end
        return match_mt(t) -- 遍历更新 t 的元表
    end
		-- 更新 userdata
    local function match_userdata(u)
        local uv = getuservalue(u)
        local nv = replace_map[uv]
        if nv then
            nv = getnv(uv)
            setuservalue(u, nv)
        end
        return match_mt(u)
    end
		-- 更新函数信息
    local function match_funcinfo(info)
        local func = info.func
        local nups = info.nups
        for i=1,nups do --更新上值
            local name, upv = getupvalue(func, i)
            local nv = replace_map[upv]
            if nv then
                nv = getnv(upv)
                setupvalue(func, i, nv)
            elseif upv then
                match_value(upv)
            end
        end

        local level = info.level
        local curco = info.curco
        if not level then
            return
        end
        local i = 1
        while true do -- 更新 local 值
            local name, v = getlocal(curco, level, i)
            if name == nil then
                break
            end
            if replace_map[v] then
                local nv = getnv(v)
                setlocal(curco, level, i, nv)
            elseif v then
                match_value(v)
            end
            i = i + 1
        end
    end
		-- 更新一个函数内的值
    local function match_function(f)
        local info = getinfo(f, "uf")
        return match_funcinfo(info)
    end

    local function match_thread(co, level)
        -- match stackvalues
        local values = {}
        local n = stackvalues(co, values)
        for i=1,n do
            local v = values[i]
            match_value(v)
        end

        local uplevel = co == coroutine.running() and 1 or 0
        level = level or 1
        while true do
            local info = getinfo(co, level, "uf")
            if not info then
                break
            end
            info.level = level + uplevel
            info.curco = co
            match_funcinfo(info)
            level = level + 1
        end
    end

    local function prepare_match()
        local co = coroutine.running()
        record_map[co] = true
        record_map[match] = true
        record_map[RECORD] = true
        record_map[record_map] = true
        record_map[replace_map] = true
        record_map[insert_replace] = true
        record_map[resolve_replace] = true
        assert(getinfo(co, 3, "f").func == sharetable.update)
        match_thread(co, 5) -- ignore match_thread and match_funcinfo frame
    end

    match["table"] = match_table
    match["function"] = match_function
    match["userdata"] = match_userdata
    match["thread"] = match_thread

    prepare_match()
    match_internmt()

    local root = debug.getregistry()
    assert(replace_map[root] == nil)
    match_table(root)
end

```



# 关于如何使用的讨论

见 [https://github.com/cloudwu/skynet/discussions/1429](https://github.com/cloudwu/skynet/discussions/1429)
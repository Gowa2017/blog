---
title: luasocket中的http实现
categories:
  - Lua
date: 2019-11-11 12:17:52
updated: 2019-11-11 12:17:52
tags: 
  - Lua
---
看他的实现很有意思，又有同学遇到了在下载大文件的时候， OOM 的错误，所以就看一下是如何解决这个问题。事实上其是依赖于 ltn12，Lua Technicl Notes 12 ，实现了数据的分段处理，有点类似响应式编程和观察者模式。

<!--more-->

# 简介
LuaSocket 的官方地址这 [http://w3.impa.br/~diego/software/luasocket/](http://w3.impa.br/~diego/software/luasocket/)。

http 模块，位于 `socket.http` 中，其是基于 socket 之上的一个 http 实现。有两种形式，其只导出了三个函数：

- http.open(host, port, create)
- http.request(url [, body])
- http.request {}

我们来看一下具体的实现。

# http.open()
这个函数返回一个表 {}，这个表代表了连接本身，我们可以将其看成是一个 *handle*。表中元素 **c** 代表了套接字，这个表没有任何其他元素，除了套接字。对于这个 handle 上的所有操作都封装在了其元表中。

```lua
function _M.open(host, port, create)
    -- create socket with user connect function, or with default
    local c = socket.try((create or socket.tcp)())
    local h = base.setmetatable({ c = c }, metat)
    -- create finalized try
    h.try = socket.newtry(function() h:close() end)
    -- set timeout before connecting
    h.try(c:settimeout(_M.TIMEOUT))
    h.try(c:connect(host, port or _M.PORT))
    -- here everything worked
    return h
end

local metat = { __index = {} }

-- 发送请求
function metat.__index:sendrequestline(method, uri)
    local reqline = string.format("%s %s HTTP/1.1\r\n", method or "GET", uri)
    return self.try(self.c:send(reqline))
end

-- 发送 http 头
function metat.__index:sendheaders(tosend)
    local canonic = headers.canonic
    local h = "\r\n"
    for f, v in base.pairs(tosend) do
        h = (canonic[f] or f) .. ": " .. v .. "\r\n" .. h
    end
    self.try(self.c:send(h))
    return 1
end

-- 发送 body
function metat.__index:sendbody(headers, source, step)
    source = source or ltn12.source.empty()
    step = step or ltn12.pump.step
    -- if we don't know the size in advance, send chunked and hope for the best
    local mode = "http-chunked"
    if headers["content-length"] then mode = "keep-open" end
    return self.try(ltn12.pump.all(source, socket.sink(mode, self.c), step))
end

-- 读取返回的状态行
function metat.__index:receivestatusline()
    local status = self.try(self.c:receive(5))
    -- identify HTTP/0.9 responses, which do not contain a status line
    -- this is just a heuristic, but is what the RFC recommends
    if status ~= "HTTP/" then return nil, status end
    -- otherwise proceed reading a status line
    status = self.try(self.c:receive("*l", status))
    local code = socket.skip(2, string.find(status, "HTTP/%d*%.%d* (%d%d%d)"))
    return self.try(base.tonumber(code), status)
end

-- 读取返回的头
function metat.__index:receiveheaders()
    return self.try(receiveheaders(self.c))
end

-- 读取返回的body
function metat.__index:receivebody(headers, sink, step)
    sink = sink or ltn12.sink.null()
    step = step or ltn12.pump.step
    local length = base.tonumber(headers["content-length"])
    local t = headers["transfer-encoding"] -- shortcut
    local mode = "default" -- connection close
    if t and t ~= "identity" then mode = "http-chunked"
    elseif base.tonumber(headers["content-length"]) then mode = "by-length" end
    return self.try(ltn12.pump.all(socket.source(mode, self.c, length),
            sink, step))
end

-- ? 这个09 代表了什么鬼？
function metat.__index:receive09body(status, sink, step)
    local source = ltn12.source.rewind(socket.source("until-closed", self.c))
    source(status)
    return self.try(ltn12.pump.all(source, sink, step))
end

--  关闭连接
function metat.__index:close()
    return self.c:close()
end
```

OK，我们已经知道了是如何封装一个 http client，我们来看一下具体访问的时候呢。

# http.request()

这个函数，会在 socket.protect 的包装下，根据请求类型（是一个字符串，还是表）来决定访问的的形式。
```lua
_M.request = socket.protect(function(reqt, body)
    if base.type(reqt) == "string" then return srequest(reqt, body)
    else return trequest(reqt) end
end)
```

## srequest(u, b)
当我们传递给 request 的参数是一个 http URL 地址，加上一个可选的 body 参数的时候，会采用这种形式。

```lua
local function srequest(u, b)
    local t = {}
    local reqt = {
        url = u,
        sink = ltn12.sink.table(t)
    }
    if b then
        reqt.source = ltn12.source.string(b)
        reqt.headers = {
            ["content-length"] = string.len(b),
            ["content-type"] = "application/x-www-form-urlencoded"
        }
        reqt.method = "POST"
    end
    local code, headers, status = socket.skip(1, trequest(reqt))
    return table.concat(t), code, headers, status
end
```

看实现，我们能理解出，实际上，其也是用 url 来构造了一个 表，表中指定了我们采用表形式需要的信息。

可以看到，构造的表中有如下 信息：

```lua
{
  url = string,
  [sink = LTN12 sink,]
  [method = string,]
  [headers = header-table,]
  [source = LTN12 source],
  [step = LTN12 pump step,]
  [proxy = string,]
  [redirect = boolean,]
  [create = function]
}
```

其中  url , sink 是不然会有的。
## trequest()

请求最终都会走到这个函数：

```lua
--[[local]] function trequest(reqt)
    -- we loop until we get what we want, or
    -- until we are sure there is no way to get it
    local nreqt = adjustrequest(reqt)
    local h = _M.open(nreqt.host, nreqt.port, nreqt.create)
    -- send request line and headers
    h:sendrequestline(nreqt.method, nreqt.uri)
    h:sendheaders(nreqt.headers)
    -- if there is a body, send it
    if nreqt.source then
        h:sendbody(nreqt.headers, nreqt.source, nreqt.step)
    end
    local code, status = h:receivestatusline()
    -- if it is an HTTP/0.9 server, simply get the body and we are done
    if not code then
        h:receive09body(status, nreqt.sink, nreqt.step)
        return 1, 200
    end
    local headers
    -- ignore any 100-continue messages
    while code == 100 do
        headers = h:receiveheaders()
        code, status = h:receivestatusline()
    end
    headers = h:receiveheaders()
    -- at this point we should have a honest reply from the server
    -- we can't redirect if we already used the source, so we report the error
    if shouldredirect(nreqt, code, headers) and not nreqt.source then
        h:close()
        return tredirect(reqt, headers.location)
    end
    -- here we are finally done
    if shouldreceivebody(nreqt, code) then
        h:receivebody(headers, nreqt.sink, nreqt.step)
    end
    h:close()
    return 1, code, headers, status
end
```
过程很简单，我们来看看我们在接收 body 的时候过程是怎么样的。

## receivebody

```lua
function metat.__index:receivebody(headers, sink, step)
    sink = sink or ltn12.sink.null()
    step = step or ltn12.pump.step
    local length = base.tonumber(headers["content-length"])
    local t = headers["transfer-encoding"] -- shortcut
    local mode = "default" -- connection close
    if t and t ~= "identity" then mode = "http-chunked"
    elseif base.tonumber(headers["content-length"]) then mode = "by-length" end
    return self.try(ltn12.pump.all(socket.source(mode, self.c, length),
            sink, step))
end
```
默认情况下， sink 会是一个  ltn12.source.table(t)

```lua
function sink.table(t)
    t = t or {}
    local f = function(chunk, err)
        if chunk then table.insert(t, chunk) end
        return 1
    end
    return f, t

```
这个函数，返回的 sink 函数，会有一个上值，t，这个上值就是最终存储数据的所在。

step 则是 ltn12.pump.step。

ltn12.pump.all 则会将套接字上，length 长度的内容一次次的给弹出来。

```lua
-- pumps one chunk from the source to the sink
function pump.step(src, snk)
    local chunk, src_err = src()
    local ret, snk_err = snk(chunk, src_err)
    if chunk and ret then return 1
    else return nil, src_err or snk_err end
end

-- pumps all data from a source to a sink, using a step function
function pump.all(src, snk, step)
    base.assert(src and snk)
    step = step or pump.step
    while true do
        local ret, err = step(src, snk)
        if not ret then
            if err then return nil, err
            else return 1 end
        end
    end
end
```
其本质，就是循环调用  step ，一步步的从 src 内拿取数据，直到出错或者完成。

而对于 source，会根据 http 访问的模式，来决定，对应的函数，其实现在 socket.lua 中。

---
title: Skynet中sproto使用示例
categories:
  - [Lua]
  - [Skynet]
date: 2018-02-16 01:09:23
updated: 2018-02-16 01:09:23
tags:
  - Lua
  - Skynet
---

Sproto 是一个用 C 编写的高效序列化库，主要是想用来做 Lua 绑定。类似 Google 的 protocol buffers，但是速度更快。其设计得非常简单。只支持 Lua 支持的几种数据类型，其可以很容易的绑定到其他动态语言，或者直接在 C 中使用。

<!--more-->

# 简介

其项目开源到 [github.com/cloudwu/sproto](https://github.com/cloudwu/sproto)

其主要包含一些提供给 Lua 使用的 API，一个语法解析模块(parser) `sprotoparser`，还有一个 RPC API，加上 C 库。

# 解析器

```lua
local parser = require "sprotoparser"
```

- `parser.parse` 把一个 sproto 协议框架解析为一个二进制字符串

在解析的时候需要用到这个。可以用它来产生二进制字符串。框架文本和解析器在程序运行的时候并不需要

# Lua API

我们先看看看它提供给 Lua 使用的 API。

```lua
local sproto = require "sproto"
local sprotocore = require "sproto.core" -- optional
```

- **sproto.parse(schema)** 通过一个 文本字符串 的框架生成一个 sproto 对象。
- **sproto.new(spbin)** 通过一个 二进制的字符串（parser 生成） 生成一个 sproto 对象。
- **sprotocore.newproto(spbin)** 通过一个 二进制的字符串（parser 生成） 生成一个 C sproto 对象。
- **sproto.sharenew(spbin)** 从一个 sproto C 对象（sprotocore.newproto()生成）共享一个 sproto 对象。
- **sproto:exist_type(typename)** 检查 sproto 对象中是否存在此类型。
- **sproto:encode(typename, luatable)** 把一个 Lua 表以 _typename_ 编码到二进制字符串内。
- **sproto:decode(typename, blob [,sz])** 以*typename*来解码一个 `sproto:encode()`产生的二进制字符串。如果 _blob_ 是一个 lightuserdata (C 指针），_sz_ 是必须的。
- **sproto:pencode(typename, luatable)** 类似`sproto:encode`，但是会压缩结果。
- **sproto:pdecode(typename, blob [,sz])** 类似 `sproto.decode`，但是会先解压缩对象。
- **sproto:default(typename, type)** 以类型名的默认值来建立一个表。类型可以是 _nil, REQUEST, RESPONSE_。

# RPC API

这些 API 是对 core API 的封装。

**`sproto:host([packagename])`** 以 _packagename_ 建立一个宿主对象 _host_ 来传输 RPC 消息。

**`host:dispatch(blob [,sz])`** 以 _host_ 对象内的（packagename）来解压并解码（`sproto:pdecode`）二进制字符串。

如果 _.type_ 存在，这是一个 有*.type* `REQUEST` 消息，返回*REQUEST, protoname, message, responser, .ud*。*responser*是一个用来编码 响应消息的函数。 当*.session*不存在时，*responser*将会是 _nil_。

如果 _.type_ 不存在，这是一个给 _.session_ 的 `RESPONSE`消息。返回 _REPONSE, .session, message, .ud_。

**`host:attach(sprotoobj)`** 建立一个以 _sprotoobj_ 来压缩和编码请求消息的函数 `function(protoname, message, session, ud)`。

如果不想使用主机对象，可以用下面的 API 来编码和解码 RPC 消息。

**`sproto:request_encode(protoname, tbl)`** 以*protoname* 来编码一个请求消息。

**`sproto:response_encode(protoname, tbl)`** 以*protoname* 来编码一个响应消息。

**`sproto:request_decode(protoname, blob [,sz])`** 解码一个请求消息。

**`sproto:response_decode(protoname, blob [,sz]`** 解码一个响应消息

# 数据类型

- **string** : string
- **binary** : binary string (字符串的子类型)
- **integer** : 整型，最大整型是有符号 64 位的。 可以是一个不动点的特定精度的数字。
- **boolean** : true or false

在类型前面添加一个 `*` 来表示一个数组。

可以指定一个主索引，数组将会被编码成一个无序的 map。

用户定义的类型可以是任何非保留的名字，也支持嵌套类型。

没有双精度或者实数类型。作者认为，这些类型非常少使用。如果果真需要的话，可以用字符串来序列化双精度数。如果需要十进制数，可以指定固定的精度。

枚举类型并不十分实用。我们在 Lua 定义一个 enum 表来实现。

# 协议定义

sproto 是一个协议封装库。所以我们要定义我们自己的协议格式(schema)。

sproto 消息是强类型的，而且不是自描述的。所以必须用一个特殊的语言来定义我们自己的消息结构。

然后调用 _sprotoparser_ 来把 协议格式 解析为二进制字符串，这样 sproto 库就可以使用它。

可以离线解析，然后保存这些字符串，或者可以在程序运行的时候解析。

一个协议框架可能会像这样：

```
# 注释

.Person {	# . 表示一个用户定义数据类型
    name 0 : string	# 内建数据类型 string
    id 1 : integer
    email 2 : string

    .PhoneNumber {	# 可以嵌套用户自定义数据类型
        number 0 : string
        type 1 : integer
    }

    phone 3 : *PhoneNumber	# *PhoneNumber 表示数组
    height 4 : integer(2)	# (2) means a 1/100 精度的数    data 5 : binary		# 二进制数据
}

.AddressBook {
    person 0 : *Person(id)	# (id) 可选， Person.id 是一个主索引}

foobar 1 {	# 定义一个新协议  (for RPC used)  tag 1
    request Person	# 把数据类型 Person与 foobar.request 相关联
    response {	# 定义 foobar.response 的数据类型
        ok 0 : boolean
    }
}

```

一个框架可以是 被 sproto 框架语言自描述的：

```
.type {
    .field {
        name 0 : string
        buildin	1 : integer
        type 2 : integer	# type is fixed-point number precision when buildin is SPROTO_TINTEGER; When buildin is SPROTO_TSTRING, it means binary string when type is 1.
        tag 3 : integer
        array 4	: boolean
        key 5 : integer # If key exists, array must be true, and it's a map.
    }
    name 0 : string
    fields 1 : *field
}

.protocol {
    name 0 : string
    tag 1 : integer
    request 2 : integer # index
    response 3 : integer # index
    confirm 4 : boolean # response nil where confirm == true
}

.group {
    type 0 : *type
    protocol 1 : *protocol
}
```

# Wire protocol

每个整数以小端（little endian）格式序列化。

sproto 消息必须是一个用户定义类型结构，每个结构编码成三个部分。_header, field, data_（头部，字段，数据）。标签（tag）和 小的整数 或 布尔值 会被编码到 field 部分，其他的都在 data 部分。

所有的字段必须以升序编码（通过 标签 tag，从 0 开始）。当有字段是 *nil*的时候（lua 中的默认值），不要在消息中进行编码。 字段的标签因此可能是不连续的。

头部（header）是一个 16bit 整数。就是字段数两。

字段部分的所有字段都是一个 16bit 整数(n)。如果 _n_ 为 0，表示这个字段的数据编码在数据部分；

如果 _n_ 是不为 0 的偶数，字段的值是 _n/2-1_，tag（标签）会增加 1；
如果 _n_ 是奇数，表示标签是不连续的，我们应该把当前标签 增加 _(n+1)/2_。

数组总是被编码到数据部分，4 bytes 来表示大小，接下来的字节就是内容。（len-value)二元组。查看 例子 2 来了解 结构数组； 例子 3/4 展示整数数组； 例子 5 是布尔数组。

对于一个整型数组，一个额外的字节(4 or 8)来表示这个值是 32bit 还是 64bit。

查看下面的例子。

> 注意：如果 标签没有在 框架内声明，解码器为了协议版本的兼容，会忽略那些字段。

```
.Person {
    name 0 : string
    age 1 : integer
    marital 2 : boolean
    children 3 : *Person
}

.Data {
	numbers 0 : *integer
	bools 1 : *boolean
	number 2 : integer
	bignumber 3 : integer
}
```

## 例子 1

```
person { name = "Alice" ,  age = 13, marital = false }

03 00 (fn = 3)
00 00 (id = 0, value in data part)
1C 00 (id = 1, value = 13)
02 00 (id = 2, value = false)
05 00 00 00 (sizeof "Alice")
41 6C 69 63 65 （“Alice)
```

## 例子 2

```
person {
    name = "Bob",
    age = 40,
    children = {
        { name = "Alice" ,  age = 13 },
        { name = "Carol" ,  age = 5 },
    }
}

04 00 (fn = 4)
00 00 (id = 0, value in data part)
52 00 (id = 1, value = 40)
01 00 (skip id = 2)
00 00 (id = 3, value in data part)

03 00 00 00 (sizeof "Bob")
42 6F 62 ("Bob")

26 00 00 00 (sizeof children)

0F 00 00 00 (sizeof child 1)
02 00 (fn = 2)
00 00 (id = 0, value in data part)
1C 00 (id = 1, value = 13)
05 00 00 00 (sizeof "Alice")
41 6C 69 63 65 ("Alice")

0F 00 00 00 (sizeof child 2)
02 00 (fn = 2)
00 00 (id = 0, value in data part)
0C 00 (id = 1, value = 5)
05 00 00 00 (sizeof "Carol")
43 61 72 6F 6C ("Carol")
```

## 例子 3

```
data {
    numbers = { 1,2,3,4,5 }
}

01 00 (fn = 1)
00 00 (id = 0, value in data part)

15 00 00 00 (sizeof numbers)
04 ( sizeof int32 )
01 00 00 00 (1)
02 00 00 00 (2)
03 00 00 00 (3)
04 00 00 00 (4)
05 00 00 00 (5)
```

## 例子 4

```
data {
    numbers = {
        (1<<32)+1,
        (1<<32)+2,
        (1<<32)+3,
    }
}

01 00 (fn = 1)
00 00 (id = 0, value in data part)

19 00 00 00 (sizeof numbers)
08 ( sizeof int64 )
01 00 00 00 01 00 00 00 ( (1<32) + 1)
02 00 00 00 01 00 00 00 ( (1<32) + 2)
03 00 00 00 01 00 00 00 ( (1<32) + 3)
```

# 例子 5:

```
data {
    bools = { false, true, false }
}

02 00 (fn = 2)
01 00 (skip id = 0)
00 00 (id = 1, value in data part)

03 00 00 00 (sizeof bools)
00 (false)
01 (true)
00 (false)
```

# 例子 6:

```
data {
    number = 100000,
    bignumber = -10000000000,
}

03 00 (fn = 3)
03 00 (skip id = 1)
00 00 (id = 2, value in data part)
00 00 (id = 3, value in data part)

04 00 00 00 (sizeof number, data part)
A0 86 01 00 (100000, 32bit integer)

08 00 00 00 (sizeof bignumber, data part)
00 1C F4 AB FD FF FF FF (-10000000000, 64bit integer)
```

# 0 Packing

算法类似 [Cap'n proto](http://kentonv.github.io/capnproto/),但是不特别对待 _0x00_。

在打包的格式中，消息会被填充到 8。每个标签背后的都是 8 字节的倍数。

标签字节的位对应了未打包字的字节数，最不重要的位对应第一个字节。

每个为 0 的位表示对应的字节是 0。而非 0 的字节被打包到 标签后面。

比如：

```
unpacked (hex):  08 00 00 00 03 00 02 00   19 00 00 00 aa 01 00 00
packed (hex):  51 08 03 02   31 19 aa 01
```

_0xff_ 标签会被特别对待。一个数字 _N_ 会跟在 _0xff_ 标签后面，表示 _(N+1)\*8_ 字节应该被直接复制。

字节可能包含也可能不包含 0 值。因为这个规则，最行的空间浪费就是每 2 KB 输入只打包了 2 字节数据。

例如：

```
unpacked (hex):  8a (x 30 bytes)
packed (hex):  ff 03 8a (x 30 bytes) 00 00
```

# C API

```C
struct sproto * sproto_create(const void * proto, size_t sz);
```

以一个被 sprotoparser 编码的 框架字符串来建立一个 sproto 对象。

```C
void sproto_release(struct sproto *);
```

释放 sproto object:

```C
int sproto_prototag(struct sproto *, const char * name);
const char * sproto_protoname(struct sproto *, int proto);
// SPROTO_REQUEST(0) : request, SPROTO_RESPONSE(1): response
struct sproto_type * sproto_protoquery(struct sproto *, int proto, int what);
```

在一个协议的 标签和名字间转换，并查询对象的类型。

```C
struct sproto_type * sproto_type(struct sproto *, const char * typename);
```

从一个 sproto 对象查询类型对象。

```C
struct sproto_arg {
	void *ud;
	const char *tagname;
	int tagid;
	int type;
	struct sproto_type *subtype;
	void *value;
	int length;
	int index;	// array base 1
	int mainindex;	// for map
	int extra; // SPROTO_TINTEGER: fixed-point presision ; SPROTO_TSTRING 0:utf8 string 1:binary
};

typedef int (*sproto_callback)(const struct sproto_arg *args);

int sproto_decode(struct sproto_type *, const void * data, int size, sproto_callback cb, void *ud);
int sproto_encode(struct sproto_type *, void * buffer, int size, sproto_callback cb, void *ud);
```

以一个用户定义的回调函数编码和解码 sproto 消息。查看 lsproto.c 的实现来看更多的信息。

```C
int sproto_pack(const void * src, int srcsz, void * buffer, int bufsz);
int sproto_unpack(const void * src, int srcsz, void * buffer, int bufsz);
```

以 0 packing 算法来打包和解包消息。

# 总结

在 TCP 连接上，我们发送和读取的的数据，都是连续的字节流。我们无法知道我应该读取的内容到底是什么，内容到底是什么，是由我们自己定义的协议所确定的。

而在基本的套接字编程示例中，我们都是调用系统的 `read(int fd, void * buffer, ssize_t sz)` 来将从文件描述符上将内存缓冲区的数据，读到我们自己的缓冲区内。

对此，在 skynet 的使用示例中，其把每个消息的前两个字节定义为 消息的长度，后面跟上真正的消息内容。

然后在我们以我们指定的协议进行解码。协议内容总是会包含一个协议头部：

```
.package {
    type 0 : integer--消息类型
    session 1 : integer--回应消息对应的关系
}
```

跟上真正的协议内容，然后以 `0-packing`方式打包。

?type 的值，表明了我们定义的协议中类型的标签值？

# 消息类型与请求类型

在[云风的博客上](https://blog.codingnow.com/2015/04/sproto_rpc.html)提到：

> 对于 request/response 的 RPC 方案，除了消息本身打包外，还有两个重要的信息需要传输。它们分别是请求的类型以及请求的 session 。
> 不要把请求的类型和消息的类型混为一谈。因为不同的请求可以用相同的消息类型，所以在 sproto 中，需要对 rpc 请求额外编码。你也不一定为每个请求额外设计一个消息类型，可以直接在定义 rpc 协议时内联写上请求（以及回应）的消息结构。

> 通常，我们用数字作为消息类型的标识，当然你也可以使用字符串。在用类 json 的无 schema 的协议中使用字符串多一些，但在 sproto 这种带 schema 的协议中，使用数字会更高效。同样，session 作为一条消息的唯一标识，你也可以用数字或字符串。而生成唯一数字 session 更容易，编码也更高效。

> 所以，每当我们发送一次远程请求，需要传输的数据就有三项：请求的类型、一个请求方自己保证唯一的 session id 以及请求的数据内容。

> 服务方收到请求后，应根据请求的类型对请求的数据内容解码，并根据类型分发给相应的处理器。同时应该把 session id 记录下来。等处理器处理完毕后，根据类型去打包回应的消息，并附加上 session id ，发送回客户端。

> 注意：回应是不需要传输消息类型的。这是因为 session id 就唯一标识了这是对哪一条请求的回应。而 session id 是客户端保证唯一的，它在产生 session id 时，就保存了这个 session 对应的请求的类型，所以也就有能力对回应消息解码。

> btw ，如果只是单向推送消息（也就是 publish/subscribe 模式），直接省略 session 就可以了，也不需要回应。

在上面一节中，我们说道 `.package` 就是一个我们定义的消息类型，而其中的 `type` 字段，定义了我们的请求类型。

对于每个包，都以这个 package 开头，后面接上 (padding）消息体。最后连在一起，用 sproto 自带的 0-pack 方式压缩。

我们可以这样理解：

消息类型 `.package` 定义了我们消息包含的内容。

而 `.type` 定义了我们消息内容是怎么表示的。

# client.lua 使用示例

我们先来看一下一般性的代码：

```lua
-- 加载 socket, proto, sproto 库
local socket = require "client.socket"

-- proto是我们自己定义的协议库（模块）
local proto = require "proto"
local sproto = require "sproto"

local host = sproto.new(proto.s2c):host "package"
local request = host:attach(sproto.new(proto.c2s))

local fd = assert(socket.connect("127.0.0.1", 8888))
```

首先，我们先要定义我们的协议，然后通过 parser 来解析成为一个二进制字符串，最后，调用 `sproto.new`来建立一个 sproto 对象。

## 协议定义

这是通过 `parser.parse`来解析一个我们用 schema 语言定义的框架，然后生成的字符串保存在 表中进行了返回。

其中对于 `c2s` 的协议，我们定义了一个 消息类型 `.package`，四个请求（协议）类型。

而对于 `s2c`的协议，我们只定义了一个请求（协议）类型。

```lua
proto.c2s = sprotoparser.parse [[
.package {
        type 0 : integer  -- 消息类型
        session 1 : integer  -- 会话ID
}

handshake 1 {
        response {
                msg 0  : string
        }
}

get 2 {
        request {
                what 0 : string
        }
        response {
                result 0 : string
        }
}

set 3 {
        request {
                what 0 : string
                value 1 : string
        }
}

quit 4 {}

]]

proto.s2c = sprotoparser.parse [[
.package {
        type 0 : integer
        session 1 : integer
}

heartbeat 1 {}
]]
```

## 对象建立

我们先来看看第一个调用：

```lua
local host = sproto.new(proto.s2c):host "package"
```

这个调用实际上就是：

```lua
local sobj = sproto.new(proto.s2c)
local host = sobj:host "package"
```

我们先看看第一步 `sproto.new`的定义：

```lua
local weak_mt = { __mode = "kv" }
local sproto_mt = { __index = sproto }
local sproto_nogc = { __index = sproto }
local host_mt = { __index = host }

function sproto.new(bin)
        local cobj = assert(core.newproto(bin))
        local self = {
                __cobj = cobj,
                __tcache = setmetatable( {} , weak_mt ),
                __pcache = setmetatable( {} , weak_mt ),
        }
        return setmetatable(self, sproto_mt)
end
```

其实是调用 注册出的的 core.newproto API，来建立了一个 sproto 对象。返回值就是 一个表 ，此表中的 `__cobj` 引用了 这个建立的 对象。这个表的元表已经被设置为 `sproto_mt`

```lua
        sobj = {
                __cobj = cobj,
                __tcache = setmetatable( {} , weak_mt ),
                __pcache = setmetatable( {} , weak_mt ),
        }
```

接下来我们调用的`sobj:host`，在 sobj 表内并不存在方法 `host`，所以其转而去寻找去 `__index`事件的元方法，这是一个表，就是 _sproto_，其实其调用的就是下面的这个方法。

```lua
function sproto:host( packagename )
        packagename = packagename or  "package"
        local obj = {
                __proto = self,
                __package = assert(core.querytype(self.__cobj, packagename), "type package not found"),
                __session = {},
        }
        return setmetatable(obj, host_mt)
end
```

会根据我们给定的 _packagename_ 消息类型来建立一个表对象 _obj_，这个表内的 `__proto` 事件就指向了我们的 *sproto*表，然后`__package`事件引用了 _packagename_ 在 建立的 sproto 对象中的位置。*host*对象的元表被设置成了 `host_mt`，其中具有 `dispatch, attach`两个方法。所以当 _host_，不存在对应方法时会调用元表中的方法。

最终我们可以得到一个表，也可以说是一个对象。_host_，

```lua
host =  {
         __proto = sobj,
         __package = assert(core.querytype(self.__cobj, packagename), "type package not found"),
         __session = {},
        }
```

## 消息分发器

实际上，我们对一个 sproto 对象调用 `:host`方法，就是为它绑定一个有两个方法 `dispatch, attach` 的元表。这样当访问这两个方法的时候就会直接访问我们绑定的方法。

### host:attach

我们来看一下 `attach` 方法：

```lua
function host:attach(sp)
        return function(name, args, session, ud)
        			// 在 sproto 对象内查找 name 协议
                local proto = queryproto(sp, name)
                // 消息头部 { type, session, ud}
                header_tmp.type = proto.tag
                header_tmp.session = session
                header_tmp.ud = ud
                // 头部进行 0 packing
                local header = core.encode(self.__package, header_tmp)

                if session then
                        self.__session[session] = proto.response or true
                end

                // 封装请求内容
                if proto.request then
                        local content = core.encode(proto.request, args)
                        return core.pack(header ..  content)
                else
                        return core.pack(header)
                end
        end
end
```

这个函数会返回一个函数：

```lua
function (name, args, session, ud) ... end
```

其会根据 `name`（协议类型/请求类型）来把 代表内容的 _args, session_ 打包。

### host:dispatch

我们先来看一下 `dispatch`方法：

```lua
function host:dispatch(...)
        local bin = core.unpack(...)
        header_tmp.type = nil
        header_tmp.session = nil
        header_tmp.ud = nil
        local header, size = core.decode(self.__package, bin, header_tmp)
        local content = bin:sub(size + 1)
        if header.type then
                -- request
                local proto = queryproto(self.__proto, header.type)
                local result
                if proto.request then
                        result = core.decode(proto.request, content)
                end
                if header_tmp.session then
                        return "REQUEST", proto.name, result, gen_response(self, proto.response, header_tmp.session), header.ud
                else
                        return "REQUEST", proto.name, result, nil, header.ud
                end
        else
                -- response
                local session = assert(header_tmp.session, "session not found")
                local response = assert(self.__session[session], "Unknown session")
                self.__session[session] = nil
                if response == true then
                        return "RESPONSE", session, nil, header.ud
                else
                        local result = core.decode(response, content)
                        return "RESPONSE", session, result, header.ud
                end
        end
end
```

## 消息发送

在调用 `local request = host:attach(sproto.new(proto.c2s))`后，建立了一个消息封装函数*request*。

函数 ：

```lua
local function send_request(name, args)
        session = session + 1
        local str = request(name, args, session)
        send_package(fd, str)
        print("Request:", session)
end
```

会将 会话 ID，协议名，参数传递给 消息封装函数。之后，函数：

```lua
local function send_package(fd, pack)
        local package = string.pack(">s2", pack)
        socket.send(fd, package)
end
```

会将打包好的消息，进行大端封装后发送到套接字去。

## 消息接收

服务端使用了 `snax.gateserver` 的实例 `gate`来实现连接管理，当收到一个消息时，如果有 agent，就会将消息转发到 agent 去：

```lua
-- services/gate.lua
function handler.message(fd, msg, sz)
        -- recv a package, forward it
        local c = connection[fd]
        local agent = c.agent
        if agent then
                skynet.redirect(agent, c.client, "client", 1, msg, sz)
        else
                skynet.send(watchdog, "lua", "socket", "data", fd, netpack.tostring(msg, sz))
        end
end
```

我们的 agent 服务在启动时即注册了 _client_ 类型的消息：

```lua
-- examples/agent.lua
skynet.register_protocol {
        name = "client",
        id = skynet.PTYPE_CLIENT,
        unpack = function (msg, sz)
                return host:dispatch(msg, sz)
        end,
        dispatch = function (_, _, type, ...)
                if type == "REQUEST" then
                        local ok, result  = pcall(request, ...)
                        if ok then
                                if result then
                                        send_package(result)
                                end
                        else
                                skynet.error(result)
                        end
                else
                        assert(type == "RESPONSE")
                        error "This example doesn't support request client"
                end
        end
}
```

其会使用 `host:dispatch`来解压消息，然后注册了自己的消息回调函数。

我们注意到，在服务端中，建立消息消息分发器的方式同客户端似乎都不一样：

```lua
-- examples/agent.lua
function CMD.start(conf)
        local fd = conf.client
        local gate = conf.gate
        WATCHDOG = conf.watchdog
        -- slot 1,2 set at main.lua
        host = sprotoloader.load(1):host "package"
        send_request = host:attach(sprotoloader.load(2))
        skynet.fork(function()
                while true do
                        send_package(send_request "heartbeat")
                        skynet.sleep(500)
                end
        end)

        client_fd = fd
        skynet.call(gate, "lua", "forward", fd)
end
```

其是通过 `sprotoloader.load(1):host "package"`来建立的。我们有理由去猜测，这个其实应该等价与：

```lua
sproto.new(proto.c2s):host "package"
```

因为其处理的，是从客户端到服务端的消息。

## sprotoloader

如果想要在程序中，各个服务中共享同样的消息类型和协议类型，为每个服务都单独的保存这些协议信息似乎是非常浪费的。所以就有了把共享的协议由一个服务来提供的想法。

其先启动了一个全局唯一的协议加载服务：

```lua
skynet.uniqueservice("protoloader")
```

```lua
skynet.start(function()
        sprotoloader.save(proto.c2s, 1)
        sprotoloader.save(proto.s2c, 2)
        -- don't call skynet.exit() , because sproto.core may unload and the global slot become invalid
end)
```

把 客户端到服务端的消息类型保存为索引 1。

这样当我们通过 `sprotoloader.load(1)`，就得出了这个索引对应的对象指针，在通过 `sproto.sharenew()`来把这个对象给返回给调用者。

```lua
-- lualib/sprotoloader.lua
function loader.load(index)
        local sp = core.loadproto(index)
        --  no __gc in metatable
        return sproto.sharenew(sp)
end

return loader
```

```lua
-- lualib/sproto.lua
function sproto.sharenew(cobj)
        local self = {
                __cobj = cobj,
                __tcache = setmetatable( {} , weak_mt ),
                __pcache = setmetatable( {} , weak_mt ),
        }
        return setmetatable(self, sproto_nogc)
end
```

这个函数其实是 `sproto.new`返回的值一样，不过其是直接传过去的对象，而不是二进制的字符串。

如此，我们的消息处理流程就完美了。

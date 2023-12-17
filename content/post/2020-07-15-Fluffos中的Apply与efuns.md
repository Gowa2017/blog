---
title: Fluffos中的Apply与efuns
categories:
  - Mud
date: 2020-07-15 23:59:53
updated: 2020-07-15 23:59:53
tags: 
  - Mud
  - FluffOS
---

从 {% post_link FluffOS的简单架构分析 FluffOS的简单架构分析 %} 中我们知道，Fluffos 实际上分为 driver 和 VM 两层，但其实更确切的说的话，在 driver 内还可以分为网络通信层，驱动与VM 的接口层这样。不过，暂时不用分那么细来理解就够了。

<!--more-->

# 几个数据结构

## interactive_t

这个代表的是一个可交互对象，也就是玩家 User 本身，在玩家进行网络连接的时候初始化。它会与一个 `object_t` 对象联系起来，还会保存此 User 所代表的，网络客户端上的 IP 端口等信息。

在添加新的 User，连线的时候，其是与 `master_ob` 相联系在一起的，同时，它也会与一个 bufferevent 套接字相关联，当这个网络套接字上有数据或者事件的时候，就会拿到这个 `interactive_t` ，将逻辑转交到与其关联的 `object_t` 对象，也就是 LPC 内。



## object_t

这个结构是一个 LPC 内对象的 C 层的表示，他会与一个编译后的 LPC 程序 `program_t` 相关联，也会保持对 `interactive_t` 的引用。

## program_t

一段可执行的，编译后的 LPC 代码了。

## sentence_t	

```cpp
//object.h
struct sentence_t {
#ifndef NO_ADD_ACTION
  char *verb;
#endif
  struct sentence_t *next;
  struct object_t *ob;
  union string_or_func function;
  int flags;
};
```

这个代表的是，一个用户输入岁的动词，与可执行的函数的绑定。

如果绑定的是一个 object_t 的函数，会使用 apply 的形式调用。

而如果就直接是一个函数指针，那么就直接调用就是了。

## VM 的栈

类似 Lua 的栈一样，这些函数调用，都是需要先将参数压入到栈上，然后由函数从栈上再去取的。同样，对于一个正在执行的 LPC　代码。

其会记录一个　current_obj , current_prog 这两者有可能并不一致。



# 网络数据的传输执行过程

我们从用户输入数据到最终执行返回来看一次结构。我们知道，在用户进行连接的时候，就会建立一个 `interactive_t` 对象，然后会对此对象连接上来的套接子建立一个 bufferevent 结构，设置回调。

添加新用户的时候还会设置一个命令处理器：

```cpp
// comm.cc
  // Command handler
  auto base = evconnlistener_get_base(port->ev_conn);
  user->ev_command = evtimer_new(base, on_user_command, user);
```



对于使用  telnet 会初始化一个 telnet 的处理器：

```cpp
// comm.cc
    if (user->connection_type == PORT_TELNET) {
      user->telnet = net_telnet_init(user);
      send_initial_telnet_negotiations(user);
    }
```



```cpp
// comm.cc
void new_user_event_listener(event_base *base, interactive_t *user) {
  auto bev =
      bufferevent_socket_new(base, user->fd, BEV_OPT_CLOSE_ON_FREE | BEV_OPT_DEFER_CALLBACKS);
  bufferevent_setcb(bev, on_user_read, on_user_write, on_user_events, user);
  bufferevent_enable(bev, EV_READ | EV_WRITE);

  bufferevent_set_timeouts(bev, nullptr, nullptr);

  user->ev_buffer = bev;
}
```

## on_user_read

```cpp
// comm.cc
void on_user_read(bufferevent *bev, void *arg) {
  auto user = reinterpret_cast<interactive_t *>(arg);

  if (user == nullptr) {
    fatal("on_user_read: user == NULL, Driver BUG.");
    return;
  }

  // Read user input
  get_user_data(user);

  // TODO: currently get_user_data() will schedule command execution.
  // should probably move it here.
}
```

针对用户的输入，会读取数据进行执行。最终都会对 `interactive_t` 这个结构上所关联的 `object_t` 进行 `apply()` 调用：

对于使用  telnet 连接的，读取了数据后，就会触发定时器的执行：

```cpp
// comm.cc
    case PORT_TELNET: {
      int start = ip->text_end;

      // this will read data into ip->text
      telnet_recv(ip->telnet, reinterpret_cast<const char *>(&buf[0]), num_bytes);

      // If we read something
      if (ip->text_end > start) {
        /* handle snooping - snooper does not see type-ahead due to
         telnet being in linemode */
        if (!(ip->iflags & NOECHO)) {
          handle_snoop(ip->text + start, ip->text_end - start, ip);
        }

        // search for command.
        if (cmd_in_buf(ip)) {
          ip->iflags |= CMD_IN_BUF;
          struct timeval zero_sec = {0, 0};
          evtimer_del(ip->ev_command);
          evtimer_add(ip->ev_command, &zero_sec);
        }
      }
      break;
    }
```



### on_user_command

在这里，会将用户的输入都解析，跳过一些不必要的命令和输入，然后调用 `process_input()`

### process_input

在这里才会解析命令。这个还要看，看所关联的 object_t 是不是有 `process_input` 这个方法。如果没有就在本地解析，如果有的话，就要压到 VM 里面去解析；在VM解析失败，还是会回到驱动层来解析。

对于 ES2 来说，角色继承了 `alias.c` 这个对象，其具有 `process_input` 方法，所以先会走一遍，然后返回回来后，继续执行。

```cpp
// comm.cc
  if (!(ip->iflags & HAS_PROCESS_INPUT)) {
    parse_command(user_command, command_giver);
    return;
  }

  /*
   * send a copy of user input back to user object to provide
   * support for things like command history and mud shell
   * programming languages.
   */
  copy_and_push_string(user_command);
  ret = apply(APPLY_PROCESS_INPUT, command_giver, 1, ORIGIN_DRIVER);
  if (!IP_VALID(ip, command_giver)) {
    return;
  }
  if (!ret) {
    ip->iflags &= ~HAS_PROCESS_INPUT;
    parse_command(user_command, command_giver);
    return;
  }

#ifndef NO_ADD_ACTION
  if (ret->type == T_STRING) {
    static char buf[MAX_TEXT];

    strncpy(buf, ret->u.string, MAX_TEXT - 1);
    parse_command(buf, command_giver);
  } else {
    if (ret->type != T_NUMBER || !ret->u.number) {
      parse_command(user_command, command_giver);
    }
  }
#endif
}
```

`parse_command` 最终会走到 `user_parser()` 这个函数去，这个函数，就是根据玩家的输入的命令找到一个 `sentence_t`，然后进行执行。

### user_parser

```cpp
// add_action.cc

static int user_parser(char *buff) {
  char verb_buff[MAX_VERB_BUFF];
  sentence_t *s;
  char *p;
  int length;
  char *user_verb = nullptr;
  int where;
  int save_illegal_sentence_action;

  debug(add_action, "cmd [/%s]: '%s'\n", command_giver->obname, buff);

  /* strip trailing spaces. */
  for (p = buff + strlen(buff) - 1; p >= buff; p--) {
    if (*p != ' ') {
      break;
    }
    *p = '\0';
  }
  if (buff[0] == '\0') {
    return 0;
  }
  length = p - buff + 1;
  p = strchr(buff, ' ');
  if (p == nullptr) {
    user_verb = findstring(buff);
  } else {
    *p = '\0';
    user_verb = findstring(buff);
    *p = ' ';
    length = p - buff;
  }
  if (!user_verb) {
    /* either an xverb or a verb without a specific add_action */
    user_verb = buff;
  }
  /*
   * copy user_verb into a static character buffer to be pointed to by
   * last_verb.
   */
  strncpy(verb_buff, user_verb, MAX_VERB_BUFF - 1);
  if (p) {
    int pos;

    pos = p - buff;
    if (pos < MAX_VERB_BUFF) {
      verb_buff[pos] = '\0';
    }
  }

  save_illegal_sentence_action = illegal_sentence_action;
  illegal_sentence_action = 0;
  for (s = command_giver->sent; s; s = s->next) {
    svalue_t *ret;

    if (s->flags & (V_NOSPACE | V_SHORT)) {
      if (strncmp(buff, s->verb, strlen(s->verb)) != 0) {
        continue;
      }
    } else {
      /* note: if was add_action(blah, "") then accept it */
      if (s->verb[0] && (user_verb != s->verb)) {
        continue;
      }
    }
    /*
     * Now we have found a special sentence !
     */

    if (!(s->flags & V_FUNCTION))
      debug(add_action, "Local command %s on /%s\n", s->function.s, s->ob->obname);

    if (s->flags & V_NOSPACE) {
      int l1 = strlen(s->verb);
      int l2 = strlen(verb_buff);

      if (l1 < l2) {
        last_verb = verb_buff + l1;
      } else {
        last_verb = "";
      }
    } else {
      if (!s->verb[0] || (s->flags & V_SHORT)) {
        last_verb = verb_buff;
      } else {
        last_verb = s->verb;
      }
    }
    /*
     * If the function is static and not defined by current object, then
     * it will fail. If this is called directly from user input, then
     * the origin is the driver and it will be allowed.
     */
    where = (current_object ? ORIGIN_EFUN : ORIGIN_DRIVER);

    /*
     * Remember the object, to update moves.
     */
    save_command_giver(command_giver);
    if (s->flags & V_NOSPACE) {
      copy_and_push_string(&buff[strlen(s->verb)]);
    } else if (buff[length] == ' ') {
      copy_and_push_string(&buff[length + 1]);
    } else {
      push_undefined();
    }
    if (s->flags & V_FUNCTION) {
      ret = call_function_pointer(s->function.f, 1);
    } else {
      if (s->function.s[0] == APPLY___INIT_SPECIAL_CHAR) {
        error("Illegal function name.\n");
      }
      ret = apply(s->function.s, s->ob, 1, where);
    }
    /* s may be dangling at this point */

    restore_command_giver();

    last_verb = nullptr;

    /* was this the right verb? */
    if (ret == nullptr) {
      /* is it still around?  Otherwise, ignore this ...
         it moved somewhere or dested itself */
      if (s == command_giver->sent) {
        char buf[256];
        if (s->flags & V_FUNCTION) {
          sprintf(buf, "Verb '%s' bound to uncallable function pointer.\n", s->verb);
          error(buf);
        } else {
          sprintf(buf, "Function for verb '%s' not found.\n", s->verb);
          error(buf);
        }
      }
    }

    if (ret && (ret->type != T_NUMBER || ret->u.number != 0)) {
#ifdef PACKAGE_MUDLIB_STATS
      if (command_giver && command_giver->interactive
#ifndef NO_WIZARDS
          && !(command_giver->flags & O_IS_WIZARD)
#endif
      ) {
        add_moves(&s->ob->stats, 1);
      }
#endif
      if (!illegal_sentence_action) {
        illegal_sentence_action = save_illegal_sentence_action;
      }
      return 1;
    }
    if (illegal_sentence_action) {
      switch (illegal_sentence_action) {
        case 1:
          error(
              "Illegal to call remove_action() [caller was /%s] from a verb "
              "returning zero.\n",
              illegal_sentence_ob->obname);
          break;
        case 2:
          error(
              "Illegal to move or destruct an object (/%s) defining actions "
              "from a verb function(%s) in object(/%s) which returns zero.\n",
              illegal_sentence_ob->obname, user_verb, s->ob->obname);
          break;
      }
    }
  }
  notify_no_command();
  illegal_sentence_action = save_illegal_sentence_action;

  return 0;
}

```

这收到输入的时候，会遍历 `object_t` 上的 `sentence_t` 链表。根据此 `sentence_t` 的 flag 还有  verb 来进行匹配。在两种情况下会匹配不上：

1. `sentence_t` 的 flag 是 (V_NOSPACE | V_SHORT)，同时输入匹配不上其 verb。
2. `sentence_t` 的 flag 是 V_FUNCTION ，且 verb 不为空，用户输入verb 匹配不上。

这就提供了一个情况：

当 sentence_t 的 flag 是 V_FUNCTION，但 verb 为空，那么就会全部都匹配上了。

ES2 就是利用了这么一个事实，来对所有的指令进行了 hook。

## on_user_write

```cpp
// comm.cc
void on_user_write(bufferevent *bev, void *arg) {
  auto user = reinterpret_cast<interactive_t *>(arg);
  if (user == nullptr) {
    fatal("on_user_write: user == NULL, Driver BUG.");
    return;
  }
  // nothing to do.
}
```

这个为什么不对用户进行写入？可能语义不太一样。

## on_user_events

```cpp
// comm.cc
void on_user_events(bufferevent *bev, short events, void *arg) {
  auto user = reinterpret_cast<interactive_t *>(arg);

  if (user == nullptr) {
    fatal("on_user_events: user == NULL, Driver BUG.");
    return;
  }

  if (events & (BEV_EVENT_ERROR | BEV_EVENT_EOF)) {
    user->iflags |= NET_DEAD;
    remove_interactive(user->ob, 0);
  } else {
    debug(event, "on_user_events: ignored unknown events: %d\n", events);
  }
```

针对玩家出现了错误事件，关闭套接子事件的处理。

# apply

我们在前面的描述中知道，实际上，加载一个文件，就是会分配一个 `object_t` 的数据结构（C层），然后 C 层这个数据结构会引用一个编译后的 LPC 程序（我想应该是二进制码，操作码，参数的形式）。然后解释器就会执行这样的二进制码。

最终，如果我们要调用 `create` 的话，其实要走的路还很多。

```cpp
svalue_t *apply(const char *fun, object_t *ob, int num_arg, int where);
```

- *fun* 要调用的 `object_t` 对象上的方法
- *ob* 应 apply 的`object_t` 对象
- *num_arg* 参数个数，这些参数会在 `apply()` 前就压到栈上去
- *where* 从哪里调用的这个函数

apply 实际上最终还调用的是 `apply_low()` 函数。

任何我们从网络端点上收到的数据想要传递到 VM 中的逻辑，都是通过 `apply()` 的形式进行实现的。

## apply_low

```cpp
int apply_low(const char *fun, object_t *ob, int num_arg) 
```

这个函数，会在 object_t 对象引用的 LPC 程序上查找 *fun* 函数执行，同时将传递 *num_arg* 个已压到参栈上的参数。

如果这个函数在 LPC 程序中找不到，那么就会在被 *inherit pointer* 指向的继承对象中进行查找（如果函数名是以 `::` 开头的）

需要注意的是，对于 apply 的时候的几个概念：

- current_object 当前在哪个对象上进行 apply
- current_prog 当前在哪个 LPC 程序上进行 apply 一般来说，这个和 current_object.prog 是一样的。但是，当执行的是继承的父对象的时候，就不一样了。这个是 current_prog 表示的是父对象上的 LPC 程序。
- :: 开头的方法，不会在 current_object 进行查找。类似于 super 一样。

## call_create

首先会调用 object 上的 `__init`方法，这个方法总是会在 LPC 对象的最后一个方法。而且这个方法是会自动生成的。最后，在 `apply()`  `create` 方法。

```cpp
void call_create(object_t *ob, int num_arg) {
  /* Be sure to update time first ! */
  set_nextreset(ob);

  call___INIT(ob);

  if (ob->flags & O_DESTRUCTED) {
    pop_n_elems(num_arg);
    return; /* sigh */
  }

  apply(APPLY_CREATE, ob, num_arg, ORIGIN_DRIVER);

  ob->flags |= O_RESET_STATE;
}
```

## call_program

调用一个程序，实际上，就是让 C 来执行 LPC 编译后的代码，从某个地址开始。

```cpp
#define call_program(prog, offset) eval_instruction(prog->program + offset)
```

VM 会有一个堆栈， 一些全局变量。VM 在 C 层执行一个 LPC 程序，实际上，执行的是 LPC 编译过后的字节码。

然后 VM 的解释器，会负责解释这些字节码，进行相关的内存操作，函数调用（C 层）。



# efuns

efuns 是由 驱动层提供的，具体是在 `src/packages` 里面。在编译的时候，会调用 `src/tools/make_func` 来生成相关的文件。对 efuns 的调用，实际上会被编译成为一个索引（操作码？）。然后根据这个索引前往 `efun_table` 找到对应的函数进行调用。

服务器与客户端的通信，在 VM 中，都是通过 某个 `object_t` 对象，找到  `interactivate_t` 对象，然后进行套接字的读写的。

我们创作的核心其实就是利用提供的 efuns 来编写我们自己的函数来实现各种功能。

我们可以看一下一些关键的 efuns 。

## add_action

这个函数，实际上就是将一个命令，与一个特定的 LPC 对象中的方法进行关联，形成一个 sentence_t。

```c
void add_action( string | function fun, string | string * cmd);
void add_action( string | function fun, string | string * cmd, int flag );
```

按照文档上的说明：

>当玩家键入的指令匹配 `cmd` 时，呼叫局部函数 `fun`，玩家指令的参数会做为字符串参数传给函数 `fun`，如果指令错误必须返回0，否则 `fun` 必须返回1。

> 如果第二个参数 `cmd` 是一个数组，所有在数组中的指令都会呼叫函数 `fun`，你可以使用 query_verb() 外部函数找到呼叫函数的指令。

> 如果指令错误，会继续查找其它命令，直到返回 true 或错误信息给玩家。

> 通常 add_action() 只会从 init() apply 方法中调用，而且定义命令的对象必须是玩家可以接触到的，或者是玩家本身，或者被玩家携带，或者是玩家所在的房间，或者是玩家所在房间中的其它对象。

> 如果不指定参数 `flag`，默认为0，代表输入指令必须完全匹配 `cmd` 才可生效，如果参数 `flag` 大于 0，只需玩家输入的指令前面部分匹配 `cmd` 即可生效。其中如果 `flag` 是 1 时 query_verb() 会返回输入的完整指令，如果参数 `flag` 是 2 时 query_verb() 会返回匹配部分以后的内容，而且这两种模式下获取的参数也不一样。

再看一下其 C 代码的实现：

```cpp
// add_action.cc
void f_add_action(void) {
  LPC_INT flag;

  if (st_num_arg == 3) {
    flag = (sp--)->u.number;
  } else {
    flag = 0;
  }

  if (sp->type == T_ARRAY) {
    int i, n = sp->u.arr->size;
    svalue_t *sv = sp->u.arr->item;

    for (i = 0; i < n; i++) {
      if (sv[i].type == T_STRING) {
        add_action(sp - 1, sv[i].u.string, flag & 3);
      }
    }
    free_array((sp--)->u.arr);
  } else {
    add_action((sp - 1), sp->u.string, flag & 3);
    free_string_svalue(sp--);
  }
  pop_stack();
}
```

第三个参数，可以没有，默认就是 0。

第二个参数如果是数组，那么就会添加多个 action去。

最终的实际实现如下：

```cpp
// add_action.cc
static void add_action(svalue_t *str, const char *cmd, int flag) {
  sentence_t *p;
  object_t *ob;

  if (current_object->flags & O_DESTRUCTED) {
    return;
  }
  ob = current_object;
#ifndef NO_SHADOWS
  while (ob->shadowing) {
    ob = ob->shadowing;
  }
  /* don't allow add_actions of a static function from a shadowing object */
  if ((ob != current_object) && str->type == T_STRING && is_static(str->u.string, ob)) {
    return;
  }
#endif
  if (command_giver == nullptr || (command_giver->flags & O_DESTRUCTED)) {
    return;
  }
  if (ob != command_giver
#ifndef NO_ENVIRONMENT
      && ob->super != command_giver && ob->super != command_giver->super &&
      ob != command_giver->super
#endif
  ) {
    return;
  } /* No need for an error, they know what they
     * did wrong. */
  p = alloc_sentence();
  if (str->type == T_STRING) {
    debug(add_action, "--Add action '%s' (ob: %s func: '%s')\n", cmd, ob->obname, str->u.string);
    p->function.s = make_shared_string(str->u.string);
    p->flags = flag;
  } else {
    debug(add_action, "--Add action '%s' (ob: %s func: <function>)\n", cmd, ob->obname);

    p->function.f = str->u.fp;
    str->u.fp->hdr.ref++;
    p->flags = flag | V_FUNCTION;
  }
  p->ob = ob;
  p->verb = make_shared_string(cmd);
  /* This is ok; adding to the top of the list doesn't harm anything */
  p->next = command_giver->sent;
  command_giver->sent = p;
}

```

关键的地方在于，分配了一个 `sentence_t` 结构，设置此结构中的 `function` 字段为一个字符串，或者是一个函数；设置 `sentence_t` 结构的 LPC 对象，动词等；最后，将此结构放到函数调用者的 `sentence_t` 链表中。

在解析用户的命令，`user_parser()` 中就会遍历这个链表，然后进行 verb 的匹配。注意上面提到细节上，可以进行全局匹配的问题。
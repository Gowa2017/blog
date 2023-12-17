---
title: FluffOS中VM中对象的建立过程
categories:
  - MUD
date: 2020-05-11 17:09:01
updated: 2020-05-11 17:09:01
tags: 
  - MUD
---
前一篇文章 {% post_link FluffOS的简单架构分析 FluffOS的简单架构分析 %} 大概看了一下驱动层面的实现，现在来看一下具体一个虚拟机对象的建立过程

<!--more-->

# load_object

这是一个虚拟机内部的函数。

`load_object(const * char lname, int callcreate)`

一切的开始。从一个 LPC 文件，来构造对象。

- lname 代表了文件名
- callcreate 表示在加载了文件后是否需要调用对象的 `create` 方法。

究其过程而言，大体分为：

1. 读取文件内容。
2. 解释-编译文件为 LPC 程序 
3. 继承文件加载判断
4. 构造  object_t 数据结构
5. 新对象放入 ObjectTable 内
6. 新对象压入虚拟机
7. 调用  master_ob 来检查对象的有效性
8. 初始化对象，调用 LPC 对象的 `create`。

```cpp
// vm/internal/simulate.cc

object_t *load_object(const char *lname, int callcreate) {
  ScopedTracer _tracer("LPC Load Object", EventCategory::VM_LOAD_OBJECT, {lname});

  auto inherit_chain_size = CONFIG_INT(__INHERIT_CHAIN_SIZE__);

  int f;
  program_t *prog;
  object_t *ob;
  svalue_t *mret;
  struct stat c_st;
  char real_name[400], name[400], actualname[400], obname[400];

  const char *pname = check_valid_path(lname, master_ob, "load_object", 0);
  if (!pname) {
    error("Read access denied.\n");
  }
  if (++num_objects_this_thread > inherit_chain_size) {
    error("Inherit chain too deep: > %d when trying to load '%s'.\n", inherit_chain_size, lname);
  }
#ifdef PACKAGE_UIDS
  if (current_object && current_object->euid == nullptr) {
    error("Can't load objects when no effective user.\n");
  }
#endif
  if (strrchr(pname, '#')) {
    error("Cannot load a clone.\n");
  }
  if (!filename_to_obname(lname, name, sizeof name)) {
    error("Filenames with consecutive /'s in them aren't allowed (%s).\n", lname);
  }
  if (!filename_to_obname(pname, actualname, sizeof actualname)) {
    error("Filenames with consecutive /'s in them aren't allowed (%s).\n", pname);
  }

  /*
   * First check that the c-file exists.
   */
  (void)strcpy(real_name, actualname);
  (void)strcat(real_name, ".c");

  (void)strcpy(obname, name);
  (void)strcat(obname, ".c");

  if (stat(real_name, &c_st) == -1 || S_ISDIR(c_st.st_mode)) {
    save_command_giver(command_giver);
    ob = load_virtual_object(actualname, 0);
    restore_command_giver();
    num_objects_this_thread--;
    return ob;
  }
  /*
   * Check if it's a legal name.
   */
  if (!legal_path(real_name)) {
    debug_message("Illegal pathname: /%s\n", real_name);
    error("Illegal path name '/%s'.\n", real_name);
  }

  f = open(real_name, O_RDONLY);
#ifdef _WIN32
  // TODO: change everything to use fopen instead.
  _setmode(f, _O_BINARY);
#endif
  if (f == -1) {
    debug_perror("compile_file", real_name);
    error("Could not read the file '/%s'.\n", real_name);
  }
  save_command_giver(command_giver);
  prog = compile_file(f, obname);
  restore_command_giver();
  update_compile_av(total_lines);
  total_lines = 0;
  close(f);

  /* Sorry, can't handle objects without programs yet. */
  if (inherit_file == nullptr && (num_parse_error > 0 || prog == nullptr)) {
    if (num_parse_error == 0 && prog == nullptr) {
      error("No program in object '/%s'!\n", name);
    }

    if (prog) {
      free_prog(&prog);
    }
    error("Error in loading object '/%s'\n", name);
  }
  /*
   * This is an iterative process. If this object wants to inherit an
   * unloaded object, then discard current object, load the object to be
   * inherited and reload the current object again. The global variable
   * "inherit_file" will be set by lang.y to point to a file name.
   */
  if (inherit_file) {
    object_t *inh_obj;
    char inhbuf[MAX_OBJECT_NAME_SIZE];

    if (!filename_to_obname(inherit_file, inhbuf, sizeof inhbuf)) {
      strcpy(inhbuf, inherit_file);
    }
    FREE(inherit_file);
    inherit_file = nullptr;

    if (prog) {
      free_prog(&prog);
      prog = nullptr;
    }
    if (strcmp(inhbuf, name) == 0) {
      error("Illegal to inherit self.\n");
    }

    if ((inh_obj = ObjectTable::instance().find(inhbuf))) {
#ifdef DEBUG
      fatal("Inherited object is already loaded!");
#endif
    } else {
      inh_obj = load_object(inhbuf, 1);
    }
    if (!inh_obj) error("Inherited file '/%s' does not exist!\n", inhbuf);

    /*
     * Yes, the following is necessary.  It is possible that when we
     * loaded the inherited object, it loaded this object from it's
     * create function. Without this check, that would crash the driver.
     * -Beek
     */
    if (!(ob = ObjectTable::instance().find(name))) {
      ob = load_object(name, 1);
      /* sigh, loading the inherited file removed us */
      if (!ob) {
        num_objects_this_thread--;
        return nullptr;
      }
      ob->load_time = get_current_time();
    }
    num_objects_this_thread--;
    return ob;
  }
  ob = get_empty_object(prog->num_variables_total);
  /* Shared string is no good here */
  SETOBNAME(ob, alloc_cstring(name, "load_object"));
  SET_TAG(ob->obname, TAG_OBJ_NAME);
  ob->prog = prog;
  ob->flags |= O_WILL_RESET; /* must be before reset is first called */
  ob->next_all = obj_list;
  ob->prev_all = nullptr;
  if (obj_list) {
    obj_list->prev_all = ob;
  }
  obj_list = ob;
  ObjectTable::instance().insert(ob->obname, ob); /* add name to fast object lookup table */
  save_command_giver(command_giver);
  push_object(ob);
  mret = apply_master_ob(APPLY_VALID_OBJECT, 1);
  if (mret && !MASTER_APPROVED(mret)) {
    destruct_object(ob);
    error("master object: %s() denied permission to load '/%s'.\n",
          applies_table[APPLY_VALID_OBJECT], name);
  }

  if (init_object(ob) && callcreate) {
    call_create(ob, 0);
  }
  if (!(ob->flags & O_DESTRUCTED) && function_exists(APPLY_CLEAN_UP, ob, 1)) {
    ob->flags |= O_WILL_CLEAN_UP;
  }
  restore_command_giver();

  if (ob) {
    debug(d_flag, "--/%s loaded.\n", ob->obname);
  }

  ob->load_time = get_current_time();
  num_objects_this_thread--;

  return ob;
}
```

## 文件加载

这部分包括了几个过程：

1. 权限检查
2. 继承链检查
3. 是否复制对象
4. 生成对象名
5. 路径是目录，那么就加载虚拟对象
6. 路径的合法性检查
7. 文件读取



## 权限检查

这是通过 `file.cc` 中的 `check_valid_path` 函数来实现的：

> 其用来检查一个文件是否可以进行读或者写。
>
> 具体的实现，则是由 master_ob 中的函数来处理。
>
> 文件路径总是会被当成绝对路径，返回的时候会去掉开头的 `/`
>
> 如果路径是 `/` 那么就返回 `.`
>
> 否则的话，返回的路径将是由 `apply()` 临时分配在内存空间的，在下一次 `apply()` 的时候会被析构。

```cpp
check_valid_path(lname, master_ob, "load_object", 0);
```

认真看一下其检查逻辑：

1. 如果 master_ob 和函数指定的 call_object 都为空的时候，实际上就是直接返回一个路径的复制
2. 如果 master_ob 不为空，然后就将路径压入栈，再将 call_obj 压入栈，再将调用的函数 压入栈。
3. apply master_ob 上的  'APPLY_VALID_WRITE/APPLY_VALID_READ' 方法。
4. 如果检查通过，实际上和第 1 个步骤返回的值是一样的。

```cpp
// file.cc
const char *check_valid_path(const char *path, object_t *call_object, const char *const call_fun,
                             int writeflg) {
  svalue_t *v;

  if (!master_ob && !call_object) {
    // early startup, ignore security
    free_svalue(&apply_ret_value, "check_valid_path");
    apply_ret_value.type = T_STRING;
    apply_ret_value.subtype = STRING_MALLOC;
    path = apply_ret_value.u.string = string_copy(path, "check_valid_path");
    return path;
  }

  if (call_object == nullptr || call_object->flags & O_DESTRUCTED) {
    return nullptr;
  }

  copy_and_push_string(path);
  push_object(call_object);
  push_constant_string(call_fun);
  // 检查文件的可读可写性
  if (writeflg) {
    v = apply_master_ob(APPLY_VALID_WRITE, 3);
  } else {
    v = apply_master_ob(APPLY_VALID_READ, 3);
  }

  if (v == (svalue_t *)-1) {
    v = nullptr;
  }

  if (v && v->type == T_NUMBER && v->u.number == 0) {
    return nullptr;
  }
  // 如果返回了路径，那么就直接使用
  if (v && v->type == T_STRING) {
    path = v->u.string;
  // 否则就动态的生成一个路径
  } else {
    extern svalue_t apply_ret_value;

    free_svalue(&apply_ret_value, "check_valid_path");
    apply_ret_value.type = T_STRING;
    apply_ret_value.subtype = STRING_MALLOC;
    path = apply_ret_value.u.string = string_copy(path, "check_valid_path");
  }

  if (path[0] == '/') {
    path++;
  }
  if (path[0] == '\0') {
    path = ".";
  }
  if (legal_path(path)) {
    return path;
  }

  return nullptr;
}
```

## 克隆对象

对于文件名中含有  `#` 的，被认为是一个克隆品，不可进行加载。



## 对象名

一般我们会用类似 `/adm/obj/master` 这样的文件来加载对象，不过在使用的时候，一般会去掉开头的那个 `/`

前面一个路径会被程 `lname`，后一个叫做 `pname`。

最终我们的对象的 `obname` 将会是绝对路径那个，`lname`。

| 变量       | 值               | 来源                      |
| ---------- | ---------------- | ------------------------- |
| lname      | /adm/obj/master  | 指定                      |
| pname      | adm/obj/master   | 从 lname 计算             |
| name       | adm/obj/master   | filename_to_obname(lname) |
| actualname | adm/obj/master   | filename_to_obname(pname) |
| real_name  | adm/obj/master.c | actualname                |
| obname     | adm/obj/master.c | name                      |

其中  real_name 用来打开文件

obname 用来给编译器使用

name 用来给 设置 ob->obname，同时也会作为 ObjectTable 中的索引。这里暂时没有发现这几个有差异的地方。

## 编译

当我们将文件读入到内存中，就进行要进行编译了。所以的编译，我认为，实际上是因为对于 LPC 文件，其只是一些符号的标记，最终还是要通过解释器解释后，调用 C 层的函数来进行实现功能。

```cpp
program_t *compile_file(int f, char *name) {
  int yyparse(void);
  static int guard = 0;
  program_t *prog;
  extern int func_present;

  /* The parser isn't reentrant.  On a few occasions (compile
   * errors, valid_override) LPC code is called during compilation,
   * causing the possibility of arriving here again.
   */
  if (guard || current_file) {
    error("Object cannot be loaded during compilation.\n");
  }
  guard = 1;

  {
    prolog(f, name);
    func_present = 0;
    yyparse();
    prog = epilog();
  }

  guard = 0;
  return prog;
}
```

最终返回的是将是一个编译后的 LPC 程序。

在这个过程中，我们的如果在解析的过程中，发现了文件有继承其他文件，那么就会导致 全局变量 `inherit_file` 被置 1。

如果 inherit_file 并没有加载进 ObjectTable 中，就会释放已加载的内容，先加载继承的文件，再重新加载一次。

然后再开始分配内存对象。

## 分配 object_t 结构

```cpp
  ob = get_empty_object(prog->num_variables_total);
  /* Shared string is no good here */
  SETOBNAME(ob, alloc_cstring(name, "load_object"));
  SET_TAG(ob->obname, TAG_OBJ_NAME);
  ob->prog = prog;
  ob->flags |= O_WILL_RESET; /* must be before reset is first called */
  ob->next_all = obj_list;
  ob->prev_all = nullptr;
  if (obj_list) {
    obj_list->prev_all = ob;
  }
  obj_list = ob;
  ObjectTable::instance().insert(ob->obname, ob); /* add name to fast object lookup table */
  save_command_giver(command_giver);
  push_object(ob);
  mret = apply_master_ob(APPLY_VALID_OBJECT, 1);
  if (mret && !MASTER_APPROVED(mret)) {
    destruct_object(ob);
    error("master object: %s() denied permission to load '/%s'.\n",
          applies_table[APPLY_VALID_OBJECT], name);
  }

```

这个数据结构归 Driver 管理，同时其会引用已编译的 LPC 程序，最后，再将此结构，压入到 VM 中去。

然后再根据参数决定是否要调用 `create` 方法，这个方法会调用 object_t 的 `__INIT` 方法，再调用 LPC　中的　`create` 方法。

这样就加载完成了。






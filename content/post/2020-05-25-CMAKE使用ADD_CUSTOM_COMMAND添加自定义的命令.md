---
title: CMAKE使用ADD_CUSTOM_COMMAND添加自定义的命令
categories:
  - Linux/Unix
date: 2020-05-25 22:41:36
updated: 2020-05-25 22:41:36
tags: 
  - Linux/Unix
---

这个问题源于在看 `Fluffos` 的编译过程中，实在是很难发现一下中间自动生成的文件，最终仔细看了CMAKELIST文件，才发现来源。这主要再于 `add_custom_command` 这么一个命令

<!--more-->

根据 [官方文档](https://cmake.org/cmake/help/latest/command/add_custom_command.html)，其是用来为构建系统添加自定义的构建规则其有两种用法：

1. 用来生成特定的文件
2. 对一个构建目标添加一个命令。



# 生成文件

```cmake
add_custom_command(OUTPUT output1 [output2 ...]
                   COMMAND command1 [ARGS] [args1...]
                   [COMMAND command2 [ARGS] [args2...] ...]
                   [MAIN_DEPENDENCY depend]
                   [DEPENDS [depends...]]
                   [BYPRODUCTS [files...]]
                   [IMPLICIT_DEPENDS <lang1> depend1
                                    [<lang2> depend2] ...]
                   [WORKING_DIRECTORY dir]
                   [COMMENT comment]
                   [DEPFILE depfile]
                   [JOB_POOL job_pool]
                   [VERBATIM] [APPEND] [USES_TERMINAL]
                   [COMMAND_EXPAND_LISTS])
```

这样，定义了一个命令，用来生成 *OUTPUT* 文件。任何与 *CMakeList.txt*  文件处于同级目录中建立的目标，如果指定了任意一个此自定义命令的输出作为源文件，都会自动添加一个规则：我们自定义的这个命令在构建时来生成文件。

几个需要注意的地方：

- **COMMAND** 这个是用来指定需要执行的命令行工具。指定多个的话会按序执行。如果这个命令指定是一个构建后的可执行目标名字（通过 `add_executable()` 来建立），那么将会被自动的为构建时的可执行文件的位置（只需要满足下面的条件）。

  - 目标不是交叉编译的。**CMAKE_CROSSCOMPILING** 不为 True
  - 目标被交叉编译，但是提供了一个仿真。在这样的情况下，**[`CROSSCOMPILING_EMULATOR`](https://cmake.org/cmake/help/latest/prop_tgt/CROSSCOMPILING_EMULATOR.html#prop_tgt:CROSSCOMPILING_EMULATOR)** 的内容会被放在命令之前。

  如果不满足上面的条件，那么就会假设这个命令是放在 *PATH* 里面。

# 例如


```cmake
add_subdirectory(tools)

# Generated sources

set(TOOL_SRC
  "tools/build_applies.cc"
  "tools/make_func.cc"
  )

set(GENERATRED_SOURCE
  "${CMAKE_CURRENT_BINARY_DIR}/packages.fullspec"
  "${CMAKE_CURRENT_BINARY_DIR}/applies_table.autogen.cc"
  "${CMAKE_CURRENT_BINARY_DIR}/applies_table.autogen.h"
  "${CMAKE_CURRENT_BINARY_DIR}/efuns.autogen.cc"
  "${CMAKE_CURRENT_BINARY_DIR}/efuns.autogen.h"
  "${CMAKE_CURRENT_BINARY_DIR}/options.autogen.h"
  )

add_custom_command(
  OUTPUT "applies_table.autogen.cc" "applies_table.autogen.h"
  COMMAND "build_applies" "${CMAKE_CURRENT_SOURCE_DIR}"
  DEPENDS "vm/internal/applies"
)

add_custom_command(
  OUTPUT "packages.fullspec"
  COMMAND "${CMAKE_CURRENT_SOURCE_DIR}/tools/build_packages_genfiles.sh" "${CMAKE_CURRENT_SOURCE_DIR}/../" "${CMAKE_CXX_COMPILER}" "-I${CMAKE_CURRENT_BINARY_DIR}" "-I${CMAKE_CURRENT_SOURCE_DIR}"
  DEPENDS "tools/build_packages_genfiles.sh" "base/internal/options_incl.h"
)
file(GLOB ALL_SPEC_FILES files "packages/*/*.spec")
foreach (file ${ALL_SPEC_FILES})
  add_custom_command(OUTPUT "packages.fullspec"
          DEPENDS "${file}" APPEND)
endforeach ()

add_custom_command(
  OUTPUT "efuns.autogen.cc" "efuns.autogen.h"
  COMMAND "make_func" "${CMAKE_CURRENT_BINARY_DIR}/packages.fullspec"
  DEPENDS "make_func" "packages.fullspec"
)
add_custom_command(
  OUTPUT "options.autogen.h"
  COMMAND "make_options_defs" "-I${CMAKE_CURRENT_BINARY_DIR}" "-I${CMAKE_CURRENT_SOURCE_DIR}" "${CMAKE_CURRENT_SOURCE_DIR}/base/internal/options_incl.h"
  DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/base/internal/options_incl.h"
)

add_custom_command(
  OUTPUT "grammar.autogen.y"
  COMMAND "${CMAKE_CURRENT_SOURCE_DIR}/tools/make_grammar.sh" "${CMAKE_CURRENT_SOURCE_DIR}/../" "${CMAKE_CURRENT_SOURCE_DIR}/compiler/internal/grammar.y.pre" "${CMAKE_CXX_COMPILER}" "-I${CMAKE_CURRENT_BINARY_DIR}" "-I${CMAKE_CURRENT_SOURCE_DIR}"
  DEPENDS "tools/make_grammar.sh" "compiler/internal/grammar.y.pre"
)

find_package(BISON REQUIRED)
BISON_TARGET(Grammar ${CMAKE_CURRENT_BINARY_DIR}/grammar.autogen.y ${CMAKE_CURRENT_BINARY_DIR}/grammar.autogen.cc
  DEFINES_FILE "${CMAKE_CURRENT_BINARY_DIR}/grammar.autogen.h")

set(GENERATRED_GRAMMAR_FILES
  "${CMAKE_CURRENT_BINARY_DIR}/grammar.autogen.y"
  "${BISON_Grammar_OUTPUT_SOURCE}"
  "${BISON_Grammar_OUTPUT_HEADER}"
  )

set(SRC
  "backend.cc"
  "comm.cc"
  "mainlib.cc"
  "user.cc"
  "net/telnet.cc"
  "net/websocket.cc"
  "net/ws_ascii.cc"
  "base/internal/debugmalloc.cc"
  "base/internal/external_port.cc"
  "base/internal/file.cc"
  "base/internal/hash.cc"
  "base/internal/log.cc"
  "base/internal/md.cc"
  "base/internal/outbuf.cc"
  "base/internal/port.cc"
  "base/internal/rc.cc"
  "base/internal/rusage.cc"
  "base/internal/stats.cc"
  "base/internal/stralloc.cc"
  "base/internal/strput.cc"
  "base/internal/strutils.cc"
  "base/internal/tracing.cc"
  "compiler/internal/compiler.cc"
  "compiler/internal/disassembler.cc"
  "compiler/internal/generate.cc"
  "compiler/internal/icode.cc"
  "compiler/internal/lex.cc"
  "compiler/internal/scratchpad.cc"
  "compiler/internal/trees.cc"
  "vm/internal/base/apply_cache.cc"
  "vm/internal/base/array.cc"
  "vm/internal/base/buffer.cc"
  "vm/internal/base/class.cc"
  "vm/internal/base/function.cc"
  "vm/internal/base/interpret.cc"
  "vm/internal/base/mapping.cc"
  "vm/internal/base/object.cc"
  "vm/internal/base/program.cc"
  "vm/internal/base/svalue.cc"
  "vm/internal/apply.cc"
  "vm/internal/eval_limit.cc"
  "vm/internal/posix_timers.cc"
  "vm/internal/master.cc"
  "vm/internal/otable.cc"
  "vm/internal/simul_efun.cc"
  "vm/internal/simulate.cc"
  "vm/internal/trace.cc"
  "vm/internal/vm.cc"
  )

set(TEST_SRC
  "vm/internal/otable_test.cc"
  )

file(GLOB_RECURSE SRC_HEADER
  "*.h"
  )

set_source_files_properties(${GENERATED_HEADERS} ${GENERATRED_SOURCE}
  ${GENERATRED_GRAMMAR_FILES} PROPERTIES GENERATED TRUE)

add_library(autogen ${GENERATED_HEADERS} ${GENERATRED_SOURCE}
  ${GENERATRED_GRAMMAR_FILES})
add_dependencies(autogen make_options_defs)

```

在上面的文件中，首先会添加 `tools` 目录下面的 *Cmake* 文件，在后续又定义了几个自定义的命令 `build_applies, make_func`。世界上，这两个命令的可执行文件就是在 `tools` 里面进行编译出来的。

```cmake
# Make Func
find_package(BISON REQUIRED)
BISON_TARGET(MakeFunc make_func.y ${CMAKE_CURRENT_BINARY_DIR}/make_func.cc)
add_executable(make_func ${BISON_MakeFunc_OUTPUT_SOURCE})

#build_applies
add_executable(build_applies "build_applies.cc")

add_executable(make_options_defs "make_options_defs.cc" "preprocessor.hpp")
```

这里需要注意的就是，make_func 命令是通过  BISON 语法来解析相应的 yacc 文件，然后编译出来的。

如此，就能理解这些中间命令的用法了。



其中 ，对于我们的 

```cmake
set(GENERATRED_SOURCE
  "${CMAKE_CURRENT_BINARY_DIR}/packages.fullspec"
  "${CMAKE_CURRENT_BINARY_DIR}/applies_table.autogen.cc"
  "${CMAKE_CURRENT_BINARY_DIR}/applies_table.autogen.h"
  "${CMAKE_CURRENT_BINARY_DIR}/efuns.autogen.cc"
  "${CMAKE_CURRENT_BINARY_DIR}/efuns.autogen.h"
  "${CMAKE_CURRENT_BINARY_DIR}/options.autogen.h"
  )

add_custom_command(
  OUTPUT "efuns.autogen.cc" "efuns.autogen.h"
  COMMAND "make_func" "${CMAKE_CURRENT_BINARY_DIR}/packages.fullspec"
  DEPENDS "make_func" "packages.fullspec"
)

add_library(autogen ${GENERATED_HEADERS} ${GENERATRED_SOURCE}
  ${GENERATRED_GRAMMAR_FILES})
add_dependencies(autogen make_options_defs)
```

其用到了我们的自定义命令 `make_func` 生成的源文件，那么在其构建规则中，肯定会有一条对于 `make_func` 命令的调用。

我们查看一下 *autogen* 的编译规则 *CMakeFiles/Makefile2 src/CMakeFiles/autogen.dir/rule*：

```make
src/efuns.autogen.cc: src/tools/make_func
src/efuns.autogen.cc: src/packages.fullspec
	@$(CMAKE_COMMAND) -E cmake_echo_color --switch=$(COLOR) --blue --bold --progress-dir=/Users/gowa/Repo/fluffos/build/CMakeFiles --progress-num=$(CMAKE_PROGRESS_3) "Generating efuns.autogen.cc, efuns.autogen.h"
	cd /Users/gowa/Repo/fluffos/build/src && tools/make_func /Users/gowa/Repo/fluffos/build/src/packages.fullspec

```

确实是可以找到相关的规则定义的。同时也可以看到，*make_func* 被自动的替换成了我们在构建 `tools/make_func` 时的路径。
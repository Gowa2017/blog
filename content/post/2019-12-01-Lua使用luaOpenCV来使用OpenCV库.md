---
title: Lua使用luaOpenCV来使用OpenCV库
categories:
  - Lua
date: 2019-12-01 21:47:44
updated: 2019-12-01 21:47:44
tags: 
  - Lua
  - OpenCV
  - LuaBinding
  - Kaguya
---
Python 使用 OpenCV 是非常简单的，但是 Lua 使用起来就没有这么容易了，概因为没有人来维护这个东西，没有现程可以用的模块。究其原理，也就是将 OpenCV 库进行一下 LuaBinding 然后就可以在 Lua 虚拟机中使用了，在 GitHub 上就看到了 [https://github.com/satoren/luaOpenCV](https://github.com/satoren/luaOpenCV) 这么一个库。

<!--more-->
虽然说是可用的，但是其过程也是比较困难的，需要使用 Cmake 等编译工具链进行自行编译，这个环境都会出现不小的问题。

# 项目指引

## 前提条件

- Lua 5.1 to 5.3 (recommended: 5.3)
- C++11 compiler(gcc 4.8+,clang 3.4+,MSVC2015).
- CMake 2.8 or later

## 编译步骤(macOS)

### 初始化第三库依赖
```sh
https://github.com/satoren/luaOpenCV.git
git submodule update --init --recursive
```
这里说明一下，因为 luaOpenCV 需要将 OpenCV 库进行绑定，所以其首先肯定要将 OpenCV 源代码拷贝过来了。所以其第三方依赖中就有 OpenCV 项目。

我们这样就会将所有的第三方 submodule 都给拉到本地来了。依赖有三个：

- [OpenCV](https://github.com/opencv/opencv) 这个就不说了，这个依赖的是当时一个特定的版本分支
- [kaguya ](https://github.com/satoren/kaguya)一个 C++ 到 Lua 的绑定。
- [Google Test](https://github.com/google/googletest) 谷歌的 C++ 测试框架。


### 编译 OpenCV 本地库

```sh
cd third_party/opencv
mkdir build
cd build
cmake ../ -DCMAKE_INSTALL_PREFIX=../../opencvlib -DCMAKE_BUILD_TYPE=Release -DBUILD_SHARED_LIBS=Off
cmake --build . 
cmake --build . --target install
```

### 编译 luaOpenCV 库

```sh
cd ../../.. # return to root of source tree
mkdir build
cd build
cmake ../ -DCMAKE_BUILD_TYPE=Release
cmake --build .
```

### demo

```sh
 lua samples/hello_opencv.lua
```


# luaOpenCV Cmakefile 文件解析

我们在编译 OpenCV 的时候指定了几个变量：

```sh
cmake ../ -DCMAKE_INSTALL_PREFIX=../../opencvlib -DCMAKE_BUILD_TYPE=Release -DBUILD_SHARED_LIBS=Off
```

其中：  

- `CMAKE_INSTALL_PREFIX` 指定将我们的 OpenCV 静态库安装到  thirdl_party/opencvlib 目录下（包括头文件，静态库等）
- `DBUILD_SHARED_LIBS` 指明，我们在编译 OpenCV 的时候，不要生成共享库，而是生成静态库，方便我们后面编译的时候进行链接。

下面是 cmake 文件

```mk
cmake_minimum_required(VERSION 2.8)
project(luaOpenCV)



option(BUILD_TESTS "Builds the unit test" TRUE)
# 指定 lua 源码目录
option(LOCAL_LUA_DIRECTORY "local lua library directory" "third_party/lua-5.3.3")
set(LOCAL_LUA_DIRECTORY "third_party/lua-5.3.3")

# 查找 lua 库，如果找不到我们指定的路径这就起作用了
include(cmake/FindLua.cmake)

# 前缀路径，会在 CMAKE_PREFIX_PATH 内查找 openCV 
set(CMAKE_PREFIX_PATH third_party/opencvlib)
FIND_PACKAGE(OpenCV 3.1.0 REQUIRED)

# 包含 lua 头文件及静态库文件
include_directories(BEFORE ${LUA_INCLUDE_DIRS})
link_directories(${LUA_LIBRARY_DIRS})

# 包含 kaguya 头文件
include_directories(third_party/kaguya/include)
# 包含 OpenCV 头文件
include_directories(BEFORE ${OpenCV_INCLUDE_DIRS})


if(MSVC)
 set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /bigobj")
else(MSVC)
 set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
endif(MSVC)


# luaOpenCV 的源文件其实就两个
set(LUA_OPENCV_SRCS src/raw_bind.cc src/manual_bind.cc)
add_library(cv SHARED ${LUA_OPENCV_SRCS})
# 将 opencv 静态库， lua 静态库链接到我们的代码内
target_link_libraries(cv ${OpenCV_LIBS} ${LUA_LIBRARIES})
```

我们重点来看看这 luaOpenCV 的两个代码文件，因为其是用 kaguya 来进行 C++ 绑定的，看明白了这个就很简单了。

# Kaguya
要说到 Kaguya ，就要说到 LuaBinding，就不能不提 [lua_CFunction](https://cloudwu.github.io/lua53doc/manual.html#lua_CFunction)，所有想要给 Lua 进行调用的 函数（方法） 都必须是 lua_CFunction 格式，**也就是说，接受一个 lua_State 作为参数，返回一个整型值（代表了返回值的个数）**。

那么，我们 Kaguya 就必须能将所有的 C++ 类的静态方法，类对象方法都转换成这种格式。

它是怎么做的呢？

在我们的 raw_bind.cc 和  manual_bind.cc 里面，其实都指向了一个地方，我们来看一个函数 putText

## putText

在 OpenCV 中 putText 的定义：

```cpp
void putText( InputOutputArray _img, const String& text, Point org,
              int fontFace, double fontScale, Scalar color,
              int thickness, int line_type, bool bottomLeftOrigin )

{
// ... ...
}
```

而我们的绑定代码则是如下，指定了函数的绑定后的名称，函数的地址，最小参数数量，最大参数数量，函数签名。这个宏，会生成我们要绑定函数的包装函数。

```cpp
namespace gen_wrap_cv {
KAGUYA_FUNCTION_OVERLOADS_WITH_SIGNATURE(putText_wrap_obj, cv::putText, 6, 9, void (*)(InputOutputArray, const cv::String &, Point, int, double, Scalar, int, int, bool));
auto putText = putText_wrap_obj();
}
```

按着宏往下追，我们看到，这个时候，会以我们传递的函数签名我类型参数，调用 create<T>() 方法。

```cpp
// kaguya/native_function.hpp 

/// @brief Generate wrapper function object for count based overloads with nonvoid return function. Include default arguments parameter function
/// @param GENERATE_NAME generate function object name
/// @param FNAME target function name
/// @param MINARG minimum arguments count
/// @param MAXARG maximum arguments count
/// @param SIGNATURE function signature. e,g, int(int)
#define KAGUYA_FUNCTION_OVERLOADS_WITH_SIGNATURE(GENERATE_NAME,FNAME, MINARG, MAXARG,SIGNATURE) KAGUYA_FUNCTION_OVERLOADS_INTERNAL(GENERATE_NAME,FNAME, MINARG, MAXARG,create<SIGNATURE>())

// 这个针对的是成员函数的
/// @brief Generate wrapper function object for count based overloads with nonvoid return function. Include default arguments parameter function
/// @param GENERATE_NAME generate function object name
/// @param CLASS target class name
/// @param FNAME target function name
/// @param MINARG minimum arguments count
/// @param MAXARG maximum arguments count
#define KAGUYA_MEMBER_FUNCTION_OVERLOADS(GENERATE_NAME,CLASS,FNAME, MINARG, MAXARG) KAGUYA_MEMBER_FUNCTION_OVERLOADS_INTERNAL(GENERATE_NAME,CLASS,FNAME, MINARG, MAXARG,create(&CLASS::FNAME))
```

最终是定义了一个结构体，就是我们绑定后的函数名：

```cpp
#define KAGUYA_FUNCTION_OVERLOADS_INTERNAL(GENERATE_NAME,FNAME, MINARG, MAXARG,CREATE_FN)\
struct GENERATE_NAME\
{\
	template<typename F>\
	struct Function : kaguya::OverloadFunctionImpl<F>\
	{\
		typedef typename kaguya::OverloadFunctionImpl<F>::result_type result_type;\
		virtual result_type invoke_type(lua_State *state)\
		{\
			using namespace kaguya::nativefunction;\
			int argcount = lua_gettop(state);\
			KAGUYA_PP_REPEAT_DEF_VA_ARG(KAGUYA_PP_INC(KAGUYA_PP_SUB(MAXARG, MINARG)), KAGUYA_INTERNAL_OVERLOAD_FUNCTION_INVOKE, FNAME, MINARG, MAXARG)\
			throw kaguya::LuaTypeMismatch("argument count mismatch");\
		}\
		virtual int minArgCount()const { return MINARG;	}\
		virtual int maxArgCount()const { return MAXARG; }\
	};\
	template<typename F> kaguya::PolymorphicInvoker::holder_type create(F)\
	{\
		kaguya::OverloadFunctionImpl<F>* ptr = new Function<F>();\
		return kaguya::PolymorphicInvoker::holder_type(ptr);\
	}\
	template<typename F> kaguya::PolymorphicInvoker::holder_type create()\
	{\
		kaguya::OverloadFunctionImpl<F>* ptr = new Function<F>();\
		return kaguya::PolymorphicInvoker::holder_type(ptr);\
	}\
	kaguya::PolymorphicInvoker operator()() { return CREATE_FN; }\
}GENERATE_NAME;
```

当我们调用这个宏定义好的结构：

```cpp
auto putText = putText_wrap_obj();
```

实际调用的是：

```cpp
	template<typename F> kaguya::PolymorphicInvoker::holder_type create()\
	{\
		kaguya::OverloadFunctionImpl<F>* ptr = new Function<F>();\
		return kaguya::PolymorphicInvoker::holder_type(ptr);\
	}
```

这会返回一个 Function 的实现，然后将其放到一个 PolymorphicInvoker 中进行返回。

最终，我们会将这函数注册到 Lua 内去。

##  KAGUYA_BINDINGS

```cpp
KAGUYA_BINDINGS(cv) {
using namespace kaguya;
  function("addText", gen_wrap_cv::addText);
  function("putText", gen_wrap_cv::putText);
  
```

如果是类：

```cpp
  class_<cv::RotatedRect>("RotatedRect")
    .constructors<void (),void (const Point2f &, const Size2f &, float),void (const Point2f &, const Point2f &, const Point2f &)>()
    .function("boundingRect", gen_wrap_cv::gen_wrap_RotatedRect::boundingRect)
    .function("points", gen_wrap_cv::gen_wrap_RotatedRect::points)
    .property("center", &cv::RotatedRect::center)
    .property("size", &cv::RotatedRect::size)
    .property("angle", &cv::RotatedRect::angle)
  ;
```

我们看一下我们这个绑定宏都干了什么：

```cpp
#define KAGUYA_BINDINGS(MODULE_NAME)\
void KAGUYA_PP_CAT(kaguya_bind_internal_,MODULE_NAME)();\
KAGUYA_EXPORT int KAGUYA_PP_CAT(luaopen_, MODULE_NAME)(lua_State* L) { return kaguya::detail::bind_internal(L,&KAGUYA_PP_CAT(kaguya_bind_internal_, MODULE_NAME)); }\
void KAGUYA_PP_CAT(kaguya_bind_internal_, MODULE_NAME)()
```

我们将他展开就是：

```cpp
void kaguya_bind_internal_cv();
extern "C" __attribute__((visibility("default"))) int luaopen_cv(lua_State* L) {
return kaguya::detail::bind_internal(L,&kaguya_bind_internal_cv); }
void kaguya_bind_internal_cv(){
using namespace kaguya;
  function("addText", gen_wrap_cv::addText);
  function("putText", gen_wrap_cv::putText);
  class_<cv::RotatedRect>("RotatedRect")
    .constructors<void (),void (const Point2f &, const Size2f &, float),void (const Point2f &, const Point2f &, const Point2f &)>()
    .function("boundingRect", gen_wrap_cv::gen_wrap_RotatedRect::boundingRect)
    .function("points", gen_wrap_cv::gen_wrap_RotatedRect::points)
    .property("center", &cv::RotatedRect::center)
    .property("size", &cv::RotatedRect::size)
    .property("angle", &cv::RotatedRect::angle)
  ;
}
```

然后就会调用我们的绑定函数了：

```cpp
		inline int bind_internal(lua_State* L, void(*bindfn)()) {
			int count = lua_gettop(L);
			kaguya::State state(L);
			LuaTable l = state.newTable();
			l.push();
			scope scope(l);
			bindfn();
			return lua_gettop(L) - count;
		}
```

背后就不细说了，反正就是在里面建个表，然后把在这个表上执行相关的绑定操作了。

# 类的实现

类的实现要复杂得多，所以自己看代码并。但我猜测也不外乎将构造器导出，然后构造的时候就设置其元表这样了。不过，其方法都应该是被包装过的。


# 编译过程中出现的错误

```
In file included from /Users/gowa/Repo/luaOpenCV/third_party/opencv/modules/videoio/src/cap_ffmpeg.cpp:47:
cap_ffmpeg_impl.hpp:1581:21: error: use of
      undeclared identifier 'CODEC_FLAG_GLOBAL_HEADER'
        c->flags |= CODEC_FLAG_GLOBAL_HEADER;
                    ^
/Users/gowa/Repo/luaOpenCV/third_party/opencv/modules/videoio/src/cap_ffmpeg_impl.hpp:1609:30: error: use of
      undeclared identifier 'AVFMT_RAWPICTURE'
    if (oc->oformat->flags & AVFMT_RAWPICTURE) {
                             ^
/Users/gowa/Repo/luaOpenCV/third_party/opencv/modules/videoio/src/cap_ffmpeg_impl.hpp:1783:35: error: use of
      undeclared identifier 'AVFMT_RAWPICTURE'
        if( (oc->oformat->flags & AVFMT_RAWPICTURE) == 0 )
                                  ^
/Users/gowa/Repo/luaOpenCV/third_party/opencv/modules/videoio/src/cap_ffmpeg_impl.hpp:2079:32: error: use of
      undeclared identifier 'AVFMT_RAWPICTURE'
    if (!(oc->oformat->flags & AVFMT_RAWPICTURE)) {
                               ^
/Users/gowa/Repo/luaOpenCV/third_party/opencv/modules/videoio/src/cap_ffmpeg_impl.hpp:2378:25: error: use of
      undeclared identifier 'CODEC_FLAG_GLOBAL_HEADER'
            c->flags |= CODEC_FLAG_GLOBAL_HEADER;
                        ^
/Users/gowa/Repo/luaOpenCV/third_party/opencv/modules/videoio/src/cap_ffmpeg_impl.hpp:2461:9: warning: ignoring return
      value of function declared with 'warn_unused_result' attribute [-Wunused-result]
        avformat_write_header(oc_, NULL);
        ^~~~~~~~~~~~~~~~~~~~~ ~~~~~~~~~
```

手动在文件中加上：

```cpp
#define AV_CODEC_FLAG_GLOBAL_HEADER (1 << 22)
#define CODEC_FLAG_GLOBAL_HEADER AV_CODEC_FLAG_GLOBAL_HEADER
#define AVFMT_RAWPICTURE 0x0020
```

还有个错误：

```cpp
error: cannot initialize a variable
      of type 'char *' with an rvalue of type 'const char *'
    char* str = PyString_AsString(obj);
          ^     ~~~~~~~~~~~~~~~~~~~~~~
```

改为 const 即可：

```cpp
    const char* str = PyString_AsString(obj);
```


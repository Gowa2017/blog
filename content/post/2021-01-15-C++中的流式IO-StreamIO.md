---
title: C++中的流式IO-StreamIO
categories:
  - Cpp
date: 2021-01-15 23:43:15
updated: 2021-01-15 23:43:15
tags:
  - Cpp
  - Stream IO
---

C++ 中的 IO 用起来很简单，全局的 std::cin, std::cout，我们直接用 `<<` 操作符就可以写出东西了，而不需要像 C 那样调用函数来进行写出读入等，非常的灵活。很多人都多，C++ 的成功，这个流式 IO 起了很大的作用。C++ 包含了两个 I/O 库：现代的、基于流的 I/O 库和标准的 C 风格的 I/O 函数。

<!--more-->

基于流的 I/O 库围绕对输入输出设备的抽象组织。这些抽象的设备，允许同样的代码来处理输入输出到文件、内存流、或者自定义的适配器能进行任何操作的设备。

大部分的类都是模板类，所以它们能适配使用任何字符类型。对于最基本的字符类型（`char`，`wchar_t`）提供了 typedefs。它们按如下层级进行组织。

![](https://upload.cppreference.com/mwiki/images/0/06/std-io-complete-inheritance.svg)

而我们的 `cin, cout` 则是一个全局的定义：

```cpp
extern std::istream cin;
extern std::ostream cout;
```

# 抽象

| 抽象                                                         |                                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `<ios>`                                                      |                                                              |
| [ios_base](https://en.cppreference.com/w/cpp/io/ios_base)    | 管理格式化标志和输入/输出异常（class）                       |
| [basic_ios](https://en.cppreference.com/w/cpp/io/basic_ios)  | 管理一个任意的流式缓冲区。 (class template)                  |
| `<streambuf>`                                                |                                                              |
| [basic_streambuf](https://en.cppreference.com/w/cpp/io/basic_streambuf) | 抽象一个裸设备。 (class template)                            |
| `<ostream>`                                                  |                                                              |
| [basic_ostream](https://en.cppreference.com/w/cpp/io/basic_ostream) | 包装一个给定的抽象设备([std::basic_streambuf](https://en.cppreference.com/w/cpp/io/basic_streambuf)) ，提供高级的输出接口。 (class template) |
| `<istream>`                                                  |                                                              |
| [basic_istream](https://en.cppreference.com/w/cpp/io/basic_istream) | 包装一个给定的抽象设备([std::basic_streambuf](https://en.cppreference.com/w/cpp/io/basic_streambuf)) ，提供高级的输入接口。 (class template) |
| [basic_iostream](https://en.cppreference.com/w/cpp/io/basic_iostream) | 包装一个给定的抽象设备([std::basic_streambuf](https://en.cppreference.com/w/cpp/io/basic_streambuf)) ，提供高级的输入/输出接口。 (class template) |

## 文件IO实现

|    `<fstream>`                                                          |                                                              |
|---|---|
| [basic_filebuf](https://en.cppreference.com/w/cpp/io/basic_filebuf) | 实现一个裸文件设备。 (class template)                        |
| [basic_ifstream](https://en.cppreference.com/w/cpp/io/basic_ifstream) | 实现高级的文件流输入操作。 (class template)                  |
| [basic_ofstream](https://en.cppreference.com/w/cpp/io/basic_ofstream) | 实现高级的文件流输出操作 (class template)                    |
| [basic_fstream](https://en.cppreference.com/w/cpp/io/basic_fstream) | 实现高级文件流输入/输出操作。 (class template)               |

## 字符串IO实现

| `<sstream>`                                                  | 字符串 I/O 实现                                  |
| ------------------------------------------------------------ | ------------------------------------------------ |
| [basic_stringbuf](https://en.cppreference.com/w/cpp/io/basic_stringbuf) | 实现裸字符串设备。 (class template)              |
| [basic_istringstream](https://en.cppreference.com/w/cpp/io/basic_istringstream) | 实现高级的字符流输入操作。 (class template)      |
| [basic_ostringstream](https://en.cppreference.com/w/cpp/io/basic_ostringstream) | 实现高级的字符流输出操作。 (class template)      |
| [basic_stringstream](https://en.cppreference.com/w/cpp/io/basic_stringstream) | 实现高级的字符流输入/输出操作。 (class template) |

## 数组IO实现

| `<strstream>`                                                | 数组 I/O 实现                                              |
| ------------------------------------------------------------ | ---------------------------------------------------------- |
| [strstreambuf](https://en.cppreference.com/w/cpp/io/strstreambuf)(deprecated in C++98) | implements raw character array device (class)              |
| [istrstream](https://en.cppreference.com/w/cpp/io/istrstream)(deprecated in C++98) | implements character array input operations (class)        |
| [ostrstream](https://en.cppreference.com/w/cpp/io/ostrstream)(deprecated in C++98) | implements character array output operations (class)       |
| [strstream](https://en.cppreference.com/w/cpp/io/strstream)(deprecated in C++98) | implements character array input/output operations (class) |
| **Synchronized output**<BR>Defined in header `<syncstream>`  | 同步输出                                                   |
| [basic_syncbuf](https://en.cppreference.com/w/cpp/io/basic_syncbuf)(C++20) | 同步输出设备封装。 (class template)                        |
| [basic_osyncstream](https://en.cppreference.com/w/cpp/io/basic_osyncstream)(C++20) | 同部输出流封装 (class template)                            |

# Typedefs

常用字符类型的 typedefs。

```cpp
typedef basic_ios<char>                ios;
typedef basic_ios<wchar_t>            wios;

typedef basic_streambuf<char>     streambuf;
typedef basic_streambuf<wchar_t> wstreambuf;
typedef basic_filebuf<char>         filebuf;
typedef basic_filebuf<wchar_t>     wfilebuf;
typedef basic_stringbuf<char>     stringbuf;
typedef basic_stringbuf<wchar_t> wstringbuf;
typedef basic_syncbuf<char>         syncbuf;
typedef basic_syncbuf<wchar_t>     wsyncbuf;

typedef basic_istream<char>         istream;
typedef basic_istream<wchar_t>     wistream;
typedef basic_ostream<char>         ostream;
typedef basic_ostream<wchar_t>     wostream;
typedef basic_iostream<char>       iostream;
typedef basic_iostream<wchar_t>   wiostream;

typedef basic_ifstream<char>       ifstream;
typedef basic_ifstream<wchar_t>   wifstream;
typedef basic_ofstream<char>       ofstream;
typedef basic_ofstream<wchar_t>   wofstream;
typedef basic_fstream<char>         fstream;
typedef basic_fstream<wchar_t>     wfstream;

typedef basic_istringstream<char>     istringstream;
typedef basic_istringstream<wchar_t> wistringstream;
typedef basic_ostringstream<char>     ostringstream;
typedef basic_ostringstream<wchar_t> wostringstream;
typedef basic_stringstream<char>       stringstream;
typedef basic_stringstream<wchar_t>   wstringstream;

typedef basic_osyncstream<char>     osyncstream;
typedef basic_osyncstream<wchar_t> wosyncstream;
```

## 预定义的标准流对象

| Defined in header `<iostream>`                               |                                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [cin<BR>wcin](https://en.cppreference.com/w/cpp/io/cin)      | reads from the standard C input stream [stdin](https://en.cppreference.com/w/cpp/io/c) (global object) |
| [cout<BR>wcout](https://en.cppreference.com/w/cpp/io/cout)   | writes to the standard C output stream [stdout](https://en.cppreference.com/w/cpp/io/c) (global object) |
| [cerr<br />wcerr](https://en.cppreference.com/w/cpp/io/cerr) | writes to the standard C error stream [stderr](https://en.cppreference.com/w/cpp/io/c), unbuffered (global object) |
| [clog<br />wclog](https://en.cppreference.com/w/cpp/io/clog) | writes to the standard C error stream [stderr](https://en.cppreference.com/w/cpp/io/c) (global object) |

# [I/O 操作](https://en.cppreference.com/w/cpp/io/manip)

这些基于流 I/O 库使用 [I/O manipulators](https://en.cppreference.com/w/cpp/io/manip) (e.g. [std::boolalpha](https://en.cppreference.com/w/cpp/io/manip/boolalpha), [std::hex](https://en.cppreference.com/w/cpp/io/manip/hex), etc.) 来控制流的表现。

# Types

下面是一些定义的辅助类型。

| Defined in header `<ios>`                                     |                                                                                 |
| ------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| [streamoff](https://en.cppreference.com/w/cpp/io/streamoff)   | 表示相对的文件/流位置（从 fpos 处的偏移），足够用来表示任意文件大小。 (typedef) |
| [streamsize](https://en.cppreference.com/w/cpp/io/streamsize) | 表示在一个 I/O 操作中传输的字符数量或一个 I/O 缓冲区的大小。(typedef)           |
| [fpos](https://en.cppreference.com/w/cpp/io/fpos)             | 代表一个流或者文件中的绝对位置。(class template)                                |

提供了下面这两个特殊的 [std::fpos](https://en.cppreference.com/w/cpp/io/fpos) :

| Defined in header `<ios>` |                                                                                                                                                     |
| ------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| Type                      | Definition                                                                                                                                          |
| `streampos`               | [std::fpos](http://en.cppreference.com/w/cpp/io/fpos)<[std::char_traits](http://en.cppreference.com/w/cpp/string/char_traits)<char>::state_type>    |
| `wstreampos`              | [std::fpos](http://en.cppreference.com/w/cpp/io/fpos)<[std::char_traits](http://en.cppreference.com/w/cpp/string/char_traits)<wchar_t>::state_type> |

#### Error category interface

| Defined in header `<ios>`                                                          |                                                   |
| ---------------------------------------------------------------------------------- | ------------------------------------------------- |
| [io_errc](https://en.cppreference.com/w/cpp/io/io_errc)(C++11)                     | the IO stream error codes (enum)                  |
| [iostream_category](https://en.cppreference.com/w/cpp/io/iostream_category)(C++11) | identifies the iostream error category (function) |

# std::cin

这是一个预定义的流:

```cpp
// <iostream>
extern istream cin;		/// Linked to standard input
```

输入流：

```cpp
 // <istream>
 typedef basic_istream<char> 		istream;
```

基本输入流：

```cpp
	// <istream>
  template<typename _CharT, typename _Traits>
    class basic_istream : virtual public basic_ios<_CharT, _Traits>
    {

    }
```

`basic_istream` 是一个模板类，需要两个类型参数：

- \_CharT 代表了字符的类型
- \_Traits 类型的特性

而我们看到，默认情况下，是没有传入 `_Traits` 类型参数的。那么，这到底是个什么东西呢。

# Traits

这个意思呢，就是相当于特性萃取了，按照 C++ 发明者的说法：

> _Think of a trait as a small object whose main purpose is to carry information used by another object or algorithm to determine "policy" or "implementation details". - Bjarne Stroustrup_

> 将 trait 看成一个小小的对象，它的主要用途是用来携带一些被其他对象或者算法进行使用以决定 `策略` 或 `实现细节`。

通常，他们会被实现为 `struct`。

## Type Traits（ C++11 开始）

类型萃取定义了一个运行时基于模板的接口，我们可以用它来查询和修改类型的属性。



## std::basic_string

比如我们的这个字符串的模板类。

```cpp
template<
    class CharT,
    class Traits = std::char_traits<CharT>,
    class Allocator = std::allocator<CharT>
> class basic_string;
```

他就有一个类型参数，`std::char_traits` ，我们看看它的 GCC 实现：

## std::char_traits

```cpp
  template<class _CharT>
    struct char_traits : public __gnu_cxx::char_traits<_CharT>
    { };
```

traits，特性，定义了一组操作和类型：

### Member types

| Type         | Definition                                                         |
| ------------ | ------------------------------------------------------------------ |
| `char_type`  | `CharT`                                                            |
| `int_type`   | an integer type that can hold all values of `char_type` plus `EOF` |
| `off_type`   | _implementation-defined_                                           |
| `pos_type`   | _implementation-defined_                                           |
| `state_type` | _implementation-defined_                                           |

### Member functions

| [assign](./char_traits/assign.html)[static]             | assigns a character (public static member function)                                |
| ------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| [eq<br>lt](./char_traits/cmp.html)[static]              | compares two characters (public static member function)                            |
| [move](./char_traits/move.html)[static]                 | moves one character sequence onto another (public static member function)          |
| [copy](./char_traits/copy.html)[static]                 | copies a character sequence (public static member function)                        |
| [compare](./char_traits/compare.html)[static]           | lexicographically compares two character sequences (public static member function) |
| [length](./char_traits/length.html)[static]             | returns the length of a character sequence (public static member function)         |
| [find](./char_traits/find.html)[static]                 | finds a character in a character sequence (public static member function)          |
| [to_char_type](./char_traits/to_char_type.html)[static] | converts `int_type` to equivalent `char_type` (public static member function)      |
| [to_int_type](./char_traits/to_int_type.html)[static]   | converts `char_type` to equivalent `int_type` (public static member function)      |
| [eq_int_type](./char_traits/eq_int_type.html)[static]   | compares two `int_type` values (public static member function)                     |
| [eof](./char_traits/eof.html)[static]                   | returns an _eof_ value (public static member function)                             |
| [not_eof](./char_traits/not_eof.html)[static]           | checks whether a character is _eof_ value (public static member function)          |

## 标准的 traits：

| Type                            | Definition                       |
| ------------------------------- | -------------------------------- |
| **std::string**                 | std::basic_string<char>          |
| **std::wstring**                | std::basic_string<wchar_t>       |
| std::u8string (C++20)           | std::basic_string<char8_t>       |
| **std::u16string** (C++11)      | std::basic_string<char16_t>      |
| **std::u32string** (C++11)      | std::basic_string<char32_t>      |
| **std::pmr::string** (C++17)    | std::pmr::basic_string<char>     |
| **std::pmr::wstring** (C++17)   | std::pmr::basic_string<wchar_t>  |
| std::pmr::u8string (C++20)      | std::pmr::basic_string<char8_t>  |
| **std::pmr::u16string** (C++17) | std::pmr::basic_string<char16_t> |
| **std::pmr::u32string** (C++17) | std::pmr::basic_string<char32_t> |

### Template parameters

| CharT     | -   | character type                                                                                              |
| --------- | --- | ----------------------------------------------------------------------------------------------------------- |
| Traits    | -   | traits class specifying the operations on the character type                                                |
| Allocator | -   | [_Allocator_](https://en.cppreference.com/w/cpp/named_req/Allocator) type used to allocate internal storage |

## 终于

我们可以把 traits 看成一些可对类型进行的操作，以及一些内部使用的类型，会被其他的对象或者算法用到的地方，实际上，通常我们使用的时候，可能并不需要关注这些内容。

比如：`std::string`。



# I/Omanipulators 操纵器

manipulators (操纵器) 是一些帮助函数，这些函数让使用 `<<, >>` 来控制输入/输出成为可能。

调用的时候没有参数的 manipulators （如 `std::cout << std::boolalpha`或者 `std::cin >> std::hex`） 被实现为使用到一个流的引用作为他们的唯一参数的函数。`basic_ostream::operator <<` 和 `basic_istream::opeartor>>` 的特殊重载接收指针。

至于调用的时候有参数的 manipulators （`std::cout << std::setw(10)`）被实现为返回不特定类型的对象的函数。这些 manipulators  定义了他们自己的 `<<, >>`。

| Defined in header `<ios>`                                    |                                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [boolalpha<br />noboolalpha](./manip/boolalpha.html)         | switches between textual and numeric representation of booleans  (function) |
| [showbase<br />noshowbase](./manip/showbase.html)            | controls whether prefix is used to indicate numeric base  (function) |
| [showpoint<br />noshowpoint](./manip/showpoint.html)         | controls whether decimal point is always included in floating-point representation  (function) |
| [showpos<br />noshowpos](./manip/showpos.html)               | controls whether the `+` sign used with non-negative numbers  (function) |
| [skipws<br />noskipws](./manip/skipws.html)                  | controls whether leading whitespace is skipped on input  (function) |
| [uppercase<br />nouppercase](./manip/uppercase.html)         | controls whether uppercase characters are used with some output formats  (function) |
| [unitbuf<br />nounitbuf](./manip/unitbuf.html)               | controls whether output is flushed after each operation  (function) |
| [internal<br />left<br />right](./manip/left.html)           | sets the placement of fill characters  (function)            |
| [dec<br />hex<br />oct](./manip/hex.html)                    | changes the base used for integer I/O  (function)            |
| [fixed<br />scientific<br />hexfloat<br />defaultfloat](./manip/fixed.html)(C++11)(C++11) | changes formatting used for floating-point I/O  (function)   |
|                                                              |                                                              |
| Defined in header `<istream>`                                |                                                              |
| [ws](./manip/ws.html)                                        | consumes whitespace  (function template)                     |
|                                                              |                                                              |
| Defined in header `<ostream>`                                |                                                              |
| [ends](./manip/ends.html)                                    | outputs '**\0**'  (function template)                        |
| [flush](./manip/flush.html)                                  | flushes the output stream  (function template)               |
| [endl](./manip/endl.html)                                    | outputs '**\n**' and flushes the output stream  (function template) |
| [emit_on_flush<br />no_emit_on_flush](./manip/emit_on_flush.html)(C++20) | controls whether a stream's [`basic_syncbuf`](./basic_syncbuf.html) emits on flush  (function template) |
| [flush_emit](./manip/flush_emit.html)(C++20)                 | flushes a stream and emits the content if it is using a [`basic_syncbuf`](./basic_syncbuf.html)  (function template) |
|                                                              |                                                              |
| Defined in header `<iomanip>`                                |                                                              |
| [resetiosflags](./manip/resetiosflags.html)                  | clears the specified ios_base flags  (function)              |
| [setiosflags](./manip/setiosflags.html)                      | sets the specified ios_base flags  (function)                |
| [setbase](./manip/setbase.html)                              | changes the base used for integer I/O  (function)            |
| [setfill](./manip/setfill.html)                              | changes the fill character  (function template)              |
| [setprecision](./manip/setprecision.html)                    | changes floating-point precision  (function)                 |
| [setw](./manip/setw.html)                                    | changes the width of the next input/output field  (function) |
| [get_money](./manip/get_money.html)(C++11)                   | parses a monetary value  (function template)                 |
| [put_money](./manip/put_money.html)(C++11)                   | formats and outputs a monetary value  (function template)    |
| [get_time](./manip/get_time.html)(C++11)                     | parses a date/time value of specified format  (function template) |
| [put_time](./manip/put_time.html)(C++11)                     | formats and outputs a date/time value according to the specified format  (function template) |
| [quoted](./manip/quoted.html)(C++14)                         | inserts and extracts quoted strings with embedded spaces  (function template) |
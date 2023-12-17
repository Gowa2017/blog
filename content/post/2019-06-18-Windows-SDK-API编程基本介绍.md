---
title: Windows-SDK-API编程基本介绍
categories:
  - Windows
date: 2019-06-18 22:59:32
updated: 2019-06-18 22:59:32
tags: 
  - Windows
---
教我们怎么样用 Win32 及 COM API 编写桌面应用。
原文 [Get Started with Win32 and C++](https://docs.microsoft.com/zh-cn/windows/desktop/learnwin32/learn-to-program-for-windows)
# 准备开发环境
想要用 C/C++ 来编写在 Windows 中进行开发，就必须先安装微软的SDK，或者包括了 SDK 的集成开发环境如 VC++。SDK 包含了编译和链接我们的程序必要的 **头文件** 和**库文件**。也包含了命令行工具来构建应用。虽然可以通过命令行来编译和构建 Windows 应用，但是官方推荐的是用 VS 开发环境来干这个活
# Windows 代码约定

Windows 的 API 代码命名有些奇怪，比如你会看到 **DWORD_PTR,LPRECT** 这样的类型，或者 *hWnd， pwsz* 这样的变量名字。

Windows API 包括很多的函数和组件对象模型（COM）接口。单纯以 C++ 类形式提供的 API 很少。

## typdef

Windows API 头文件包含了很多的 typedef。很多是在 *WinDef.h* 头文件内定义的。

## Boolean

BOOL 被 typedef 为一个整型值。

```c++
#define FALSE    0 
#define TRUE     1
```

所以说，很多返回 **BOOL** 类型的函数可能会返回一个非0的值来表示 **真**。因此，我们要注意：

```c++
// Right way.
BOOL result = SomeFunctionThatReturnsBoolean();
if (result) 
{ 
    ...
}
```

而不要这样写：

```cpp
// Wrong!
if (result == TRUE) 
{
    ... 
}
```

要记住，**BOOL** 与 C++ 的 **bool** 并不是一个东西。

## Pointers 指针

指针类型大多有 **P,LP** 前缀。下面的声明是等价的。

```cpp
RECT*  rect;  // Pointer to a RECT structure.
LPRECT rect;  // The same
PRECT  rect;  // Also the same.
```

**P** 代表指针，**LP** 代表 长指针。

## 指针精度类型

下面的数据类型的大小永远是一个指针大小大小————32（32位程序），64（64位程序）。大小是在编译时决定的。当 32 位的程序在 64 位的系统上运行的时候，数据类型始终还是 4 字节的。

- DWORD_PTR
- INT_PTR
- LONG_PTR
- ULONG_PTR
- UINT_PTR

这些类型用在可能会将一个整型强制转换为指针的时候。
# Strings

Windows 在UI元素，文件名等中原生支持 Unicode 字符串。 Windows 用 UTF-16 编码来表示 Unicode 字符，意思就是每个字符都是一个 16bit 的值。UTF-16字符，也被叫做 **宽字符**，用以和 8-bit 的 ANSI 字符区别。 VC++ 编译器支持内建的 **wchar_t** 类型，**WinNT.h** 文件也定义了：

```cpp
typedef wchar_t WCHAR;
```

这两者我们可能都会遇到。如果要声明一个字面意义的宽字符，在字面字符前加上 **L**。

```cpp
wchar_t a = L'a';
wchar_t *str = L"hello";
```

## Unicode 与 ANSI 函数

在 Windows 引入  Unicode 的时候，其提供了一过渡的方式，也就是提供了两套 API，一套接受的是 ANSI 字符，一套接受的是 Unicode 字符。

- **SetWindowTextA** takes an ANSI string.
- **SetWindowTextW** takes a Unicode string.

内部会将 ANSI 版本转换成 Unicode 版本。 Windows 头文件也定义了一个宏来转换：

```cpp
#ifdef UNICODE
#define SetWindowText  SetWindowTextW
#else
#define SetWindowText  SetWindowTextA
#endif
```
# windows(窗口)到底是什么

Winodows 的核心就是窗口。

![](https://docs.microsoft.com/zh-cn/windows/desktop/learnwin32/images/window01.png)

脑海中关于窗口的概念可能会是这样的。

这样的窗口被叫做 **应用窗口或主窗口**。典型的其有个含有 标题栏，最大化，最小化按钮的框。这个框被叫做 **非客户区**，意思就是这部分是归 Windows 系统管理的。框里面的部分就是 **客户区**，由我们的程序管理。

![](https://docs.microsoft.com/zh-cn/windows/desktop/learnwin32/images/window02.png)

这也是一个窗口。 UI 元素本身，其实也是一个窗口。主要的不同就是：UI 控制元素并不是自身单独存在的，其会相对于一个主窗口而存在。当拖动主窗口的时候，这些控制元素也会移动。。当然，这些控制元素是可以与主窗口进行通信的。（例如，主窗口最到从按钮的点击事件。）

所以当我们想象一个窗口的时候，不要只想到**应用窗口**，而是从程序的角度去看待他：

- 占据屏幕的一个区域
- 在一个时刻可见也可能不可见。
- 知道如何绘制自身
- 响应用户或者系统的事件。

## 父窗口与所有者窗口

如果是一个UI控制窗口，其被称为是主窗口的一个子窗口。主窗口是这个UI窗口的父窗口。父窗口提供了用来定位一个子窗口的坐标系统。有父窗口的情况下会影响窗口的风格：例如，子窗口会被裁剪到不会超出父窗口。

另外一个相关的东西就是主窗口与对话框了，当显示一个对话框的时候，主窗口就是所有者窗口，对话框就是被拥有的窗口。一个被拥有的窗口总是在其拥有者之前。当其所有者最小化的时候它会被隐藏，其与所有者一起销毁。

![](https://docs.microsoft.com/zh-cn/windows/desktop/learnwin32/images/window03.png)

应用窗口拥有对话框，对话框是 两个按钮的父窗口。

## Windows Hanldes 句柄

Windows 是对象——他们拥有代码和数据——但他们并不是 C++ 类。程序通过一个叫做 **handle** 的东西来引用窗口。 句柄，是一个不透明的类型。**实质上，其只是一个操作系统用来标识对象的整数**。窗口句柄的类型是 **HWND**。句柄通过创建窗口的函数返回：`CreateWindow(), CreateWindowEx()`。

为了对一个窗口进行操作，我们通过会调用一些以 **HWND** 为参数的函数。比如，如果我们想重新定位一个窗口：

```cpp
BOOL MoveWindow(HWND hWnd, int X, int Y, int nWidth, int nHeight, BOOL bRepaint);
```

记住 ：**句柄** 不是指针。

## 屏幕窗口坐标

坐标以设备无关的像素进行度量。根据任务不同，我们可能会相对于屏幕，窗口，或者窗口的客户区来进行测量坐标。坐标 (0,0) 总是位于左上。
# WinMAIN：应用的入口

每个 Windows 程序都有一个入口函数，其被命名为 **WinMain 或 wWinMain**。

```cpp
int WINAPI wWinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, PWSTR pCmdLine, int nCmdShow);
```

这四个参数分别是：

- hInstance 这是一个被叫做**一个实例的句柄** 或 **模块的句柄**。操作系统使用这个来标识一个 exe 。一些特定的窗口函数会需要这个实例句柄——例如，加载图标或位图。
- hPrevInstance 没什么意义。在 16-bit 的系统中使用，现在总是0.
- pCmdLine 包含了 Unicode 字符串的命令行参数
- nCmdShow 表示窗口怎么显示。是最小化，最大化，还是正常。

此函数会返回一  **int** 值。这个值随后就被操作系统使用。

**WINAPI** 是一个调用约定。一个 **调用约定** 定义了一个函数如何从调用者处获得参数。例如，其定义了参数在栈上的顺序。

`WinMain` 与 `wWinMain` 是一样的，只是其接收的命令行参数是 ANSI 字符。

编译器是如何知道该去调用 `wWinMain` 而不是 `main()` 呢？ 微软的C运行时库（CRT）提供了一个 `main()` 的实现会调用 `wWinMain 或者 WinMain`。

下面是一个空的 `WinMain` 函数：

```cpp
INT WinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance,
    PSTR lpCmdLine, INT nCmdShow)
{
    return 0;
}
```

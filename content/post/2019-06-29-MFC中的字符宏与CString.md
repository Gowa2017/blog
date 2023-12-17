---
title: MFC中的字符宏与CString
categories:
  - Windows
date: 2019-06-29 23:26:00
updated: 2019-06-29 23:26:00
tags:
  - Windows
  - MFC
---

MFC 的宏太多了，让人目不暇接，回想　 Linux 下的多简单呢，各种各样的接口和调用都都非常少，哪里像 MFC 这么恶心。

 <!--more-->

# 字符集(CharacterSet)与编码(Encoding)

我们先要区分开这两个概念才能理解后面文中说的那么多符号到底是什么东西。

- 字符集。符号的集合。
- 码表（Code Pages）。字符集中的每个符号赋予一个整型值，这个叫做码点(Code Point）。这些码点的集合就形成了码表，编码集
- 编码。将编码集中的码点，用字节来表示，以方便存储到计算机。

一般来说，前两个概念经常被混在一起。比如说：ASCII，代表了一个字符集，也代表了一个编码方式，实际上我们这里提到的应该是 ASCII 码表。

## ANSI

ANSI 是一套字符集，其编码就是 ANSI 编码。

ASNI 中有 8bit 来存储一个字符，所以其支持的符号是有限的，只有最多 256 个，这对于我们中文来说肯定是不够的。

## Unicode

Unicode 是一套字符集，其没有定义编码，所以说其有多种实现。也就是说对于 Unicode 有多种形式来存储在计算机上。

## UTF-8

UTF-8 是 Unicode 的一个编码实现，其使用变长字节来存储 Unicode 编码。

UTF-8 的存储规则如下：

- 单字节符号。第一位为 0，后面 7 位是其 Unicode 码，在英文字母中， ANSI 与 Unicode 编码一致。
- N 字节的符号（N>1）。第 1 字节的前 n 位置 1，n+1 位置 0，后面所有字节的前两位都是 0。其余位就保存了 Unicode 编码。

UTF-8 中，中文字符需要三个字节来存储，英文字符只需要一个字节来存储。

## UTF-16/UCS-2

固定使用 2byte 的 Unicode 编码。

## UTF-32/UCS-4

固定使用 4byte 的 Unicode 编码。

# MFC 中的应用

## char wchar_t

一般来说，char 是 8bit 的，其只能表示应该 ANSI 字符，而这两个类型是由 C/C++ 语言本身所定义的数据类型。
wchar_t 表示一个宽字符类型，意思是多字节的，可以存储任何 Unicode 字符，因此我们在涉及中文的时候一般都会用到。也是 C/C++ 定义的数据类型。

## L

这个很简单，这个一般用来放在字符前面，告诉编译器，这是一个 Unicode 字符。这也是 C/C++ 定义的。

## \_T \_TEXT

其定义为：

```cpp
#define _T(x)       __T(x)
#define _TEXT(x)       __T(x)
```

而根据我们程序使用字符集的不同，\_\_T 有两种定义形式：

```cpp
#define __T(x)      L ## x // Unicode

#define __T(x)      x //ANSI
```

所以说，\_T 会根据我们程序的字符集类型来自动匹配字符的编码。

## TEXT

`_TEXT` 定义在 winnt.h 中，其作用与 `_T` 一致。

## 使用

我们在用到字符的时候，必须告诉编译器，我们的字符是什么类型的，以此才能决定如何存储在文件中去。

因此我们可以使用 `_T, _TEXT, TEXT`

# CString

CString, CStringA, CStringW 都是模板类 CStringT 的一个特殊实现。不同在于：

- CStringW 包含了 wchar_t 类型，支持 Unicode 字符。
- CStringA 包含了 char 类型，支持单字节和多字节字符。
- CString 根据编译时指定的字符类型决定其应该是什么类型。Unicode 和单字节多字节都支持。

```cpp
typedef ATL::CStringT< wchar_t, StrTraitMFC_DLL< wchar_t > > CStringW;
typedef ATL::CStringT< char, StrTraitMFC_DLL< char > > CStringA;
typedef ATL::CStringT< TCHAR, StrTraitMFC_DLL< TCHAR > > CString;
```

你可能会问 TCHAR 又是什么，其实也是一个宏，其在两个地方进行了定义:

```cpp
// winnt.h,
typedef char TCHAR, *PTCHAR;
typedef wchar_t WCHAR;    // wc,   16-bit UNICODE character
typedef WCHAR TCHAR, *PTCHAR;

// ucrt/tchar.h
typedef char            TCHAR;
typedef wchar_t     TCHAR;

```

事实上，其就是 char 或者 wchar_t，不过是会在编译时我们需要支持的字符集来选择具体实现成哪个类型。

CString 对象支持将字符存在在一个 CStringData 对象内，其接受 C 风格的 NULL 结尾的字符串。CString 可以更高性能的跟踪字符串的长度，但它也支持从存储的字符数据内获取 NULL 字符来支持转换到 LPCWSTR 类型。

有几种类型不需要链接到 MFC 库就能使用：CAtlString[AW]。

## CString 实现

CString 是 CStringT 针对一个特定的泛型参数而来，其就是 `CStringT<TCHAR>`。

我们可以看到 CStringT<TCHAR> 继承自 CSimpleStringT<TCHAR>。

```cpp
template< typename BaseType, class StringTraits >
class CStringT :
	public CSimpleStringT< BaseType, _CSTRING_IMPL_::_MFCDLLTraitsCheck<BaseType, StringTraits>::c_bIsMFCDLLTraits >
{
 }
```

其中，泛型参数 **StringTraits** （字符串特性）用来决定字符串类是否需要 CRT 支持及在何处定位字符串资源，其可以为下列值：

- **StrTraitATL< wchar_t** | **char** | **TCHAR, ChTraitsCRT< wchar_t** | **char** | **TCHAR > >**：类需要 CRT 支持，并且从 **m_hInstResource** 指定的模块来获取字符串资源。

- **StrTraitATL< wchar_t** | **char** | **TCHAR, ChTraitsOS< wchar_t** | **char** | **TCHAR > >**：类不需要 CRT 支持，并且从 **m_hInstResource** 指定的模块来获取字符串资源。
- **StrTraitMFC< wchar_t** | **char** | **TCHAR, ChTraitsCRT< wchar_t** | **char** | **TCHAR > >**：类需要 CRT 支持，字符串资源使用标准的 MFC 搜索算法。
- **StrTraitMFC< wchar_t** | **char** | **TCHAR, ChTraitsOS< wchar_t** | **char** | **TCHAR > >**：类不需要 CRT 支持，字符串资源使用标准的 MFC 搜索算法。

## CStringT 内建了一些数据类型：

```cpp
	// CStringT
	typedef CSimpleStringT< BaseType, _CSTRING_IMPL_::_MFCDLLTraitsCheck<BaseType, StringTraits>::c_bIsMFCDLLTraits > CThisSimpleString;
	typedef StringTraits StrTraits;
	typedef typename CThisSimpleString::XCHAR XCHAR;
	typedef typename CThisSimpleString::PXSTR PXSTR;
	typedef typename CThisSimpleString::PCXSTR PCXSTR;
	typedef typename CThisSimpleString::YCHAR YCHAR;
	typedef typename CThisSimpleString::PYSTR PYSTR;
	typedef typename CThisSimpleString::PCYSTR PCYSTR;
```

```cpp

template< typename BaseType = char >
class ChTraitsBase
{
public:
	typedef char XCHAR;
	typedef LPSTR PXSTR;
	typedef LPCSTR PCXSTR;
	typedef wchar_t YCHAR;
	typedef LPWSTR PYSTR;
	typedef LPCWSTR PCYSTR;
};

template<>
class ChTraitsBase< wchar_t >
{
public:
	typedef wchar_t XCHAR;
	typedef LPWSTR PXSTR;
	typedef LPCWSTR PCXSTR;
	typedef char YCHAR;
	typedef LPSTR PYSTR;
	typedef LPCSTR PCYSTR;
};

	// CSimpleStringT
	typedef typename ChTraitsBase< BaseType >::XCHAR XCHAR;
	typedef typename ChTraitsBase< BaseType >::PXSTR PXSTR;
	typedef typename ChTraitsBase< BaseType >::PCXSTR PCXSTR;
	typedef typename ChTraitsBase< BaseType >::YCHAR YCHAR;
	typedef typename ChTraitsBase< BaseType >::PYSTR PYSTR;
	typedef typename ChTraitsBase< BaseType >::PCYSTR PCYSTR;


```

- XCHAR 与 CStringT 对象泛型参数类型一致的单个字符（char, wchar_t）
- YCHAR 与 CStringT 对象泛型参数类型相对的单个字符（char, wchar_t）。如果 XCHAR 是 char，那么 YCHAR 就是 wchar_t。
- PXSTR 字符串。指向 XCHAR。
- PYSTR 字符串。指向 YCHAR。
- PCXSTR 字符串。指向 consta XCHAR
- PCYSTR 字符串。指向 consta YCHAR

## CStringData

CStringT 没有数据成员，我们所需要的字符是存在 CSimpleStringT 的 m_pszData 内。

```cpp
private:
	PXSTR m_pszData;
```

当我们构造 CStringT 的时候，实际上也会构造 CSimpleStringT：

```cpp
	explicit CSimpleStringT(_Inout_ IAtlStringMgr* pStringMgr)
	{
		ATLENSURE( pStringMgr != NULL );
		CStringData* pData = pStringMgr->GetNilString();
		Attach( pData );
	}

	void Attach(_Inout_ CStringData* pData) throw()
	{
		m_pszData = static_cast< PXSTR >( pData->data() );
	}

```

所以说呢是，强制将 IAtlStringMgr 分配的内存转换为一个 PXSTR 指针的。

```cpp
struct CStringData
{
	IAtlStringMgr* pStringMgr;  // String manager for this CStringData
	int nDataLength;  // Length of currently used data in XCHARs (not including terminating null)
	int nAllocLength;  // Length of allocated data in XCHARs (not including terminating null)
	long nRefs;     // Reference count: negative == locked
	// XCHAR data[nAllocLength+1]  // A CStringData is always followed in memory by the actual array of character data

	void* data() throw()
	{
		return (this+1);
	}

	void AddRef() throw()
	{
		ATLASSERT(nRefs > 0);
		_InterlockedIncrement(&nRefs);
	}
	bool IsLocked() const throw()
	{
		return nRefs < 0;
	}
	bool IsShared() const throw()
	{
		return( nRefs > 1 );
	}
	void Lock() throw()
	{
		ATLASSERT( nRefs <= 1 );
		nRefs--;  // Locked buffers can't be shared, so no interlocked operation necessary
		if( nRefs == 0 )
		{
			nRefs = -1;
		}
	}
	void Release() throw()
	{
		ATLASSERT( nRefs != 0 );

		if( _InterlockedDecrement( &nRefs ) <= 0 )
		{
			pStringMgr->Free( this );
		}
	}
	void Unlock() throw()
	{
		ATLASSERT( IsLocked() );

		if(IsLocked())
		{
			nRefs++;  // Locked buffers can't be shared, so no interlocked operation necessary
			if( nRefs == 0 )
			{
				nRefs = 1;
			}
		}
	}
};

```

在这里，我们来回顾一下这过程:

1. CStringT 继承自 CSimpleStringT。
2. CSimpleStringT 的成员 m_pszData 指向一块由 CStringData 引用的内存，而这块内存是由 StringManager（构造 CSimpleStringT 的时候获得） 来进行分配的。
3. CSimpleStringT 没有对 CStringData 对象的成员引用，而是使用一个强制转换的指针来指向它，需要访问的石油又强制转换回来。

构造一个 CString：

```cpp
#include <atlstr.h>

int main() {
    CString aCString = CString(_T("A string"));
    _tprintf(_T("%s"), (LPCTSTR) aCString);
}
```

但是我们肯定会遇到，在程序中并不是很多地方都会用 CString ，而有的地方可能是需要用 char \* 这样的，比如说一些标准库的函数。

### data()

这个方法定义得有点让我感觉奇怪，

```cpp
	void* data() throw()
	{
		return (this+1);
	}
```

## CString 类型间转换

前文我们知道，是不是上 CStringT 内部的数据是有类型的，如果我们想要将 CStringA 转换到 char， CStringW 转换到 wchar_t，这是没有什么难度的。

## GetBuffer() 与 GetString()

### GetBuffer()

```cpp
	PXSTR GetBuffer()
	{
		CStringData* pData = GetData();
		if( pData->IsShared() )
		{
			Fork( pData->nDataLength );
		}

		return( m_pszData );
	}

	CStringData* GetData() const throw()
	{
		return( reinterpret_cast< CStringData* >( m_pszData )-1 );
	}

	ATL_NOINLINE void Fork(_In_ int nLength)
	{
		CStringData* pOldData = GetData();
		int nOldLength = pOldData->nDataLength;
		CStringData* pNewData = pOldData->pStringMgr->Clone()->Allocate( nLength, sizeof( XCHAR ) );
		if( pNewData == NULL )
		{
			ThrowMemoryException();
		}
		int nCharsToCopy = ((nOldLength < nLength) ? nOldLength : nLength)+1;  // Copy '\0'
		CopyChars( PXSTR( pNewData->data() ), nCharsToCopy,
			PCXSTR( pOldData->data() ), nCharsToCopy );
		pNewData->nDataLength = nOldLength;
		pOldData->Release();
		Attach( pNewData );
	}

```

可以看到，这个有几步操作：

1. 获取到 CStringData 对象的指针。
2. 如果是有多个 CString 指向同一个 CStringObject ，那么会复制一个出来，进行返回。
3. 否则的话就直接返回 m_pszData。

我们来讨论非共享形式下的的返回，我们知道， m_pszData 实际上是指向 CStringData 的 data 成员：

```cpp
private:
	PXSTR m_pszData;
m_pszData = static_cast< PXSTR >( pData->data() );
```

也就是直接返回了一个 `char *` 或者 `wchar_t` 的缓冲区，由于其类型是 PXSTR，所以我们是可以对这个缓冲区进行读写的。共享状态下则不然，你的修改不会修改到被共享的那个缓冲区。

### GetString()

这个定义就非常简单了：

```cpp
	PCXSTR GetString() const throw()
	{
		return( m_pszData );
	}

```

就是直接返回了缓冲区，不过返回的是 PCXSTR，不可以进行写入了。

# CString 与 STL string

很多人建议如果想要用 string 的话，就定义一个新的类型来适配：

```cpp
typedef std::basic_string<TCHAR> tstring;
```

或者

```cpp
#ifdef UNICODE
	typedef std::basic_string<char> String;
#else
	typedef std::basic_string<wchar_t> String;
#endif
```

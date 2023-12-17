---
title: JNI使用-LuaJava实现
categories:
  - [Java]
  - [Lua]
date: 2019-11-03 15:35:49
updated: 2019-11-03 15:35:49
tags: 
  - Java
  - JNI
  - Android
---

LuaJava 就是一个使用 JNI 的一个例子。其能在 Java 代码中，让 Lua 虚拟机去执行 Lua 代码，也能在 Lua 虚拟机中执行 Java 中的代码，其实现，就是利用了 JNI 的双向通信技术。

<!--more-->

[LuaJava项目地址](https://github.com/jasonsantos/luajava)我们可以从几个关键的地方来看他的实现。

概括而言，Lua 运行在 C 层，从 Java 的概念上来说的话就是 Native 层。我们想要让 Lua 来执行 代码，必然就需要有一个 lua_State ，我们就来看这个过程是如何的。

# LuaState.java

这个类，是对 C 内 lua_State 的一个包装，其自身含有一个对当前 LuaState 实力的ID，以及，对于 C 内 lua_State 的一个引用。 CPtr 就是一个 C 指针的实现，其内只是简单的用整型的形式，来保存一个指针。

```java
// LuaState.java
/**
   * Opens the library containing the luajava API
   */
  static
  {
    System.loadLibrary(LUAJAVA_LIB);
  }

  private CPtr luaState;

  private int stateId;

  /**
   * Constructor to instance a new LuaState and initialize it with LuaJava's functions
   * @param stateId
   */
  protected LuaState(int stateId)
  {
    luaState = _open();
    luajava_open(luaState, stateId);
    this.stateId = stateId;
  }

  /**
   * Receives a existing state and initializes it
   * @param luaState
   */
  protected LuaState(CPtr luaState)
  {
    this.luaState = luaState;
    this.stateId = LuaStateFactory.insertLuaState(this);
    luajava_open(luaState, stateId);
  }


```

```java
// CPtr.java
public class CPtr
{
    
    /**
     * Compares this <code>CPtr</code> to the specified object.
     *
     * @param other a <code>CPtr</code>
     * @return      true if the class of this <code>CPtr</code> object and the
     *		    class of <code>other</code> are exactly equal, and the C
     *		    pointers being pointed to by these objects are also
     *		    equal. Returns false otherwise.
     */
	public boolean equals(Object other)
	{
		if (other == null)
			return false;
		if (other == this)
	    return true;
		if (CPtr.class != other.getClass())
	    return false;
		return peer == ((CPtr)other).peer;
   }


    /* Pointer value of the real C pointer. Use long to be 64-bit safe. */
    private long peer;
    
    /**
     * Gets the value of the C pointer abstraction
     * @return long
     */
    protected long getPeer()
    {
    	return peer;
    }

    /* No-args constructor. */
    CPtr() {}
 
}

```

我们可以肯定的是，当我们构造一个 LuaState 的时候，调用了一个 `_open()` 函数，返回了一个 CPtr 指针，代表 C 内的 lua_State。有理由相信，这个函数应该是一个 native 方法。

在 LuaState.java 文件内我们是能看到很多 native 方法的：

```java
  /********************* Lua Native Interface *************************/

  private synchronized native CPtr _open();
  private synchronized native void _close(CPtr ptr);
  private synchronized native CPtr _newthread(CPtr ptr);

  // Stack manipulation
  private synchronized native int  _getTop(CPtr ptr);
  private synchronized native void _setTop(CPtr ptr, int idx);
  private synchronized native void _pushValue(CPtr ptr, int idx);
  private synchronized native void _remove(CPtr ptr, int idx);
  private synchronized native void _insert(CPtr ptr, int idx);
  private synchronized native void _replace(CPtr ptr, int idx);
  private synchronized native int  _checkStack(CPtr ptr, int sz);

```

找了一下才发现，对于 `_` 符号的方法，在 JNI 生成的头文件中表示还不太一样，居然会加了一个 1 在方法名前。

## _open()

在生成的头文件  luajava.h 中，我们可以看到这个方法的 C 实现：

```c
JNIEXPORT jobject JNICALL Java_org_keplerproject_luajava_LuaState__1open
  (JNIEnv * env , jobject jobj)
{
   lua_State * L = lua_open();

   jobject obj;
   jclass tempClass;

   tempClass = ( *env )->FindClass( env , "org/keplerproject/luajava/CPtr" );
    
   obj = ( *env )->AllocObject( env , tempClass );
   if ( obj )
   {
      ( *env )->SetLongField( env , obj , ( *env )->GetFieldID( env , tempClass , "peer", "J" ) , ( jlong ) L );
   }
   return obj;

}
```

使用的是 lua5.1 所以说呢，很多方法实际上现在都不用了，如果是 5.3 应该会用的是 `luaL_newstate() `了。

看其操作过程，就是在 C 内新建虚拟机后，然后在 Java 中分配一个 Cptr 实例，并且对这个实例赋值我们刚才 虚拟机的内存地址就完事。

## luajava_open()

这个方法会将 luajava 的函数给压入到 lua_State 内去：

```java
  private synchronized native void luajava_open(CPtr cptr, int stateId);
```

```c
JNIEXPORT void JNICALL Java_org_keplerproject_luajava_LuaState_luajava_1open
  ( JNIEnv * env , jobject jobj , jobject cptr , jint stateId )
{
  lua_State* L;

  jclass tempClass;

  L = getStateFromCPtr( env , cptr );

  lua_pushstring( L , LUAJAVASTATEINDEX );
  lua_pushnumber( L , (lua_Number)stateId );
  lua_settable( L , LUA_REGISTRYINDEX );


  lua_newtable( L );

  lua_setglobal( L , "luajava" );

  lua_getglobal( L , "luajava" );
  
  set_info( L);
  
  lua_pushstring( L , "bindClass" );
  lua_pushcfunction( L , &javaBindClass );
  lua_settable( L , -3 );

  lua_pushstring( L , "new" );
  lua_pushcfunction( L , &javaNew );
  lua_settable( L , -3 );

  lua_pushstring( L , "newInstance" );
  lua_pushcfunction( L , &javaNewInstance );
  lua_settable( L , -3 );

  lua_pushstring( L , "loadLib" );
  lua_pushcfunction( L , &javaLoadLib );
  lua_settable( L , -3 );

  lua_pushstring( L , "createProxy" );
  lua_pushcfunction( L , &createProxy );
  lua_settable( L , -3 );

  lua_pop( L , 1 );

  if ( luajava_api_class == NULL )
  {
    tempClass = ( *env )->FindClass( env , "org/keplerproject/luajava/LuaJavaAPI" );

    if ( tempClass == NULL )
    {
      fprintf( stderr , "Could not find LuaJavaAPI class\n" );
      exit( 1 );
    }

    if ( ( luajava_api_class = ( *env )->NewGlobalRef( env , tempClass ) ) == NULL )
    {
      fprintf( stderr , "Could not bind to LuaJavaAPI class\n" );
      exit( 1 );
    }
  }

  if ( java_function_class == NULL )
  {
    tempClass = ( *env )->FindClass( env , "org/keplerproject/luajava/JavaFunction" );

    if ( tempClass == NULL )
    {
      fprintf( stderr , "Could not find JavaFunction interface\n" );
      exit( 1 );
    }

    if ( ( java_function_class = ( *env )->NewGlobalRef( env , tempClass ) ) == NULL )
    {
      fprintf( stderr , "Could not bind to JavaFunction interface\n" );
      exit( 1 );
    }
  }

  if ( java_function_method == NULL )
  {
    java_function_method = ( *env )->GetMethodID( env , java_function_class , "execute" , "()I");
    if ( !java_function_method )
    {
      fprintf( stderr , "Could not find <execute> method in JavaFunction\n" );
      exit( 1 );
    }
  }

  if ( throwable_class == NULL )
  {
    tempClass = ( *env )->FindClass( env , "java/lang/Throwable" );

    if ( tempClass == NULL )
    {
      fprintf( stderr , "Error. Couldn't bind java class java.lang.Throwable\n" );
      exit( 1 );
    }

    throwable_class = ( *env )->NewGlobalRef( env , tempClass );

    if ( throwable_class == NULL )
    {
      fprintf( stderr , "Error. Couldn't bind java class java.lang.Throwable\n" );
      exit( 1 );
    }
  }

  if ( get_message_method == NULL )
  {
    get_message_method = ( *env )->GetMethodID( env , throwable_class , "getMessage" ,
                                                "()Ljava/lang/String;" );

    if ( get_message_method == NULL )
    {
      fprintf(stderr, "Could not find <getMessage> method in java.lang.Throwable\n");
      exit(1);
    }
  }

  if ( java_lang_class == NULL )
  {
    tempClass = ( *env )->FindClass( env , "java/lang/Class" );

    if ( tempClass == NULL )
    {
      fprintf( stderr , "Error. Coundn't bind java class java.lang.Class\n" );
      exit( 1 );
    }

    java_lang_class = ( *env )->NewGlobalRef( env , tempClass );

    if ( java_lang_class == NULL )
    {
      fprintf( stderr , "Error. Couldn't bind java class java.lang.Throwable\n" );
      exit( 1 );
    }
  }

  pushJNIEnv( env , L );
}

```

分几步进行操作。

### getStateFromCPtr

简单的描述一下就是，在 C 内通过对象 cptr 查询出去字段 peer 的值，然后强制转换成 lua_State 。

因为我们在 _open 函数中，是将地址直接存在 CPtr 的长整型字段 peer 内的。

也就是说，lua_State 在 Java 内的表现是一个 CPtr。当我们想要执行 Lua 代码的时候，就必须通过 Java 内的表现来找到具体 C 内的对象是。

```c
lua_State * getStateFromCPtr( JNIEnv * env , jobject cptr )
{
   lua_State * L;

   jclass classPtr       = ( *env )->GetObjectClass( env , cptr );
   jfieldID CPtr_peer_ID = ( *env )->GetFieldID( env , classPtr , "peer" , "J" );
   jbyte * peer          = ( jbyte * ) ( *env )->GetLongField( env , cptr , CPtr_peer_ID );

   L = ( lua_State * ) peer;

   pushJNIEnv( env ,  L );

   return L;
}

```

### pushJNIEnv

这一步非常的关键。在 lua_State 内，对 JNIEnv 进行引用，实际上就是表示为一个 full userdata，然后放在全局注册表内，键为：LUAJAVAJNIENVTAG。

这个函数会被调用两次：一次是在 getStateFromCPtr的时候（只会建立 userdata），一次是在 luajava_open 完成的时候的时候（真正压入环境）

```c
void pushJNIEnv( JNIEnv * env , lua_State * L )
{
   JNIEnv ** udEnv;

   lua_pushstring( L , LUAJAVAJNIENVTAG );
   lua_rawget( L , LUA_REGISTRYINDEX );

   if ( !lua_isnil( L , -1 ) )
   {
      udEnv = ( JNIEnv ** ) lua_touserdata( L , -1 );
      *udEnv = env;
      lua_pop( L , 1 );
   }
   else
   {
      lua_pop( L , 1 );
      udEnv = ( JNIEnv ** ) lua_newuserdata( L , sizeof( JNIEnv * ) );
      *udEnv = env;

      lua_pushstring( L , LUAJAVAJNIENVTAG );
      lua_insert( L , -2 );
      lua_rawset( L , LUA_REGISTRYINDEX );
   }
}
```

### 记录 lua_State 在 Java 内的索引值

```c
  lua_pushstring( L , LUAJAVASTATEINDEX );
  lua_pushnumber( L , (lua_Number)stateId );
  lua_settable( L , LUA_REGISTRYINDEX );
```

这个没啥说的，将索引放在全局注册表内。

###  luajava 表

```c
  lua_newtable( L );

  lua_setglobal( L , "luajava" );

  lua_getglobal( L , "luajava" );
```



### luajava 常量设置



```c
static void set_info (lua_State *L) {
	lua_pushliteral (L, "_COPYRIGHT");
	lua_pushliteral (L, "Copyright (C) 2003-2007 Kepler Project");
	lua_settable (L, -3);
	lua_pushliteral (L, "_DESCRIPTION");
	lua_pushliteral (L, "LuaJava is a script tool for Java");
	lua_settable (L, -3);
	lua_pushliteral (L, "_NAME");
	lua_pushliteral (L, "LuaJava");
	lua_settable (L, -3);
	lua_pushliteral (L, "_VERSION");
	lua_pushliteral (L, "1.1");
	lua_settable (L, -3);
}
```

### luajava  关键函数

在这里会将几个关键的函数注册到 lua_State 的注册表内，这是 lua 能够调用  Java 的关键所在：

```lua
{
  "bindClass": javaBindClass,
  "new": javaNew,
  "newInstance": javaNewInstance,
  "loadLib": javaLoadLib,
  "createProxy": createProxy,
}
```



```
  lua_pushstring( L , "bindClass" );
  lua_pushcfunction( L , &javaBindClass );
  lua_settable( L , -3 );

  lua_pushstring( L , "new" );
  lua_pushcfunction( L , &javaNew );
  lua_settable( L , -3 );

  lua_pushstring( L , "newInstance" );
  lua_pushcfunction( L , &javaNewInstance );
  lua_settable( L , -3 );

  lua_pushstring( L , "loadLib" );
  lua_pushcfunction( L , &javaLoadLib );
  lua_settable( L , -3 );

  lua_pushstring( L , "createProxy" );
  lua_pushcfunction( L , &createProxy );
  lua_settable( L , -3 );

  lua_pop( L , 1 );

```



### luajava 关键类

现在我们考虑这么一个问题，我们可以把 Java 的 对象，方法，等传递给 Lua，但是如何自动的将相关的信息传递过去呢？比如一个 Java 对象，如何能够直接调用到 对象中的方法呢？所以，在 Java 内，实现了一些给 Lua 调用的方法。

有了几个全局函数还不够，因为这几个全局函数还需要一些 Java 内的桥梁来进行交互。

- **java_function_class -> JavaFunction.java** 用来实现 Lua 中的函数。这是一个抽象类，我们必须实现 `execute()` 方法。我们从 Lua 中调用这个 Java 函数的时候，就会调用到这个 `execute()` 方法。可以用 `register(String name)` 来将此函数注册到 Lua中。
- **java_function_method  -> JavaFunction.execute()**  默认的 JavaFunction 方法。
- **throwable_class -> java/lang/Throwable**  违例类
- **get_message_method ->  java/lang/Throwable.getMessage() **  获取异常信息
- **java_lang_class -> java/lanb/String** 默认使用的 String 类。
- **lua_java_api_class -> LuaJavaAPI.java** 这是对于 Lua 中关于元表等概念在 Java 对象上的实现。

现在 Java 层面的东西在 Lua 内的表示和桥梁已经打通。

```c
static jclass    throwable_class      = NULL;
static jmethodID get_message_method   = NULL;
static jclass    java_function_class  = NULL;
static jmethodID java_function_method = NULL;
static jclass    luajava_api_class    = NULL;
static jclass    java_lang_class      = NULL;
```

下面的这些代码，也就是将 Java 中的类传递给 luajava 进行保存， Lua 在操纵 Java 的时候会需要：

```c
  if ( luajava_api_class == NULL )
  {
    tempClass = ( *env )->FindClass( env , "org/keplerproject/luajava/LuaJavaAPI" );

    if ( tempClass == NULL )
    {
      fprintf( stderr , "Could not find LuaJavaAPI class\n" );
      exit( 1 );
    }

    if ( ( luajava_api_class = ( *env )->NewGlobalRef( env , tempClass ) ) == NULL )
    {
      fprintf( stderr , "Could not bind to LuaJavaAPI class\n" );
      exit( 1 );
    }
  }

  if ( java_function_class == NULL )
  {
    tempClass = ( *env )->FindClass( env , "org/keplerproject/luajava/JavaFunction" );

    if ( tempClass == NULL )
    {
      fprintf( stderr , "Could not find JavaFunction interface\n" );
      exit( 1 );
    }

    if ( ( java_function_class = ( *env )->NewGlobalRef( env , tempClass ) ) == NULL )
    {
      fprintf( stderr , "Could not bind to JavaFunction interface\n" );
      exit( 1 );
    }
  }

  if ( java_function_method == NULL )
  {
    java_function_method = ( *env )->GetMethodID( env , java_function_class , "execute" , "()I");
    if ( !java_function_method )
    {
      fprintf( stderr , "Could not find <execute> method in JavaFunction\n" );
      exit( 1 );
    }
  }

  if ( throwable_class == NULL )
  {
    tempClass = ( *env )->FindClass( env , "java/lang/Throwable" );

    if ( tempClass == NULL )
    {
      fprintf( stderr , "Error. Couldn't bind java class java.lang.Throwable\n" );
      exit( 1 );
    }

    throwable_class = ( *env )->NewGlobalRef( env , tempClass );

    if ( throwable_class == NULL )
    {
      fprintf( stderr , "Error. Couldn't bind java class java.lang.Throwable\n" );
      exit( 1 );
    }
  }

  if ( get_message_method == NULL )
  {
    get_message_method = ( *env )->GetMethodID( env , throwable_class , "getMessage" ,
                                                "()Ljava/lang/String;" );

    if ( get_message_method == NULL )
    {
      fprintf(stderr, "Could not find <getMessage> method in java.lang.Throwable\n");
      exit(1);
    }
  }

  if ( java_lang_class == NULL )
  {
    tempClass = ( *env )->FindClass( env , "java/lang/Class" );

    if ( tempClass == NULL )
    {
      fprintf( stderr , "Error. Coundn't bind java class java.lang.Class\n" );
      exit( 1 );
    }

    java_lang_class = ( *env )->NewGlobalRef( env , tempClass );

    if ( java_lang_class == NULL )
    {
      fprintf( stderr , "Error. Couldn't bind java class java.lang.Throwable\n" );
      exit( 1 );
    }
  }

  pushJNIEnv( env , L );

```

## pushJavaObject

现在我们来看看如何将一个 Java对象交给 Lua 使用。

过程也不是很复杂：

1. Java 调用 pushJavaObject 原生方法。
2. 通过 lua_State 获取 JNI 环境（这个之前已经存在注册表内了）
3. 在 C 内用 globalRef 引用  Java 对象。
4. 在 Lua 内以 userdata 的形式，表示  globalRef 。
5. 设置 userdata 元表。元表的元方法 `__index, __newindex,_gc` 等都会进行设置。

```c
int pushJavaObject( lua_State * L , jobject javaObject )
{
   jobject * userData , globalRef;

   /* Gets the JNI Environment */
   JNIEnv * javaEnv = getEnvFromState( L );
   if ( javaEnv == NULL )
   {
      lua_pushstring( L , "Invalid JNI Environment." );
      lua_error( L );
   }

   globalRef = ( *javaEnv )->NewGlobalRef( javaEnv , javaObject );

   userData = ( jobject * ) lua_newuserdata( L , sizeof( jobject ) );
   *userData = globalRef;

   /* Creates metatable */
   lua_newtable( L );

   /* pushes the __index metamethod */
   lua_pushstring( L , LUAINDEXMETAMETHODTAG );
   lua_pushcfunction( L , &objectIndex );
   lua_rawset( L , -3 );

   /* pushes the __newindex metamethod */
   lua_pushstring( L , LUANEWINDEXMETAMETHODTAG );
   lua_pushcfunction( L , &objectNewIndex );
   lua_rawset( L , -3 );

   /* pushes the __gc metamethod */
   lua_pushstring( L , LUAGCMETAMETHODTAG );
   lua_pushcfunction( L , &gc );
   lua_rawset( L , -3 );

   /* Is Java Object boolean */
   lua_pushstring( L , LUAJAVAOBJECTIND );
   lua_pushboolean( L , 1 );
   lua_rawset( L , -3 );

   if ( lua_setmetatable( L , -2 ) == 0 )
   {
		( *javaEnv )->DeleteGlobalRef( javaEnv , globalRef );
      lua_pushstring( L , "Cannot create proxy to java object." );
      lua_error( L );
   }

   return 1;
}

```

## objectIndex

简单描述一下这个过程：

1. 拿到当前 lua_State 的在 luajava 内的索引。
2. 在 Lua 中，对于 __index 元方法是一个函数的，会以 (t, k) 参数进行调用。那么我们现在就需要拿到键 key。
3. 通过 JavaAPI 中的 checkField 方法来检查，Java 对象内是否含有 对应的键（实际上利用的是反射）。如果检查到有这个字段，那么就直接将此字段的值压入 Lua 栈内。实现在 JavaAPI.java 这个类的 checkField 方法内。
4. 如果找不到，那么会在对象的元表内，放入一个 `__FunctionCalled` 的元方法，其值就是 key 。接着会将 objectIndexReturn 函数压入栈。
5. 之后就调用这个返回的函数了

**问题是：如果是我确实只是想要应该字段的值，但是这个字段在 Java 对象不存在，他给我返回的是一个函数，那么我是调用呢还是不调用呢？**

结果是，确实是的如果字段不存在，那么他会返回一个函数，但是调用这个函数没有卵用，因为没这个方法啊。

测试代码：

```lua
str = luajava.bindClass("java.lang.String")
strInstance = luajava.new(str,"Hello Java")
local aa = strInstance.count
print(strInstance:concat("fff"))
print(aa)
print(aa())

> Hello Javafff
> function: 0x7f8e8efc09a0
> Not a valid OO function call.
```



```c
int objectIndex( lua_State * L )
{
   /* 当进行 index 元方法是一个函数的时候,会以 table, key 作为参数调用 __index 方法
    * 栈结构： table(obj), key
    */
   lua_Number stateIndex;
   const char * key;
   jmethodID method;
   jint checkField;
   jobject * obj;
   jstring str;
   jthrowable exp;
   JNIEnv * javaEnv;

   /* Gets the luaState index */
   lua_pushstring( L , LUAJAVASTATEINDEX );
   lua_rawget( L , LUA_REGISTRYINDEX );

   if ( !lua_isnumber( L , -1 ) )
   {
      lua_pushstring( L , "Impossible to identify luaState id." );
      lua_error( L );
   }

   stateIndex = lua_tonumber( L , -1 );
   lua_pop( L , 1 );

   if ( !lua_isstring( L , -1 ) )
   {
      lua_pushstring( L , "Invalid object index. Must be string." );
      lua_error( L );
   }

   key = lua_tostring( L , -1 );

   if ( !isJavaObject( L , 1 ) )
   {
      lua_pushstring( L , "Not a valid Java Object." );
      lua_error( L );
   }

   javaEnv = getEnvFromState( L );
   if ( javaEnv == NULL )
   {
      lua_pushstring( L , "Invalid JNI Environment." );
      lua_error( L );
   }

   obj = ( jobject * ) lua_touserdata( L , 1 );

   method = ( *javaEnv )->GetStaticMethodID( javaEnv , luajava_api_class , "checkField" ,
                                             "(ILjava/lang/Object;Ljava/lang/String;)I" );

   str = ( *javaEnv )->NewStringUTF( javaEnv , key );

   checkField = ( *javaEnv )->CallStaticIntMethod( javaEnv , luajava_api_class , method ,
                                                   (jint)stateIndex , *obj , str );

   exp = ( *javaEnv )->ExceptionOccurred( javaEnv );

   /* Handles exception */
   if ( exp != NULL )
   {
      jobject jstr;
      const char * cStr;
      
      ( *javaEnv )->ExceptionClear( javaEnv );
      jstr = ( *javaEnv )->CallObjectMethod( javaEnv , exp , get_message_method );

      ( *javaEnv )->DeleteLocalRef( javaEnv , str );

      if ( jstr == NULL )
      {
         jmethodID methodId;

         methodId = ( *javaEnv )->GetMethodID( javaEnv , throwable_class , "toString" , "()Ljava/lang/String;" );
         jstr = ( *javaEnv )->CallObjectMethod( javaEnv , exp , methodId );
      }

      cStr = ( *javaEnv )->GetStringUTFChars( javaEnv , jstr , NULL );

      lua_pushstring( L , cStr );

      ( *javaEnv )->ReleaseStringUTFChars( javaEnv , jstr, cStr );

      lua_error( L );
   }

   ( *javaEnv )->DeleteLocalRef( javaEnv , str );

   if ( checkField != 0 )
   {
      return checkField;
   }

   lua_getmetatable( L , 1 );

   if ( !lua_istable( L , -1 ) )
   {
      lua_pushstring( L , "Invalid MetaTable." );
      lua_error( L );
   }

   lua_pushstring( L , LUAJAVAOBJFUNCCALLED );
   lua_pushstring( L , key );
   lua_rawset( L , -3 );

   lua_pop( L , 1 );

   lua_pushcfunction( L , &objectIndexReturn );

   return 1;
}

```

objectIndexReturn 这个没啥好说的，原理差不多，都是利用反射的形式，来找到对应的对象方法，进行调用，调用后结果直接压入 Lua 栈上。



# 几个关键函数

```lua
{
  "bindClass": javaBindClass,
  "new": javaNew,
  "newInstance": javaNewInstance,
  "loadLib": javaLoadLib,
  "createProxy": createProxy,
}
```

## javaBindClass

想要在 Lua 中使用  Java 的类，首先就需要将这个类绑定到 Lua。

可以看到，其基本的搞法，就是将一个类名称，通过 Class.forName() 的形式 返回一个 Class 对象，然后调用 pushJavaClass 给压到 lua_State 内去。

```cpp
/***************************************************************************
*
*  Function: javaBindClass
*  ****/

int javaBindClass( lua_State * L )
{
   int top;
   jmethodID method;
   const char * className;
   jstring javaClassName;
   jobject classInstance;
   jthrowable exp;
   JNIEnv * javaEnv;

   top = lua_gettop( L );

   if ( top != 1 )
   {
      luaL_error( L , "Error. Function javaBindClass received %d arguments, expected 1." , top );
   }

   /* Gets the JNI Environment */
   javaEnv = getEnvFromState( L );
   if ( javaEnv == NULL )
   {
      lua_pushstring( L , "Invalid JNI Environment." );
      lua_error( L );
   }

   /* get the string parameter */
   if ( !lua_isstring( L , 1 ) )
   {
      lua_pushstring( L , "Invalid parameter type. String expected." );
      lua_error( L );
   }
   className = lua_tostring( L , 1 );

   method = ( *javaEnv )->GetStaticMethodID( javaEnv , java_lang_class , "forName" , 
                                             "(Ljava/lang/String;)Ljava/lang/Class;" );

   javaClassName = ( *javaEnv )->NewStringUTF( javaEnv , className );

   classInstance = ( *javaEnv )->CallStaticObjectMethod( javaEnv , java_lang_class ,
                                                         method , javaClassName );

   exp = ( *javaEnv )->ExceptionOccurred( javaEnv );

   /* Handles exception */
   if ( exp != NULL )
   {
      jobject jstr;
      const char * cStr;
      
      ( *javaEnv )->ExceptionClear( javaEnv );
      jstr = ( *javaEnv )->CallObjectMethod( javaEnv , exp , get_message_method );

      ( *javaEnv )->DeleteLocalRef( javaEnv , javaClassName );

      if ( jstr == NULL )
      {
         jmethodID methodId;

         methodId = ( *javaEnv )->GetMethodID( javaEnv , throwable_class , "toString" , "()Ljava/lang/String;" );
         jstr = ( *javaEnv )->CallObjectMethod( javaEnv , exp , methodId );
      }

      cStr = ( *javaEnv )->GetStringUTFChars( javaEnv , jstr , NULL );

      lua_pushstring( L , cStr );

      ( *javaEnv )->ReleaseStringUTFChars( javaEnv , jstr, cStr );

      lua_error( L );
   }

   ( *javaEnv )->DeleteLocalRef( javaEnv , javaClassName );

   /* pushes new object into lua stack */

   return pushJavaClass( L , classInstance );
}


```

### pushJavaClass

OK，将 Java Class 实例，在 Lua 中用 userdata 进行存储，同时设置其元表的 **__gc, __index, __newindex, ___IsJavaObject** 元方法。

```cpp
/***************************************************************************
*
*  Function: pushJavaClass
*  ****/

int pushJavaClass( lua_State * L , jobject javaObject )
{
   jobject * userData , globalRef;

   /* Gets the JNI Environment */
   JNIEnv * javaEnv = getEnvFromState( L );
   if ( javaEnv == NULL )
   {
      lua_pushstring( L , "Invalid JNI Environment." );
      lua_error( L );
   }

   globalRef = ( *javaEnv )->NewGlobalRef( javaEnv , javaObject );

   userData = ( jobject * ) lua_newuserdata( L , sizeof( jobject ) );
   *userData = globalRef;

   /* Creates metatable */
   lua_newtable( L );

   /* pushes the __index metamethod */
   lua_pushstring( L , LUAINDEXMETAMETHODTAG );
   lua_pushcfunction( L , &classIndex );
   lua_rawset( L , -3 );

   /* pushes the __newindex metamethod */
   lua_pushstring( L , LUANEWINDEXMETAMETHODTAG );
   lua_pushcfunction( L , &objectNewIndex );
   lua_rawset( L , -3 );

   /* pushes the __gc metamethod */
   lua_pushstring( L , LUAGCMETAMETHODTAG );
   lua_pushcfunction( L , &gc );
   lua_rawset( L , -3 );

   /* Is Java Object boolean */
   lua_pushstring( L , LUAJAVAOBJECTIND );
   lua_pushboolean( L , 1 );
   lua_rawset( L , -3 );

   if ( lua_setmetatable( L , -2 ) == 0 )
   {
		( *javaEnv )->DeleteGlobalRef( javaEnv , globalRef );
      lua_pushstring( L , "Cannot create proxy to java class." );
      lua_error( L );
   }

   return 1;
}

```

## javaNew

有了 Class 实力，我们就可以用其来进行实例化其他对象。最终是调用的 **LuaJavaAPI.javaNew** 来得出一个类的实例了，得到类的时候后，就会调用 pushJavaObject 来压入到 lua_State 中。

```cpp
int javaNew( lua_State * L )
{
   int top;
   jint ret;
   jclass clazz;
   jmethodID method;
   jobject classInstance ;
   jthrowable exp;
   jobject * userData;
   lua_Number stateIndex;
   JNIEnv * javaEnv;

   top = lua_gettop( L );

   if ( top == 0 )
   {
      lua_pushstring( L , "Error. Invalid number of parameters." );
      lua_error( L );
   }

   /* Gets the luaState index */
   lua_pushstring( L , LUAJAVASTATEINDEX );
   lua_rawget( L , LUA_REGISTRYINDEX );

   if ( !lua_isnumber( L , -1 ) )
   {
      lua_pushstring( L , "Impossible to identify luaState id." );
      lua_error( L );
   }

   stateIndex = lua_tonumber( L , -1 );
   lua_pop( L , 1 );

   /* Gets the java Class reference */
   if ( !isJavaObject( L , 1 ) )
   {
      lua_pushstring( L , "Argument not a valid Java Class." );
      lua_error( L );
   }

   /* Gets the JNI Environment */
   javaEnv = getEnvFromState( L );
   if ( javaEnv == NULL )
   {
      lua_pushstring( L , "Invalid JNI Environment." );
      lua_error( L );
   }

   clazz = ( *javaEnv )->FindClass( javaEnv , "java/lang/Class" );

   userData = ( jobject * ) lua_touserdata( L , 1 );

   classInstance = ( jobject ) *userData;

   if ( ( *javaEnv )->IsInstanceOf( javaEnv , classInstance , clazz ) == JNI_FALSE )
   {
      lua_pushstring( L , "Argument not a valid Java Class." );
      lua_error( L );
   }

   method = ( *javaEnv )->GetStaticMethodID( javaEnv , luajava_api_class , "javaNew" , 
                                             "(ILjava/lang/Class;)I" );

   if ( clazz == NULL || method == NULL )
   {
      lua_pushstring( L , "Invalid method org.keplerproject.luajava.LuaJavaAPI.javaNew." );
      lua_error( L );
   }

   ret = ( *javaEnv )->CallStaticIntMethod( javaEnv , clazz , method , (jint)stateIndex , classInstance );

   exp = ( *javaEnv )->ExceptionOccurred( javaEnv );

   /* Handles exception */
   if ( exp != NULL )
   {
      jobject jstr;
      const char * str;
      
      ( *javaEnv )->ExceptionClear( javaEnv );
      jstr = ( *javaEnv )->CallObjectMethod( javaEnv , exp , get_message_method );

      if ( jstr == NULL )
      {
         jmethodID methodId;

         methodId = ( *javaEnv )->GetMethodID( javaEnv , throwable_class , "toString" , "()Ljava/lang/String;" );
         jstr = ( *javaEnv )->CallObjectMethod( javaEnv , exp , methodId );
      }

      str = ( *javaEnv )->GetStringUTFChars( javaEnv , jstr , NULL );

      lua_pushstring( L , str );

      ( *javaEnv )->ReleaseStringUTFChars( javaEnv , jstr, str );

      lua_error( L );
   }
  return ret;
}

```

## javaNewInstance

这个方法与 javaNew 不他的，其不需要一个 Class 对象作为参数，而是以类名作为参数，不需要 bindClass 即可使用。

对应的直接调用  LuaJavaAPI.javaNewInstance 进行实例化。在此方法中还是用了 Class.forName 的形式来获取类对象，然后反射获取构造器进行构造。最终调用  pushJavaObject。

```cpp
int javaNewInstance( lua_State * L )
{
   jint ret;
   jmethodID method;
   const char * className;
   jstring javaClassName;
   jthrowable exp;
   lua_Number stateIndex;
   JNIEnv * javaEnv;

   /* Gets the luaState index */
   lua_pushstring( L , LUAJAVASTATEINDEX );
   lua_rawget( L , LUA_REGISTRYINDEX );

   if ( !lua_isnumber( L , -1 ) )
   {
      lua_pushstring( L , "Impossible to identify luaState id." );
      lua_error( L );
   }

   stateIndex = lua_tonumber( L , -1 );
   lua_pop( L , 1 );

   /* get the string parameter */
   if ( !lua_isstring( L , 1 ) )
   {
      lua_pushstring( L , "Invalid parameter type. String expected as first parameter." );
      lua_error( L );
   }

   className = lua_tostring( L , 1 );

   /* Gets the JNI Environment */
   javaEnv = getEnvFromState( L );
   if ( javaEnv == NULL )
   {
      lua_pushstring( L , "Invalid JNI Environment." );
      lua_error( L );
   }

   method = ( *javaEnv )->GetStaticMethodID( javaEnv , luajava_api_class , "javaNewInstance" ,
                                             "(ILjava/lang/String;)I" );

   javaClassName = ( *javaEnv )->NewStringUTF( javaEnv , className );
   
   ret = ( *javaEnv )->CallStaticIntMethod( javaEnv , luajava_api_class , method, (jint)stateIndex , 
                                            javaClassName );

   exp = ( *javaEnv )->ExceptionOccurred( javaEnv );

   /* Handles exception */
   if ( exp != NULL )
   {
      jobject jstr;
      const char * str;
      
      ( *javaEnv )->ExceptionClear( javaEnv );
      jstr = ( *javaEnv )->CallObjectMethod( javaEnv , exp , get_message_method );

      ( *javaEnv )->DeleteLocalRef( javaEnv , javaClassName );

      if ( jstr == NULL )
      {
         jmethodID methodId;

         methodId = ( *javaEnv )->GetMethodID( javaEnv , throwable_class , "toString" , "()Ljava/lang/String;" );
         jstr = ( *javaEnv )->CallObjectMethod( javaEnv , exp , methodId );
      }

      str = ( *javaEnv )->GetStringUTFChars( javaEnv , jstr , NULL );

      lua_pushstring( L , str );

      ( *javaEnv )->ReleaseStringUTFChars( javaEnv , jstr, str );

      lua_error( L );
   }

   ( *javaEnv )->DeleteLocalRef( javaEnv , javaClassName );

   return ret;
}

```

## javaLoadLib

用指定的类来加载 lib 没啥说的。

## createProxy(string,table)

这 个方法用到了建立多个接口的代理类。

```java
  /**
   * Function that creates an object proxy and pushes it into the stack
   * 
   * @param luaState int that represents the state to be used
   * @param implem interfaces implemented separated by comma (<code>,</code>)
   * @return number of returned objects
   * @throws LuaException
   */
  public static int createProxyObject(int luaState, String implem)
    throws LuaException
  {
    LuaState L = LuaStateFactory.getExistingState(luaState);

    synchronized (L)
    {
      try
      {
        if (!(L.isTable(2)))
          throw new LuaException(
              "Parameter is not a table. Can't create proxy.");

        LuaObject luaObj = L.getLuaObject(2);

        Object proxy = luaObj.createProxy(implem);
        L.pushJavaObject(proxy);
      }
      catch (Exception e)
      {
        throw new LuaException(e);
      }

      return 1;
    }
  }

```



# 使用

我们通过 luajava 能够操作 lua 虚拟机，但是当前，我们的入口是在 Java 中，所以在 Java 层，事实上是实现了对于 Lua 的一些初始化的操作，和将 Lua 代码压入 Lua 进行执行的这些逻辑。

从概念上讲，其是在 Java 层实现了对 Lua 中一些概念的封装。我们来简单的看一下这几个 Java 类：



- CPtr.java  对一个 C 指针类型的 Java 抽象，事实上就是用一个 long 类型的成员字段来保存 C 中的指针。我们前文说过的就是，对于一个 lua_State 实际上就是将其指针保存在这个 CPtr 内的，在使用的时候，会将这个 long 值强制转换为指针。
- JavaFunction.java 用来实现 Lua 中的函数。这是一个抽象类，我们必须实现 `execute()` 方法。我们从 Lua 中调用这个 Java 函数的时候，就会调用到这个 `execute()` 方法。可以用 `register(String name)` 来将此函数注册到 Lua中。
- LuaException.java 异常类。
- LuaInvocationHandler.java 实现 InvocationHandler 接口。在 LuaJava 中的代理系统内使用。用来从 Lua 中访问一个代理对象的方法。
- LuaJavaAPI.java 这是对于 Lua 中关于元表等概念在 Java 对象上的实现。
- LuaObject.java Lua 对象在 Java 中的实现。
- LuaState.java lua_State 的实现。包括了很多针对 lua_State 的 C API 实现。我们在 Java 层对 lua_State 的所有操作，就在封装在此，其本质，还是通过 JNI 接口，来访问 luajava 持有的 lua_State。
- LuaStateFactory.java 用来初始化一个 lua_State。
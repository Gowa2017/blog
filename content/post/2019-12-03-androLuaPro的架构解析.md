---
title: androLuaPro的架构解析
categories:
  - [Lua]
  - [Android]
date: 2019-12-03 15:55:09
updated: 2019-12-03 15:55:09
tags: 
  - Lua
  - Android
---
架不住喜欢 Lua，所以看了一下 Lua 在安卓上的使用，但是项目都不多。最终找到了一个击于 [luajava](https://github.com/jasonsantos/luajava) 的实现  [AndroiLua_pro](https://github.com/nirenr/AndroLua_pro)。下面我们来看一下他是怎么样让 Lua 用起来的。

<!--more-->

关于 LuaJava 的解析可以看 {% post_link JNI使用-LuaJava实现 JNI使用-LuaJava实现%}。

# LuaApplication

LuaApplication  在启动的时候定义了几个用来存储模块和脚本的路径：

```java
    @Override
    public void onCreate() {
        super.onCreate();
        mApp = this;
        CrashHandler crashHandler = CrashHandler.getInstance();
        // 注册crashHandler
        crashHandler.init(getApplicationContext());
        mSharedPreferences = getSharedPreferences(this);
        //初始化AndroLua工作目录
        if (Environment.getExternalStorageState().equals(Environment.MEDIA_MOUNTED)) {
            String sdDir = Environment.getExternalStorageDirectory().getAbsolutePath();
            luaExtDir = sdDir + "/AndroLua";
        } else {
            File[] fs = new File("/storage").listFiles();
            for (File f : fs) {
                String[] ls = f.list();
                if (ls == null) {
                    continue;
                }
                if (ls.length > 5) {
                    luaExtDir = f.getAbsolutePath() + "/AndroLua";
                }
            }
            if (luaExtDir == null) {
                luaExtDir = getDir("AndroLua", Context.MODE_PRIVATE).getAbsolutePath();
            }
        }

        File destDir = new File(luaExtDir);
        if (!destDir.exists()) {
            destDir.mkdirs();
        }

        //定义文件夹
        luaFilesDir = getFilesDir().getAbsolutePath();
        odexDir = getDir("odex", Context.MODE_PRIVATE).getAbsolutePath();
        luaNativeLibDir = getDir("lib", Context.MODE_PRIVATE).getAbsolutePath();
        luaModuleDir = getDir("lua", Context.MODE_PRIVATE).getAbsolutePath();
        luaCpath = getApplicationInfo().nativeLibraryDir + "/lib?.so" + ";" + luaNativeLibDir + "/lib?.so";
        //luaDir = extDir;
        luaLpath = luaModuleDir + "/?.lua;" + luaModuleDir + "/lua/?.lua;" + luaModuleDir + "/?/init.lua;";
        //checkInfo();
    }

```
简单来说就是这几个路径：

- luaExtDir：/sdcard/AndroLua 存储于外部存储中的目录，lua文件目录。
- appFilesDir： /data/data/com.androlua/files
- odexDir:  /data/data/com.androlua/app_odex
- luaNativeLibDir: /data/data/com.androlua/app_lib  Lua So 库路径
- luaModuleDir: /data/data/com.androlua/app_lua 
- luaCpath： /data/app/com.androlua.../lib 加上我们的 libDir 目录，在此下查找 so 库。
- luaLpath：在 luaModuleDir 下查找的 lua 模块路径，在此路径下查找 .lua 文件 

# Welcome

这个是在第一次启动，或者是有更新的时候启动的 Activity，其做的一个最重要的工作就是：
将 apk 内 assets 目录下的文件解压到 appFilesDir。
将 apk 内 lua （位于 resources ）目录下的文件解压到 luaModuleDir。

所以看着有点蛋疼。

```java
                unApk("assets", appFilesDir);
                unApk("lua", luaModuleDir);

```

```java
        private void unApk(String dir, String extDir) throws IOException {
            int i = dir.length() + 1;
            ZipFile zip = new ZipFile(getApplicationInfo().publicSourceDir);
            Enumeration<? extends ZipEntry> entries = zip.entries();
            while (entries.hasMoreElements()) {
                ZipEntry entry = entries.nextElement();
                String name = entry.getName();
                if (name.indexOf(dir) != 0)
                    continue;
                String path = name.substring(i);
                if (entry.isDirectory()) {
                    File f = new File(extDir + File.separator + path);
                    if (!f.exists()) {
                        //noinspection ResultOfMethodCallIgnored
                        f.mkdirs();
                    }
                } else {
                    String fname = extDir + File.separator + path;
                    File ff = new File(fname);
                    File temp = new File(fname).getParentFile();
                    if (!temp.exists()) {
                        if (!temp.mkdirs()) {
                            throw new RuntimeException("create file " + temp.getName() + " fail");
                        }
                    }
                    try {
                        if (ff.exists() && entry.getSize() == ff.length() && LuaUtil.getFileMD5(zip.getInputStream(entry)).equals(LuaUtil.getFileMD5(ff)))
                            continue;
                    } catch (NullPointerException ignored) {
                    }
                    FileOutputStream out = new FileOutputStream(extDir + File.separator + path);
                    InputStream in = zip.getInputStream(entry);
                    byte[] buf = new byte[4096];
                    int count = 0;
                    while ((count = in.read(buf)) != -1) {
                        out.write(buf, 0, count);
                    }
                    out.close();
                    in.close();
                }
            }
            zip.close();
        }

    }

```

其中 getApplicationInfo().publicSourceDir 获取的是我们安装后的 base.apk 路径，相当于是将 apk (zip压缩包)内的目录解压到指定目录。

# LuaActivity

## initLua()

在建立的时候会初始化一个 Lua 虚拟机，同时压入很多的全局变量：

```java
    private void initLua() throws Exception {
        L = LuaStateFactory.newLuaState();
        L.openLibs();
        // 压入 this, activity 两个全局变量，其都代表了 Activity 本身。
        L.pushJavaObject(this);
        L.setGlobal("activity");
        L.getGlobal("activity");
        L.setGlobal("this");
        // 压入全局变量 _LuaContext 表示当前虚拟机所在的上下文信息。
        L.pushContext(this);
        
        // 压入一些路径信息给 luajava
        L.getGlobal("luajava");
        L.pushString(luaExtDir);
        L.setField(-2, "luaextdir");
        L.pushString(luaDir);
        L.setField(-2, "luadir");
        L.pushString(luaPath);
        L.setField(-2, "luapath");
        L.pop(1);
        initENV();

        // 压入一个 print 函数
        JavaFunction print = new LuaPrint(this, L);
        print.register("print");

        // 设置包查找路径
        L.getGlobal("package");
        L.pushString(luaLpath);
        L.setField(-2, "path");
        L.pushString(luaCpath);
        L.setField(-2, "cpath");
        L.pop(1);

        // set/call 函数
        JavaFunction set = new JavaFunction(L) {
            @Override
            public int execute() throws LuaException {
                LuaThread thread = (LuaThread) L.toJavaObject(2);

                thread.set(L.toString(3), L.toJavaObject(4));
                return 0;
            }
        };
        set.register("set");

        JavaFunction call = new JavaFunction(L) {
            @Override
            public int execute() throws LuaException {
                LuaThread thread = (LuaThread) L.toJavaObject(2);

                int top = L.getTop();
                if (top > 3) {
                    Object[] args = new Object[top - 3];
                    for (int i = 4; i <= top; i++) {
                        args[i - 4] = L.toJavaObject(i);
                    }
                    thread.call(L.toString(3), args);
                } else if (top == 3) {
                    thread.call(L.toString(3));
                }

                return 0;
            }

            ;
        };
        call.register("call");

    }

```

他做了这么一个假设，就是每个 Activity 都应该有一个需要执行的 Lua 文件，这个是通过 Intent 传递过来的，所以就需要从 Intent 里面拿出这个 Lua 文件路径。

## getLuaFilePath

这个方法有三个重载。



```java
// 从 intent 获取 lua 文件
public String getLuaFilePath();

// 从指定的目录下获取 lua 文件。目录默认情况下是设置为 appFilesDir 的
public String getLuaFilePath(String path);

// 没用到
public String getLuaFilePath(String dir, String name);
```
---
title: Android中的两大框架之一Xposed
categories:
  - Android
date: 2020-08-01 23:45:39
updated: 2020-08-01 23:45:39
tags: 
  - Android
  - Xposed
---

在 {% post_link Android系统的启动过程 Android系统的启动过程 %} 过程中，我们知道，Android 启动在启动的时候，首先会调用 `app_process` 这个程序，来启动一个 Zygote 进程，然后其他所有的进程都会有这个进程进行 `fork` 而来。因此，在这个进程里面所加载的任何内容，在新的 APP 进程中都是可用的，所以呢，这 Xposed 和 Magsik 都利用了子进程会继承父进程环境的这么一个事实来进行实现。

<!--more-->

在那篇文章我们提到了两个概念：

- AndroidRuntime 每个安卓进程都需要的运行时
- AppRuntime 应用运行时

AppRuntime 继承自 AndroidRuntime ，定义了一些回调方法：

- onExit(int code) 进程退出时回调
- onVmCreated(JNIEnv* env) JVM 建立后回调，在调用了这个回调后，AndroidRuntime 内会调用  `startReg()` 来注册很多 Android Native 层内容。
- onStarted() 非 Zygote 进程启动会由 *com.android.internal.os.RuntimeInit* 回调
- onZygoteInit() Zygote 进程启动的，由 *com.android.internal.os.ZygoteInit* 时候回调

# Xposed

Xposed 的搞法，是通过替换了我们的 app_process 程序，具体来说，是自己写了一个 AppRunTime，把回调都进行了一些变化处理，在 onVmCreated 中，加入属于 Xposed 的东西：

1. 先初始化 xposed 环境。
2. 在 `onVmCreated ` 回调中调用同名方法。

# C 层

## xposed::initialize

在命名空间 xposed 定义了一个全局变量  `XposedShared *xposed`。这个函数主要做两个事情：

1. 启动 xposed 的服务 `xposed::service::startAll()`
2. 添加一个 Jar */system/framework/XposedBridge.jar* 到 classpath 变量里面。`addJarToClasspath()`


### xposed::service::startAll()

这个开了两个子进程，分别来启动 `systemService()` 和 `appService()`。

systemService 会直接通过 Binder 机制注册到 *ServiceManager*，而对于  appService 则不会直接进行注册，而是会注册 systemService 中进行处理，也就是说将 systemService 作为了代理。

```c
#define XPOSED_BINDER_SYSTEM_SERVICE_NAME "user.xposed.system"
#define XPOSED_BINDER_APP_SERVICE_NAME    "user.xposed.app"
```

```cpp
bool startAll() {
    if (xposed->isSELinuxEnabled && !membased::init()) {
        return false;
    }

    // system context service
    pid_t pid;
    if ((pid = fork()) < 0) {
        ALOGE("Fork for Xposed service in system context failed: %s", strerror(errno));
        return false;
    } else if (pid == 0) {
        systemService();
        // Should never reach this point
        exit(EXIT_FAILURE);
    }

    // app context service
    if ((pid = fork()) < 0) {
        ALOGE("Fork for Xposed service in app context failed: %s", strerror(errno));
        return false;
    } else if (pid == 0) {
        appService();
        // Should never reach this point
        exit(EXIT_FAILURE);
    }

    if (xposed->isSELinuxEnabled && !checkMembasedRunning()) {
        return false;
    }

    return true;
}
```

## xposed::onVmCreated(env)

这个方法，实际上就是根据我们运行环境的不同（DVM，ART） 来加载不同的动态库：

```c
#define XPOSED_LIB_DALVIK        XPOSED_LIB_DIR "libxposed_dalvik.so"
#define XPOSED_LIB_ART           XPOSED_LIB_DIR "libxposed_art.so"
```
```c
/** Load the libxposed_*.so library for the currently active runtime. */
void onVmCreated(JNIEnv* env) {
    // Determine the currently active runtime
    const char* xposedLibPath = NULL;
    if (!determineRuntime(&xposedLibPath)) {
        ALOGE("Could not determine runtime, not loading Xposed");
        return;
    }

    // Load the suitable libxposed_*.so for it
    void* xposedLibHandle = dlopen(xposedLibPath, RTLD_NOW);
    if (!xposedLibHandle) {
        ALOGE("Could not load libxposed: %s", dlerror());
        return;
    }

    // Clear previous errors
    dlerror();

    // Initialize the library
    bool (*xposedInitLib)(XposedShared* shared) = NULL;
    *(void **) (&xposedInitLib) = dlsym(xposedLibHandle, "xposedInitLib");
    if (!xposedInitLib)  {
        ALOGE("Could not find function xposedInitLib");
        return;
    }

#if XPOSED_WITH_SELINUX
    xposed->zygoteservice_accessFile = &service::membased::accessFile;
    xposed->zygoteservice_statFile   = &service::membased::statFile;
    xposed->zygoteservice_readFile   = &service::membased::readFile;
#endif  // XPOSED_WITH_SELINUX

    if (xposedInitLib(xposed)) {
        xposed->onVmCreated(env);
    }
}
```

至于哪个库初始化了哪些东西我们也可以看一下。DVM 已经淘汰了，所以我们看一下 ART 就行了。

```c
// libxposed_art.cpp
/** Called by Xposed's app_process replacement. */
bool xposedInitLib(XposedShared* shared) {
    xposed = shared;
    xposed->onVmCreated = &onVmCreatedCommon;
    return true;
}

/** Called very early during VM startup. */
bool onVmCreated(JNIEnv*) {
    // TODO: Handle CLASS_MIUI_RESOURCES?
    ArtMethod::xposed_callback_class = classXposedBridge;
    ArtMethod::xposed_callback_method = methodXposedBridgeHandleHookedMethod;
    return true;
}
```

## libxposed_common

这个就是做实际的初始化工作了，然后再调用 libxposed_art 中的 `onVmCreated()` 函数。

```c
void onVmCreatedCommon(JNIEnv* env) {
    if (!initXposedBridge(env) || !initZygoteService(env)) {
        return;
    }

    if (!onVmCreated(env)) {
        return;
    }

    xposedLoadedSuccessfully = true;
    return;
}
```

在这里干的事情，就是向 VM 层注入内容了。

```c
#define CLASS_XPOSED_BRIDGE  "de/robv/android/xposed/XposedBridge"
#define CLASS_XRESOURCES     "android/content/res/XResources"
#define CLASS_MIUI_RESOURCES "android/content/res/MiuiResources"
#define CLASS_ZYGOTE_SERVICE "de/robv/android/xposed/services/ZygoteService"
#define CLASS_FILE_RESULT    "de/robv/android/xposed/services/FileResult"
```

- initXposedBridge 这个就是向我们添加的 XposedBridge 的类，注入了 Native 方法。
- initZygoteService 注入几个 CLASS_ZYGOTE_SERVICE 的 Native 方法。

## 总结

到此 xposed 就已经启动完了，我们来看看我们都做了些什么事情：

1. 开启了两个服务 
  #define XPOSED_BINDER_SYSTEM_SERVICE_NAME "user.xposed.system"
  #define XPOSED_BINDER_APP_SERVICE_NAME    "user.xposed.app"
2. 将 Jar 包添加到环境变量
  #define XPOSED_JAR               "/system/framework/XposedBridge.jar"
3. 对两个 Java 类注入了 Native 方法
  #define CLASS_ZYGOTE_SERVICE "de/robv/android/xposed/services/ZygoteService"
  #define CLASS_XPOSED_BRIDGE  "de/robv/android/xposed/XposedBridge"
4. 入口替换

```c
#define XPOSED_CLASS_DOTS_ZYGOTE "de.robv.android.xposed.XposedBridge"
#define XPOSED_CLASS_DOTS_TOOLS  "de.robv.android.xposed.XposedBridge$ToolEntryPoint"
```
```c
    if (zygote) {
        isXposedLoaded = xposed::initialize(true, startSystemServer, NULL, argc, argv);
        runtimeStart(runtime, isXposedLoaded ? XPOSED_CLASS_DOTS_ZYGOTE : "com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        isXposedLoaded = xposed::initialize(false, false, className, argc, argv);
        runtimeStart(runtime, isXposedLoaded ? XPOSED_CLASS_DOTS_TOOLS : "com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
        return 10;
    }
```

# Java 层

在 C 层，我们通过开启服务，加载 Jar 包，注入 Native 方法，已经提供了最基本的能力，但是很多框架层的东西，还在 Java 层内实现的。

我们知道，入口已经被替换成这两个：

```c
#define XPOSED_CLASS_DOTS_ZYGOTE "de.robv.android.xposed.XposedBridge"
#define XPOSED_CLASS_DOTS_TOOLS  "de.robv.android.xposed.XposedBridge$ToolEntryPoint"
```
后一个实际上走到的逻辑是一样的。

## XposedBridge$ToolEntryPoint

```java
	protected static final class ToolEntryPoint {
		protected static void main(String[] args) {
			isZygote = false;
			XposedBridge.main(args);
		}
	}
```

## XposedBridge.main()

```java
	protected static void main(String[] args) {
		// Initialize the Xposed framework and modules
		try {
			if (!hadInitErrors()) {
				initXResources();

				SELinuxHelper.initOnce();
				SELinuxHelper.initForProcess(null);

				runtime = getRuntime();
				XPOSED_BRIDGE_VERSION = getXposedVersion();

				if (isZygote) {
					XposedInit.hookResources();
					XposedInit.initForZygote();
				}

				XposedInit.loadModules();
			} else {
				Log.e(TAG, "Not initializing Xposed because of previous errors");
			}
		} catch (Throwable t) {
			Log.e(TAG, "Errors during Xposed initialization", t);
			disableHooks = true;
		}

		// Call the original startup code
		if (isZygote) {
			ZygoteInit.main(args);
		} else {
			RuntimeInit.main(args);
		}
    }
```

## XposedInit.initForZygote();

如果是 Zygote 进程，那么就会初始化很多的 HOOK 方法。

## XposedBridge.loadModules()

这个就是会尝试加载所有的模块了。

```java
	private static final String INSTALLER_PACKAGE_NAME = "de.robv.android.xposed.installer";
	@SuppressLint("SdCardPath")
	private static final String BASE_DIR = Build.VERSION.SDK_INT >= 24
			? "/data/user_de/0/" + INSTALLER_PACKAGE_NAME + "/"
            : "/data/data/" + INSTALLER_PACKAGE_NAME + "/";
```

先长处读取相应目录下 `conf/modules.list` 里面定义的模块，其实里面记录的是 APK 的 路径。

```java
	/*package*/ static void loadModules() throws IOException {
		final String filename = BASE_DIR + "conf/modules.list";
		BaseService service = SELinuxHelper.getAppDataFileService();
		if (!service.checkFileExists(filename)) {
			Log.e(TAG, "Cannot load any modules because " + filename + " was not found");
			return;
		}

		ClassLoader topClassLoader = XposedBridge.BOOTCLASSLOADER;
		ClassLoader parent;
		while ((parent = topClassLoader.getParent()) != null) {
			topClassLoader = parent;
		}

		InputStream stream = service.getFileInputStream(filename);
		BufferedReader apks = new BufferedReader(new InputStreamReader(stream));
		String apk;
		while ((apk = apks.readLine()) != null) {
			loadModule(apk, topClassLoader);
		}
		apks.close();
    }
```

## XposedBridge.loadModules(apk,clsloader)

这个才是具体的加载某个 APK 内的内容。

1. 先看一下 APK 是否存在。
2. 读取 apk 中 *assets/xposed_init* 文件内容。
3. 每行定义了一个类，尝试加载类，然后查看类的的实现类型，来决定不同的操作。

**如果当前进程是 zygote**，会处理三种 HOOK。
- IXposedHookZygoteInit
- IXposedHookLoadPackage
- IXposedHookInitPackageResources


下面的这个已经被放弃了。

**如果当前是普通进程**，也会处理：

- IXposedHookCmdInit 

```java
	private static void loadModule(String apk, ClassLoader topClassLoader) {
		Log.i(TAG, "Loading modules from " + apk);

		if (!new File(apk).exists()) {
			Log.e(TAG, "  File does not exist");
			return;
		}

		DexFile dexFile;
		try {
			dexFile = new DexFile(apk);
		} catch (IOException e) {
			Log.e(TAG, "  Cannot load module", e);
			return;
		}

		if (dexFile.loadClass(INSTANT_RUN_CLASS, topClassLoader) != null) {
			Log.e(TAG, "  Cannot load module, please disable \"Instant Run\" in Android Studio.");
			closeSilently(dexFile);
			return;
		}

		if (dexFile.loadClass(XposedBridge.class.getName(), topClassLoader) != null) {
			Log.e(TAG, "  Cannot load module:");
			Log.e(TAG, "  The Xposed API classes are compiled into the module's APK.");
			Log.e(TAG, "  This may cause strange issues and must be fixed by the module developer.");
			Log.e(TAG, "  For details, see: http://api.xposed.info/using.html");
			closeSilently(dexFile);
			return;
		}

		closeSilently(dexFile);

		ZipFile zipFile = null;
		InputStream is;
		try {
			zipFile = new ZipFile(apk);
			ZipEntry zipEntry = zipFile.getEntry("assets/xposed_init");
			if (zipEntry == null) {
				Log.e(TAG, "  assets/xposed_init not found in the APK");
				closeSilently(zipFile);
				return;
			}
			is = zipFile.getInputStream(zipEntry);
		} catch (IOException e) {
			Log.e(TAG, "  Cannot read assets/xposed_init in the APK", e);
			closeSilently(zipFile);
			return;
		}

		ClassLoader mcl = new PathClassLoader(apk, XposedBridge.BOOTCLASSLOADER);
		BufferedReader moduleClassesReader = new BufferedReader(new InputStreamReader(is));
		try {
			String moduleClassName;
			while ((moduleClassName = moduleClassesReader.readLine()) != null) {
				moduleClassName = moduleClassName.trim();
				if (moduleClassName.isEmpty() || moduleClassName.startsWith("#"))
					continue;

				try {
					Log.i(TAG, "  Loading class " + moduleClassName);
					Class<?> moduleClass = mcl.loadClass(moduleClassName);

					if (!IXposedMod.class.isAssignableFrom(moduleClass)) {
						Log.e(TAG, "    This class doesn't implement any sub-interface of IXposedMod, skipping it");
						continue;
					} else if (disableResources && IXposedHookInitPackageResources.class.isAssignableFrom(moduleClass)) {
						Log.e(TAG, "    This class requires resource-related hooks (which are disabled), skipping it.");
						continue;
					}

					final Object moduleInstance = moduleClass.newInstance();
					if (XposedBridge.isZygote) {
						if (moduleInstance instanceof IXposedHookZygoteInit) {
							IXposedHookZygoteInit.StartupParam param = new IXposedHookZygoteInit.StartupParam();
							param.modulePath = apk;
							param.startsSystemServer = startsSystemServer;
							((IXposedHookZygoteInit) moduleInstance).initZygote(param);
						}

						if (moduleInstance instanceof IXposedHookLoadPackage)
							XposedBridge.hookLoadPackage(new IXposedHookLoadPackage.Wrapper((IXposedHookLoadPackage) moduleInstance));

						if (moduleInstance instanceof IXposedHookInitPackageResources)
							XposedBridge.hookInitPackageResources(new IXposedHookInitPackageResources.Wrapper((IXposedHookInitPackageResources) moduleInstance));
					} else {
						if (moduleInstance instanceof IXposedHookCmdInit) {
							IXposedHookCmdInit.StartupParam param = new IXposedHookCmdInit.StartupParam();
							param.modulePath = apk;
							param.startClassName = startClassName;
							((IXposedHookCmdInit) moduleInstance).initCmdApp(param);
						}
					}
				} catch (Throwable t) {
					Log.e(TAG, "    Failed to load class " + moduleClassName, t);
				}
			}
		} catch (IOException e) {
			Log.e(TAG, "  Failed to load module from " + apk, e);
		} finally {
			closeSilently(is);
			closeSilently(zipFile);
		}
	}
}
```

## Hook类型
**如果当前进程是 zygote**，会处理三种 HOOK。

- IXposedHookZygoteInit -> Zygote 初始化的时候进行 Hook
- IXposedHookLoadPackage -> XposedBridge.hookLoadPackage()
- IXposedHookInitPackageResources -> XposedBridge.hookInitPackageResources()

# 总结

1. 最终，所有 APP 拥有的运行时开始的时候都是一样的。
2. 所有的 APP 都是从 XposedBridge 进行启动的。这些都是静态类方法来启动。
3. 对于我们安装的所有模块，XposedBridge 都会尝试加载，也就是说所有进程都会尝试加载进来，然后执行对应的 HOOK。因此，一般还会在模块的 HOOK 类实现里面指定要 HOOK 的对应 APP，否则的话就太坑了。
4. 最后，我们的 XposedIntaller 会对我们的安装模块进行记录，更新 `/data/data/de.robv.android.xposed.installer/conf/modules.list` 文件。
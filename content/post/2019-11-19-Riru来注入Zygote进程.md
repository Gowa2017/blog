---
title: Riru来注入Zygote进程
categories:
  - Android
date: 2019-11-19 23:52:05
updated: 2019-11-19 23:52:05
tags: 
  - Android
  - Riru
  - Hooking
  - 逆向
---

Riru 使用替换系统共享库 libmemtrack.so 来实现注入，因为 ptrace 一般来说都会被进程进行检测，所以说注入系统进程是比较轻松的一点。

<!-- more-->

# 项目地址
[https://github.com/RikkaApps/Riru](https://github.com/RikkaApps/Riru) 非

这个项目要使用的前提是手机进行了 root，因为只有这样才能替换系统文件。

查看打包出来的 magisk core 文件，里面除了替换 libmemtrack.so 外，还会在系统目录下放入 `zygote_restart_[arch]` 的命令行工具。用途不明，后面再看。这样当我们的 magisk 获取了 root 的时候，就可以安装其他的 so 库了。

# 原理

因为在 app_process 启动的时候会加载 libmemtrack.so，之后调用 fork 后，加载的模块会在子进程可用。然后，我们改造过后的 libmemtrack.so 含有了一些黑科技函数，所以就能干我们自己想干的事情了。



>简而言之，替换一个会被 zygote 进程加载的共享库。

> 首先要找到那个共享库，而且那个共享库要越简单越好，所以就盯上了只有 10 个导出函数的 libmemtrack。 然后就可以自己提供一个叫 libmemtrack 并且也提供了原来的函数们的库，这样就可以进去 zygote 进程也不会发生爆炸。（然而现在看来选 libmemtrack 也不是很好）

> 接着如何知道自己已经在应用进程或者系统服务进程里面。 JNI 函数 (`com.android.internal.os.Zygote#nativeForkAndSpecialize` & `com.android.internal.os.Zygote#nativeForkSystemServer`) 会在应用进程或者系统服务进程被 fork 出来的时候被调用。 所以只要把这两个函数换成自己的。这部分很简单，只要 hook `jniRegisterNativeMethods` 因为所有 libandroid_runtime 里面的 JNI 方法都是通过这个注册，然后就可以再调用 `RegisterNatives` 来替换它们。

# libmemtrack.so

在源码中 `riru-core/jini/main/main.cpp` 定义了 我们要替换的代码，当被加载的时候，其 **constructor** 会被执行：

其就是利用了 xhook 框架，然后将 jniRegisterNativeMethods 方法给 hook 掉了。之后调用这个函数注册 native 方法到 JNI 内的时候，都会逐一检查要注册进 JNI 的方法，看是不是我们要替换的，如果是，那就把它给 hook掉。

```cpp
extern "C" void constructor() __attribute__((constructor));

void constructor() {
    static int loaded = 0;
    if (loaded)
        return;

    loaded = 1;

    if (getuid() != 0)
        return;

    char cmdline[ARG_MAX + 1];
    get_self_cmdline(cmdline);
     
    // 查看是否是启动的 zygote 进程
    if (!strstr(cmdline, "--zygote"))
        return;

#ifdef __LP64__
    LOGI("Riru %s in zygote64", VERSION_NAME);
#else
    LOGI("Riru %s in zygote", VERSION_NAME);
#endif

    LOGI("config dir is %s", get_config_dir());

    char path[PATH_MAX];
    // 查看是否禁止
    snprintf(path, PATH_MAX, "%s/.disable", get_config_dir());

    if (access(path, F_OK) == 0) {
        LOGI("%s exists, do nothing.", path);
        return;
    }

    read_prop();

   // 关键在这里，是将 libandroid_runtime.so 中的 jniRegisterNativeMethods 进行 hook 了
   // 后续调用进行将 native 方法注入到 JNI 的时候，都会被劫持。
   // 这一步只是将要 hook 的函数进行了注册到 xhook 框架内
    XHOOK_REGISTER(".*\\libandroid_runtime.so$", jniRegisterNativeMethods);

    // 执行代码的 hook
    if (xhook_refresh(0) == 0) {
        xhook_clear();
        LOGI("hook installed");
    } else {
        LOGE("failed to refresh hook");
    }

    load_modules();
}
```

# xhook

## xhook_register

利用宏来减少写参数。首先，NEW_FUNC_DEF 会将 jniRegisterNativeMethods 定义是一个同样签名的新函数 new_jniRegisterNativeMethods，而用一个 old_jniRegisterNativeMethods 来指向以前 hook 前的函数。



```cpp
#define XHOOK_REGISTER(PATH_REGEX, NAME) \
    if (xhook_register(PATH_REGEX, #NAME, (void*) new_##NAME, (void **) &old_##NAME) != 0) \
        LOGE("failed to register hook " #NAME "."); \

#define NEW_FUNC_DEF(ret, func, ...) \
    static ret (*old_##func)(__VA_ARGS__); \
    static ret new_##func(__VA_ARGS__)

NEW_FUNC_DEF(int, jniRegisterNativeMethods, JNIEnv *env, const char *className,
             const JNINativeMethod *methods, int numMethods) {
    put_native_method(className, methods, numMethods);

    LOGV("jniRegisterNativeMethods %s", className);

    JNINativeMethod *newMethods = nullptr;
    if (strcmp("com/android/internal/os/Zygote", className) == 0) {
        newMethods = onRegisterZygote(env, className, methods, numMethods);
    } else if (strcmp("android/os/SystemProperties", className) == 0) {
        // hook android.os.SystemProperties#native_set to prevent a critical problem on Android 9+
        // see comment of SystemProperties_set in jni_native_method.cpp for detail
        newMethods = onRegisterSystemProperties(env, className, methods, numMethods);
    }

    int res = old_jniRegisterNativeMethods(env, className, newMethods ? newMethods : methods,
                                           numMethods);
    delete newMethods;
    return res;
}

```

如果将 XHOOK_REGISTER 进行展开，那么结果如下：

```cpp
    if (xhook_register(PATH_REGEX, jniRegisterNativeMethods, (void*) new_jniRegisterNativeMethods, (void **) &old_jniRegisterNativeMethods) != 0) \
        LOGE("failed to register hook " #NAME "."); 
```



## xhook_refresh

调用这个函数，就会将相应函数的地址进行替换了。

# new_jniRegisterNativeMethods

我们这个方法展开来看起来是这样的：

```cpp
static int new_jniRegisterNativeMethods(JNIEnv *env, const char *className,
             const JNINativeMethod *methods, int numMethods) {
    put_native_method(className, methods, numMethods);

    LOGV("jniRegisterNativeMethods %s", className);

    JNINativeMethod *newMethods = nullptr;
    if (strcmp("com/android/internal/os/Zygote", className) == 0) {
        newMethods = onRegisterZygote(env, className, methods, numMethods);
    } else if (strcmp("android/os/SystemProperties", className) == 0) {
        // hook android.os.SystemProperties#native_set to prevent a critical problem on Android 9+
        // see comment of SystemProperties_set in jni_native_method.cpp for detail
        newMethods = onRegisterSystemProperties(env, className, methods, numMethods);
    }

    int res = old_jniRegisterNativeMethods(env, className, newMethods ? newMethods : methods,
                                           numMethods);
    delete newMethods;
    return res;
}
```

可以看到，这个 hook 只会针对 *com/android/internal/os/Zygote* 类和  *android/os/SystemProperties* 类中的方法进行过滤，查看一下是不是我们需要进行 hook 的。

## put_native_method

v这个方法会将原始要注册的方法进行一个引用，放到 map 内。这是一个以类名作为键，方法数组的 JNINativeMethodHolder 值的 map 。

```cpp
static auto *native_methods = new std::map<std::string, JNINativeMethodHolder*>();

void put_native_method(const char *className, const JNINativeMethod *methods, int numMethods) {
    (*native_methods)[className] = new JNINativeMethodHolder(methods, numMethods);

```



## onRegisterZygote

好了，当我们向 *com/android/internal/os/Zygote* 注册方法的时候，就要过滤一下是不是我们的想要劫持的方法了。

1. 构造一个新的方法数组，然后将旧有的方法数组复制过去。
2. 记录原始方法 `set_nativeForkAndSpecialize(method.fnPtr);`
3. 替换新的方法数组中的对应方法指针。
4. 调用未劫持的 new_jniRegisterNativeMethods 进行注册。

根据不同的版本，注册的方法也是不一样的。

```cpp
static JNINativeMethod *onRegisterZygote(JNIEnv *env, const char *className,
                                         const JNINativeMethod *methods, int numMethods) {
    int replaced = 0;

    auto *newMethods = new JNINativeMethod[numMethods];
    memcpy(newMethods, methods, sizeof(JNINativeMethod) * numMethods);

    JNINativeMethod method;
    for (int i = 0; i < numMethods; ++i) {
        method = methods[i];

        if (strcmp(method.name, "nativeForkAndSpecialize") == 0) {
            set_nativeForkAndSpecialize(method.fnPtr);

            if (strcmp(nativeForkAndSpecialize_marshmallow_sig, method.signature) == 0)
                newMethods[i].fnPtr = (void *) nativeForkAndSpecialize_marshmallow;
            else if (strcmp(nativeForkAndSpecialize_oreo_sig, method.signature) == 0)
                newMethods[i].fnPtr = (void *) nativeForkAndSpecialize_oreo;
            else if (strcmp(nativeForkAndSpecialize_p_sig, method.signature) == 0)
                newMethods[i].fnPtr = (void *) nativeForkAndSpecialize_p;
            else if (strcmp(nativeForkAndSpecialize_q_beta4_sig, method.signature) == 0)
                newMethods[i].fnPtr = (void *) nativeForkAndSpecialize_q_beta4;
            else if (strcmp(nativeForkAndSpecialize_samsung_p_sig, method.signature) == 0)
                newMethods[i].fnPtr = (void *) nativeForkAndSpecialize_samsung_p;
            else if (strcmp(nativeForkAndSpecialize_samsung_o_sig, method.signature) == 0)
                newMethods[i].fnPtr = (void *) nativeForkAndSpecialize_samsung_o;
            else if (strcmp(nativeForkAndSpecialize_samsung_n_sig, method.signature) == 0)
                newMethods[i].fnPtr = (void *) nativeForkAndSpecialize_samsung_n;
            else if (strcmp(nativeForkAndSpecialize_samsung_m_sig, method.signature) == 0)
                newMethods[i].fnPtr = (void *) nativeForkAndSpecialize_samsung_m;
            else
                LOGW("found nativeForkAndSpecialize but signature %s mismatch", method.signature);

            if (newMethods[i].fnPtr != methods[i].fnPtr) {
                LOGI("replaced com.android.internal.os.Zygote#nativeForkAndSpecialize");
                riru_set_native_method_func(MODULE_NAME_CORE, className, newMethods[i].name,
                                            newMethods[i].signature, newMethods[i].fnPtr);

                replaced += 1;
            }
        } else if (strcmp(method.name, "nativeSpecializeAppProcess") == 0) {
            set_nativeSpecializeAppProcess(method.fnPtr);

            if (strcmp(nativeSpecializeAppProcess_sig_q_beta4, method.signature) == 0)
                newMethods[i].fnPtr = (void *) nativeSpecializeAppProcess_q_beta4;
            else if (strcmp(nativeSpecializeAppProcess_sig_q, method.signature) == 0)
                newMethods[i].fnPtr = (void *) nativeSpecializeAppProcess_q;
            else
                LOGW("found nativeSpecializeAppProcess but signature %s mismatch",
                     method.signature);

            if (newMethods[i].fnPtr != methods[i].fnPtr) {
                LOGI("replaced com.android.internal.os.Zygote#nativeSpecializeAppProcess");
                riru_set_native_method_func(MODULE_NAME_CORE, className, newMethods[i].name,
                                            newMethods[i].signature, newMethods[i].fnPtr);

                //replaced += 1;
            }
        } else if (strcmp(method.name, "nativeForkSystemServer") == 0) {
            set_nativeForkSystemServer(method.fnPtr);

            if (strcmp(nativeForkSystemServer_sig, method.signature) == 0)
                newMethods[i].fnPtr = (void *) nativeForkSystemServer;
            else
                LOGW("found nativeForkSystemServer but signature %s mismatch", method.signature);

            if (newMethods[i].fnPtr != methods[i].fnPtr) {
                LOGI("replaced com.android.internal.os.Zygote#nativeForkSystemServer");
                riru_set_native_method_func(MODULE_NAME_CORE, className, newMethods[i].name,
                                            newMethods[i].signature, newMethods[i].fnPtr);

                replaced += 1;
            }
        }
    }

    methods_replaced = replaced == 2/*(isQ() ? 3 : 2)*/;

    return newMethods;
}
 
```

## nativeForkAndSpecialize_oreo

当我们的是 Android_O 的时候，就会将 ForkAndSpecialize 替换成这个。

- pre 调用我们在 fork 前要做的事情。
- 正常的  ForkAndSpecialize 方法。
- post 调用我们的 fork 后要做的事情。

```cpp
jint nativeForkAndSpecialize_oreo(
        JNIEnv *env, jclass clazz, jint uid, jint gid, jintArray gids, jint debug_flags,
        jobjectArray rlimits, jint mount_external, jstring se_info, jstring se_name,
        jintArray fdsToClose, jintArray fdsToIgnore, jstring instructionSet, jstring appDataDir) {

    jboolean is_child_zygote = JNI_FALSE;
    jstring packageName = nullptr;
    jobjectArray packagesForUID = nullptr;
    jstring sandboxId = nullptr;

    nativeForkAndSpecialize_pre(env, clazz, uid, gid, gids, debug_flags, rlimits, mount_external,
                                se_info, se_name, fdsToClose, fdsToIgnore, is_child_zygote,
                                instructionSet, appDataDir, packageName, packagesForUID,
                                sandboxId);

    jint res = ((nativeForkAndSpecialize_oreo_t) _nativeForkAndSpecialize)(
            env, clazz, uid, gid, gids, debug_flags, rlimits, mount_external, se_info, se_name,
            fdsToClose, fdsToIgnore, instructionSet, appDataDir);

    nativeForkAndSpecialize_post(env, clazz, uid, res);
    return res;
}

```



# load_modules

此会将 **/data/misc/riru/modules** 目录下的 so 都加载起来，并查看对应的 module 标准的方法，以方便在 hook 的 jniRegisterNativeMethods 中调用。

```cpp
static void load_modules() {
    DIR *dir;
    struct dirent *entry;
    char path[PATH_MAX], modules_path[PATH_MAX], module_prop[PATH_MAX], api[PATH_MAX];
    int moduleApiVersion;
    void *handle;

    snprintf(modules_path, PATH_MAX, "%s/modules", get_config_dir());

    if (!(dir = _opendir(modules_path)))
        return;

    while ((entry = _readdir(dir))) {
        if (entry->d_type == DT_DIR) {
            if (entry->d_name[0] == '.')
                continue;

            snprintf(path, PATH_MAX, MODULE_PATH_FMT, entry->d_name);

            if (access(path, F_OK) != 0) {
                PLOGE("access %s", path);
                continue;
            }

            snprintf(module_prop, PATH_MAX, "%s/%s/module.prop", modules_path, entry->d_name);
            if (access(module_prop, F_OK) != 0) {
                PLOGE("access %s", module_prop);
                continue;
            }

            moduleApiVersion = -1;
            if (get_prop(module_prop, "api", api) > 0) {
                moduleApiVersion = atoi(api);
            }

            if (isQ() && moduleApiVersion < 3) {
                LOGW("module %s does not support Android Q", entry->d_name);
                continue;
            }

            handle = dlopen(path, RTLD_NOW | RTLD_GLOBAL);
            if (!handle) {
                PLOGE("dlopen %s", path);
                continue;
            }

            auto *module = new struct module(strdup(entry->d_name));
            module->handle = handle;
            module->onModuleLoaded = dlsym(handle, "onModuleLoaded");
            module->forkAndSpecializePre = dlsym(handle, "nativeForkAndSpecializePre");
            module->forkAndSpecializePost = dlsym(handle, "nativeForkAndSpecializePost");
            module->forkSystemServerPre = dlsym(handle, "nativeForkSystemServerPre");
            module->forkSystemServerPost = dlsym(handle, "nativeForkSystemServerPost");
            module->specializeAppProcessPre = dlsym(handle, "specializeAppProcessPre");
            module->specializeAppProcessPost = dlsym(handle, "specializeAppProcessPost");
            module->shouldSkipUid = dlsym(handle, "shouldSkipUid");
            get_modules()->push_back(module);

            if (moduleApiVersion == -1) {
                // only for api v2
                module->getApiVersion = dlsym(handle, "getApiVersion");

                if (module->getApiVersion) {
                    module->apiVersion = ((getApiVersion_t *) module->getApiVersion)();
                }
            } else {
                module->apiVersion = moduleApiVersion;
            }

            void *sym = dlsym(handle, "riru_set_module_name");
            if (sym)
                ((void (*)(const char *)) sym)(module->name);

            LOGI("module loaded: %s (api %d)", module->name, module->apiVersion);

            if (module->onModuleLoaded) {
                LOGV("%s: onModuleLoaded", module->name);

                ((loaded_t *) module->onModuleLoaded)();
            }
        }
    }

    closedir(dir);
}

```


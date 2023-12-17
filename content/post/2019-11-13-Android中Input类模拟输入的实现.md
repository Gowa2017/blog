---
title: Android中Input类模拟输入的实现
categories:
  - Android
date: 2019-11-13 22:29:54
updated: 2019-11-13 22:29:54
tags: 
  - Android
  - Android Input
  - 逆向
---
我们在用 adb 进行调试的时候，可以使用类似 `adb shell input ...` 的形式来进行模拟输入，输入的不止是文字，还可以是模拟按键等内容，我们来看一下其是如何实现的。我们已经看到了 {% post_link InputService-安卓的输入系统 InputService-安卓的输入系统 %} 对 InputManagerService 是如何启动了。

<!--more-->

input 命令，本身是一个 shell 脚本，位于 /system/bin/input ，打开可以看到其内容非常的简单，通过 app_process 命令启动了一个类而已。**com.android.commands.input.Input** 在 SDK 内有，但是并不是公开的类，所以需要自己想办法查看后使用。

```sh
base=/system
export CLASSPATH=$base/framework/input.jar
exec app_process $base/bin com.android.commands.input.Input "$@"
```

那么，这个过程到底是怎么样的呢？我们在 {% post_link InputService-安卓的输入系统 InputService-安卓的输入系统 %} 一文中曾经介绍过系统的启动流程，我们的 Zygote 也是用这个命令来进行启动的。其启动的参数如下：

```
/system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
```

而我们的 input 是不带任何参数，只传递了一个类名，这个过程就要简单一些了。

# app_process.main

通过对参数的解析，app_process 来进行不同的启动模式。

```cpp
    while (i < argc) {
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = ZYGOTE_NICE_NAME;
        } else if (strcmp(arg, "--start-system-server") == 0) {
            startSystemServer = true;
        } else if (strcmp(arg, "--application") == 0) {
            application = true;
        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
            niceName.setTo(arg + 12);
        // 解析出要启动的类名
        } else if (strncmp(arg, "--", 2) != 0) {
            className.setTo(arg);
            break;
        } else {
            --i;
            break;
        }
    }
    Vector<String8> args;
    if (!className.isEmpty()) {
        // We're not in zygote mode, the only argument we need to pass
        // to RuntimeInit is the application argument.
        //
        // The Remainder of args get passed to startup class main(). Make
        // copies of them before we overwrite them with the process name.
        args.add(application ? String8("application") : String8("tool"));
        runtime.setClassNameAndArgs(className, argc - i, argv + i);

        if (!LOG_NDEBUG) {
          String8 restOfArgs;
          char* const* argv_new = argv + i;
          int argc_new = argc - i;
          for (int k = 0; k < argc_new; ++k) {
            restOfArgs.append("\"");
            restOfArgs.append(argv_new[k]);
            restOfArgs.append("\" ");
          }
          ALOGV("Class name = %s, args = %s", className.string(), restOfArgs.string());
        }
    } else {
        // systemserver 的参数逻辑
    }
    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    // 最终我们会执行到这里。
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
    }

```

我们知道，app_process 会构造一个 AppRuntime(AndroidRuntime)，然后在这个运行环境中启动相关的类，调用启动类的 main() 方法。在我们启动 input 命令的时候，实际上调用  `AppRuntime.setClassNameAndArgs()` 已经设置了要启动的内容。

# RuntimeInit.main()

安卓运行时的启动。

```java
    public static final void main(String[] argv) {
        enableDdms();
        if (argv.length == 2 && argv[1].equals("application")) {
            if (DEBUG) Slog.d(TAG, "RuntimeInit: Starting application");
            redirectLogStreams();
        } else {
            if (DEBUG) Slog.d(TAG, "RuntimeInit: Starting tool");
        }

        commonInit();

        /*
         * Now that we're running in interpreted code, call back into native code
         * to run the system.
         */
        nativeFinishInit();

        if (DEBUG) Slog.d(TAG, "Leaving RuntimeInit!");
    }

```
当 Java 层面调用完 `commonInit()` 后，那么就会在 AndroidRuntime 内进行初始化了。

# nativeFinishInit()

```cpp
static void com_android_internal_os_RuntimeInit_nativeFinishInit(JNIEnv* env, jobject clazz)
{
    gCurRuntime->onStarted();
}
static void com_android_internal_os_ZygoteInit_nativeZygoteInit(JNIEnv* env, jobject clazz)
{
    gCurRuntime->onZygoteInit();
}
```

而这两个虚函数，是在我们的 AppRuntime 内实现的。

```cpp
    virtual void onStarted()
    {
        sp<ProcessState> proc = ProcessState::self();
        ALOGV("App process: starting thread pool.\n");
        proc->startThreadPool();

        AndroidRuntime* ar = AndroidRuntime::getRuntime();
        ar->callMain(mClassName, mClass, mArgs);

        IPCThreadState::self()->stopProcess();
        hardware::IPCThreadState::self()->stopProcess();
    }

    virtual void onZygoteInit()
    {
        sp<ProcessState> proc = ProcessState::self();
        ALOGV("App process: starting thread pool.\n");
        proc->startThreadPool();
    }

```

在这里就直接通过JNI 反射，调用我们的要启动的类的 main 方法：

```cpp
status_t AndroidRuntime::callMain(const String8& className, jclass clazz,
    const Vector<String8>& args)
{
    JNIEnv* env;
    jmethodID methodId;

    ALOGD("Calling main entry %s", className.string());

    env = getJNIEnv();
    if (clazz == NULL || env == NULL) {
        return UNKNOWN_ERROR;
    }

    methodId = env->GetStaticMethodID(clazz, "main", "([Ljava/lang/String;)V");
    if (methodId == NULL) {
        ALOGE("ERROR: could not find method %s.main(String[])\n", className.string());
        return UNKNOWN_ERROR;
    }

    /*
     * We want to call main() with a String array with our arguments in it.
     * Create an array and populate it.
     */
    jclass stringClass;
    jobjectArray strArray;

    const size_t numArgs = args.size();
    stringClass = env->FindClass("java/lang/String");
    strArray = env->NewObjectArray(numArgs, stringClass, NULL);

    for (size_t i = 0; i < numArgs; i++) {
        jstring argStr = env->NewStringUTF(args[i].string());
        env->SetObjectArrayElement(strArray, i, argStr);
    }

    env->CallStaticVoidMethod(clazz, methodId, strArray);
    return NO_ERROR;
}
```

# com.android.commands.input.Input

在这里就会构造出对应的命令了，包括 source，command，等了，根据命令的不同，来执行不同的方法。

回顾之前我们的过程，其实就是开启一个进程，然后打开 AndroidRuntime，然后在这运行时里面执行我们指定类的 main 方法。

```java
    /**
     * Command-line entry point.
     *
     * @param args The command-line arguments
     */
    public static void main(String[] args) {
        (new Input()).run(args);
    }

    private void run(String[] args) {
        if (args.length < 1) {
            showUsage();
            return;
        }

        int index = 0;
        String command = args[index];
        int inputSource = InputDevice.SOURCE_UNKNOWN;
        if (SOURCES.containsKey(command)) {
            inputSource = SOURCES.get(command);
            index++;
            command = args[index];
        }
        final int length = args.length - index;

        try {
            if (command.equals("text")) {
                if (length == 2) {
                    inputSource = getSource(inputSource, InputDevice.SOURCE_KEYBOARD);
                    sendText(inputSource, args[index+1]);
                    return;
                }
            } else if (command.equals("keyevent")) {
                if (length >= 2) {
                    final boolean longpress = "--longpress".equals(args[index + 1]);
                    final int start = longpress ? index + 2 : index + 1;
                    inputSource = getSource(inputSource, InputDevice.SOURCE_KEYBOARD);
                    if (args.length > start) {
                        for (int i = start; i < args.length; i++) {
                            int keyCode = KeyEvent.keyCodeFromString(args[i]);
                            if (keyCode == KeyEvent.KEYCODE_UNKNOWN) {
                                keyCode = KeyEvent.keyCodeFromString("KEYCODE_" + args[i]);
                            }
                            sendKeyEvent(inputSource, keyCode, longpress);
                        }
                        return;
                    }
                }
            } else if (command.equals("tap")) {
                if (length == 3) {
                    inputSource = getSource(inputSource, InputDevice.SOURCE_TOUCHSCREEN);
                    sendTap(inputSource, Float.parseFloat(args[index+1]),
                            Float.parseFloat(args[index+2]));
                    return;
                }
            } else if (command.equals("swipe")) {
                int duration = -1;
                inputSource = getSource(inputSource, InputDevice.SOURCE_TOUCHSCREEN);
                switch (length) {
                    case 6:
                        duration = Integer.parseInt(args[index+5]);
                    case 5:
                        sendSwipe(inputSource,
                                Float.parseFloat(args[index+1]), Float.parseFloat(args[index+2]),
                                Float.parseFloat(args[index+3]), Float.parseFloat(args[index+4]),
                                duration);
                        return;
                }
            } else if (command.equals("draganddrop")) {
                int duration = -1;
                inputSource = getSource(inputSource, InputDevice.SOURCE_TOUCHSCREEN);
                switch (length) {
                    case 6:
                        duration = Integer.parseInt(args[index+5]);
                    case 5:
                        sendDragAndDrop(inputSource,
                                Float.parseFloat(args[index+1]), Float.parseFloat(args[index+2]),
                                Float.parseFloat(args[index+3]), Float.parseFloat(args[index+4]),
                                duration);
                        return;
                }
            } else if (command.equals("press")) {
                inputSource = getSource(inputSource, InputDevice.SOURCE_TRACKBALL);
                if (length == 1) {
                    sendTap(inputSource, 0.0f, 0.0f);
                    return;
                }
            } else if (command.equals("roll")) {
                inputSource = getSource(inputSource, InputDevice.SOURCE_TRACKBALL);
                if (length == 3) {
                    sendMove(inputSource, Float.parseFloat(args[index+1]),
                            Float.parseFloat(args[index+2]));
                    return;
                }
            } else {
                System.err.println("Error: Unknown command: " + command);
                showUsage();
                return;
            }
        } catch (NumberFormatException ex) {
        }
        System.err.println(INVALID_ARGUMENTS + command);
        showUsage();
    }

```

## sendText

这个方法实际上是将我们要发送的字符串转换成虚拟按键对应的按键码发送过去。

```java
    /**
     * Convert the characters of string text into key event's and send to
     * device.
     *
     * @param text is a string of characters you want to input to the device.
     */
    private void sendText(int source, String text) {

        StringBuffer buff = new StringBuffer(text);

        boolean escapeFlag = false;
        for (int i=0; i<buff.length(); i++) {
            if (escapeFlag) {
                escapeFlag = false;
                if (buff.charAt(i) == 's') {
                    buff.setCharAt(i, ' ');
                    buff.deleteCharAt(--i);
                }
            }
            if (buff.charAt(i) == '%') {
                escapeFlag = true;
            }
        }

        char[] chars = buff.toString().toCharArray();

        KeyCharacterMap kcm = KeyCharacterMap.load(KeyCharacterMap.VIRTUAL_KEYBOARD);
        KeyEvent[] events = kcm.getEvents(chars);
        for(int i = 0; i < events.length; i++) {
            KeyEvent e = events[i];
            if (source != e.getSource()) {
                e.setSource(source);
            }
            injectKeyEvent(e);
        }
    }


```

## InputManager 与 InputManagerService

InputManager   用来获取 输入设备的信息以及可用的键盘布局。

而 InputManagerService 是对 C 层的 InputManagerService 的一个包装，提供了服务的回调。其实现 了 InputManager  需要的 IInputManager 接口：

```java
public class InputManagerService extends IInputManager.Stub
        implements Watchdog.Monitor {
    static final String TAG = "InputManager";
    // ...
  }
```

## injectKeyEvent

实际上，下面的代码中，会获取到 InputManagerService 

```java
    private void injectKeyEvent(KeyEvent event) {
        Log.i(TAG, "injectKeyEvent: " + event);
        InputManager.getInstance().injectInputEvent(event,
                InputManager.INJECT_INPUT_EVENT_MODE_WAIT_FOR_FINISH);
    }
    
    /**
     * Injects an input event into the event system on behalf of an application.
     * The synchronization mode determines whether the method blocks while waiting for
     * input injection to proceed.
     * <p>
     * Requires {@link android.Manifest.permission.INJECT_EVENTS} to inject into
     * windows that are owned by other applications.
     * </p><p>
     * Make sure you correctly set the event time and input source of the event
     * before calling this method.
     * </p>
     *
     * @param event The event to inject.
     * @param mode The synchronization mode.  One of:
     * {@link #INJECT_INPUT_EVENT_MODE_ASYNC},
     * {@link #INJECT_INPUT_EVENT_MODE_WAIT_FOR_RESULT}, or
     * {@link #INJECT_INPUT_EVENT_MODE_WAIT_FOR_FINISH}.
     * @return True if input event injection succeeded.
     *
     * @hide
     */
    public boolean injectInputEvent(InputEvent event, int mode) {
        if (event == null) {
            throw new IllegalArgumentException("event must not be null");
        }
        if (mode != INJECT_INPUT_EVENT_MODE_ASYNC
                && mode != INJECT_INPUT_EVENT_MODE_WAIT_FOR_FINISH
                && mode != INJECT_INPUT_EVENT_MODE_WAIT_FOR_RESULT) {
            throw new IllegalArgumentException("mode is invalid");
        }

        try {
            return mIm.injectInputEvent(event, mode);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }

```

这里用到了 InputManager 这个类来进行初步的注入流程，最终还是通过 InputManagerService 来实现的。

### InputManager

这段代码看起来有点难懂，关键在于理解 IInputManager 到底是什么。通过 ServiceManager 获得 INPUT_SERVICE 的 Binder，然后构造 InputManager。

```java
ServiceManager.getServiceOrThrow(Context.INPUT_SERVICE)
```

会返回一个 IBinder，然后将此 IBinder 作为 IInputManager  的实现，进而交给 InputManager 进行引用。

```java
    public static InputManager getInstance() {
        synchronized (InputManager.class) {
            if (sInstance == null) {
                try {
                    sInstance = new InputManager(IInputManager.Stub
                            .asInterface(ServiceManager.getServiceOrThrow(Context.INPUT_SERVICE)));
                } catch (ServiceNotFoundException e) {
                    throw new IllegalStateException(e);
                }
            }
            return sInstance;
        }
    }
```

### IInputManager

这是一个 AIDL 定义的接口 其是，具体文档参考 [https://developer.android.com/guide/components/aidl](https://developer.android.com/guide/components/aidl)。

> Android SDK 工具会生成以 `.aidl` 文件命名的 `.java` 接口文件。生成的接口包含一个名为 `Stub` 的子类（例如，`YourInterface.Stub`），该子类是其父接口的抽象实现，并且会声明 `.aidl` 文件中的所有方法
>
> **注意：**`Stub` 还会定义几个辅助方法，其中最值得注意的是 `asInterface()`，该方法会接收 `IBinder`（通常是传递给客户端 `onServiceConnected()` 回调方法的参数），并返回 Stub 接口的实例。如需了解如何进行此转换的更多详情，请参阅[调用 IPC 方法](https://developer.android.com/guide/components/aidl#Calling)部分。

我们可以理解，InputManager 获得的底层是引用 InputManagerService 的 Binder 进行处理的。

```java
// http://androidxref.com/9.0.0_r3/xref/frameworks/base/core/java/android/os/ServiceManager.java
    /**
     * Returns a reference to a service with the given name.
     *
     * @param name the name of the service to get
     * @return a reference to the service, or <code>null</code> if the service doesn't exist
     */
    public static IBinder getService(String name) {
        try {
            IBinder service = sCache.get(name);
            if (service != null) {
                return service;
            } else {
                return Binder.allowBlocking(rawGetService(name));
            }
        } catch (RemoteException e) {
            Log.e(TAG, "error in getService", e);
        }
        return null;
    }
```

### InputManagerService.injectInputEvent()

```java
// http://androidxref.com/9.0.0_r3/xref/frameworks/base/services/core/java/com/android/server/input/InputManagerService.java
    @Override // Binder call
    public boolean injectInputEvent(InputEvent event, int mode) {
        return injectInputEventInternal(event, Display.DEFAULT_DISPLAY, mode);
    }

    private boolean injectInputEventInternal(InputEvent event, int displayId, int mode) {
        if (event == null) {
            throw new IllegalArgumentException("event must not be null");
        }
        if (mode != InputManager.INJECT_INPUT_EVENT_MODE_ASYNC
                && mode != InputManager.INJECT_INPUT_EVENT_MODE_WAIT_FOR_FINISH
                && mode != InputManager.INJECT_INPUT_EVENT_MODE_WAIT_FOR_RESULT) {
            throw new IllegalArgumentException("mode is invalid");
        }

        final int pid = Binder.getCallingPid();
        final int uid = Binder.getCallingUid();
        final long ident = Binder.clearCallingIdentity();
        final int result;
        try {
            result = nativeInjectInputEvent(mPtr, event, displayId, pid, uid, mode,
                    INJECTION_TIMEOUT_MILLIS, WindowManagerPolicy.FLAG_DISABLE_KEY_REPEAT);
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
        switch (result) {
            case INPUT_EVENT_INJECTION_PERMISSION_DENIED:
                Slog.w(TAG, "Input event injection from pid " + pid + " permission denied.");
                throw new SecurityException(
                        "Injecting to another application requires INJECT_EVENTS permission");
            case INPUT_EVENT_INJECTION_SUCCEEDED:
                return true;
            case INPUT_EVENT_INJECTION_TIMED_OUT:
                Slog.w(TAG, "Input event injection from pid " + pid + " timed out.");
                return false;
            case INPUT_EVENT_INJECTION_FAILED:
            default:
                Slog.w(TAG, "Input event injection from pid " + pid + " failed.");
                return false;
        }
    }

```

不出意外，最终还是走到了 native 层去

```cpp
// http://androidxref.com/9.0.0_r3/xref/frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp

static jint nativeInjectInputEvent(JNIEnv* env, jclass /* clazz */,
        jlong ptr, jobject inputEventObj, jint displayId, jint injectorPid, jint injectorUid,
        jint syncMode, jint timeoutMillis, jint policyFlags) {
    NativeInputManager* im = reinterpret_cast<NativeInputManager*>(ptr);

    if (env->IsInstanceOf(inputEventObj, gKeyEventClassInfo.clazz)) {
        KeyEvent keyEvent;
        status_t status = android_view_KeyEvent_toNative(env, inputEventObj, & keyEvent);
        if (status) {
            jniThrowRuntimeException(env, "Could not read contents of KeyEvent object.");
            return INPUT_EVENT_INJECTION_FAILED;
        }

        return (jint) im->getInputManager()->getDispatcher()->injectInputEvent(
                & keyEvent, displayId, injectorPid, injectorUid, syncMode, timeoutMillis,
                uint32_t(policyFlags));
    } else if (env->IsInstanceOf(inputEventObj, gMotionEventClassInfo.clazz)) {
        const MotionEvent* motionEvent = android_view_MotionEvent_getNativePtr(env, inputEventObj);
        if (!motionEvent) {
            jniThrowRuntimeException(env, "Could not read contents of MotionEvent object.");
            return INPUT_EVENT_INJECTION_FAILED;
        }

        return (jint) im->getInputManager()->getDispatcher()->injectInputEvent(
                motionEvent, displayId, injectorPid, injectorUid, syncMode, timeoutMillis,
                uint32_t(policyFlags));
    } else {
        jniThrowRuntimeException(env, "Invalid input event type.");
        return INPUT_EVENT_INJECTION_FAILED;
    }
}

```

关键代码是通过  NativeInputManager 来获取到 InputDispacher，然后通过 InputDispathcer 进行事件的注入。

### InputDispatcher::injectInputEvent

在这里还会检测是否拥有注入权限，然后将事件封装为 EventEntry 然后放到 InputDispatcher 的队列当中去，等待 InputDispatcherThread 来进行分发事件过去。

```cpp
// http://androidxref.com/9.0.0_r3/xref/frameworks/native/services/inputflinger/InputDispatcher.cpp

int32_t InputDispatcher::injectInputEvent(const InputEvent* event, int32_t displayId,
        int32_t injectorPid, int32_t injectorUid, int32_t syncMode, int32_t timeoutMillis,
        uint32_t policyFlags) {
#if DEBUG_INBOUND_EVENT_DETAILS
    ALOGD("injectInputEvent - eventType=%d, injectorPid=%d, injectorUid=%d, "
            "syncMode=%d, timeoutMillis=%d, policyFlags=0x%08x, displayId=%d",
            event->getType(), injectorPid, injectorUid, syncMode, timeoutMillis, policyFlags,
            displayId);
#endif

    nsecs_t endTime = now() + milliseconds_to_nanoseconds(timeoutMillis);

    policyFlags |= POLICY_FLAG_INJECTED;
    if (hasInjectionPermission(injectorPid, injectorUid)) {
        policyFlags |= POLICY_FLAG_TRUSTED;
    }

    EventEntry* firstInjectedEntry;
    EventEntry* lastInjectedEntry;
    switch (event->getType()) {
    case AINPUT_EVENT_TYPE_KEY: {
        const KeyEvent* keyEvent = static_cast<const KeyEvent*>(event);
        int32_t action = keyEvent->getAction();
        if (! validateKeyEvent(action)) {
            return INPUT_EVENT_INJECTION_FAILED;
        }

        int32_t flags = keyEvent->getFlags();
        if (flags & AKEY_EVENT_FLAG_VIRTUAL_HARD_KEY) {
            policyFlags |= POLICY_FLAG_VIRTUAL;
        }

        if (!(policyFlags & POLICY_FLAG_FILTERED)) {
            android::base::Timer t;
            mPolicy->interceptKeyBeforeQueueing(keyEvent, /*byref*/ policyFlags);
            if (t.duration() > SLOW_INTERCEPTION_THRESHOLD) {
                ALOGW("Excessive delay in interceptKeyBeforeQueueing; took %s ms",
                        std::to_string(t.duration().count()).c_str());
            }
        }

        mLock.lock();
        firstInjectedEntry = new KeyEntry(keyEvent->getEventTime(),
                keyEvent->getDeviceId(), keyEvent->getSource(),
                policyFlags, action, flags,
                keyEvent->getKeyCode(), keyEvent->getScanCode(), keyEvent->getMetaState(),
                keyEvent->getRepeatCount(), keyEvent->getDownTime());
        lastInjectedEntry = firstInjectedEntry;
        break;
    }

    case AINPUT_EVENT_TYPE_MOTION: {
        const MotionEvent* motionEvent = static_cast<const MotionEvent*>(event);
        int32_t action = motionEvent->getAction();
        size_t pointerCount = motionEvent->getPointerCount();
        const PointerProperties* pointerProperties = motionEvent->getPointerProperties();
        int32_t actionButton = motionEvent->getActionButton();
        if (! validateMotionEvent(action, actionButton, pointerCount, pointerProperties)) {
            return INPUT_EVENT_INJECTION_FAILED;
        }

        if (!(policyFlags & POLICY_FLAG_FILTERED)) {
            nsecs_t eventTime = motionEvent->getEventTime();
            android::base::Timer t;
            mPolicy->interceptMotionBeforeQueueing(eventTime, /*byref*/ policyFlags);
            if (t.duration() > SLOW_INTERCEPTION_THRESHOLD) {
                ALOGW("Excessive delay in interceptMotionBeforeQueueing; took %s ms",
                        std::to_string(t.duration().count()).c_str());
            }
        }

        mLock.lock();
        const nsecs_t* sampleEventTimes = motionEvent->getSampleEventTimes();
        const PointerCoords* samplePointerCoords = motionEvent->getSamplePointerCoords();
        firstInjectedEntry = new MotionEntry(*sampleEventTimes,
                motionEvent->getDeviceId(), motionEvent->getSource(), policyFlags,
                action, actionButton, motionEvent->getFlags(),
                motionEvent->getMetaState(), motionEvent->getButtonState(),
                motionEvent->getEdgeFlags(),
                motionEvent->getXPrecision(), motionEvent->getYPrecision(),
                motionEvent->getDownTime(), displayId,
                uint32_t(pointerCount), pointerProperties, samplePointerCoords,
                motionEvent->getXOffset(), motionEvent->getYOffset());
        lastInjectedEntry = firstInjectedEntry;
        for (size_t i = motionEvent->getHistorySize(); i > 0; i--) {
            sampleEventTimes += 1;
            samplePointerCoords += pointerCount;
            MotionEntry* nextInjectedEntry = new MotionEntry(*sampleEventTimes,
                    motionEvent->getDeviceId(), motionEvent->getSource(), policyFlags,
                    action, actionButton, motionEvent->getFlags(),
                    motionEvent->getMetaState(), motionEvent->getButtonState(),
                    motionEvent->getEdgeFlags(),
                    motionEvent->getXPrecision(), motionEvent->getYPrecision(),
                    motionEvent->getDownTime(), displayId,
                    uint32_t(pointerCount), pointerProperties, samplePointerCoords,
                    motionEvent->getXOffset(), motionEvent->getYOffset());
            lastInjectedEntry->next = nextInjectedEntry;
            lastInjectedEntry = nextInjectedEntry;
        }
        break;
    }

    default:
        ALOGW("Cannot inject event of type %d", event->getType());
        return INPUT_EVENT_INJECTION_FAILED;
    }

    InjectionState* injectionState = new InjectionState(injectorPid, injectorUid);
    if (syncMode == INPUT_EVENT_INJECTION_SYNC_NONE) {
        injectionState->injectionIsAsync = true;
    }

    injectionState->refCount += 1;
    lastInjectedEntry->injectionState = injectionState;

    bool needWake = false;
    for (EventEntry* entry = firstInjectedEntry; entry != NULL; ) {
        EventEntry* nextEntry = entry->next;
        needWake |= enqueueInboundEventLocked(entry);
        entry = nextEntry;
    }

    mLock.unlock();

    if (needWake) {
        mLooper->wake();
    }

    int32_t injectionResult;
    { // acquire lock
        AutoMutex _l(mLock);

        if (syncMode == INPUT_EVENT_INJECTION_SYNC_NONE) {
            injectionResult = INPUT_EVENT_INJECTION_SUCCEEDED;
        } else {
            for (;;) {
                injectionResult = injectionState->injectionResult;
                if (injectionResult != INPUT_EVENT_INJECTION_PENDING) {
                    break;
                }

                nsecs_t remainingTimeout = endTime - now();
                if (remainingTimeout <= 0) {
#if DEBUG_INJECTION
                    ALOGD("injectInputEvent - Timed out waiting for injection result "
                            "to become available.");
#endif
                    injectionResult = INPUT_EVENT_INJECTION_TIMED_OUT;
                    break;
                }

                mInjectionResultAvailableCondition.waitRelative(mLock, remainingTimeout);
            }

            if (injectionResult == INPUT_EVENT_INJECTION_SUCCEEDED
                    && syncMode == INPUT_EVENT_INJECTION_SYNC_WAIT_FOR_FINISHED) {
                while (injectionState->pendingForegroundDispatches != 0) {
#if DEBUG_INJECTION
                    ALOGD("injectInputEvent - Waiting for %d pending foreground dispatches.",
                            injectionState->pendingForegroundDispatches);
#endif
                    nsecs_t remainingTimeout = endTime - now();
                    if (remainingTimeout <= 0) {
#if DEBUG_INJECTION
                    ALOGD("injectInputEvent - Timed out waiting for pending foreground "
                            "dispatches to finish.");
#endif
                        injectionResult = INPUT_EVENT_INJECTION_TIMED_OUT;
                        break;
                    }

                    mInjectionSyncFinishedCondition.waitRelative(mLock, remainingTimeout);
                }
            }
        }

        injectionState->release();
    } // release lock

#if DEBUG_INJECTION
    ALOGD("injectInputEvent - Finished with result %d.  "
            "injectorPid=%d, injectorUid=%d",
            injectionResult, injectorPid, injectorUid);
#endif

    return injectionResult;
}

bool InputDispatcher::hasInjectionPermission(int32_t injectorPid, int32_t injectorUid) {
    return injectorUid == 0
            || mPolicy->checkInjectEventsPermissionNonReentrant(injectorPid, injectorUid);
}
```



# 问题

 InputManagerService 是从 SystemServer 启动的，我们在一个新的进程中是如何获取到其 Binder 的呢？

事实上这和在进程进行启动的时候，打开了 binder 机制有关。

# AppRuntime

```cpp
 virtual void onStarted()
    {
        sp<ProcessState> proc = ProcessState::self();
        ALOGV("App process: starting thread pool.\n");
        proc->startThreadPool();

        AndroidRuntime* ar = AndroidRuntime::getRuntime();
        ar->callMain(mClassName, mClass, mArgs);

        IPCThreadState::self()->stopProcess();
        hardware::IPCThreadState::self()->stopProcess();
    }

    virtual void onZygoteInit()
    {
        sp<ProcessState> proc = ProcessState::self();
        ALOGV("App process: starting thread pool.\n");
        proc->startThreadPool();
    }
```

在 AndroidRuntime 建立后都会回调这两个方法之一，然后建立 ProcessState 对象：

# ProcessState

这个会与 /dev/binder 驱动间建立联系，将自己的文件描述符映射过去。

## ProcessState::self()

```cpp
sp<ProcessState> ProcessState::self()
{
    Mutex::Autolock _l(gProcessMutex);
    if (gProcess != NULL) {
        return gProcess;
    }
    gProcess = new ProcessState("/dev/binder");
    return gProcess;
}

ProcessState::ProcessState(const char *driver)
    : mDriverName(String8(driver))
    , mDriverFD(open_driver(driver))
    , mVMStart(MAP_FAILED)
    , mThreadCountLock(PTHREAD_MUTEX_INITIALIZER)
    , mThreadCountDecrement(PTHREAD_COND_INITIALIZER)
    , mExecutingThreadsCount(0)
    , mMaxThreads(DEFAULT_MAX_BINDER_THREADS)
    , mStarvationStartTimeMs(0)
    , mManagesContexts(false)
    , mBinderContextCheckFunc(NULL)
    , mBinderContextUserData(NULL)
    , mThreadPoolStarted(false)
    , mThreadPoolSeq(1)
{
    if (mDriverFD >= 0) {
        // mmap the binder, providing a chunk of virtual address space to receive transactions.
        mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
        if (mVMStart == MAP_FAILED) {
            // *sigh*
            ALOGE("Using %s failed: unable to mmap transaction memory.\n", mDriverName.c_str());
            close(mDriverFD);
            mDriverFD = -1;
            mDriverName.clear();
        }
    }

    LOG_ALWAYS_FATAL_IF(mDriverFD < 0, "Binder driver could not be opened.  Terminating.");
}

static int open_driver(const char *driver)
{
    int fd = open(driver, O_RDWR | O_CLOEXEC);
    if (fd >= 0) {
        int vers = 0;
        status_t result = ioctl(fd, BINDER_VERSION, &vers);
        if (result == -1) {
            ALOGE("Binder ioctl to obtain version failed: %s", strerror(errno));
            close(fd);
            fd = -1;
        }
        if (result != 0 || vers != BINDER_CURRENT_PROTOCOL_VERSION) {
          ALOGE("Binder driver protocol(%d) does not match user space protocol(%d)! ioctl() return value: %d",
                vers, BINDER_CURRENT_PROTOCOL_VERSION, result);
            close(fd);
            fd = -1;
        }
        size_t maxThreads = DEFAULT_MAX_BINDER_THREADS;
        result = ioctl(fd, BINDER_SET_MAX_THREADS, &maxThreads);
        if (result == -1) {
            ALOGE("Binder ioctl to set max threads failed: %s", strerror(errno));
        }
    } else {
        ALOGW("Opening '%s' failed: %s\n", driver, strerror(errno));
    }
    return fd;
}

```



# ServiceManager

我们所有通过 Binder 机制来通信的都会通过这个服务来进行，这个服务，是由 init 进程启动的。

```
# init.rc
    # Start essential services.
    start servicemanager
    start hwservicemanager
    start vndservicemanager
```

## service_manager.c

这里面重要的是打开了 binder 驱动后，调用的 `binder_become_context_manager` 成为了整个 Binder 的上下文管理者，整个驱动只有一个。

```cpp
// http://androidxref.com/9.0.0_r3/xref/frameworks/native/cmds/servicemanager/service_manager.c

int main(int argc, char** argv)
{
    struct binder_state *bs;
    union selinux_callback cb;
    char *driver;

    if (argc > 1) {
        driver = argv[1];
    } else {
        driver = "/dev/binder";
    }

    bs = binder_open(driver, 128*1024);
    if (!bs) {
#ifdef VENDORSERVICEMANAGER
        ALOGW("failed to open binder driver %s\n", driver);
        while (true) {
            sleep(UINT_MAX);
        }
#else
        ALOGE("failed to open binder driver %s\n", driver);
#endif
        return -1;
    }

    if (binder_become_context_manager(bs)) {
        ALOGE("cannot become context manager (%s)\n", strerror(errno));
        return -1;
    }

    cb.func_audit = audit_callback;
    selinux_set_callback(SELINUX_CB_AUDIT, cb);
    cb.func_log = selinux_log_callback;
    selinux_set_callback(SELINUX_CB_LOG, cb);

#ifdef VENDORSERVICEMANAGER
    sehandle = selinux_android_vendor_service_context_handle();
#else
    sehandle = selinux_android_service_context_handle();
#endif
    selinux_status_open(true);

    if (sehandle == NULL) {
        ALOGE("SELinux: Failed to acquire sehandle. Aborting.\n");
        abort();
    }

    if (getcon(&service_manager_context) != 0) {
        ALOGE("SELinux: Failed to acquire service_manager context. Aborting.\n");
        abort();
    }


    binder_loop(bs, svcmgr_handler);

    return 0;
}

```

## ServiceManager.java

这是一个单例服务，当我们添加或者注册服务时，需要找到C 层的 ServiceManager

```java
// http://androidxref.com/9.0.0_r3/xref/frameworks/base/core/java/android/os/ServiceManager.java
    private static IServiceManager getIServiceManager() {
        if (sServiceManager != null) {
            return sServiceManager;
        }

        // Find the service manager
        sServiceManager = ServiceManagerNative
                .asInterface(Binder.allowBlocking(BinderInternal.getContextObject()));
        return sServiceManager;
    }

```

### ServiceManagerProxy

上面的代码中 ServiceManagerNative 实际上会在拿到了 ServiceManager 的 Binder 后建立一个代理类。

```java
    static public IServiceManager asInterface(IBinder obj)
    {
        if (obj == null) {
            return null;
        }
        IServiceManager in =
            (IServiceManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }
        
        return new ServiceManagerProxy(obj);
    }

```



```cpp
// http://androidxref.com/9.0.0_r3/xref/frameworks/base/core/jni/android_util_Binder.cpp#977
static jobject android_os_BinderInternal_getContextObject(JNIEnv* env, jobject clazz)
{
    sp<IBinder> b = ProcessState::self()->getContextObject(NULL);
    return javaObjectForIBinder(env, b);
}
```



```cpp
// http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/binder/ProcessState.cpp#110
sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& /*caller*/)
{
    return getStrongProxyForHandle(0);
}

sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{
    sp<IBinder> result;

    AutoMutex _l(mLock);

    handle_entry* e = lookupHandleLocked(handle);

    if (e != NULL) {
        // We need to create a new BpBinder if there isn't currently one, OR we
        // are unable to acquire a weak reference on this current one.  See comment
        // in getWeakProxyForHandle() for more info about this.
        IBinder* b = e->binder;
        if (b == NULL || !e->refs->attemptIncWeak(this)) {
            if (handle == 0) {
                // Special case for context manager...
                // The context manager is the only object for which we create
                // a BpBinder proxy without already holding a reference.
                // Perform a dummy transaction to ensure the context manager
                // is registered before we create the first local reference
                // to it (which will occur when creating the BpBinder).
                // If a local reference is created for the BpBinder when the
                // context manager is not present, the driver will fail to
                // provide a reference to the context manager, but the
                // driver API does not return status.
                //
                // Note that this is not race-free if the context manager
                // dies while this code runs.
                //
                // TODO: add a driver API to wait for context manager, or
                // stop special casing handle 0 for context manager and add
                // a driver API to get a handle to the context manager with
                // proper reference counting.

                Parcel data;
                status_t status = IPCThreadState::self()->transact(
                        0, IBinder::PING_TRANSACTION, data, NULL, 0);
                if (status == DEAD_OBJECT)
                   return NULL;
            }

            b = BpBinder::create(handle);
            e->binder = b;
            if (b) e->refs = b->getWeakRefs();
            result = b;
        } else {
            // This little bit of nastyness is to allow us to add a primary
            // reference to the remote proxy when this team doesn't have one
            // but another team is sending the handle to us.
            result.force_set(b);
            e->refs->decWeak(this);
        }
    }

    return result;
}
```

在这里 handler = 0 表示要查找的是 ServiceManager 的句柄，那么就会查找到已经运行的ServiceManager 的，并建立一个代理 BpBinder。

### addService

```java
    /**
     * Place a new @a service called @a name into the service
     * manager.
     * 
     * @param name the name of the new service
     * @param service the service object
     */
    public static void addService(String name, IBinder service) {
        try {
            getIServiceManager().addService(name, service, false);
        } catch (RemoteException e) {
            Log.e(TAG, "error in addService", e);
        }
    }

```

本质上是调用的是 ServiceManagerProxy 的方法：

```java
    public void addService(String name, IBinder service, boolean allowIsolated)
            throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IServiceManager.descriptor);
        data.writeString(name);
        data.writeStrongBinder(service);
        data.writeInt(allowIsolated ? 1 : 0);
        mRemote.transact(ADD_SERVICE_TRANSACTION, data, reply, 0);
        reply.recycle();
        data.recycle();
    }

```

简单l的说，就是像 servicemanager 的文件描述符里面写入数据。

```cpp
int do_add_service(struct binder_state *bs, const uint16_t *s, size_t len, uint32_t handle,
                   uid_t uid, int allow_isolated, uint32_t dumpsys_priority, pid_t spid) {
    struct svcinfo *si;

    //ALOGI("add_service('%s',%x,%s) uid=%d\n", str8(s, len), handle,
    //        allow_isolated ? "allow_isolated" : "!allow_isolated", uid);

    if (!handle || (len == 0) || (len > 127))
        return -1;

    if (!svc_can_register(s, len, spid, uid)) {
        ALOGE("add_service('%s',%x) uid=%d - PERMISSION DENIED\n",
             str8(s, len), handle, uid);
        return -1;
    }

    si = find_svc(s, len);
    if (si) {
        if (si->handle) {
            ALOGE("add_service('%s',%x) uid=%d - ALREADY REGISTERED, OVERRIDE\n",
                 str8(s, len), handle, uid);
            svcinfo_death(bs, si);
        }
        si->handle = handle;
    } else {
        si = malloc(sizeof(*si) + (len + 1) * sizeof(uint16_t));
        if (!si) {
            ALOGE("add_service('%s',%x) uid=%d - OUT OF MEMORY\n",
                 str8(s, len), handle, uid);
            return -1;
        }
        si->handle = handle;
        si->len = len;
        memcpy(si->name, s, (len + 1) * sizeof(uint16_t));
        si->name[len] = '\0';
        si->death.func = (void*) svcinfo_death;
        si->death.ptr = si;
        si->allow_isolated = allow_isolated;
        si->dumpsys_priority = dumpsys_priority;
        si->next = svclist;
        svclist = si;
    }

    binder_acquire(bs, handle);
    binder_link_to_death(bs, handle, &si->death);
    return 0;
}

```


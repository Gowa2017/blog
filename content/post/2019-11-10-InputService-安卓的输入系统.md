---
title: InputService-安卓的输入系统
categories:

  - Android
date: 2019-11-10 23:59:44
updated: 2019-11-10 23:59:44
tags: 
  - Android
  - Android Input
  - 逆向
---
我们来探究一下，从我们手指按压屏幕，到界面上做出响应，背后的过程是怎么样的。我们能否做到模拟一下这个过程呢？不过这又有想远了。在 {% post_link Android系统的启动过程 Android系统的启动过程 %} 我们介绍了安卓系统的启动过程。我们就以  InputManagerService 为例来看 一下服务的启动。

<!--more-->
一直想来研究一下这个过程的，但是这个过程实在是复杂，在对 C 和 Linux
， Jni 不熟的情况下会是一脸懵逼的。但这个就需要从安卓系统的启动说起了。

# 关于安卓的输入

[官方有一个描述性的文档说明了输入过程](https://source.android.com/devices/input/touch-devices)：

下面简要汇总了 Android 上的触摸设备操作。

1. `EventHub` 从 `evdev` 驱动程序读取原始事件。
2. `InputReader` 消耗原始事件，并更新关于每个工具的位置和其他特征的内部状态。它还会跟踪按钮状态。
3. 如果按下或释放“后退”或“前进”按钮，`InputReader` 会向 `InputDispatcher` 发出按键事件通知。
4. `InputReader` 确定是否发生了虚拟按键的按压操作。如果是，它会向 `InputDispatcher` 发出按键事件通知。
5. `InputReader` 确定触摸行为是否在显示范围内发起的。如果是，它会向 `InputDispatcher` 发出触摸事件通知。
6. 如果没有触摸工具，但至少有一个悬停工具，则 `InputReader` 会向 `InputDispatcher` 发出悬停事件通知。
7. 如果触摸设备类型是指控设备，则 `InputReader` 会执行指针手势检测，相应地移动指针和相关点，并通知 `InputDispatcher` 指针事件。
8. `InputDispatcher` 使用 `WindowManagerPolicy` 来确定是否应该调度这些事件，以及它们是否应该唤醒设备。然后，`InputDispatcher` 将事件传递给相应的应用。

# 安卓的输入栈

![image-20191113173055018](../res/image-20191113173055018.png)

# InputManagerService

这个服务在 `startOtherService()` 中启动：

```java
        InputManagerService inputManager = null;
        WindowManagerService wm = null;

            traceBeginAndSlog("StartInputManagerService");
            inputManager = new InputManagerService(context);
            wm = WindowManagerService.main(context, inputManager,
                    mFactoryTestMode != FactoryTest.FACTORY_TEST_LOW_LEVEL,
                    !mFirstBoot, mOnlyCore, new PhoneWindowManager());
            ServiceManager.addService(Context.WINDOW_SERVICE, wm, /* allowIsolated= */ false,
                    DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PROTO);
            ServiceManager.addService(Context.INPUT_SERVICE, inputManager,
                    /* allowIsolated= */ false, DUMP_FLAG_PRIORITY_CRITICAL);



            traceBeginAndSlog("StartInputManager");
            inputManager.setWindowManagerCallbacks(wm.getInputMonitor());
            inputManager.start();
            traceEnd();

```

可以看到， InputManagerService 与 WindowManagerService 是相互持有，相互通信的。

InputManagerService 的回调设置成了 WindowManagerService 的监控器。

## InputManagerService()

InputManagerService 服务的初始化，实际上还会在 C 层面有一个对象来关联，而 InputManagerService 则通过 一个 C 指针 **mPtr**来进行引用**。

```java
    public InputManagerService(Context context) {
        this.mContext = context;
        this.mHandler = new InputManagerHandler(DisplayThread.get().getLooper());

        mUseDevInputEventForAudioJack =
                context.getResources().getBoolean(R.bool.config_useDevInputEventForAudioJack);
        Slog.i(TAG, "Initializing input manager, mUseDevInputEventForAudioJack="
                + mUseDevInputEventForAudioJack);
        mPtr = nativeInit(this, mContext, mHandler.getLooper().getQueue());

        String doubleTouchGestureEnablePath = context.getResources().getString(
                R.string.config_doubleTouchGestureEnableFile);
        mDoubleTouchGestureEnableFile = TextUtils.isEmpty(doubleTouchGestureEnablePath) ? null :
            new File(doubleTouchGestureEnablePath);

        LocalServices.addService(InputManagerInternal.class, new LocalService());
    }

```

## nativeInit()

```cpp
//http://androidxref.com/9.0.0_r3/xref/frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp

static jlong nativeInit(JNIEnv* env, jclass /* clazz */,
        jobject serviceObj, jobject contextObj, jobject messageQueueObj) {
    sp<MessageQueue> messageQueue = android_os_MessageQueue_getMessageQueue(env, messageQueueObj);
    if (messageQueue == NULL) {
        jniThrowRuntimeException(env, "MessageQueue is not initialized.");
        return 0;
    }

    NativeInputManager* im = new NativeInputManager(contextObj, serviceObj,
            messageQueue->getLooper());
    im->incStrong(0);
    return reinterpret_cast<jlong>(im);
}

```

### NativeInputManager

NativeInputManager 属于 C 层的输入管理器，用来管理相关输入逻辑。其构造了一个 InputManager 来管理 EventHub 上的事件。

```cpp
NativeInputManager::NativeInputManager(jobject contextObj,
        jobject serviceObj, const sp<Looper>& looper) :
        mLooper(looper), mInteractive(true) {
    JNIEnv* env = jniEnv();

    mContextObj = env->NewGlobalRef(contextObj);
    mServiceObj = env->NewGlobalRef(serviceObj);

    {
        AutoMutex _l(mLock);
        mLocked.systemUiVisibility = ASYSTEM_UI_VISIBILITY_STATUS_BAR_VISIBLE;
        mLocked.pointerSpeed = 0;
        mLocked.pointerGesturesEnabled = true;
        mLocked.showTouches = false;
        mLocked.pointerCapture = false;
    }
    mInteractive = true;

    sp<EventHub> eventHub = new EventHub();
    mInputManager = new InputManager(eventHub, this, this);
}
```

### EventHub

EventHub 会监控 `/dev/input` 目录下的删除和新建事件，同时还建立了一个管道，根据管道是否有数据来确定是否有唤醒事件。

```cpp
// http://androidxref.com/9.0.0_r3/xref/frameworks/native/services/inputflinger/EventHub.cpp#201

EventHub::EventHub(void) :
        mBuiltInKeyboardId(NO_BUILT_IN_KEYBOARD), mNextDeviceId(1), mControllerNumbers(),
        mOpeningDevices(0), mClosingDevices(0),
        mNeedToSendFinishedDeviceScan(false),
        mNeedToReopenDevices(false), mNeedToScanDevices(true),
        mPendingEventCount(0), mPendingEventIndex(0), mPendingINotify(false) {
    acquire_wake_lock(PARTIAL_WAKE_LOCK, WAKE_LOCK_ID);

    mEpollFd = epoll_create(EPOLL_SIZE_HINT);
    LOG_ALWAYS_FATAL_IF(mEpollFd < 0, "Could not create epoll instance.  errno=%d", errno);

    mINotifyFd = inotify_init();
    int result = inotify_add_watch(mINotifyFd, DEVICE_PATH, IN_DELETE | IN_CREATE);
    LOG_ALWAYS_FATAL_IF(result < 0, "Could not register INotify for %s.  errno=%d",
            DEVICE_PATH, errno);

    struct epoll_event eventItem;
    memset(&eventItem, 0, sizeof(eventItem));
    eventItem.events = EPOLLIN;
    eventItem.data.u32 = EPOLL_ID_INOTIFY;
    result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mINotifyFd, &eventItem);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not add INotify to epoll instance.  errno=%d", errno);

    int wakeFds[2];
    result = pipe(wakeFds);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not create wake pipe.  errno=%d", errno);

    mWakeReadPipeFd = wakeFds[0];
    mWakeWritePipeFd = wakeFds[1];

    result = fcntl(mWakeReadPipeFd, F_SETFL, O_NONBLOCK);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not make wake read pipe non-blocking.  errno=%d",
            errno);

    result = fcntl(mWakeWritePipeFd, F_SETFL, O_NONBLOCK);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not make wake write pipe non-blocking.  errno=%d",
            errno);

    eventItem.data.u32 = EPOLL_ID_WAKE;
    result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeReadPipeFd, &eventItem);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not add wake read pipe to epoll instance.  errno=%d",
            errno);

    int major, minor;
    getLinuxRelease(&major, &minor);
    // EPOLLWAKEUP was introduced in kernel 3.5
    mUsingEpollWakeup = major > 3 || (major == 3 && minor >= 5);
}
```

#### EventHub.getEvents()

EventHub 在构造的时候会监控 /dev/input 目录的新增和删除事件，还会监控一个管道 mWakeReadPipeFd 的数据到达事件。但是，这个时候实际上还没有将所有的设备监控起来，只有在我们第一地调用这个方法的时候，会进行设备的扫描。

在这里面如果没有其他事件，只是获取事件的话，那么会调用  epoll_wait 等待事件到达。如果有事件，会将事件放到 buff 里面，供 InputReader 使用。

```cpp
size_t EventHub::getEvents(int timeoutMillis, RawEvent* buffer, size_t bufferSize) {
    ALOG_ASSERT(bufferSize >= 1);

    AutoMutex _l(mLock);

    struct input_event readBuffer[bufferSize];

    RawEvent* event = buffer;
    size_t capacity = bufferSize;
    bool awoken = false;
    // 循环读取
    for (;;) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);

        // Reopen input devices if needed.
        if (mNeedToReopenDevices) {
            mNeedToReopenDevices = false;

            ALOGI("Reopening all input devices due to a configuration change.");

            closeAllDevicesLocked();
            mNeedToScanDevices = true;
            break; // return to the caller before we actually rescan
        }

        // Report any devices that had last been added/removed.
        while (mClosingDevices) {
            Device* device = mClosingDevices;
            ALOGV("Reporting device closed: id=%d, name=%s\n",
                 device->id, device->path.string());
            mClosingDevices = device->next;
            event->when = now;
            event->deviceId = device->id == mBuiltInKeyboardId ? BUILT_IN_KEYBOARD_ID : device->id;
            event->type = DEVICE_REMOVED;
            event += 1;
            delete device;
            mNeedToSendFinishedDeviceScan = true;
            if (--capacity == 0) {
                break;
            }
        }

      	// 第一次调用的时候，会执行这里，进行设备的扫描
        if (mNeedToScanDevices) {
            mNeedToScanDevices = false;
            scanDevicesLocked();
            mNeedToSendFinishedDeviceScan = true;
        }

        while (mOpeningDevices != NULL) {
            Device* device = mOpeningDevices;
            ALOGV("Reporting device opened: id=%d, name=%s\n",
                 device->id, device->path.string());
            mOpeningDevices = device->next;
            event->when = now;
            event->deviceId = device->id == mBuiltInKeyboardId ? 0 : device->id;
            event->type = DEVICE_ADDED;
            event += 1;
            mNeedToSendFinishedDeviceScan = true;
            if (--capacity == 0) {
                break;
            }
        }


        if (mNeedToSendFinishedDeviceScan) {
            mNeedToSendFinishedDeviceScan = false;
            event->when = now;
            event->type = FINISHED_DEVICE_SCAN;
            event += 1;
            if (--capacity == 0) {
                break;
            }
        }

        // Grab the next input event.
        bool deviceChanged = false;
        while (mPendingEventIndex < mPendingEventCount) {
            const struct epoll_event& eventItem = mPendingEventItems[mPendingEventIndex++];
            if (eventItem.data.u32 == EPOLL_ID_INOTIFY) {
                if (eventItem.events & EPOLLIN) {
                    mPendingINotify = true;
                } else {
                    ALOGW("Received unexpected epoll event 0x%08x for INotify.", eventItem.events);
                }
                continue;
            }

            if (eventItem.data.u32 == EPOLL_ID_WAKE) {
                if (eventItem.events & EPOLLIN) {
                    ALOGV("awoken after wake()");
                    awoken = true;
                    char buffer[16];
                    ssize_t nRead;
                    do {
                        nRead = read(mWakeReadPipeFd, buffer, sizeof(buffer));
                    } while ((nRead == -1 && errno == EINTR) || nRead == sizeof(buffer));
                } else {
                    ALOGW("Received unexpected epoll event 0x%08x for wake read pipe.",
                            eventItem.events);
                }
                continue;
            }

            ssize_t deviceIndex = mDevices.indexOfKey(eventItem.data.u32);
            if (deviceIndex < 0) {
                ALOGW("Received unexpected epoll event 0x%08x for unknown device id %d.",
                        eventItem.events, eventItem.data.u32);
                continue;
            }

            Device* device = mDevices.valueAt(deviceIndex);
            if (eventItem.events & EPOLLIN) {
                int32_t readSize = read(device->fd, readBuffer,
                        sizeof(struct input_event) * capacity);
                if (readSize == 0 || (readSize < 0 && errno == ENODEV)) {
                    // Device was removed before INotify noticed.
                    ALOGW("could not get event, removed? (fd: %d size: %" PRId32
                            " bufferSize: %zu capacity: %zu errno: %d)\n",
                            device->fd, readSize, bufferSize, capacity, errno);
                    deviceChanged = true;
                    closeDeviceLocked(device);
                } else if (readSize < 0) {
                    if (errno != EAGAIN && errno != EINTR) {
                        ALOGW("could not get event (errno=%d)", errno);
                    }
                } else if ((readSize % sizeof(struct input_event)) != 0) {
                    ALOGE("could not get event (wrong size: %d)", readSize);
                } else {
                    int32_t deviceId = device->id == mBuiltInKeyboardId ? 0 : device->id;

                    size_t count = size_t(readSize) / sizeof(struct input_event);
                    for (size_t i = 0; i < count; i++) {
                        struct input_event& iev = readBuffer[i];
                        ALOGV("%s got: time=%d.%06d, type=%d, code=%d, value=%d",
                                device->path.string(),
                                (int) iev.time.tv_sec, (int) iev.time.tv_usec,
                                iev.type, iev.code, iev.value);

                        // Some input devices may have a better concept of the time
                        // when an input event was actually generated than the kernel
                        // which simply timestamps all events on entry to evdev.
                        // This is a custom Android extension of the input protocol
                        // mainly intended for use with uinput based device drivers.
                        if (iev.type == EV_MSC) {
                            if (iev.code == MSC_ANDROID_TIME_SEC) {
                                device->timestampOverrideSec = iev.value;
                                continue;
                            } else if (iev.code == MSC_ANDROID_TIME_USEC) {
                                device->timestampOverrideUsec = iev.value;
                                continue;
                            }
                        }
                        if (device->timestampOverrideSec || device->timestampOverrideUsec) {
                            iev.time.tv_sec = device->timestampOverrideSec;
                            iev.time.tv_usec = device->timestampOverrideUsec;
                            if (iev.type == EV_SYN && iev.code == SYN_REPORT) {
                                device->timestampOverrideSec = 0;
                                device->timestampOverrideUsec = 0;
                            }
                            ALOGV("applied override time %d.%06d",
                                    int(iev.time.tv_sec), int(iev.time.tv_usec));
                        }

                        // Use the time specified in the event instead of the current time
                        // so that downstream code can get more accurate estimates of
                        // event dispatch latency from the time the event is enqueued onto
                        // the evdev client buffer.
                        //
                        // The event's timestamp fortuitously uses the same monotonic clock
                        // time base as the rest of Android.  The kernel event device driver
                        // (drivers/input/evdev.c) obtains timestamps using ktime_get_ts().
                        // The systemTime(SYSTEM_TIME_MONOTONIC) function we use everywhere
                        // calls clock_gettime(CLOCK_MONOTONIC) which is implemented as a
                        // system call that also queries ktime_get_ts().
                        event->when = nsecs_t(iev.time.tv_sec) * 1000000000LL
                                + nsecs_t(iev.time.tv_usec) * 1000LL;
                        ALOGV("event time %" PRId64 ", now %" PRId64, event->when, now);

                        // Bug 7291243: Add a guard in case the kernel generates timestamps
                        // that appear to be far into the future because they were generated
                        // using the wrong clock source.
                        //
                        // This can happen because when the input device is initially opened
                        // it has a default clock source of CLOCK_REALTIME.  Any input events
                        // enqueued right after the device is opened will have timestamps
                        // generated using CLOCK_REALTIME.  We later set the clock source
                        // to CLOCK_MONOTONIC but it is already too late.
                        //
                        // Invalid input event timestamps can result in ANRs, crashes and
                        // and other issues that are hard to track down.  We must not let them
                        // propagate through the system.
                        //
                        // Log a warning so that we notice the problem and recover gracefully.
                        if (event->when >= now + 10 * 1000000000LL) {
                            // Double-check.  Time may have moved on.
                            nsecs_t time = systemTime(SYSTEM_TIME_MONOTONIC);
                            if (event->when > time) {
                                ALOGW("An input event from %s has a timestamp that appears to "
                                        "have been generated using the wrong clock source "
                                        "(expected CLOCK_MONOTONIC): "
                                        "event time %" PRId64 ", current time %" PRId64
                                        ", call time %" PRId64 ".  "
                                        "Using current time instead.",
                                        device->path.string(), event->when, time, now);
                                event->when = time;
                            } else {
                                ALOGV("Event time is ok but failed the fast path and required "
                                        "an extra call to systemTime: "
                                        "event time %" PRId64 ", current time %" PRId64
                                        ", call time %" PRId64 ".",
                                        event->when, time, now);
                            }
                        }
                        event->deviceId = deviceId;
                        event->type = iev.type;
                        event->code = iev.code;
                        event->value = iev.value;
                        event += 1;
                        capacity -= 1;
                    }
                    if (capacity == 0) {
                        // The result buffer is full.  Reset the pending event index
                        // so we will try to read the device again on the next iteration.
                        mPendingEventIndex -= 1;
                        break;
                    }
                }
            } else if (eventItem.events & EPOLLHUP) {
                ALOGI("Removing device %s due to epoll hang-up event.",
                        device->identifier.name.string());
                deviceChanged = true;
                closeDeviceLocked(device);
            } else {
                ALOGW("Received unexpected epoll event 0x%08x for device %s.",
                        eventItem.events, device->identifier.name.string());
            }
        }

        // readNotify() will modify the list of devices so this must be done after
        // processing all other events to ensure that we read all remaining events
        // before closing the devices.
        if (mPendingINotify && mPendingEventIndex >= mPendingEventCount) {
            mPendingINotify = false;
            readNotifyLocked();
            deviceChanged = true;
        }

        // Report added or removed devices immediately.
        if (deviceChanged) {
            continue;
        }

        // Return now if we have collected any events or if we were explicitly awoken.
        if (event != buffer || awoken) {
            break;
        }

        // Poll for events.  Mind the wake lock dance!
        // We hold a wake lock at all times except during epoll_wait().  This works due to some
        // subtle choreography.  When a device driver has pending (unread) events, it acquires
        // a kernel wake lock.  However, once the last pending event has been read, the device
        // driver will release the kernel wake lock.  To prevent the system from going to sleep
        // when this happens, the EventHub holds onto its own user wake lock while the client
        // is processing events.  Thus the system can only sleep if there are no events
        // pending or currently being processed.
        //
        // The timeout is advisory only.  If the device is asleep, it will not wake just to
        // service the timeout.
        mPendingEventIndex = 0;

        mLock.unlock(); // release lock before poll, must be before release_wake_lock
        release_wake_lock(WAKE_LOCK_ID);

        int pollResult = epoll_wait(mEpollFd, mPendingEventItems, EPOLL_MAX_EVENTS, timeoutMillis);

        acquire_wake_lock(PARTIAL_WAKE_LOCK, WAKE_LOCK_ID);
        mLock.lock(); // reacquire lock after poll, must be after acquire_wake_lock

        if (pollResult == 0) {
            // Timed out.
            mPendingEventCount = 0;
            break;
        }

        if (pollResult < 0) {
            // An error occurred.
            mPendingEventCount = 0;

            // Sleep after errors to avoid locking up the system.
            // Hopefully the error is transient.
            if (errno != EINTR) {
                ALOGW("poll failed (errno=%d)\n", errno);
                usleep(100000);
            }
        } else {
            // Some events occurred.
            mPendingEventCount = size_t(pollResult);
        }
    }

    // All done, return the number of events we read.
    return event - buffer;
}
```

#### 设备扫描

```cpp
void EventHub::scanDevicesLocked() {
    status_t res = scanDirLocked(DEVICE_PATH);
    if(res < 0) {
        ALOGE("scan dir failed for %s\n", DEVICE_PATH);
    }
    if (mDevices.indexOfKey(VIRTUAL_KEYBOARD_ID) < 0) {
        createVirtualKeyboardLocked();
    }
}

status_t EventHub::scanDirLocked(const char *dirname)
{
    char devname[PATH_MAX];
    char *filename;
    DIR *dir;
    struct dirent *de;
    dir = opendir(dirname);
    if(dir == NULL)
        return -1;
    strcpy(devname, dirname);
    filename = devname + strlen(devname);
    *filename++ = '/';
    while((de = readdir(dir))) {
        if(de->d_name[0] == '.' &&
           (de->d_name[1] == '\0' ||
            (de->d_name[1] == '.' && de->d_name[2] == '\0')))
            continue;
        strcpy(filename, de->d_name);
        openDeviceLocked(devname);
    }
    closedir(dir);
    return 0;
}
```



#### 打开设备

```cpp
status_t EventHub::openDeviceLocked(const char *devicePath) {
    char buffer[80];

    ALOGV("Opening device: %s", devicePath);

    int fd = open(devicePath, O_RDWR | O_CLOEXEC | O_NONBLOCK);
    if(fd < 0) {
        ALOGE("could not open %s, %s\n", devicePath, strerror(errno));
        return -1;
    }

    InputDeviceIdentifier identifier;

    // Get device name.
    if(ioctl(fd, EVIOCGNAME(sizeof(buffer) - 1), &buffer) < 1) {
        //fprintf(stderr, "could not get device name for %s, %s\n", devicePath, strerror(errno));
    } else {
        buffer[sizeof(buffer) - 1] = '\0';
        identifier.name.setTo(buffer);
    }

    // Check to see if the device is on our excluded list
    for (size_t i = 0; i < mExcludedDevices.size(); i++) {
        const String8& item = mExcludedDevices.itemAt(i);
        if (identifier.name == item) {
            ALOGI("ignoring event id %s driver %s\n", devicePath, item.string());
            close(fd);
            return -1;
        }
    }

    // Get device driver version.
    int driverVersion;
    if(ioctl(fd, EVIOCGVERSION, &driverVersion)) {
        ALOGE("could not get driver version for %s, %s\n", devicePath, strerror(errno));
        close(fd);
        return -1;
    }

    // Get device identifier.
    struct input_id inputId;
    if(ioctl(fd, EVIOCGID, &inputId)) {
        ALOGE("could not get device input id for %s, %s\n", devicePath, strerror(errno));
        close(fd);
        return -1;
    }
    identifier.bus = inputId.bustype;
    identifier.product = inputId.product;
    identifier.vendor = inputId.vendor;
    identifier.version = inputId.version;

    // Get device physical location.
    if(ioctl(fd, EVIOCGPHYS(sizeof(buffer) - 1), &buffer) < 1) {
        //fprintf(stderr, "could not get location for %s, %s\n", devicePath, strerror(errno));
    } else {
        buffer[sizeof(buffer) - 1] = '\0';
        identifier.location.setTo(buffer);
    }

    // Get device unique id.
    if(ioctl(fd, EVIOCGUNIQ(sizeof(buffer) - 1), &buffer) < 1) {
        //fprintf(stderr, "could not get idstring for %s, %s\n", devicePath, strerror(errno));
    } else {
        buffer[sizeof(buffer) - 1] = '\0';
        identifier.uniqueId.setTo(buffer);
    }

    // Fill in the descriptor.
    assignDescriptorLocked(identifier);

    // Allocate device.  (The device object takes ownership of the fd at this point.)
    int32_t deviceId = mNextDeviceId++;
    Device* device = new Device(fd, deviceId, String8(devicePath), identifier);

    ALOGV("add device %d: %s\n", deviceId, devicePath);
    ALOGV("  bus:        %04x\n"
         "  vendor      %04x\n"
         "  product     %04x\n"
         "  version     %04x\n",
        identifier.bus, identifier.vendor, identifier.product, identifier.version);
    ALOGV("  name:       \"%s\"\n", identifier.name.string());
    ALOGV("  location:   \"%s\"\n", identifier.location.string());
    ALOGV("  unique id:  \"%s\"\n", identifier.uniqueId.string());
    ALOGV("  descriptor: \"%s\"\n", identifier.descriptor.string());
    ALOGV("  driver:     v%d.%d.%d\n",
        driverVersion >> 16, (driverVersion >> 8) & 0xff, driverVersion & 0xff);

    // Load the configuration file for the device.
    loadConfigurationLocked(device);

    // Figure out the kinds of events the device reports.
    ioctl(fd, EVIOCGBIT(EV_KEY, sizeof(device->keyBitmask)), device->keyBitmask);
    ioctl(fd, EVIOCGBIT(EV_ABS, sizeof(device->absBitmask)), device->absBitmask);
    ioctl(fd, EVIOCGBIT(EV_REL, sizeof(device->relBitmask)), device->relBitmask);
    ioctl(fd, EVIOCGBIT(EV_SW, sizeof(device->swBitmask)), device->swBitmask);
    ioctl(fd, EVIOCGBIT(EV_LED, sizeof(device->ledBitmask)), device->ledBitmask);
    ioctl(fd, EVIOCGBIT(EV_FF, sizeof(device->ffBitmask)), device->ffBitmask);
    ioctl(fd, EVIOCGPROP(sizeof(device->propBitmask)), device->propBitmask);

    // See if this is a keyboard.  Ignore everything in the button range except for
    // joystick and gamepad buttons which are handled like keyboards for the most part.
    bool haveKeyboardKeys = containsNonZeroByte(device->keyBitmask, 0, sizeof_bit_array(BTN_MISC))
            || containsNonZeroByte(device->keyBitmask, sizeof_bit_array(KEY_OK),
                    sizeof_bit_array(KEY_MAX + 1));
    bool haveGamepadButtons = containsNonZeroByte(device->keyBitmask, sizeof_bit_array(BTN_MISC),
                    sizeof_bit_array(BTN_MOUSE))
            || containsNonZeroByte(device->keyBitmask, sizeof_bit_array(BTN_JOYSTICK),
                    sizeof_bit_array(BTN_DIGI));
    if (haveKeyboardKeys || haveGamepadButtons) {
        device->classes |= INPUT_DEVICE_CLASS_KEYBOARD;
    }

    // See if this is a cursor device such as a trackball or mouse.
    if (test_bit(BTN_MOUSE, device->keyBitmask)
            && test_bit(REL_X, device->relBitmask)
            && test_bit(REL_Y, device->relBitmask)) {
        device->classes |= INPUT_DEVICE_CLASS_CURSOR;
    }

    // See if this is a rotary encoder type device.
    String8 deviceType = String8();
    if (device->configuration &&
        device->configuration->tryGetProperty(String8("device.type"), deviceType)) {
            if (!deviceType.compare(String8("rotaryEncoder"))) {
                device->classes |= INPUT_DEVICE_CLASS_ROTARY_ENCODER;
            }
    }

    // See if this is a touch pad.
    // Is this a new modern multi-touch driver?
    if (test_bit(ABS_MT_POSITION_X, device->absBitmask)
            && test_bit(ABS_MT_POSITION_Y, device->absBitmask)) {
        // Some joysticks such as the PS3 controller report axes that conflict
        // with the ABS_MT range.  Try to confirm that the device really is
        // a touch screen.
        if (test_bit(BTN_TOUCH, device->keyBitmask) || !haveGamepadButtons) {
            device->classes |= INPUT_DEVICE_CLASS_TOUCH | INPUT_DEVICE_CLASS_TOUCH_MT;
        }
    // Is this an old style single-touch driver?
    } else if (test_bit(BTN_TOUCH, device->keyBitmask)
            && test_bit(ABS_X, device->absBitmask)
            && test_bit(ABS_Y, device->absBitmask)) {
        device->classes |= INPUT_DEVICE_CLASS_TOUCH;
    // Is this a BT stylus?
    } else if ((test_bit(ABS_PRESSURE, device->absBitmask) ||
                test_bit(BTN_TOUCH, device->keyBitmask))
            && !test_bit(ABS_X, device->absBitmask)
            && !test_bit(ABS_Y, device->absBitmask)) {
        device->classes |= INPUT_DEVICE_CLASS_EXTERNAL_STYLUS;
        // Keyboard will try to claim some of the buttons but we really want to reserve those so we
        // can fuse it with the touch screen data, so just take them back. Note this means an
        // external stylus cannot also be a keyboard device.
        device->classes &= ~INPUT_DEVICE_CLASS_KEYBOARD;
    }

    // See if this device is a joystick.
    // Assumes that joysticks always have gamepad buttons in order to distinguish them
    // from other devices such as accelerometers that also have absolute axes.
    if (haveGamepadButtons) {
        uint32_t assumedClasses = device->classes | INPUT_DEVICE_CLASS_JOYSTICK;
        for (int i = 0; i <= ABS_MAX; i++) {
            if (test_bit(i, device->absBitmask)
                    && (getAbsAxisUsage(i, assumedClasses) & INPUT_DEVICE_CLASS_JOYSTICK)) {
                device->classes = assumedClasses;
                break;
            }
        }
    }

    // Check whether this device has switches.
    for (int i = 0; i <= SW_MAX; i++) {
        if (test_bit(i, device->swBitmask)) {
            device->classes |= INPUT_DEVICE_CLASS_SWITCH;
            break;
        }
    }

    // Check whether this device supports the vibrator.
    if (test_bit(FF_RUMBLE, device->ffBitmask)) {
        device->classes |= INPUT_DEVICE_CLASS_VIBRATOR;
    }

    // Configure virtual keys.
    if ((device->classes & INPUT_DEVICE_CLASS_TOUCH)) {
        // Load the virtual keys for the touch screen, if any.
        // We do this now so that we can make sure to load the keymap if necessary.
        status_t status = loadVirtualKeyMapLocked(device);
        if (!status) {
            device->classes |= INPUT_DEVICE_CLASS_KEYBOARD;
        }
    }

    // Load the key map.
    // We need to do this for joysticks too because the key layout may specify axes.
    status_t keyMapStatus = NAME_NOT_FOUND;
    if (device->classes & (INPUT_DEVICE_CLASS_KEYBOARD | INPUT_DEVICE_CLASS_JOYSTICK)) {
        // Load the keymap for the device.
        keyMapStatus = loadKeyMapLocked(device);
    }

    // Configure the keyboard, gamepad or virtual keyboard.
    if (device->classes & INPUT_DEVICE_CLASS_KEYBOARD) {
        // Register the keyboard as a built-in keyboard if it is eligible.
        if (!keyMapStatus
                && mBuiltInKeyboardId == NO_BUILT_IN_KEYBOARD
                && isEligibleBuiltInKeyboard(device->identifier,
                        device->configuration, &device->keyMap)) {
            mBuiltInKeyboardId = device->id;
        }

        // 'Q' key support = cheap test of whether this is an alpha-capable kbd
        if (hasKeycodeLocked(device, AKEYCODE_Q)) {
            device->classes |= INPUT_DEVICE_CLASS_ALPHAKEY;
        }

        // See if this device has a DPAD.
        if (hasKeycodeLocked(device, AKEYCODE_DPAD_UP) &&
                hasKeycodeLocked(device, AKEYCODE_DPAD_DOWN) &&
                hasKeycodeLocked(device, AKEYCODE_DPAD_LEFT) &&
                hasKeycodeLocked(device, AKEYCODE_DPAD_RIGHT) &&
                hasKeycodeLocked(device, AKEYCODE_DPAD_CENTER)) {
            device->classes |= INPUT_DEVICE_CLASS_DPAD;
        }

        // See if this device has a gamepad.
        for (size_t i = 0; i < sizeof(GAMEPAD_KEYCODES)/sizeof(GAMEPAD_KEYCODES[0]); i++) {
            if (hasKeycodeLocked(device, GAMEPAD_KEYCODES[i])) {
                device->classes |= INPUT_DEVICE_CLASS_GAMEPAD;
                break;
            }
        }
    }

    // If the device isn't recognized as something we handle, don't monitor it.
    if (device->classes == 0) {
        ALOGV("Dropping device: id=%d, path='%s', name='%s'",
                deviceId, devicePath, device->identifier.name.string());
        delete device;
        return -1;
    }

    // Determine whether the device has a mic.
    if (deviceHasMicLocked(device)) {
        device->classes |= INPUT_DEVICE_CLASS_MIC;
    }

    // Determine whether the device is external or internal.
    if (isExternalDeviceLocked(device)) {
        device->classes |= INPUT_DEVICE_CLASS_EXTERNAL;
    }

    if (device->classes & (INPUT_DEVICE_CLASS_JOYSTICK | INPUT_DEVICE_CLASS_DPAD)
            && device->classes & INPUT_DEVICE_CLASS_GAMEPAD) {
        device->controllerNumber = getNextControllerNumberLocked(device);
        setLedForControllerLocked(device);
    }


    if (registerDeviceForEpollLocked(device) != OK) {
        delete device;
        return -1;
    }

    configureFd(device);

    ALOGI("New device: id=%d, fd=%d, path='%s', name='%s', classes=0x%x, "
            "configuration='%s', keyLayout='%s', keyCharacterMap='%s', builtinKeyboard=%s, ",
         deviceId, fd, devicePath, device->identifier.name.string(),
         device->classes,
         device->configurationFile.string(),
         device->keyMap.keyLayoutFile.string(),
         device->keyMap.keyCharacterMapFile.string(),
         toString(mBuiltInKeyboardId == deviceId));

    addDeviceLocked(device);
    return OK;
}

// 添加到设备列表
void EventHub::addDeviceLocked(Device* device) {
    mDevices.add(device->id, device);
    device->next = mOpeningDevices;
    mOpeningDevices = device;
}
```

#### 注册到epoll监控事件

```cpp
status_t EventHub::registerDeviceForEpollLocked(Device* device) {
    struct epoll_event eventItem;
    memset(&eventItem, 0, sizeof(eventItem));
    eventItem.events = EPOLLIN;
    if (mUsingEpollWakeup) {
        eventItem.events |= EPOLLWAKEUP;
    }
    eventItem.data.u32 = device->id;
    if (epoll_ctl(mEpollFd, EPOLL_CTL_ADD, device->fd, &eventItem)) {
        ALOGE("Could not add device fd to epoll instance.  errno=%d", errno);
        return -errno;
    }
    return OK;
}
```



### InputManager

```cpp
//http://androidxref.com/9.0.0_r3/xref/frameworks/native/services/inputflinger/InputManager.cpp
InputManager::InputManager(
        const sp<EventHubInterface>& eventHub,
        const sp<InputReaderPolicyInterface>& readerPolicy,
        const sp<InputDispatcherPolicyInterface>& dispatcherPolicy) {
    mDispatcher = new InputDispatcher(dispatcherPolicy);
    mReader = new InputReader(eventHub, readerPolicy, mDispatcher);
    initialize();
}

void InputManager::initialize() {
    mReaderThread = new InputReaderThread(mReader);
    mDispatcherThread = new InputDispatcherThread(mDispatcher);
}
```

InputManager 建立了一个 InputReader 进行读取输入，一个 InputDispatcher 进行分发输出。同时，将 InputReader 的回调设置为 InputDispatcher。

同时开启了两个线程：InputReaderThread，InputDispatcherThread。



## InputManagerService.start()

InputManagerService 会先启动 C 层的服务，再进行 Java 层的操作。主要是注册了几个设置。

```java
//http://androidxref.com/9.0.0_r3/xref/frameworks/base/services/core/java/com/android/server/input/InputManagerService.java
    public void start() {
        Slog.i(TAG, "Starting input manager");
        nativeStart(mPtr);

        // Add ourself to the Watchdog monitors.
        Watchdog.getInstance().addMonitor(this);

        registerPointerSpeedSettingObserver();
        registerShowTouchesSettingObserver();
        registerAccessibilityLargePointerSettingObserver();

        mContext.registerReceiver(new BroadcastReceiver() {
            @Override
            public void onReceive(Context context, Intent intent) {
                updatePointerSpeedFromSettings();
                updateShowTouchesFromSettings();
                updateAccessibilityLargePointerFromSettings();
            }
        }, new IntentFilter(Intent.ACTION_USER_SWITCHED), null, mHandler);

        updatePointerSpeedFromSettings();
        updateShowTouchesFromSettings();
        updateAccessibilityLargePointerFromSettings();
    }
```



### nativeStart()

在 C 层面，将输入事件读取线程，分发线程都启动了起来。

```cpp
http://androidxref.com/9.0.0_r3/xref/frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp
static void nativeStart(JNIEnv* env, jclass /* clazz */, jlong ptr) {
    NativeInputManager* im = reinterpret_cast<NativeInputManager*>(ptr);

    status_t result = im->getInputManager()->start();
    if (result) {
        jniThrowRuntimeException(env, "Input manager could not be started.");
    }
}

```



```cpp
status_t InputManager::start() {
    status_t result = mDispatcherThread->run("InputDispatcher", PRIORITY_URGENT_DISPLAY);
    if (result) {
        ALOGE("Could not start InputDispatcher thread due to error %d.", result);
        return result;
    }

    result = mReaderThread->run("InputReader", PRIORITY_URGENT_DISPLAY);
    if (result) {
        ALOGE("Could not start InputReader thread due to error %d.", result);

        mDispatcherThread->requestExit();
        return result;
    }

    return OK;
}
```



## InputReader

InputReader 在构造的时候，会有一个队列，队列的回调就是 Dipatcher：

```cpp

InputReader::InputReader(const sp<EventHubInterface>& eventHub,
        const sp<InputReaderPolicyInterface>& policy,
        const sp<InputListenerInterface>& listener) :
        mContext(this), mEventHub(eventHub), mPolicy(policy),
        mGlobalMetaState(0), mGeneration(1),
        mDisableVirtualKeysTimeout(LLONG_MIN), mNextTimeout(LLONG_MAX),
        mConfigurationChangesToRefresh(0) {
    mQueuedListener = new QueuedInputListener(listener);

    { // acquire lock
        AutoMutex _l(mLock);

        refreshConfigurationLocked(0);
        updateGlobalMetaStateLocked();
    } // release lock
}
```



## InputReaderThread

InputReaderThread 继承自 **utils/thread.h**中的  Thread。

```cpp
// http://androidxref.com/9.0.0_r3/xref/system/core/include/utils/Thread.h

class Thread : virtual public RefBase
{
public:
    // Create a Thread object, but doesn't create or start the associated
    // thread. See the run() method.
    explicit            Thread(bool canCallJava = true);
    virtual             ~Thread();

    // Start the thread in threadLoop() which needs to be implemented.
    virtual status_t    run(    const char* name,
                                int32_t priority = PRIORITY_DEFAULT,
                                size_t stack = 0);

    // Ask this object's thread to exit. This function is asynchronous, when the
    // function returns the thread might still be running. Of course, this
    // function can be called from a different thread.
    virtual void        requestExit();

    // Good place to do one-time initializations
    virtual status_t    readyToRun();

    // Call requestExit() and wait until this object's thread exits.
    // BE VERY CAREFUL of deadlocks. In particular, it would be silly to call
    // this function from this object's thread. Will return WOULD_BLOCK in
    // that case.
            status_t    requestExitAndWait();

    // Wait until this object's thread exits. Returns immediately if not yet running.
    // Do not call from this object's thread; will return WOULD_BLOCK in that case.
            status_t    join();

    // Indicates whether this thread is running or not.
            bool        isRunning() const;

#if defined(__ANDROID__)
    // Return the thread's kernel ID, same as the thread itself calling gettid(),
    // or -1 if the thread is not running.
            pid_t       getTid() const;
#endif

protected:
    // exitPending() returns true if requestExit() has been called.
            bool        exitPending() const;

private:
    // Derived class must implement threadLoop(). The thread starts its life
    // here. There are two ways of using the Thread object:
    // 1) loop: if threadLoop() returns true, it will be called again if
    //          requestExit() wasn't called.
    // 2) once: if threadLoop() returns false, the thread will exit upon return.
    virtual bool        threadLoop() = 0;

private:
    Thread& operator=(const Thread&);
    static  int             _threadLoop(void* user);
    const   bool            mCanCallJava;
    // always hold mLock when reading or writing
            thread_id_t     mThread;
    mutable Mutex           mLock;
            Condition       mThreadExitedCondition;
            status_t        mStatus;
    // note that all accesses of mExitPending and mRunning need to hold mLock
    volatile bool           mExitPending;
    volatile bool           mRunning;
            sp<Thread>      mHoldSelf;
#if defined(__ANDROID__)
    // legacy for debugging, not used by getTid() as it is set by the child thread
    // and so is not initialized until the child reaches that point
            pid_t           mTid;
#endif
};
```

调用 run 方法，实际上就是开始线程的 threadLoop 循环而已了。

#### InputReader::loopOnce

```cpp
// http://androidxref.com/9.0.0_r3/xref/frameworks/native/services/inputflinger/InputReader.cpp
bool InputReaderThread::threadLoop() {
    mReader->loopOnce();
    return true;
}

void InputReader::loopOnce() {
    int32_t oldGeneration;
    int32_t timeoutMillis;
    bool inputDevicesChanged = false;
    Vector<InputDeviceInfo> inputDevices;
    { // acquire lock
        AutoMutex _l(mLock);

        oldGeneration = mGeneration;
        timeoutMillis = -1;

        uint32_t changes = mConfigurationChangesToRefresh;
        if (changes) {
            mConfigurationChangesToRefresh = 0;
            timeoutMillis = 0;
            refreshConfigurationLocked(changes);
        } else if (mNextTimeout != LLONG_MAX) {
            nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
            timeoutMillis = toMillisecondTimeoutDelay(now, mNextTimeout);
        }
    } // release lock

    size_t count = mEventHub->getEvents(timeoutMillis, mEventBuffer, EVENT_BUFFER_SIZE);

    { // acquire lock
        AutoMutex _l(mLock);
        mReaderIsAliveCondition.broadcast();

        if (count) {
            processEventsLocked(mEventBuffer, count);
        }

        if (mNextTimeout != LLONG_MAX) {
            nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
            if (now >= mNextTimeout) {
#if DEBUG_RAW_EVENTS
                ALOGD("Timeout expired, latency=%0.3fms", (now - mNextTimeout) * 0.000001f);
#endif
                mNextTimeout = LLONG_MAX;
                timeoutExpiredLocked(now);
            }
        }

        if (oldGeneration != mGeneration) {
            inputDevicesChanged = true;
            getInputDevicesLocked(inputDevices);
        }
    } // release lock

    // Send out a message that the describes the changed input devices.
    if (inputDevicesChanged) {
        mPolicy->notifyInputDevicesChanged(inputDevices);
    }

    // Flush queued events out to the listener.
    // This must happen outside of the lock because the listener could potentially call
    // back into the InputReader's methods, such as getScanCodeState, or become blocked
    // on another thread similarly waiting to acquire the InputReader lock thereby
    // resulting in a deadlock.  This situation is actually quite plausible because the
    // listener is actually the input dispatcher, which calls into the window manager,
    // which occasionally calls into the input reader.
    mQueuedListener->flush();
}
```

#### InputReader::processEventsLocked

每次循环都会从 EventHub 中读取事件，然后进行处理，对于在 EventHub 内扫描后会出现的新增，删除事件， InputReader 会被对应的设备进行存储，同时为设备建立 mapper。

```cpp
void InputReader::processEventsLocked(const RawEvent* rawEvents, size_t count) {
    for (const RawEvent* rawEvent = rawEvents; count;) {
        int32_t type = rawEvent->type;
        size_t batchSize = 1;
        if (type < EventHubInterface::FIRST_SYNTHETIC_EVENT) {
            int32_t deviceId = rawEvent->deviceId;
            while (batchSize < count) {
                if (rawEvent[batchSize].type >= EventHubInterface::FIRST_SYNTHETIC_EVENT
                        || rawEvent[batchSize].deviceId != deviceId) {
                    break;
                }
                batchSize += 1;
            }
#if DEBUG_RAW_EVENTS
            ALOGD("BatchSize: %zu Count: %zu", batchSize, count);
#endif
            processEventsForDeviceLocked(deviceId, rawEvent, batchSize);
        } else {
            switch (rawEvent->type) {
            case EventHubInterface::DEVICE_ADDED:
                addDeviceLocked(rawEvent->when, rawEvent->deviceId);
                break;
            case EventHubInterface::DEVICE_REMOVED:
                removeDeviceLocked(rawEvent->when, rawEvent->deviceId);
                break;
            case EventHubInterface::FINISHED_DEVICE_SCAN:
                handleConfigurationChangedLocked(rawEvent->when);
                break;
            default:
                ALOG_ASSERT(false); // can't happen
                break;
            }
        }
        count -= batchSize;
        rawEvent += batchSize;
    }
}
```
```cpp
void InputReader::addDeviceLocked(nsecs_t when, int32_t deviceId) {
    ssize_t deviceIndex = mDevices.indexOfKey(deviceId);
    if (deviceIndex >= 0) {
        ALOGW("Ignoring spurious device added event for deviceId %d.", deviceId);
        return;
    }

    InputDeviceIdentifier identifier = mEventHub->getDeviceIdentifier(deviceId);
    uint32_t classes = mEventHub->getDeviceClasses(deviceId);
    int32_t controllerNumber = mEventHub->getDeviceControllerNumber(deviceId);

    InputDevice* device = createDeviceLocked(deviceId, controllerNumber, identifier, classes);
    device->configure(when, &mConfig, 0);
    device->reset(when);

    if (device->isIgnored()) {
        ALOGI("Device added: id=%d, name='%s' (ignored non-input device)", deviceId,
                identifier.name.string());
    } else {
        ALOGI("Device added: id=%d, name='%s', sources=0x%08x", deviceId,
                identifier.name.string(), device->getSources());
    }

    mDevices.add(deviceId, device);
    bumpGenerationLocked();

    if (device->getClasses() & INPUT_DEVICE_CLASS_EXTERNAL_STYLUS) {
        notifyExternalStylusPresenceChanged();
    }
}


InputDevice* InputReader::createDeviceLocked(int32_t deviceId, int32_t controllerNumber,
        const InputDeviceIdentifier& identifier, uint32_t classes) {
    InputDevice* device = new InputDevice(&mContext, deviceId, bumpGenerationLocked(),
            controllerNumber, identifier, classes);

    // External devices.
    if (classes & INPUT_DEVICE_CLASS_EXTERNAL) {
        device->setExternal(true);
    }

    // Devices with mics.
    if (classes & INPUT_DEVICE_CLASS_MIC) {
        device->setMic(true);
    }

    // Switch-like devices.
    if (classes & INPUT_DEVICE_CLASS_SWITCH) {
        device->addMapper(new SwitchInputMapper(device));
    }

    // Scroll wheel-like devices.
    if (classes & INPUT_DEVICE_CLASS_ROTARY_ENCODER) {
        device->addMapper(new RotaryEncoderInputMapper(device));
    }

    // Vibrator-like devices.
    if (classes & INPUT_DEVICE_CLASS_VIBRATOR) {
        device->addMapper(new VibratorInputMapper(device));
    }

    // Keyboard-like devices.
    uint32_t keyboardSource = 0;
    int32_t keyboardType = AINPUT_KEYBOARD_TYPE_NON_ALPHABETIC;
    if (classes & INPUT_DEVICE_CLASS_KEYBOARD) {
        keyboardSource |= AINPUT_SOURCE_KEYBOARD;
    }
    if (classes & INPUT_DEVICE_CLASS_ALPHAKEY) {
        keyboardType = AINPUT_KEYBOARD_TYPE_ALPHABETIC;
    }
    if (classes & INPUT_DEVICE_CLASS_DPAD) {
        keyboardSource |= AINPUT_SOURCE_DPAD;
    }
    if (classes & INPUT_DEVICE_CLASS_GAMEPAD) {
        keyboardSource |= AINPUT_SOURCE_GAMEPAD;
    }

    if (keyboardSource != 0) {
        device->addMapper(new KeyboardInputMapper(device, keyboardSource, keyboardType));
    }

    // Cursor-like devices.
    if (classes & INPUT_DEVICE_CLASS_CURSOR) {
        device->addMapper(new CursorInputMapper(device));
    }

    // Touchscreens and touchpad devices.
    if (classes & INPUT_DEVICE_CLASS_TOUCH_MT) {
        device->addMapper(new MultiTouchInputMapper(device));
    } else if (classes & INPUT_DEVICE_CLASS_TOUCH) {
        device->addMapper(new SingleTouchInputMapper(device));
    }

    // Joystick-like devices.
    if (classes & INPUT_DEVICE_CLASS_JOYSTICK) {
        device->addMapper(new JoystickInputMapper(device));
    }

    // External stylus-like devices.
    if (classes & INPUT_DEVICE_CLASS_EXTERNAL_STYLUS) {
        device->addMapper(new ExternalStylusInputMapper(device));
    }

    return device;
}
```



#### InputReader::processEventsForDeviceLocked

```cpp
// 处理设备添加，移除，扫描外的事件
void InputReader::processEventsForDeviceLocked(int32_t deviceId,
        const RawEvent* rawEvents, size_t count) {
    ssize_t deviceIndex = mDevices.indexOfKey(deviceId);
    if (deviceIndex < 0) {
        ALOGW("Discarding event for unknown deviceId %d.", deviceId);
        return;
    }

    InputDevice* device = mDevices.valueAt(deviceIndex);
    if (device->isIgnored()) {
        //ALOGD("Discarding event for ignored deviceId %d.", deviceId);
        return;
    }

    device->process(rawEvents, count);
}
```

#### InputDevice::process

```cpp
void InputDevice::process(const RawEvent* rawEvents, size_t count) {
    // Process all of the events in order for each mapper.
    // We cannot simply ask each mapper to process them in bulk because mappers may
    // have side-effects that must be interleaved.  For example, joystick movement events and
    // gamepad button presses are handled by different mappers but they should be dispatched
    // in the order received.
    size_t numMappers = mMappers.size();
    for (const RawEvent* rawEvent = rawEvents; count != 0; rawEvent++) {
#if DEBUG_RAW_EVENTS
        ALOGD("Input event: device=%d type=0x%04x code=0x%04x value=0x%08x when=%" PRId64,
                rawEvent->deviceId, rawEvent->type, rawEvent->code, rawEvent->value,
                rawEvent->when);
#endif

        if (mDropUntilNextSync) {
            if (rawEvent->type == EV_SYN && rawEvent->code == SYN_REPORT) {
                mDropUntilNextSync = false;
#if DEBUG_RAW_EVENTS
                ALOGD("Recovered from input event buffer overrun.");
#endif
            } else {
#if DEBUG_RAW_EVENTS
                ALOGD("Dropped input event while waiting for next input sync.");
#endif
            }
        } else if (rawEvent->type == EV_SYN && rawEvent->code == SYN_DROPPED) {
            ALOGI("Detected input event buffer overrun for device %s.", getName().string());
            mDropUntilNextSync = true;
            reset(rawEvent->when);
        } else {
            for (size_t i = 0; i < numMappers; i++) {
                InputMapper* mapper = mMappers[i];
                mapper->process(rawEvent);
            }
        }
        --count;
    }
}

```

#### KeyboardInputMapper::process

```cpp
void KeyboardInputMapper::process(const RawEvent* rawEvent) {
    switch (rawEvent->type) {
    case EV_KEY: {
        int32_t scanCode = rawEvent->code;
        int32_t usageCode = mCurrentHidUsage;
        mCurrentHidUsage = 0;

        if (isKeyboardOrGamepadKey(scanCode)) {
            processKey(rawEvent->when, rawEvent->value != 0, scanCode, usageCode);
        }
        break;
    }
    case EV_MSC: {
        if (rawEvent->code == MSC_SCAN) {
            mCurrentHidUsage = rawEvent->value;
        }
        break;
    }
    case EV_SYN: {
        if (rawEvent->code == SYN_REPORT) {
            mCurrentHidUsage = 0;
        }
    }
    }
}
```

#### KeyboardInputMapper::processKey

```cpp
void KeyboardInputMapper::processKey(nsecs_t when, bool down, int32_t scanCode,
        int32_t usageCode) {
    int32_t keyCode;
    int32_t keyMetaState;
    uint32_t policyFlags;

    if (getEventHub()->mapKey(getDeviceId(), scanCode, usageCode, mMetaState,
                              &keyCode, &keyMetaState, &policyFlags)) {
        keyCode = AKEYCODE_UNKNOWN;
        keyMetaState = mMetaState;
        policyFlags = 0;
    }

    if (down) {
        // Rotate key codes according to orientation if needed.
        if (mParameters.orientationAware && mParameters.hasAssociatedDisplay) {
            keyCode = rotateKeyCode(keyCode, mOrientation);
        }

        // Add key down.
        ssize_t keyDownIndex = findKeyDown(scanCode);
        if (keyDownIndex >= 0) {
            // key repeat, be sure to use same keycode as before in case of rotation
            keyCode = mKeyDowns.itemAt(keyDownIndex).keyCode;
        } else {
            // key down
            if ((policyFlags & POLICY_FLAG_VIRTUAL)
                    && mContext->shouldDropVirtualKey(when,
                            getDevice(), keyCode, scanCode)) {
                return;
            }
            if (policyFlags & POLICY_FLAG_GESTURE) {
                mDevice->cancelTouch(when);
            }

            mKeyDowns.push();
            KeyDown& keyDown = mKeyDowns.editTop();
            keyDown.keyCode = keyCode;
            keyDown.scanCode = scanCode;
        }

        mDownTime = when;
    } else {
        // Remove key down.
        ssize_t keyDownIndex = findKeyDown(scanCode);
        if (keyDownIndex >= 0) {
            // key up, be sure to use same keycode as before in case of rotation
            keyCode = mKeyDowns.itemAt(keyDownIndex).keyCode;
            mKeyDowns.removeAt(size_t(keyDownIndex));
        } else {
            // key was not actually down
            ALOGI("Dropping key up from device %s because the key was not down.  "
                    "keyCode=%d, scanCode=%d",
                    getDeviceName().string(), keyCode, scanCode);
            return;
        }
    }

    if (updateMetaStateIfNeeded(keyCode, down)) {
        // If global meta state changed send it along with the key.
        // If it has not changed then we'll use what keymap gave us,
        // since key replacement logic might temporarily reset a few
        // meta bits for given key.
        keyMetaState = mMetaState;
    }

    nsecs_t downTime = mDownTime;

    // Key down on external an keyboard should wake the device.
    // We don't do this for internal keyboards to prevent them from waking up in your pocket.
    // For internal keyboards, the key layout file should specify the policy flags for
    // each wake key individually.
    // TODO: Use the input device configuration to control this behavior more finely.
    if (down && getDevice()->isExternal() && !isMediaKey(keyCode)) {
        policyFlags |= POLICY_FLAG_WAKE;
    }

    if (mParameters.handlesKeyRepeat) {
        policyFlags |= POLICY_FLAG_DISABLE_KEY_REPEAT;
    }

    NotifyKeyArgs args(when, getDeviceId(), mSource, policyFlags,
            down ? AKEY_EVENT_ACTION_DOWN : AKEY_EVENT_ACTION_UP,
            AKEY_EVENT_FLAG_FROM_SYSTEM, keyCode, scanCode, keyMetaState, downTime);
    getListener()->notifyKey(&args);
}
```

在这里就会将按键信息，通知到 mQueuedListener。

#### QueuedInputListener::notifyKey

```cpp
void QueuedInputListener::notifyKey(const NotifyKeyArgs* args) {
    mArgsQueue.push(new NotifyKeyArgs(*args));
}
```

#### QueuedInputListener::flush

```cpp
void QueuedInputListener::flush() {
    size_t count = mArgsQueue.size();
    for (size_t i = 0; i < count; i++) {
        NotifyArgs* args = mArgsQueue[i];
        args->notify(mInnerListener);
        delete args;
    }
    mArgsQueue.clear();
}
```



mQueuedListener 中。mQueuedListener.flush() 会将所有的事件通知到 Dispatcher（mInnerListener 就是 Dispatcher）。

#### InputDispatcher::notifyKey

```cpp

void InputDispatcher::notifyKey(const NotifyKeyArgs* args) {
#if DEBUG_INBOUND_EVENT_DETAILS
    ALOGD("notifyKey - eventTime=%" PRId64
            ", deviceId=%d, source=0x%x, policyFlags=0x%x, action=0x%x, "
            "flags=0x%x, keyCode=0x%x, scanCode=0x%x, metaState=0x%x, downTime=%" PRId64,
            args->eventTime, args->deviceId, args->source, args->policyFlags,
            args->action, args->flags, args->keyCode, args->scanCode,
            args->metaState, args->downTime);
#endif
    if (!validateKeyEvent(args->action)) {
        return;
    }

    uint32_t policyFlags = args->policyFlags;
    int32_t flags = args->flags;
    int32_t metaState = args->metaState;
    if ((policyFlags & POLICY_FLAG_VIRTUAL) || (flags & AKEY_EVENT_FLAG_VIRTUAL_HARD_KEY)) {
        policyFlags |= POLICY_FLAG_VIRTUAL;
        flags |= AKEY_EVENT_FLAG_VIRTUAL_HARD_KEY;
    }
    if (policyFlags & POLICY_FLAG_FUNCTION) {
        metaState |= AMETA_FUNCTION_ON;
    }

    policyFlags |= POLICY_FLAG_TRUSTED;

    int32_t keyCode = args->keyCode;
    if (metaState & AMETA_META_ON && args->action == AKEY_EVENT_ACTION_DOWN) {
        int32_t newKeyCode = AKEYCODE_UNKNOWN;
        if (keyCode == AKEYCODE_DEL) {
            newKeyCode = AKEYCODE_BACK;
        } else if (keyCode == AKEYCODE_ENTER) {
            newKeyCode = AKEYCODE_HOME;
        }
        if (newKeyCode != AKEYCODE_UNKNOWN) {
            AutoMutex _l(mLock);
            struct KeyReplacement replacement = {keyCode, args->deviceId};
            mReplacedKeys.add(replacement, newKeyCode);
            keyCode = newKeyCode;
            metaState &= ~(AMETA_META_ON | AMETA_META_LEFT_ON | AMETA_META_RIGHT_ON);
        }
    } else if (args->action == AKEY_EVENT_ACTION_UP) {
        // In order to maintain a consistent stream of up and down events, check to see if the key
        // going up is one we've replaced in a down event and haven't yet replaced in an up event,
        // even if the modifier was released between the down and the up events.
        AutoMutex _l(mLock);
        struct KeyReplacement replacement = {keyCode, args->deviceId};
        ssize_t index = mReplacedKeys.indexOfKey(replacement);
        if (index >= 0) {
            keyCode = mReplacedKeys.valueAt(index);
            mReplacedKeys.removeItemsAt(index);
            metaState &= ~(AMETA_META_ON | AMETA_META_LEFT_ON | AMETA_META_RIGHT_ON);
        }
    }

    KeyEvent event;
    event.initialize(args->deviceId, args->source, args->action,
            flags, keyCode, args->scanCode, metaState, 0,
            args->downTime, args->eventTime);

    android::base::Timer t;
    mPolicy->interceptKeyBeforeQueueing(&event, /*byref*/ policyFlags);
    if (t.duration() > SLOW_INTERCEPTION_THRESHOLD) {
        ALOGW("Excessive delay in interceptKeyBeforeQueueing; took %s ms",
                std::to_string(t.duration().count()).c_str());
    }

    bool needWake;
    { // acquire lock
        mLock.lock();

        if (shouldSendKeyToInputFilterLocked(args)) {
            mLock.unlock();

            policyFlags |= POLICY_FLAG_FILTERED;
            if (!mPolicy->filterInputEvent(&event, policyFlags)) {
                return; // event was consumed by the filter
            }

            mLock.lock();
        }

        int32_t repeatCount = 0;
        KeyEntry* newEntry = new KeyEntry(args->eventTime,
                args->deviceId, args->source, policyFlags,
                args->action, flags, keyCode, args->scanCode,
                metaState, repeatCount, args->downTime);

        needWake = enqueueInboundEventLocked(newEntry);
        mLock.unlock();
    } // release lock

    if (needWake) {
        mLooper->wake();
    }
}

```

#### NativeInputManager::interceptKeyBeforeQueueing

```cpp

void NativeInputManager::interceptKeyBeforeQueueing(const KeyEvent* keyEvent,
        uint32_t& policyFlags) {
    ATRACE_CALL();
    // Policy:
    // - Ignore untrusted events and pass them along.
    // - Ask the window manager what to do with normal events and trusted injected events.
    // - For normal events wake and brighten the screen if currently off or dim.
    bool interactive = mInteractive.load();
    if (interactive) {
        policyFlags |= POLICY_FLAG_INTERACTIVE;
    }
    if ((policyFlags & POLICY_FLAG_TRUSTED)) {
        nsecs_t when = keyEvent->getEventTime();
        JNIEnv* env = jniEnv();
        jobject keyEventObj = android_view_KeyEvent_fromNative(env, keyEvent);
        jint wmActions;
        if (keyEventObj) {
            wmActions = env->CallIntMethod(mServiceObj,
                    gServiceClassInfo.interceptKeyBeforeQueueing,
                    keyEventObj, policyFlags);
            if (checkAndClearExceptionFromCallback(env, "interceptKeyBeforeQueueing")) {
                wmActions = 0;
            }
            android_view_KeyEvent_recycle(env, keyEventObj);
            env->DeleteLocalRef(keyEventObj);
        } else {
            ALOGE("Failed to obtain key event object for interceptKeyBeforeQueueing.");
            wmActions = 0;
        }

        handleInterceptActions(wmActions, when, /*byref*/ policyFlags);
    } else {
        if (interactive) {
            policyFlags |= POLICY_FLAG_PASS_TO_USER;
        }
    }
}

```

#### InputManagerService.interceptKeyBeforeQueueing

```java
    // Native callback.
    private int interceptKeyBeforeQueueing(KeyEvent event, int policyFlags) {
        return mWindowManagerCallbacks.interceptKeyBeforeQueueing(event, policyFlags);
    }
```



```java
// http://androidxref.com/9.0.0_r3/xref/frameworks/base/services/core/java/com/android/server/wm/InputMonitor.java

		/* Provides an opportunity for the window manager policy to intercept early key
     * processing as soon as the key has been read from the device. */
    @Override
    public int interceptKeyBeforeQueueing(KeyEvent event, int policyFlags) {
        return mService.mPolicy.interceptKeyBeforeQueueing(event, policyFlags);
    }

```

#### PhoneWindowManager:interceptKeyBeforeQueueing

```java
// http://androidxref.com/9.0.0_r3/xref/frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java

/** {@inheritDoc} */
    @Override
    public int interceptKeyBeforeQueueing(KeyEvent event, int policyFlags) {
        if (!mSystemBooted) {
            // If we have not yet booted, don't let key events do anything.
            return 0;
        }

        final boolean interactive = (policyFlags & FLAG_INTERACTIVE) != 0;
        final boolean down = event.getAction() == KeyEvent.ACTION_DOWN;
        final boolean canceled = event.isCanceled();
        final int keyCode = event.getKeyCode();

        final boolean isInjected = (policyFlags & WindowManagerPolicy.FLAG_INJECTED) != 0;

        // If screen is off then we treat the case where the keyguard is open but hidden
        // the same as if it were open and in front.
        // This will prevent any keys other than the power button from waking the screen
        // when the keyguard is hidden by another activity.
        final boolean keyguardActive = (mKeyguardDelegate == null ? false :
                                            (interactive ?
                                                isKeyguardShowingAndNotOccluded() :
                                                mKeyguardDelegate.isShowing()));

        if (DEBUG_INPUT) {
            Log.d(TAG, "interceptKeyTq keycode=" + keyCode
                    + " interactive=" + interactive + " keyguardActive=" + keyguardActive
                    + " policyFlags=" + Integer.toHexString(policyFlags));
        }

        // Basic policy based on interactive state.
        int result;
        boolean isWakeKey = (policyFlags & WindowManagerPolicy.FLAG_WAKE) != 0
                || event.isWakeKey();
        if (interactive || (isInjected && !isWakeKey)) {
            // When the device is interactive or the key is injected pass the
            // key to the application.
            result = ACTION_PASS_TO_USER;
            isWakeKey = false;

            if (interactive) {
                // If the screen is awake, but the button pressed was the one that woke the device
                // then don't pass it to the application
                if (keyCode == mPendingWakeKey && !down) {
                    result = 0;
                }
                // Reset the pending key
                mPendingWakeKey = PENDING_KEY_NULL;
            }
        } else if (!interactive && shouldDispatchInputWhenNonInteractive(event)) {
            // If we're currently dozing with the screen on and the keyguard showing, pass the key
            // to the application but preserve its wake key status to make sure we still move
            // from dozing to fully interactive if we would normally go from off to fully
            // interactive.
            result = ACTION_PASS_TO_USER;
            // Since we're dispatching the input, reset the pending key
            mPendingWakeKey = PENDING_KEY_NULL;
        } else {
            // When the screen is off and the key is not injected, determine whether
            // to wake the device but don't pass the key to the application.
            result = 0;
            if (isWakeKey && (!down || !isWakeKeyWhenScreenOff(keyCode))) {
                isWakeKey = false;
            }
            // Cache the wake key on down event so we can also avoid sending the up event to the app
            if (isWakeKey && down) {
                mPendingWakeKey = keyCode;
            }
        }

        // If the key would be handled globally, just return the result, don't worry about special
        // key processing.
        if (isValidGlobalKey(keyCode)
                && mGlobalKeyManager.shouldHandleGlobalKey(keyCode, event)) {
            if (isWakeKey) {
                wakeUp(event.getEventTime(), mAllowTheaterModeWakeFromKey, "android.policy:KEY");
            }
            return result;
        }

        // Enable haptics if down and virtual key without multiple repetitions. If this is a hard
        // virtual key such as a navigation bar button, only vibrate if flag is enabled.
        final boolean isNavBarVirtKey = ((event.getFlags() & KeyEvent.FLAG_VIRTUAL_HARD_KEY) != 0);
        boolean useHapticFeedback = down
                && (policyFlags & WindowManagerPolicy.FLAG_VIRTUAL) != 0
                && (!isNavBarVirtKey || mNavBarVirtualKeyHapticFeedbackEnabled)
                && event.getRepeatCount() == 0;

        // Handle special keys.
        switch (keyCode) {
            case KeyEvent.KEYCODE_BACK: {
                if (down) {
                    interceptBackKeyDown();
                } else {
                    boolean handled = interceptBackKeyUp(event);

                    // Don't pass back press to app if we've already handled it via long press
                    if (handled) {
                        result &= ~ACTION_PASS_TO_USER;
                    }
                }
                break;
            }

            case KeyEvent.KEYCODE_VOLUME_DOWN:
            case KeyEvent.KEYCODE_VOLUME_UP:
            case KeyEvent.KEYCODE_VOLUME_MUTE: {
                if (keyCode == KeyEvent.KEYCODE_VOLUME_DOWN) {
                    if (down) {
                        // Any activity on the vol down button stops the ringer toggle shortcut
                        cancelPendingRingerToggleChordAction();

                        if (interactive && !mScreenshotChordVolumeDownKeyTriggered
                                && (event.getFlags() & KeyEvent.FLAG_FALLBACK) == 0) {
                            mScreenshotChordVolumeDownKeyTriggered = true;
                            mScreenshotChordVolumeDownKeyTime = event.getDownTime();
                            mScreenshotChordVolumeDownKeyConsumed = false;
                            cancelPendingPowerKeyAction();
                            interceptScreenshotChord();
                            interceptAccessibilityShortcutChord();
                        }
                    } else {
                        mScreenshotChordVolumeDownKeyTriggered = false;
                        cancelPendingScreenshotChordAction();
                        cancelPendingAccessibilityShortcutAction();
                    }
                } else if (keyCode == KeyEvent.KEYCODE_VOLUME_UP) {
                    if (down) {
                        if (interactive && !mA11yShortcutChordVolumeUpKeyTriggered
                                && (event.getFlags() & KeyEvent.FLAG_FALLBACK) == 0) {
                            mA11yShortcutChordVolumeUpKeyTriggered = true;
                            mA11yShortcutChordVolumeUpKeyTime = event.getDownTime();
                            mA11yShortcutChordVolumeUpKeyConsumed = false;
                            cancelPendingPowerKeyAction();
                            cancelPendingScreenshotChordAction();
                            cancelPendingRingerToggleChordAction();

                            interceptAccessibilityShortcutChord();
                            interceptRingerToggleChord();
                        }
                    } else {
                        mA11yShortcutChordVolumeUpKeyTriggered = false;
                        cancelPendingScreenshotChordAction();
                        cancelPendingAccessibilityShortcutAction();
                        cancelPendingRingerToggleChordAction();
                    }
                }
                if (down) {
                    sendSystemKeyToStatusBarAsync(event.getKeyCode());

                    TelecomManager telecomManager = getTelecommService();
                    if (telecomManager != null && !mHandleVolumeKeysInWM) {
                        // When {@link #mHandleVolumeKeysInWM} is set, volume key events
                        // should be dispatched to WM.
                        if (telecomManager.isRinging()) {
                            // If an incoming call is ringing, either VOLUME key means
                            // "silence ringer".  We handle these keys here, rather than
                            // in the InCallScreen, to make sure we'll respond to them
                            // even if the InCallScreen hasn't come to the foreground yet.
                            // Look for the DOWN event here, to agree with the "fallback"
                            // behavior in the InCallScreen.
                            Log.i(TAG, "interceptKeyBeforeQueueing:"
                                  + " VOLUME key-down while ringing: Silence ringer!");

                            // Silence the ringer.  (It's safe to call this
                            // even if the ringer has already been silenced.)
                            telecomManager.silenceRinger();

                            // And *don't* pass this key thru to the current activity
                            // (which is probably the InCallScreen.)
                            result &= ~ACTION_PASS_TO_USER;
                            break;
                        }
                    }
                    int audioMode = AudioManager.MODE_NORMAL;
                    try {
                        audioMode = getAudioService().getMode();
                    } catch (Exception e) {
                        Log.e(TAG, "Error getting AudioService in interceptKeyBeforeQueueing.", e);
                    }
                    boolean isInCall = (telecomManager != null && telecomManager.isInCall()) ||
                            audioMode == AudioManager.MODE_IN_COMMUNICATION;
                    if (isInCall && (result & ACTION_PASS_TO_USER) == 0) {
                        // If we are in call but we decided not to pass the key to
                        // the application, just pass it to the session service.
                        MediaSessionLegacyHelper.getHelper(mContext).sendVolumeKeyEvent(
                                event, AudioManager.USE_DEFAULT_STREAM_TYPE, false);
                        break;
                    }
                }
                if (mUseTvRouting || mHandleVolumeKeysInWM) {
                    // Defer special key handlings to
                    // {@link interceptKeyBeforeDispatching()}.
                    result |= ACTION_PASS_TO_USER;
                } else if ((result & ACTION_PASS_TO_USER) == 0) {
                    // If we aren't passing to the user and no one else
                    // handled it send it to the session manager to
                    // figure out.
                    MediaSessionLegacyHelper.getHelper(mContext).sendVolumeKeyEvent(
                            event, AudioManager.USE_DEFAULT_STREAM_TYPE, true);
                }
                break;
            }

            case KeyEvent.KEYCODE_ENDCALL: {
                result &= ~ACTION_PASS_TO_USER;
                if (down) {
                    TelecomManager telecomManager = getTelecommService();
                    boolean hungUp = false;
                    if (telecomManager != null) {
                        hungUp = telecomManager.endCall();
                    }
                    if (interactive && !hungUp) {
                        mEndCallKeyHandled = false;
                        mHandler.postDelayed(mEndCallLongPress,
                                ViewConfiguration.get(mContext).getDeviceGlobalActionKeyTimeout());
                    } else {
                        mEndCallKeyHandled = true;
                    }
                } else {
                    if (!mEndCallKeyHandled) {
                        mHandler.removeCallbacks(mEndCallLongPress);
                        if (!canceled) {
                            if ((mEndcallBehavior
                                    & Settings.System.END_BUTTON_BEHAVIOR_HOME) != 0) {
                                if (goHome()) {
                                    break;
                                }
                            }
                            if ((mEndcallBehavior
                                    & Settings.System.END_BUTTON_BEHAVIOR_SLEEP) != 0) {
                                goToSleep(event.getEventTime(),
                                        PowerManager.GO_TO_SLEEP_REASON_POWER_BUTTON, 0);
                                isWakeKey = false;
                            }
                        }
                    }
                }
                break;
            }

            case KeyEvent.KEYCODE_POWER: {
                // Any activity on the power button stops the accessibility shortcut
                cancelPendingAccessibilityShortcutAction();
                result &= ~ACTION_PASS_TO_USER;
                isWakeKey = false; // wake-up will be handled separately
                if (down) {
                    interceptPowerKeyDown(event, interactive);
                } else {
                    interceptPowerKeyUp(event, interactive, canceled);
                }
                break;
            }

            case KeyEvent.KEYCODE_SYSTEM_NAVIGATION_DOWN:
                // fall through
            case KeyEvent.KEYCODE_SYSTEM_NAVIGATION_UP:
                // fall through
            case KeyEvent.KEYCODE_SYSTEM_NAVIGATION_LEFT:
                // fall through
            case KeyEvent.KEYCODE_SYSTEM_NAVIGATION_RIGHT: {
                result &= ~ACTION_PASS_TO_USER;
                interceptSystemNavigationKey(event);
                break;
            }

            case KeyEvent.KEYCODE_SLEEP: {
                result &= ~ACTION_PASS_TO_USER;
                isWakeKey = false;
                if (!mPowerManager.isInteractive()) {
                    useHapticFeedback = false; // suppress feedback if already non-interactive
                }
                if (down) {
                    sleepPress();
                } else {
                    sleepRelease(event.getEventTime());
                }
                break;
            }

            case KeyEvent.KEYCODE_SOFT_SLEEP: {
                result &= ~ACTION_PASS_TO_USER;
                isWakeKey = false;
                if (!down) {
                    mPowerManagerInternal.setUserInactiveOverrideFromWindowManager();
                }
                break;
            }

            case KeyEvent.KEYCODE_WAKEUP: {
                result &= ~ACTION_PASS_TO_USER;
                isWakeKey = true;
                break;
            }

            case KeyEvent.KEYCODE_MEDIA_PLAY:
            case KeyEvent.KEYCODE_MEDIA_PAUSE:
            case KeyEvent.KEYCODE_MEDIA_PLAY_PAUSE:
            case KeyEvent.KEYCODE_HEADSETHOOK:
            case KeyEvent.KEYCODE_MUTE:
            case KeyEvent.KEYCODE_MEDIA_STOP:
            case KeyEvent.KEYCODE_MEDIA_NEXT:
            case KeyEvent.KEYCODE_MEDIA_PREVIOUS:
            case KeyEvent.KEYCODE_MEDIA_REWIND:
            case KeyEvent.KEYCODE_MEDIA_RECORD:
            case KeyEvent.KEYCODE_MEDIA_FAST_FORWARD:
            case KeyEvent.KEYCODE_MEDIA_AUDIO_TRACK: {
                if (MediaSessionLegacyHelper.getHelper(mContext).isGlobalPriorityActive()) {
                    // If the global session is active pass all media keys to it
                    // instead of the active window.
                    result &= ~ACTION_PASS_TO_USER;
                }
                if ((result & ACTION_PASS_TO_USER) == 0) {
                    // Only do this if we would otherwise not pass it to the user. In that
                    // case, the PhoneWindow class will do the same thing, except it will
                    // only do it if the showing app doesn't process the key on its own.
                    // Note that we need to make a copy of the key event here because the
                    // original key event will be recycled when we return.
                    mBroadcastWakeLock.acquire();
                    Message msg = mHandler.obtainMessage(MSG_DISPATCH_MEDIA_KEY_WITH_WAKE_LOCK,
                            new KeyEvent(event));
                    msg.setAsynchronous(true);
                    msg.sendToTarget();
                }
                break;
            }

            case KeyEvent.KEYCODE_CALL: {
                if (down) {
                    TelecomManager telecomManager = getTelecommService();
                    if (telecomManager != null) {
                        if (telecomManager.isRinging()) {
                            Log.i(TAG, "interceptKeyBeforeQueueing:"
                                  + " CALL key-down while ringing: Answer the call!");
                            telecomManager.acceptRingingCall();

                            // And *don't* pass this key thru to the current activity
                            // (which is presumably the InCallScreen.)
                            result &= ~ACTION_PASS_TO_USER;
                        }
                    }
                }
                break;
            }
            case KeyEvent.KEYCODE_ASSIST: {
                final boolean longPressed = event.getRepeatCount() > 0;
                if (down && longPressed) {
                    Message msg = mHandler.obtainMessage(MSG_LAUNCH_ASSIST_LONG_PRESS);
                    msg.setAsynchronous(true);
                    msg.sendToTarget();
                }
                if (!down && !longPressed) {
                    Message msg = mHandler.obtainMessage(MSG_LAUNCH_ASSIST, event.getDeviceId(),
                            0 /* unused */, null /* hint */);
                    msg.setAsynchronous(true);
                    msg.sendToTarget();
                }
                result &= ~ACTION_PASS_TO_USER;
                break;
            }
            case KeyEvent.KEYCODE_VOICE_ASSIST: {
                if (!down) {
                    mBroadcastWakeLock.acquire();
                    Message msg = mHandler.obtainMessage(MSG_LAUNCH_VOICE_ASSIST_WITH_WAKE_LOCK);
                    msg.setAsynchronous(true);
                    msg.sendToTarget();
                }
                result &= ~ACTION_PASS_TO_USER;
                break;
            }
            case KeyEvent.KEYCODE_WINDOW: {
                if (mShortPressOnWindowBehavior == SHORT_PRESS_WINDOW_PICTURE_IN_PICTURE) {
                    if (mPictureInPictureVisible) {
                        // Consumes the key only if picture-in-picture is visible to show
                        // picture-in-picture control menu. This gives a chance to the foreground
                        // activity to customize PIP key behavior.
                        if (!down) {
                            showPictureInPictureMenu(event);
                        }
                        result &= ~ACTION_PASS_TO_USER;
                    }
                }
                break;
            }
        }

        if (useHapticFeedback) {
            performHapticFeedbackLw(null, HapticFeedbackConstants.VIRTUAL_KEY, false);
        }

        if (isWakeKey) {
            wakeUp(event.getEventTime(), mAllowTheaterModeWakeFromKey, "android.policy:KEY");
        }

        return result;
    }
```

这个时候，事件已经进入了 dipatcher 的队列了，他就要准备开发发射数据。

\## InputDispatcher

```cpp

InputDispatcher::InputDispatcher(const sp<InputDispatcherPolicyInterface>& policy) :
    mPolicy(policy),
    mPendingEvent(NULL), mLastDropReason(DROP_REASON_NOT_DROPPED),
    mAppSwitchSawKeyDown(false), mAppSwitchDueTime(LONG_LONG_MAX),
    mNextUnblockedEvent(NULL),
    mDispatchEnabled(false), mDispatchFrozen(false), mInputFilterEnabled(false),
    mInputTargetWaitCause(INPUT_TARGET_WAIT_CAUSE_NONE) {
    mLooper = new Looper(false);

    mKeyRepeatState.lastKeyEntry = NULL;

    policy->getDispatcherConfiguration(&mConfig);
}
```



## InputDispatcher::dispatchOnce

```cpp
void InputDispatcher::dispatchOnce() {
    nsecs_t nextWakeupTime = LONG_LONG_MAX;
    { // acquire lock
        AutoMutex _l(mLock);
        mDispatcherIsAliveCondition.broadcast();

        // Run a dispatch loop if there are no pending commands.
        // The dispatch loop might enqueue commands to run afterwards.
        if (!haveCommandsLocked()) {
            dispatchOnceInnerLocked(&nextWakeupTime);
        }

        // Run all pending commands if there are any.
        // If any commands were run then force the next poll to wake up immediately.
        if (runCommandsLockedInterruptible()) {
            nextWakeupTime = LONG_LONG_MIN;
        }
    } // release lock

    // Wait for callback or timeout or wake.  (make sure we round up, not down)
    nsecs_t currentTime = now();
    int timeoutMillis = toMillisecondTimeoutDelay(currentTime, nextWakeupTime);
    mLooper->pollOnce(timeoutMillis);
}

```

#### InputDispatcher::dispatchOnceInnerLocked

```cpp

void InputDispatcher::dispatchOnceInnerLocked(nsecs_t* nextWakeupTime) {
    nsecs_t currentTime = now();

    // Reset the key repeat timer whenever normal dispatch is suspended while the
    // device is in a non-interactive state.  This is to ensure that we abort a key
    // repeat if the device is just coming out of sleep.
    if (!mDispatchEnabled) {
        resetKeyRepeatLocked();
    }

    // If dispatching is frozen, do not process timeouts or try to deliver any new events.
    if (mDispatchFrozen) {
#if DEBUG_FOCUS
        ALOGD("Dispatch frozen.  Waiting some more.");
#endif
        return;
    }

    // Optimize latency of app switches.
    // Essentially we start a short timeout when an app switch key (HOME / ENDCALL) has
    // been pressed.  When it expires, we preempt dispatch and drop all other pending events.
    bool isAppSwitchDue = mAppSwitchDueTime <= currentTime;
    if (mAppSwitchDueTime < *nextWakeupTime) {
        *nextWakeupTime = mAppSwitchDueTime;
    }

    // Ready to start a new event.
    // If we don't already have a pending event, go grab one.
    if (! mPendingEvent) {
        if (mInboundQueue.isEmpty()) {
            if (isAppSwitchDue) {
                // The inbound queue is empty so the app switch key we were waiting
                // for will never arrive.  Stop waiting for it.
                resetPendingAppSwitchLocked(false);
                isAppSwitchDue = false;
            }

            // Synthesize a key repeat if appropriate.
            if (mKeyRepeatState.lastKeyEntry) {
                if (currentTime >= mKeyRepeatState.nextRepeatTime) {
                    mPendingEvent = synthesizeKeyRepeatLocked(currentTime);
                } else {
                    if (mKeyRepeatState.nextRepeatTime < *nextWakeupTime) {
                        *nextWakeupTime = mKeyRepeatState.nextRepeatTime;
                    }
                }
            }

            // Nothing to do if there is no pending event.
            if (!mPendingEvent) {
                return;
            }
        } else {
            // Inbound queue has at least one entry.
            mPendingEvent = mInboundQueue.dequeueAtHead();
            traceInboundQueueLengthLocked();
        }

        // Poke user activity for this event.
        if (mPendingEvent->policyFlags & POLICY_FLAG_PASS_TO_USER) {
            pokeUserActivityLocked(mPendingEvent);
        }

        // Get ready to dispatch the event.
        resetANRTimeoutsLocked();
    }

    // Now we have an event to dispatch.
    // All events are eventually dequeued and processed this way, even if we intend to drop them.
    ALOG_ASSERT(mPendingEvent != NULL);
    bool done = false;
    DropReason dropReason = DROP_REASON_NOT_DROPPED;
    if (!(mPendingEvent->policyFlags & POLICY_FLAG_PASS_TO_USER)) {
        dropReason = DROP_REASON_POLICY;
    } else if (!mDispatchEnabled) {
        dropReason = DROP_REASON_DISABLED;
    }

    if (mNextUnblockedEvent == mPendingEvent) {
        mNextUnblockedEvent = NULL;
    }

    switch (mPendingEvent->type) {
    case EventEntry::TYPE_CONFIGURATION_CHANGED: {
        ConfigurationChangedEntry* typedEntry =
                static_cast<ConfigurationChangedEntry*>(mPendingEvent);
        done = dispatchConfigurationChangedLocked(currentTime, typedEntry);
        dropReason = DROP_REASON_NOT_DROPPED; // configuration changes are never dropped
        break;
    }

    case EventEntry::TYPE_DEVICE_RESET: {
        DeviceResetEntry* typedEntry =
                static_cast<DeviceResetEntry*>(mPendingEvent);
        done = dispatchDeviceResetLocked(currentTime, typedEntry);
        dropReason = DROP_REASON_NOT_DROPPED; // device resets are never dropped
        break;
    }

    case EventEntry::TYPE_KEY: {
        KeyEntry* typedEntry = static_cast<KeyEntry*>(mPendingEvent);
        if (isAppSwitchDue) {
            if (isAppSwitchKeyEventLocked(typedEntry)) {
                resetPendingAppSwitchLocked(true);
                isAppSwitchDue = false;
            } else if (dropReason == DROP_REASON_NOT_DROPPED) {
                dropReason = DROP_REASON_APP_SWITCH;
            }
        }
        if (dropReason == DROP_REASON_NOT_DROPPED
                && isStaleEventLocked(currentTime, typedEntry)) {
            dropReason = DROP_REASON_STALE;
        }
        if (dropReason == DROP_REASON_NOT_DROPPED && mNextUnblockedEvent) {
            dropReason = DROP_REASON_BLOCKED;
        }
        done = dispatchKeyLocked(currentTime, typedEntry, &dropReason, nextWakeupTime);
        break;
    }

    case EventEntry::TYPE_MOTION: {
        MotionEntry* typedEntry = static_cast<MotionEntry*>(mPendingEvent);
        if (dropReason == DROP_REASON_NOT_DROPPED && isAppSwitchDue) {
            dropReason = DROP_REASON_APP_SWITCH;
        }
        if (dropReason == DROP_REASON_NOT_DROPPED
                && isStaleEventLocked(currentTime, typedEntry)) {
            dropReason = DROP_REASON_STALE;
        }
        if (dropReason == DROP_REASON_NOT_DROPPED && mNextUnblockedEvent) {
            dropReason = DROP_REASON_BLOCKED;
        }
        done = dispatchMotionLocked(currentTime, typedEntry,
                &dropReason, nextWakeupTime);
        break;
    }

    default:
        ALOG_ASSERT(false);
        break;
    }

    if (done) {
        if (dropReason != DROP_REASON_NOT_DROPPED) {
            dropInboundEventLocked(mPendingEvent, dropReason);
        }
        mLastDropReason = dropReason;

        releasePendingEventLocked();
        *nextWakeupTime = LONG_LONG_MIN;  // force next poll to wake up immediately
    }
}
```

#### InputDispatcher::dispatchKeyLocked

```cpp
bool InputDispatcher::dispatchKeyLocked(nsecs_t currentTime, KeyEntry* entry,
        DropReason* dropReason, nsecs_t* nextWakeupTime) {
    // Preprocessing.
    if (! entry->dispatchInProgress) {
        if (entry->repeatCount == 0
                && entry->action == AKEY_EVENT_ACTION_DOWN
                && (entry->policyFlags & POLICY_FLAG_TRUSTED)
                && (!(entry->policyFlags & POLICY_FLAG_DISABLE_KEY_REPEAT))) {
            if (mKeyRepeatState.lastKeyEntry
                    && mKeyRepeatState.lastKeyEntry->keyCode == entry->keyCode) {
                // We have seen two identical key downs in a row which indicates that the device
                // driver is automatically generating key repeats itself.  We take note of the
                // repeat here, but we disable our own next key repeat timer since it is clear that
                // we will not need to synthesize key repeats ourselves.
                entry->repeatCount = mKeyRepeatState.lastKeyEntry->repeatCount + 1;
                resetKeyRepeatLocked();
                mKeyRepeatState.nextRepeatTime = LONG_LONG_MAX; // don't generate repeats ourselves
            } else {
                // Not a repeat.  Save key down state in case we do see a repeat later.
                resetKeyRepeatLocked();
                mKeyRepeatState.nextRepeatTime = entry->eventTime + mConfig.keyRepeatTimeout;
            }
            mKeyRepeatState.lastKeyEntry = entry;
            entry->refCount += 1;
        } else if (! entry->syntheticRepeat) {
            resetKeyRepeatLocked();
        }

        if (entry->repeatCount == 1) {
            entry->flags |= AKEY_EVENT_FLAG_LONG_PRESS;
        } else {
            entry->flags &= ~AKEY_EVENT_FLAG_LONG_PRESS;
        }

        entry->dispatchInProgress = true;

        logOutboundKeyDetailsLocked("dispatchKey - ", entry);
    }

    // Handle case where the policy asked us to try again later last time.
    if (entry->interceptKeyResult == KeyEntry::INTERCEPT_KEY_RESULT_TRY_AGAIN_LATER) {
        if (currentTime < entry->interceptKeyWakeupTime) {
            if (entry->interceptKeyWakeupTime < *nextWakeupTime) {
                *nextWakeupTime = entry->interceptKeyWakeupTime;
            }
            return false; // wait until next wakeup
        }
        entry->interceptKeyResult = KeyEntry::INTERCEPT_KEY_RESULT_UNKNOWN;
        entry->interceptKeyWakeupTime = 0;
    }

    // Give the policy a chance to intercept the key.
    if (entry->interceptKeyResult == KeyEntry::INTERCEPT_KEY_RESULT_UNKNOWN) {
        if (entry->policyFlags & POLICY_FLAG_PASS_TO_USER) {
            CommandEntry* commandEntry = postCommandLocked(
                    & InputDispatcher::doInterceptKeyBeforeDispatchingLockedInterruptible);
            if (mFocusedWindowHandle != NULL) {
                commandEntry->inputWindowHandle = mFocusedWindowHandle;
            }
            commandEntry->keyEntry = entry;
            entry->refCount += 1;
            return false; // wait for the command to run
        } else {
            entry->interceptKeyResult = KeyEntry::INTERCEPT_KEY_RESULT_CONTINUE;
        }
    } else if (entry->interceptKeyResult == KeyEntry::INTERCEPT_KEY_RESULT_SKIP) {
        if (*dropReason == DROP_REASON_NOT_DROPPED) {
            *dropReason = DROP_REASON_POLICY;
        }
    }

    // Clean up if dropping the event.
    if (*dropReason != DROP_REASON_NOT_DROPPED) {
        setInjectionResultLocked(entry, *dropReason == DROP_REASON_POLICY
                ? INPUT_EVENT_INJECTION_SUCCEEDED : INPUT_EVENT_INJECTION_FAILED);
        return true;
    }

    // Identify targets.
    Vector<InputTarget> inputTargets;
    int32_t injectionResult = findFocusedWindowTargetsLocked(currentTime,
            entry, inputTargets, nextWakeupTime);
    if (injectionResult == INPUT_EVENT_INJECTION_PENDING) {
        return false;
    }

    setInjectionResultLocked(entry, injectionResult);
    if (injectionResult != INPUT_EVENT_INJECTION_SUCCEEDED) {
        return true;
    }

    addMonitoringTargetsLocked(inputTargets);

    // Dispatch the key.
    dispatchEventLocked(currentTime, entry, inputTargets);
    return true;
}

```

#### findFocusedWindowTargetsLocked

```cpp
int32_t InputDispatcher::findFocusedWindowTargetsLocked(nsecs_t currentTime,
        const EventEntry* entry, Vector<InputTarget>& inputTargets, nsecs_t* nextWakeupTime) {
    int32_t injectionResult;
    std::string reason;

    // If there is no currently focused window and no focused application
    // then drop the event.
    if (mFocusedWindowHandle == NULL) {
        if (mFocusedApplicationHandle != NULL) {
            injectionResult = handleTargetsNotReadyLocked(currentTime, entry,
                    mFocusedApplicationHandle, NULL, nextWakeupTime,
                    "Waiting because no window has focus but there is a "
                    "focused application that may eventually add a window "
                    "when it finishes starting up.");
            goto Unresponsive;
        }

        ALOGI("Dropping event because there is no focused window or focused application.");
        injectionResult = INPUT_EVENT_INJECTION_FAILED;
        goto Failed;
    }

    // Check permissions.
    if (! checkInjectionPermission(mFocusedWindowHandle, entry->injectionState)) {
        injectionResult = INPUT_EVENT_INJECTION_PERMISSION_DENIED;
        goto Failed;
    }

    // Check whether the window is ready for more input.
    reason = checkWindowReadyForMoreInputLocked(currentTime,
            mFocusedWindowHandle, entry, "focused");
    if (!reason.empty()) {
        injectionResult = handleTargetsNotReadyLocked(currentTime, entry,
                mFocusedApplicationHandle, mFocusedWindowHandle, nextWakeupTime, reason.c_str());
        goto Unresponsive;
    }

    // Success!  Output targets.
    injectionResult = INPUT_EVENT_INJECTION_SUCCEEDED;
    addWindowTargetLocked(mFocusedWindowHandle,
            InputTarget::FLAG_FOREGROUND | InputTarget::FLAG_DISPATCH_AS_IS, BitSet32(0),
            inputTargets);

    // Done.
Failed:
Unresponsive:
    nsecs_t timeSpentWaitingForApplication = getTimeSpentWaitingForApplicationLocked(currentTime);
    updateDispatchStatisticsLocked(currentTime, entry,
            injectionResult, timeSpentWaitingForApplication);
#if DEBUG_FOCUS
    ALOGD("findFocusedWindow finished: injectionResult=%d, "
            "timeSpentWaitingForApplication=%0.1fms",
            injectionResult, timeSpentWaitingForApplication / 1000000.0);
#endif
    return injectionResult;
}
```

#### dispatchEventLocked

```cpp
void InputDispatcher::dispatchEventLocked(nsecs_t currentTime,
        EventEntry* eventEntry, const Vector<InputTarget>& inputTargets) {
#if DEBUG_DISPATCH_CYCLE
    ALOGD("dispatchEventToCurrentInputTargets");
#endif

    ALOG_ASSERT(eventEntry->dispatchInProgress); // should already have been set to true

    pokeUserActivityLocked(eventEntry);

    for (size_t i = 0; i < inputTargets.size(); i++) {
        const InputTarget& inputTarget = inputTargets.itemAt(i);

        ssize_t connectionIndex = getConnectionIndexLocked(inputTarget.inputChannel);
        if (connectionIndex >= 0) {
            sp<Connection> connection = mConnectionsByFd.valueAt(connectionIndex);
            prepareDispatchCycleLocked(currentTime, connection, eventEntry, &inputTarget);
        } else {
#if DEBUG_FOCUS
            ALOGD("Dropping event delivery to target with channel '%s' because it "
                    "is no longer registered with the input dispatcher.",
                    inputTarget.inputChannel->getName().c_str());
#endif
        }
    }
}
```

#### prepareDispatchCycleLocked

```cpp
void InputDispatcher::prepareDispatchCycleLocked(nsecs_t currentTime,
        const sp<Connection>& connection, EventEntry* eventEntry, const InputTarget* inputTarget) {
#if DEBUG_DISPATCH_CYCLE
    ALOGD("channel '%s' ~ prepareDispatchCycle - flags=0x%08x, "
            "xOffset=%f, yOffset=%f, scaleFactor=%f, "
            "pointerIds=0x%x",
            connection->getInputChannelName().c_str(), inputTarget->flags,
            inputTarget->xOffset, inputTarget->yOffset,
            inputTarget->scaleFactor, inputTarget->pointerIds.value);
#endif

    // Skip this event if the connection status is not normal.
    // We don't want to enqueue additional outbound events if the connection is broken.
    if (connection->status != Connection::STATUS_NORMAL) {
#if DEBUG_DISPATCH_CYCLE
        ALOGD("channel '%s' ~ Dropping event because the channel status is %s",
                connection->getInputChannelName().c_str(), connection->getStatusLabel());
#endif
        return;
    }

    // Split a motion event if needed.
    if (inputTarget->flags & InputTarget::FLAG_SPLIT) {
        ALOG_ASSERT(eventEntry->type == EventEntry::TYPE_MOTION);

        MotionEntry* originalMotionEntry = static_cast<MotionEntry*>(eventEntry);
        if (inputTarget->pointerIds.count() != originalMotionEntry->pointerCount) {
            MotionEntry* splitMotionEntry = splitMotionEvent(
                    originalMotionEntry, inputTarget->pointerIds);
            if (!splitMotionEntry) {
                return; // split event was dropped
            }
#if DEBUG_FOCUS
            ALOGD("channel '%s' ~ Split motion event.",
                    connection->getInputChannelName().c_str());
            logOutboundMotionDetailsLocked("  ", splitMotionEntry);
#endif
            enqueueDispatchEntriesLocked(currentTime, connection,
                    splitMotionEntry, inputTarget);
            splitMotionEntry->release();
            return;
        }
    }

    // Not splitting.  Enqueue dispatch entries for the event as is.
    enqueueDispatchEntriesLocked(currentTime, connection, eventEntry, inputTarget);
}


```

#### enqueueDispatchEntriesLocked

```cpp
void InputDispatcher::enqueueDispatchEntriesLocked(nsecs_t currentTime,
        const sp<Connection>& connection, EventEntry* eventEntry, const InputTarget* inputTarget) {
    bool wasEmpty = connection->outboundQueue.isEmpty();

    // Enqueue dispatch entries for the requested modes.
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
            InputTarget::FLAG_DISPATCH_AS_HOVER_EXIT);
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
            InputTarget::FLAG_DISPATCH_AS_OUTSIDE);
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
            InputTarget::FLAG_DISPATCH_AS_HOVER_ENTER);
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
            InputTarget::FLAG_DISPATCH_AS_IS);
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
            InputTarget::FLAG_DISPATCH_AS_SLIPPERY_EXIT);
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
            InputTarget::FLAG_DISPATCH_AS_SLIPPERY_ENTER);

    // If the outbound queue was previously empty, start the dispatch cycle going.
    if (wasEmpty && !connection->outboundQueue.isEmpty()) {
        startDispatchCycleLocked(currentTime, connection);
    }
}

void InputDispatcher::enqueueDispatchEntryLocked(
        const sp<Connection>& connection, EventEntry* eventEntry, const InputTarget* inputTarget,
        int32_t dispatchMode) {
    int32_t inputTargetFlags = inputTarget->flags;
    if (!(inputTargetFlags & dispatchMode)) {
        return;
    }
    inputTargetFlags = (inputTargetFlags & ~InputTarget::FLAG_DISPATCH_MASK) | dispatchMode;

    // This is a new event.
    // Enqueue a new dispatch entry onto the outbound queue for this connection.
    DispatchEntry* dispatchEntry = new DispatchEntry(eventEntry, // increments ref
            inputTargetFlags, inputTarget->xOffset, inputTarget->yOffset,
            inputTarget->scaleFactor);

    // Apply target flags and update the connection's input state.
    switch (eventEntry->type) {
    case EventEntry::TYPE_KEY: {
        KeyEntry* keyEntry = static_cast<KeyEntry*>(eventEntry);
        dispatchEntry->resolvedAction = keyEntry->action;
        dispatchEntry->resolvedFlags = keyEntry->flags;

        if (!connection->inputState.trackKey(keyEntry,
                dispatchEntry->resolvedAction, dispatchEntry->resolvedFlags)) {
#if DEBUG_DISPATCH_CYCLE
            ALOGD("channel '%s' ~ enqueueDispatchEntryLocked: skipping inconsistent key event",
                    connection->getInputChannelName().c_str());
#endif
            delete dispatchEntry;
            return; // skip the inconsistent event
        }
        break;
    }

    case EventEntry::TYPE_MOTION: {
        MotionEntry* motionEntry = static_cast<MotionEntry*>(eventEntry);
        if (dispatchMode & InputTarget::FLAG_DISPATCH_AS_OUTSIDE) {
            dispatchEntry->resolvedAction = AMOTION_EVENT_ACTION_OUTSIDE;
        } else if (dispatchMode & InputTarget::FLAG_DISPATCH_AS_HOVER_EXIT) {
            dispatchEntry->resolvedAction = AMOTION_EVENT_ACTION_HOVER_EXIT;
        } else if (dispatchMode & InputTarget::FLAG_DISPATCH_AS_HOVER_ENTER) {
            dispatchEntry->resolvedAction = AMOTION_EVENT_ACTION_HOVER_ENTER;
        } else if (dispatchMode & InputTarget::FLAG_DISPATCH_AS_SLIPPERY_EXIT) {
            dispatchEntry->resolvedAction = AMOTION_EVENT_ACTION_CANCEL;
        } else if (dispatchMode & InputTarget::FLAG_DISPATCH_AS_SLIPPERY_ENTER) {
            dispatchEntry->resolvedAction = AMOTION_EVENT_ACTION_DOWN;
        } else {
            dispatchEntry->resolvedAction = motionEntry->action;
        }
        if (dispatchEntry->resolvedAction == AMOTION_EVENT_ACTION_HOVER_MOVE
                && !connection->inputState.isHovering(
                        motionEntry->deviceId, motionEntry->source, motionEntry->displayId)) {
#if DEBUG_DISPATCH_CYCLE
        ALOGD("channel '%s' ~ enqueueDispatchEntryLocked: filling in missing hover enter event",
                connection->getInputChannelName().c_str());
#endif
            dispatchEntry->resolvedAction = AMOTION_EVENT_ACTION_HOVER_ENTER;
        }

        dispatchEntry->resolvedFlags = motionEntry->flags;
        if (dispatchEntry->targetFlags & InputTarget::FLAG_WINDOW_IS_OBSCURED) {
            dispatchEntry->resolvedFlags |= AMOTION_EVENT_FLAG_WINDOW_IS_OBSCURED;
        }
        if (dispatchEntry->targetFlags & InputTarget::FLAG_WINDOW_IS_PARTIALLY_OBSCURED) {
            dispatchEntry->resolvedFlags |= AMOTION_EVENT_FLAG_WINDOW_IS_PARTIALLY_OBSCURED;
        }

        if (!connection->inputState.trackMotion(motionEntry,
                dispatchEntry->resolvedAction, dispatchEntry->resolvedFlags)) {
#if DEBUG_DISPATCH_CYCLE
            ALOGD("channel '%s' ~ enqueueDispatchEntryLocked: skipping inconsistent motion event",
                    connection->getInputChannelName().c_str());
#endif
            delete dispatchEntry;
            return; // skip the inconsistent event
        }
        break;
    }
    }

    // Remember that we are waiting for this dispatch to complete.
    if (dispatchEntry->hasForegroundTarget()) {
        incrementPendingForegroundDispatchesLocked(eventEntry);
    }

    // Enqueue the dispatch entry.
    connection->outboundQueue.enqueueAtTail(dispatchEntry);
    traceOutboundQueueLengthLocked(connection);
}

void InputDispatcher::startDispatchCycleLocked(nsecs_t currentTime,
        const sp<Connection>& connection) {
#if DEBUG_DISPATCH_CYCLE
    ALOGD("channel '%s' ~ startDispatchCycle",
            connection->getInputChannelName().c_str());
#endif

    while (connection->status == Connection::STATUS_NORMAL
            && !connection->outboundQueue.isEmpty()) {
        DispatchEntry* dispatchEntry = connection->outboundQueue.head;
        dispatchEntry->deliveryTime = currentTime;

        // Publish the event.
        status_t status;
        EventEntry* eventEntry = dispatchEntry->eventEntry;
        switch (eventEntry->type) {
        case EventEntry::TYPE_KEY: {
            KeyEntry* keyEntry = static_cast<KeyEntry*>(eventEntry);

            // Publish the key event.
            status = connection->inputPublisher.publishKeyEvent(dispatchEntry->seq,
                    keyEntry->deviceId, keyEntry->source,
                    dispatchEntry->resolvedAction, dispatchEntry->resolvedFlags,
                    keyEntry->keyCode, keyEntry->scanCode,
                    keyEntry->metaState, keyEntry->repeatCount, keyEntry->downTime,
                    keyEntry->eventTime);
            break;
        }

        case EventEntry::TYPE_MOTION: {
            MotionEntry* motionEntry = static_cast<MotionEntry*>(eventEntry);

            PointerCoords scaledCoords[MAX_POINTERS];
            const PointerCoords* usingCoords = motionEntry->pointerCoords;

            // Set the X and Y offset depending on the input source.
            float xOffset, yOffset;
            if ((motionEntry->source & AINPUT_SOURCE_CLASS_POINTER)
                    && !(dispatchEntry->targetFlags & InputTarget::FLAG_ZERO_COORDS)) {
                float scaleFactor = dispatchEntry->scaleFactor;
                xOffset = dispatchEntry->xOffset * scaleFactor;
                yOffset = dispatchEntry->yOffset * scaleFactor;
                if (scaleFactor != 1.0f) {
                    for (uint32_t i = 0; i < motionEntry->pointerCount; i++) {
                        scaledCoords[i] = motionEntry->pointerCoords[i];
                        scaledCoords[i].scale(scaleFactor);
                    }
                    usingCoords = scaledCoords;
                }
            } else {
                xOffset = 0.0f;
                yOffset = 0.0f;

                // We don't want the dispatch target to know.
                if (dispatchEntry->targetFlags & InputTarget::FLAG_ZERO_COORDS) {
                    for (uint32_t i = 0; i < motionEntry->pointerCount; i++) {
                        scaledCoords[i].clear();
                    }
                    usingCoords = scaledCoords;
                }
            }

            // Publish the motion event.
            status = connection->inputPublisher.publishMotionEvent(dispatchEntry->seq,
                    motionEntry->deviceId, motionEntry->source, motionEntry->displayId,
                    dispatchEntry->resolvedAction, motionEntry->actionButton,
                    dispatchEntry->resolvedFlags, motionEntry->edgeFlags,
                    motionEntry->metaState, motionEntry->buttonState,
                    xOffset, yOffset, motionEntry->xPrecision, motionEntry->yPrecision,
                    motionEntry->downTime, motionEntry->eventTime,
                    motionEntry->pointerCount, motionEntry->pointerProperties,
                    usingCoords);
            break;
        }

        default:
            ALOG_ASSERT(false);
            return;
        }

        // Check the result.
        if (status) {
            if (status == WOULD_BLOCK) {
                if (connection->waitQueue.isEmpty()) {
                    ALOGE("channel '%s' ~ Could not publish event because the pipe is full. "
                            "This is unexpected because the wait queue is empty, so the pipe "
                            "should be empty and we shouldn't have any problems writing an "
                            "event to it, status=%d", connection->getInputChannelName().c_str(),
                            status);
                    abortBrokenDispatchCycleLocked(currentTime, connection, true /*notify*/);
                } else {
                    // Pipe is full and we are waiting for the app to finish process some events
                    // before sending more events to it.
#if DEBUG_DISPATCH_CYCLE
                    ALOGD("channel '%s' ~ Could not publish event because the pipe is full, "
                            "waiting for the application to catch up",
                            connection->getInputChannelName().c_str());
#endif
                    connection->inputPublisherBlocked = true;
                }
            } else {
                ALOGE("channel '%s' ~ Could not publish event due to an unexpected error, "
                        "status=%d", connection->getInputChannelName().c_str(), status);
                abortBrokenDispatchCycleLocked(currentTime, connection, true /*notify*/);
            }
            return;
        }

        // Re-enqueue the event on the wait queue.
        connection->outboundQueue.dequeue(dispatchEntry);
        traceOutboundQueueLengthLocked(connection);
        connection->waitQueue.enqueueAtTail(dispatchEntry);
        traceWaitQueueLengthLocked(connection);
    }
}
```

#### startDispatchCycleLocked

```cpp
void InputDispatcher::startDispatchCycleLocked(nsecs_t currentTime,
        const sp<Connection>& connection) {
#if DEBUG_DISPATCH_CYCLE
    ALOGD("channel '%s' ~ startDispatchCycle",
            connection->getInputChannelName().c_str());
#endif

    while (connection->status == Connection::STATUS_NORMAL
            && !connection->outboundQueue.isEmpty()) {
        DispatchEntry* dispatchEntry = connection->outboundQueue.head;
        dispatchEntry->deliveryTime = currentTime;

        // Publish the event.
        status_t status;
        EventEntry* eventEntry = dispatchEntry->eventEntry;
        switch (eventEntry->type) {
        case EventEntry::TYPE_KEY: {
            KeyEntry* keyEntry = static_cast<KeyEntry*>(eventEntry);

            // Publish the key event.
            status = connection->inputPublisher.publishKeyEvent(dispatchEntry->seq,
                    keyEntry->deviceId, keyEntry->source,
                    dispatchEntry->resolvedAction, dispatchEntry->resolvedFlags,
                    keyEntry->keyCode, keyEntry->scanCode,
                    keyEntry->metaState, keyEntry->repeatCount, keyEntry->downTime,
                    keyEntry->eventTime);
            break;
        }

        case EventEntry::TYPE_MOTION: {
            MotionEntry* motionEntry = static_cast<MotionEntry*>(eventEntry);

            PointerCoords scaledCoords[MAX_POINTERS];
            const PointerCoords* usingCoords = motionEntry->pointerCoords;

            // Set the X and Y offset depending on the input source.
            float xOffset, yOffset;
            if ((motionEntry->source & AINPUT_SOURCE_CLASS_POINTER)
                    && !(dispatchEntry->targetFlags & InputTarget::FLAG_ZERO_COORDS)) {
                float scaleFactor = dispatchEntry->scaleFactor;
                xOffset = dispatchEntry->xOffset * scaleFactor;
                yOffset = dispatchEntry->yOffset * scaleFactor;
                if (scaleFactor != 1.0f) {
                    for (uint32_t i = 0; i < motionEntry->pointerCount; i++) {
                        scaledCoords[i] = motionEntry->pointerCoords[i];
                        scaledCoords[i].scale(scaleFactor);
                    }
                    usingCoords = scaledCoords;
                }
            } else {
                xOffset = 0.0f;
                yOffset = 0.0f;

                // We don't want the dispatch target to know.
                if (dispatchEntry->targetFlags & InputTarget::FLAG_ZERO_COORDS) {
                    for (uint32_t i = 0; i < motionEntry->pointerCount; i++) {
                        scaledCoords[i].clear();
                    }
                    usingCoords = scaledCoords;
                }
            }

            // Publish the motion event.
            status = connection->inputPublisher.publishMotionEvent(dispatchEntry->seq,
                    motionEntry->deviceId, motionEntry->source, motionEntry->displayId,
                    dispatchEntry->resolvedAction, motionEntry->actionButton,
                    dispatchEntry->resolvedFlags, motionEntry->edgeFlags,
                    motionEntry->metaState, motionEntry->buttonState,
                    xOffset, yOffset, motionEntry->xPrecision, motionEntry->yPrecision,
                    motionEntry->downTime, motionEntry->eventTime,
                    motionEntry->pointerCount, motionEntry->pointerProperties,
                    usingCoords);
            break;
        }

        default:
            ALOG_ASSERT(false);
            return;
        }

        // Check the result.
        if (status) {
            if (status == WOULD_BLOCK) {
                if (connection->waitQueue.isEmpty()) {
                    ALOGE("channel '%s' ~ Could not publish event because the pipe is full. "
                            "This is unexpected because the wait queue is empty, so the pipe "
                            "should be empty and we shouldn't have any problems writing an "
                            "event to it, status=%d", connection->getInputChannelName().c_str(),
                            status);
                    abortBrokenDispatchCycleLocked(currentTime, connection, true /*notify*/);
                } else {
                    // Pipe is full and we are waiting for the app to finish process some events
                    // before sending more events to it.
#if DEBUG_DISPATCH_CYCLE
                    ALOGD("channel '%s' ~ Could not publish event because the pipe is full, "
                            "waiting for the application to catch up",
                            connection->getInputChannelName().c_str());
#endif
                    connection->inputPublisherBlocked = true;
                }
            } else {
                ALOGE("channel '%s' ~ Could not publish event due to an unexpected error, "
                        "status=%d", connection->getInputChannelName().c_str(), status);
                abortBrokenDispatchCycleLocked(currentTime, connection, true /*notify*/);
            }
            return;
        }

        // Re-enqueue the event on the wait queue.
        connection->outboundQueue.dequeue(dispatchEntry);
        traceOutboundQueueLengthLocked(connection);
        connection->waitQueue.enqueueAtTail(dispatchEntry);
        traceWaitQueueLengthLocked(connection);
    }
}
```

#### handleReceiveCallback

```cpp
int InputDispatcher::handleReceiveCallback(int fd, int events, void* data) {
    InputDispatcher* d = static_cast<InputDispatcher*>(data);

    { // acquire lock
        AutoMutex _l(d->mLock);

        ssize_t connectionIndex = d->mConnectionsByFd.indexOfKey(fd);
        if (connectionIndex < 0) {
            ALOGE("Received spurious receive callback for unknown input channel.  "
                    "fd=%d, events=0x%x", fd, events);
            return 0; // remove the callback
        }

        bool notify;
        sp<Connection> connection = d->mConnectionsByFd.valueAt(connectionIndex);
        if (!(events & (ALOOPER_EVENT_ERROR | ALOOPER_EVENT_HANGUP))) {
            if (!(events & ALOOPER_EVENT_INPUT)) {
                ALOGW("channel '%s' ~ Received spurious callback for unhandled poll event.  "
                        "events=0x%x", connection->getInputChannelName().c_str(), events);
                return 1;
            }

            nsecs_t currentTime = now();
            bool gotOne = false;
            status_t status;
            for (;;) {
                uint32_t seq;
                bool handled;
                status = connection->inputPublisher.receiveFinishedSignal(&seq, &handled);
                if (status) {
                    break;
                }
                d->finishDispatchCycleLocked(currentTime, connection, seq, handled);
                gotOne = true;
            }
            if (gotOne) {
                d->runCommandsLockedInterruptible();
                if (status == WOULD_BLOCK) {
                    return 1;
                }
            }

            notify = status != DEAD_OBJECT || !connection->monitor;
            if (notify) {
                ALOGE("channel '%s' ~ Failed to receive finished signal.  status=%d",
                        connection->getInputChannelName().c_str(), status);
            }
        } else {
            // Monitor channels are never explicitly unregistered.
            // We do it automatically when the remote endpoint is closed so don't warn
            // about them.
            notify = !connection->monitor;
            if (notify) {
                ALOGW("channel '%s' ~ Consumer closed input channel or an error occurred.  "
                        "events=0x%x", connection->getInputChannelName().c_str(), events);
            }
        }

        // Unregister the channel.
        d->unregisterInputChannelLocked(connection->inputChannel, notify);
        return 0; // remove the callback
    } // release lock
}
```


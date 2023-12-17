---
title: Android的Service
categories:
  - Android
date: 2019-12-05 10:53:36
updated: 2019-12-05 10:53:36
tags: 
  - Android
  - Service
---

Service 是安卓的四大组件之一，其官方的介绍  [https://developer.android.com/guide/components/services](https://developer.android.com/guide/components/services) 上有一个应用上的介绍。但是，我想要的是从代码层面来看一下一个服务，是如何启动，如何提供服务的呢。

<!--more-->

# 服务状态

根据官方文档，服务会有两种状态：

- 启动状态：以无限期运行
- 绑定状态：当应用组件通过调用 `bindService()` 绑定到服务时，服务即处于*绑定*状态。绑定服务会提供客户端-服务器接口，以便组件与服务进行交互、发送请求、接收结果，甚至是利用进程间通信 (IPC) 跨进程执行这些操作。仅当与另一个应用组件绑定时，绑定服务才会运行。多个组件可同时绑定到该服务，但全部取消绑定后，该服务即会被销毁。

> 无论服务是处于启动状态还是绑定状态（或同时处于这两种状态），任何应用组件均可像使用 Activity 那样，通过调用 `Intent` 来使用服务（即使此服务来自另一应用）。不过，您可以通过清单文件将服务声明为*私有*服务，并阻止其他应用访问该服务。[使用清单文件声明服务](https://developer.android.com/guide/components/services#Declaring)部分将对此做更详尽的阐述。
>
> **注意：**服务在其托管进程的主线程中运行，它既**不**创建自己的线程，也**不**在单独的进程中运行（除非另行指定）。如果服务将执行任何 CPU 密集型工作或阻止性操作（例如 MP3 播放或联网），则应通过在服务内创建新线程来完成这项工作。通过使用单独的线程，您可以降低发生“应用无响应”(ANR) 错误的风险，而应用的主线程仍可继续专注于运行用户与 Activity 之间的交互。

服务处于什么状态，在于其是否实现了一组回调方法：

- `onStartCommand()`（让组件启动服务）当另一个组件（如 Activity）请求启动服务时，系统会通过调用 `startService()` 来调用此方法。执行此方法时，服务即会启动并可在后台无限期运行。如果您实现此方法，则在服务工作完成后，您需负责通过调用 `stopSelf()` 或 `stopService()` 来停止服务。（如果您只想提供绑定，则无需实现此方法。）
-  `onBind()`（实现服务绑定）。当另一个组件想要与服务绑定（例如执行 RPC）时，系统会通过调用 `bindService()` 来调用此方法。在此方法的实现中，您必须通过返回 `IBinder` 提供一个接口，以供客户端用来与服务进行通信。请务必实现此方法；但是，如果您并不希望允许绑定，则应返回 null。

其他回调：

- `onCreate()` 系统会（在调用 `onStartCommand()` 或 `onBind()` 之前）调用此方法来执行一次性设置程序。如果服务已在运行，则不会调用此方法。
- `onDestroy()` 当不再使用服务且准备将其销毁时，系统会调用此方法。服务应通过实现此方法来清理任何资源，如线程、注册的侦听器、接收器等。这是服务接收的最后一个调用。

任何一个 Service 都是类  **android.app.Service** 的子类，所以我们可以从这个方面来看一下。任何 **启动** 服务，都是由另外一个组件调用  `startService()` 方法来实现的。比如我们在 Activity 里面启动一个服务。

```java
Intent intent = new Intent(this, HelloService.class);
startService(intent);
```

我们知道 Activity 继承于 Context ，而 Context 的实现在  ContextImpl 中，

# ContextImpl

在这里需要注意 `mMainThread.getApplicationThread()`  这个代表了什么呢，这个是 IApplicationThread.Stub 的子类，ApplicationThread。而 mMainThread 代表的是 ActivityThread。

- ActivityThread 负责管理应用进程内主线程的执行，调度和执行 activities，广播 和其他 AMS （Activity Manager Service ）请求的操作。此线程也是运行在 SystemServer 内。
- ApplicationThread 从表示上来看，其并不是一个线程对象，其更像是一个桥梁，用来将消息发往 ActivityThread 的 Handler 去。

## startService

```java
    @Override
    public ComponentName startService(Intent service) {
        warnIfCallingFromSystemProcess(); //如果是系统进程直接调用记录警告日志
        return startServiceCommon(service, false, mUser); // 不非前台服务启动
    }
    
    private ComponentName startServiceCommon(Intent service, boolean requireForeground,
            UserHandle user) {
        try {
            validateServiceIntent(service); // Service 在高版本上，必须是显式指定的，低版本上会记录警告日志
            service.prepareToLeaveProcess(this);
         	// 启动后会返回一个 ComponentName。
          // 利用 AMS 进行启动的
            ComponentName cn = ActivityManager.getService().startService(
                mMainThread.getApplicationThread(), service, service.resolveTypeIfNeeded(
                            getContentResolver()), requireForeground,
                            getOpPackageName(), user.getIdentifier());
            if (cn != null) {
                if (cn.getPackageName().equals("!")) {
                    throw new SecurityException(
                            "Not allowed to start service " + service
                            + " without permission " + cn.getClassName());
                } else if (cn.getPackageName().equals("!!")) {
                    throw new SecurityException(
                            "Unable to start service " + service
                            + ": " + cn.getClassName());
                } else if (cn.getPackageName().equals("?")) {
                    throw new IllegalStateException(
                            "Not allowed to start service " + service + ": " + cn.getClassName());
                }
            }
            return cn;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

```

很不意外的，走到了 ActivityManager 来。最终是将 ActivityManagerService 的 Binder 引入了进来。

# ActivityManagerService

这个使用 ActiveServices 来启动服务。我们知道 ，AMS 实际上是运行在 system-server 中的服务，所以这里面的 ActiveServices 代表了所有活跃的服务。

## startService

```java
    @Override
    public ComponentName startService(IApplicationThread caller, Intent service,
            String resolvedType, boolean requireForeground, String callingPackage, int userId)
            throws TransactionTooLargeException {
        enforceNotIsolatedCaller("startService");
        // Refuse possible leaked file descriptors
        if (service != null && service.hasFileDescriptors() == true) {
            throw new IllegalArgumentException("File descriptors passed in Intent");
        }

        if (callingPackage == null) {
            throw new IllegalArgumentException("callingPackage cannot be null");
        }

        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE,
                "*** startService: " + service + " type=" + resolvedType + " fg=" + requireForeground);
        synchronized(this) {
          // 获取调用进程 pid 和 uid 
          // 然后 userId 是存储在 userHandler 里面的 整型 id 数。
            final int callingPid = Binder.getCallingPid();
            final int callingUid = Binder.getCallingUid();
            final long origId = Binder.clearCallingIdentity();
            ComponentName res;
            try {
                res = mServices.startServiceLocked(caller, service,
                        resolvedType, callingPid, callingUid,
                        requireForeground, callingPackage, userId);
            } finally {
                Binder.restoreCallingIdentity(origId);
            }
            return res;
        }
    }
```

# ActiveServices

## startServiceLocked

这里面做的工作太多，我们简要说明就行了，首先通过 IApplicationThread caller 从 AMS 获取到对应的 APP 的进程记录。来决定是否需要进行前台启动。

接着通过  retrieveServiceLocked 来查看一下这个服务是否已经启动，如果没有启动，那么就会构建一个服务。然后调用 startServiceInnerLocked 进行下一步动作。

```java
    ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
            int callingPid, int callingUid, boolean fgRequired, String callingPackage, final int userId)
            throws TransactionTooLargeException {
        if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "startService: " + service
                + " type=" + resolvedType + " args=" + service.getExtras());

        final boolean callerFg;
      	// 调用方 app 线程不为 null
        if (caller != null) {
          	// 获取进程记录
            final ProcessRecord callerApp = mAm.getRecordForAppLocked(caller);
            if (callerApp == null) {
                throw new SecurityException(
                        "Unable to find app for caller " + caller
                        + " (pid=" + callingPid
                        + ") when starting service " + service);
            }
          	// 确定一下调用 app 是否是在前台的
            callerFg = callerApp.setSchedGroup != ProcessList.SCHED_GROUP_BACKGROUND;
        } else {
            callerFg = true;
        }
				// 查找 是否有已经启动的 Service 存在
        ServiceLookupResult res =
            retrieveServiceLocked(service, resolvedType, callingPackage,
                    callingPid, callingUid, userId, true, callerFg, false, false);
        if (res == null) {
            return null;
        }
        if (res.record == null) {
            return new ComponentName("!", res.permission != null
                    ? res.permission : "private to package");
        }

        ServiceRecord r = res.record;

        if (!mAm.mUserController.exists(r.userId)) {
            Slog.w(TAG, "Trying to start service with non-existent user! " + r.userId);
            return null;
        }

        // If we're starting indirectly (e.g. from PendingIntent), figure out whether
        // we're launching into an app in a background state.  This keys off of the same
        // idleness state tracking as e.g. O+ background service start policy.
        final boolean bgLaunch = !mAm.isUidActiveLocked(r.appInfo.uid);

        // If the app has strict background restrictions, we treat any bg service
        // start analogously to the legacy-app forced-restrictions case, regardless
        // of its target SDK version.
        boolean forcedStandby = false;
        if (bgLaunch && appRestrictedAnyInBackground(r.appInfo.uid, r.packageName)) {
            if (DEBUG_FOREGROUND_SERVICE) {
                Slog.d(TAG, "Forcing bg-only service start only for " + r.shortName
                        + " : bgLaunch=" + bgLaunch + " callerFg=" + callerFg);
            }
            forcedStandby = true;
        }

        // If this is a direct-to-foreground start, make sure it is allowed as per the app op.
        boolean forceSilentAbort = false;
        if (fgRequired) {
            final int mode = mAm.mAppOpsService.checkOperation(
                    AppOpsManager.OP_START_FOREGROUND, r.appInfo.uid, r.packageName);
            switch (mode) {
                case AppOpsManager.MODE_ALLOWED:
                case AppOpsManager.MODE_DEFAULT:
                    // All okay.
                    break;
                case AppOpsManager.MODE_IGNORED:
                    // Not allowed, fall back to normal start service, failing siliently
                    // if background check restricts that.
                    Slog.w(TAG, "startForegroundService not allowed due to app op: service "
                            + service + " to " + r.name.flattenToShortString()
                            + " from pid=" + callingPid + " uid=" + callingUid
                            + " pkg=" + callingPackage);
                    fgRequired = false;
                    forceSilentAbort = true;
                    break;
                default:
                    return new ComponentName("!!", "foreground not allowed as per app op");
            }
        }

        // If this isn't a direct-to-foreground start, check our ability to kick off an
        // arbitrary service
        if (forcedStandby || (!r.startRequested && !fgRequired)) {
            // Before going further -- if this app is not allowed to start services in the
            // background, then at this point we aren't going to let it period.
            final int allowed = mAm.getAppStartModeLocked(r.appInfo.uid, r.packageName,
                    r.appInfo.targetSdkVersion, callingPid, false, false, forcedStandby);
            if (allowed != ActivityManager.APP_START_MODE_NORMAL) {
                Slog.w(TAG, "Background start not allowed: service "
                        + service + " to " + r.name.flattenToShortString()
                        + " from pid=" + callingPid + " uid=" + callingUid
                        + " pkg=" + callingPackage + " startFg?=" + fgRequired);
                if (allowed == ActivityManager.APP_START_MODE_DELAYED || forceSilentAbort) {
                    // In this case we are silently disabling the app, to disrupt as
                    // little as possible existing apps.
                    return null;
                }
                if (forcedStandby) {
                    // This is an O+ app, but we might be here because the user has placed
                    // it under strict background restrictions.  Don't punish the app if it's
                    // trying to do the right thing but we're denying it for that reason.
                    if (fgRequired) {
                        if (DEBUG_BACKGROUND_CHECK) {
                            Slog.v(TAG, "Silently dropping foreground service launch due to FAS");
                        }
                        return null;
                    }
                }
                // This app knows it is in the new model where this operation is not
                // allowed, so tell it what has happened.
                UidRecord uidRec = mAm.mActiveUids.get(r.appInfo.uid);
                return new ComponentName("?", "app is in background uid " + uidRec);
            }
        }

        // At this point we've applied allowed-to-start policy based on whether this was
        // an ordinary startService() or a startForegroundService().  Now, only require that
        // the app follow through on the startForegroundService() -> startForeground()
        // contract if it actually targets O+.
        if (r.appInfo.targetSdkVersion < Build.VERSION_CODES.O && fgRequired) {
            if (DEBUG_BACKGROUND_CHECK || DEBUG_FOREGROUND_SERVICE) {
                Slog.i(TAG, "startForegroundService() but host targets "
                        + r.appInfo.targetSdkVersion + " - not requiring startForeground()");
            }
            fgRequired = false;
        }

        NeededUriGrants neededGrants = mAm.checkGrantUriPermissionFromIntentLocked(
                callingUid, r.packageName, service, service.getFlags(), null, r.userId);

        // If permissions need a review before any of the app components can run,
        // we do not start the service and launch a review activity if the calling app
        // is in the foreground passing it a pending intent to start the service when
        // review is completed.
        if (mAm.mPermissionReviewRequired) {
            // XXX This is not dealing with fgRequired!
            if (!requestStartTargetPermissionsReviewIfNeededLocked(r, callingPackage,
                    callingUid, service, callerFg, userId)) {
                return null;
            }
        }

        if (unscheduleServiceRestartLocked(r, callingUid, false)) {
            if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "START SERVICE WHILE RESTART PENDING: " + r);
        }
        r.lastActivity = SystemClock.uptimeMillis();
        r.startRequested = true;
        r.delayedStop = false;
        r.fgRequired = fgRequired;
        r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                service, neededGrants, callingUid));

        if (fgRequired) {
            // We are now effectively running a foreground service.
            mAm.mAppOpsService.startOperation(AppOpsManager.getToken(mAm.mAppOpsService),
                    AppOpsManager.OP_START_FOREGROUND, r.appInfo.uid, r.packageName, true);
        }

        final ServiceMap smap = getServiceMapLocked(r.userId);
        boolean addToStarting = false;
        if (!callerFg && !fgRequired && r.app == null
                && mAm.mUserController.hasStartedUserState(r.userId)) {
            ProcessRecord proc = mAm.getProcessRecordLocked(r.processName, r.appInfo.uid, false);
            if (proc == null || proc.curProcState > ActivityManager.PROCESS_STATE_RECEIVER) {
                // If this is not coming from a foreground caller, then we may want
                // to delay the start if there are already other background services
                // that are starting.  This is to avoid process start spam when lots
                // of applications are all handling things like connectivity broadcasts.
                // We only do this for cached processes, because otherwise an application
                // can have assumptions about calling startService() for a service to run
                // in its own process, and for that process to not be killed before the
                // service is started.  This is especially the case for receivers, which
                // may start a service in onReceive() to do some additional work and have
                // initialized some global state as part of that.
                if (DEBUG_DELAYED_SERVICE) Slog.v(TAG_SERVICE, "Potential start delay of "
                        + r + " in " + proc);
                if (r.delayed) {
                    // This service is already scheduled for a delayed start; just leave
                    // it still waiting.
                    if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "Continuing to delay: " + r);
                    return r.name;
                }
                if (smap.mStartingBackground.size() >= mMaxStartingBackground) {
                    // Something else is starting, delay!
                    Slog.i(TAG_SERVICE, "Delaying start of: " + r);
                    smap.mDelayedStartList.add(r);
                    r.delayed = true;
                    return r.name;
                }
                if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "Not delaying: " + r);
                addToStarting = true;
            } else if (proc.curProcState >= ActivityManager.PROCESS_STATE_SERVICE) {
                // We slightly loosen when we will enqueue this new service as a background
                // starting service we are waiting for, to also include processes that are
                // currently running other services or receivers.
                addToStarting = true;
                if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE,
                        "Not delaying, but counting as bg: " + r);
            } else if (DEBUG_DELAYED_STARTS) {
                StringBuilder sb = new StringBuilder(128);
                sb.append("Not potential delay (state=").append(proc.curProcState)
                        .append(' ').append(proc.adjType);
                String reason = proc.makeAdjReason();
                if (reason != null) {
                    sb.append(' ');
                    sb.append(reason);
                }
                sb.append("): ");
                sb.append(r.toString());
                Slog.v(TAG_SERVICE, sb.toString());
            }
        } else if (DEBUG_DELAYED_STARTS) {
            if (callerFg || fgRequired) {
                Slog.v(TAG_SERVICE, "Not potential delay (callerFg=" + callerFg + " uid="
                        + callingUid + " pid=" + callingPid + " fgRequired=" + fgRequired + "): " + r);
            } else if (r.app != null) {
                Slog.v(TAG_SERVICE, "Not potential delay (cur app=" + r.app + "): " + r);
            } else {
                Slog.v(TAG_SERVICE,
                        "Not potential delay (user " + r.userId + " not started): " + r);
            }
        }

        ComponentName cmp = startServiceInnerLocked(smap, service, r, callerFg, addToStarting);
        return cmp;
    }

```

## retrieveServiceLocked

```java
    private ServiceLookupResult retrieveServiceLocked(Intent service,
            String resolvedType, String callingPackage, int callingPid, int callingUid, int userId,
            boolean createIfNeeded, boolean callingFromFg, boolean isBindExternal,
            boolean allowInstant) {
        ServiceRecord r = null;
        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "retrieveServiceLocked: " + service
                + " type=" + resolvedType + " callingUid=" + callingUid);

        userId = mAm.mUserController.handleIncomingUser(callingPid, callingUid, userId, false,
                ActivityManagerService.ALLOW_NON_FULL_IN_PROFILE, "service", null);

        ServiceMap smap = getServiceMapLocked(userId);
        final ComponentName comp = service.getComponent();
        if (comp != null) {
            r = smap.mServicesByName.get(comp);
            if (DEBUG_SERVICE && r != null) Slog.v(TAG_SERVICE, "Retrieved by component: " + r);
        }
        if (r == null && !isBindExternal) {
            Intent.FilterComparison filter = new Intent.FilterComparison(service);
            r = smap.mServicesByIntent.get(filter);
            if (DEBUG_SERVICE && r != null) Slog.v(TAG_SERVICE, "Retrieved by intent: " + r);
        }
        if (r != null && (r.serviceInfo.flags & ServiceInfo.FLAG_EXTERNAL_SERVICE) != 0
                && !callingPackage.equals(r.packageName)) {
            // If an external service is running within its own package, other packages
            // should not bind to that instance.
            r = null;
            if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Whoops, can't use existing external service");
        }
        if (r == null) {
            try {
                int flags = ActivityManagerService.STOCK_PM_FLAGS
                        | PackageManager.MATCH_DEBUG_TRIAGED_MISSING;
                if (allowInstant) {
                    flags |= PackageManager.MATCH_INSTANT;
                }
                // TODO: come back and remove this assumption to triage all services
                ResolveInfo rInfo = mAm.getPackageManagerInternalLocked().resolveService(service,
                        resolvedType, flags, userId, callingUid);
                ServiceInfo sInfo =
                    rInfo != null ? rInfo.serviceInfo : null;
                if (sInfo == null) {
                    Slog.w(TAG_SERVICE, "Unable to start service " + service + " U=" + userId +
                          ": not found");
                    return null;
                }
                ComponentName name = new ComponentName(
                        sInfo.applicationInfo.packageName, sInfo.name);
                if ((sInfo.flags & ServiceInfo.FLAG_EXTERNAL_SERVICE) != 0) {
                    if (isBindExternal) {
                        if (!sInfo.exported) {
                            throw new SecurityException("BIND_EXTERNAL_SERVICE failed, " + name +
                                    " is not exported");
                        }
                        if ((sInfo.flags & ServiceInfo.FLAG_ISOLATED_PROCESS) == 0) {
                            throw new SecurityException("BIND_EXTERNAL_SERVICE failed, " + name +
                                    " is not an isolatedProcess");
                        }
                        // Run the service under the calling package's application.
                        ApplicationInfo aInfo = AppGlobals.getPackageManager().getApplicationInfo(
                                callingPackage, ActivityManagerService.STOCK_PM_FLAGS, userId);
                        if (aInfo == null) {
                            throw new SecurityException("BIND_EXTERNAL_SERVICE failed, " +
                                    "could not resolve client package " + callingPackage);
                        }
                        sInfo = new ServiceInfo(sInfo);
                        sInfo.applicationInfo = new ApplicationInfo(sInfo.applicationInfo);
                        sInfo.applicationInfo.packageName = aInfo.packageName;
                        sInfo.applicationInfo.uid = aInfo.uid;
                        name = new ComponentName(aInfo.packageName, name.getClassName());
                        service.setComponent(name);
                    } else {
                        throw new SecurityException("BIND_EXTERNAL_SERVICE required for " +
                                name);
                    }
                } else if (isBindExternal) {
                    throw new SecurityException("BIND_EXTERNAL_SERVICE failed, " + name +
                            " is not an externalService");
                }
                if (userId > 0) {
                    if (mAm.isSingleton(sInfo.processName, sInfo.applicationInfo,
                            sInfo.name, sInfo.flags)
                            && mAm.isValidSingletonCall(callingUid, sInfo.applicationInfo.uid)) {
                        userId = 0;
                        smap = getServiceMapLocked(0);
                    }
                    sInfo = new ServiceInfo(sInfo);
                    sInfo.applicationInfo = mAm.getAppInfoForUser(sInfo.applicationInfo, userId);
                }
                r = smap.mServicesByName.get(name);
                if (DEBUG_SERVICE && r != null) Slog.v(TAG_SERVICE,
                        "Retrieved via pm by intent: " + r);
                if (r == null && createIfNeeded) {
                    final Intent.FilterComparison filter
                            = new Intent.FilterComparison(service.cloneFilter());
                    final ServiceRestarter res = new ServiceRestarter();
                    final BatteryStatsImpl.Uid.Pkg.Serv ss;
                    final BatteryStatsImpl stats = mAm.mBatteryStatsService.getActiveStatistics();
                    synchronized (stats) {
                        ss = stats.getServiceStatsLocked(
                                sInfo.applicationInfo.uid, sInfo.packageName,
                                sInfo.name);
                    }
                    r = new ServiceRecord(mAm, ss, name, filter, sInfo, callingFromFg, res);
                    res.setService(r);
                    smap.mServicesByName.put(name, r);
                    smap.mServicesByIntent.put(filter, r);

                    // Make sure this component isn't in the pending list.
                    for (int i=mPendingServices.size()-1; i>=0; i--) {
                        final ServiceRecord pr = mPendingServices.get(i);
                        if (pr.serviceInfo.applicationInfo.uid == sInfo.applicationInfo.uid
                                && pr.name.equals(name)) {
                            if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Remove pending: " + pr);
                            mPendingServices.remove(i);
                        }
                    }
                    if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Retrieve created new service: " + r);
                }
            } catch (RemoteException ex) {
                // pm is in same process, this will never happen.
            }
        }
        if (r != null) {
            if (mAm.checkComponentPermission(r.permission,
                    callingPid, callingUid, r.appInfo.uid, r.exported) != PERMISSION_GRANTED) {
                if (!r.exported) {
                    Slog.w(TAG, "Permission Denial: Accessing service " + r.name
                            + " from pid=" + callingPid
                            + ", uid=" + callingUid
                            + " that is not exported from uid " + r.appInfo.uid);
                    return new ServiceLookupResult(null, "not exported from uid "
                            + r.appInfo.uid);
                }
                Slog.w(TAG, "Permission Denial: Accessing service " + r.name
                        + " from pid=" + callingPid
                        + ", uid=" + callingUid
                        + " requires " + r.permission);
                return new ServiceLookupResult(null, r.permission);
            } else if (r.permission != null && callingPackage != null) {
                final int opCode = AppOpsManager.permissionToOpCode(r.permission);
                if (opCode != AppOpsManager.OP_NONE && mAm.mAppOpsService.noteOperation(
                        opCode, callingUid, callingPackage) != AppOpsManager.MODE_ALLOWED) {
                    Slog.w(TAG, "Appop Denial: Accessing service " + r.name
                            + " from pid=" + callingPid
                            + ", uid=" + callingUid
                            + " requires appop " + AppOpsManager.opToName(opCode));
                    return null;
                }
            }

            if (!mAm.mIntentFirewall.checkService(r.name, service, callingUid, callingPid,
                    resolvedType, r.appInfo)) {
                return null;
            }
            return new ServiceLookupResult(r, null);
        }
        return null;
    }

```



## startServiceInnerLocked

```java
    ComponentName startServiceInnerLocked(ServiceMap smap, Intent service, ServiceRecord r,
            boolean callerFg, boolean addToStarting) throws TransactionTooLargeException {
        ServiceState stracker = r.getTracker();
        if (stracker != null) {
            stracker.setStarted(true, mAm.mProcessStats.getMemFactorLocked(), r.lastActivity);
        }
        r.callStart = false;
        synchronized (r.stats.getBatteryStats()) {
            r.stats.startRunningLocked();
        }
        String error = bringUpServiceLocked(r, service.getFlags(), callerFg, false, false);
        if (error != null) {
            return new ComponentName("!!", error);
        }

        if (r.startRequested && addToStarting) {
            boolean first = smap.mStartingBackground.size() == 0;
            smap.mStartingBackground.add(r);
            r.startingBgTimeout = SystemClock.uptimeMillis() + mAm.mConstants.BG_START_TIMEOUT;
            if (DEBUG_DELAYED_SERVICE) {
                RuntimeException here = new RuntimeException("here");
                here.fillInStackTrace();
                Slog.v(TAG_SERVICE, "Starting background (first=" + first + "): " + r, here);
            } else if (DEBUG_DELAYED_STARTS) {
                Slog.v(TAG_SERVICE, "Starting background (first=" + first + "): " + r);
            }
            if (first) {
                smap.rescheduleDelayedStartsLocked();
            }
        } else if (callerFg || r.fgRequired) {
            smap.ensureNotStartingBackgroundLocked(r);
        }

        return r.name;
    }

```

## bringUpServiceLocked

```java
    private String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
            boolean whileRestarting, boolean permissionsReviewRequired)
            throws TransactionTooLargeException {
        //Slog.i(TAG, "Bring up service:");
        //r.dump("  ");
				// 如果 r 中的 app（代表了 r 运行在哪个进程，及 进程中的 线程不为空，表示是已经启动过了的，直接重新传参数过去即可 否则流程继续往下走
        if (r.app != null && r.app.thread != null) {
            sendServiceArgsLocked(r, execInFg, false);
            return null;
        }

        if (!whileRestarting && mRestartingServices.contains(r)) {
            // If waiting for a restart, then do nothing.
            return null;
        }

        if (DEBUG_SERVICE) {
            Slog.v(TAG_SERVICE, "Bringing up " + r + " " + r.intent + " fg=" + r.fgRequired);
        }

        // We are now bringing the service up, so no longer in the
        // restarting state.
        if (mRestartingServices.remove(r)) {
            clearRestartingIfNeededLocked(r);
        }

        // Make sure this service is no longer considered delayed, we are starting it now.
        if (r.delayed) {
            if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "REM FR DELAY LIST (bring up): " + r);
            getServiceMapLocked(r.userId).mDelayedStartList.remove(r);
            r.delayed = false;
        }

        // Make sure that the user who owns this service is started.  If not,
        // we don't want to allow it to run.
        if (!mAm.mUserController.hasStartedUserState(r.userId)) {
            String msg = "Unable to launch app "
                    + r.appInfo.packageName + "/"
                    + r.appInfo.uid + " for service "
                    + r.intent.getIntent() + ": user " + r.userId + " is stopped";
            Slog.w(TAG, msg);
            bringDownServiceLocked(r);
            return msg;
        }

        // Service is now being launched, its package can't be stopped.
        try {
            AppGlobals.getPackageManager().setPackageStoppedState(
                    r.packageName, false, r.userId);
        } catch (RemoteException e) {
        } catch (IllegalArgumentException e) {
            Slog.w(TAG, "Failed trying to unstop package "
                    + r.packageName + ": " + e);
        }

        final boolean isolated = (r.serviceInfo.flags&ServiceInfo.FLAG_ISOLATED_PROCESS) != 0;
        final String procName = r.processName;
        String hostingType = "service";
        ProcessRecord app;

      	// 非隔离启动，会在与 proName 同一个进程进行执行。
        if (!isolated) {
          	// 根据 packagename 和 uid 来获取执行中的进程信息
            app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);
            if (DEBUG_MU) Slog.v(TAG_MU, "bringUpServiceLocked: appInfo.uid=" + r.appInfo.uid
                        + " app=" + app);
          	// 如果 app 是启动的那就 启动好服务咯
            if (app != null && app.thread != null) {
                try {
                    app.addPackage(r.appInfo.packageName, r.appInfo.longVersionCode, mAm.mProcessStats);
                    realStartServiceLocked(r, app, execInFg);
                    return null;
                } catch (TransactionTooLargeException e) {
                    throw e;
                } catch (RemoteException e) {
                    Slog.w(TAG, "Exception when starting service " + r.shortName, e);
                }

                // If a dead object exception was thrown -- fall through to
                // restart the application.
            }
        } else {
            // If this service runs in an isolated process, then each time
            // we call startProcessLocked() we will get a new isolated
            // process, starting another process if we are currently waiting
            // for a previous process to come up.  To deal with this, we store
            // in the service any current isolated process it is running in or
            // waiting to have come up.
            app = r.isolatedProc;
            if (WebViewZygote.isMultiprocessEnabled()
                    && r.serviceInfo.packageName.equals(WebViewZygote.getPackageName())) {
                hostingType = "webview_service";
            }
        }

        // Not running -- get it started, and enqueue this service record
        // to be executed when the app comes up.
        if (app == null && !permissionsReviewRequired) {
            if ((app=mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                    hostingType, r.name, false, isolated, false)) == null) {
                String msg = "Unable to launch app "
                        + r.appInfo.packageName + "/"
                        + r.appInfo.uid + " for service "
                        + r.intent.getIntent() + ": process is bad";
                Slog.w(TAG, msg);
                bringDownServiceLocked(r);
                return msg;
            }
            if (isolated) {
                r.isolatedProc = app;
            }
        }

        if (r.fgRequired) {
            if (DEBUG_FOREGROUND_SERVICE) {
                Slog.v(TAG, "Whitelisting " + UserHandle.formatUid(r.appInfo.uid)
                        + " for fg-service launch");
            }
            mAm.tempWhitelistUidLocked(r.appInfo.uid,
                    SERVICE_START_FOREGROUND_TIMEOUT, "fg-service-launch");
        }

        if (!mPendingServices.contains(r)) {
            mPendingServices.add(r);
        }

        if (r.delayedStop) {
            // Oh and hey we've already been asked to stop!
            r.delayedStop = false;
            if (r.startRequested) {
                if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE,
                        "Applying delayed stop (in bring up): " + r);
                stopServiceLocked(r);
            }
        }

        return null;
    }

```

## realStartServiceLocked

这里会将逻辑交给 app.thread 也就是调用进程（线程）进行调度，建立好服务。

然后再发送参数过去，进行启动，触发调用  `onStartCommand`

```java
    private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {
        if (app.thread == null) {
            throw new RemoteException();
        }
        if (DEBUG_MU)
            Slog.v(TAG_MU, "realStartServiceLocked, ServiceRecord.uid = " + r.appInfo.uid
                    + ", ProcessRecord.uid = " + app.uid);
        r.app = app;
        r.restartTime = r.lastActivity = SystemClock.uptimeMillis();

        final boolean newService = app.services.add(r);
        bumpServiceExecutingLocked(r, execInFg, "create");
        mAm.updateLruProcessLocked(app, false, null);
        updateServiceForegroundLocked(r.app, /* oomAdj= */ false);
        mAm.updateOomAdjLocked();

        boolean created = false;
        try {
            if (LOG_SERVICE_START_STOP) {
                String nameTerm;
                int lastPeriod = r.shortName.lastIndexOf('.');
                nameTerm = lastPeriod >= 0 ? r.shortName.substring(lastPeriod) : r.shortName;
                EventLogTags.writeAmCreateService(
                        r.userId, System.identityHashCode(r), nameTerm, r.app.uid, r.app.pid);
            }
            synchronized (r.stats.getBatteryStats()) {
                r.stats.startLaunchedLocked();
            }
            mAm.notifyPackageUse(r.serviceInfo.packageName,
                                 PackageManager.NOTIFY_PACKAGE_USE_SERVICE);
            app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
            app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                    app.repProcState);
            r.postNotification();
            created = true;
        } catch (DeadObjectException e) {
            Slog.w(TAG, "Application dead when creating service " + r);
            mAm.appDiedLocked(app);
            throw e;
        } finally {
            if (!created) {
                // Keep the executeNesting count accurate.
                final boolean inDestroying = mDestroyingServices.contains(r);
                serviceDoneExecutingLocked(r, inDestroying, inDestroying);

                // Cleanup.
                if (newService) {
                    app.services.remove(r);
                    r.app = null;
                }

                // Retry.
                if (!inDestroying) {
                    scheduleServiceRestartLocked(r, false);
                }
            }
        }

        if (r.whitelistManager) {
            app.whitelistManager = true;
        }

        requestServiceBindingsLocked(r, execInFg);

        updateServiceClientActivitiesLocked(app, null, true);

        // If the service is in the started state, and there are no
        // pending arguments, then fake up one so its onStartCommand() will
        // be called.
        if (r.startRequested && r.callStart && r.pendingStarts.size() == 0) {
            r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                    null, null, 0));
        }

        sendServiceArgsLocked(r, execInFg, true);

        if (r.delayed) {
            if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "REM FR DELAY LIST (new proc): " + r);
            getServiceMapLocked(r.userId).mDelayedStartList.remove(r);
            r.delayed = false;
        }

        if (r.delayedStop) {
            // Oh and hey we've already been asked to stop!
            r.delayedStop = false;
            if (r.startRequested) {
                if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE,
                        "Applying delayed stop (from start): " + r);
                stopServiceLocked(r);
            }
        }
    }

```

## requestServiceBindingsLocked

## updateServiceClientActivitiesLocked

## sendServiceArgsLocked

```java
    private final void sendServiceArgsLocked(ServiceRecord r, boolean execInFg,
            boolean oomAdjusted) throws TransactionTooLargeException {
        final int N = r.pendingStarts.size();
        if (N == 0) {
            return;
        }

        ArrayList<ServiceStartArgs> args = new ArrayList<>();

        while (r.pendingStarts.size() > 0) {
            ServiceRecord.StartItem si = r.pendingStarts.remove(0);
            if (DEBUG_SERVICE) {
                Slog.v(TAG_SERVICE, "Sending arguments to: "
                        + r + " " + r.intent + " args=" + si.intent);
            }
            if (si.intent == null && N > 1) {
                // If somehow we got a dummy null intent in the middle,
                // then skip it.  DO NOT skip a null intent when it is
                // the only one in the list -- this is to support the
                // onStartCommand(null) case.
                continue;
            }
            si.deliveredTime = SystemClock.uptimeMillis();
            r.deliveredStarts.add(si);
            si.deliveryCount++;
            if (si.neededGrants != null) {
                mAm.grantUriPermissionUncheckedFromIntentLocked(si.neededGrants,
                        si.getUriPermissionsLocked());
            }
            mAm.grantEphemeralAccessLocked(r.userId, si.intent,
                    r.appInfo.uid, UserHandle.getAppId(si.callingId));
            bumpServiceExecutingLocked(r, execInFg, "start");
            if (!oomAdjusted) {
                oomAdjusted = true;
                mAm.updateOomAdjLocked(r.app, true);
            }
            if (r.fgRequired && !r.fgWaiting) {
                if (!r.isForeground) {
                    if (DEBUG_BACKGROUND_CHECK) {
                        Slog.i(TAG, "Launched service must call startForeground() within timeout: " + r);
                    }
                    scheduleServiceForegroundTransitionTimeoutLocked(r);
                } else {
                    if (DEBUG_BACKGROUND_CHECK) {
                        Slog.i(TAG, "Service already foreground; no new timeout: " + r);
                    }
                    r.fgRequired = false;
                }
            }
            int flags = 0;
            if (si.deliveryCount > 1) {
                flags |= Service.START_FLAG_RETRY;
            }
            if (si.doneExecutingCount > 0) {
                flags |= Service.START_FLAG_REDELIVERY;
            }
            args.add(new ServiceStartArgs(si.taskRemoved, si.id, flags, si.intent));
        }

        ParceledListSlice<ServiceStartArgs> slice = new ParceledListSlice<>(args);
        slice.setInlineCountLimit(4);
        Exception caughtException = null;
        try {
            r.app.thread.scheduleServiceArgs(r, slice);
        } catch (TransactionTooLargeException e) {
            if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Transaction too large for " + args.size()
                    + " args, first: " + args.get(0).args);
            Slog.w(TAG, "Failed delivering service starts", e);
            caughtException = e;
        } catch (RemoteException e) {
            // Remote process gone...  we'll let the normal cleanup take care of this.
            if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Crashed while sending args: " + r);
            Slog.w(TAG, "Failed delivering service starts", e);
            caughtException = e;
        } catch (Exception e) {
            Slog.w(TAG, "Unexpected exception", e);
            caughtException = e;
        }

        if (caughtException != null) {
            // Keep nesting count correct
            final boolean inDestroying = mDestroyingServices.contains(r);
            for (int i = 0; i < args.size(); i++) {
                serviceDoneExecutingLocked(r, inDestroying, inDestroying);
            }
            if (caughtException instanceof TransactionTooLargeException) {
                throw (TransactionTooLargeException)caughtException;
            }
        }
    }

```



# ApplicationThread

## scheduleCreateService

```java
        public final void scheduleCreateService(IBinder token,
                ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
            updateProcessState(processState, false);
            CreateServiceData s = new CreateServiceData();
            s.token = token;
            s.info = info;
            s.compatInfo = compatInfo;
            sendMessage(H.CREATE_SERVICE, s);
        }

```

## scheduleServiceArgs

```java
        public final void scheduleServiceArgs(IBinder token, ParceledListSlice args) {
            List<ServiceStartArgs> list = args.getList();

            for (int i = 0; i < list.size(); i++) {
                ServiceStartArgs ssa = list.get(i);
                ServiceArgsData s = new ServiceArgsData();
                s.token = token;
                s.taskRemoved = ssa.taskRemoved;
                s.startId = ssa.startId;
                s.flags = ssa.flags;
                s.args = ssa.args;
                sendMessage(H.SERVICE_ARGS, s);
            }
        }

```

## ActivityThread.H

```java
    void sendMessage(int what, Object obj) {
        sendMessage(what, obj, 0, 0, false);
    }

    private void sendMessage(int what, Object obj, int arg1) {
        sendMessage(what, obj, arg1, 0, false);
    }

    private void sendMessage(int what, Object obj, int arg1, int arg2) {
        sendMessage(what, obj, arg1, arg2, false);
    }

    private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
        if (DEBUG_MESSAGES) Slog.v(
            TAG, "SCHEDULE " + what + " " + mH.codeToString(what)
            + ": " + arg1 + " / " + obj);
        Message msg = Message.obtain();
        msg.what = what;
        msg.obj = obj;
        msg.arg1 = arg1;
        msg.arg2 = arg2;
        if (async) {
            msg.setAsynchronous(true);
        }
        mH.sendMessage(msg);
    }

```

### handleMessage

```java
        public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
            switch (msg.what) {
                case CREATE_SERVICE:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, ("serviceCreate: " + String.valueOf(msg.obj)));
                    handleCreateService((CreateServiceData)msg.obj);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
                 case SERVICE_ARGS:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, ("serviceStart: " + String.valueOf(msg.obj)));
                    handleServiceArgs((ServiceArgsData)msg.obj);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
               
            }
            Object obj = msg.obj;
            if (obj instanceof SomeArgs) {
                ((SomeArgs) obj).recycle();
            }
            if (DEBUG_MESSAGES) Slog.v(TAG, "<<< done: " + codeToString(msg.what));
        }

```

### handleCreateService

这里才真正的构建服务。

1. 通过 classloader 来初始化类。
2. 设置上下文
3. 设置 ActivityThread 和 AMS

```java
private void handleCreateService(CreateServiceData data) {
    // If we are getting ready to gc after going to the background, well
    // we are back active so skip it.
    unscheduleGcIdler();

    LoadedApk packageInfo = getPackageInfoNoCheck(
            data.info.applicationInfo, data.compatInfo);
    Service service = null;
    try {
        java.lang.ClassLoader cl = packageInfo.getClassLoader();
        service = packageInfo.getAppFactory()
                .instantiateService(cl, data.info.name, data.intent);
    } catch (Exception e) {
        if (!mInstrumentation.onException(service, e)) {
            throw new RuntimeException(
                "Unable to instantiate service " + data.info.name
                + ": " + e.toString(), e);
        }
    }

    try {
        if (localLOGV) Slog.v(TAG, "Creating service " + data.info.name);
				// 主要的启动逻辑就在这里了
        ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
        context.setOuterContext(service);

        Application app = packageInfo.makeApplication(false, mInstrumentation);
        service.attach(context, this, data.info.name, data.token, app,
                ActivityManager.getService());
      	// 调用  onCreate
        service.onCreate();
      	// 放入本地服务列表中
        mServices.put(data.token, service);
        try {
          	// 通知 AMS 
            ActivityManager.getService().serviceDoneExecuting(
                    data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    } catch (Exception e) {
        if (!mInstrumentation.onException(service, e)) {
            throw new RuntimeException(
                "Unable to create service " + data.info.name
                + ": " + e.toString(), e);
        }
    }
}
```

### handleServiceArgs

```java
    private void handleServiceArgs(ServiceArgsData data) {
        Service s = mServices.get(data.token);
        if (s != null) {
            try {
                if (data.args != null) {
                    data.args.setExtrasClassLoader(s.getClassLoader());
                    data.args.prepareToEnterProcess();
                }
                int res;
                if (!data.taskRemoved) {
                    res = s.onStartCommand(data.args, data.flags, data.startId);
                } else {
                    s.onTaskRemoved(data.args);
                    res = Service.START_TASK_REMOVED_COMPLETE;
                }

                QueuedWork.waitToFinish();

                try {
                    ActivityManager.getService().serviceDoneExecuting(
                            data.token, SERVICE_DONE_EXECUTING_START, data.startId, res);
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
                ensureJitEnabled();
            } catch (Exception e) {
                if (!mInstrumentation.onException(s, e)) {
                    throw new RuntimeException(
                            "Unable to start service " + s
                            + " with " + data.args + ": " + e.toString(), e);
                }
            }
        }
    }

```

# ActivityManagerService

## serviceDoneExecuting

```java
    public void serviceDoneExecuting(IBinder token, int type, int startId, int res) {
        synchronized(this) {
            if (!(token instanceof ServiceRecord)) {
                Slog.e(TAG, "serviceDoneExecuting: Invalid service token=" + token);
                throw new IllegalArgumentException("Invalid service token");
            }
            mServices.serviceDoneExecutingLocked((ServiceRecord)token, type, startId, res);
        }
    }

```

# ActiveServices

## serviceDoneExecutingLocked

```java
    void serviceDoneExecutingLocked(ServiceRecord r, int type, int startId, int res) {
        boolean inDestroying = mDestroyingServices.contains(r);
        if (r != null) {
            if (type == ActivityThread.SERVICE_DONE_EXECUTING_START) {
                // This is a call from a service start...  take care of
                // book-keeping.
                r.callStart = true;
                switch (res) {
                    case Service.START_STICKY_COMPATIBILITY:
                    case Service.START_STICKY: {
                        // We are done with the associated start arguments.
                        r.findDeliveredStart(startId, false, true);
                        // Don't stop if killed.
                        r.stopIfKilled = false;
                        break;
                    }
                    case Service.START_NOT_STICKY: {
                        // We are done with the associated start arguments.
                        r.findDeliveredStart(startId, false, true);
                        if (r.getLastStartId() == startId) {
                            // There is no more work, and this service
                            // doesn't want to hang around if killed.
                            r.stopIfKilled = true;
                        }
                        break;
                    }
                    case Service.START_REDELIVER_INTENT: {
                        // We'll keep this item until they explicitly
                        // call stop for it, but keep track of the fact
                        // that it was delivered.
                        ServiceRecord.StartItem si = r.findDeliveredStart(startId, false, false);
                        if (si != null) {
                            si.deliveryCount = 0;
                            si.doneExecutingCount++;
                            // Don't stop if killed.
                            r.stopIfKilled = true;
                        }
                        break;
                    }
                    case Service.START_TASK_REMOVED_COMPLETE: {
                        // Special processing for onTaskRemoved().  Don't
                        // impact normal onStartCommand() processing.
                        r.findDeliveredStart(startId, true, true);
                        break;
                    }
                    default:
                        throw new IllegalArgumentException(
                                "Unknown service start result: " + res);
                }
                if (res == Service.START_STICKY_COMPATIBILITY) {
                    r.callStart = false;
                }
            } else if (type == ActivityThread.SERVICE_DONE_EXECUTING_STOP) {
                // This is the final call from destroying the service...  we should
                // actually be getting rid of the service at this point.  Do some
                // validation of its state, and ensure it will be fully removed.
                if (!inDestroying) {
                    // Not sure what else to do with this...  if it is not actually in the
                    // destroying list, we don't need to make sure to remove it from it.
                    // If the app is null, then it was probably removed because the process died,
                    // otherwise wtf
                    if (r.app != null) {
                        Slog.w(TAG, "Service done with onDestroy, but not inDestroying: "
                                + r + ", app=" + r.app);
                    }
                } else if (r.executeNesting != 1) {
                    Slog.w(TAG, "Service done with onDestroy, but executeNesting="
                            + r.executeNesting + ": " + r);
                    // Fake it to keep from ANR due to orphaned entry.
                    r.executeNesting = 1;
                }
            }
            final long origId = Binder.clearCallingIdentity();
            serviceDoneExecutingLocked(r, inDestroying, inDestroying);
            Binder.restoreCallingIdentity(origId);
        } else {
            Slog.w(TAG, "Done executing unknown service from pid "
                    + Binder.getCallingPid());
        }
    }

```

## serviceDoneExecutingLocked

```java
    private void serviceDoneExecutingLocked(ServiceRecord r, boolean inDestroying,
            boolean finishing) {
        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "<<< DONE EXECUTING " + r
                + ": nesting=" + r.executeNesting
                + ", inDestroying=" + inDestroying + ", app=" + r.app);
        else if (DEBUG_SERVICE_EXECUTING) Slog.v(TAG_SERVICE_EXECUTING,
                "<<< DONE EXECUTING " + r.shortName);
        r.executeNesting--;
        if (r.executeNesting <= 0) {
            if (r.app != null) {
                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE,
                        "Nesting at 0 of " + r.shortName);
                r.app.execServicesFg = false;
                r.app.executingServices.remove(r);
                if (r.app.executingServices.size() == 0) {
                    if (DEBUG_SERVICE || DEBUG_SERVICE_EXECUTING) Slog.v(TAG_SERVICE_EXECUTING,
                            "No more executingServices of " + r.shortName);
                    mAm.mHandler.removeMessages(ActivityManagerService.SERVICE_TIMEOUT_MSG, r.app);
                } else if (r.executeFg) {
                    // Need to re-evaluate whether the app still needs to be in the foreground.
                    for (int i=r.app.executingServices.size()-1; i>=0; i--) {
                        if (r.app.executingServices.valueAt(i).executeFg) {
                            r.app.execServicesFg = true;
                            break;
                        }
                    }
                }
                if (inDestroying) {
                    if (DEBUG_SERVICE) Slog.v(TAG_SERVICE,
                            "doneExecuting remove destroying " + r);
                    mDestroyingServices.remove(r);
                    r.bindings.clear();
                }
                mAm.updateOomAdjLocked(r.app, true);
            }
            r.executeFg = false;
            if (r.tracker != null) {
                r.tracker.setExecuting(false, mAm.mProcessStats.getMemFactorLocked(),
                        SystemClock.uptimeMillis());
                if (finishing) {
                    r.tracker.clearCurrentOwner(r, false);
                    r.tracker = null;
                }
            }
            if (finishing) {
                if (r.app != null && !r.app.persistent) {
                    r.app.services.remove(r);
                    if (r.whitelistManager) {
                        updateWhitelistManagerLocked(r.app);
                    }
                }
                r.app = null;
            }
        }
    }

```



# 总结

1. 若我们在 Activity 里面进行启动服务，那么会通过 AMS 的 binder 将消息发送过去。
2. AMS 通过其中的 ActiveServices 来进行启动检查。
3. 将服务参数传递给调用进程的 ApplicationThread 进行调度。
4. ApplicationThread 将消息发给 ActivityThread。
5. 告知 AMS 已经构建服务完毕。AMS 执行了一系列的操作后，会再次往 ApplicationThread  发送启动消息。
6. ActivityThread 由其调用 onStartCommand 回调进行启动逻辑，完毕后告知 AMS。
7. 启动完毕。
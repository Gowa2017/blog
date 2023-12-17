---
title: Android的Launcher启动流程代码
categories:
  - Android
date: 2019-12-06 00:56:26
updated: 2019-12-06 00:56:26
tags: 
  - Android
---

在 {% post_link Android系统的启动过程 Android系统的启动过程 %} 中我们已经知道了 SystemServer 怎么启动的，其会启动很多的系统服务。包括 AMS WMS IMS 等。在启动这些服务过程中，其还会让 AMS 启动我们的主界面。

<!--more-->



# SystemServer.startOtherService

在这里会给 AMS 构造一个回调：

```java
        mActivityManagerService.systemReady(() -> {
            Slog.i(TAG, "Making services ready");
            traceBeginAndSlog("StartActivityManagerReadyPhase");
            mSystemServiceManager.startBootPhase(
                    SystemService.PHASE_ACTIVITY_MANAGER_READY);
            traceEnd();
            traceBeginAndSlog("StartObservingNativeCrashes");
            try {
                mActivityManagerService.startObservingNativeCrashes();
            } catch (Throwable e) {
                reportWtf("observing native crashes", e);
            }
            traceEnd();
						....
        }
```

那么，构造回调的时候，实际上就会进行启动我们的主界面操作了。



# ActivityManagerService

## systemReady

```java
            mStackSupervisor.resumeFocusedStackTopActivityLocked();
            mUserController.sendUserSwitchBroadcasts(-1, currentUserId);

```

# ActivityStackSupervisor

ActivityStackSupervisor 是什么？其就是一个 Activty 栈的管理器。

## resumeFocusedStackTopActivityLocked

这个方法，将最顶层的 Activity 给获取焦点，放在最顶层去。我们启动的时候，栈是空的，所以会执行一个相对来说比较清晰的方法。

```java
    boolean resumeFocusedStackTopActivityLocked() {
        return resumeFocusedStackTopActivityLocked(null, null, null);
    }


boolean resumeFocusedStackTopActivityLocked(
        ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {

    if (!readyToResume()) {
        return false;
    }

    if (targetStack != null && isFocusedStack(targetStack)) {
        return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
    }

    final ActivityRecord r = mFocusedStack.topRunningActivityLocked();
    if (r == null || !r.isState(RESUMED)) {
        mFocusedStack.resumeTopActivityUncheckedLocked(null, null);
    } else if (r.isState(RESUMED)) {
        // Kick off any lingering app transitions form the MoveTaskToFront operation.
        mFocusedStack.executeAppTransition(targetOptions);
    }

    return false;
}
```

# ActivityStack

## resumeTopActivityUncheckedLocked

```java
    boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
        if (mStackSupervisor.inResumeTopActivity) {
            // Don't even start recursing.
            return false;
        }

        boolean result = false;
        try {
            // Protect against recursion.
            mStackSupervisor.inResumeTopActivity = true;
            result = resumeTopActivityInnerLocked(prev, options);

            // When resuming the top activity, it may be necessary to pause the top activity (for
            // example, returning to the lock screen. We suppress the normal pause logic in
            // {@link #resumeTopActivityUncheckedLocked}, since the top activity is resumed at the
            // end. We call the {@link ActivityStackSupervisor#checkReadyForSleepLocked} again here
            // to ensure any necessary pause logic occurs. In the case where the Activity will be
            // shown regardless of the lock screen, the call to
            // {@link ActivityStackSupervisor#checkReadyForSleepLocked} is skipped.
            final ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */);
            if (next == null || !next.canTurnScreenOn()) {
                checkReadyForSleep();
            }
        } finally {
            mStackSupervisor.inResumeTopActivity = false;
        }

        return result;
    }

```

## resumeTopActivityInnerLocked

因为当前并没有已经运行的 Activity，所以从其他地方找一下要启动的 Activity。

```java
@GuardedBy("mService")
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
    if (!mService.mBooting && !mService.mBooted) {
        // Not ready yet!
        return false;
    }

    // Find the next top-most activity to resume in this stack that is not finishing and is
    // focusable. If it is not focusable, we will fall into the case below to resume the
    // top activity in the next focusable task.
    final ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */);

    final boolean hasRunningActivity = next != null;

    // TODO: Maybe this entire condition can get removed?
    if (hasRunningActivity && !isAttached()) {
        return false;
    }

    mStackSupervisor.cancelInitializingActivities();

    // Remember how we'll process this pause/resume situation, and ensure
    // that the state is reset however we wind up proceeding.
    boolean userLeaving = mStackSupervisor.mUserLeaving;
    mStackSupervisor.mUserLeaving = false;

    if (!hasRunningActivity) {
        // There are no activities left in the stack, let's look somewhere else.
        return resumeTopActivityInNextFocusableStack(prev, options, "noMoreActivities");
    }
  	// ....
}
```

## resumeTopActivityInNextFocusableStack

从其他栈上，，，如果找到，就把焦点移动到其他栈上。不过启动的时候肯定找不到的，所以往下走。

```java
private boolean resumeTopActivityInNextFocusableStack(ActivityRecord prev,
        ActivityOptions options, String reason) {
    if (adjustFocusToNextFocusableStack(reason)) {
        // Try to move focus to the next visible stack with a running activity if this
        // stack is not covering the entire screen or is on a secondary display (with no home
        // stack).
        return mStackSupervisor.resumeFocusedStackTopActivityLocked(
                mStackSupervisor.getFocusedStack(), prev, null);
    }

    // Let's just start up the Launcher...
  	// 启动 Launcher 了
    ActivityOptions.abort(options);
    if (DEBUG_STATES) Slog.d(TAG_STATES,
            "resumeTopActivityInNextFocusableStack: " + reason + ", go home");
    if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
    // Only resume home if on home display
    return isOnHomeDisplay() &&
            mStackSupervisor.resumeHomeStackTask(prev, reason);
}
```

# ActivityStackSupervisor

启动默认的主界面。

## resumeHomeStackTask

会尝试查找  HomeActivity 是否存在，如果找到了，直接就把焦点移过去，如果找不到，那就启动 HomeActivity。

```java
boolean resumeHomeStackTask(ActivityRecord prev, String reason) {
    if (!mService.mBooting && !mService.mBooted) {
        // Not ready yet!
        return false;
    }

    mHomeStack.moveHomeStackTaskToTop();
    ActivityRecord r = getHomeActivity();
    final String myReason = reason + " resumeHomeStackTask";

    // Only resume home activity if isn't finishing.
    if (r != null && !r.finishing) {
        moveFocusableActivityStackToFrontLocked(r, myReason);
        return resumeFocusedStackTopActivityLocked(mHomeStack, prev, null);
    }
    return mService.startHomeActivityLocked(mCurrentUser, myReason);
}
```



# ActivityManagerService

## startHomeActivityLocked

```java
boolean startHomeActivityLocked(int userId, String reason) {
    if (mFactoryTest == FactoryTest.FACTORY_TEST_LOW_LEVEL
            && mTopAction == null) {
        // We are running in factory test mode, but unable to find
        // the factory test app, so just sit around displaying the
        // error message and don't try to start anything.
        return false;
    }
    Intent intent = getHomeIntent();
    ActivityInfo aInfo = resolveActivityInfo(intent, STOCK_PM_FLAGS, userId);
    if (aInfo != null) {
        intent.setComponent(new ComponentName(aInfo.applicationInfo.packageName, aInfo.name));
        // Don't do this if the home app is currently being
        // instrumented.
        aInfo = new ActivityInfo(aInfo);
        aInfo.applicationInfo = getAppInfoForUser(aInfo.applicationInfo, userId);
        ProcessRecord app = getProcessRecordLocked(aInfo.processName,
                aInfo.applicationInfo.uid, true);
        if (app == null || app.instr == null) {
            intent.setFlags(intent.getFlags() | FLAG_ACTIVITY_NEW_TASK);
            final int resolvedUserId = UserHandle.getUserId(aInfo.applicationInfo.uid);
            // For ANR debugging to verify if the user activity is the one that actually
            // launched.
            final String myReason = reason + ":" + userId + ":" + resolvedUserId;
            mActivityStartController.startHomeActivity(intent, aInfo, myReason);
        }
    } else {
        Slog.wtf(TAG, "No home screen found for " + intent, new Throwable());
    }

    return true;
}
```



# ActivityStartController

这个用来控制 Activity 的启动。

## startHomeActivity

```java
void startHomeActivity(Intent intent, ActivityInfo aInfo, String reason) {
    mSupervisor.moveHomeStackTaskToTop(reason);

    mLastHomeActivityStartResult = obtainStarter(intent, "startHomeActivity: " + reason)
            .setOutActivity(tmpOutRecord)
            .setCallingUid(0)
            .setActivityInfo(aInfo)
            .execute();
    mLastHomeActivityStartRecord = tmpOutRecord[0];
    if (mSupervisor.inResumeTopActivity) {
        // If we are in resume section already, home activity will be initialized, but not
        // resumed (to avoid recursive resume) and will stay that way until something pokes it
        // again. We need to schedule another resume.
        mSupervisor.scheduleResumeTopActivities();
    }
}
```

# ActivityStarter

## execute

```java
int execute() {
    try {
        // TODO(b/64750076): Look into passing request directly to these methods to allow
        // for transactional diffs and preprocessing.
        if (mRequest.mayWait) {
            return startActivityMayWait(mRequest.caller, mRequest.callingUid,
                    mRequest.callingPackage, mRequest.intent, mRequest.resolvedType,
                    mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                    mRequest.resultWho, mRequest.requestCode, mRequest.startFlags,
                    mRequest.profilerInfo, mRequest.waitResult, mRequest.globalConfig,
                    mRequest.activityOptions, mRequest.ignoreTargetSecurity, mRequest.userId,
                    mRequest.inTask, mRequest.reason,
                    mRequest.allowPendingRemoteAnimationRegistryLookup);
        } else {
            return startActivity(mRequest.caller, mRequest.intent, mRequest.ephemeralIntent,
                    mRequest.resolvedType, mRequest.activityInfo, mRequest.resolveInfo,
                    mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                    mRequest.resultWho, mRequest.requestCode, mRequest.callingPid,
                    mRequest.callingUid, mRequest.callingPackage, mRequest.realCallingPid,
                    mRequest.realCallingUid, mRequest.startFlags, mRequest.activityOptions,
                    mRequest.ignoreTargetSecurity, mRequest.componentSpecified,
                    mRequest.outActivity, mRequest.inTask, mRequest.reason,
                    mRequest.allowPendingRemoteAnimationRegistryLookup);
        }
    } finally {
        onExecutionComplete();
    }
}
```

## startActivity

```java
private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
        String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
        String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
        SafeActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
        ActivityRecord[] outActivity, TaskRecord inTask, String reason,
        boolean allowPendingRemoteAnimationRegistryLookup) {

    if (TextUtils.isEmpty(reason)) {
        throw new IllegalArgumentException("Need to specify a reason.");
    }
    mLastStartReason = reason;
    mLastStartActivityTimeMs = System.currentTimeMillis();
    mLastStartActivityRecord[0] = null;

    mLastStartActivityResult = startActivity(caller, intent, ephemeralIntent, resolvedType,
            aInfo, rInfo, voiceSession, voiceInteractor, resultTo, resultWho, requestCode,
            callingPid, callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
            options, ignoreTargetSecurity, componentSpecified, mLastStartActivityRecord,
            inTask, allowPendingRemoteAnimationRegistryLookup);

    if (outActivity != null) {
        // mLastStartActivityRecord[0] is set in the call to startActivity above.
        outActivity[0] = mLastStartActivityRecord[0];
    }

    return getExternalResult(mLastStartActivityResult);
}
```

最终会执行到一个重载的方法来启动。



```java
    private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
            ActivityRecord[] outActivity) {
....
  
    }
```

更多的细节就暂时不看了，太多了，要得花很多心思才能弄得通透。
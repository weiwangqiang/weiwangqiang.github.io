---
layout:     post
title:      "【framework】startActivity流程"
subtitle:   " \"带你细看Android13上activity的启动流程\""
date:       2021-06-08 22:30:00
author:     "Weiwq"
header-img: "img/background/post-bg-nextgen-web-pwa.jpg"
catalog:  true
isTop:  true
tags:
    - Android

---

> 一步步看，你就会对activity的启动流程有深刻的认知。

## 引言

从Android11开始，Activity的启动流程与Android10的实现（可以参考[Activity的启动过程详解（基于10.0源码）](https://cloud.tencent.com/developer/article/1666801)）又不一样了，但是万变不离其中，变的更多是代码上的优化。

如果不想看代码，可以直接看对应的时序图。

## 1 startActivity流程

### 1.1 Activity

> 代码路径：frameworks\base\core\java\android\app\Activity.java

稍微了解activity的启动流程的读者应该知道，startActivity里面大概的工作：主要是通过Instrumentation用ActivityManagerService的代理对象，执行启动请求，中间涉及到binder的通信，如果大家不熟悉，可以看看笔者之前的文章 [Android跨进程之ADIL原理](https://weiwangqiang.github.io/2021/05/16/android-AIDL-analyze/) 。

那我们来看看Android13 又是如何实现的呢？

startActivity调用的是startActivityForResult方法，直接看startActivityForResult的实现好了

#### startActivityForResult

```java
public class Activity extends ContextThemeWrapper{
    public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
      ....
      Instrumentation.ActivityResult ar = mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
      ....
    }
}
```

### 1.2 Instrumentation

> 代码路径：frameworks\base\core\java\android\app\Instrumentation.java

#### execStartActivity 

execStartActivity 方法的主要实现如下。大概能看出，这里是直接拿到ActivityTaskManager 服务去startActivity。

```java
public class Instrumentation {
    ....
    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
      ....
      int result = ActivityTaskManager.getService().startActivity(whoThread,
                    who.getBasePackageName(), who.getAttributionTag(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()), token,
                    target != null ? target.mEmbeddedID : null, requestCode, 0, null, options);
      // 检查启动结果
      checkStartActivityResult(result, intent);
      ....
    }
}
```

#### checkStartActivityResult

先看checkStartActivityResult的实现，就是处理启动activity失败的场景，典型的问题比如activity没有注册。

```java
// 检查启动结果
 public static void checkStartActivityResult(int res, Object intent) {
        ....
        switch (res) {
            case ActivityManager.START_INTENT_NOT_RESOLVED:
            case ActivityManager.START_CLASS_NOT_FOUND:
                if (intent instanceof Intent && ((Intent)intent).getComponent() != null)
                    throw new ActivityNotFoundException(
                            "Unable to find explicit activity class "
                            + ((Intent)intent).getComponent().toShortString()
                            + "; have you declared this activity in your AndroidManifest.xml?");
                throw new ActivityNotFoundException(
                        "No Activity found to handle " + intent);
            case ActivityManager.START_PERMISSION_DENIED:
                throw new SecurityException("Not allowed to start activity "
                        + intent);
             ....
            default:
                throw new AndroidRuntimeException("Unknown error code "
                        + res + " when starting " + intent);
        }
    }
```

### 1.3 ActivityTaskManager

> 代码路径：frameworks\base\core\java\android\app\ActivityTaskManager.java

#### getService

回到ActivityTaskManager.getService()的实现，通过下面的代码可以看出来getService方法返回的是IActivityTaskManager binder对象

```java
public class ActivityTaskManager {
    ....
    // 一个单例的实现
    private static final Singleton<IActivityTaskManager> IActivityTaskManagerSingleton =
        new Singleton<IActivityTaskManager>() {
        @Override
        protected IActivityTaskManager create() {
           final IBinder b = ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE);
           return IActivityTaskManager.Stub.asInterface(b);
        }
    };
    ....
    public static IActivityTaskManager getService() {
        return IActivityTaskManagerSingleton.get();
   } 
}

```

看过ADIL实现的同学，应该很快反应出怎么找IActivityTaskManager 的实现类，全局搜一下 `IActivityTaskManager.Stub`即可，代码如下

```java
public class ActivityTaskManagerService extends IActivityTaskManager.Stub {
   ....
}
```

那客户端又是怎么获取到ActivityTaskManagerService的binder对象的呢？

从单例的实现，我们可以看到是通过 `ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE)` 方式拿到指定name对应的binder。ServiceManager 的实现如下

```java
public final class ServiceManager {
   // Returns a reference to a service with the given name.
   // 返回对具有给定名称的服务的引用。
    public static IBinder getService(String name) {
        try {
            IBinder service = sCache.get(name);
            if (service != null) {
                // 有缓存，直接从缓存获取
                return service;
            } else {
                // 否则重新通过ServiceManager获取
                return Binder.allowBlocking(rawGetService(name));
            }
        } catch (RemoteException e) {
            Log.e(TAG, "error in getService", e);
        }
        return null;
    }
    // 通过指定name获取service的binder引用
    private static IBinder rawGetService(String name) throws RemoteException {
        final IBinder binder = getIServiceManager().getService(name);
        ....
        return binder;
    }
    
   // 获取ServiceManager的Binder对象
    private static IServiceManager getIServiceManager() {
        if (sServiceManager != null) {
            return sServiceManager;
        }
        // Find the service manager
        sServiceManager = ServiceManagerNative
                .asInterface(Binder.allowBlocking(BinderInternal.getContextObject()));
        return sServiceManager;
    }
}
```

那客户端又是怎么拿到serviceManager的binder对象呢？有一个很重要的实现

```java
public class BinderInternal {
    // 返回系统的全局“上下文对象”。这通常是 IServiceManager 的实现，
    // 您可以使用它来查找其他服务。
    public static final native IBinder getContextObject();
}	
```

### 1.4 时序图

startActivity对应的时序图如下：

<img src="/img/blog_start_activity_flow/activity_clinet.png" width="100%" height="40%"> 

## 2 ATMS流程

### 2.1 ActivityTaskManagerService

> 代码路径：frameworks\base\services\core\java\com\android\server\wm\ActivityTaskManagerService.java

与Android10上不一样的是，ActivityTaskManagerService 替换掉了ActivityManagerService的部分工作，用于管理活动及其容器（任务、堆栈、显示...）的系统服务。

#### startActivity

```java
@Override
public final int startActivity(IApplicationThread caller, String callingPackage,
        String callingFeatureId, Intent intent, String resolvedType, IBinder resultTo,
        String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo,
        Bundle bOptions) {
    return startActivityAsUser(caller, callingPackage, callingFeatureId, 
                               intent, resolvedType, resultTo, resultWho, requestCode, 
                               startFlags, profilerInfo, bOptions,
                               UserHandle.getCallingUserId());
}
```

#### startActivityAsUser

```java
// 最终调用这个方法
private int startActivityAsUser(IApplicationThread caller, String callingPackage,
        @Nullable String callingFeatureId, Intent intent, String resolvedType,
        IBinder resultTo, String resultWho, int requestCode, int startFlags,
        ProfilerInfo profilerInfo, Bundle bOptions, int userId, boolean validateIncomingUser) {
    ....
    userId = getActivityStartController().checkTargetUser(userId, validateIncomingUser,
            Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");
    // TODO: Switch to user app stacks here.
    return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
            .setCaller(caller)
            .setCallingPackage(callingPackage)
            .setCallingFeatureId(callingFeatureId)
            .setResolvedType(resolvedType)
            .setResultTo(resultTo)
            .setResultWho(resultWho)
            .setRequestCode(requestCode)
            .setStartFlags(startFlags)
            .setProfilerInfo(profilerInfo)
            .setActivityOptions(bOptions)
            .setUserId(userId)
            .execute();
}
// 获取ActivityStartController
ActivityStartController getActivityStartController() {
    return mActivityStartController;
}
```

顺便提一下，ActivityManagerService中的startActivity 方法，实际上也是调用ActivityTaskManagerService的接口，不过这个方法已经被废弃了。

```java
public class ActivityManagerService extends IActivityManager.Stub
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
    public ActivityTaskManagerService mActivityTaskManager;
  
    @Deprecated
    @Override
    public int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
        return mActivityTaskManager.startActivity(caller, callingPackage, null, intent,
                resolvedType, resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions);
    }
}
```

接着看。getActivityStartController 方法返回一个ActivityStartController对象，看看它的obtainStarter方法实现

```java
// 用于委托活动启动的控制器。
// 该类的主要目标是获取外部活动启动请求，并将它们准备为一系列可由 
// {@link ActivityStarter} 处理的离散活动启动。它还负责处理活动启动周围发生的逻辑，但不一定影响活动启动。
// 示例包括电源提示管理、处理待处理活动列表以及记录home activity启动。
public class ActivityStartController {
     ....
     // @return一个启动程序，用于配置和执行启动活动。有效期直到调用{@link ActivityStarter#execute}之后。
     // 在这一点上，启动器应被视为无效，并且不再被修改或使用。 
     ActivityStarter obtainStarter(Intent intent, String reason) {
        return mFactory.obtain().setIntent(intent).setReason(reason);
    }
}
```

这个方法使用工厂模式，返回了一个ActivityStarter对象，接着用ActivityStarter对象设置了一些启动上的配置，直接看execute方法好了。

### 2.2 ActivityStarter

> frameworks\base\services\core\java\com\android\server\wm\ActivityStarter.java

ActivityStarter用于解释如何然后启动活动的控制器。此类收集用于确定意图和标志应如何转换为活动以及相关任务和堆栈的所有逻辑。

#### execute

```java
// 根据前面提供的请求参数解析必要的信息，执行开始活动的请求。 
// @return 起始结果。
int execute() {
  ....
  if (mRequest.activityInfo == null) {
      mRequest.resolveActivity(mSupervisor);
  }
  int res;
  synchronized (mService.mGlobalLock) {
     ....
     res = executeRequest(mRequest);
  }
}
```

里面是直接把request对象给到了executeRequest 方法，在里面会做一些参数的检查

#### executeRequest

```java
	// 执行活动启动请求，开启活动启动之旅。首先是执行几个初步检查。
    // 通常的 Activity 启动流程会经过 {@link startActivityUnchecked} 到 {@link startActivityInner}。
    private int executeRequest(Request request) {
        int err = ActivityManager.START_SUCCESS;
        ....
        // 我们找不到可以处理给定 Intent 的类。
        if (err == ActivityManager.START_SUCCESS && intent.getComponent() == null) {
            // We couldn't find a class that can handle the given Intent.
            // That's the end of that!
            err = ActivityManager.START_INTENT_NOT_RESOLVED;
        }
        // 我们找不到 Intent 中指定的特定类。
        if (err == ActivityManager.START_SUCCESS && aInfo == null) {
            // We couldn't find the specific class specified in the Intent.
            // Also the end of the line.
            err = ActivityManager.START_CLASS_NOT_FOUND;
        }
        ....
        // 检查启动activity的权限
        boolean abort = !mSupervisor.checkStartAnyActivityPermission(intent, aInfo, resultWho,
                requestCode, callingPid, callingUid, callingPackage, callingFeatureId,
                request.ignoreTargetSecurity, inTask != null, callerApp, resultRecord, resultStack);
        abort |= !mService.mIntentFirewall.checkStartActivity(intent, callingUid,
                callingPid, resolvedType, aInfo.applicationInfo);
        abort |= !mService.getPermissionPolicyInternal().checkStartActivity(intent, callingUid,
                callingPackage);  
        boolean restrictedBgActivity = false;
        if (!abort) {
            try {
                // 检查是否允许后台启动activity，以下情况会允许后台启动activity
                // 1、一些重要的UId比如System UID，NFC UID
                // 2、callingUid具有可见窗口或是持久性系统进程（即persistent的系统进程）
                // 3、callingUid具有START_ACTIVITIES_FROM_BACKGROUND权限
                // 4、调用方的uid与最近使用的组件具有相同的uid
                // 5、callingUid是设备所有者或者有伴侣设备
                // 6、callingUid具有SYSTEM_ALERT_WINDOW权限
                // 7、调用者在任何前台任务中有活动
                // 8、调用者被当前前台的 UID 绑定
                restrictedBgActivity = shouldAbortBackgroundActivityStart(callingUid,
                        callingPid, callingPackage, realCallingUid, realCallingPid, callerApp,
                        request.originatingPendingIntent, request.allowBackgroundActivityStart,
                        intent);
            } finally {
            }
        }
        // 如果权限需要审阅才能运行任何应用程序组件，我们将启动审阅活动，
        // 并传递未决的意向以启动在审阅完成后现在要启动的活动。
        if (aInfo != null) {
            // 获取此包使用的某些权限是否需要用户审核才能运行任何应用程序组件。
            if (mService.getPackageManagerInternalLocked().isPermissionsReviewRequired(
                    aInfo.packageName, userId)) {
        				....
                // 启动权限审核
                Intent newIntent = new Intent(Intent.ACTION_REVIEW_PERMISSIONS);
                intent = newIntent;   
                ....
            }
        }
        ....
        // 创建ActivityRecord对象，用于保存activity信息
        final ActivityRecord r = new ActivityRecord(mService, callerApp, callingPid, callingUid,
                callingPackage, callingFeatureId, intent, resolvedType, aInfo,
                mService.getGlobalConfiguration(), resultRecord, resultWho, requestCode,
                request.componentSpecified, voiceSession != null, mSupervisor, checkedOptions,
                sourceRecord);
        mLastStartActivityRecord = r;
        // 进入启动activity的流程（跳过检查）
        mLastStartActivityResult = startActivityUnchecked(r, sourceRecord, voiceSession,
                request.voiceInteractor, startFlags, true /* doResume */, checkedOptions, inTask,
                restrictedBgActivity, intentGrants);
        ....
        return mLastStartActivityResult;
   }
```

`shouldAbortBackgroundActivityStart` 这个方法很重要，里面主要处理是否允许后台启动activity的逻辑（高版本对service启动activity做了限制，防止广告泛滥），详情可见上面的备注。

#### startActivityUnchecked 

```java
// 在大多数初步检查都已完成并且已确认呼叫者拥有必要的权限的情况下，开始活动。
// 如果启动失败，这里还可以确保删除启动活动。
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
                                   IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                                   int startFlags, boolean do Resume, ActivityOptions options, Task inTask,
                                   boolean restrictedBgActivity, NeededUriGrants intentGrants) {
    int result = START_CANCELED;
    final ActivityStack startedActivityStack;
    try {
        .....
        result = startActivityInner(r, sourceRecord, voiceSession, voiceInteractor,
                startFlags, doResume, options, inTask, restrictedBgActivity, intentGrants);
    } finally {
       // 如果启动结果成功，请确保启动活动的配置与当前显示匹配。否则清理不相关的容器以避免泄漏。
        startedActivityStack = handleStartResult(r, result);
    }
    ....
    return result;
}
```

startActivityUnchecked 没做什么事情，把启动的动作交给了startActivityInner方法。

#### startActivityInner

```java
// 启动一个活动，并确定该活动是应该添加到现有任务的顶部还是应该向现有活动传递新的意图。
// 还可以将活动任务操纵到请求的或有效的堆栈显示上。
int startActivityInner(final ActivityRecord r, ActivityRecord sourceRecord,
                       IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                       int startFlags, boolean doResume, ActivityOptions options, Task inTask,
                       boolean restrictedBgActivity, NeededUriGrants intentGrants) {
    // 处理activity启动模式  
    computeLaunchingTaskFlags();
    // 确定是否应将新活动插入现有任务。如果不是，则返回null，
    // 或者返回带有应将新活动添加到其中的任务的ActivityRecord。
    final Task reusedTask = getReusableTask();
    // 计算是否存在可以使用的任务栈
    final Task targetTask = reusedTask != null ? reusedTask : computeTargetTask();
    // 检查是否允许在给定任务或新任务上启动活动。
    int startResult = isAllowedToStart(r, newTask, targetTask);
    if (startResult != START_SUCCESS) {
        ....
        return startResult;
    }
    // 如果正在启动的活动与当前位于顶部的活动相同，则我们需要检查它是否应该只启动一次
    final Task topRootTask = mPreferredTaskDisplayArea.getFocusedRootTask();
    if (topRootTask != null) {
        startResult = deliverToCurrentTopIfNeeded(topRootTask, intentGrants);
        if (startResult != START_SUCCESS) {
            // 不需要启动新的activity，直接返回
            // 并且 deliverToCurrentTopIfNeeded 里面会触发onNewIntent
            return startResult;
        }
    }
    ....
    if (mTargetRootTask == null) {
        // 创建根任务栈
        mTargetRootTask = getOrCreateRootTask(mStartActivity, mLaunchFlags, targetTask,
                mOptions);
    }
    if (newTask) {
       final Task taskToAffiliate = (mLaunchTaskBehind && mSourceRecord != null)
               ? mSourceRecord.getTask() : null;
       setNewTask(taskToAffiliate);
    } else if (mAddingToTask) {
        // 复用之前的task
        addOrReparentStartingActivity(targetTask, "adding to task");
    }
    ....
    // 检查是否需要触发过渡动画和开始窗口
    final boolean isTaskSwitch = startedTask != prevTopTask && !startedTask.isEmbedded();
    mTargetStack.startActivityLocked(mStartActivity, topStack.getTopNonFinishingActivity(),
            newTask, mKeepCurTransition, mOptions);
    if (mDoResume) {
        final ActivityRecord topTaskActivity = startedTask.topRunningActivityLocked();
        if (!mTargetRootTask.isTopActivityFocusable()
                || (topTaskActivity != null && topTaskActivity.isTaskOverlay()
                && mStartActivity != topTaskActivity)) {
            // 如果活动不可聚焦，我们无法恢复它，但仍希望确保它在开始时变得可见（这也将触发条目动画）。
            // 这方面的一个例子是PIP活动。此外，我们不希望在当前具有覆盖层的任务中恢复活动，
            // 因为启动活动只需要处于可见的暂停状态，直到删除结束。传递 {@code null} 作为启动参数可确保所有活动都可见。
            mTargetRootTask.ensureActivitiesVisible(null /* starting */,
                    0 /* configChanges */, !PRESERVE_WINDOWS);
            // 继续并告诉窗口管理器为此活动执行应用程序转换，因为应用程序转换不会通过恢复通道触发。
            mTargetRootTask.mDisplayContent.executeAppTransition();
        } else {
            // 如果目标根任务以前不可聚焦（该根任务上先前的顶级运行活动不可见），则任何先前  将根任务移动到 的调用都不会更新重点根任务。
            // 如果现在启动新活动允许任务根任务可聚焦，请确保我们现在相应地更新重点根任务。
            if (mTargetRootTask.isTopActivityFocusable()
                    && !mRootWindowContainer.isTopDisplayFocusedRootTask(mTargetRootTask)) {
                mTargetRootTask.moveToFront("startActivityInner");
            }
            // 启动目标activity
            mRootWindowContainer.resumeFocusedTasksTopActivities(
                    mTargetRootTask, mStartActivity, mOptions, mTransientLaunch);
        }
    }
    ....
    return START_SUCCESS;
}
```

startActivityInner 负责的任务就是准备好堆栈，为启动activity做最后的准备。

其中computeLaunchingTaskFlags 是负责处理启动模式相关的逻辑

```java
private void computeLaunchingTaskFlags() {
   ....
   if (mInTask == null) { // 表示activity要加入的栈不存在
       if (mSourceRecord == null) {
           // activity 还没有被启动，在这种情况下，我们都会创建一个新的任务栈
           if ((mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) == 0 && mInTask == null) {
               Slog.w(TAG, "startActivity called from non-Activity context; forcing " +
                       "Intent.FLAG_ACTIVITY_NEW_TASK for: " + mIntent);
               mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK;
           }
       } else if (mSourceRecord.launchMode == LAUNCH_SINGLE_INSTANCE) {
           // 当前activity是由singleInstance的activity启动，新activity则需要在自己的任务栈中启动
           mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK;
       } else if (isLaunchModeOneOf(LAUNCH_SINGLE_INSTANCE, LAUNCH_SINGLE_TASK)) {
           // 如果设置了singeInstance或者singleTask，也需要创建一个新栈
           mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK;
       }
   }
}
```

接下来就把流程交给了RootWindowContainer的resumeFocusedStacksTopActivities方法。

### 2.3 RootWindowContainer

> 代码路径： frameworks\base\services\core\java\com\android\server\wm\RootWindowContainer.java

#### resumeFocusedTasksTopActivities

```java
boolean resumeFocusedTasksTopActivities(
            Task targetRootTask, ActivityRecord target, ActivityOptions targetOptions,
            boolean deferPause) {
        .... 
        boolean result = false;
        if (targetRootTask != null && (targetRootTask.isTopRootTaskInDisplayArea()
                || getTopDisplayFocusedRootTask() == targetRootTask)) {
            result = targetRootTask.resumeTopActivityUncheckedLocked(target, targetOptions,
                    deferPause);
        }
        ....
        return result;
}

```

resumeFocusedStacksTopActivities 接着交给了Task的resumeTopActivityUncheckedLocked 方法。

### 2.4 Task

> 代码路径： frameworks\base\services\core\java\com\android\server\wm\Task.java

#### resumeTopActivityUncheckedLocked

```java
// 确保恢复堆栈中的顶部活动。
// @param prev 之前恢复的活动，用于暂停过程中；从别处调用的时候可以为 null 
// @param options 活动选项。
// @param deferPause 如果是true，将不会暂停任务。
// 注意：直接调用此方法是不安全的，因为它会导致非焦点堆栈中的活动被恢复。
//     需要使用 {@link RootWindowContainer#resumeFocusedStacksTopActivities} 恢复当前系统状态的正确活动。
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options,
            boolean deferPause) {
    ....
    boolean result = false;
    try {
        // Protect against recursion.
        mInResumeTopActivity = true;
        if (isLeafTask()) {
            if (isFocusableAndVisible()) {
                someActivityResumed = resumeTopActivityInnerLocked(prev, options, deferPause);
        }
        .....
        // 恢复top Activity时，可能需要暂停top Activity
        // （比如返回锁屏。我们在{@link resumeTopActivityUncheckedLocked}中抑制了正常的暂停逻辑，
        // 因为top Activity最后是恢复的。我们调用此处再次调用
        // {@link ActivityStackSupervisor#checkReadyForSleepLocked}
        // 以确保发生任何必要的暂停逻辑。在无论锁定屏幕如何都会显示 Activity 的情况下，将跳过对
        // {@link ActivityStackSupervisor#checkReadyForSleepLocked} 的调用。
        final ActivityRecord next = topRunningActivity(true /* focusableOnly */);
        if (next == null || !next.canTurnScreenOn()) {
            checkReadyForSleep();
        }
    } finally {
        mInResumeTopActivity = false;
    }
    return result;
}
```

#### resumeTopActivityInnerLocked

```java
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options,
            boolean deferPause) {
        final ActivityRecord topActivity = topRunningActivity(true /* focusableOnly */);
        if (topActivity == null) {
            // task中没有activity, 就从其他地方找。
            return resumeNextFocusableActivityWhenRootTaskIsEmpty(prev, options);
        }
        final boolean[] resumed = new boolean[1];
        final TaskFragment topFragment = topActivity.getTaskFragment();
        resumed[0] = topFragment.resumeTopActivity(prev, options, deferPause);
        forAllLeafTaskFragments(f -> {
            if (topFragment == f) {
                return;
            }
            if (!f.canBeResumed(null /* starting */)) {
                return;
            }
            resumed[0] |= f.resumeTopActivity(prev, options, deferPause);
        }, true);
        return resumed[0];   
 }
```

### 2.5 TaskFragment

> 代码路径：frameworks\base\services\core\java\com\android\server\wm\TaskFragment.java

TaskFragment是一个基本的容器，可用于包含活动或其他TaskFragment，也能够管理活动生命周期和更新活动的可见性。

Android 13 提供了一个新特性：平行世界，官方称之为[activity 嵌入](https://developer.android.google.cn/guide/topics/large-screens/activity-embedding?hl=zh-cn)。TaskFragment 正是实现该特性的关键。

<img src="/img/blog_start_activity_flow/settings_app.png" width="50%" height="50%"> 

#### resumeTopActivity

```java
final boolean resumeTopActivity(ActivityRecord prev, ActivityOptions options,
        boolean deferPause) {
    ActivityRecord next = topRunningActivity(true /* focusableOnly */);
    final TaskDisplayArea taskDisplayArea = getDisplayArea();
    ActivityRecord lastResumed = null;
    boolean pausing = !deferPause && taskDisplayArea.pauseBackTasks(next);
    if (mResumedActivity != null) {
        // 将当前resume的activity 转到pause状态
        pausing |= startPausing(mTaskSupervisor.mUserLeaving, false /* uiSleeping */,
                next, "resumeTopActivity");
    }
    // pausing 为true表示调用了上一个activity的pause周期
    if (pausing) {
        // 在这一点上，我们要把即将到来的activity的进程放在LRU列表的顶部
        // 因为我们知道,我们将很快需要它，如果碰巧在未来让他被杀，这会导致浪费。
        if (next.attachedToProcess()) {// 下一个应用的进程已经启动
            next.app.updateProcessInfo(false /* updateServiceConnectionActivities */,
                    true /* activityChange */, false /* updateOomAdj */,
                    false /* addPendingTopUid */);
        } else if (!next.isProcessRunning()) { // 如果进程还没启动
            // 自起动过程是异步的,如果我们已经知道下一个活动的过程不是跑步,我们可以开始这个过程之前保存时间等待当前活动暂停。
            final boolean isTop = this == taskDisplayArea.getFocusedRootTask();
            mAtmService.startProcessAsync(next, false /* knownToBeDead */, isTop,
                    isTop ? HostingRecord.HOSTING_TYPE_NEXT_TOP_ACTIVITY
                            : HostingRecord.HOSTING_TYPE_NEXT_ACTIVITY);
        }
        return true;
    } else if (mResumedActivity == next && next.isState(RESUMED)
            && taskDisplayArea.allResumedActivitiesComplete()) {
        // 活动有可能恢复正常,当我们停下来回堆栈上面如果下一个活动不需要等待暂停来完成。
        // 所以,没有其他的任务除了:确保我们有执行任何悬而未决的过渡,因为此时应该一无所有。
        executeAppTransition(options);
        return true;
    }

    // 我们是启动下一个活动,所以告诉窗口管理器上一个很快就会被隐藏。这种方式可以知道忽略它当计算所需的屏幕方向。
    boolean anim = true;
    final DisplayContent dc = taskDisplayArea.mDisplayContent;
    if (prev != null) {
        if (prev.finishing) {
            // 启动退出动画
            if (mTaskSupervisor.mNoAnimActivities.contains(prev)) {
                anim = false;
                dc.prepareAppTransition(TRANSIT_NONE);
            } else {
                dc.prepareAppTransition(TRANSIT_CLOSE);
            }
            prev.setVisibility(false);
        } else {
            // 启动打开动画
            if (mTaskSupervisor.mNoAnimActivities.contains(next)) {
                anim = false;
                dc.prepareAppTransition(TRANSIT_NONE);
            } else {
                dc.prepareAppTransition(TRANSIT_OPEN,
                        next.mLaunchTaskBehind ? TRANSIT_FLAG_OPEN_BEHIND : 0);
            }
        }
    } 
    
    if (next.attachedToProcess()) { // activity已经存在
        .....
        try {
            // 创建一个事务来调用activity的周期
            final ClientTransaction transaction =
                    ClientTransaction.obtain(next.app.getThread(), next.token);
            if (next.newIntents != null) {
                // 需要调用onNewIntent周期
                transaction.addCallback(
                        NewIntentItem.obtain(next.newIntents, true /* resume */));
            } 
            // 应用程序将不再停止。明确软件令牌在窗口管理器停止状态。
            next.notifyAppResumed(next.stopped);
            mAtmService.getAppWarningsLocked().onResumeActivity(next);
            next.app.setPendingUiCleanAndForceProcessStateUpTo(mAtmService.mTopProcessState);
            next.abortAndClearOptionsAnimation();
            // 设置activity 生命周期最终的状态为onResume
            transaction.setLifecycleStateRequest(
                    ResumeActivityItem.obtain(next.app.getReportedProcState(),
                            dc.isNextTransitionForward()));
            // 启动activity
            mAtmService.getLifecycleManager().scheduleTransaction(transaction);
        } catch (Exception e) {
            // resume异常，需要重新启动activity
            ....
            mTaskSupervisor.startSpecificActivity(next, true, false);
            return true;
        }
    }  else { // 目标activity还没有启动
       ....
       // 调用ActivityTaskSupervisor的startSpecificActivity
       mTaskSupervisor.startSpecificActivity(next, true, true);
    }
}
```

我们先看看startPausing 方法

#### startPausing

该将当前resumed的activity 转到pause状态。如果已经有activity被暂停或没有resume的activity，不能调用这个方法

```java
// return true：如果一个activity正处于暂停状态,我们正在等待它告诉我们何时完成。
boolean startPausing(boolean userLeaving, boolean uiSleeping, ActivityRecord resuming,
        String reason) {
    ActivityRecord prev = mResumedActivity;
    ...
    mPausingActivity = prev;
    mLastPausedActivity = prev;
    
    boolean pauseImmediately = false;
    ...
    if (prev.attachedToProcess()) {
        if (shouldAutoPip) {
            boolean didAutoPip = mAtmService.enterPictureInPictureMode(
                    prev, prev.pictureInPictureArgs);
        } else {
            // 调用pause周期
            schedulePauseActivity(prev, userLeaving, pauseImmediately, reason);
        }
    }
    if (mPausingActivity != null) {
       if (pauseImmediately) { // 正常情况，pauseImmediately 为false
           // 如果调用者说他们不想等待暂停,然后完成暂停了。
           completePause(false, resuming);
           return false;
       } else {
           // 启动定时
           prev.schedulePauseTimeout();
           // 未设置的准备,因为我们现在需要等到完成此暂停。
           mTransitionController.setReady(this, false /* ready */);
           return true;
       }
    }
}
```

#### schedulePauseActivity

该方法主要发送一个onPause的事务给客户端，调用activity的onPause周期

```java
void schedulePauseActivity(ActivityRecord prev, boolean userLeaving,
        boolean pauseImmediately, String reason) {
   try {
        // 发送Pause的事务
        mAtmService.getLifecycleManager().scheduleTransaction(prev.app.getThread(),
                prev.token, PauseActivityItem.obtain(prev.finishing, userLeaving,
                        prev.configChangeFlags, pauseImmediately));
    } catch (Exception e) {
       ....
    }
}
```

我们继续回到创建activity的流程

### 2.6 ActivityTaskSupervisor

> 代码路径： frameworks\base\services\core\java\com\android\server\wm\ActivityTaskSupervisor.java

#### startSpecificActivity

```java
 void startSpecificActivity(ActivityRecord r, boolean andResume, boolean checkConfig) {
    // 用于判断app 进程是否已经启动
    final WindowProcessController wpc =
            mService.getProcessController(r.processName, r.info.applicationInfo.uid);
    if (wpc != null && wpc.hasThread()) {
        try {
            // 真正启动activity的地方
            realStartActivityLocked(r, wpc, andResume, checkConfig);
            return;
        } catch (RemoteException e) {
        }
        ...
    }
    // 启动失败，对应的应用程序没有运行，就创建一个进程
    // 这里的mService即ActivityTaskManagerService
    r.notifyUnknownVisibilityLaunchedForKeyguardTransition();
    final boolean isTop = andResume && r.isTopRunningActivity();
    mService.startProcessAsync(r, knownToBeDead, isTop,
            isTop ? HostingRecord.HOSTING_TYPE_TOP_ACTIVITY
                    : HostingRecord.HOSTING_TYPE_ACTIVITY);
 }
```

#### realStartActivityLocked

```java
 boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
         boolean andResume, boolean checkConfig) throws RemoteException {
     ....
     try {
         // 创建活动启动事务。
         final ClientTransaction clientTransaction = ClientTransaction.obtain(
                         proc.getThread(), r.appToken); 
         // 在回调序列的末尾添加一条消息。
         // 注意，这里会给客户端用于创建activity
         clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                         System.identityHashCode(r), r.info,
                         mergedConfiguration.getGlobalConfiguration(),
                         mergedConfiguration.getOverrideConfiguration(), r.compat,
                         r.launchedFromPackage, task.voiceInteractor, proc.getReportedProcState(),
                         r.getSavedState(), r.getPersistentSavedState(), results, newIntents,
                         dc.isNextTransitionForward(), proc.createProfilerInfoIfNeeded(),
                         r.assistToken, r.createFixedRotationAdjustmentsIfNeeded()));
         // 设置所需的最终状态。
         final ActivityLifecycleItem lifecycleItem;
         // 这里创建的是 ResumeActivityItem
         if (andResume) {
              lifecycleItem = ResumeActivityItem.obtain(dc.isNextTransitionForward());
         } else {
              lifecycleItem = PauseActivityItem.obtain();
         }
          // 设置lifecycleItem
         clientTransaction.setLifecycleStateRequest(lifecycleItem);
         //  安排一个事务,通知给客户端
         mService.getLifecycleManager().scheduleTransaction(clientTransaction);
     } catch (RemoteException e) {
         ....
         throw e;
     }
     return true;
 }
```

getLifecycleManager() 返回的是一个ClientLifecycleManager对象

### 2.7 ClientLifecycleManager

> 代码路径：frameworks\base\services\core\java\com\android\server\wm\ClientLifecycleManager.java

ClientLifecycleManager 能够组合多个客户端生命周期转换请求和回调的类，并将它们作为单个事务执行。

#### scheduleTransaction

```java
// 安排一个事务，它可能包含多个回调和一个生命周期请求。
void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
   // 拿到客户端的binder对象。即ActivityThread内部类ApplicationThread
    final IApplicationThread client = transaction.getClient();
    // 执行 schedule 方法
    transaction.schedule();
    if (!(client instanceof Binder)) {
        transaction.recycle();
    }
}
```

### 2.8 ClientTransaction

> 代码路径：frameworks\base\core\java\android\app\servertransaction\ClientTransaction.java

ClientTransaction 作为调用activity周期的事务，内部结构如下

```java
public class ClientTransaction implements Parcelable, ObjectPoolItem {
    // 一个单独的回调客户端列表。比如通知客户端创建activity
    private List<ClientTransactionItem> mActivityCallbacks;
    // 表示当前事务最终的activity周期，比如onResume
    private ActivityLifecycleItem mLifecycleStateRequest;
    ....
}
```

#### schedule

schedule 调用IApplicationThread#scheduleTransaction 方法，将当前事务交给客户端

```java
private IApplicationThread mClient;
public void schedule() throws RemoteException {
     mClient.scheduleTransaction(this);
}
```

### 2.9 时序图

服务端流程对应的时序图如下：

<img src="/img/blog_start_activity_flow/AMS_server.png" width="100%" height="40%">

## 3、onCreate流程

现在流程又回到了客户端

### 3.1 ApplicationThread

ApplicationThread是客户端的代理类，主要是处理AMS端的请求。ActivityThread在启动的时候会将ApplicationThread 实现类传给AMS

```java
    final ApplicationThread mAppThread = new ApplicationThread();

	private void attach(boolean system, long startSeq) {
        ....
        final IActivityManager mgr = ActivityManager.getService();
        try {
            // 将ApplicationThread传给ActivityManagerService
            mgr.attachApplication(mAppThread, startSeq);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }
```

#### scheduleTransaction

```java
// 它管理应用程序进程中主线程的执行，根据活动管理器的请求，在其上调度和执行活动、广播和其他操作。
public final class ActivityThread extends android.app.ClientTransactionHandler {

    // 主要是处理AMS端的请求
     private class ApplicationThread extends IApplicationThread.Stub {
        ....
        @Override
        public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
            ActivityThread.this.scheduleTransaction(transaction);
        }
    }
}
```

### 3.2 ActivityThread

> 代码路径： frameworks\base\core\java\android\app\ActivityThread.java

#### scheduleTransaction

ApplicationThread 调用的实际是ClientTransactionHandler的scheduleTransaction方法

```java
void scheduleTransaction(ClientTransaction transaction) {
    transaction.preExecute(this);
    sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
}
```

#### sendMessage

ActivityThread实现了sendMessage的方法

```java
public final class ActivityThread extends android.app.ClientTransactionHandler {
    final H mH = new H();
    ....
    void sendMessage(int what, Object obj) {
        sendMessage(what, obj, 0, 0, false);
    }
    private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
        Message msg = Message.obtain();
        msg.what = what;
        msg.obj = obj;
        msg.arg1 = arg1;
        msg.arg2 = arg2;
        if (async) {
            // 发送异步消息
            msg.setAsynchronous(true);
        }
        mH.sendMessage(msg);
    }
}
```

这里用到了ActivityThread内部类H，H实际是一个handler，那就直接看handleMessage实现好了：

```java
 private final TransactionExecutor mTransactionExecutor = new TransactionExecutor(this);
 // 用于处理消息同步，将binder线程的消息同步到主线程中
 class H extends Handler {
     ....
     public static final int EXECUTE_TRANSACTION = 159;
     public void handleMessage(Message msg) {
            switch (msg.what) {
               case EXECUTE_TRANSACTION:
                    final ClientTransaction transaction = (ClientTransaction) msg.obj;
                    mTransactionExecutor.execute(transaction);
                    if (isSystem()) {
                        transaction.recycle();
                    }
                    break;
                    .....
            }
     }
 }
```

这里是把服务端传过来的transaction转交给了TransactionExecutor去执行

### 3.3 TransactionExecutor

> 代码路径： frameworks\base\core\java\android\app\servertransaction\TransactionExecutor.java

TransactionExecutor以正确顺序管理事务

#### execute

用于处理服务端传递过来的事物

```java
private ClientTransactionHandler mTransactionHandler;
public void execute(ClientTransaction transaction) {
    final IBinder token = transaction.getActivityToken();
    if (token != null) {
       // 处理销毁activity的事物
        final Map<IBinder, ClientTransactionItem> activitiesToBeDestroyed =
                mTransactionHandler.getActivitiesToBeDestroyed();
        final ClientTransactionItem destroyItem = activitiesToBeDestroyed.get(token);
        if (destroyItem != null) {
            if (transaction.getLifecycleStateRequest() == destroyItem) {
                // 它将使用令牌执行将销毁活动的事务，因此可以删除相应的待销毁记录。
                activitiesToBeDestroyed.remove(token);
            }
            if (mTransactionHandler.getActivityClient(token) == null) {
                 // 活动尚未创建，但已被请求销毁，因此令牌的所有交易就像被取消一样。
                Slog.w(TAG, tId(transaction) + "Skip pre-destroyed transaction:\n"
                        + transactionToString(transaction, mTransactionHandler));
                return;
            }
        }
    }
    // 处理事务的回调
    executeCallbacks(transaction);
    // 处理周期状态
    executeLifecycleState(transaction);
    mPendingActions.clear();
}
```

#### executeCallbacks

```java
// 循环遍历回调请求的所有状态并在适当的时间执行它们。
public void executeCallbacks(ClientTransaction transaction) {
    final List<ClientTransactionItem> callbacks = transaction.getCallbacks();
    final IBinder token = transaction.getActivityToken();
    ActivityClientRecord r = mTransactionHandler.getActivityClient(token);
    final ActivityLifecycleItem finalStateRequest = transaction.getLifecycleStateRequest();
    final int finalState = finalStateRequest != null ? finalStateRequest.getTargetState()
            : UNDEFINED;
    final int lastCallbackRequestingState = lastCallbackRequestingState(transaction);
    final int size = callbacks.size();
    // 遍历ClientTransactionItem
    for (int i = 0; i < size; ++i) {
        final ClientTransactionItem item = callbacks.get(i);
        final int postExecutionState = item.getPostExecutionState();
        final int closestPreExecutionState = mHelper.getClosestPreExecutionState(r,
                item.getPostExecutionState());
        if (closestPreExecutionState != UNDEFINED) {
            cycleToPath(r, closestPreExecutionState, transaction);
        }
        // 执行具体动作
        // mTransactionHandler 实际上是ActivityThread，具体可以见 ActivityThread成员变量mTransactionExecutor
        item.execute(mTransactionHandler, token, mPendingActions);
        item.postExecute(mTransactionHandler, token, mPendingActions);
        if (r == null) {
            // 启动活动请求将创建活动记录。
            r = mTransactionHandler.getActivityClient(token);
        }
        if (postExecutionState != UNDEFINED && r != null) {
            // 跳过最后一个转换并通过显式状态请求执行它。
            final boolean shouldExcludeLastTransition =
                    i == lastCallbackRequestingState && finalState == postExecutionState;
            cycleToPath(r, postExecutionState, shouldExcludeLastTransition, transaction);
        }
    }
}
```

在服务端提交事务的时候，通过 `clientTransaction.addCallback`方式将LaunchActivityItem添加到mActivityCallbacks里面（详情见服务端流程）

所以在遍历ClientTransactionItem过程中会取到LaunchActivityItem。

#### executeLifecycleState

该方法用于调用最终的周期，由AMS端的流程得知，lifecycleItem 对应是ResumeActivityItem

```java
    private void executeLifecycleState(ClientTransaction transaction) {
        final ActivityLifecycleItem lifecycleItem = transaction.getLifecycleStateRequest();
        final IBinder token = transaction.getActivityToken();
        final ActivityClientRecord r = mTransactionHandler.getActivityClient(token);
        // 补充对应的周期调用
        cycleToPath(r, lifecycleItem.getTargetState(), true /* excludeLastState */, transaction);
        // 执行最终的周期和对应的参数
        lifecycleItem.execute(mTransactionHandler, token, mPendingActions);
        // 主要处理收尾工作
        lifecycleItem.postExecute(mTransactionHandler, token, mPendingActions);
    }
```



### 3.4 LaunchActivityItem

> 代码路径： frameworks\base\core\java\android\app\servertransaction\LaunchActivityItem.java

#### execute

```java
// 请求发起一项活动。
public class LaunchActivityItem extends ClientTransactionItem {
    // client: 实际上是ActivityThread
    @Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
                mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
                mPendingResults, mPendingNewIntents, mIsForward,
                mProfilerInfo, client, mAssistToken, mFixedRotationAdjustments);
        // 又调用了ClientTransactionHandler的handleLaunchActivity
        client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
    }
}
```

那又回到了ActivityThread类的handleLaunchActivity方法

### 3.5 ActivityThread

#### handleLaunchActivity

```java
 //  活动启动的扩展实施。当服务器请求启动或重新启动时使用。
 @Override
 public Activity handleLaunchActivity(ActivityClientRecord r,
         PendingTransactionActions pendingActions, Intent customIntent) {
     // 提示 GraphicsEnvironment 活动正在进程中启动。
     GraphicsEnvironment.hintActivityLaunch();
     // 启动activity的核心实现
     final Activity a = performLaunchActivity(r, customIntent);
     if (a != null) {
         r.createdConfig = new Configuration(mConfiguration);
         reportSizeConfigurations(r);
         if (!r.activity.mFinished && pendingActions != null) {
             pendingActions.setOldState(r.state);
             pendingActions.setRestoreInstanceState(true);
             pendingActions.setCallOnPostCreate(true);
         }
     } else {
  		 // 如果出现错误，无论出于何种原因，通知activity manager阻止我们。
         try {
             android.app.ActivityTaskManager.getService()
                     .finishActivity(r.token, Activity.RESULT_CANCELED, null,
                             Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
         } catch (RemoteException ex) {
             throw ex.rethrowFromSystemServer();
         }
     }
     return a;
 }
```

#### performLaunchActivity

performLaunchActivity方法主要是负责创建activity

```java
// 活动启动的核心实现。
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
     ....
     ComponentName component = r.intent.getComponent();
     if (component == null) {
         // component 为null，则通过intent查询指定的component
         component = r.intent.resolveActivity(mInitialApplication.getPackageManager());
         r.intent.setComponent(component);
     }
     // 创建context对象
     ContextImpl appContext = createBaseContextForActivity(r);
     Activity activity = null;
     try {
         // 通过反射的方式创建activity实例
         ClassLoader cl = appContext.getClassLoader();
         activity = mInstrumentation.newActivity(
                 cl, component.getClassName(), r.intent);
         StrictMode.incrementExpectedActivityCount(activity.getClass());
         r.intent.setExtrasClassLoader(cl);
         r.intent.prepareToEnterProcess();
         if (r.state != null) {
             r.state.setClassLoader(cl);
         }
     } catch (Exception e) {
     }
     try {
         // 创建Application实例
         Application app = r.packageInfo.makeApplication(false, mInstrumentation);
         if (activity != null) {
             CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
             // 创建配置信息
             Configuration config = new Configuration(mCompatConfiguration);
             if (r.overrideConfig != null) {
                 config.updateFrom(r.overrideConfig);
             }
             // 用于给activity创建PhoneWindow
             Window window = null;
             if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                 window = r.mPendingRemoveWindow;
                 r.mPendingRemoveWindow = null;
                 r.mPendingRemoveWindowManager = null;
             }             // 设置Resource加载器
             appContext.getResources().addLoaders(
                     app.getResources().getLoaders().toArray(new ResourcesLoader[0]));             appContext.setOuterContext(activity);
             // 调用activity的attach ，传入context，ActivityThread等参数，初始化window相关的内容
             activity.attach(appContext, this, getInstrumentation(), r.token,
                     r.ident, app, r.intent, r.activityInfo, title, r.parent,
                     r.embeddedID, r.lastNonConfigurationInstances, config,
                     r.referrer, r.voiceInteractor, window, r.configCallback,
                     r.assistToken);            
             if (customIntent != null) {
                 activity.mIntent = customIntent;
             }
             r.lastNonConfigurationInstances = null;
             checkAndBlockForNetworkAccess();
             activity.mStartedActivity = false;
             // 设置主题
             int theme = r.activityInfo.getThemeResource();
             if (theme != 0) {
                 activity.setTheme(theme);
             }             activity.mCalled = false;
             // 通知Instrumentation调用activity的onCreate方法
             if (r.isPersistable()) {
                 mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
             } else {
                 mInstrumentation.callActivityOnCreate(activity, r.state);
             }
             r.activity = activity;
             mLastReportedWindowingMode.put(activity.getActivityToken(),
                     config.windowConfiguration.getWindowingMode());
         }
         r.setState(ON_CREATE);         // 保存token与ActivityClientRecord的映射
         synchronized (mResourcesManager) {
             mActivities.put(r.token, r);
         }
     } catch (SuperNotCalledException e) {
         throw e;
     }
    ....
    return activity;
}
```

流程到了Instrumentation 这里

### 3.6 Instrumentation 

> 代码路径： frameworks\base\core\java\android\app\Instrumentation.java

#### callActivityOnCreate

```java
public void callActivityOnCreate(Activity activity, Bundle icicle,
        PersistableBundle persistentState) {
    prePerformCreate(activity);
    activity.performCreate(icicle, persistentState);
    postPerformCreate(activity);
}

public void callActivityOnCreate(Activity activity, Bundle icicle) {
    prePerformCreate(activity);
    activity.performCreate(icicle);
    postPerformCreate(activity);
}
```

最后是调用了activity的performCreate方法，里面调用onCreate方法。至此，完成一次onCreate周期的调用

### 3.7 Activity

> 代码路径： frameworks\base\core\java\android\app\Activity.java

#### attach

attach 方法主要是初始化参数和创建window对象

```java
  final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken,
            IBinder shareableActivityToken) {
        attachBaseContext(context);
        // 创建window
        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        mWindow.setWindowControllerCallback(mWindowControllerCallback);
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        ...
        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        if (mParent != null) {
            mWindow.setContainer(mParent.getWindow());
        }
        mWindowManager = mWindow.getWindowManager();
        mCurrentConfig = config;

        mWindow.setColorMode(info.colorMode);
        mWindow.setPreferMinimalPostProcessing(
                (info.flags & ActivityInfo.FLAG_PREFER_MINIMAL_POST_PROCESSING) != 0);

        getAutofillClientController().onActivityAttached(application);
        setContentCaptureOptions(application.getContentCaptureOptions());
    }
```

#### performCreate

```java
final void performCreate(Bundle icicle) {
    performCreate(icicle, null);
}

@UnsupportedAppUsage
final void performCreate(Bundle icicle, PersistableBundle persistentState) {
    if (persistentState != null) {
        onCreate(icicle, persistentState);
    } else {
        // 即我们常重写的onCreate方法
        onCreate(icicle);
    }
    ....
    // 分发create 事件给fragment
    mFragments.dispatchActivityCreated();
    mActivityTransitionState.setEnterActivityOptions(this, getActivityOptions());
    dispatchActivityPostCreated(icicle);
    ....
}   
```

其他的生命周期调用方式类似，都是继承了ClientTransactionItem，具体流程见[framework之Activity 生命周期解析](https://weiwangqiang.github.io/2021/06/29/activity-lifecycle.markdown/)。

### 3.8 ResumeActivityItem

在executeLifecycleState 方法中，调用了ResumeActivityItem的execute和postExecute方法，这里不做过多解析，更多见[framework之Activity 生命周期解析](https://weiwangqiang.github.io/2021/06/29/activity-lifecycle.markdown/)。

#### execute

```java
@Override
public void execute(ClientTransactionHandler client, ActivityClientRecord r,
        PendingTransactionActions pendingActions) {
    // 调用activity的onResum
    client.handleResumeActivity(r, true /* finalStateRequest */, mIsForward,
            "RESUME_ACTIVITY");
}
```

#### postExecute

```java
 @Override
 public void postExecute(ClientTransactionHandler client, IBinder token,
         PendingTransactionActions pendingActions) {
     // 通知AMS，activity成功调用onResume周期
     ActivityClient.getInstance().activityResumed(token, client.isHandleSplashScreenExit(token));
 }
```

### 3.9 时序图

ApplicationThread调用activity的onCreate周期时序图如下：

<img src="/img/blog_start_activity_flow/activityThread.png" width="100%" height="40%">

## 4 小结

### 4.1 关于activity

一顿代码分析后，我们很容易发现，整个activity创建流程十分清晰（虽然代码量很大）。核心就是服务端（ATMS端）校验，客户端（App进程端）创建实例和执行调用周期的事务。

- 服务端：负责检查要启动方是否有权限启动、目标activity信息是否正确、准备创建activity的环境。
- 客户端：负责发起创建请求、处理请求结果、执行创建动作。

activity其实没那么神秘，离开AMS，它也是一个普通的类。只是AMS通过任务栈的切换逻辑，赋予了activity所谓的生命周期，看起来有了生命力。

### 4.2 关于插件化

由于沙盒机制，我们只能对自身进程进行相关Hook操作，没法Hook AMS端的逻辑。通常的插件化思路就是：在发起启动插件activity请求前，通过Hook技术，将目标activity替换成壳activity（预先在Androidmanifest中声明的空activity），并且在intent中保存插件activity的信息，用壳activity来欺骗ATMS。当客户端准备创建activity的时候，再把目标activity替换插件的，从而实现到动态加载activity的功能。

——Weiwq  于 2021.06 广州
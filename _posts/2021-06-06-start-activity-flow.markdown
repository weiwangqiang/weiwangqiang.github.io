---
layout:     post
title:      "framework之Activity启动流程（基于Android11源码）"
subtitle:   " \"带你细看Android11上activity的启动流程\""
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

Android11上，Activity的启动流程与Android10的实现（可以参考[Activity的启动过程详解（基于10.0源码）](https://cloud.tencent.com/developer/article/1666801)）又不一样了，但是万变不离其中，变的更多是代码上的优化。

如果不想看代码，可以直接看对应的时序图。

## 1、startActivity流程

### 代码分析

稍微了解activity的启动流程的读者应该知道，startActivity里面大概的工作：主要是通过Instrumentation用ActivityManagerService的代理对象，执行启动请求，中间涉及到binder的通信，如果大家不熟悉，可以看看笔者之前的文章 [Android跨进程之ADIL原理](https://weiwangqiang.github.io/2021/05/16/android-AIDL-analyze/) 。

那我们来看看Android11 又是如何实现的呢？

startActivity调用的是startActivityForResult方法，直接看startActivityForResult的实现好了

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

Instrumentation#execStartActivity 方法的主要实现如下。大概能看出，这里是直接拿到ActivityTaskManager 服务去startActivity。

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
    @Override
    public final int startActivity(IApplicationThread caller, String callingPackage,
            String callingFeatureId, Intent intent, String resolvedType, IBinder resultTo,
            String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo,
            Bundle bOptions) {
        return startActivityAsUser(caller, callingPackage, callingFeatureId, intent, resolvedType,
                resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions,
                UserHandle.getCallingUserId());
    }
}
```

那客户端又是怎么获取到IActivityTaskManager 的binder对象的呢？

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

所以，客户端是通过ServiceManager对象拿到指定的service，再进行相关通信

### 时序图

startActivity对应的时序图如下：

<img src="/img/blog_start_activity_flow/activity_clinet.png" width="100%" height="40%"> 

## 2、服务端流程

### 服务注册

ActivityTaskManagerService 内部有一个Lifecycle的类

```java

public class ActivityTaskManagerService extends IActivityTaskManager.Stub {
    ....
    public static final class Lifecycle extends SystemService {
        private final ActivityTaskManagerService mService;

        public Lifecycle(Context context) {
            super(context);
            mService = new ActivityTaskManagerService(context);
        }
        .... 

        @Override
        public void onStart() {
            publishBinderService(Context.ACTIVITY_TASK_SERVICE, mService);
            mService.start();
        }

        public ActivityTaskManagerService getService() {
            return mService;
        }
    }
}
```

SystemServer在启动的时候，会把Lifecycle 注册到SystemServiceManager里面，这样客户端就可以拿到对应的service

```java
public final class SystemServer {

    private void startBootstrapServices(@NonNull TimingsTraceAndSlog t) {
        ....
        ActivityTaskManagerService atm = mSystemServiceManager.startService(
                ActivityTaskManagerService.Lifecycle.class).getService();
        mActivityManagerService = ActivityManagerService.Lifecycle.startService(
                mSystemServiceManager, atm);
        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
        mActivityManagerService.setInstaller(installer);
        ....
    }
}
```

### 代码分析

与Android10上不一样的是，ActivityTaskManagerService 替换掉了ActivityManagerService的部分工作，负责activity的创建任务。

```java
// 用于管理活动及其容器（任务、堆栈、显示...）的系统服务。
public class ActivityTaskManagerService extends IActivityTaskManager.Stub {
    private ActivityStartController mActivityStartController;
    ....
    @Override
    public final int startActivity(IApplicationThread caller, String callingPackage,
            String callingFeatureId, Intent intent, String resolvedType, IBinder resultTo,
            String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo,
            Bundle bOptions) {
        return startActivityAsUser(caller, callingPackage, callingFeatureId, intent, resolvedType,
                resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions,
                UserHandle.getCallingUserId());
    }
    ....
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
    ....
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

```java
// 用于解释如何然后启动活动的控制器。此类收集用于确定意图和标志应如何转换为活动以及相关任务和堆栈的所有逻辑。
class ActivityStarter {
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
         ....
      }
    }
}
```

里面是直接把request对象给到了executeRequest 方法，在里面会做一些参数的检查

```java
class ActivityStarter {
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
                Trace.traceBegin(Trace.TRACE_TAG_WINDOW_MANAGER,
                        "shouldAbortBackgroundActivityStart");
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
                Trace.traceEnd(Trace.TRACE_TAG_WINDOW_MANAGER);
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
        // 创建ActivityRecord对象
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
}
```

`shouldAbortBackgroundActivityStart` 这个方法很重要，里面主要处理是否允许后台启动activity的逻辑（高版本对service启动activity做了限制，防止广告泛滥），详情可见上面的备注。

接着看startActivityUnchecked 实现

```java
class ActivityStarter {
    // 在大多数初步检查都已完成并且已确认呼叫者拥有必要的权限的情况下，开始活动。
    // 如果启动失败，这里还可以确保删除启动活动。
    private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
                                       IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                                       int startFlags, boolean doResume, ActivityOptions options, Task inTask,
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
}
```

startActivityUnchecked 没做什么事情，把启动的动作交给了startActivityInner方法

```java
class ActivityStarter {
    ....
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
            return startResult;
        }
        final ActivityStack topStack = mRootWindowContainer.getTopDisplayFocusedStack();
        if (topStack != null) {
            // 检查正在启动的活动是否与当前位于顶部的活动相同，并且应该只启动一次
            startResult = deliverToCurrentTopIfNeeded(topStack, intentGrants);
            if (startResult != START_SUCCESS) {
                return startResult;
            }
        }
        ....
        if (mTargetStack == null) {
           // 复用或者创建堆栈
            mTargetStack = getLaunchStack(mStartActivity, mLaunchFlags, targetTask, mOptions);
        }
        if (newTask) {
            // 新建一个task
            final Task taskToAffiliate = (mLaunchTaskBehind && mSourceRecord != null)
                    ? mSourceRecord.getTask() : null;
            setNewTask(taskToAffiliate);
            if (mService.getLockTaskController().isLockTaskModeViolation(
                    mStartActivity.getTask())) {
                Slog.e(TAG, "Attempted Lock Task Mode violation mStartActivity=" + mStartActivity);
                return START_RETURN_LOCK_TASK_MODE_VIOLATION;
            }
        } else if (mAddingToTask) {
            // 复用之前的task
            addOrReparentStartingActivity(targetTask, "adding to task");
        }
        ....
        // 检查是否需要触发过渡动画和开始窗口
        mTargetStack.startActivityLocked(mStartActivity, topStack.getTopNonFinishingActivity(),
                newTask, mKeepCurTransition, mOptions);
        if (mDoResume) {
           ....
           mRootWindowContainer.resumeFocusedStacksTopActivities(
                        mTargetStack, mStartActivity, mOptions);
           ....
        }
        ....
        return START_SUCCESS;
    }
}
```

startActivityInner 负责的任务就是准备好堆栈，为启动activity做最后的准备。

接下来就把流程交给了RootWindowContainer的resumeFocusedStacksTopActivities方法

```java

class RootWindowContainer extends WindowContainer<DisplayContent> implements DisplayManager.DisplayListener {
      boolean resumeFocusedStacksTopActivities(ActivityStack targetStack, 
                                               ActivityRecord target, ActivityOptions targetOptions) {
        ....
        boolean result = false;
        if (targetStack != null && (targetStack.isTopStackInDisplayArea()
                || getTopDisplayFocusedStack() == targetStack)) {
            result = targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
        }
        ....
        return result;
}

```

resumeFocusedStacksTopActivities 接着交给了ActivityStack的resumeTopActivityUncheckedLocked 方法。

```java
// 单个活动堆栈的状态和管理。
class ActivityStack extends Task {
    ....
    // 确保恢复堆栈中的顶部活动。
    // @param prev 之前恢复的活动，用于暂停过程中；从别处调用的时候可以为 null 
    // @param options 活动选项。
    // 注意：直接调用此方法是不安全的，因为它会导致非焦点堆栈中的活动被恢复。
    //     需要使用 {@link RootWindowContainer#resumeFocusedStacksTopActivities} 恢复当前系统状态的正确活动。
    boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
        ....
        boolean result = false;
        try {
            // Protect against recursion.
            mInResumeTopActivity = true;
            result = resumeTopActivityInnerLocked(prev, options);
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
}

```

流程来到了resumeTopActivityInnerLocked

```java
class ActivityStack extends Task {
      private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
        
        // 在此堆栈中找到下一个最顶层的活动，该活动尚未完成且可聚焦。如果它不是可聚焦的，我们将陷入下面的情况，
        // 在下一个可聚焦的任务中恢复最上面的活动。
        ActivityRecord next = topRunningActivity(true /* focusableOnly */);
        final boolean hasRunningActivity = next != null;
       if (next.attachedToProcess()) {
          ....
       } else {
         // Whoops, need to restart this activity!
         ....
         mStackSupervisor.startSpecificActivity(next, true, true);
       }       
        
      }
}

```

这个方法的实现特别长，主要是处理前一个、下一个activity显示的逻辑，直接看startSpecificActivity的实现

mStackSupervisor 是Task的一个成员变量，对应是ActivityStackSupervisor类

```java
// 这个类后面估计也会被移除
public class ActivityStackSupervisor implements RecentTasks.Callbacks {
    final ActivityTaskManagerService mService;
    ....
    void startSpecificActivity(ActivityRecord r, boolean andResume, boolean checkConfig) {
        // 此活动的应用程序是否已在运行？
        final com.android.server.wm.WindowProcessController wpc =
                mService.getProcessController(r.processName, r.info.applicationInfo.uid);
        boolean knownToBeDead = false;
        if (wpc != null && wpc.hasThread()) {
            try {
                // 真正启动activity的地方
                realStartActivityLocked(r, wpc, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting activity "
                        + r.intent.getComponent().flattenToShortString(), e);
            }
            // If a dead object exception was thrown -- fall through to
            // restart the application.
            knownToBeDead = true;
        }
        r.notifyUnknownVisibilityLaunchedForKeyguardTransition();
        final boolean isTop = andResume && r.isTopRunningActivity();
        // 启动失败，对应的应用程序没有运行，就创建一个进程
        // 这里的mService即ActivityTaskManagerService
        mService.startProcessAsync(r, knownToBeDead, isTop, isTop ? "top-activity" : "activity");
    }
   
    boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
                                    boolean andResume, boolean checkConfig) throws RemoteException {
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
        //  安排一个事务
        mService.getLifecycleManager().scheduleTransaction(clientTransaction);
    }
}

```

getLifecycleManager() 返回的是一个ClientLifecycleManager对象

```java
// 能够组合多个客户端生命周期转换请求和/或回调的类，并将它们作为单个事务执行。
class ClientLifecycleManager {
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
}
```

最终ClientLifecycleManager 把创建activity事务提交给了客户端的ApplicationThread类。

### 时序图

服务端流程对应的时序图如下：

<img src="/img/blog_start_activity_flow/AMS_server.png" width="100%" height="40%">

## 3、onCreate流程

### 代码分析

现在流程又回到了客户端

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

这里ApplicationThread 调用的实际是ClientTransactionHandler的scheduleTransaction方法

```java
// 定义 {@link android.app.servertransaction.ClientTransaction} 或其Items可以在客户端上执行的操作。
public abstract class ClientTransactionHandler {

    // Schedule phase related logic and handlers.

    /** Prepare and schedule transaction for execution. */
    void scheduleTransaction(ClientTransaction transaction) {
        transaction.preExecute(this);
        sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
    }
    abstract void sendMessage(int what, Object obj);
    ....
}
```

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
public final class ActivityThread extends android.app.ClientTransactionHandler {
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
                             // Client transactions inside system process are recycled on the client side
                             // instead of ClientLifecycleManager to avoid being cleared before this
                             // message is handled.
                             transaction.recycle();
                         }
                         // TODO(lifecycler): Recycle locally scheduled transactions.
                         break;
                         .....
                 }
          }
      }
}
```

这里是把服务端传过来的transaction转交给了TransactionExecutor去执行

```java
// 以正确顺序管理事务执行的类。
public class TransactionExecutor {
    private ClientTransactionHandler mTransactionHandler;

    public TransactionExecutor(ClientTransactionHandler clientTransactionHandler) {
        mTransactionHandler = clientTransactionHandler;
    }
 
    // 处理服务端传递过来的事物
    public void execute(ClientTransaction transaction) {
        if (DEBUG_RESOLVER) Slog.d(TAG, tId(transaction) + "Start resolving transaction");

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
        executeCallbacks(transaction);
        executeLifecycleState(transaction);
        mPendingActions.clear();
        if (DEBUG_RESOLVER) Slog.d(TAG, tId(transaction) + "End resolving transaction");
    }
}
```

看看executeCallbacks的实现

```java
public class TransactionExecutor {
    private ClientTransactionHandler mTransactionHandler;

    public TransactionExecutor(ClientTransactionHandler clientTransactionHandler) {
        mTransactionHandler = clientTransactionHandler;
    }
 
    // 循环遍历回调请求的所有状态并在适当的时间执行它们。
    public void executeCallbacks(ClientTransaction transaction) {
        final List<ClientTransactionItem> callbacks = transaction.getCallbacks();
        if (callbacks == null || callbacks.isEmpty()) {
            // No callbacks to execute, return early.
            return;
        }
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
}
```

在服务端提交事务的时候，通过 `clientTransaction.addCallback`方式将LaunchActivityItem添加到mActivityCallbacks里面（详情见服务端流程）

所以在遍历ClientTransactionItem过程中会取到LaunchActivityItem。

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

```java
public final class ActivityThread extends android.app.ClientTransactionHandler {

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
}

```

performLaunchActivity方法主要是负责创建activity

```java
public final class ActivityThread extends android.app.ClientTransactionHandler {   
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
                }

                // 设置Resource加载器
                appContext.getResources().addLoaders(
                        app.getResources().getLoaders().toArray(new ResourcesLoader[0]));

                appContext.setOuterContext(activity);
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
                }

                activity.mCalled = false;
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
            r.setState(ON_CREATE);

            // 保存token与ActivityClientRecord的映射
            synchronized (mResourcesManager) {
                mActivities.put(r.token, r);
            }
        } catch (SuperNotCalledException e) {
            throw e;
        }
       ....
       return activity;
   }
}
```

流程到了Instrumentation 这里

```java
public class Instrumentation {
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
    ....
}
```

最后是调用了activity的performCreate方法，里面调用onCreate方法。至此，完成一次onCreate周期的调用

```java

public class Activity extends ContextThemeWrapper
        implements LayoutInflater.Factory2,
        Window.Callback, KeyEvent.Callback,
        OnCreateContextMenuListener, ComponentCallbacks2,
        Window.OnWindowDismissedCallback,
        AutofillManager.AutofillClient, ContentCaptureManager.ContentCaptureClient {
       
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
    ....  
}

```

其他的生命周期调用方式类似，都是继承了ClientTransactionItem，实现对应的周期调用。后面会出一篇文章单独讲解。

### 时序图

ApplicationThread调用activity的onCreate周期时序图如下：

<img src="/img/blog_start_activity_flow/activityThread.png" width="100%" height="40%">

## 3、小结

### 关于activity

一顿代码分析后，我们很容易发现，整个activity创建流程十分清晰（虽然代码量很大）。核心就是服务端（ATMS端）校验，客户端（App进程端）创建实例和执行调用周期的事务。

- 服务端：负责检查要启动方是否有权限启动、目标activity信息是否正确、准备创建activity的环境。
- 客户端：负责发起创建请求、处理请求结果、执行创建动作。

activity其实没那么神秘，离开AMS，它也是一个普通的类。只是AMS通过任务栈的切换逻辑，赋予了activity所谓的生命周期，看起来有了生命力。

### 关于插件化

由于沙盒机制，我们只能对自身进程进行相关Hook操作，没法Hook AMS端的逻辑。通常的插件化思路就是：在发起启动插件activity请求前，通过Hook技术，将目标activity替换成壳activity（预先在Androidmanifest中声明的空activity），并且在intent中保存插件activity的信息，用壳activity来欺骗ATMS。当客户端准备创建activity的时候，再把目标activity替换插件的，从而实现到动态加载activity的功能。

——Weiwq  于 2021.06 广州
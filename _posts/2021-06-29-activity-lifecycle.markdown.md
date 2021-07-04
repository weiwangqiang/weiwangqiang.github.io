---
layout:     post
title:      "framework之Activity 生命周期解析（基于Android11源码）"
subtitle:   " \"activity的生命周期，是如此朴实无华\""
date:       2021-06-29 22:08:00
author:     "Weiwq"
header-img: "img/background/post-bg-android-1.jpg"
catalog:  true
tags:
    - Android

---

> 上一篇讲了activity的创建过程，那这一篇就讲讲activity的生命周期

## 引言

上一篇讲了activity的创建过程（没看过的小伙伴移步 [<font color="red">点我前往</font>](https://weiwangqiang.github.io/2021/06/08/start-activity-flow/)），那这一篇就讲讲activity的生命周期。

在高版本上，activity的周期都是以事务的方式调用，activityThread里面H类的`EXECUTE_TRANSACTION` 消息正是接收、处理事务的入口，实际最终由TransactionExecutor 处理该事务。（PS：ATMS即ActivityTaskManagerService的简写）

```java
public final class ActivityThread extends ClientTransactionHandler {
    private final TransactionExecutor mTransactionExecutor = new TransactionExecutor(this);
    class H extends Handler {      
             public void handleMessage(Message msg) {
                  case EXECUTE_TRANSACTION:
                        final ClientTransaction transaction = (ClientTransaction) msg.obj;
                        mTransactionExecutor.execute(transaction);
                        if (isSystem()) {
                            transaction.recycle();
                        }
                   .....
             }
        }
}

```

我们来看看整个activity启动过程会走哪些周期，请记住下面的周期和顺序，接下来就会讲讲里面的调用逻辑了

```java
06-20 13:58:04.476 17073 17073 D MainActivity: onCreate:
06-20 13:58:04.481 17073 17073 D MainActivity: onStart:
06-20 13:58:04.483 17073 17073 D MainActivity: onPostCreate:
06-20 13:58:04.483 17073 17073 D MainActivity: onResume:
06-20 13:58:04.494 17073 17073 D MainActivity: onPostResume:
06-20 13:58:08.690 17073 17073 D MainActivity: onPause:
06-20 13:58:10.207 17073 17073 D MainActivity: onStop:
06-20 13:58:10.210 17073 17073 D MainActivity: onDestroy:

```

## 1、onCreate

[framework之Activity启动流程](https://weiwangqiang.github.io/2021/06/08/start-activity-flow/)，里面已经很详细描述了onCreate的调用流程，对应的流程如下

<img src="/img/blog_start_activity_flow/activityThread.png" width="100%" height="40%">

简单来讲，通过LaunchActivityItem调用了ActivityThread的handleLaunchActivity方法，ActivityThread在创建activity实例后，会设置config、window、resource、theme相关资源，在调用activity的attach方法后，会接着调用onCreate方法。

## 2、onStart

onCreate之后，onStart又是如何调用的呢？这里我们回顾一下TransactionExecutor#execute方法

```java
// 以正确顺序管理事务执行的类。
public class TransactionExecutor {
   ....
    // 处理服务端传递过来的事务
    public void execute(ClientTransaction transaction) {
        ....
        // 该方法中通过遍历 transaction#callbacks 获取到LaunchActivityItem，然后调用onCreate方法
        executeCallbacks(transaction);
        executeLifecycleState(transaction);
    }
}
```

不知读者发现没有，executeLifecycleState方法还没分析过，很好，续杯 executeLifecycleState！

```java

public class TransactionExecutor {
     // 将请求的事务转为最终的生命周期
    private void executeLifecycleState(ClientTransaction transaction) {
        ....
        final IBinder token = transaction.getActivityToken();
        // 通过token获取到对应的activityRecord
        final ActivityClientRecord r = mTransactionHandler.getActivityClient(token);
        // 循环到最终请求状态之前的状态。
        cycleToPath(r, lifecycleItem.getTargetState(), true /* excludeLastState */, transaction);
        // 使用适当的参数执行最终转换。
        lifecycleItem.execute(mTransactionHandler, token, mPendingActions);
        lifecycleItem.postExecute(mTransactionHandler, token, mPendingActions);
    }
}
```

看看cycleToPath方法，这里在拿到lifeCyclePath后就交给了performLifecycleSequence

```java
// 循环到最终请求状态之前的状态。
private void cycleToPath(ActivityClientRecord r, int finish, boolean excludeLastState,
            ClientTransaction transaction) {
        final int start = r.getLifecycleState();
        // 计算活动的主要生命周期状态的路径，并使用从初始状态之后的状态开始的值填充
        // 比如onStart,onStop周期就是在这里额外加入的
        final IntArray path = mHelper.getLifecyclePath(start, finish, excludeLastState);
        performLifecycleSequence(r, path, transaction);
}

```

我们来到了performLifecycleSequence 方法，看起来是处理全部周期的地方，问题来了，我们怎么知道走到哪个case？

```java

public class TransactionExecutor {
    private PendingTransactionActions mPendingActions = new PendingTransactionActions();
    private ClientTransactionHandler mTransactionHandler;
    ...
    // 通过之前的序列状态过渡为客户端的状态
    private void performLifecycleSequence(ActivityClientRecord r, IntArray path,
            ClientTransaction transaction) {
        final int size = path.size();
        for (int i = 0, state; i < size; i++) {
              state = path.get(i);
              switch (state) {
                case ON_CREATE:
                    mTransactionHandler.handleLaunchActivity(r, mPendingActions,
                            null /* customIntent */);
                    break;
                case ON_START:
                    mTransactionHandler.handleStartActivity(r.token, mPendingActions);
                    break;
                case ON_RESUME:
                    mTransactionHandler.handleResumeActivity(r.token, false /* finalStateRequest */,
                            r.isForward, "LIFECYCLER_RESUME_ACTIVITY");
                    break;
                case ON_PAUSE:
                    mTransactionHandler.handlePauseActivity(r.token, false /* finished */,
                            false /* userLeaving */, 0 /* configChanges */, mPendingActions,
                            "LIFECYCLER_PAUSE_ACTIVITY");
                    break;
                case ON_STOP:
                    mTransactionHandler.handleStopActivity(r.token, 0 /* configChanges */,
                            mPendingActions, false /* finalStateRequest */,
                            "LIFECYCLER_STOP_ACTIVITY");
                    break;
                case ON_DESTROY:
                    mTransactionHandler.handleDestroyActivity(r.token, false /* finishing */,
                            0 /* configChanges */, false /* getNonConfigInstance */,
                            "performLifecycleSequence. cycling to:" + path.get(size - 1));
                    break;
                case ON_RESTART:
                    mTransactionHandler.performRestartActivity(r.token, false /* start */);
                    break;
                   ....
            }
        }
    }
}

public abstract class ActivityLifecycleItem extends ClientTransactionItem {
    ....
    public static final int ON_CREATE = 1;
    public static final int ON_START = 2;
    public static final int ON_RESUME = 3;
    public static final int ON_PAUSE = 4;
    public static final int ON_STOP = 5;
    public static final int ON_DESTROY = 6;
    public static final int ON_RESTART = 7;
}
```

通过debug发现，path的size是1，mValues的第一个值是2，即state为2，显然走到ON_START case，调用mTransactionHandler#handleStartActivity方法。

<img src="/img/blog_activity_lifecycle/1.png" width="100%" height="40%">

ClientTransactionHandler其实是一个抽象类，ActivityThread才是具体实现类

```java
public final class ActivityThread extends ClientTransactionHandler {
    
    @Override
    public void handleStartActivity(IBinder token, PendingTransactionActions pendingActions) {
        final ActivityClientRecord r = mActivities.get(token);
        final Activity activity = r.activity;
        // Start，即调用start周期
        activity.performStart("handleStartActivity");
        // 更新当前的状态
        r.setState(ON_START);
        // 恢复实例的状态，即OnRestoreInstanceState周期
        if (pendingActions.shouldRestoreInstanceState()) {
            if (r.isPersistable()) {
               mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                            r.persistentState);
            } else if (r.state != null) {
                mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
            }
        }
        // 调用 postOnCreate() 周期
        if (pendingActions.shouldCallOnPostCreate()) {
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnPostCreate(activity, r.state,
                        r.persistentState);
            } else {
                mInstrumentation.callActivityOnPostCreate(activity, r.state);
            }
        }
        // 将activity设置为可见
        updateVisibility(r, true /* show */);
    }
    // 设置activity的可见状态
    private void updateVisibility(ActivityClientRecord r, boolean show) {
        View v = r.activity.mDecor;
        if (v != null) {
            if (show) {
                if (r.newConfig != null) {
                    // 回调onConfigurationChanged方法
                    performConfigurationChangedForActivity(r, r.newConfig);
                    r.newConfig = null;
                }
            }
            ....
        }
    }
}
```

从上面可以看到，handleStartActivity 方法主要调用`onStart`、`OnRestoreInstanceState`和 `postOnCreate` 周期。并在最后将activity设置为可见

```java
public class Activity extends ContextThemeWrapper .... {
    // start 周期的入口
    final void performStart(String reason) {
        // 将onStart周期开始事件分发给监听器ActivityLifecycleCallbacks
        dispatchActivityPreStarted();
        // 将onStart的逻辑交给mInstrumentation
        mInstrumentation.callActivityOnStart(this);
        // 将周期分发给fragment
        mFragments.dispatchStart();
        ...
        // 将onSart周期结束事件分发给监听器ActivityLifecycleCallbacks
        dispatchActivityPostStarted();
    }
}

```

Instrumentation 也只是调用activity的onStart方法，这样做的好处是，可以将周期调用的时机暴露出去。

```java
public class Instrumentation {
    public void callActivityOnStart(Activity activity) {
        activity.onStart();
    }
}
```

### onRestart

至于onRestart周期，这里也贴一下对应的path值

<img src="/img/blog_activity_lifecycle/6.png" width="100%" height="40%">

其实就是在onStart的前面插入了onStart 下标值，对应的case调用的是performRestartActivity方法

```java
public final class ActivityThread extends ClientTransactionHandler {
    
    @Override
    public void performRestartActivity(IBinder token, boolean start) {
        ActivityClientRecord r = mActivities.get(token);
        if (r.stopped) {
            r.activity.performRestart(start, "performRestartActivity");
            if (start) {
                r.setState(ON_START);
            }
        }
    }
}
```

Activity对应的方法是

```java
public class Activity extends ContextThemeWrapper .... {
      final void performRestart(boolean start, String reason) {
            ....
            // 直接调用onRestart周期
            mInstrumentation.callActivityOnRestart(this);
       }
}
```

## 3、onResume

通过分析onStart周期调用，估计有些同学很快就会想到onResume也是在performLifecycleSequence方法中处理的。因为里面也有ON_RESUME的case。这么想就 大错特错了！！！

上面有debug出path的size是1，所以在处理onStart周期后，就会退出循环。那onResume是在哪里调用的呢？

我们回到executeLifecycleState方法，方法最后调用了lifecycleItem.execute。

```java
public class TransactionExecutor {
     // 将请求的事务转为最终的生命周期
    private void executeLifecycleState(ClientTransaction transaction) {
        final ActivityLifecycleItem lifecycleItem = transaction.getLifecycleStateRequest();
        ....
        final IBinder token = transaction.getActivityToken();
        // 通过token获取到对应的activityRecord
        final ActivityClientRecord r = mTransactionHandler.getActivityClient(token);
        // 循环到最终请求状态之前的状态。
        cycleToPath(r, lifecycleItem.getTargetState(), true /* excludeLastState */, transaction);
        // 使用适当的参数执行最终转换。
        lifecycleItem.execute(mTransactionHandler, token, mPendingActions);
        // 通知ActivityTaskManager
        lifecycleItem.postExecute(mTransactionHandler, token, mPendingActions);
    }
}
```

而ActivityLifecycleItem是一个抽象类，那就再次debug看看lifecycleItem是哪个具体类吧

<img src="/img/blog_activity_lifecycle/2.png" width="100%" height="40%">

lifecycleItem是ResumeActivityItem的实例。

```java
public class ResumeActivityItem extends ActivityLifecycleItem {
    .....
    @Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        // 调用了ActivityThread的handleResumeActivity
        client.handleResumeActivity(token, true /* finalStateRequest */, mIsForward,
                "RESUME_ACTIVITY");
    }
    
    @Override
    public void postExecute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        try {
            // 通知ATMS 当前完成onResume
            ActivityTaskManager.getService().activityResumed(token);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }
}
```

很好，又回到了ActivityThread（这家伙出镜率相当高）

```java
public final class ActivityThread extends ClientTransactionHandler {
     @Override
    public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
            String reason) {
        // 准备调用Resume周期
        final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
        ....
    }
}
```

handleResumeActivity 方法里面调用了performResumeActivity 

```java
public final class ActivityThread extends ClientTransactionHandler {
    public ActivityClientRecord performResumeActivity(IBinder token, boolean finalStateRequest,
            String reason) {
        final ActivityClientRecord r = mActivities.get(token);
        ....
        try {
            // 调用activity的performResume
             r.activity.performResume(r.startsNotResumed, reason);
             ....
        } catch (Exception e) {
             ....
        }
    }
}

```

最后来到了activity的performResume，调用activity的onResume

```java
public class Activity extends ContextThemeWrapper .... {
        final void performResume(boolean followedByPause, String reason) {
            // 将onResume开始事件分发给监听器ActivityLifecycleCallbacks
             dispatchActivityPreResumed();
             // 调用Activity的onResume方法
             mInstrumentation.callActivityOnResume(this);
             // 将OnResume周期分发给fragment
             mFragments.dispatchResume();
             // 调用onPostResume 周期
             onPostResume();
             // 将onResume结束的事件分发给监听器ActivityLifecycleCallbacks
             dispatchActivityPostResumed();
        }
}
```

## 4、中场小结

经过上面的分析，想必大家对activity周期调用链路有了一定的了解（绕晕了 doge），其中最重要的地方是这里

```java
public class TransactionExecutor {
      // 是负责activity周期调用的总入口，处理ATMS 的周期事务请求，其中transaction 是ATMS传过来的参数
       public void execute(ClientTransaction transaction) {
        final IBinder token = transaction.getActivityToken();
        if (token != null) {
            // 处理activity onDestory的情况
            final Map<IBinder, ClientTransactionItem> activitiesToBeDestroyed =
                    mTransactionHandler.getActivitiesToBeDestroyed();
            final ClientTransactionItem destroyItem = activitiesToBeDestroyed.get(token);
            if (destroyItem != null) {
                if (transaction.getLifecycleStateRequest() == destroyItem) {
                    activitiesToBeDestroyed.remove(token);
                }
                if (mTransactionHandler.getActivityClient(token) == null) {
                    return;
                }
            }
        }
        // 周期回调主要是在这两个方法中
        executeCallbacks(transaction);
        executeLifecycleState(transaction);
       }
}
```

我们先看execute方法参数ClientTransaction的结构，里面有两个十分重要的成员变量

```java
public class ClientTransaction implements Parcelable, ObjectPoolItem {
    // 对客户端的单个回调列表。
    private List<ClientTransactionItem> mActivityCallbacks;
    // 执行事务后客户端活动应处于的最终生命周期状态。
    private ActivityLifecycleItem mLifecycleStateRequest;
    ....
}
```

ClientTransactionItem 又是干啥的？

```java
// 可以调度和执行的客户端回调消息。这些示例可能是活动配置更改、多窗口模式更改、活动结果交付等。
public abstract class ClientTransactionItem implements BaseClientRequest, Parcelable {
    @LifecycleState
    public int getPostExecutionState() {
        return UNDEFINED;
    }
    @Override
    public int describeContents() {
        return 0;
    }
}
// 从服务器到客户端的单个请求的基本接口。它们中的每一个都可以在调度之前准备好，并最终执行。
public interface BaseClientRequest extends ObjectPoolItem {
    // 在调度之前准备客户端请求。这方面的一个示例可能是通知某些值的挂起更新。
    default void preExecute(ClientTransactionHandler client, IBinder token) {
    }
    // 执行请求。
    void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions);
    // 执行执行后需要发生的所有操作，例如将结果报告给server 端。
    default void postExecute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
    }
}

```

而ActivityLifecycleItem 是继承ClientTransactionItem。ActivityLifecycleItem就多了一个getTargetState方法

```java
public abstract class ActivityLifecycleItem extends ClientTransactionItem {
    // 生命周期的下标
    public static final int UNDEFINED = -1;
    public static final int PRE_ON_CREATE = 0;
    public static final int ON_CREATE = 1;
    public static final int ON_START = 2;
    public static final int ON_RESUME = 3;
    public static final int ON_PAUSE = 4;
    public static final int ON_STOP = 5;
    public static final int ON_DESTROY = 6;
    public static final int ON_RESTART = 7;

    // 活动应达到的最终生命周期状态。
    @LifecycleState
    public abstract int getTargetState();
}
```

我们看看ActivityLifecycleItem的子类，会发现有一些关键的周期实现

<img src="/img/blog_activity_lifecycle/3.png" width="100%" height="40%">

结合上面ResumeActivityItem的实现，可以看出ActivityLifecycleItem的子类是负责client端**`最终周期`**调用和周期上报。这里提出到的**`最终周期`** 以及 executeLifecycleState 方法也备注了**`最终周期`**，是什么意思呢？

我们都知道activity在创建的时候，会一口气走完onCreate、onstart、onResume方法（不知道的看上面周期分析！！！！）。但是在退出的时候就显得有点小气了，先走onPause、过了一会才走onStop、onDestory。这里的onResume、onPause、onDestory就是最终周期，ATMS在发周期事务的时候，就是直接发这些状态(即mLifecycleStateRequest 变量)。至于中间的周期，比如onCreate、onStart就会插在这个事务的过程中调用。

我们回顾一下onCreate的调用：在TransactionExecutor的executeCallbacks方法里面，通过遍历mActivityCallbacks 获取到LaunchActivityItem，由它调用ActivityThread的handleLaunchActivity方法，从而触发onCreate。而onStart则是通过performLifecycleSequence 方法调用ActivityThread的handleStartActivity。

## 5、onPause

好的，通过上面的小结，大家应该知道onPause周期如何分析了，我们直接看PauseActivityItem的实现

```java
public class PauseActivityItem extends ActivityLifecycleItem {
    ....
    @Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        // 调用ActivityThread的handlePauseActivity
        client.handlePauseActivity(token, mFinished, mUserLeaving, mConfigChanges, pendingActions,
                "PAUSE_ACTIVITY_ITEM");
    }

    @Override
    public void postExecute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        try {
            //将当前的周期状态告诉ActivityTaskManagerService
            ActivityTaskManager.getService().activityPaused(token);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }
}
```

来到了ActivityThread的handlePauseActivity方法

```java
public final class ActivityThread extends ClientTransactionHandler {
   @Override
    public void handlePauseActivity(IBinder token, boolean finished, boolean userLeaving,
            int configChanges, PendingTransactionActions pendingActions, String reason) {
        ActivityClientRecord r = mActivities.get(token);
        if (r != null) {
            if (userLeaving) {
                performUserLeavingActivity(r);
            }
            r.activity.mConfigChangeFlags |= configChanges;
            performPauseActivity(r, finished, reason, pendingActions);
            ....
        }
    }
    // 准备调用onPause方法
     private Bundle performPauseActivity(ActivityClientRecord r, boolean finished, String reason,
            PendingTransactionActions pendingActions) {
        ....
        final boolean shouldSaveState = !r.activity.mFinished && r.isPreHoneycomb();
        if (shouldSaveState) {
            // 调用onSaveInstanceState周期
            callActivityOnSaveInstanceState(r);
        }
        performPauseActivityIfNeeded(r, reason);
         .....
        return shouldSaveState ? r.state : null;
    }
    // 调用callActivityOnPause 方法
    private void performPauseActivityIfNeeded(ActivityClientRecord r, String reason) {
        ....
        try {
            r.activity.mCalled = false;
            mInstrumentation.callActivityOnPause(r.activity);
        } catch (SuperNotCalledException e) {
            .....
        }
        r.setState(ON_PAUSE);
    }
}
```

Instrumentation的callActivityOnPause方法是直接掉用activity的performPause

```java
public class Activity extends ContextThemeWrapper{
    final void performPause() {
        mFragments.dispatchPause();
        mCalled = false;
        // 调用onPause方法
        onPause();
    }
}
```

## 6、onStop

在处理完onPause后，ATMS实际发的是OnDestory事务，可以通过debug方式看出来

<img src="/img/blog_activity_lifecycle/5.png" width="100%" height="40%">

那onStop周期的事务跑哪去了？在分析onStart周期的时候，有提到 mHelper.getLifecyclePath 这么一个接口，主要的作用就是帮ATMS填充缺省的周期，比如onStart和onStop：

```java
public class TransactionExecutorHelper {
    // 计算活动的主要生命周期状态的路径，并使用从初始状态之后的状态开始的值填充
    public IntArray getLifecyclePath(int start, int finish, boolean excludeLastState) {
        mLifecycleSequence.clear();
        if (finish >= start) {
            if (start == ON_START && finish == ON_STOP) {
                // A case when we from start to stop state soon, we don't need to go
                // through the resumed, paused state.
                mLifecycleSequence.add(ON_STOP);
            } else {
                // just go there
                for (int i = start + 1; i <= finish; i++) {
                    mLifecycleSequence.add(i);
                }
            }
        } 
        ....
        return mLifecycleSequence;
    }
}
```

通过debug我们也可以看出来

<img src="/img/blog_activity_lifecycle/4.png" width="100%" height="40%">

```java
   private void performLifecycleSequence(ActivityClientRecord r, IntArray path, ClientTransaction transaction) {
           switch (state) {
                case ON_STOP:
                    mTransactionHandler.handleStopActivity(r.token, 0 /* configChanges */,
                            mPendingActions, false /* finalStateRequest */,
                            "LIFECYCLER_STOP_ACTIVITY");
                 break;
                ....
           }
   }
```

这就回到了ActivityThread的handleStopActivity方法

```java
public final class ActivityThread extends ClientTransactionHandler {
    @Override
    public void handleStopActivity(IBinder token, int configChanges,
            PendingTransactionActions pendingActions, boolean finalStateRequest, String reason) {
        final ActivityClientRecord r = mActivities.get(token);
        r.activity.mConfigChangeFlags |= configChanges;
        final StopInfo stopInfo = new StopInfo();
        performStopActivityInner(r, stopInfo, true /* saveState */, finalStateRequest,
                reason);
      // 将activity设置为不可见
      updateVisibility(r, false);
    }
        
    private void performStopActivityInner(ActivityClientRecord r, StopInfo info,
            boolean saveState, boolean finalStateRequest, String reason) {
           .....
           callActivityOnStop(r, saveState, reason);
    }
   
    private void callActivityOnStop(ActivityClientRecord r, boolean saveState, String reason) {
        try {
            r.activity.performStop(r.mPreserveWindow, reason);
        } catch (SuperNotCalledException e) {
             ....
        }
   }
}
```

最后到了Activity的performStop方法

```java
public class Activity extends ContextThemeWrapper{
     final void performStop(boolean preserveWindow, String reason) {
           mFragments.dispatchStop();
         // 直接调用activity的onStop方法
          mInstrumentation.callActivityOnStop(this);
     }
}
```

## 7、onDestory

在分析onStop的时候，通过debug知道ATMS传过来的lifecycleItem是DestroyActivityItem。注意，这里并没有将执行结果通知ATMS。

```java
public class DestroyActivityItem extends ActivityLifecycleItem {
    
    @Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        client.handleDestroyActivity(token, mFinished, mConfigChanges,
                false /* getNonConfigInstance */, "DestroyActivityItem");
    }

    @Override
    public int getTargetState() {
        return ON_DESTROY;
    }
    .....
}
```

来到ActivityThread

```java
public final class ActivityThread extends ClientTransactionHandler {
    
    @Override
    public void handleDestroyActivity(IBinder token, boolean finishing, int configChanges,
            boolean getNonConfigInstance, String reason) {
        ActivityClientRecord r = performDestroyActivity(token, finishing,
                configChanges, getNonConfigInstance, reason);
        if (r != null) {
             // 移除 DecorView
             cleanUpPendingRemoveWindows(r, finishing);
             .....
             // 将DecorView设置为null
             r.activity.mDecor = null;
            ....
        }
        if (finishing) {
            try {
                // 通知ATMS 当前activity已经销毁
                ActivityTaskManager.getService().activityDestroyed(token);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
        }
    }
    // 活动销毁调用的核心实现。
    ActivityClientRecord performDestroyActivity(IBinder token, boolean finishing,
            int configChanges, boolean getNonConfigInstance, String reason) {
        ActivityClientRecord r = mActivities.get(token);
          try {
                r.activity.mCalled = false;
                // 直接调用activity的performDestroy方法
                mInstrumentation.callActivityOnDestroy(r.activity);
            } catch (SuperNotCalledException e) {
                throw e;
          }
        // 将activity对应的token从mActivities中移除
        synchronized (mResourcesManager) {
            mActivities.remove(token);
        }
        .....
    }
}
```

最后来到了activity里面

```java
public class Activity extends ContextThemeWrapper{
    final void performDestroy() {
        mDestroyed = true;
        mWindow.destroy();
        // 将周期分发给fragment
        mFragments.dispatchDestroy();
        // 调用onDestory周期
        onDestroy();
        mFragments.doLoaderDestroy();
    }    
}
```

至此，activity完整的生命周期就分析完了。

## 8、完整时序图

献上呕心沥血画出来的activity生命周期时序图！

<img src="/img/blog_activity_lifecycle/lifecycler_uml.png" width="100%" height="60%">

## 后记

一顿分析下来，其实核心还是ActivityThread这个类。ATMS通过ClientTransactionItem对象，实现部分的周期调用（OnResume，onPause，onDestory），其他的周期要么通过TransactionExecutorHelper补充调用（比如onCreate，onStart、onStop），要么就是在调用最终周期的时候补充调用（比如onPostCreate，onPostResume）。

ATMS对于周期的调用，就发了三个请求事务item：LaunchActivityItem、PauseActivityItem、DestroyActivityItem。为啥？binder通信虽然很快，但是总会有消耗的。那为啥不去掉PauseActivityItem？这个就涉及到两个activity之间启动的生命周期顺序了。

——Weiwq  于 2021.06 广州









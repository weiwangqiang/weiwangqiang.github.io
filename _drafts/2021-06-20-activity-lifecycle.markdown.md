layout:     post
title:      "framework之Activity 生命周期回调（基于Android11源码）"
subtitle:   " \"activity的生命周期，是如此朴实无华\""
date:       2021-06-20 15:08:00
author:     "Weiwq"
header-img: "img/background/post-bg-nextgen-web-pwa.jpg"
catalog:  true
tags:

    - Android

---

> 上一篇讲了activity的创建过程，那这一篇就讲讲activity的生命周期

## 引言

上一篇讲了activity的创建过程（没看过的小伙伴移步<font color="red"> [点我前往](https://weiwangqiang.github.io/2021/06/08/start-activity-flow/)</font>），那这一篇就讲讲activity的生命周期

在高版本上，activity的周期都是以事务的方式调用，activityThread里面H类的`EXECUTE_TRANSACTION` 消息正是接收、处理事务的入口，实际最终由TransactionExecutor 处理该事务。

```java
public final class ActivityThread extends ClientTransactionHandler {
    private final TransactionExecutor mTransactionExecutor = new TransactionExecutor(this);
    class H extends Handler {      
             public void handleMessage(Message msg) {
                  case EXECUTE_TRANSACTION:
                        final ClientTransaction transaction = (ClientTransaction) msg.obj;
                        mTransactionExecutor.execute(transaction);
                        if (isSystem()) {
                            // Client transactions inside system process are recycled on the client side
                            // instead of ClientLifecycleManager to avoid being cleared before this
                            // message is handled.
                            transaction.recycle();
                        }
                   .....
             }
        }
}

```

我们来看看整个activity启动过程会走哪些周期，请记住下面的周期和顺序，接下来就会讲讲里面的调用逻辑了

```java
06-20 13:58:04.476 17073 17073 D CycleActivity: onCreate:
06-20 13:58:04.481 17073 17073 D CycleActivity: onStart:
06-20 13:58:04.483 17073 17073 D CycleActivity: onPostCreate:
06-20 13:58:04.483 17073 17073 D CycleActivity: onResume:
06-20 13:58:04.494 17073 17073 D CycleActivity: onPostResume:
06-20 13:58:08.690 17073 17073 D CycleActivity: onPause:
06-20 13:58:10.207 17073 17073 D CycleActivity: onStop:
06-20 13:58:10.210 17073 17073 D CycleActivity: onDestroy:

```

## 1、onCreate

[framework之Activity启动流程](https://weiwangqiang.github.io/2021/06/08/start-activity-flow/)，里面已经很详细描述了onCreate的调用流程，对应的流程如下

<img src="/img/blog_start_acivity_flow/activityThread.png" width="100%" height="40%">

简单来讲，通过LaunchActivityItem调用了ActivityThread的handleLaunchActivity方法，ActivityThread在创建activity实例后，会设置config、window、resource、theme相关资源，在调用activity的attach方法后，会接着调用onCreate方法，完成onCreate周期的调用。

## 2、onStart

onCreate之后，onStart又是如何调用的呢？这里我们回顾一下onCreate的调用：TransactionExecutor#execute

```java
// 以正确顺序管理事务执行的类。
public class TransactionExecutor {
   ....
    // 处理服务端传递过来的事务
    public void execute(ClientTransaction transaction) {
        ....
        // 该方法中通过遍历 transaction#callbacks 获取到LaunchActivityItem，然后调用onCreate方法
        executeCallbacks(transaction);
        // 这个？
        executeLifecycleState(transaction);
    }
}
```

不知读者发现没有，executeLifecycleState这个方法是在onCreate周期调用完后，才会接着调用的？

很好，续杯！接着看

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

看看cycleToPath方法，这里在拿到lifeCycle后就交给了performLifecycleSequence

```java
// 循环到最终请求状态之前的状态。
private void cycleToPath(ActivityClientRecord r, int finish, boolean excludeLastState,
            ClientTransaction transaction) {
        final int start = r.getLifecycleState();
        final IntArray path = mHelper.getLifecyclePath(start, finish, excludeLastState);
        performLifecycleSequence(r, path, transaction);
}

```

我们来到了performLifecycleSequence 方法，问题来了，

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
                case ON_START:
                    mTransactionHandler.handleStartActivity(r.token, mPendingActions);
                    break;
                case ON_RESUME:
                    mTransactionHandler.handleResumeActivity(r.token, false /* finalStateRequest */,
                            r.isForward, "LIFECYCLER_RESUME_ACTIVITY");
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
}
```

通过debug发现，path的size是1，mValues的第一个值是2，即state为2，显然走到ON_START case，调用mTransactionHandler#handleStartActivity方法。

<img src="img/blog_activity_lifecycle/1.png" width="100%" height="40%">

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

<img src="img/blog_activity_lifecycle/2.png" width="100%" height="40%">

lifecycleItem是ResumeActivityItem的实例。

```java
public class ResumeActivityItem extends ActivityLifecycleItem {

    @Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityResume");
        // 调用了ActivityThread的handleResumeActivity
        client.handleResumeActivity(token, true /* finalStateRequest */, mIsForward,
                "RESUME_ACTIVITY");
        Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
    }
    
    @Override
    public void postExecute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        try {
            // 通知ATM 当前完成onResume
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






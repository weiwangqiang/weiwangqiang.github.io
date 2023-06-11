---
layout:     post
title:      "Service启动流程之startService"
subtitle:   " \" 带你看Android 13下startService的实现 \""
date:       2023-06-04 13:30:00
author:     "Weiwq"
header-img: "img/background/home-bg-o.jpg"
catalog:  true
top: false
tags:
    - Android

---

# 前言

启动service有两种方式：startService和bindService。 这一篇先讲startService，读者如果只想看流程图，可以直接跳到总结。

# 1. ContextImpl

> 代码路径：frameworks\base\core\java\android\app\ContextImpl.java

## 1.1 startService

我们在调用startService的时候，实际是调用到ContextImpl的startService

```java
@Override
public ComponentName startService(In tent service) {
    warnIfCallingFromSystemProcess();
    return startServiceCommon(service, false, mUser);
}
```

## 1.2 startServiceCommon

startServiceCommon通过ActivityManager获取AMS的接口IActivityManager，将启动service请求发送给AMS

```java
private ComponentName startServiceCommon(Intent service, boolean requireForeground,
                                         UserHandle user) {
    try {
        // 调用AMS的IActivityManager接口启动service
        ComponentName cn = ActivityManager.getService().startService(
                mMainThread.getApplicationThread(), service,
                service.resolveTypeIfNeeded(getContentResolver()), requireForeground,
                getOpPackageName(), getAttributionTag(), user.getIdentifier());
        if (cn != null) {
            if (cn.getPackageName().equals("!")) {
                throw new SecurityException("Not allowed to start service " + service
                        + " without permission " + cn.getClassName());
            } else if (cn.getPackageName().equals("!!")) {
                throw new SecurityException("Unable to start service " + service
                        + ": " + cn.getClassName());
            } else if (cn.getPackageName().equals("?")) {
                throw ServiceStartNotAllowedException.newInstance(requireForeground,
                        "Not allowed to start service " + service + ": " + cn.getClassName());
            }
        }
        return cn;
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```

# 2. ActivityManagerService

> 代码路径：frameworks\base\services\core\java\com\android\server\am\ActivityManagerService.java

## 2.1 startService

AMS 的startService 实现如下

```java
final ActiveServices mServices;
@Override
public ComponentName startService(IApplicationThread caller, Intent service ...) throws TransactionTooLargeException {
    synchronized(this) {
        // 通过Binder获取调用方的进程id
        final int callingPid = Binder.getCallingPid();
        // 通过Binder获取调用方的uid
        final int callingUid = Binder.getCallingUid();
        final long origId = Binder.clearCallingIdentity();
        ComponentName res;
        try {
            res = mServices.startServiceLocked(caller, service,
                    resolvedType, callingPid, callingUid,
                    requireForeground, callingPackage, callingFeatureId, userId);
        } finally {
            Binder.restoreCallingIdentity(origId);
        }
        return res;
    }
}
```

# 3. ActiveServices

> 代码路径：frameworks\base\services\core\java\com\android\server\am\ActiveServices.java

## 3.1 startServiceLocked

```java
ComponentName startServiceLocked(IApplicationThread caller, Intent service ...) throws TransactionTooLargeException {
    return startServiceLocked(caller, service ....);
}
```

## 3.2 startServiceLocked

```java
ComponentName startServiceLocked(IApplicationThread caller, Intent service ...) throws TransactionTooLargeException {
    ....
    // 如果客户端尝试启动/连接的是别名，那么我们需要返回
    // 客户端的别名组件名称，而不是“目标”组件名称，
    final ComponentName realResult = startServiceInnerLocked(r, service, callingUid, callingPid, fgRequired, callerFg, allowBackgroundActivityStarts, backgroundActivityStartsToken);
    if (res.aliasComponent != null
            && !realResult.getPackageName().startsWith("!")
            && !realResult.getPackageName().startsWith("?")) {
        return res.aliasComponent;
    } else {
        return realResult;
    }
}
```

## 3.3 startServiceInnerLocked

内部有两个startServiceInnerLocked 的重载，这里只分析最后一个：

```java
ComponentName startServiceInnerLocked(ServiceMap smap, Intent service, ServiceRecord r ...) throws TransactionTooLargeException {
    final int uid = r.appInfo.uid;
    final String packageName = r.name.getPackageName();
    final String serviceName = r.name.getClassName();
    mAm.mBatteryStatsService.noteServiceStartRunning(uid, packageName, serviceName);
    // 准备启动service
    String error = bringUpServiceLocked(r, service.getFlags(), callerFg,
            false /* whileRestarting */,
            false /* permissionsReviewRequired */,
            false /* packageFrozen */,
            true /* enqueueOomAdj */);
    mAm.updateOomAdjPendingTargetsLocked(OomAdjuster.OOM_ADJ_REASON_START_SERVICE);
    if (error != null) {
        return new ComponentName("!!", error);
    }
    return r.name;
}

```

参数中的ServiceRecord 是用于描述Servic，结构如下

> 代码路径：frameworks\base\services\core\java\com\android\server\am\ServiceRecord.java

```java
final class ServiceRecord extends Binder implements ComponentName.WithComponentName {
    final ComponentName name; // service 组件.
    private final ArrayMap<IBinder, ArrayList<ConnectionRecord>> connections
            = new ArrayMap<IBinder, ArrayList<ConnectionRecord>>();
                            // 所有的绑定Service的客户端
}
```

## 3.4 bringUpServiceLocked

```java
private String bringUpServiceLocked(ServiceRecord r, int intentFlags ...) throws TransactionTooLargeException {
    if (r.app != null && r.app.getThread() != null) {
         // 调用service#onSatrtCommand周期
         sendServiceArgsLocked(r, execInFg, false);
         return null;
	}
    if (!whileRestarting && mRestartingServices.contains(r)) {
        // 等待重启，不需要做任何事情
        return null;
    }
    ....
    // 描述运行的应用程序进程的信息
    ProcessRecord app;
    if (!isolated) {
        // 获取进程信息
        app = mAm.getProcessRecordLocked(procName, r.appInfo.uid);
        if (app != null) {
            final IApplicationThread thread = app.getThread();
            final int pid = app.getPid();
            final UidRecord uidRecord = app.getUidRecord();
            // 如果运行Service的应用进程存在，就启动Service
            if (thread != null) {
                try {
                    app.addPackage(r.appInfo.packageName, r.appInfo.longVersionCode,
                            mAm.mProcessStats);
                    // 真正启动service的地方
                    realStartServiceLocked(r, app, thread, pid, uidRecord, execInFg,
                            enqueueOomAdj);
                    return null;
                } catch (TransactionTooLargeException e) {
                    ....
                }
            }
        }
    }
    ...
    // App未运行 -- 启动并排队此服务记录，在应用程序启动时执行。
    if (app == null && !permissionsReviewRequired && !packageFrozen) {
        ....
        // 创建进程，其中procName用来描述Service想要在哪个进程上运行，默认是当前进程
        app = mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                    hostingRecord, ZYGOTE_POLICY_FLAG_EMPTY, false, isolated);
        if (app == null) {
            // 启动进程失败
            String msg = "Unable to launch app "
                    + r.appInfo.packageName + "/"
                    + r.appInfo.uid + " for service "
                    + r.intent.getIntent() + ": process is bad";
            Slog.w(TAG, msg);
            bringDownServiceLocked(r, enqueueOomAdj);
            return msg;
        }
    }
    if (r.delayedStop) {
        // 嘿，我们已经被要求停止了！
        r.delayedStop = false;
        if (r.startRequested) {
            // 将服务停止
            stopServiceLocked(r, enqueueOomAdj);
        }
    }
    return null;
}
```

## 3.5 realStartServiceLocked

该方法会通过IApplicationThread 调用到客户端进程。

```java
/**
 * 请注意，此方法的名称不应与启动的服务概念混淆。
 * 这里的“start”表示在客户端中启动实例，此方法也会被 bindService（）调用
 */
private void realStartServiceLocked(ServiceRecord r, ProcessRecord app,
        IApplicationThread thread, int pid, UidRecord uidRecord, boolean execInFg,
        boolean enqueueOomAdj) throws RemoteException {
    ....
    bumpServiceExecutingLocked(r, execInFg, "create", null /* oomAdjReason */);
    boolean created = false;
    try {
        // 通知客户端创建service
        thread.scheduleCreateService(r, r.serviceInfo,
                mAm.compatibilityInfoForPackage(r.serviceInfo.applicationInfo),
                app.mState.getReportedProcState());
        r.postNotification();
        created = true;
    } catch (DeadObjectException e) {
        // 应用进程已经死亡
        mAm.appDiedLocked(app, "Died when creating service");
        throw e;
    } finally {
        if (!created) {
            final boolean inDestroying = mDestroyingServices.contains(r);
            serviceDoneExecutingLocked(r, inDestroying, inDestroying, false);
            // 清理
            if (newService) {
                psr.stopService(r);
                r.setProcess(null, null, 0, null);
            }
            // 尝试重启服务
            if (!inDestroying) {
                scheduleServiceRestartLocked(r, false);
            }
        }
    }
    // 调用Service的onBind方法
    requestServiceBindingsLocked(r, execInFg);
    updateServiceClientActivitiesLocked(psr, null, true);
    if (newService && created) {
        psr.addBoundClientUidsOfNewService(r);
    }
    // 如果服务处于已启动状态，并且没有
    // pending参数，然后伪造一个，以便调用onStartCommand（）
    if (r.startRequested && r.callStart && r.pendingStarts.size() == 0) {
        r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                null, null, 0));
    }
    // 通知客户端调用onStartCommand 方法
    sendServiceArgsLocked(r, execInFg, true);
    if (r.delayedStop) {
        // 需要停止服务
        r.delayedStop = false;
        if (r.startRequested) {
            if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE,
                    "Applying delayed stop (from start): " + r);
            stopServiceLocked(r, enqueueOomAdj);
        }
    }
}
```

### 3.5.1 bumpServiceExecutingLocked

```java
// 将给定的服务记录提升到执行状态。
private boolean bumpServiceExecutingLocked(ServiceRecord r, boolean fg, String why, String oomAdjReason) {
    boolean timeoutNeeded = true;
    ProcessServiceRecord psr;
    if (r.executeNesting == 0) {
        if (r.app != null) {
            psr = r.app.mServices;
            if (timeoutNeeded && psr.numberOfExecutingServices() == 1) {
                // 启动服务超时任务,即开启ANR计时
                scheduleServiceTimeoutLocked(r.app);
            }
        }
    } else if (r.app != null && fg) {
        psr = r.app.mServices;
        if (!psr.shouldExecServicesFg()) {
            if (timeoutNeeded) {
                scheduleServiceTimeoutLocked(r.app);
            }
        }
    }
    ....
    return oomAdjusted;
}
```

### 3.5.2 scheduleServiceTimeoutLocked

这里通过handler延时检查service是否发生ANR，其中

- SERVICE_TIMEOUT = 20s。
- SERVICE_BACKGROUND_TIMEOUT = SERVICE_TIMEOUT * 10，即200s。

```java
void scheduleServiceTimeoutLocked(ProcessRecord proc) {
    if (proc.mServices.numberOfExecutingServices() == 0 || proc.getThread() == null) {
        return;
    }
    Message msg = mAm.mHandler.obtainMessage(
            ActivityManagerService.SERVICE_TIMEOUT_MSG);
    msg.obj = proc;
    mAm.mHandler.sendMessageDelayed(msg, proc.mServices.shouldExecServicesFg()
            ? SERVICE_TIMEOUT : SERVICE_BACKGROUND_TIMEOUT);
}
```

### 3.5.3 sendServiceArgsLocked

该方法主要设置参数，并传给客户端

```java
private final void sendServiceArgsLocked(ServiceRecord r, boolean execInFg,
        boolean oomAdjusted) throws TransactionTooLargeException {
     ParceledListSlice<ServiceStartArgs> slice = new ParceledListSlice<>(args);
     slice.setInlineCountLimit(4);
     Exception caughtException = null;
     try {
         // 调用客户客户端的scheduleServiceArgs，触发onStartCommand周期
         r.app.getThread().scheduleServiceArgs(r, slice);
     } catch (TransactionTooLargeException e) {
         caughtException = e;
     }
}
```

# 4. ApplicationThread

> 代码路径：frameworks\base\core\java\android\app\ActivityThread.java

## 4.1 scheduleCreateService 

在realStartServiceLocked 方法中会调用到IApplicationThread的scheduleCreateService 接口。对应的实现类是在ActivityThread中，这个时候，调用流程就来到了应用端。

```java
private class ApplicationThread extends IApplicationThread.Stub {
   public final void scheduleCreateService(IBinder token,
           ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
       updateProcessState(processState, false);
       CreateServiceData s = new CreateServiceData();
       s.token = token;
       s.info = info;
       s.compatInfo = compatInfo;
       sendMessage(H.CREATE_SERVICE, s);
   }
}
```

scheduleCreateService将参数封装在CreateServiceData 对象中，并且将消息发送给H类，实现binder线程切换到主线程

```java
private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
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

## 4.2 scheduleServiceArgs

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

# 5. ActivityThread

> 代码路径：frameworks\base\core\java\android\app\ActivityThread.java

## 5.1 H类

在ActivityThread中有一个名为H的Handler类，对应的handleMessage实现如下

```java
class H extends Handler {
    public void handleMessage(Message msg) {
       case CREATE_SERVICE:
          handleCreateService((CreateServiceData)msg.obj);
          Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
          break;
       case SERVICE_ARGS:
          handleServiceArgs((ServiceArgsData)msg.obj);
          break;
    }
}
```

## 5.2 handleCreateService

```java
  private void handleCreateService(CreateServiceData data) {
        LoadedApk packageInfo = getPackageInfoNoCheck(
                data.info.applicationInfo, data.compatInfo);
        Service service = null;
        try {
            // 获取application对象
            Application app = packageInfo.makeApplicationInner(false, mInstrumentation);
            // 获取classLoader
            final java.lang.ClassLoader cl;
            if (data.info.splitName != null) {
                cl = packageInfo.getSplitClassLoader(data.info.splitName);
            } else {
                cl = packageInfo.getClassLoader();
            }
            // 通过工厂模式创建Service对象
            service = packageInfo.getAppFactory()
                    .instantiateService(cl, data.info.name, data.intent);
            // 获取context对象
            ContextImpl context = ContextImpl.getImpl(service
                    .createServiceBaseContext(this, packageInfo));
            // service 的resource必须使用app的loader
            context.getResources().addLoaders(
                    app.getResources().getLoaders().toArray(new ResourcesLoader[0]));
            // 将Service赋值给context的outerContext字段
            context.setOuterContext(service);
            // attach方法内部与创建的context，application，activityThread进行关联
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManager.getService());
            // 调用Service的onCreate方法
            service.onCreate();
            mServicesData.put(data.token, data);
            mServices.put(data.token, service);
            try {
                // 通知AMS，Service成功启动
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

## 5.3 handleServiceArgs

```java
 private void handleServiceArgs(ServiceArgsData data) {
        CreateServiceData createData = mServicesData.get(data.token);
        Service s = mServices.get(data.token);
        if (s != null) {
            try {
                int res;
                if (!data.taskRemoved) {
                    // 调用onStartCommand周期
                    res = s.onStartCommand(data.args, data.flags, data.startId);
                } else {
                    s.onTaskRemoved(data.args);
                    res = Service.START_TASK_REMOVED_COMPLETE;
                }
                try {
                    // 将res（即启动状态）返回给AMS，通知Service已经启动
                    ActivityManager.getService().serviceDoneExecuting(
                            data.token, SERVICE_DONE_EXECUTING_START, data.startId, res);
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
            }
        }
    }
```

# 6. ActivityManagerService

> 代码路径：frameworks\base\services\core\java\com\android\server\am\ActivityManagerService.java

##  6.1 serviceDoneExecuting

该方法只调用了ActiveServices的serviceDoneExecuting

```java
final ActiveServices mServices;

public void serviceDoneExecuting(IBinder token, int type, int startId, int res) {
    synchronized(this) {
        mServices.serviceDoneExecutingLocked((ServiceRecord) token, type, startId, res, false);
    }
}
```

# 7. ActiveServices

> 代码路径：frameworks\base\services\core\java\com\android\server\am\ActiveServices.java

## 7.1 serviceDoneExecutingLocked

这里的type是SERVICE_DONE_EXECUTING_ANON

```java
    void serviceDoneExecutingLocked(ServiceRecord r, int type, int startId, int res,
            boolean enqueueOomAdj) {
        boolean inDestroying = mDestroyingServices.contains(r);
        if (r != null) {
            // service 已启动，即已调用onStartCommand周期
            if (type == ActivityThread.SERVICE_DONE_EXECUTING_START) {
                // 判断onStartCommand的返回值
                r.callStart = true;
                switch (res) {
                    case Service.START_STICKY_COMPATIBILITY:
                    case Service.START_STICKY: { 
                        // 服务如果被杀，会被系统重新启动
                        r.findDeliveredStart(startId, false, true);
                        // 如果被杀，不要停止服务
                        r.stopIfKilled = false;
                        break;
                    }
                    case Service.START_NOT_STICKY: {
                        // 服务被杀，不会被系统启动
                        r.findDeliveredStart(startId, false, true);
                        if (r.getLastStartId() == startId) {
                            r.stopIfKilled = true;
                        }
                        break;
                    }
                    case Service.START_REDELIVER_INTENT: {
                        // 将保留此项目，但是不重新启动Service，直到调用stop方法。
                        ServiceRecord.StartItem si = r.findDeliveredStart(startId, false, false);
                        if (si != null) {
                            si.deliveryCount = 0;
                            si.doneExecutingCount++;
                            r.stopIfKilled = true;
                        }
                        break;
                    }
                    case Service.START_TASK_REMOVED_COMPLETE: {
                        r.findDeliveredStart(startId, true, true);
                        break;
                    }
                }
            }
            .... 
            final long origId = Binder.clearCallingIdentity();
            // 调用重载方法
            serviceDoneExecutingLocked(r, inDestroying, inDestroying, enqueueOomAdj);
            Binder.restoreCallingIdentity(origId);
        }
    }
```

## 7.2 serviceDoneExecutingLocked

```java
private void serviceDoneExecutingLocked(ServiceRecord r, boolean inDestroying,
            boolean finishing, boolean enqueueOomAdj) {
      ....
      if (psr.numberOfExecutingServices() == 0) {
	    // 移除启动超时的任务
	    mAm.mHandler.removeMessages(ActivityManagerService.SERVICE_TIMEOUT_MSG, r.app);
      } else if (r.executeFg) {
          // 需要重新评估应用是否仍需要位于前台。
         for (int i = psr.numberOfExecutingServices() - 1; i >= 0; i--) {
             if (psr.getExecutingServiceAt(i).executeFg) {
                 psr.setExecServicesFg(true);
                 break;
             }
        }
    }
    // 如果service处于回收状态，就移除记录，并清空binding的列表
    if (inDestroying) {
        mDestroyingServices.remove(r);
        r.bindings.clear();
    }
    if (finishing) {
        // 应用没有声明persistent属性
        if (r.app != null && !r.app.isPersistent()) {
            stopServiceAndUpdateAllowlistManagerLocked(r);
        }
        r.setProcess(null, null, 0, null);
    }
}
```

## 7.3 stopServiceAndUpdateAllowlistManagerLocked

```java
private void stopServiceAndUpdateAllowlistManagerLocked(ServiceRecord service) {
    final ProcessServiceRecord psr = service.app.mServices;
    // 将记录设置为服务停止，注意该方法不会停止服务，只做记录
    psr.stopService(service);
    psr.updateBoundClientUids();
    if (service.allowlistManager) {
        updateAllowlistManagerLocked(psr);
    }
}
```

# 总结

对应的uml图如下



![](\img\blog_start_service_flow\1.png)


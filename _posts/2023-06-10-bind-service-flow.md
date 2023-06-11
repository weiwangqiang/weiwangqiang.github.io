---
layout:     post
title:      "Service启动流程之bindService"
subtitle:   " \" 带你看Android 13下bindService的实现 \""
date:       2023-06-10 13:30:00
author:     "Weiwq"
header-img: "img/background/home-bg-o.jpg"
catalog:  true
top: false
tags:
    - Android

---



# 前言

在[【Service启动流程之startService】](https://weiwangqiang.github.io/2023/06/04/start-service-flow/) 中，我们已经分析了startService的流程，这篇就继续讲bindService的流程，他们两有很多相似之处。同样，流程图在总结处。

我们在调用bindService方法时候，实际调用的是ContextImpl的实现。

# 1. ContextImpl

> 代码路径：frameworks\base\core\java\android\app\ContextImpl.java

## 1.1 bindService

```java
@Override
public boolean bindService(Intent service, ServiceConnection conn, int flags) {
    warnIfCallingFromSystemProcess();
    return bindServiceCommon(service, conn, flags, null, mMainThread.getHandler(), null,
            getUser());
}
```

## 1.2 bindServiceCommon

```java
private boolean bindServiceCommon(Intent service, ServiceConnection conn, 
                                  int flags, String instanceName, Handler handler, 
                                  Executor executor, UserHandle user) {
    IServiceConnection sd;
    if (mPackageInfo != null) {
        // 将ServiceConnection 通过LoadedApk包装到 ServiceDispatcher 中
        if (executor != null) {
            sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), executor, flags);
        } else {
            sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), handler, flags);
        }
    }
    try {
        IBinder token = getActivityToken();
        if (token == null && (flags&BIND_AUTO_CREATE) == 0 && mPackageInfo != null
                && mPackageInfo.getApplicationInfo().targetSdkVersion
                < android.os.Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
            flags |= BIND_WAIVE_PRIORITY;
        }
        service.prepareToLeaveProcess(this);
        // 调用 AMS 接口
        int res = ActivityManager.getService().bindServiceInstance(
                mMainThread.getApplicationThread(), getActivityToken(), service,
                service.resolveTypeIfNeeded(getContentResolver()),
                sd, flags, instanceName, getOpPackageName(), user.getIdentifier());
        return res != 0;
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```

# 2. LoaderApk

> 代码路径：frameworks\base\core\java\android\app\LoadedApk.java

getServiceDispatcher 直接调用getServiceDispatcherCommon 方法

## 2.1 getServiceDispatcherCommon

```java
private IServiceConnection getServiceDispatcherCommon(ServiceConnection c,
            Context context, Handler handler, Executor executor, int flags) {
        synchronized (mServices) {
            LoadedApk.ServiceDispatcher sd = null;
            ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher> map = mServices.get(context);
            if (map != null) {
                sd = map.get(c);
            }
            if (sd == null) {
                if (executor != null) {
                    // 将ServiceConnection 保存到ServiceDispatcher中
                    sd = new ServiceDispatcher(c, context, executor, flags);
                } else {
                    sd = new ServiceDispatcher(c, context, handler, flags);
                }
                if (map == null) {
                    map = new ArrayMap<>();
                    mServices.put(context, map);
                }
                map.put(c, sd);
            } else {
                sd.validate(context, handler, executor);
            }
            // 最后返回的是一个bind对象，即IServiceConnection
            return sd.getIServiceConnection();
        }
    }
```

# 3. ActivityManagerService

> 代码路径：frameworks\base\services\core\java\com\android\server\am\ActivityManagerService.java

## 3.1 bindServiceInstance

```java
    private int bindServiceInstance(IApplicationThread caller, IBinder token, 
                                    Intent service, String resolvedType,
                                    ServiceConnection connection ...) {
		....
        try {
            synchronized (this) {
                // 调用ActiveServices的bindServiceLocked
                return mServices.bindServiceLocked(caller, token, service, resolvedType, connection,flags, instanceName, isSdkSandboxService, sdkSandboxClientAppUid,        sdkSandboxClientAppPackage, callingPackage, userId);
            }
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        }
    }
```

# 4.  ActiveService

> 代码路径：frameworks\base\services\core\java\com\android\server\am\ActiveServices.java

## 4.1 bindServiceLocked

```java
int bindServiceLocked(IApplicationThread caller, IBinder token, Intent service,
        String resolvedType, final IServiceConnection connection ...)
        throws TransactionTooLargeException {
    final ProcessRecord callerApp = mAm.getRecordForAppLOSP(caller);
    ActivityServiceConnectionsHolder<ConnectionRecord> activity = null;
    int clientLabel = 0;
    PendingIntent clientIntent = null;
    // 获取 ServiceRecord
    ServiceLookupResult res = retrieveServiceLocked(service, instanceName,
            isSdkSandboxService, sdkSandboxClientAppUid, sdkSandboxClientAppPackage,
            resolvedType, callingPackage, callingPid, callingUid, userId, true, callerFg,
            isBindExternal, allowInstant);
    ServiceRecord s = res.record;
    try {
        AppBindRecord b = s.retrieveAppBindingLocked(service, callerApp);
        // 用于保存 serviceConnect 信息
        ConnectionRecord c = new ConnectionRecord(b, activity,
                connection, flags, clientLabel, clientIntent,
                callerApp.uid, callerApp.processName, callingPackage, res.aliasComponent);
        IBinder binder = connection.asBinder();
        // 将connection binder对象添加到map中
        s.addConnection(binder, c);
        b.connections.add(c);
        boolean needOomAdj = false;
        // 传入flag是BIND_AUTO_CREATE
        if ((flags&Context.BIND_AUTO_CREATE) != 0) {
            // Service还没启动，就走启动流程
            // bringUpServiceLocked 方法返回null意味着成功启动Service
            if (bringUpServiceLocked(s, service.getFlags(), callerFg, false,
                    permissionsReviewRequired, packageFrozen, true) != null) {
                // 启动失败，添加排队
                mAm.updateOomAdjPendingTargetsLocked(OomAdjuster.OOM_ADJ_REASON_BIND_SERVICE);
                return 0;
            }
        }
        if (s.app != null && b.intent.received) {
            // 服务已经在运行，可以直接调用connected
            //如果客户端尝试启动/连接的是别名，那么需要将别名组件名称传递给客户端。
            final ComponentName clientSideComponentName =
                    res.aliasComponent != null ? res.aliasComponent : s.name;
            try {
                // 这里的c.conn 即 IServiceConnection对象
                c.conn.connected(clientSideComponentName, b.intent.binder, false);
            } catch (Exception e) {
            }

            // 如果这是连接回此绑定的第一个应用， 该服务此前曾要求被告知何时rebind，则需要调用requestServiceBindingLocked。
            if (b.intent.apps.size() == 1 && b.intent.doRebind) {
                requestServiceBindingLocked(s, b.intent, callerFg, true);
            }
        } else if (!b.intent.requested) {
            requestServiceBindingLocked(s, b.intent, callerFg, false);
        }

    } finally {
        Binder.restoreCallingIdentity(origId);
    }
    return 1;
}
```

## 4.2 realStartServiceLocked

如startService流程中的分析，bringUpServiceLocked 调用 realStartServiceLocked来启动Service

```java
private void realStartServiceLocked(ServiceRecord r, ProcessRecord app,
        IApplicationThread thread ...) throws RemoteException {
    // 通知客户端创建service
    thread.scheduleCreateService(r, r.serviceInfo,
            mAm.compatibilityInfoForPackage(r.serviceInfo.applicationInfo),
            app.mState.getReportedProcState());
    // 调用Service的onBind方法
    requestServiceBindingsLocked(r, execInFg);
    .....
}
```

## 4.3 requestServiceBindingsLocked

```java
    private final void requestServiceBindingsLocked(ServiceRecord r, boolean execInFg)
            throws TransactionTooLargeException {
        for (int i=r.bindings.size()-1; i>=0; i--) {
            IntentBindRecord ibr = r.bindings.valueAt(i);
            if (!requestServiceBindingLocked(r, ibr, execInFg, false)) {
                break;
            }
        }
    }
```

## 4.4 requestServiceBindingLocked

```java
private final boolean requestServiceBindingLocked(ServiceRecord r, IntentBindRecord i,
            boolean execInFg, boolean rebind) throws TransactionTooLargeException {
      try {
          // 启动计时
          bumpServiceExecutingLocked(r, execInFg, "bind",
                  OomAdjuster.OOM_ADJ_REASON_BIND_SERVICE);
          // 通知客户端调用bind
          r.app.getThread().scheduleBindService(r, i.intent.getIntent(), rebind,
                  r.app.mState.getReportedProcState());
          ....
      }  catch (RemoteException e) {
          .....
          return false;
      }
      return true;
}
```

# 5. ApplicationThread

> 代码路径：frameworks\base\core\java\android\app\ActivityThread.java

## 5.1 scheduleBindService

```java
public final void scheduleBindService(IBinder token, Intent intent,
        boolean rebind, int processState) {
    updateProcessState(processState, false);
    BindServiceData s = new BindServiceData();
    s.token = token;
    s.intent = intent;
    s.rebind = rebind;
    sendMessage(H.BIND_SERVICE, s);
}

```

# 6. ActivityThread

> 代码路径：frameworks\base\core\java\android\app\ActivityThread.java

在发送BIND_SERVICE 消息后，最后调用到ActivityThread#handleBindService 方法

## 6.1 handleBindService

```java
    private void handleBindService(BindServiceData data) {
        CreateServiceData createData = mServicesData.get(data.token);
        Service s = mServices.get(data.token);
        try {
           if (!data.rebind) {
               // 不是重新bind，调用onBind方法
               IBinder binder = s.onBind(data.intent);
               // 将binder对象传给AMS
               ActivityManager.getService().publishService(
                       data.token, data.intent, binder);
           } else {
               // 调用onRebind方法
               s.onRebind(data.intent);
               ActivityManager.getService().serviceDoneExecuting(
                       data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
           }
       } catch (RemoteException ex) {
           throw ex.rethrowFromSystemServer();
       }
    }
```

# 7. ActivityManagerService

>  代码路径：frameworks\base\services\core\java\com\android\server\am\ActivityManagerService.java

## 7.1 publishService

```java
public void publishService(IBinder token, Intent intent, IBinder service) {
    ....
    synchronized(this) {
        mServices.publishServiceLocked((ServiceRecord)token, intent, service);
    }
}
```

# 8. ActiveServices

> 代码路径：frameworks\base\services\core\java\com\android\server\am\ActiveServices.java

## 8.1 publishServiceLocked

```java
void publishServiceLocked(ServiceRecord r, Intent intent, IBinder service) {
        final long origId = Binder.clearCallingIdentity();
        try {
            if (r != null) {
                Intent.FilterComparison filter
                        = new Intent.FilterComparison(intent);
                IntentBindRecord b = r.bindings.get(filter);
                if (b != null && !b.received) {
                    b.binder = service; // 保存Service#onBind方法返回的binder对象
                    b.requested = true;
                    b.received = true;
                    ArrayMap<IBinder, ArrayList<ConnectionRecord>> connections = r.getConnections();
                    for (int conni = connections.size() - 1; conni >= 0; conni--) {
                        ArrayList<ConnectionRecord> clist = connections.valueAt(conni);
                        for (int i=0; i<clist.size(); i++) {
                            ConnectionRecord c = clist.get(i);
                            if (!filter.equals(c.binding.intent.intent)) {
                                continue;
                            }
                            try {
                                // 调用ServiceConnection的回调
                                c.conn.connected(clientSideComponentName, service, false);
                            } catch (Exception e) {
                            }
                        }
                    }
                }
                // 移除定时任务等
                serviceDoneExecutingLocked(r, mDestroyingServices.contains(r), false, false);
            }
        } finally {
            Binder.restoreCallingIdentity(origId);
        }
    }
```

## 8.2 ConnectionRecord

> 代码路径：frameworks\base\services\core\java\com\android\server\am\ConnectionRecord.java

该类定义如下

```java

final class ConnectionRecord {
    final IServiceConnection conn;  // The client connection.
    .....
}
```

这里的IServiceConnection 即我们在bindService的时候，生成的ServiceDispatcher中内部mIServiceConnection 变量



# 9. LoadedApk

> 代码路径：frameworks\base\core\java\android\app\LoadedApk.java

## 9.1 InnerConnection

```java
private static class InnerConnection extends IServiceConnection.Stub {
    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.R, trackingBug = 170729553)
    final WeakReference<LoadedApk.ServiceDispatcher> mDispatcher;

    InnerConnection(LoadedApk.ServiceDispatcher sd) {
        mDispatcher = new WeakReference<LoadedApk.ServiceDispatcher>(sd);
    }

    public void connected(ComponentName name, IBinder service, boolean dead)
            throws RemoteException {
        LoadedApk.ServiceDispatcher sd = mDispatcher.get();
        if (sd != null) {
            sd.connected(name, service, dead);
        }
    }
}
```

## 9.2 ServiceDispatcher

connect方法中会做切线程的动作，RunConnection 内部实际是调用到doConnected 方法。

```
static final class ServiceDispatcher {
  public void connected(ComponentName name, IBinder service, boolean dead) {
      if (mActivityExecutor != null) {
          mActivityExecutor.execute(new RunConnection(name, service, 0, dead));
      } else if (mActivityThread != null) {
          mActivityThread.post(new RunConnection(name, service, 0, dead));
      } else {
          doConnected(name, service, dead);
      }
  }
}
```

## 9.3 doConnected

```java
public void doConnected(ComponentName name, IBinder service, boolean dead) {
    ServiceDispatcher.ConnectionInfo old;
    ServiceDispatcher.ConnectionInfo info;
    synchronized (this) {
        old = mActiveConnections.get(name)
        if (service != null) {
            info = new ConnectionInfo();
            info.binder = service;
            info.deathMonitor = new DeathMonitor(name, service);
            try {
                // 监听bind的死亡回调
                service.linkToDeath(info.deathMonitor, 0);
                mActiveConnections.put(name, info);
            } catch (RemoteException e) {
                mActiveConnections.remove(name);
                return;
            }
        } else {
            // The named service is being disconnected... clean up.
            mActiveConnections.remove(name);
        }
        if (old != null) {
            old.binder.unlinkToDeath(old.deathMonitor, 0);
        }
    }
    // 如果有老的服务，那就通知断开
    if (old != null) {
        mConnection.onServiceDisconnected(name);
    }
    if (dead) {
        // 如果service端已经dead，就调用onBindingDied
        mConnection.onBindingDied(name);
    } else {
        // 通知service已经连接上，并且远程服务返回的bind对象不为null
        if (service != null) {
            mConnection.onServiceConnected(name, service);
        } else {
            // 远程服务返回null的bind对象，
            mConnection.onNullBinding(name);
        }
    }
}
```

# 总结

对应的uml图如下

![](\img\blog_bind_service_flow\1.png)

---
layout:     post
title:      "带你细看Android input系统中ANR的机制"
subtitle:   " \"从点击事件卡顿，到显示 Application not responding 对话框，你知道Android都做了哪些工作吗？\""
date:       2022-12-31 23:30:00
author:     "Weiwq"
header-img: "img/background/post-bg-nextgen-web-pwa.jpg"
catalog:  true
isTop:  true
tags:
   - Android

---

> “本文基于Android13源码，分析Input系统的Anr实现原理“


# 前言

在文章之前，先提几个问题：

- 如果在activity任意周期（onCreate,onResume等），同步执行耗时超过5s（ANR时间）的任务，期间不进行点击，那会触发ANR吗？
- 如果在button点击的时候，在onClick回调同步执行耗时超过5s的任务。点击一次会触发ANR吗？点击2次呢，3次呢？

# 1、anr 分类

首先看一下anr的分类：
- Input ANR：按键或触摸事件在5s内没有相应，主要在activity、fragment中。
- Service anr：前台service 响应时间是20s，后台service是200s。
- Broadcast anr：前台广播是10s，后台广播是60s。
- ContentProvider anr：publish执行未在10s内完成。
- startForgoundService：应用调用startForegroundService，然后5s内未调用startForeground出现ANR或者Crash

有些小伙伴可能好奇，为啥没有Activity ANR的分类？Activity ANR准确的来说是——Input系统检测，触发activity 的anr。所以本文将通过input系统来讲述Android是如何触发activity的anr。

# 2、InputDispatcher

在了解Input Anr 原理之前，我们简单了解一下InputDispatcher是如何分发按键事件的。

Inputdispatcher中，在线程里面调用到dispatchOnce方法，该方法中主要做：

- 通过dispatchOnceInnerLocked()，取出mInboundQueue 里面的 EventEntry事件
- 通过enqueueDispatchEntryLocked()，生成事件DispatchEntry并加入connection的`outbound`队列。
- 通过startDispatchCycleLocked()，从outboundQueue中取出事件DispatchEntry, 重新放入connection的`waitQueue`队列。同时通过inputPublisher.publishKeyEvent() 方法将按键事件分发给java层。
- 通过processAnrsLocked()，判断是否需要触发ANR。

按键事件存储在3个queue中：

1. InputDispatcher的mInboundQueue：存储的是从InputReader 送来的输入事件。
2. Connection的outboundQueue：该队列是存储即将要发送给应用的输入事件。
3. Connection的waitQueue：队列存储的是已经发给应用的事件，但是应用还未处理完成的。

![](\img/blog_activity_anr/3.png)

## 2.1 dispatchOnce

dispatchOnce() 中主要就是调用如下的两个方法：

- 分发事件：dispatchOnceInnerLocked() 
- 检查ANR：processAnrsLocked()

```java
> frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp

void InputDispatcher::dispatchOnce() {
    nsecs_t nextWakeupTime = LONG_LONG_MAX; 
    {
        ...
        // 如果没有挂起的命令，则运行调度循环。调度循环可能会将命令排入队列，以便稍后运行。
        if (!haveCommandsLocked()) {
            dispatchOnceInnerLocked(&nextWakeupTime);
        }
        // 运行所有挂起的命令（如果有）。如果运行了任何命令，则强制下一次轮询立即唤醒。
        if (runCommandsLockedInterruptable()) {
            nextWakeupTime = LONG_LONG_MIN;
        }
        ...
        // 我们可能必须早点醒来以检查应用程序是否正处于anr
        const nsecs_t nextAnrCheck = processAnrsLocked();
    } 
    // 等待回调、超时或唤醒。
    nsecs_t currentTime = now();
    int timeoutMillis = toMillisecondTimeoutDelay(currentTime, nextWakeupTime);
    mLooper->pollOnce(timeoutMillis);
}
```

我们先简单回顾下事件分发过程

# 3、事件分发

## 3.1 dispatchOnceInnerLocked

该方法主要是：

- 从mInboundQueue 中取出mPendingEvent

- 通过mPendingEvent的type决定事件类型和分发方式。比如当前是key类型。

主要代码如下：

```java
> frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp
  
void InputDispatcher::dispatchOnceInnerLocked(nsecs_t* nextWakeupTime) {
    nsecs_t currentTime = now();
    ...
    // 优化应用切换的延迟。本质上，当按下应用程序切换键（HOME）时，我们会开始一个短暂的超时。
    // 当它过期时，我们会抢占调度并删除所有其他挂起的事件。
    bool isAppSwitchDue = mAppSwitchDueTime <= currentTime;
    // 当前没有PendingEvent（即EventEntry），则取一个
    if (!mPendingEvent) {
         ...
        //  mInboundQueue不为空 ，就从队列前面取一个PendingEvent
            mPendingEvent = mInboundQueue.front();
            mInboundQueue.pop_front();
            traceInboundQueueLengthLocked();
    }
    ...
}
```

## 3.2 enqueueDispatchEntryLocked

enqueueDispatchEntryLocked() 会创建一个新的DispatchEntry，然后将DispatchEntry 加入到connection#outboundQueue 中

```java
> frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp
  
void InputDispatcher::enqueueDispatchEntryLocked(const sp<Connection>& connection,
                                                 std::shared_ptr<EventEntry> eventEntry,
                                                 const InputTarget& inputTarget,
                                                 int32_t dispatchMode) {
    // 这是一个新事件。将新的调度条目排队到此连接的出站队列中。
    std::unique_ptr<DispatchEntry> dispatchEntry =
            createDispatchEntry(inputTarget, eventEntry, inputTargetFlags);
    ...
    // 将生成的dispatchEntry 加入到 connection的outboundQueue 中
    connection->outboundQueue.push_back(dispatchEntry.release());
    traceOutboundQueueLength(*connection);
}
```

## 3.3 startDispatchCycleLocked

该方法主要通过connection 发布最终的事件，至此，InputDispatcher完成事件的发布，并且将发布的事件保存在connection的waitQueue中。

```java
> frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp
  
void InputDispatcher::startDispatchCycleLocked(nsecs_t currentTime,
                                               const sp<Connection>& connection) {
    while (connection->status == Connection::Status::NORMAL && !connection->outboundQueue.empty()) {
         // 从outboundQueue 队列中取出 DispatchEntry
        DispatchEntry* dispatchEntry = connection->outboundQueue.front();
        const std::chrono::nanoseconds timeout = getDispatchingTimeoutLocked(connection);
        // 设置超时时间
        dispatchEntry->timeoutTime = currentTime + timeout.count();
        // 发布事件
        status_t status;
        const EventEntry& eventEntry = *(dispatchEntry->eventEntry);
        ...
        // 将事件从outboundQueue中移除
        connection->outboundQueue.erase(std::remove(connection->outboundQueue.begin(),
                                                    connection->outboundQueue.end(),
                                                    dispatchEntry));
        // 在waitQueue 尾部重新插入
        connection->waitQueue.push_back(dispatchEntry);
        if (connection->responsive) {
            // 插入事件对应的anr检查时间
            mAnrTracker.insert(dispatchEntry->timeoutTime,
                               connection->inputChannel->getConnectionToken());
        }
    }
}
```

## 3.4 ANR超时时间

由 startDispatchCycleLocked() 方法，知道是通过getDispatchingTimeoutLocked() 获取到超时时间

```java
> frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp
  
//  如果没有用于确定适当调度超时的焦点应用程序或暂停窗口，则默认输入调度超时。
const std::chrono::duration DEFAULT_INPUT_DISPATCHING_TIMEOUT = std::chrono::milliseconds(
        android::os::IInputConstants::UNMULTIPLIED_DEFAULT_DISPATCHING_TIMEOUT_MILLIS *
        HwTimeoutMultiplier());

std::chrono::nanoseconds InputDispatcher::getDispatchingTimeoutLocked(
        const sp<Connection>& connection) {
    if (connection->monitor) {
         // 返回监控的超时时间
        return mMonitorDispatchingTimeout;
    }
    const sp<WindowInfoHandle> window =
            getWindowHandleLocked(connection->inputChannel->getConnectionToken());
    if (window != nullptr) {
        // 可以找到focused Window
        return window->getDispatchingTimeout(DEFAULT_INPUT_DISPATCHING_TIMEOUT);
    }
    // 获取默认的值
    return DEFAULT_INPUT_DISPATCHING_TIMEOUT;
}
```

WindowInfoHandle#getDispatchingTimeout 返回的值如下

```java
> libs/gui/include/gui/WindowInfo.h

class WindowInfoHandle : public RefBase {
  inline std::chrono::nanoseconds getDispatchingTimeout(
           std::chrono::nanoseconds defaultValue) const {
      return mInfo.token ? std::chrono::nanoseconds(mInfo.dispatchingTimeout) : defaultValue;
  }
}

struct WindowInfo : public Parcelable {
    std::chrono::nanoseconds dispatchingTimeout = std::chrono::seconds(5); // 5 秒
}
```

DEFAULT_INPUT_DISPATCHING_TIMEOUT 主要由UNMULTIPLIED_DEFAULT_DISPATCHING_TIMEOUT_MILLIS * HwTimeoutMultiplier() 计算得到

UNMULTIPLIED_DEFAULT_DISPATCHING_TIMEOUT_MILLIS的值如下

```java
> android/os/IInputConstants.h
  
class IInputConstants : public ::android::IInterface {
public:
  enum : int32_t { UNMULTIPLIED_DEFAULT_DISPATCHING_TIMEOUT_MILLIS = 5000 };
  ....
};  // class IInputConstants
}
```

HwTimeoutMultiplier() 方法定义如下，即读`ro.hw_timeout_multiplier` 属性值，默认是1。

```java
> system/libbase/include/android-base/properties.h
 
static inline int HwTimeoutMultiplier() {
  return android::base::GetIntProperty("ro.hw_timeout_multiplier", 1);
}
```

## 3.5 调用栈

native层的事件分发调用栈如下

```java
libs/input/InputTransport.cpp : InputPublisher::publishMotionEvent()
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::startDispatchCycleLocked()
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::enqueueDispatchEntriesLocked()
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::prepareDispatchCycleLocked()
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::dispatchKeyLocked()
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::dispatchOnceInnerLocked()
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::dispatchOnce()
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::start()
```

# 4、ANR触发

在dispatchOnce()，会调用processAnrsLocked 方法来决定是否需要触发anr

```java
> frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp
  
void InputDispatcher::dispatchOnce() {
    ...
    // 我们可能必须早点醒来以检查应用程序是否正处于anr
    const nsecs_t nextAnrCheck = processAnrsLocked();
    ....
}
```

## 4.1 processAnrsLocked

该方法是用于检查队列中是否有太旧的事件，如果存在就触发ANR

```java
> frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp
  
// 检查是否有任何连接的等待队列具有太旧的事件。如果我们等待事件被确认的时间超过窗口超时，
// 请引发 ANR。返回我们下次应该醒来的时间。
nsecs_t InputDispatcher::processAnrsLocked() {
    const nsecs_t currentTime = now();
    nsecs_t nextAnrCheck = LONG_LONG_MAX; // 下一次检查anr的时间
    // 检查我们是否正在等待一个聚焦窗口出现。如果等待时间过长就报 ANR
    if (mNoFocusedWindowTimeoutTime.has_value() && mAwaitedFocusedApplication != nullptr) {
        if (currentTime >= *mNoFocusedWindowTimeoutTime) {
            // 场景1: 当前时间 >= 等待focusedWindow的时间。触发noFocusedWindow的anr
            processNoFocusedWindowAnrLocked();
            mAwaitedFocusedApplication.reset();
            mNoFocusedWindowTimeoutTime = std::nullopt;
            return LONG_LONG_MIN;
        } else {
            // 请继续等待。我们将在mNoFocusedWindowTimeoutTime到来时放弃该事件。
            nextAnrCheck = *mNoFocusedWindowTimeoutTime;
        }
    }
    // 检查是否有任何连接 ANR 到期，mAnrTracker 中保存所有已分发事件（未被确认消费的事件）的超时时间
    nextAnrCheck = std::min(nextAnrCheck, mAnrTracker.firstTimeout());
    if (currentTime < nextAnrCheck) { // 最有可能的情况
        // 一切正常，在 nextAnrCheck 再检查一次
        return nextAnrCheck;
    }
    // 如果我们到达这里，则连接无响应。
    sp<Connection> connection = getConnectionLocked(mAnrTracker.firstToken());
    // 停止为此无响应的连接唤醒
    mAnrTracker.eraseToken(connection->inputChannel->getConnectionToken());
    // 场景2: 能找到window，并且事件已经超时，触发ANR
    onAnrLocked(connection);
    return LONG_LONG_MIN;
}
```

其中，mAnrTracker 存储已经成功分发给应用的事件。详情见startDispatchCycleLocked() 方法。

mNoFocusedWindowTimeoutTime 是在findFocusedWindowTargetsLocked() 方法中赋值的，在分发事件的时候会调用到findFocusedWindowTargetsLocked() :

```java
> frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp
  
InputEventInjectionResult InputDispatcher::findFocusedWindowTargetsLocked(
        nsecs_t currentTime, const EventEntry& entry, std::vector<InputTarget>& inputTargets,
        nsecs_t* nextWakeupTime) {
  ...
    // 兼容性行为：如果存在焦点应用程序但没有焦点窗口，则引发 ANR。只有当我们有重点事件要调度时，才开始计数。
    // 如果我们开始通过触摸（应用程序开关）与另一个应用程序交互，则 ANR 将被取消。
    // 如果将“无聚焦窗口 ANR”移动到策略中，则可以删除此代码。输入不知道应用是否应具有焦点窗口。
    if (focusedWindowHandle == nullptr && focusedApplicationHandle != nullptr) {
        if (!mNoFocusedWindowTimeoutTime.has_value()) {
            // 发现没有focusedWindow，就添加ANR定时器。
            std::chrono::nanoseconds timeout = focusedApplicationHandle->getDispatchingTimeout(
                    DEFAULT_INPUT_DISPATCHING_TIMEOUT);
            mNoFocusedWindowTimeoutTime = currentTime + timeout.count();
            ....
            return InputEventInjectionResult::PENDING;
        }
    }
  
    // 找到一个focusedwindow，就取消ANR定时器
    resetNoFocusedWindowTimeoutLocked();
  ...
}

void InputDispatcher::resetNoFocusedWindowTimeoutLocked() {
    // 取消ANR定时器
    mNoFocusedWindowTimeoutTime = std::nullopt;
    mAwaitedFocusedApplication.reset();
}
```

从上面的代码我们能小结出两个场景ANR的条件：

- 当前有等待获取焦点的应用，并且当前时间超过Timeout，调用processNoFocusedWindowAnrLocked() 进一步确认是否触发ANR。
- 当前时间超过事件响应的超时时间。调用onAnrLocked() 进一步确认是否触发ANR。

## 4.2 processNoFocusedWindowAnrLocked

该方法触发anr的条件是： 

1. 当前关注的应用程序必须与我们等待的应用程序相同。
2. 确保我们仍然没有聚焦窗口。

processNoFocusedWindowAnrLocked()  最后也是调用到onAnrLocked()。

```java
> frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp

//  如果没有聚焦窗口，请触发ANR。在触发 ANR 之前，请执行最终状态检查： 
void InputDispatcher::processNoFocusedWindowAnrLocked() {
    std::shared_ptr<InputApplicationHandle> focusedApplication =
            getValueByKey(mFocusedApplicationHandlesByDisplay, mAwaitedApplicationDisplayId);
    if (focusedApplication == nullptr ||
        focusedApplication->getApplicationToken() !=
                mAwaitedFocusedApplication->getApplicationToken()) {
        // 出乎意料，因为当前焦点应用程序已被更改，我们应该重置 ANR 计时器
        return;
    }
    const sp<WindowInfoHandle>& focusedWindowHandle =
            getFocusedWindowHandleLocked(mAwaitedApplicationDisplayId);
    if (focusedWindowHandle != nullptr) {
        //我们现在有一个焦点window，不需要再触发ANR
        return;
    }
    onAnrLocked(mAwaitedFocusedApplication);
}
```

onAnrLocked() 有两种实现：

- 能找到当前focus的window
- 找不到当前focus的window，但是可以找到当前fousedApplication。

我们先看情况1

## 4.3 onAnrLocked（connection）

```java
> frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp
  
//情况1: 能找到window的情况
void InputDispatcher::onAnrLocked(const sp<Connection>& connection) {
    // 由于我们允许策略延长超时，因此 waitQueue 可能已经再次正常运行。在这种情况下不要触发 ANR
    if (connection->waitQueue.empty()) {
        return;
    }
     // “最旧的条目”是首次发送到应用程序的条目。但是，该条目可能不是导致超时发生的条目。
     // 一种可能性是窗口超时已更改。这可能会导致较新的条目在已分派的条目之前超时。
     // 在这种情况下，最新条目会导致 ANR。但很有可能，该应用程序会线性处理事件。
     // 因此，提供有关最早条目的信息似乎是最有用的。
    DispatchEntry* oldestEntry = *connection->waitQueue.begin();
    // 获取到超时时长
    const nsecs_t currentWait = now() - oldestEntry->deliveryTime;
    std::string reason =  
            android::base::StringPrintf("%s is not responding. Waited %" PRId64 "ms for %s",
                                        connection->inputChannel->getName().c_str(),
                                        ns2ms(currentWait),
                                        oldestEntry->eventEntry->getDescription().c_str());
    sp<IBinder> connectionToken = connection->inputChannel->getConnectionToken();
    // 生成 reason 报告
    updateLastAnrStateLocked(getWindowHandleLocked(connectionToken), reason);
    processConnectionUnresponsiveLocked(*connection, std::move(reason));
    // 停止唤醒此连接上的事件，它已经没有响应
    cancelEventsForAnrLocked(connection);
}
// 捕获 ANR 时 InputDispatcher 状态的记录。
void InputDispatcher::updateLastAnrStateLocked(const std::string& windowLabel,
                                               const std::string& reason) {
    ....
    dumpDispatchStateLocked(mLastAnrState);
}

```
### 4.3.1 dumpDispatchStateLocked

dumpDispatchStateLocked() 函数主要打印当前window和事件队列信息。执行`dumpsys input` 命令，dumpDispatchStateLocked函数输出的内容如下：

```java
Input Dispatcher State:
    ....
  PendingEvent: <none> // 当前正在调度转储事件。
  InboundQueue: <empty> // Inbound 队列
  ReplacedKeys: <empty>
  Connections:
    317: channelName='cf1eda9 com.example.anrdemo/com.example.anrdemo.MainActivity (server)', windowName='cf1eda9 com.example.anrdemo/com.example.anrdemo.MainActivity (server)', status=NORMAL, monitor=false, responsive=true
      OutboundQueue: <empty>
      WaitQueue: length=4
        MotionEvent(deviceId=9, source=0x00001002, displayId=0, action=DOWN, actionButton=0x00000000, flags=0x00000000, metaState=0x00000000, buttonState=0x00000000, classification=NONE, edgeFlags=0x00000000, xPrecision=22.8, yPrecision=10.8, xCursorPosition=nan, yCursorPosition=nan, pointers=[0: (700.0, 1633.9)]), policyFlags=0x62000000, targetFlags=0x00000105, resolvedAction=0, age=4129ms, wait=4128ms
        MotionEvent(deviceId=9, source=0x00001002, displayId=0, action=UP, actionButton=0x00000000, flags=0x00000000, metaState=0x00000000, buttonState=0x00000000, classification=NONE, edgeFlags=0x00000000, xPrecision=22.8, yPrecision=10.8, xCursorPosition=nan, yCursorPosition=nan, pointers=[0: (700.0, 1633.9)]), policyFlags=0x62000000, targetFlags=0x00000105, resolvedAction=1, age=4011ms, wait=4010ms
   ....
```

从上面可以看到InboundQueue，OutboundQueue，WaitQueue 3个Queue的状态。其中WaitQueue的size为4，即两对点击事件在等待`com.example.anrdemo`消费。    

### 4.3.2 processConnectionUnresponsiveLocked

在调用完updateLastAnrStateLocked() 后，接着调用到processConnectionUnresponsiveLocked()

```java
> frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp
  
// 该方法告诉策略连接已变得无响应，以便它可以启动 ANR。检查感兴趣的连接是监视器还是窗口，并将相应的命令条目添加到命令队列中。
void InputDispatcher::processConnectionUnresponsiveLocked(const Connection& connection,
                                                          std::string reason) {
    const sp<IBinder>& connectionToken = connection.inputChannel->getConnectionToken();
    .... 
    sendWindowUnresponsiveCommandLocked(connectionToken, pid, std::move(reason));
}

void InputDispatcher::sendWindowUnresponsiveCommandLocked(const sp<IBinder>& token,
                                                          std::optional<int32_t> pid,
                                                          std::string reason) {
    auto command = [this, token, pid, reason = std::move(reason)]() REQUIRES(mLock) {
        scoped_unlock unlock(mLock);
        mPolicy->notifyWindowUnresponsive(token, pid, reason);
    };
    postCommandLocked(std::move(command));
}
```

sendWindowUnresponsiveCommandLocked() 中将command添加到mCommandQueue队列后，最终调用到mPolicy的notifyWindowUnresponsive() 。

通过InputDispatcher头文件可以知道mPolicy是InputDispatcherPolicyInterface 接口的实例。

```java
> frameworks/native/services/inputflinger/dispatcher/InputDispatcher.h
  
class InputDispatcher : public android::InputDispatcherInterface {
.... 
private:
    sp<InputDispatcherPolicyInterface> mPolicy;
}
```

NativeInputManager 类实现了InputDispatcherPolicyInterface接口

```java
> frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp
  
class NativeInputManager : public virtual RefBase,
    public virtual InputReaderPolicyInterface,
    public virtual InputDispatcherPolicyInterface,
    public virtual PointerControllerPolicyInterface {
  ....
}
```

## 4.4 onAnrLocked（application）

我们看情况2：

```java
//情况2: 找不到focus的window，但是找到当前获取input事件的应用。
void InputDispatcher::onAnrLocked(std::shared_ptr<InputApplicationHandle> application) {
    std::string reason =
            StringPrintf("%s does not have a focused window", application->getName().c_str());
    // 收集anr的window、reason信息
    updateLastAnrStateLocked(*application, reason);
    auto command = [this, application = std::move(application)]() REQUIRES(mLock) {
        scoped_unlock unlock(mLock);
        mPolicy->notifyNoFocusedWindowAnr(application);
    };
    // 将anr的命令添加到 mCommandQueue 中
    postCommandLocked(std::move(command));
}
```

同理，这里也调用到mPolicy（即NativeInputManager）的notifyNoFocusedWindowAnr() 方法。

**onAnrLocked 方法最后会分别调用到NativeInputManager的`notifyWindowUnresponsive()` 和 `notifyNoFocusedWindowAnr()`**

## 4.5 notifyWindowUnresponsive

该方法用于通知窗口无响应

```java
> frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp
  
void NativeInputManager::notifyWindowUnresponsive(const sp<IBinder>& token,
                                                  std::optional<int32_t> pid,
                                                  const std::string& reason) {
    ....
    jobject tokenObj = javaObjectForIBinder(env, token);
    ScopedLocalRef<jstring> reasonObj(env, env->NewStringUTF(reason.c_str()));
    // 重点：这里调用到Java层 InputManagerService的notifyWindowUnresponsive方法
    env->CallVoidMethod(mServiceObj, gServiceClassInfo.notifyWindowUnresponsive, tokenObj,
                        pid.value_or(0), pid.has_value(), reasonObj.get());
}

```

gServiceClassInfo 是一个结构体

```java
> frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp
  
static struct {
    jclass clazz;
    jmethodID notifyWindowUnresponsive; // 对应java层的方法
    .... 
} gServiceClassInfo;
```

对应的clazz初始化如下

```java
> frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp
  
int register_android_server_InputManager(JNIEnv* env) {
    // Callbacks
    jclass clazz;
    FIND_CLASS(clazz, "com/android/server/input/InputManagerService");
    gServiceClassInfo.clazz = reinterpret_cast<jclass>(env->NewGlobalRef(clazz));
    ....
}
```

这样，anr的消息就抛到了java层的InputManagerService#notifyWindowUnresponsive()

## 4.6 notifyNoFocusedWindowAnr

该方法用于通知无焦点ANR

```java
> frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp

void NativeInputManager::notifyNoFocusedWindowAnr(
        const std::shared_ptr<InputApplicationHandle>& inputApplicationHandle) {
    JNIEnv* env = jniEnv();
    ScopedLocalFrame localFrame(env);
    jobject inputApplicationHandleObj =
            getInputApplicationHandleObjLocalRef(env, inputApplicationHandle);
    env->CallVoidMethod(mServiceObj, gServiceClassInfo.notifyNoFocusedWindowAnr,
                        inputApplicationHandleObj);
    checkAndClearExceptionFromCallback(env, "notifyNoFocusedWindowAnr");
}
```

同理，该方法最后通过JNI调用到Java层 InputManagerService#notifyNoFocusedWindowAnr()

## 4.7 调用栈

`能找到window的connection` 对应的调用栈

```java
services/core/jni/com_android_server_input_InputManagerService.cpp : NativeInputManager::notifyWindowUnresponsive()
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::sendWindowUnresponsiveCommandLocked()
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::processConnectionUnresponsiveLocked()
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::onAnrLocked()
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::processAnrsLocked()
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::dispatchOnce()
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::start()
```

`没有找到window，但是能找到当前应用` 对应的调用栈

```java
services/core/jni/com_android_server_input_InputManagerService.cpp : NativeInputManager::notifyNoFocusedWindowAnr()
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::onAnrLocked()
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::processAnrsLocked()
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::dispatchOnce()
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::start()
```

# 5、ANR对话框显示流程

## 5.1 InputManagerService

上面提到ANR分两种场景：                                                                                                                                                                                                                                                                                                                                                                                                                        

- 能找到window的connection，常见提示：`"%s is not responding. Waited %d ms for %s"`
- 没有找到window，但是能找到当前应用，常见异常信息：`"Application does not have a focused window"`

这里就分开讨论一下

### 5.1.1 notifyWindowUnresponsive

InputManagerService#notifyWindowUnresponsive(                                                                                                                                                                                                                                                                                                         ) 方法实现如下，很简单，只调用mWindowManagerCallbacks 做转发

```java
> frameworks/base/services/core/java/com/android/server/input/InputManagerService.java
  
public class InputManagerService {
     // Native callback
     private void notifyWindowUnresponsive(IBinder token, int pid, boolean isPidValid, String reason) {
        mWindowManagerCallbacks.notifyWindowUnresponsive(token,
                isPidValid ? OptionalInt.of(pid) : OptionalInt.empty(), reason);
    } 
}
```

在InputManagerCallback 中，将事件传给了WindowManagerService#mAnrController。

```java
> frameworks/base/services/core/java/com/android/server/wm/InputManagerCallback.java
  
final class InputManagerCallback implements InputManagerService.WindowManagerCallbacks {
     private final WindowManagerService mService;
    @Override
    public void notifyWindowUnresponsive(@NonNull IBinder token, @NonNull OptionalInt pid,
            @NonNull String reason) {
        mService.mAnrController.notifyWindowUnresponsive(token, pid, reason);
    }
}
```

mAnrController即AnrController的实例，AnrController#notifyWindowUnresponsive() 实现如下

```java
> frameworks/base/services/core/java/com/android/server/wm/AnrController.java
  
// 通知由其输入令牌标识的窗口无响应。 @return 如果窗口由给定的输入令牌标识并且请求已处理，则返回 true，否则返回 false。
private boolean notifyWindowUnresponsive(@NonNull IBinder inputToken, String reason) {
        final int pid;
        final boolean aboveSystem;
        final ActivityRecord activity;
        synchronized (mService.mGlobalLock) {
 						InputTarget target = mService.getInputTargetFromToken(inputToken);
            WindowState windowState = target.getWindowState();
            pid = target.getPid();
            // 如果输入令牌属于窗口，则归咎于Activity。如果目标是嵌入式的，那么我们就会归咎 pid。
            activity = (windowState.mInputChannelToken == inputToken)
                    ? windowState.mActivityRecord : null;
            aboveSystem = isWindowAboveSystem(windowState);
            // 调用WindowManagerService#saveANRStateLocked 保存ANR相关信息
            dumpAnrStateLocked(activity, windowState, reason);
        }
        // 这里的activity是ActivityRecord类的实例
        if (activity != null) {
            // 情况1: 能找到当前window对应的activityRecord
            activity.inputDispatchingTimedOut(reason, pid);
        } else {
            // 情况2: 找不到，直接调用mAmInternal的inputDispatchingTimedOut
            mService.mAmInternal.inputDispatchingTimedOut(pid, aboveSystem, reason);
        }
        return true;
    }
```

无论上面的activity是否为null，最终都会调用到mAmInternal.inputDispatchingTimedOut() 方法，只是传的参数不一样。

### 5.1.2 notifyNoFocusedWindowAnr

实现如下，只是作为转发给mWindowManagerCallbacks

```java
> frameworks/base/services/core/java/com/android/server/input/InputManagerService.java

    // Native callback.
    private void notifyNoFocusedWindowAnr(InputApplicationHandle inputApplicationHandle) {
        mWindowManagerCallbacks.notifyNoFocusedWindowAnr(inputApplicationHandle);
    }
```

mWindowManagerCallbacks 是一个WindowManagerCallbacks接口，对应实现类是InputManagerCallback

```java
> frameworks/base/services/core/java/com/android/server/wm/InputManagerCallback.java
  
final class InputManagerCallback implements InputManagerService.WindowManagerCallbacks {
     private final WindowManagerService mService;
    // 通知窗口管理器有关由于没有焦点窗口而没有响应的应用程序。
    @Override
    public void notifyNoFocusedWindowAnr(@NonNull InputApplicationHandle applicationHandle) {
        mService.mAnrController.notifyAppUnresponsive(
                applicationHandle, "Application does not have a focused window");
    }
}
```

mAnrController 是AnrController 的实例，notifyAppUnresponsive() 实现如下：

```java
> frameworks/base/services/core/java/com/android/server/wm/AnrController.java
    
    void notifyAppUnresponsive(InputApplicationHandle applicationHandle, String reason) {
        preDumpIfLockTooSlow();
        final ActivityRecord activity;
        synchronized (mService.mGlobalLock) {
            // 获取activity
            activity = ActivityRecord.forTokenLocked(applicationHandle.token);
            // 保存ANR 信息
            dumpAnrStateLocked(activity, null /* windowState */, reason);
            mUnresponsiveAppByDisplay.put(activity.getDisplayId(), activity);
        }
        activity.inputDispatchingTimedOut(reason, INVALID_PID);
    }
```

最后也会调用到ActivityRecord#inputDispatchingTimedOut() 方法。

## 5.2 ActivityRecord

inputDispatchingTimedOut 实现如下

```java
> frameworks/base/services/core/java/com/android/server/wm/ActivityRecord.java
 
// 当输入分派到与应用程序窗口容器关联的窗口超时时调用。
 public boolean inputDispatchingTimedOut(String reason, int windowPid) {
        ActivityRecord anrActivity;
        boolean blameActivityProcess;
        synchronized (mAtmService.mGlobalLock) {
            // 找到anr的activity
            anrActivity = getWaitingHistoryRecordLocked();
            ...
        }
        if (blameActivityProcess) {
            return mAtmService.mAmInternal.inputDispatchingTimedOut(anrApp.mOwner,
                    anrActivity.shortComponentName, anrActivity.info.applicationInfo,
                    shortComponentName, app, false, reason);
        }
       ....
 }
```

## 5.3 ActivityManagerService

inputDispatchingTimedOut 方法最后调用到ActivityManagerInternal的inputDispatchingTimedOut，ActivityManagerInternal 是一个抽象类，对应实现是LocalService

```java
> frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
  
public final class LocalService extends ActivityManagerInternal {
        @Override
        public boolean inputDispatchingTimedOut(....) {
            return ActivityManagerService.this.inputDispatchingTimedOut(....);
        }
}
```

接着来到了ActivityManagerService的inputDispatchingTimedOut

```java
> frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java

    // 处理输入调度超时。
    boolean inputDispatchingTimedOut(ProcessRecord proc, ... String reason) {
        if (reason == null) {
            annotation = "Input dispatching timed out";
        } else {
            annotation = "Input dispatching timed out (" + reason + ")";
        }
        ....
        mAnrHelper.appNotResponding(proc, activityShortComponentName, aInfo,
                    parentShortComponentName, parentProcess, aboveSystem, annotation);
    }
```

## 5.4 AnrHelper

AnrHelper.appNotResponding方法实现如下

```java
> frameworks/base/services/core/java/com/android/server/am/AnrHelper.java
 
    void appNotResponding(com.android.server.am.ProcessRecord anrProcess, String activityShortComponentName,
                          ApplicationInfo aInfo, String parentShortComponentName,
                          WindowProcessController parentProcess, boolean aboveSystem, String annotation) {
        final int incomingPid = anrProcess.mPid;
        synchronized (mAnrRecords) {
            ....
            // 将anr信息添加到list中
            mAnrRecords.add(new AnrRecord(anrProcess, activityShortComponentName ....));
        }
        // 启动anr检查线程
        startAnrConsumerIfNeeded();
    }
  
    private void startAnrConsumerIfNeeded() {
        if (mRunning.compareAndSet(false, true)) {
            new AnrConsumerThread().start();
        }
    }
```

AnrConsumerThread 定义如下，该线程主要是遍历mAnrRecords，然后调用AnrRecord的appNotResponding方法。

```java
> frameworks/base/services/core/java/com/android/server/am/AnrHelper.java
   
  private class AnrConsumerThread extends Thread {
        .... 
        @Override
        public void run() {
            AnrRecord r;
            while ((r = next()) != null) {
                ....
                // 是否要求仅转储自身
                final boolean onlyDumpSelf = reportLatency > EXPIRED_REPORT_TIME_MS;
                r.appNotResponding(onlyDumpSelf);
                .....
            }
        }
    }
```

AnrRecord.appNotResponding() 实现如下

```java
> frameworks/base/services/core/java/com/android/server/am/AnrHelper.java
  
private static class AnrRecord {
        final com.android.server.am.ProcessRecord mApp;
        void appNotResponding(boolean onlyDumpSelf) {
            mApp.mErrorState.appNotResponding(mActivityShortComponentName, mAppInfo,
                    mParentShortComponentName, mParentProcess, mAboveSystem, mAnnotation,
                    onlyDumpSelf);
        }
}
```

mErrorState 是 ProcessErrorStateRecord 类的实例。

## 5.5 ProcessErrorStateRecord

ProcessErrorStateRecord.appNotResponding() 方法很长，主要做

- 将 ANR 记录到主日志中。
- 转储堆栈信息到跟踪文件中
- 发出显示anr对话框的消息

```java
> frameworks/base/services/core/java/com/android/server/am/ProcessErrorStateRecord.java

void appNotResponding(String activityShortComponentName, ApplicationInfo aInfo ...) {
        ....
        synchronized (mService) {
           // 如果是后台anr，就直接kill掉app，并打印原因
            if (isSilentAnr() && !mApp.isDebugging()) {
                mApp.killLocked("bg anr", ApplicationExitInfo.REASON_ANR, true);
                return;
            }
            synchronized (mProcLock) {
                // 设置app的notResponding状态，并查找errorReportReceiver
                makeAppNotRespondingLSP(activityShortComponentName,
                        annotation != null ? "ANR " + annotation : "ANR", info.toString());
                mDialogController.setAnrController(anrController);
            }
            if (mService.mUiHandler != null) {
                // 调出臭名昭著的 App Not Responding 对话框
                Message msg = Message.obtain();
                msg.what = ActivityManagerService.SHOW_NOT_RESPONDING_UI_MSG;
                msg.obj = new AppNotRespondingDialog.Data(mApp, aInfo, aboveSystem);
                mService.mUiHandler.sendMessageDelayed(msg, anrDialogDelayMs);
            }
        }
}
```

mUiHandler 是Activity Manager Service中的handler，其实现如下

```java
> frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
  
final class UiHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                .....
                case SHOW_NOT_RESPONDING_UI_MSG: {
                   // 处理显示ANR对话框的消息
                    mAppErrors.handleShowAnrUi(msg);
                } break;
            }
        }
}
```

## 5.6 AppErrors

handleShowAnrUi对应实现如下，主要用于判断是否需要显示ANR对话框

```java
> frameworks/base/services/core/java/com/android/server/am/AppErrors.java

void handleShowAnrUi(Message msg) {
   final ProcessErrorStateRecord errState = proc.mErrorState;
   synchronized (mProcLock) {
            if (errState.getDialogController().hasAnrDialogs()) {
                // 已经显示ANR对话框，就直接return
                MetricsLogger.action(mContext, MetricsProto.MetricsEvent.ACTION_APP_ANR,
                        AppNotRespondingDialog.ALREADY_SHOWING);
                return;
            }
            // 满足弹出ANR对话框的条件
            if (mService.mAtmInternal.canShowErrorDialogs() || showBackground) {
                 ....
                 // 调用controler 显示对话框
                 errState.getDialogController().showAnrDialogs(data);
            } 
            ...
        }
}
```

ProcessErrorStateRecord.getDialogController() 方法返回ErrorDialogController 对象

## 5.7 ErrorDialogController

ErrorDialogController 主要就是控制对话框的显示跟隐藏。

```java
> frameworks/base/services/core/java/com/android/server/am/ErrorDialogController.java
  
void showAnrDialogs(AppNotRespondingDialog.Data data) {
     List<Context> contexts = getDisplayContexts(
             mApp.mErrorState.isSilentAnr() /* lastUsedOnly */);
     mAnrDialogs = new ArrayList<>();
     for (int i = contexts.size() - 1; i >= 0; i--) {
         final Context c = contexts.get(i);
         // 创建新的ANR 对话框，AppNotRespondingDialog 即为我们见到的anr对话框
         mAnrDialogs.add(new AppNotRespondingDialog(mService, c, data));
     }
     // 显示dialog
     scheduleForAllDialogs(mAnrDialogs, Dialog::show);
}

void scheduleForAllDialogs(List<? extends BaseErrorDialog> dialogs,
          Consumer<BaseErrorDialog> c) {
     mService.mUiHandler.post(() -> {
         if (dialogs != null) {
             forAllDialogs(dialogs, c);
         }
     });
 }
// 遍历 dialog列表，调用show方法
void forAllDialogs(List<? extends BaseErrorDialog> dialogs, Consumer<BaseErrorDialog> c) {
     for (int i = dialogs.size() - 1; i >= 0; i--) {
         c.accept(dialogs.get(i));
     }
}
```

## 5.8 AppNotRespondingDialog

ANR对话框的实现如下

```java
> services/core/java/com/android/server/am/AppNotRespondingDialog.java
  
final class AppNotRespondingDialog extends BaseErrorDialog implements View.OnClickListener {
    ....
    @Override
    public void onClick(View v) {
        switch (v.getId()) {
            case com.android.internal.R.id.aerr_report:
                 // 点击了等待并且上报，该按钮只有在有ErrorReportReceiver的时候才显示
                mHandler.obtainMessage(WAIT_AND_REPORT).sendToTarget();
                break;
            case com.android.internal.R.id.aerr_close:
                // 将app杀死
                mHandler.obtainMessage(FORCE_CLOSE).sendToTarget();
                break;
            case com.android.internal.R.id.aerr_wait:
                // 继续等待
                mHandler.obtainMessage(WAIT).sendToTarget();
                break;
            default:
                break;
        }
    }
  
    private final Handler mHandler = new Handler() {
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case FORCE_CLOSE:
                    // 调用ActivityManagerService的killAppAtUsersRequest方法，将app kill掉
                    mService.killAppAtUsersRequest(mProc);
                    break;
                case WAIT_AND_REPORT:
                case WAIT:
                    // 继续等待应用
                    synchronized (mService) {
                        ProcessRecord app = mProc;
                        final ProcessErrorStateRecord errState = app.mErrorState;
                        if (msg.what == WAIT_AND_REPORT) {
                            appErrorIntent = mService.mAppErrors.createAppErrorIntentLOSP(app,
                                    System.currentTimeMillis(), null);
                        }

                        synchronized (mService.mProcLock) {
                            // 清除anr的标识位
                            errState.setNotResponding(false);
                            // 退出对话框
                            errState.getDialogController().clearAnrDialogs();
                        }
                        // 重新计算service anr的时间
                        mService.mServices.scheduleServiceTimeoutLocked(app);
                        // If the app remains unresponsive, show the dialog again after a delay.
                        mService.mInternal.rescheduleAnrDialog(mData);
                    }
                    break;
            }

            if (appErrorIntent != null) {
                try {
                    getContext().startActivity(appErrorIntent);
                } catch (ActivityNotFoundException e) {
                    Slog.w(TAG, "bug report receiver dissappeared", e);
                }
            }

            dismiss();
        }
    };
}
```

## 5.9 cancelEventsForAnrLocked

cancelEventsForAnrLocked方法用于停止唤醒此连接上的事件，这些事件已经没有响应。

```java
void InputDispatcher::cancelEventsForAnrLocked(const sp<Connection>& connection) {
    // 我们不会在这里中断任何连接，即使策略希望我们中止调度。如果策略决定关闭应用，
    // 我们将通过 unregisterInputChannel 获取通道删除事件，并以这种方式清理连接。
    // 当连接阻塞时，我们已经没有向连接发送新的指针，但重点事件将继续堆积。
    ALOGW("Canceling events for %s because it is unresponsive",
          connection->inputChannel->getName().c_str());
    if (connection->status == Connection::Status::NORMAL) {
        CancelationOptions options(CancelationOptions::CANCEL_ALL_EVENTS,
                                   "application not responding");
        synthesizeCancelationEventsForConnectionLocked(connection, options);
    }
}
```

## 5.10 调用栈

找到window场景，Java层的调用堆栈如下所示

```java
services/core/java/com/android/server/am/AppNotRespondingDialog.java : show()
services/core/java/com/android/server/am/ErrorDialogController.java : showAnrDialogs()
services/core/java/com/android/server/am/AppErrors.java : handleShowAnrUi()
services/core/java/com/android/server/am/ActivityManagerService.java : UiHandler.SHOW_NOT_RESPONDING_UI_MSG
services/core/java/com/android/server/am/ProcessErrorStateRecord.java : appNotResponding() 
services/core/java/com/android/server/am/AnrHelper.java : AnrRecord.appNotResponding()
services/core/java/com/android/server/am/AnrHelper.java : AnrConsumerThread.run()
services/core/java/com/android/server/am/AnrHelper.java : startAnrConsumerIfNeeded()
services/core/java/com/android/server/am/AnrHelper.java : appNotResponding()
services/core/java/com/android/server/am/ActivityManagerService.java : inputDispatchingTimedOut()
services/core/java/com/android/server/am/ActivityManagerService.java : LocalService.inputDispatchingTimedOut()
services/core/java/com/android/server/wm/ActivityRecord.java : inputDispatchingTimedOut()
services/core/java/com/android/server/wm/AnrController.java : notifyWindowUnresponsive()
services/core/java/com/android/server/wm/InputManagerCallback.java : notifyWindowUnresponsive()
services/core/java/com/android/server/input/InputManagerService.java : notifyWindowUnresponsive()

```

没有找到connection，但是找到对应应用，Java层的调用堆栈如下所示。

可以看到两者的调用栈区别并不大，主要是在InputManagerService中调用栈不一样。

```java
services/core/java/com/android/server/am/AppNotRespondingDialog.java : show()
services/core/java/com/android/server/am/ErrorDialogController.java : showAnrDialogs()
services/core/java/com/android/server/am/AppErrors.java : handleShowAnrUi()
services/core/java/com/android/server/am/ActivityManagerService.java : UiHandler.SHOW_NOT_RESPONDING_UI_MSG
services/core/java/com/android/server/am/ProcessErrorStateRecord.java : appNotResponding() 
services/core/java/com/android/server/am/AnrHelper.java : AnrRecord.appNotResponding()
services/core/java/com/android/server/am/AnrHelper.java : AnrConsumerThread.run()
services/core/java/com/android/server/am/AnrHelper.java : startAnrConsumerIfNeeded()
services/core/java/com/android/server/am/AnrHelper.java : appNotResponding()
services/core/java/com/android/server/am/ActivityManagerService.java : inputDispatchingTimedOut()
services/core/java/com/android/server/am/ActivityManagerService.java : LocalService.inputDispatchingTimedOut()
services/core/java/com/android/server/wm/ActivityRecord.java : inputDispatchingTimedOut()
services/core/java/com/android/server/wm/AnrController.java : notifyAppUnresponsive()
services/core/java/com/android/server/wm/InputManagerCallback.java : notifyNoFocusedWindowAnr()
services/core/java/com/android/server/input/InputManagerService.java : notifyNoFocusedWindowAnr()
```

# 6、总结

## 6.1 问题解答

先回答之前的两个问题

问题1：如果在activity任意周期（onCreate,onResume等），同步执行耗时超过5s（ANR时间）的任务，期间不进行点击，那会触发ANR吗？

答：不会。

原因：ANR的条件是waitQueue 不为空，在activity启动过程，没有触发按键事件的分发，也就 自然不会调用ANR检查流程，即processAnrsLocked()



问题2：如果在button点击的时候，在onClick回调同步执行耗时超过5s的任务。点击一次会触发ANR吗？点击2次呢，3次呢？

答：点击一次不会触发ANR，在上个耗时结束之前，再次点击会导致ANR。原因：onClilck事件是在action up的时候，通过主线程的handler重新post一下触发的。

```java
public boolean onTouchEvent(MotionEvent event) {
             ....
             switch (action) {
                case MotionEvent.ACTION_UP:
                       if (mPerformClick == null) {
                           mPerformClick = new PerformClick();
                       }
                       // 通过post延迟 触发点击事件
                       if (!post(mPerformClick)) { 
                           performClickInternal();
                       }
                .....
}
```
故在第一次点击的时候，waitQueue为空；

在第二次点击的时候，由于主线程正被卡住，waitQueue 这个时候会存放两个事件action down和action up，如下图所示：

![](\img/blog_activity_anr/4.png)

等到第二次input事件超时后，通过looper再次调用到dispatchOnce()方法，在processAnrsLocked中检测是否发生anr：通过mAnrTracker获取到最近超时时间，发现当前时间大于超时时间，就会触发ANR。
```java
nsecs_t InputDispatcher::processAnrsLocked() {
    ....
    nextAnrCheck = std::min(nextAnrCheck, mAnrTracker.firstTimeout());
    if (currentTime < nextAnrCheck) { // 这个时候当前时间是大于nextAnrCheck 
        return nextAnrCheck;         
    }
    // 触发anr
    onAnrLocked(connection);
    return LONG_LONG_MIN;
}
```

## 6.2 Input ANR分类

当发生ANR的时候，System log 会出现如下信息，tag是 ActivityManager

```java
"Input dispatching timed out (" + reason + ")"
```

其中reason取值如下：

- 能找到focusedWindow：`"%s is not responding. Waited %d ms for %s"`
- 不能找到focusedWindow，但是找到focused应用：`Application does not have a focused window`

在Android13 之前，这种分类的异常log是

```java
Waiting because no window has focus but %s may eventually add a window when it finishes starting up. Will wait for %d ms 
```

- 其他类型：reason为null。

## 6.3 Input ANR 条件

- 能找到window：waitQueue 不为空，并且当前时间 > 最早分发事件的超时时间。
- 找不到window，但是找到focused应用：当前时间 > 等待FocusedWindow的时间。

## 6.4 修改Input超时时间

这个必须是厂家或者有root权限才能修改。

- 通过修改 UNMULTIPLIED_DEFAULT_DISPATCHING_TIMEOUT_MILLIS 的值来改变timeOut值。
- 通过修改ro.hw_timeout_multiplier 属性值（倍数，int值）来修改，需要root权限，默认是1倍。

# 参考文章：

- [Input系统—ANR原理分析](http://gityuan.com/2017/01/01/input-anr/)


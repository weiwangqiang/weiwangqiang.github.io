---
layout:     post
title:      "Activity anr原理分析"
subtitle:   " \"从点击事件卡顿到显示ANR对话框，你知道Android都做了哪些工作吗？\""
date:       2021-12-29 13:30:00
author:     "Weiwq"
header-img: "img/background/home-bg-o.jpg"
catalog:  true
top: false
tags:
    - Android
---

> “本文基于Android13源码，分析Input系统的anr实现原理“
## 源码列表

  主要涉及到的源码

```java
frameworks/native/services/inputflinger/dispatcher/
    - InputDispatcher.cpp
    - Connection.cpp
    - EventHub.cpp
    
frameworks/native/services/inputflinger/reader/
    - InputReader.cpp
    - InputDevice.cpp
    - mapper/KeyboardInputMapper.cpp

```

## anr 分类

首先简单描述一下anr的分类：
- Input ANR：按键或触摸事件在5s内没有相应，常发生在activity中。
- Service anr：前台service 响应时间是20s，后台service是200s；startForground超时是5s。
- Broadcast anr：前台广播是10s，后台广播是60s。
- ContentProvider anr：publish执行未在10s内完成。
- startForgoundService：应用调用startForegroundService，然后5s内未调用startForeground出现ANR或者Crash

有些小伙伴可能好奇，为啥没有Activity ANR的分类？Activity ANR准确的来说是——Input系统检测，触发activity 的anr。所以本文将通过input系统是如何触发activity发生anr的。

### InputReader

Inputreader主要的作用是：

- 读取节点/dev/input，将Input_event 结构体转成相应的EventEntry，比如按键事件对应KeyEntry，触摸事件对应MotionEntry
- 将事件添加到mInboundQueue队列尾部。
- KeyboardInputMapper.processKey()的过程, 记录下按下down事件的时间点。

![](D:\myBlog\weiwangqiang.github.io\img/blog_activity_anr/1.jpg)

## InputDispatcher

Inputdispatcher中，在线程里面调用到dispatchOnce方法，该方法中主要做：

- 通过dispatchOnceInnerLocked()，取出mInboundQueue 里面的 EventEntry事件
- 通过enqueueDispatchEntryLocked()，生成事件DispatchEntry并加入connection的`outbound`队列。
- 通过startDispatchCycleLocked()，从outboundQueue中取出事件DispatchEntry, 重新放入connection的`waitQueue`队列。
- 通过runCommandsLockedInterruptable()，遍历mCommandQueue队列，依次处理所有命令。
- 通过processAnrsLocked()，判断是否需要触发ANR。
- 在startDispatchCycleLocked()里面，通过inputPublisher.publishKeyEvent() 方法将按键事件分发给java层。publishKeyEvent的实现是在InputTransport.cpp 中

通过上面的概括，可以知道按键事件主要存储在3个queue中：

1. InputDispatcher的mInboundQueue：存储的是从InputReader 送来的输入事件。
2. Connection的outboundQueue：该队列是存储即将要发送给应用的输入事件。
3. Connection的waitQueue：队列存储的是已经发给应用的事件，但是应用还未处理完成的。

![](D:\myBlog\weiwangqiang.github.io\img/blog_activity_anr/3.png)

### dispatchOnce

dispatchOnce 中主要就是调用如下的两个方法，一个是事件分发，一个是检查ANR

```java
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

我们先简单看看事件分发过程

## 事件分发

dispatchOnce 中，通过调用dispatchOnceInnerLocked来分发事件

### dispatchOnceInnerLocked

dispatchOnceInnerLocked主要是：

1）从mInboundQueue 中取出mPendingEvent

2）通过mPendingEvent的type决定事件类型和分发方式。比如当前是key类型。

3）最后如果处理了事件，就处理相关的回收。

主要代码如下：

```java
> services/inputflinger/dispatcher/InputDispatcher.cpp
  
void InputDispatcher::dispatchOnceInnerLocked(nsecs_t* nextWakeupTime) {
    nsecs_t currentTime = now();
    ...
    // 优化应用切换的延迟。本质上，当按下应用程序切换键（HOME）时，我们会开始一个短暂的超时。
    // 当它过期时，我们会抢占调度并删除所有其他挂起的事件。
    bool isAppSwitchDue = mAppSwitchDueTime <= currentTime;
    // 当前没有PendingEvent（即EventEntry），则取一个
    if (!mPendingEvent) {
        // 1、mInboundQueue 为空
        if (mInboundQueue.empty()) {
            // 如果适用，合成键重复。
            if (mKeyRepeatState.lastKeyEntry) {
                if (currentTime >= mKeyRepeatState.nextRepeatTime) {
                    mPendingEvent = synthesizeKeyRepeatLocked(currentTime);
                }
                ...
            }
            // 如果没有PendingEvent，就直接返回
            if (!mPendingEvent) {
                return;
            }
        } else {
        // 2、mInboundQueue不为空 ，就从队列前面取一个PendingEvent
            mPendingEvent = mInboundQueue.front();
            mInboundQueue.pop_front();
            traceInboundQueueLengthLocked();
        }
        // Poke user activity for this event.
        if (mPendingEvent->policyFlags & POLICY_FLAG_PASS_TO_USER) {
            // 根据当前的event 类型，post 一个 command 到 mCommandQueue
            pokeUserActivityLocked(*mPendingEvent);
        }
    }
    // 现在我们有一个事件要发送。所有事件最终都会以这种方式取消排队和处理，即使我们打算删除它们。
    ...
    bool done = false;
    DropReason dropReason = DropReason::NOT_DROPPED;
    switch (mPendingEvent->type) {
        ...
        case EventEntry::Type::KEY: {
            ...
            // 最后会调用到dispatchEventLocked
            done = dispatchKeyLocked(currentTime, keyEntry, &dropReason, nextWakeupTime);
            break;
        }
        case EventEntry::Type::MOTION: {
            ...
            // 最后会调用到dispatchEventLocked
            done = dispatchMotionLocked(currentTime, motionEntry, &dropReason, nextWakeupTime);
            break;
        }
    }
    if (done) { 
        if (dropReason != DropReason::NOT_DROPPED) {
            // 事件没有被丢弃。找到对应的原因并通知
            dropInboundEventLocked(*mPendingEvent, dropReason);
        }
        mLastDropReason = dropReason;
        // 将mPendingEvent 置为 Null，方便下次重新获取
        releasePendingEventLocked();
        *nextWakeupTime = LONG_LONG_MIN; // force next poll to wake up immediately
    }
}
```

从上面备注可以知道，MTION和KEY类型的事件都会调用到dispatchKeyLocked。

 ### dispatchEventLocked

dispatchEventLocked 主要是遍历inputTargets，通过prepareDispatchCycleLocked分发事件。prepareDispatchCycleLocked内部又会调用enqueueDispatchEntriesLocked方法

```java
> frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp
  
void InputDispatcher::dispatchEventLocked(nsecs_t currentTime,
                                          std::shared_ptr<EventEntry> eventEntry,
                                          const std::vector<InputTarget>& inputTargets) {
    ...
    for (const InputTarget& inputTarget : inputTargets) {
        sp<Connection> connection =
                getConnectionLocked(inputTarget.inputChannel->getConnectionToken());
        if (connection != nullptr) {
            prepareDispatchCycleLocked(currentTime, connection, eventEntry, inputTarget);
        }
    }
}
```

### enqueueDispatchEntriesLocked

主要做两个事情：1）将请求模式的调度条目排队。2）启动调度周期。

```java
> frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp
  
void InputDispatcher::enqueueDispatchEntriesLocked(nsecs_t currentTime,
                                                   const sp<Connection>& connection,
                                                   std::shared_ptr<EventEntry> eventEntry,
                                                   const InputTarget& inputTarget) {
    bool wasEmpty = connection->outboundQueue.empty();
    // 将请求模式的调度条目排队。
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
                               InputTarget::FLAG_DISPATCH_AS_HOVER_EXIT);
    ...
    // 如果出站队列以前为空，请开始调度周期。
    if (wasEmpty && !connection->outboundQueue.empty()) {
        startDispatchCycleLocked(currentTime, connection);
    }
}
```

### enqueueDispatchEntryLocked

enqueueDispatchEntryLocked 会创建一个新的DispatchEntry，然后将DispatchEntry 加入到connection的outboundQueue 中

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

### startDispatchCycleLocked

该方法主要通过connection 发布最终的事件，至此，InputDispatcher完成事件的发布，并且将发布的事件保存在connection的waitQueue中。

```java
> frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp
  
void InputDispatcher::startDispatchCycleLocked(nsecs_t currentTime,
                                               const sp<Connection>& connection) {
    while (connection->status == Connection::Status::NORMAL && !connection->outboundQueue.empty()) {
         // 从outboundQueue 队列中取出 DispatchEntry
        DispatchEntry* dispatchEntry = connection->outboundQueue.front();
        // 发布事件
        status_t status;
        const EventEntry& eventEntry = *(dispatchEntry->eventEntry);
        switch (eventEntry.type) {
            case EventEntry::Type::KEY: {
                ...
                // 发布按键事件
                status = connection->inputPublisher
                                 .publishKeyEvent(dispatchEntry->seq ...);
                break;
            }

            case EventEntry::Type::MOTION: {
                ...
                // 发布运动事件。
                status = connection->inputPublisher
                                 .publishMotionEvent(dispatchEntry->seq ...);
                break;
            }
            ....
        }
        // 如果status 已赋值
        if (status) {
            if (status == WOULD_BLOCK) {
                if (connection->waitQueue.empty()) {
                    // pip 管道满了导致无法发布事件。
                    // 该问题是出乎意料的，因为等待队列是空的，所以管道应该是空的
                    
                    // 将outboundQueue，waitQueue 队列清空，并释放队列中的DispatchEntry 对象
                    abortBrokenDispatchCycleLocked(currentTime, connection, true /*notify*/);
                } else {
                    // 管道已满，并且waitQueue 不为空，我们正在等待应用程序完成处理一些事件，然后再向其发送更多事件。
                }
            } else {
                // 将outboundQueue，waitQueue 队列清空，并释放队列中的DispatchEntry 对象
                abortBrokenDispatchCycleLocked(currentTime, connection, true /*notify*/);
            }
            return;
        }
        // 在等待队列上重新排队事件。
        connection->outboundQueue.erase(std::remove(connection->outboundQueue.begin(),
                                                    connection->outboundQueue.end(),
                                                    dispatchEntry));
        // 在waitQueue 尾部重新插入
        connection->waitQueue.push_back(dispatchEntry);
    }
}
```

### publishMotionEvent

查看connection的头文件，可以知道connection->inputPublisher 是InputPublisher类

```java
> frameworks/native/services/inputflinger/dispatcher/Connection.h

class Connection : public RefBase {
public:
    InputPublisher inputPublisher;
}
```

 InputPublisher class的定义如下，负责将输入事件发布到输入通道。

```java
> frameworks/native/include/input/InputTransport.h
  
// 负责将输入事件发布到输入通道。
class InputPublisher {
 ....
private:
    std::shared_ptr<InputChannel> mChannel;
}

class InputChannel : public Parcelable {
}
```

publishMotionEvent 对应的实现如下

```java
> frameworks/native/libs/input/InputTransport.cpp
  
status_t InputPublisher::publishMotionEvent(
             uint32_t seq, int32_t eventId, int32_t deviceId, int32_t source, int32_t displayId ..) {
    ....
    InputMessage msg;
    // 设置event的参数
    msg.header.type = InputMessage::Type::MOTION;
    msg.body.motion.action = action;
    ....
    // 调用InputChannel的sendMessage方法
    return mChannel->sendMessage(&msg);
}
```

sendMessage 方法如下

```
status_t InputChannel::sendMessage(const InputMessage* msg) {
}
```



### runCommandsLockedInterruptable

dispatchOnceInnerLocked已经分析完，我们再次回到dispatchOnce，分析runCommandsLockedInterruptable方法

```java
> frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp
  
void InputDispatcher::dispatchOnce() {
    nsecs_t nextWakeupTime = LONG_LONG_MAX;
    {
        ...
        // 运行所有挂起的命令（如果有）。如果运行了任何命令，则强制下一次轮询立即唤醒。
        if (runCommandsLockedInterruptable()) {
            nextWakeupTime = LONG_LONG_MIN;
        }
        ...
    }
}
```

该方法很简单，就是遍历mCommandQueue 执行对应的command。

```java
> frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp
  
bool InputDispatcher::runCommandsLockedInterruptable() {
    do {
        auto command = std::move(mCommandQueue.front());
        mCommandQueue.pop_front();
        // Commands are run with the lock held, but may release and re-acquire the lock from within.
        // 命令在锁保持的情况下运行，但可能会从内部释放并重新获取锁。
        command();
    } while (!mCommandQueue.empty());
    return true;
}
```

通过postCommandLocked 方法将command添加到队列中

```java
void InputDispatcher::postCommandLocked(Command&& command) {
    mCommandQueue.push_back(command);
}
```

### 调用栈

native层的事件分发调用栈如下

```java
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::startDispatchCycleLocked()
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::enqueueDispatchEntriesLocked()
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::prepareDispatchCycleLocked()
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::dispatchKeyLocked()
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::dispatchOnceInnerLocked()
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::dispatchOnce()
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::start()
```



## ANR触发

回到dispatchOnce方法，在新的唤醒中，会调用processAnrsLocked 方法来决定是否需要触发anr

```java
> frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp
  
void InputDispatcher::dispatchOnce() {
    nsecs_t nextWakeupTime = LONG_LONG_MAX;
    {
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

### processAnrsLocked

该方法是用于检查队列中是否有太旧的事件，如果存在就触发ANR

```java
> frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp
  
// 检查是否有任何连接的等待队列具有太旧的事件。如果我们等待事件被确认的时间超过窗口超时，
 // 请引发 ANR。返回我们下次应该醒来的时间。
nsecs_t InputDispatcher::processAnrsLocked() {
    const nsecs_t currentTime = now();
    nsecs_t nextAnrCheck = LONG_LONG_MAX; // 下一次anr超时的时间
    // 检查我们是否正在等待一个聚焦窗口出现。如果等待时间过长就报 ANR
    if (mNoFocusedWindowTimeoutTime.has_value() && mAwaitedFocusedApplication != nullptr) {
        if (currentTime >= *mNoFocusedWindowTimeoutTime) {
            processNoFocusedWindowAnrLocked();
            mAwaitedFocusedApplication.reset();
            mNoFocusedWindowTimeoutTime = std::nullopt;
            return LONG_LONG_MIN;
        } else {
            // 请继续等待。我们将在mNoFocusedWindowTimeoutTime到来时放弃该事件。
            nextAnrCheck = *mNoFocusedWindowTimeoutTime;
        }
    }
    // 检查是否有任何连接 ANR 到期
    nextAnrCheck = std::min(nextAnrCheck, mAnrTracker.firstTimeout());
    if (currentTime < nextAnrCheck) { // 最有可能的情况
        // 一切正常，在 nextAnrCheck 再检查一次
        return nextAnrCheck;
    }
    // 如果我们到达这里，则连接无响应。
    sp<Connection> connection = getConnectionLocked(mAnrTracker.firstToken());
    if (connection == nullptr) {
       // 获取不到事件的连接
        return nextAnrCheck;
    }
    connection->responsive = false;
    // 停止为此无响应的连接唤醒
    mAnrTracker.eraseToken(connection->inputChannel->getConnectionToken());
    onAnrLocked(connection);
    return LONG_LONG_MIN;
}
```

在找到connection，并且ANR时间已经超时，就会调用到onAnrLocked。

### onAnrLocked

onAnrLocked 有两种实现：

- 能找到当前focus的window
- 找不到当前focus的window，但是可以找到当前前台应用。

我们在processAnrsLocked 能找到对应的window，所以先看情况1

```java
> frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp
  
//情况1、 能找到window的情况
void InputDispatcher::onAnrLocked(const sp<Connection>& connection) {
    // 由于我们允许策略延长超时，因此 waitQueue 可能已经再次正常运行。在这种情况下不要触发 ANR
    if (connection->waitQueue.empty()) {
        ALOGI("Not raising ANR because the connection %s has recovered",
              connection->inputChannel->getName().c_str());
        return;
    }
     // “最旧的条目”是首次发送到应用程序的条目。但是，该条目可能不是导致超时发生的条目。
     // 一种可能性是窗口超时已更改。这可能会导致较新的条目在已分派的条目之前超时。
     // 在这种情况下，最新条目会导致 ANR。但很有可能，该应用程序会线性处理事件。
     // 因此，提供有关最早条目的信息似乎是最有用的。
    DispatchEntry* oldestEntry = *connection->waitQueue.begin();
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

//情况2、 找不到focus的window，但是找到当前获取input事件的应用。
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
// 捕获 ANR 时 InputDispatcher 状态的记录。
void InputDispatcher::updateLastAnrStateLocked(const std::string& windowLabel,
                                               const std::string& reason) {
    ....
    dumpDispatchStateLocked(mLastAnrState);
}

```
### dumpDispatchStateLocked

dumpDispatchStateLocked 函数主要打印当前window和事件队列信息。执行`dumpsys input` 命令，dumpDispatchStateLocked函数输出的内容如下：

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

### processConnectionUnresponsiveLocked

在调用完updateLastAnrStateLocked 后，接着调用到processConnectionUnresponsiveLocked

```java
> frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp
  
void InputDispatcher::onAnrLocked(std::shared_ptr<InputApplicationHandle> application) {
    ....
    processConnectionUnresponsiveLocked(*connection, std::move(reason));
    ....
}

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

sendWindowUnresponsiveCommandLocked 中将command添加到mCommandQueue队列后，最终调用到mPolicy的notifyWindowUnresponsive 。

通过InputDispatcher头文件可以知道mPolicy是InputDispatcherPolicyInterface 接口的实例，对应的实现类是NativeInputManager

```java
> frameworks/native/services/inputflinger/dispatcher/InputDispatcher.h
  
class InputDispatcher : public android::InputDispatcherInterface {
.... 
private:
    sp<InputDispatcherPolicyInterface> mPolicy;
}
```

### NativeInputManager

NativeInputManager 类实现了InputDispatcherPolicyInterface接口

```JAVA
> frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp
  
class NativeInputManager : public virtual RefBase,
    public virtual InputReaderPolicyInterface,
    public virtual InputDispatcherPolicyInterface,
    public virtual PointerControllerPolicyInterface {
  ....
}
```

notifyWindowUnresponsive方法实现如下

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

这样，anr的消息就抛到了java层的InputManagerService中

### 调用栈

触发ANR流程中，native层的调用栈如下：

```java
services/core/jni/com_android_server_input_InputManagerService.cpp : NativeInputManager::notifyWindowUnresponsive()
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::sendWindowUnresponsiveCommandLocked()
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::processConnectionUnresponsiveLocked()
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::onAnrLocked()
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::processAnrsLocked()
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::dispatchOnce()
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::start()
```



## ANR对话框显示流程

### InputManagerService

InputManagerService的notifyWindowUnresponsive方法实现如下，很简单，也是调用mWindowManagerCallbacks 做转发

```java
> frameworks/base/services/core/java/com/android/server/input/InputManagerService.java
  
public class InputManagerService {
     // Native callback
     private void notifyWindowUnresponsive(IBinder token, int pid, boolean isPidValid,
                                            String reason) {
        mWindowManagerCallbacks.notifyWindowUnresponsive(token,
                isPidValid ? OptionalInt.of(pid) : OptionalInt.empty(), reason);
    } 
}

```

mWindowManagerCallbacks 是一个WindowManagerCallbacks接口，对应实现类是InputManagerCallback

在InputManagerCallback 中，将事件传给了WindowManagerService的mAnrController。

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

### AnrController

notifyWindowUnresponsive 实现如下

```java
> frameworks/base/services/core/java/com/android/server/wm/AnrController.java
  
// 通知由其输入令牌标识的窗口无响应。 @return 如果窗口由给定的输入令牌标识并且请求已处理，则返回 true，否则返回 false。
private boolean notifyWindowUnresponsive(@NonNull IBinder inputToken, String reason) {
        preDumpIfLockTooSlow();
        final int pid;
        final boolean aboveSystem;
        final ActivityRecord activity;
        synchronized (mService.mGlobalLock) {
            com.android.server.wm.InputTarget target = mService.getInputTargetFromToken(inputToken);
            if (target == null) {
                return false;
            }
            WindowState windowState = target.getWindowState();
            pid = target.getPid();
            // Blame the activity if the input token belongs to the window. If the target is
            // embedded, then we will blame the pid instead.
            // 如果输入令牌属于窗口，则归咎于活动。如果目标是嵌入式的，那么我们就会责怪 pid。
            activity = (windowState.mInputChannelToken == inputToken)
                    ? windowState.mActivityRecord : null;
            Slog.i(TAG_WM, "ANR in " + target + ". Reason:" + reason);
            aboveSystem = isWindowAboveSystem(windowState);
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

我们直接看能找到activityRecord的情况

### ActivityRecord

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

### ActivityManagerService

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

### AnrHelper

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

### ProcessErrorStateRecord

ProcessErrorStateRecord.appNotResponding() 方法很长，主要做

- 将 ANR 记录到主日志中。
- 转储堆栈信息到跟踪文件中
- 发出显示anr对话框的消息

```JAVA
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

### AppErrors

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

### ErrorDialogController

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

### AppNotRespondingDialog

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

### 调用栈

Java层的调用堆栈如下所示

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

### cancelEventsForAnrLocked

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



## 参考文献：

- [Input系统—ANR原理分析](http://gityuan.com/2017/01/01/input-anr/)
- [Android input anr 分析](https://zhuanlan.zhihu.com/p/53331495)
- [【CSDN】源码剖析Android  anr机制](https://blog.csdn.net/xt_xiaotian/article/details/121250498)


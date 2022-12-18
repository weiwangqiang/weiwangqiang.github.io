---
layout:     post
title:      "Activity anr原理分析"
subtitle:   " \"带你剖析activity anr的实现\""
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

## Input 系统

先简单分析一下Input系统的实现

### InputReader

Inputreader主要的作用是：

- 读取节点/dev/input，将Input_event 结构体转成相应的EventEntry，比如按键事件对应KeyEntry，触摸事件对应MotionEntry
- 将事件添加到mInboundQueue队列尾部。
- KeyboardInputMapper.processKey()的过程, 记录下按下down事件的时间点。

![](D:\myBlog\weiwangqiang.github.io\img/blog_activity_anr/1.jpg)

### InputDispatcher

Inputdispatcher中，在线程里面调用到dispatchOnce方法，该方法中主要做：

- 通过dispatchOnceInnerLocked()，取出mInboundQueue 里面的 EventEntry事件
- 通过enqueueDispatchEntryLocked()，生成事件DispatchEntry并加入connection的`outbound`队列。
- 通过startDispatchCycleLocked()，从outboundQueue中取出事件DispatchEntry, 重新放入connection的`waitQueue`队列。
- 通过runCommandsLockedInterruptable()，遍历mCommandQueue队列，依次处理所有命令。
- 通过processAnrsLocked()，判断是否需要触发ANR。
- 在startDispatchCycleLocked()里面，通过inputPublisher.publishKeyEvent() 方法将按键事件分发给java层。publishKeyEvent的实现是在[InputTransport.cpp](http://aospxref.com/android-12.0.0_r3/xref/frameworks/native/libs/input/InputTransport.cpp) 中

通过上面的分析，可以知道按键事件主要存储在3个queue中：

1. InputDispatcher的mInboundQueue：存储的是从InputReader 送来的输入事件。
2. Connection的outboundQueue：该队列是存储即将要发送给应用的输入事件。
3. Connection的waitQueue：队列存储的是已经发给应用的事件，但是应用还未处理完成的。

![](D:\myBlog\weiwangqiang.github.io\img/blog_activity_anr/3.png)



## InputDispatcher解析 

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

### dispatchOnceInnerLocked

dispatchOnceInnerLocked主要是：

1）从mInboundQueue 中取出mPendingEvent

2）通过mPendingEvent的type决定事件类型和分发方式。比如当前是key类型。

3）最后如果处理了事件，就处理相关的回收。

主要代码如下：

```java
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

dispatchEventLocked 主要是遍历inputTargets，通过prepareDispatchCycleLocked分发事件。prepareDispatchCycleLockedne内部又会调用enqueueDispatchEntriesLocked方法

```java

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

主要做两个事情：1）将请求模式的调度条目排队。2）启动调度周期锁定。

```java

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

### runCommandsLockedInterruptable

dispatchOnceInnerLocked已经分析完，我们再次回到dispatchOnce，分析runCommandsLockedInterruptable方法

```java
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

ANR回调命令便是在这个时机执行。

```java
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

### processAnrsLocked

该方法是用于检查队列中是否有太旧的事件，如果存在就触发ANR

```java
 // 检查是否有任何连接的等待队列具有太旧的事件。如果我们等待事件被确认的时间超过窗口超时，
 // 请引发 ANR。返回我们下次应该醒来的时间。
nsecs_t InputDispatcher::processAnrsLocked() {
    const nsecs_t currentTime = now();
    nsecs_t nextAnrCheck = LONG_LONG_MAX; // 下一次anr超时的时间
    // 检查我们是否正在等待一个聚焦窗口出现。如果等待时间过长就报 ANR
    if (mNoFocusedWindowTimeoutTime.has_value() && mAwaitedFocusedApplication != nullptr) {
        if (currentTime >= *mNoFocusedWindowTimeoutTime) {
            // 已经超时
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

onAnrLocked 有两种实现：1）能找到当前focus的window，2）找不到当前focus的window，但是可以找到当前前台应用。

```java
// 能找到window的情况
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

// 找不到focus的window，但是找到当前获取input事件的应用。
void InputDispatcher::onAnrLocked(std::shared_ptr<InputApplicationHandle> application) {
    std::string reason =
            StringPrintf("%s does not have a focused window", application->getName().c_str());
    updateLastAnrStateLocked(*application, reason);

    auto command = [this, application = std::move(application)]() REQUIRES(mLock) {
        scoped_unlock unlock(mLock);
        mPolicy->notifyNoFocusedWindowAnr(application);
    };
    postCommandLocked(std::move(command));
}
```
### updateLastAnrStateLocked

该方法主要是收集anr的window、reason信息，接着调用dumpDispatchStateLocked 保存anr信息


```java
void InputDispatcher::updateLastAnrStateLocked(const std::string& windowLabel,
                                               const std::string& reason) {
    // 捕获 ANR 时的输入调度程序状态记录。
    time_t t = time(nullptr);
    struct tm tm;
    localtime_r(&t, &tm);
    char timestr[64];
    strftime(timestr, sizeof(timestr), "%F %T", &tm);
    mLastAnrState.clear();
    mLastAnrState += INDENT "ANR:\n";
    mLastAnrState += StringPrintf(INDENT2 "Time: %s\n", timestr);
    mLastAnrState += StringPrintf(INDENT2 "Reason: %s\n", reason.c_str());
    mLastAnrState += StringPrintf(INDENT2 "Window: %s\n", windowLabel.c_str());
    dumpDispatchStateLocked(mLastAnrState);
}
```

### dumpDispatchStateLocked





参考文献：

- [Input系统—ANR原理分析](http://gityuan.com/2017/01/01/input-anr/)
- [Android input anr 分析](https://zhuanlan.zhihu.com/p/53331495)


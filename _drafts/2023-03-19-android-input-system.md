---
layout:     post
title:      "Android Input系统事件分发分析"
subtitle:   " \" Android是怎么分发触摸事件的？\""
date:       2023-01-02 10:00:00
author:     "Weiwq"
header-img: "img/background/home-bg-o.jpg"
catalog:  true
isTop:  true
tags:
    - Android

---

> “本文基于Android13源码，分析Input系统中，事件分发的实现原理“



# 前言

在文章之前，有必要提一下InputReader。其在启动的时候，会创建一个InputReader线程，用于从/dev/input节点获取事件，转换成EventEntry事件加入到InputDispatcher的mInboundQueue。详情见 [Input系统—InputReader线程](http://gityuan.com/2016/12/11/input-reader/)

Inputdispatcher 则负责消费mInboundQueue中的事件，并将事件转化后发送给app端，他们的关系如下：

![](D:\myBlog/weiwangqiang.github.io\img/blog_activity_anr/3.png)

# InputDispatcher

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

# 事件分发

InputDispatcher 在start() 方法中会启动一个线程，在被唤醒后会调用到dispatchOnce 方法，在该方法中，通过调用dispatchOnceInnerLocked来分发事件

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
        ....
    }
    // 等待回调、超时或唤醒。
    nsecs_t currentTime = now();
    int timeoutMillis = toMillisecondTimeoutDelay(currentTime, nextWakeupTime);
    mLooper->pollOnce(timeoutMillis);
}
```

## dispatchOnceInnerLocked

dispatchOnceInnerLocked主要是：

- 从mInboundQueue 中取出mPendingEvent

- 通过mPendingEvent的type决定事件类型和分发方式。

- 最后如果处理了事件，就处理相关的回收。

主要代码如下：

```java
> frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp
  
void InputDispatcher::dispatchOnceInnerLocked(nsecs_t* nextWakeupTime) {
    nsecs_t currentTime = now();
    ...
    // 优化应用切换的延迟。本质上，当按下应用程序切换键（HOME）时，我们会开始一个短暂的超时。
    // 当它过期时，我们会抢占调度并删除所有其他挂起的事件。
    bool isAppSwitchDue = mAppSwitchDueTime <= currentTime; // mAppSwitchDueTime是应用切换到期时间
    if (mAppSwitchDueTime < *nextWakeupTime) {
        *nextWakeupTime = mAppSwitchDueTime; // 更新下一次唤醒的时间
    }
    // 当前没有PendingEvent（即EventEntry），则取一个
    if (!mPendingEvent) {
        // 1、mInboundQueue 为空
        if (mInboundQueue.empty()) {
            // 如果合适就合成一个重复按键。
            if (mKeyRepeatState.lastKeyEntry) {
                if (currentTime >= mKeyRepeatState.nextRepeatTime) {
                    // 用于创建一个新的repeat按键
                    mPendingEvent = synthesizeKeyRepeatLocked(currentTime);
                } else {
                    if (mKeyRepeatState.nextRepeatTime < *nextWakeupTime) {
                        *nextWakeupTime = mKeyRepeatState.nextRepeatTime;
                    }
                }
            }
            // 如果没有PendingEvent，就直接返回
            if (!mPendingEvent) {
                return;
            }
        } else {
        // 2、mInboundQueue不为空，就从队列前面取一个PendingEvent
            mPendingEvent = mInboundQueue.front();
            mInboundQueue.pop_front();
            traceInboundQueueLengthLocked();
        }
        if (mPendingEvent->policyFlags & POLICY_FLAG_PASS_TO_USER) {
            // 根据当前的event 类型，post 一个 command 到 mCommandQueue
            // 最后是调用到Java层的PowerManagerService#userActivityFromNative()
            pokeUserActivityLocked(*mPendingEvent);
        }
    }
    // 现在我们有一个事件要发送。所有事件最终都会以这种方式取消排队和处理，即使我们打算删除它们。
    ...
    bool done = false;
    DropReason dropReason = DropReason::NOT_DROPPED;
    ...
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

#### dispatchKeyLocked

```java
> frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp
    
bool InputDispatcher::dispatchKeyLocked(nsecs_t currentTime, std::shared_ptr<KeyEntry> entry,
                                        DropReason* dropReason, nsecs_t* nextWakeupTime) {
  //预处理：主要处理按键重复问题
  if (!entry->dispatchInProgress) {
       ...
          if (mKeyRepeatState.lastKeyEntry &&
                mKeyRepeatState.lastKeyEntry->keyCode == entry->keyCode &&
                // 我们已经看到连续两个相同的按键，这表明设备驱动程序正在自动生成键重复。
                // 我们在这里记下重复，但我们禁用了自己的下一个键重复计时器，因为很明显我们不需要自己合成键重复。
                mKeyRepeatState.lastKeyEntry->deviceId == entry->deviceId) {
                // 确保我们不会从其他设备获取密钥。如果按下了相同的设备 ID，则新的设备 ID 将替换当前设备 ID 以按住重复键并重置重复计数。
                // 将来，当设备ID上出现KEY_UP时，请将其删除，并且不要停止当前设备上的密钥重复。
                entry->repeatCount = mKeyRepeatState.lastKeyEntry->repeatCount + 1;
                resetKeyRepeatLocked();
        mKeyRepeatState.nextRepeatTime = LONG_LONG_MAX; // 不要自己生成重复
            } else {
                //不是重复。保存按键down状态，以防我们稍后遇到重复。
                resetKeyRepeatLocked();
                mKeyRepeatState.nextRepeatTime = entry->eventTime + mConfig.keyRepeatTimeout;
            }
            mKeyRepeatState.lastKeyEntry = entry;
       ...
    }
    // 处理policy 上次要求我们重试的情况
    if (entry->interceptKeyResult == KeyEntry::INTERCEPT_KEY_RESULT_TRY_AGAIN_LATER) {
         // 当前时间 < 唤醒时间，则进入等待状态
        if (currentTime < entry->interceptKeyWakeupTime) {
            if (entry->interceptKeyWakeupTime < *nextWakeupTime) {
                *nextWakeupTime = entry->interceptKeyWakeupTime;
            }
            return false; // 等到下次醒来
        }
        entry->interceptKeyResult = KeyEntry::INTERCEPT_KEY_RESULT_UNKNOWN;
        entry->interceptKeyWakeupTime = 0;
    }
    // 给policy提供拦截key的机会。
    if (entry->interceptKeyResult == KeyEntry::INTERCEPT_KEY_RESULT_UNKNOWN) {
        if (entry->policyFlags & POLICY_FLAG_PASS_TO_USER) {
            sp<IBinder> focusedWindowToken =
                    mFocusResolver.getFocusedWindowToken(getTargetDisplayId(*entry));

            auto command = [this, focusedWindowToken, entry]() REQUIRES(mLock) {
                doInterceptKeyBeforeDispatchingCommand(focusedWindowToken, *entry);
            };
            postCommandLocked(std::move(command));
            return false; // wait for the command to run
        } else {
            entry->interceptKeyResult = KeyEntry::INTERCEPT_KEY_RESULT_CONTINUE;
        }
    } else if (entry->interceptKeyResult == KeyEntry::INTERCEPT_KEY_RESULT_SKIP) {
        if (*dropReason == DropReason::NOT_DROPPED) {
            *dropReason = DropReason::POLICY;
        }
    }
    // 如果删除事件，则清理。
    if (*dropReason != DropReason::NOT_DROPPED) {
        setInjectionResult(*entry,
                           *dropReason == DropReason::POLICY ? InputEventInjectionResult::SUCCEEDED
                                                             : InputEventInjectionResult::FAILED);
        mReporter->reportDroppedKey(entry->id);
        return true;
    }
    // 寻找输入目标
    std::vector<InputTarget> inputTargets;
    InputEventInjectionResult injectionResult =
            findFocusedWindowTargetsLocked(currentTime, *entry, inputTargets, nextWakeupTime);
    if (injectionResult == InputEventInjectionResult::PENDING) {
        return false;
    }
    setInjectionResult(*entry, injectionResult);
    if (injectionResult != InputEventInjectionResult::SUCCEEDED) {
        return true;
    }
    // 从事件或焦点显示添加监视器通道。
    addGlobalMonitoringTargetsLocked(inputTargets, getTargetDisplayId(*entry));
    // 分发按键
    dispatchEventLocked(currentTime, entry, inputTargets);
    return true;
}
```

#### dispatchMotionLocked

```java
> frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp
    
bool InputDispatcher::dispatchMotionLocked(nsecs_t currentTime, std::shared_ptr<MotionEntry> entry,DropReason* dropReason, nsecs_t* nextWakeupTime) {
    ...
    // 是否是指针事件
    const bool isPointerEvent = isFromSource(entry->source, AINPUT_SOURCE_CLASS_POINTER);
    // Identify targets.
    std::vector<InputTarget> inputTargets;
    bool conflictingPointerActions = false;
    InputEventInjectionResult injectionResult;
    if (isPointerEvent) {  // 指针事件。（例如触摸屏）
        // 查找已锁定的触摸窗口目标，主要根据读取到的触控事件所属的屏幕displayid、x、y坐标位置等属性来确认目标窗口
        // 这里不是我们的重点
        injectionResult =
                findTouchedWindowTargetsLocked(currentTime, *entry, inputTargets, nextWakeupTime, &conflictingPointerActions);
    } else {  // 非触摸事件。（例如轨迹球）
        injectionResult =
                findFocusedWindowTargetsLocked(currentTime, *entry, inputTargets, nextWakeupTime);
    }
    if (injectionResult == InputEventInjectionResult::PENDING) {
        // 如果是pending就直接返回
        return false;
    }

    setInjectionResult(*entry, injectionResult);
    if (injectionResult == InputEventInjectionResult::TARGET_MISMATCH) {
        return true;
    }
    if (injectionResult != InputEventInjectionResult::SUCCEEDED) {
        CancelationOptions::Mode mode(isPointerEvent
                                              ? CancelationOptions::CANCEL_POINTER_EVENTS
                                              : CancelationOptions::CANCEL_NON_POINTER_EVENTS);
        CancelationOptions options(mode, "input event injection failed");
        synthesizeCancelationEventsForMonitorsLocked(options);
        return true;
    }

    // Add monitor channels from event's or focused display.
    addGlobalMonitoringTargetsLocked(inputTargets, getTargetDisplayId(*entry));

    // Dispatch the motion.
    if (conflictingPointerActions) {
        CancelationOptions options(CancelationOptions::CANCEL_POINTER_EVENTS,
                                   "conflicting pointer actions");
        synthesizeCancelationEventsForAllConnectionsLocked(options);
    }
    dispatchEventLocked(currentTime, entry, inputTargets);
    return true;
}

```



 ## dispatchEventLocked

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

## enqueueDispatchEntriesLocked

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

## enqueueDispatchEntryLocked

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

## startDispatchCycleLocked

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

## publishMotionEvent

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

mChannel即 InputChannel

```java
> frameworks/native/include/input/InputTransport.h

private:
    std::shared_ptr<InputChannel> mChannel;
```

## InputChannel

InputChannel的sendMessage 方法定义如下，

```java
> frameworks/native/libs/input/InputTransport.cpp
  
status_t InputChannel::sendMessage(const InputMessage* msg) {
    ....
    ssize_t nWrite;
    do {
        nWrite = ::send(getFd(), &cleanMsg, msgLength, MSG_DONTWAIT | MSG_NOSIGNAL);
    } while (nWrite == -1 && errno == EINTR);
    ....
    return OK;
}
```

## runCommandsLockedInterruptable

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



## 调用栈

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

# 事件接收



# 事件确认

## InputEventReceiver

在处理完成事件后，会调用ViewRootImpl的 finishInputEvent 方法

```java
> frameworks/base/core/java/android/view/ViewRootImpl.java
    private void finishInputEvent(QueuedInputEvent q) {
    		....
        if (q.mReceiver != null) {
             ....
                q.mReceiver.finishInputEvent(q.mEvent, handled);
             ....
        }
    }
```

接着调用InputEventReceiver的finishInputEvent

```java
> frameworks/base/core/java/android/view/InputEventReceiver.java
  
    public final void finishInputEvent(InputEvent event, boolean handled) {
   				nativeFinishInputEvent(mReceiverPtr, seq, handled);
         ....
    }
```

该方法通过nativeFinishInputEvent 调用到native层

##  android_view_InputEventReceiver

来到native层，nativeFinishInputEvent 实现如下

```java
> frameworks/native/core/jni/android_view_InputEventReceiver.cpp
  
static void nativeFinishInputEvent(JNIEnv* env, jclass clazz, jlong receiverPtr,
        jint seq, jboolean handled) {
    sp<NativeInputEventReceiver> receiver =
            reinterpret_cast<NativeInputEventReceiver*>(receiverPtr);
    status_t status = receiver->finishInputEvent(seq, handled);
    .....
}

```

调用到NativeInputEventReceiver::finishInputEvent 方法

```java
> frameworks/native/core/jni/android_view_InputEventReceiver.cpp

status_t NativeInputEventReceiver::finishInputEvent(uint32_t seq, bool handled) {
    ....
    return processOutboundEvents();
}

status_t NativeInputEventReceiver::processOutboundEvents() {
    while (!mOutboundQueue.empty()) {
        OutboundEvent& outbound = *mOutboundQueue.begin();
        status_t status;
        if (std::holds_alternative<Finish>(outbound)) {
            const Finish& finish = std::get<Finish>(outbound);
            // mInputConsumer是InputConsumer的实例
            status = mInputConsumer.sendFinishedSignal(finish.seq, finish.handled);
        } else if (std::holds_alternative<Timeline>(outbound)) {
            const Timeline& timeline = std::get<Timeline>(outbound);
            status = mInputConsumer.sendTimeline(timeline.inputEventId, timeline.timeline);
        } 
        ....
        return status;
    }
    return OK;
}
```

mInputConsumer.sendFinishedSignal() 实现是在InputTransport.cpp 中

## InputTransport

sendFinishedSignal 实现如下

```java
> frameworks/native/libs/input/InputTransport.cpp
  
status_t InputConsumer::sendFinishedSignal(uint32_t seq, bool handled) {
    if (seqChainCount) {
        ....
        while (!status && chainIndex > 0) {
            chainIndex--;
            status = sendUnchainedFinishedSignal(chainSeqs[chainIndex], handled);
        }
        ....
    }
    // 为批次中的最后一条消息发送完成信号。
    return sendUnchainedFinishedSignal(seq, handled);
}
```

主要在sendUnchainedFinishedSignal中，通过InputChannel 发送完成信号

```java
> frameworks/native/libs/input/InputTransport.cpp
  
status_t InputConsumer::sendUnchainedFinishedSignal(uint32_t seq, bool handled) {
    InputMessage msg;
    msg.header.type = InputMessage::Type::FINISHED;
    msg.header.seq = seq;
    msg.body.finished.handled = handled;
    msg.body.finished.consumeTime = getConsumeTime(seq);
    status_t result = mChannel->sendMessage(&msg);
    return result;
}
```

那这个message是在哪里接收到呢？

## InputManagerService

createInputChannel 的实现如下

```java
>  frameworks/base/services/core/java/com/android/server/input/InputManagerService.java

   // 创建一个输入通道以用作输入事件目标。
    public InputChannel createInputChannel(String name) {
        // mNative是NativeInputManagerService 接口的实例，NativeImpl 实现了该接口，所以实现在NativeImpl 中
        return mNative.createInputChannel(name);
    }

```

createInputChannel 是一个native方法

```java
>  frameworks/base/services/core/java/com/android/server/input/NativeInputManagerService.java
    class NativeImpl implements NativeInputManagerService {
       ....
       public native InputChannel createInputChannel(String name);  
    }
```

createInputChannel() 在 native层对应的方法是nativeCreateInputChannel

```java
>  frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp

// Java层的createInputChannel 方法映射到native的nativeCreateInputChannel 方法
static const JNINativeMethod gInputManagerMethods[] = {
		   ...
        {"createInputChannel", "(Ljava/lang/String;)Landroid/view/InputChannel;",
         (void*)nativeCreateInputChannel},  
}


int register_android_server_InputManager(JNIEnv* env) {
    // 注册native层的方法，与Java层的NativeInputManagerService$NativeImpl绑定
    int res = jniRegisterNativeMethods(env,
                                       "com/android/server/input/"
                                       "NativeInputManagerService$NativeImpl",
                                       gInputManagerMethods, NELEM(gInputManagerMethods));
}

static jobject nativeCreateInputChannel(JNIEnv* env, jobject nativeImplObj, jstring nameObj) {
    NativeInputManager* im = getNativeInputManager(env, nativeImplObj);
    ....
    base::Result<std::unique_ptr<InputChannel>> inputChannel = im->createInputChannel(name);
    ...
    jobject inputChannelObj =
            android_view_InputChannel_createJavaObject(env, std::move(*inputChannel));
    android_view_InputChannel_setDisposeCallback(env, inputChannelObj,
            handleInputChannelDisposed, im);
    return inputChannelObj;
}

// NativeInputManager的createInputChannel 方法定义
base::Result<std::unique_ptr<InputChannel>> NativeInputManager::createInputChannel(
        const std::string& name) {
    // mInputManager 是 InputManager 的实例
    return mInputManager->getDispatcher().createInputChannel(name);
}
```

getDispatcher() 方法返回InputDispatcher实例

## InputDispatcher

createInputChannel实现如下

```java
> frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp
  
Result<std::unique_ptr<InputChannel>> InputDispatcher::createInputChannel(const std::string& name) {
    std::unique_ptr<InputChannel> serverChannel;
    std::unique_ptr<InputChannel> clientChannel;
    // 将serverChannel（接收端） 与 clientChannel（发送端） 绑定
    status_t result = InputChannel::openInputChannelPair(name, serverChannel, clientChannel);
    { // acquire lock
        std::scoped_lock _l(mLock);
        const sp<IBinder>& token = serverChannel->getConnectionToken();
        int fd = serverChannel->getFd();
        sp<Connection> connection =
                new Connection(std::move(serverChannel), false /*monitor*/, mIdGenerator);
      
        mConnectionsByToken.emplace(token, connection);
        // 注册回调 handleReceiveCallback 方法
        std::function<int(int events)> callback = std::bind(&InputDispatcher::handleReceiveCallback,
                                                            this, std::placeholders::_1, token);
        // 将callback 与 FD（即serverChannel） 关联起来
        mLooper->addFd(fd, 0, ALOOPER_EVENT_INPUT, new LooperEventCallback(callback), nullptr);
    } // release lock
    mLooper->wake();
    return clientChannel;
}
```



**所以应用层发送的finish事件最终来到了InputDispatcher::handleReceiveCallback 方法**

handleReceiveCallback实现如下

```java
> frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp

int InputDispatcher::handleReceiveCallback(int events, sp<IBinder> connectionToken) {
    sp<Connection> connection = getConnectionLocked(connectionToken);
    bool notify;
    if (!(events & (ALOOPER_EVENT_ERROR | ALOOPER_EVENT_HANGUP))) {
        nsecs_t currentTime = now();
        bool gotOne = false;
        status_t status = OK;
        for (;;) {
            Result<InputPublisher::ConsumerResponse> result =
                    connection->inputPublisher.receiveConsumerResponse();
            if (!result.ok()) {
                status = result.error().code();
                break;
            }
            // 如果分发结束
            if (std::holds_alternative<InputPublisher::Finished>(*result)) {
                const InputPublisher::Finished& finish =
                        std::get<InputPublisher::Finished>(*result);
                // 执行事件分发结束的回收处理
                finishDispatchCycleLocked(currentTime, connection, finish.seq, finish.handled,
                                          finish.consumeTime);
            } else if (std::holds_alternative<InputPublisher::Timeline>(*result)) {
                if (shouldReportMetricsForConnection(*connection)) {
                    const InputPublisher::Timeline& timeline =
                            std::get<InputPublisher::Timeline>(*result);
                    mLatencyTracker
                            .trackGraphicsLatency(timeline.inputEventId,
                                                  connection->inputChannel->getConnectionToken(),
                                                  std::move(timeline.graphicsTimeline));
                }
            }
            gotOne = true;
        }
    }
    ....
    return 0; // remove the callback
}
```

finishDispatchCycleLocked实现如下

```java
> frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp
  
void InputDispatcher::finishDispatchCycleLocked(nsecs_t currentTime,
                                                const sp<Connection>& connection, uint32_t seq,
                                                bool handled, nsecs_t consumeTime) {
    // 通知其他系统组件并准备开始下一个调度周期。
    auto command = [this, currentTime, connection, seq, handled, consumeTime]() REQUIRES(mLock) {
        doDispatchCycleFinishedCommand(currentTime, connection, seq, handled, consumeTime);
    };
    postCommandLocked(std::move(command));
}
```

doDispatchCycleFinishedCommand 该方法主要处理执行完成后的回收动作，包括将对应的事件从waitQueue 队列中移除。

```java
> frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp
  
void InputDispatcher::doDispatchCycleFinishedCommand(... uint32_t seq ..) {
    std::deque<DispatchEntry*>::iterator dispatchEntryIt = connection->findWaitQueueEntry(seq);
    // 将事件出列并开始下一个循环。因为锁可能已经被释放，等待队列的内容可能已经被清空，所以我们需要仔细检查一些事情。
    if (dispatchEntryIt != connection->waitQueue.end()) {
        dispatchEntry = *dispatchEntryIt;
        // 将对应的事件从waitQueue移除
        connection->waitQueue.erase(dispatchEntryIt);
        const sp<IBinder>& connectionToken = connection->inputChannel->getConnectionToken();
        // 将事件对应的anr超时移除 
        mAnrTracker.erase(dispatchEntry->timeoutTime, connectionToken);
        // 如果连接没有响应
        if (!connection->responsive) {
            connection->responsive = isConnectionResponsive(*connection);
            if (connection->responsive) {
                // 连接之前没有响应，现在有响应了。
                processConnectionResponsiveLocked(*connection);
            }
        }
        ....
    }
}
```

## 调用栈

```java
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::doDispatchCycleFinishedCommand()
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::finishDispatchCycleLocked()
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::handleReceiveCallback()
services/inputflinger/dispatcher/InputDispatcher.cpp : InputDispatcher::createInputMonitor()
```


# 参考文献：

- [Input系统—事件处理全过程](http://gityuan.com/2016/12/31/input-ipc/)


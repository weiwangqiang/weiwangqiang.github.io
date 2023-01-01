---
layout:     post
title:      "Android 输入系统分析"
subtitle:   " \" Android是怎么分发触摸事件的？\""
date:       2022-12-30 10:00:00
author:     "Weiwq"
header-img: "img/background/home-bg-o.jpg"
catalog:  true
isTop:  true
tags:
    - Android
---

> “本文基于Android13源码，分析Input系统实现原理“

# InputReader

Inputreader主要的作用是：

- 读取节点/dev/input，将Input_event 结构体转成相应的EventEntry，比如按键事件对应KeyEntry，触摸事件对应MotionEntry
- 将事件添加到mInboundQueue队列尾部。
- KeyboardInputMapper.processKey()的过程, 记录下按下down事件的时间点。

![](D:\myBlog\weiwangqiang.github.io\img/blog_activity_anr/1.jpg)

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

![](D:\myBlog\weiwangqiang.github.io\img/blog_activity_anr/3.png)

## dispatchOnce

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

# 事件分发

dispatchOnce 中，通过调用dispatchOnceInnerLocked来分发事件

## dispatchOnceInnerLocked

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


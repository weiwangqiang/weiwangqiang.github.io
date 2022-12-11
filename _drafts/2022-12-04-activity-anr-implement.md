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



## 代码分析 

dispatchOnceInnerLocked：从mInboundQueue 中取出mPendingEvent，然后通过mPendingEvent的type决定事件类型和分发方式。比如当前是key类型，

最后如果处理了事件，就处理相关的回收

参考文献：

- [Input系统—ANR原理分析](http://gityuan.com/2017/01/01/input-anr/)
- [Android input anr 分析](https://zhuanlan.zhihu.com/p/53331495)


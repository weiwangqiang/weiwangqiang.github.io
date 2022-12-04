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

> “本文基于Android12源码，分析Input系统的anr实现原理“
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

![](img/blog_activity_anr/1.jpg)







## 代码分析

如果小伙伴不想看代码，可以直接跳到 **Input 系统流程**

参考文献：

- [Input系统—ANR原理分析](http://gityuan.com/2017/01/01/input-anr/)
- 


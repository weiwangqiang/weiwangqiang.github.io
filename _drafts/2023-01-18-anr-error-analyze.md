---
layout:     post
title:      "ANR 日志分析"
subtitle:   " \" 教你如何分析ANR问题 \""
date:       2023-01-18 13:30:00
author:     "Weiwq"
header-img: "img/background/home-bg-o.jpg"
catalog:  true
top: false
tags:
    - Android

---

> “分析ANR问题“
## ANR类型

### Input timeout

输入事件超时，又分两种情况：

1. 有window：

2. 没有window：Waiting because no window has focus but there is a focused application

input事件的ANR原理可以看之前的文章 [带你细看Android input系统中ANR的机制](https://weiwangqiang.github.io/2022/12/31/activity-anr-implement/)

###  executing service

在执行service的时候出现anr，日志如下，后面是service的class name。

```java

12-18 04:56:03.483 E/ActivityManager( 1968): Reason: executing service com.google.android.gms/.chimera.GmsBoundBrokerService

```

## 系统影响

### 系统负载影响

在日志中搜索`ANR in` 

```java
12-18 05:02:37.526 E/ActivityManager( 1968): ANR in com.android.vending
12-18 05:02:37.526 E/ActivityManager( 1968): PID: 28407
12-18 05:02:37.526 E/ActivityManager( 1968): Reason: executing service com.android.vending/com.google.android.finsky.scheduler.JobSchedulerEngine$PhoneskyJobSchedulerJobService
12-18 05:02:37.526 E/ActivityManager( 1968): Load: 0.0 / 0.0 / 0.0
12-18 05:02:37.526 E/ActivityManager( 1968): CPU usage from 543ms to 5963ms later (2022-12-18 05:02:31.793 to 2022-12-18 05:02:37.213):
12-18 05:02:37.526 E/ActivityManager( 1968):   61% 27772/com.google.android.youtube.tv: 35% user + 25% kernel / faults: 11530 minor 565 major
12-18 05:02:37.526 E/ActivityManager( 1968):   47% 554/kswapd0: 0% user + 47% kernel
12-18 05:02:37.526 E/ActivityManager( 1968):   48% 1968/system_server: 2% user + 46% kernel / faults: 7487 minor 724 major
  ....
12-18 05:02:37.526 E/ActivityManager( 1968): 100% TOTAL: 22% user + 75% kernel + 0% iowait + 2.3% softirq
```

-  ANR in com.android.vending：com.android.vending 即为进程名。
- Reason: 本次ANR的原因，当前是由于在运行service的时候anr。
- 2022-12-18 05:02:31.793 to 2022-12-18 05:02:37.213：CPU统计的时间区域
- 100% TOTAL: 22% user + 75% kernel + 0% iowait + 2.3% softirq：该信息提供当前系统的负载情况。100% TOTAL 说明CPU处于满载状态。其中75% kernel表示内核的CPU占用达到75%。

### 受低内存影响

还是上面的案例，kswapd的CPU占用率达到47%，一般情况，kswapd 在top3 ，很有可能是系统低内存导致的。

![](/Users/file/blog/weiwangqiang.github.io/img/blog_anr_erro_analyze/3.png)

整机一旦陷入低内存，响应度和流畅度都会受到影响，这是因为低内存往往会伴随着IO升高，内存回收线程如kswapd、HeapTaskDaemon会变得活跃。

## App自身问题

如果排除系统因素，那就需要排查一下app端的堆栈。

### 获取堆栈

在event日志中搜索 **am_anr** 关键字

![](/Users/file/blog/weiwangqiang.github.io/img/blog_anr_erro_analyze/1.png)

可以看到如下信息；

- 时间： 12-18 05:02:19.781 
-  进程号: 28427
- 包：com.google.android.gms
- ANR 类：executing service com.google.android.gms/.chimera.GmsBoundBrokerService

然后我们再到data/anr文件夹中找到对应时间的anr文件

![](/Users/file/blog/weiwangqiang.github.io/img/blog_anr_erro_analyze/2.png)

通过文件头部 **Cmd line:**  **sysTid=**信息确认堆栈是否正确，sysTid即 进程ID（也是主线程ID）

```java
----- pid 28427 at 2022-12-18 05:02:20 -----
Cmd line: com.google.android.gms
....
  
"main" prio=5 tid=1 Runnable
  | group="main" sCount=0 dsCount=0 flags=0 obj=0x73aa6ef0 self=0xa5838000
  | sysTid=28427  ....
```

### 分析堆栈

ANR的Trace文件一般包含如下实现

```jva
"main" prio=5 tid=1 Runnable
  | group="main" sCount=0 dsCount=0 flags=0 obj=0x73aa6ef0 self=0xa5838000
  | sysTid=28427 nice=0 cgrp=default sched=0/0 handle=0xa9acb494
  | state=R schedstat=( 5188234068 11420608867 15443 ) utm=42 stm=476 core=2 HZ=100
  | stack=0xbd5e2000-0xbd5e4000 stackSize=8MB
  | held mutexes= "mutator lock"(shared held)
  at android.os.Handler.sendMessageAtTime(Handler.java:690)
  at adrw.sendMessageAtTime(:com.google.android.gms@16089033@16.0.89 (100300-239467275):-1)
  at android.os.Handler.sendMessageDelayed(Handler.java:667)
  at android.os.Handler.sendMessage(Handler.java:604)
  ....
  at android.os.Looper.loop(Looper.java:193)
  at android.app.ActivityThread.main(ActivityThread.java:6670)
  at java.lang.reflect.Method.invoke(Native method)
  at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:493)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:858)
```

- Runnable：表示当前运行状态
- nice=0：表示该线程优先级，nice的取值范围为-20～19。nice值越大，进程优先级越低；nice值越大，进程优先级越高。
- utm=42：该线程在用户态所执行的时间(单位是jiffies）
- stm=476：该线程在内核态所执行的时间。线程的cpu耗时是两者相加(utm+stm)，utm,stm 单位换算成时间单位为 1 比 10ms。
- core=2：表示跑在哪个核上。

通过trace可以了解ANR时候的调用情况，再结合代码分析。

## 经典案例

### Input timeOut

比如下面的浏览器ANR异常，ANR日志如下：

```java
10-14 06:35:39.448 E/ActivityManager( 1947): Reason: Input dispatching timed out (Waiting because no window has focus but there is a focused application that may eventually add a window when it finishes starting up.)
```

kswapd 的进程在top4 ，CPU占用率不算很高，暂时排除系统原因。

![](/Users/file/blog/weiwangqiang.github.io/img/blog_anr_erro_analyze/4.png)

主要分析对应的trace文件，排查对应的方法即可。

![](/Users/file/blog/weiwangqiang.github.io/img/blog_anr_erro_analyze/5.png)

### 死锁


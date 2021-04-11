---
layout:     post
title:      "再谈Android启动模式"
subtitle:   " \"温故知新，重新认识Android的启动模式\""
date:       2021-04-11 23:30:00
author:     "Weiwq"
header-img: "img/background/post-sample-image.jpg"
catalog: true
tags:
    - Android

---

> “在用整理完启动模式后，我发现之前的理解是有误区的“

笔者会用adb的打印，看每一种启动模式下，任务栈的变化。

# 引言

再谈启动模式，貌似没啥意思。但是你能正确回答下面的问题吗？

- 问题1：singleTask启动模式，在启动新的Activity的时候，真的会重新创建新的任务栈吗？
- 问题2：设置了Intent.FLAG_ACTIVITY_NEW_TASK，每次都会创建新的任务栈吗？
- 问题3：对于singleInstance，显式指定taskAffinity为应用包名，那在启动的时候，还会创建新的任务栈吗？

## 一、ActivityRecord、TaskRecord、ActivityStack的区别 

在回答上面的问题前，我们先整理下ActivityRecord、TaskRecord、ActivityStack的区别。



——Weiwq  于 2021.04 广州
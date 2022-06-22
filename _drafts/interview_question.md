---
layout:     post
title:      "面试题"
subtitle:   " \"面试题\""
date:       2021-12-29 13:30:00
author:     "Weiwq"
header-img: "img/background/home-bg-o.jpg"
catalog:  true
top: false
tags:
    - 面试

---

> “题目“

# 字节跳动

## 一面

1.算法题
两个栈实现队列
2.算法题输入一个数组，想一种方法让这个数组尽可能的乱序，保证功能能实现的情况下时间复杂度和空间复杂度尽可能的小，可使用随机数函数。（面试官最后说了 O(n)的时间复杂度能实现）
3.写一个单例
4.ActivityA -> Activity B -> Activity A
Activity A 启动模式为 singleTask
Activity B 启动模式为常规模式
问A 启动 B，B 又启动 A 的生命周期调用顺序？
5.你刚才提到 onsaveinstancestate() ，说一下调用时机，它用来干什么的。
6.onsaveinstancestate() 保存的那个参数叫什么？Bundle 里面都放一些什么东西？怎么实现序列化？Parcelable 和 Serializable有什么区别？
Bundle 。
7.数组和链表的区别
8.HashMap 的结构以及原理
9.我看你简历上写了 retrofit，你能说一下它是做什么的，如果知道基本框架也说一下
10.了解 View 的绘制机制吗，能说一下吗
11.我看你项目里用的 Fragment 能说一下 Fragment A 启动了 Fragment B，Fragment B 中按下返回键只退出 Fragment B 怎么实现。
12.你还有什么要问的吗？

## 二面

1.算法题 一个字符串，求最长没有重复字符的字符串长度
2.string stringbuffer 和 stringbuilder 区别
3.final finally finalize区别
4.数组和链表的区别
5.HashMap 了解过吗
6.Tcp 三次握手四次挥手
7.get 与 post 的区别
8.synchronized 的作用
9.你知道哪些设计模式
10.Android 进程通信的方法
11.那你能说一下 Intent 是怎么进程通信的
12.内存泄漏有哪几种情况
13.有什么要问

# 字节社招

## 一面

1、java泛型，反射
2、进程间通信的方式，安卓中有哪些方式，为什么是基于Binder的，不用传统的操作系统进程间通信方式呢
3、一个app可以开启多个进程嘛，怎么做呢，每个进程都是在独立的虚拟机上嘛
4、异步消息处理流程，如果发送一个延时消息，messagequeue里面怎么个顺序，messagequeue是个什么数据结构
5、广播的种类，注册的方式，以及不同注册方式的生命周期。
6、局部广播和全局广播的区别分别用什么实现的。
7、activity和service的通信方式
8、进程和线程的区别
9、并发和并行分别是什么意思，多线程是并发还是并行
10、安卓11有什么新的特性。
11、HTTPS过程。
12、DNS解析过程，如果服务器ip地址改变了，客户端怎么知道呢
13、算法：二叉树的右视图。

## 二面

1、介绍一下所有的map，以及他们之间的对比，适用场景。
2、一个按钮，手抖了连续点了两次，会跳转两次页面，怎么让这种情况不发生。
3、一个商品页一个商详页，点击商详页的一个关注按钮，希望回- 到商品页也能够显示关注的状态，怎么做
4、项目中定时为什么用AlarmManager，不用postDelayed
5、项目中后台网络请求为什么用service不用线程
6、安卓的新特性。
7、内部类会有内存泄漏问题吗 内部类为什么能访问外部类的变量，为什么还能访问外部类的私有变量。
8、算法: 单链表判断有无环。

## 三面

1、介绍项目用到了contentprovider,然后问ContentProvider的生命周期，application,activity，service,contentprovider他们的 context有什么区别。
2、内存溢出和内存泄漏，提到了bitmap
3、然后问下载一个图片的时候直接下载了一个5g的图片，不压缩一定会产生OOM问题，那么怎么去获取这个图片的长宽呢，或者说这个图片的大小的大小在你没下载之前如何得到。
字节跳动
1.操作系统进程通信方式有哪几种？
2.进程间的共享内存是怎么实现的？
3.java中被static修饰的对象会被回收吗？
4.synchronized能保证可见性吗？
5.说说事件分发机制
6.说说类加载机制
7.说说双亲委派模型
8.看过哪些框架？
9.retrofit怎么实现的？
10.项目中遇到的难题以及如何解决的
11.算法：写一个函数，往一个数组中指定位置插入一个元素
12.http状态码有哪些？
13.如果自己实现AsyncTask，要怎么实现？

# OPPO

项目中的重点内容
Service两种区别
AsynTask 原理
线程池原理，是否使用过
性能优化，图片内存占用计算，持有引用，
TCP原理，如何确保稳定（与udp相比），阻塞，
文件上传下载原理，下载中流的大小；
反射如何实现
泛型
EventBus作用，，原理；
java四种引用（强软弱虚），软弱的回收区别
ListView的一些优化，如何复用，错位，现在用glide
数据结构，SparseArray和hashmap区别
操作系统，cpu调度
数据库
LRU缓存原理
死锁，锁的几种类型。是否项目中使用
继承和接口，优先使用级
四道算法原理
Linux指令；

# 快手

单例模式
volatile关键字
HashMap（红黑树的时间效率为什么是logn，怎么算出来的？）
线程、线程池
Retrofit（底层网络请求涉及到OkHttp）
Handler（原理、Looper在主线程中死循环，为啥不会ANR？、是否能在主线程更新UI、同步屏障机制等）
HTTP和HTTPS的区别
一道mid算法

# 小米：

rxjava 三大类是什么
String的对象为什么是不可变的
arraymap可能导致什么问题
thread的构造函数是什么
runnable如何创建线程
（创建线程的方式）
有哪些锁？
synchronize和lock的区别
GC root有哪些
队列如何创建、队列在Java中的类是什么
offer是什么（当时问的时候呆了一下，还以为面试官要给我offer了，搞得老激动...然后没答出来...后来才想起他问的应该是队列里的offer方法... ）
有什么查找的算法
排序的算法

# 网易：

sharepreference的commit方法和apply方法的区别

jetpack(viewmodel、lifecycle)

AndroidX有什么好处

Android帧动画会遇到的问题

属性动画和view动画的区别

子线程创建handler

Android达到ANR的条件

recyclerview使用不同的布局

# 小米

## 一面

事件分发
自定义view
给了个布局问你的实现方式
有没有了解过新的布局
Android布局优化
过度绘制及优化
讲讲你认为你Android里理解最深的点
了解过framework吗
讲讲二叉树前中后序遍历
数据库
类加载的过程
kotlin扩展方法 扩展属性
看过哪些开源库
实习过程中最有成就感的事
算法
反转链表
删除公共字符串
冒泡排序怎么排的 稳定吗

## 二面

Android
滑动时间冲突解决
handler原理
Android跨进程通信
Activity生命周期
Android为啥要分四大组件
弹一个dialog时Activity生命周期变化
onstart onresume分别执行什么类型的业务
Java
手写单例
hashmap源码
多线程，锁
操作系统
进程和线程的区别
算法
之字形打印二叉树

## 三面

Java
封装继承多态，重点说理解及应用
static
重写和重载的区别、理解及应用
hashmap底层，把面试官当小白给面试官讲
Android
四大组件的理解
activity生命周期、横竖屏生命周期、有没有不让activity销毁的方法
启动模式
两种service有啥区别
service执行耗时操作会咋样、咋解决
intentservice底层
service保活
broadcastreciver权限
Android跨进程方式
intent底层是怎么跨进程的
常用布局，重点说理解及应用
Android动画有哪几种，有没有底层研究
自定义view、自己写过的demo
内存泄漏场景及解决办法
网络
TCP三次握手/四次挥手 讲讲
有没有直接在TCP层做过操作
操作系统
进程和线程的区别

# 腾讯

## 一面

自我介绍
说一下做过的项目
两个队列实现一个栈
activity和service的区别
找出一个数组中出现次数大于数组长度一半的数
线程安全的单例模式
Android 线程切换有哪些方式
三次握手四次挥手 为什么要有三次握手(而不是两次)
说一下final关键字
讲一下listview的特点？？
http的301状态码
tcp UDP的区别
tcp如何做到可靠传输
Java gc
讲一下你对flutter的看法(简历里写了会flutter)
平时有写博客吗？可以看一下你的github主页吗？
让我问问题

## 二面

自我介绍
说项目
说一下项目中的难点
说说flutter的实现原理(绘制原理？)
说说flutter和Android在开发效率上的感受
讲一下设计模式
然后再细讲一下工厂
讲一下MVP
进程间通信
对比一下队列和栈，以及它们底部实现
对比一下C 的vector和Java的list，什么空间利用率呀，空间占用啊
还问了 有没有读研的打算
最后让我问问题，我首先问了什么时能有结果？
然后问了，如果出现了一个新技术或者新框架，团队会马上投入研究吗？
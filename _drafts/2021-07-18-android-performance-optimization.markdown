---
layout:     post
title:      "Android App性能监控、优化工具"
subtitle:   " \"可能是全网最全的优化工具大集合\""
date:       2021-07-18 17:30:00
author:     "Weiwq"
header-img: "img/background/post-sample-image.jpg"
catalog:  true
top: true
tags:
    - Android

---

> “工欲善其事，必先利其器“

## 一、LeakCanary 

LeakCanary 想必大家都有了解一些，主要用于分析activity、fragment的内存泄露的问题。

在主module下的gradle导入如下依赖即可

```java
dependencies {
  // debugImplementation because LeakCanary should only run in debug builds.
  debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.7'
}
```

在安装测试app后，点击leak的图标进入leak应用，点击`Dump Heap Now`即可，内存泄露引用链如下

<center class="half">
    <img src="D:\myBlog\weiwangqiang.github.io/img/blog_android_performance/10.png" width="30%" height="40%"/><img src="D:\myBlog\weiwangqiang.github.io/img/blog_android_performance/9.png" width="30%" height="40%"/>
</center>

## 二、Perfdog

Perfdog是由腾讯出品的移动平台性能分析工具，官网[点我前往](https://perfdog.qq.com/)

工具首页如下，默认的功能有：FPS、CPU、memory三个维度的性能。

- 右下角可以扩展更多功能
- 点击右上角可以开始记录数据，再点一下可以保存到云平台。

<img src="D:\myBlog\weiwangqiang.github.io/img/blog_android_performance/1.png" width="100%" height="40%">

在官网登录后就可以看到对应的详细数据

<img src="D:\myBlog\weiwangqiang.github.io/img/blog_android_performance/2.png" width="100%" height="40%">

## 三、profiler

Perfdog 只适合用于监控CPU，内存、FPS等情况，如果想具体排查问题，还是得用Android studio自带的Profiler

profiler支持CPU、memory、network、energy维度的分析。

<img src="D:\myBlog\weiwangqiang.github.io/img/blog_android_performance/3.png" width="100%" height="40%">

### 1、CPU

在CPU下，点击record可以开始记录一段时间内的方法耗时情况

<img src="D:\myBlog\weiwangqiang.github.io/img/blog_android_performance/4.png" width="100%" height="40%">

点击stop后，就可以看具体的执行耗时情况，比如在main线程中，clickView的方法耗时长达270ms，就可以结合代码做具体的耗时分析。

<img src="D:\myBlog\weiwangqiang.github.io/img/blog_android_performance/5.png" width="100%" height="40%">

### 2、Memory

在Memory下，可以看到每个块所占用的内存大小，如果想具体看内存分配情况，可以点击顶部的“Allocation Tracking”

<img src="D:\myBlog\weiwangqiang.github.io/img/blog_android_performance/6.png" width="100%" height="40%">

在点击stop后，就会进入如下页面

- 区域1：可以按照不同的归类来查看内存情况
- 区域2：每个类实例对应的内存分配情况，单位是byte。点击对应的分类，可以按照该分类的内存情况升序或者降序排列。
- 区域3：该实例对应的成员变量和引用链，对于分析内存泄露很有帮助
- 区域4：该类的实例列表，正常列表只有一个，如果有多个，有可能发生了内存泄露。

<img src="D:\myBlog\weiwangqiang.github.io/img/blog_android_performance/7.png" width="100%" height="40%">

### 3、network

可以测试网络的收发速度

<img src="D:\myBlog\weiwangqiang.github.io/img/blog_android_performance/8.png" width="100%" height="40%">

## 四、Memory Analyzer (MAT)

## 五、Hporf

## 六、命令

### 1）内存：dumpsys meminfo

可以通过如下命令查看包为` com.example.kotlindemo`的内存信息

```java
C:\Users> adb shell dumpsys meminfo com.example.kotlindemo
Applications Memory Usage (in Kilobytes):
Uptime: 5765033 Realtime: 5765033
** MEMINFO in pid 19462 [com.example.kotlindemo] **
                   Pss  Private  Private  SwapPss     Heap     Heap     Heap
                 Total    Dirty    Clean    Dirty     Size    Alloc     Free
                ------   ------   ------   ------   ------   ------   ------
  Native Heap    28070    28012       32      148    92160    34457    57702
  Dalvik Heap        0        0        0        0     8199     4100     4099
        Stack       96       96        0        0
       Ashmem       13        0       12        0
      Gfx dev     3568     3568        0        0
    Other dev        2        0        0        0
     .so mmap     7331      388     3484        9
    .apk mmap      168        0       20        0
    .ttf mmap      143        0       60        0
    .dex mmap     4965       12     3104        0
    .oat mmap      219        0        0        0
    .art mmap     8397     7728      348       84
   Other mmap      107        4        0        0
   EGL mtrack    24660    24660        0        0
    GL mtrack     2684     2684        0        0
      Unknown    31770    31732        4       27
        TOTAL   112461    98884     7064      268   100359    38557    61801

 App Summary
                       Pss(KB)
                        ------
           Java Heap:     8076
         Native Heap:    28012
                Code:     7068
               Stack:       96
            Graphics:    30912
       Private Other:    31784
              System:     6513

               TOTAL:   112461       TOTAL SWAP PSS:      268
 Objects
               Views:       18         ViewRootImpl:        2
         AppContexts:        4           Activities:        1
              Assets:        9        AssetManagers:        0
       Local Binders:       14        Proxy Binders:       33
       Parcel memory:        9         Parcel count:       21
    Death Recipients:        2      OpenSSL Sockets:       11
            WebViews:        0
 SQL
         MEMORY_USED:        0
  PAGECACHE_OVERFLOW:        0          MALLOC_SIZE:        0
```

这里的单位是kb，其中

- Private Dirty：是应用独占内存大小，包含独自分配的部分和应用进程从Zygote复制时被修改的Zygote分配的内存页。（重点关注之一）

- private clean：是已经映射持久文件使用的内存页，比如正在被执行的代码。

- Pss Total：实际使用的内存，将跨进程共享页也加入进来，会比在profiler中的要大一些。（重点关注之一）

- Dalvik Heap：Dalvik 虚拟机分配的内存。

- Java Heap：java堆大小。

- `Objects` 中显示持有对象的个数，从这里我们可以分析view、activity的个数。其中，可以通过看activity的个数判断是否发生内存泄漏。

### 2）systrace

`systrace`需要Python环境， Android SDK 工具软件包中提供该命令，对应路径是 `android-sdk/platform-tools/systrace/` 

语法

```java
  python systrace.py [options] [categories]
```

在执行完后，会自动生成一个html文件。可以用Chrome打开，在Chrome浏览器网址栏输入

```java
chrome://tracing/
```

点击load 按钮，选择我们的trace文件即可

<img src="/Users/wei/blog/weiwangqiang.github.io/img/blog_android_performance/11.png" width="100%" height="40%">

其中

- 区域1是总CPU的使用情况

- 区域2是指定进程的cpu使用情况

- 区域3是用于鼠标的控制功能，从上往下依次为点击、上下左右移动、点击上下拉缩放、框定时间区域。

- 顶部的processes可以选择你感兴趣的进程

- 右上角”？“可以查看操作信息

- metrics栏可以查看各项指标。

  

——Weiwq  于 2021.05 广州


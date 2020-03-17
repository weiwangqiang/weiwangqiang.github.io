---
layout:     post
title:      "android调试——教你用dumpsys命令调试"
subtitle:   " \"聊聊dumpsys的用法\""
date:       2020-03-16 13:30:00
author:     "Weiwq"
header-img: "img/background/post-bg-re-vs-ng2.jpg"
catalog: true
tags:
    - android
---

> “这一篇就聊聊dumpsys 比较常用的命令“

### dumpsys 服务

基本用法

```java

adb shell dumpsys [-t timeout] [--help | -l | --skip services | service [arguments] | -c | -h]

```

查看可与 `dumpsys` 配合使用的系统服务的完整列表，请使用以下命令：

```java
adb shell dumpsys -l
```

某些服务可能允许您传递可选参数。您可以通过将 `-h` 选项与服务名称一起传递来了解这些可选参数，如下所示：

```java
adb shell dumpsys procstats -h
```

```java
// 查看 com.demo.package 应用的 servicecs 启动记录
adb shell dumpsys activity services -p com.demo.package

// 查看activity启动记录 
adb shell dumpsys activity
```

官网有一些用法的说明 可见 [dumpsys命令](https://developer.android.com/studio/command-line/dumpsys.html?hl=zh-cn)

其实官网上的介绍也是比较简单，并且没有什么实际用途，这里就跟据不同的服务详细介绍，方便开发中的调试工作

### 1、dumpsys SurfaceFlinger

查看屏幕静态帧信息

```java
Static screen stats:
  < 1 frames: 0.335 s (1.5%) // 每帧的耗时情况
  < 2 frames: 2.049 s (9.1%)
	。。。。
  7+ frames: 19.625 s (87.1%)
```

查看BufferLayer（缓冲区）情况

```java

+ BufferLayer (WindowToken{3c467fd android.os.BinderProxy@3eec54}#0)
  Region TransparentRegion (this=a2fed158 count=1)
    [  0,   0,   0,   0]
  Region VisibleRegion (this=a2fed008 count=1)
    [  0,   0,   0,   0]
  Region SurfaceDamageRegion (this=a2fed044 count=1)
    [  0,   0,   0,   0]
      layerStack=   0, z=        0, pos=(0,0), size=(2560,2560), crop=[  0,   0,  -1,  -1], finalCrop=[  0,   0,  -1,  -1], isOpaque=0, invalidate=1, dataspace=Default, defaultPixelFormat=RGBx_8888, color=(0.000,0.000,0.000,1.000), flags=0x00000000, tr=[1.00, 0.00][0.00, 1.00]
      parent=mAboveAppWindowsContainers#0  // 所属父类
      zOrderRelativeOf=none
      activeBuffer=[   0x   0:   0,Unknown/None], queued-frames=0, mRefreshPending=0, windowType=-1, appId=-1
```

Displays信息（比如屏幕分辨率）

```java
+ DisplayDevice: Built-in Screen
   type=0, hwcId=0, layerStack=0, (1280x 720), ANativeWindow=0xa52e7808 (8:8:8:8), orient= 0 (type=00000000), flips=280, isSecure=1, powerMode=2, activeConfig=0, numLayers=2
   v:[0,0,1280,720], f:[0,0,1280,720], s:[0,0,1280,720],transform:[[1.000,0.000,-0.000][0.000,1.000,-0.000][0.000,0.000,1.000]]
   wideColorGamut=0, hdr10=0, colorMode=ColorMode::NATIVE, dataspace: Default (0)
  FramebufferSurface: dataspace: Default(0)
   
default-size=[1280x720] default-format=1 transform-hint=00 frame-counter=280 // 屏幕分辨率信息
```

其他信息

```java
EGL implementation : 1.4 Linux-r8p0-01rel0 // EGL 版本
GLES: ARM, Mali-450 MP, OpenGL ES 2.0 3d6a80e // OpenGL 架构信息
refresh-rate              : 60.000002 fps // 刷新率
x-dpi                     : 160.156998 // x dpi
y-dpi                     : 160.421005 // y dpi
VSYNC state: disabled // 垂直同步状态
soft-vsync: disabled // 当屏幕亮着的，就是disabled，如果关闭屏幕，这里为enabled
```

### 2、dumpsys activity

- **dumpsys activity o**

  获取OOM等级信息，home进程、上一次的进程内容

  ```java
  generic_x86:/ $ dumpsys activity o
    OOM levels:
      -900: SYSTEM_ADJ (   32,768K) 。。。
        
    mHomeProcess: ProcessRecord{a3cea50 18515:com.android.launcher3/u0a19}
    mPreviousProcess: ProcessRecord{3758933 23311:com.android.settings/1000}
  ```

- **dumpsys activity a**

  从顶部到底部打印 TaskRecord（activity） 信息，包含如下内容

  ```java
  * TaskRecord{9e6567 #150 A=com.android.settings U=0 StackId=1 sz=2}
    * Hist #1: ActivityRecord{e5fb34f u0 com.android.settings/.Settings$MemorySettingsActivity t150}
            packageName=com.android.settings processName=com.android.settings
            launchedFromUid=1000 launchedFromPackage=com.android.settings userId=0
            app=ProcessRecord{3758933 23311:com.android.settings/1000}
            Intent { flg=0x8000 cmp=com.android.settings/.Settings$MemorySettingsActivity (has extras) }
            frontOfTask=false task=TaskRecord{9e6567 #150 A=com.android.settings U=0 StackId=1 sz=2}
            taskAffinity=null
            realActivity=com.android.settings/.Settings$MemorySettingsActivity
            baseDir=/system/priv-app/Settings/Settings.apk
            dataDir=/data/user_de/0/com.android.settings
            state=RESUMED stopped=false delayedResume=false finishing=false
  ```

- **dumpsys activity r**

  打印最近的TaskRecord信息，信息内容与a参数的类似

- **dumpsys activity  b [PACKAGE_NAME]**

  打印指定包创建的进程信息，包含已经注册的广播接收器action信息，创建的handler

  ```java
  generic_x86:/ $ dumpsys activity b com.google.android.ext.services
  ACTIVITY MANAGER BROADCAST STATE (dumpsys activity broadcasts)
  
    mBroadcastsScheduled [foreground]=false
    mBroadcastsScheduled [background]=false
    mHandler:
      Handler (com.android.server.am.ActivityManagerService$MainHandler) {32cc7bf} @ 26049725
        Looper (ActivityManager, tid 16) {ba76e8c}
          Message 0: { when=+8m0s963ms what=27 target=com.android.server.am.ActivityManagerService$MainHandler }
          Message 1: { when=+10m14s258ms callback=com.android.server.am.ActiveServices$1 target=com.android.server.am.ActivityManagerService$MainHandler }
          Message 2: { when=+23m38s921ms callback=com.android.server.AppOpsService$1 target=com.android.server.am.ActivityManagerService$MainHandler }
          (Total messages: 3, polling=true, quitting=false)
  ```

- **dumpsys activity p [PACKAGE_NAME]**

  打印指定包创建的进程信息，包含：<br>

  requiredAbi：要求的架构<br>lastSwapPss：内存使用情况<br>lastRequestedGc：上次GC请求<br>

  ```java
  generic_x86:/ $ dumpsys activity p com.google.android.ext.services
  ACTIVITY MANAGER RUNNING PROCESSES (dumpsys activity processes)
    requiredAbi=x86 instructionSet=null
    adjSeq=18970 lruSeq=0 lastPss=4.1MB lastSwapPss=0.00 lastCachedPss=0.00 lastCachedSwapPss=0.00
    lastRequestedGc=-4h9m54s33ms lastLowMemory=-4h9m54s33ms reportLowMemory=false .....
    
  ```

- **dumpsys activity s**

  打印ServiceRecord信息，包含<br>intent：intent信息<br>packageName：包名<br>IntentBindRecord：已绑定该服务的信息

  ```java
  
  ServiceRecord{a4d66c6 u0 com.android.chrome/org.chromium.chrome.browser.omaha.OmahaClient}
      intent={act=org.chromium.chrome.browser.omaha.ACTION_REGISTER_REQUEST cmp=com.android.chrome/org.chromium.chrome.browser.omaha.OmahaClient}
      packageName=com.android.chrome ....
  ```

- **dumpsys activity settings**

  打印ServiceRecord信息，包含<br>intent：intent信息<br>packageName：包名<br>IntentBindRecord：已绑定该服务的信息

  ```java
  generic_x86:/ $ dumpsys activity settings
  TASK com.android.settings id=150
    ACTIVITY com.android.settings/.Settings$MemorySettingsActivity e5fb34f pid=23311
      Local Activity ebdaec State:
        mResumed=true mStopped=false mFinished=false .....
  ```

- **dumpsys activity all**

  打印全部的activity，包含<br>生命周期状态<br>View Hierarchy:  view的层次结构

  ```java
  generic_x86:/ $ dumpsys activity all
  TASK com.example.flutter_web id=149  ...
    
      View Hierarchy:
        DecorView@6f37f9c[MainActivity]
          android.widget.LinearLayout{5370da5 V.E...... .......D 0,0-1080,1794}
            android.view.ViewStub{137457a G.E...... ......I. 0,0-0,0 #10203e8 android:id/action_mode_bar_stub}
            android.widget.FrameLayout{20ff12b V.E...... .......D 0,0-1080,1794 #1020002 android:id/content} ....
  ```

- **dumpsys activity top**

  与all参数类似，但是只打印顶层activity的信息
  
- **dumpsys activity starter**

  打印对应的启动者

- **dumpsys activity lastanr**

  打印 ANR list信息

### 3、 dumpsys cpuinfo

查看CPU使用情况

```java
CPU usage from 1251999ms to 351197ms ago (2020-02-23 17:21:02.541 to 2020-02-23 17:36:03.343):
  4.2% 20215/com.android.chrome:sandboxed_process0: 1.1% user + 3% kernel / faults: 396490 minor 1 major
   .....
6.6% TOTAL: 1.2% user + 4.9% kernel + 0% iowait + 0% irq + 0.3% softirq
```

### 4、dumpsys diskstats

查看磁盘使用情况

```java
Latency: 1ms [512B Data Write]
Data-Free: 1195820K / 2031440K total = 58% free
Cache-Free: 61000K / 62416K total = 97% free
System-Free: 250356K / 2031440K total = 12% free
```

### 5、dumpsys display

屏幕物理信息

```java
mDefaultViewport=DisplayViewport{valid=true, displayId=0, orientation=0, logicalFrame=Rect(0, 0 - 1080, 1920), physicalFrame=Rect(0, 0 - 1080, 1920), deviceWidth=1080, deviceHeight=1920}
  mExternalTouchViewport=DisplayViewport{valid=false, displayId=0, orientation=0, logicalFrame=Rect(0, 0 - 0, 0), physicalFrame=Rect(0, 0 - 0, 0), deviceWidth=0, deviceHeight=0}


mDisplayInfos=
   PhysicalDisplayInfo{1920 x 1080, 60.000004 fps, density 1.5, 159.895 x 160.421 dpi, secure true, appVsyncOffset 1000000, bufferDeadline 16666666}
```

### 6、dumpsys ethernet 

查看IP，网关（ifconfig也能查看)

```java

onNewDhcpResults({IP address 172.19.110.23/24 Gateway 172.19.110.1  DNS servers: [ 10.254.254.254 ] Domains gz.cvte.cn DHCP server /10.22.4.10 Vendor info null lease 7200 seconds})

```

 ### 7、dumpsys  meminfo

查看系统各个应用内存使用情况

```java
Total PSS by process:
     36,511K: system (pid 1929)
     3,903K: android.ext.services (pid 2881)
     .....
Total PSS by OOM adjustment:
     76,848K: Native
          4,411K: zygote (pid 1676)
          .....
Total PSS by category:
     78,104K: Native
     22,149K: Dalvik
    	......

Total RAM:   991,316K (status critical)
 Free RAM:   412,181K (   10,761K cached pss +   144,448K cached kernel +   256,972K free)
 Used RAM:   433,189K (  326,881K used pss +   106,308K kernel)
 Lost RAM:   193,772K
     ZRAM:    16,628K physical used for    74,012K in swap (  317,216K total swap)
   Tuning: 128 (large 256), oom   184,320K, restore limit    61,440K (low-ram)
```

### 8、dumpsys overlay

查看overlay的包状态

```java
com.android.systemui.theme.dark:0 {
    mPackageName.......: com.android.systemui.theme.dark  //overlay 的apk 包名
    mUserId............: 0
    mTargetPackageName.: com.android.systemui // 目标包名
    mBaseCodePath......: /vendor/overlay/SysuiDarkTheme/SysuiDarkThemeOverlay.apk // overlay的apk路径
    mState.............: STATE_DISABLED
    mIsEnabled.........: false 
    mIsStatic..........: false 
    mPriority..........: 1
    mCategory..........: null
  }
```

### 9、dumpsys package

- **dumpsys package l**

  列举已知的lib库

  ```java
  Libraries:
    android.test.base ->  (jar) /system/framework/android.test.base.jar ....
  ```
  
- **dumpsys package f**

  列举设备支持的功能

  ```java
  Features:
    android.software.leanback_only ......
  ```
  
- **dumpsys package r**

  列举 所有的 [activity|service|receiver|content ] intent 解析器，其中包含各个应用的入口，action对应的组件
  
  ```java
  Activity Resolver Table: // activity 入口
    Non-Data Actions: 
         android.settings.HOME_SETTINGS:  // action 
            c25027 com.android.tv.settings/.EmptyStubActivity  // 响应该action的组件
       ....
  MIME Typed Actions:
        android.intent.action.INSTALL_PACKAGE:
          7215b10 com.android.packageinstaller/.InstallStart
         .....
  Receiver Resolver Table:  // 广播接收器
  Non-Data Actions:
        android.intent.action.LOCALE_CHANGED:
  			        1a15a84 com.android.providers.media/.MediaScannerReceiver
  ```
  
- **dumpsys package permission [permissionName ....]**

  列举所有申明该权限的应用信息，其中可以查看应用的版本信息，标签，使用的lib库和jar路径，首次安装时间，更新时间，签名信息，已经获取的权限，安装需要的权限以及overlay文件的路径

  ```java
  xxxx:/ $ dumpsys package permission android.permission.VIBRATE
  Packages:
    pkg=Package{a077660 com.ecloud.eshare.server}
    codePath=/system/priv-app/eshare-service
    versionCode=20190820 minSdk=8 targetSdk=8
    versionName=v5.8.20
     ..... 
     
  ```
  
- **dumpsys package preferred-xml**
  
  列举首选应用的设置，以xml的方式输出，（笔者这里没有设置，所以为空）
  
  ```java
  xxxx:/ $ dumpsys package preferred-xml
  <?xml version='1.0' encoding='utf-8' standalone='yes' ?>
  <preferred-activities />
  ```
  
- **dumpsys package prov**
  
  列举所有的内容提供者信息
  
  ```java
  xxxx:/ $ dumpsys package prov
  Registered ContentProviders:
    com.android.systemui/.keyguard.KeyguardSliceProvider:
      Provider{b98064f com.android.systemui/.keyguard.KeyguardSliceProvider}
    com.android.browser/.homepages.HomeProvider:
      Provider{1c0f1dc com.android.browser/.homepages.HomeProvider}
  ```
  
- **dumpsys package p**
  
  获取所有已经安装的apk包信息，内容展示的格式与 *dumpsys package permission* 类似，这里就不做举例
  
- **dumpsys package  <package.name>**
  
  获取指定包的信息，可以获取到四大组件的信息，包信息（版本信息，lib库），已经请求的权限，安装需要的权限，运行时权限，shareUser信息（可以判断是否是系统应用）
  
  ```java
  xxxx:/ # dumpsys package com.ecloud.eshare.server
  Activity Resolver Table:
    Non-Data Actions:
        android.intent.action.MAIN:
          9baa84b com.ecloud.eshare.server/.CifsClientActivity filter 6b0c11f
            Action: "android.intent.action.MAIN"
            Category: "android.intent.category.LAUNCHER"
             .....
  Shared users:
    SharedUser [android.uid.system] (db280e1):
  ```
  
- **dumpsys package s**
  
  列举shared uses的信息，权限，对应的gid
  
  ```java
  xxxx:/ # dumpsys package s
  Shared users:
    SharedUser [android.uid.system] (d6d5db):
      userId=10012
      install permissions:
        android.permission.ACCESS_CACHE_FILESYSTEM: granted=true
      User 0:
        gids=[2001, 1065, 1023, 3003, 3007, 1024]
  ```
  
- **dumpsys package  check-permission <permission> <package>**
  
  检查对应的包是否已经获得对应的权限
  
  ```java
  Hi3751V350:/ # dumpsys package check-permission android.permission.WRITE_EXTERNAL_STORAGE com.demo.test
    0
  Hi3751V350:/ # dumpsys package check-permission android.permission.INTERNET com.demo.test
  -1
  ```
### 10、dumpsys power

  可以查看电池状态

```java
mBatteryLevel=100 // 电量
mLowPowerModeEnabled=false //是否处于省电模式
mBatteryLevelLow=false //电池是否底
```

### 11、dumpsys procstats

可以查看应用运行时的 PSS、USS数据，包括最小值、平均值、最大值，例如查看过去一小时内存使用情况，其中数据部分是按照（最小PSS-平均PSS-最大PSS/最小USS-平均USS-最大USS）的格式显示出PSS和USS

```java
generic_x86:/ $ dumpsys procstats --hours 1
AGGREGATED OVER LAST 1 HOURS:
  * com.android.inputmethod.latin / u0a56 / v24:
           TOTAL: 100% (17MB-14MB-17MB/14MB-14MB-14MB over 2)
          Imp Bg: 100% (17MB-14MB-17MB/14MB-14MB-14MB over 2)

Memory usage:
	...	 //	系统内存占比情况
  CchEmty: 109MB (16 samples)
  TOTAL  : 516MB
```

### 12、dumpsys window

- **dumpsys window l**

  （或者 dumpsys window lastanr）用于打印当前窗口信息

```java
WINDOW MANAGER LAST ANR (dumpsys window lastanr)
  <no ANR has occurred since boot> // 获取ANR的信息，具体log可见 /data/anr/ 路径下
```

- **dumpsys window p**

  获取当前采用的政策状态，比如获取焦点的window、app

### 后记

dumpsys 命令的功能很多，但是有一些对与应用层面来说用不上，比如没有列举的 dumpsys battery（打印电池信息），

为了提高效率，这里只是把有关UI调试方面的内容列举了一下，更多的还是得看dumpsys的每一个服务提供的功能

—— Weiwq 后记于 2020.03 广州


---
layout:     post
title:      "Android App性能监控工具"
subtitle:   " \"多维度分析app的性能\""
date:       2021-08-15 23:30:00
author:     "Weiwq"
header-img: "img/background/post-sample-image.jpg"
catalog:  true
top: false
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
<img src="/img/blog_android_performance/10.png" width="35%" height="40%"/><img src="/img/blog_android_performance/9.png" width="35%" height="40%"/> 
</center>


## 二、BlockCanary

blockcanary 最新的一个版本是2017年发布的，已经很久没维护了，但是其原理还是值得借鉴的。[作者文章](http://blog.zhaiyifan.cn/2016/01/16/BlockCanaryTransparentPerformanceMonitor/)

导入依赖方式：

```java
// 当然也可以用implementation，建议用debug方式导入
debugImplementation 'com.github.markzhai:blockcanary-android:1.5.0'
```

需要注意的是由于涉及到读写文件，所以还需要声明对应的权限

```java
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
```

创建一个application，并在Androidmanifest中使用

```java
public class BlockCanaryApplication extends Application {
    @Override
    public void onCreate() {
        super.onCreate();
        BlockCanary.install(this, new AppBlockCanaryContext()).start();
    }
}

public class AppBlockCanaryContext extends BlockCanaryContext {
    private static final String TAG = "AppBlockCanaryContext";

    // block 时会回调
    @Override
    public void onBlock(Context context, BlockInfo blockInfo) {
        super.onBlock(context, blockInfo);
        Log.d(TAG, "onBlock: " + blockInfo.model);
    }

    //卡顿阀值 ，默认是1000 ms
    @Override
    public int provideBlockThreshold() {
        return 200;
    }
}
```

展示dump信息的页面基于LeakCanary界面修改，可以很清楚看到哪里卡了和卡的时长

<img src="/img/blog_android_performance/17.png" width="40%" height="40%">

## 三、Perfdog

Perfdog是由腾讯出品的移动平台性能分析工具，官网[点我前往](https://perfdog.qq.com/)，工具首页如下，默认的功能有：FPS、CPU、memory三个维度的性能。

- 右下角可以扩展更多功能
- 点击右上角可以开始记录数据，再点一下可以保存到云平台。

<img src="/img/blog_android_performance/1.png" width="100%" height="40%">

在官网登录后就可以看到对应的详细数据

<img src="/img/blog_android_performance/2.png" width="100%" height="40%">

## 四、Profiler

Perfdog 只适合用于监控CPU，内存、FPS等情况，如果想具体排查问题，还是得用Android studio自带的Profiler

profiler支持CPU、memory、network、energy维度的分析。

<img src="/img/blog_android_performance/3.png" width="100%" height="40%">

### 1、CPU

在CPU下，点击record可以开始记录一段时间内的方法耗时情况

<img src="/img/blog_android_performance/4.png" width="100%" height="40%">

点击stop后，就可以看具体的执行耗时情况，比如在main线程中，clickView的方法耗时长达270ms，就可以结合代码做具体的耗时分析。

<img src="/img/blog_android_performance/5.png" width="100%" height="40%">

### 2、Memory

在Memory下，可以看到每个块所占用的内存大小，如果想具体看内存分配情况，可以点击顶部的“Allocation Tracking”

<img src="/img/blog_android_performance/6.png" width="100%" height="40%">

在点击stop后，就会进入如下页面

- 区域1：可以按照不同的归类来查看内存情况
- 区域2：每个类实例对应的内存分配情况，单位是byte。点击对应的分类，可以按照该分类的内存情况升序或者降序排列。
- 区域3：该实例对应的成员变量和引用链，对于分析内存泄露很有帮助
- 区域4：该类的实例列表，正常列表只有一个，如果有多个，有可能发生了内存泄露。

<img src="/img/blog_android_performance/7.png" width="100%" height="40%">

### 3、Network

可以测试网络的收发速度

<img src="/img/blog_android_performance/8.png" width="100%" height="40%">

## 五、命令

### 1）dumpsys meminfo

可以通过如下命令查看包为` com.example.kotlindemo`的内存信息（也可以查看PID对应的信息）

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

- Pss Total：实际使用的内存，将跨进程共享页也加入进来，会比在profiler中的要大一些。（重点关注之一）
- Private Dirty：是应用独占内存大小，包含独自分配的部分和应用进程从Zygote复制时被修改的Zygote分配的内存页。（重点关注之一）
- Private clean：是已经映射持久文件使用的内存页，比如正在被执行的代码。
- Dalvik Heap：Dalvik 虚拟机分配的内存。
- Java Heap：java堆大小。
- `Objects` 中显示持有对象的个数，从这里我们可以分析view、activity的个数。其中，可以通过看activity的个数判断是否发生内存泄漏。

### 2）systrace

#### 基本用法

`systrace`需要Python环境，在使用前请先配置好adb，并连接上手机。

window环境可以按照如下配置

- python 2.7 环境：用 `py -2` 或者 `py -3` 切Python2和Python3
- six模块：安装命令`py -2 -m pip install six`
- win32con模块：安装命令`py -2 -m pip install pypiwin32`

 Android SDK 工具软件包中提供该命令，对应路径是 `android-sdk/platform-tools/systrace/` 语法如下：

```java
python2 systrace.py [options] [categories]
```

可用参数如下

| 命令和选项   | 说明                                                         |
| ------------ | ------------------------------------------------------------ |
| -o file      | 将 HTML 跟踪报告写入指定的文件。如果您未指定此选项，`systrace` 会将报告保存到 `systrace.py` <br>所在的目录中，并将其命名为 `trace.html`。 |
| -t N         | 跟踪设备活动 N 秒。如果您未指定此选项，`systrace` 会提示您在命令行中按 Enter 键结束跟踪。 |
| -b N         | 使用 N KB 的跟踪缓冲区大小。使用此选项，您可以限制跟踪期间收集到的数据的总大小。 |
| -k functions | 跟踪逗号分隔列表中指定的特定内核函数的活动。                 |
| -a app-name  | 启用对应用的跟踪，指定为包含[进程名称](https://developer.android.google.cn/guide/topics/manifest/application-element#proc)的逗号分隔列表。 |
| -h           | 显示帮助消息。                                               |

在执行完后，会自动生成一个html文件。可以用Chrome打开，在Chrome浏览器网址栏输入

```java
chrome://tracing/
```

点击load 按钮，选择我们的trace文件即可

<img src="/img/blog_android_performance/11.png" width="100%" height="40%">

其中

- 区域1是总CPU的使用情况
- 区域2是指定进程的cpu使用情况
- 区域3是用于鼠标的控制功能，从上往下依次为点击、上下左右移动、点击上下拉缩放、框定时间区域。
- 顶部的processes可以选择你感兴趣的进程
- 右上角”？“可以查看操作信息
- metrics栏可以查看各项指标。

更多见[浏览systrace报告](https://developer.android.google.cn/topic/performance/tracing/navigate-report)

#### GC

在启动过程，要尽量减少 GC 的次数，避免造成主线程长时间的卡顿，特别是对 Dalvik 来说，我们可以通过 systrace 单独查看整个启动过程 GC 的时间。

```java
python2 ./systrace.py dalvik -b 90960 -a com.sample.gc
```

对于GC的使用含义，可以参考[调查RAM使用情况](http://developer.android.com/studio/profile/investigate-ram?hl=zh-cn)

#### 自定义Trace

Android在关键系统回调添加了Trace，但是我们无法通过systrace准确获取到项目代码的耗时情况，因此需要在关键位置打label。

Android提供了`android.os.Trace#beginSection()` 和 `android.os.Trace#endSection` 这两个接口。比如想排查onCreate方法中，耗时的情况，可以通过如下代码实现

```java
class MainActivity : AppCompatActivity() {
   override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        // 指定label为 "MainActivity_setContentView"
        Trace.beginSection("MainActivity_setContentView")
        method1()
        method2()
        Trace.endSection()
    }
    private fun method2() {
        Thread.sleep(300)
    }

    private fun method1() {
        Thread.sleep(200)
    }
}
```

**重点：**要想使用自定义Label，就必须在gradle中，打开debuggable

```java
  buildTypes {
        debug {
            debuggable true
        }
    }
```

执行systrace，其中`-a` 表示开启指定包自定义Trace label的功能

```cmd
$  python2 .\systrace.py -a com.example.asm
```

打开应用，进入指定页面后，退出systrace。用chrome打开trace.html 文件。如下所示，可以看到systrace对性能的影响很小。

<img src="/img/blog_android_performance/20.png" width="100%" height="100%">

但是由于开启了debuggable，与正式的release版本在性能上会有较大差异，如果我们想尽可能的还原真实情况，那必须在非debuggable模式下执行。我们可以通过如下方式绕过debuggable的限制

```cmd

public class BaseApplication extends Application {
    @Override
    protected void attachBaseContext(Context base) {
        super.attachBaseContext(base);
        try {
            Class<?> trace = Class.forName("android.os.Trace");
            Method setAppTracingAllowed = trace.getDeclaredMethod("setAppTracingAllowed", boolean.class);
            setAppTracingAllowed.invoke(null, true);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

**如果自定义label，再配上插桩，那就能最接近真实情况下，详细分析每个方法的耗时情况。**

参考[手把手教你使用Systrace](https://zhuanlan.zhihu.com/p/27331842)

### 3）Perfetto

Perfetto 是 Android 10 中引入的全新平台级跟踪工具，你可以在[perfetto界面](https://ui.perfetto.dev/#!/record)中打开这些跟踪

<img src="/img/blog_android_performance/12.svg" width="100%" height="40%">

或者可以通过命令方式打开

```java
cd /path-to-traces-on-my-dev-machine
systrace --from-file trace-file-name{.ctrace | .perfetto-trace}
```

更多见[系统跟踪](https://developer.android.google.cn/topic/performance/tracing)

### 4）Trace文件

Android为我们提供了Debug工具，可以获取指定路径的trace文件，我们只需要在特定的位置加入如下代码，即可获取对应的trace文件

```java
// 设置开始记录方法调用情况
Debug.startMethodTracing("/sdcard/debug.trace");

// 结束记录方法调用情况
Debug.stopMethodTracing();
```

将trace文件pull出来后，直接把文件拖拽到Android studio中即可。区域1为各个线程的耗时情况，区域2 为对应的火焰图。

<img src="/img/blog_android_performance/18.png" width="100%" height="40%">

### 5）Hprof文件

通过如下获取hprof文件，需要注意如下代码十分的耗性能

```java
Debug.dumpHprofData("/sdcard/dump.hprof")
```

pull出来用Android studio打开如下

<img src="/img/blog_android_performance/19.png" width="100%" height="40%">

### 6）查看RAM

```cmd
$ cat /proc/meminfo | head -n 4
MemTotal:        5861796 kB // 总共内存
MemFree:           90268 kB // 空闲内存
MemAvailable:    3392216 kB // 可用内存（即剩余运行内存）
Buffers:         1049552 kB // 缓存
```

### 7）查看CPU——top

```cmd
$ top
Tasks: 743 total,   2 running, 737 sleeping,   0 stopped,   0 zombie
Mem:   5861796k total,  5683832k used,   177964k free,  1057640k buffers
Swap:  2621436k total,   478088k used,  2143348k free,  1994168k cached
800%cpu  21%user   4%nice   8%sys 764%idle   1%iow   2%irq   1%sirq   0%host
  PID USER         PR  NI VIRT  RES  SHR S[%CPU] %MEM     TIME+ ARGS
 5407 u0_a99       20   0 1.7G  69M  45M S 19.0   1.2  13:15.58 com.miui.voiceassist:voice_trigger
13736 shell        20   0  14M 3.0M 1.6M R  3.3   0.0   0:00.25 top
  704 audioserver  20   0  41M 6.3M 5.9M S  2.3   0.1   2:09.01 android.hardware.audio@2.0
```

其中

- 800%cpu：即cpu核数，这里800%指有8个核。
- 21%user：即用户态的占比占单核的21%
- 4%nice：优先级为负数的进程占用的cpu
- 8%sys：处于核心态的cpu占比
- 764%idle：CPU处于空闲状态时间比例。一般而言，idel + sys + user + nice 约等于cpu
- 1%iow：IO等待的CPU占比
- 2%irq：硬中断的cpu占比
- 1%sirq：软中断的cpu占比
- PID：即进程id
- PR：优先级
- VIRT：虚拟内存大小，包括：进程使用的库、代码、数据等。
- RES：常驻内存，当前进程使用的内存大小，不包含swap out
- SHR：除了自身进程的共享内存，也包括其他进程的共享内存。
- %CPU：即该进程占用的CPU占比。
- %MEM ：即该进程占用的内存占比。

### 8）查看CPU——dumpsys

可以查看每个进程所用的CPU百分比

```cmd
$ dumpsys cpuinfo
CPU usage from 476124ms to 176038ms ago (2021-10-17 13:35:57.070 to 2021-10-17 13:40:57.156):
  17% 5407/com.miui.voiceassist:voice_trigger: 16% user + 0.4% kernel / faults: 10382 minor
  2.6% 704/android.hardware.audio@2.0-service: 0.7% user + 1.9% kernel
```

更多dumpsys命令见[android调试——教你用dumpsys命令调试](https://weiwangqiang.github.io/2020/03/16/dumpsys-debug-detail/)

## 六、GPU

### 1）渲染速度

可以通过 设置-》开发者选项-》监控下的GPU呈现方式-》在`GPU 渲染模式分析`对话框中，选择`在屏幕上显示为竖条`

或者参考[App性能调试详解](https://weiwangqiang.github.io/2019/06/23/android-debug-code/) 用命令打开。

```java
// Possible values:
// "true", to enable profiling
// "visual_bars", to enable profiling and visualize the results on screen
// "false", to disable profiling
// @see #PROFILE_PROPERTY_VISUALIZE_BARS
adb shell setprop debug.hwui.profile #{value}
```

效果如下

<img src="/img/blog_android_performance/12.png" width="60%" height="40%">

其中， Android 6.0 及更高版本的设备时分析器输出中某个竖条的每个区段如下所示：

<img src="/img/blog_android_performance/13.png" width="100%" height="40%">

下表显示的是 Android 4.0 和 5.0 中的竖条区段。

<img src="/img/blog_android_performance/14.png" width="100%" height="40%">

### 3）过渡绘制

可通过 设置-》开发者选项-》硬件加速渲染-》调试 GPU 过度绘制-》选择**显示过度绘制区域**。

或者使用命令打开

```java
adb shell setprop debug.hwui.overdraw show
```

效果如下：

<img src="/img/blog_android_performance/16.png" width="50%" height="40%">

Android 将按如下方式为界面元素着色，以确定过度绘制的次数：

<img src="/img/blog_android_performance/15.png" width="50%" height="40%">

## 后记

到目前为止，Google一直为Android提供新的调试工具，从monitor（已经被打入冷宫）到profiler，从systrace到Perfetto，Android studio也一直在迭代更新。目的就是让开发者能开发出更优秀的产品，也愿各位大佬不辜负Google的期望！共勉！！

——Weiwq  于 2021.08 广州


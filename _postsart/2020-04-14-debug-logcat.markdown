---
layout:     post
title:      "android调试——logcat详解"
subtitle:   " \"logcat到底怎么用才爽？\""
date:       2020-04-14 13:30:00
author:     "Weiwq"
header-img: "img/background/post-bg-re-vs-ng2.jpg"
catalog: true
tags:
    - android
---

> “为了更爽的使用logcat，我决定好好研究一下“

## 1、基本命令
logcat的格式如下

```java

logcat [options] [filterspecs]

```

比如需要过滤TAG是 "demo" 的log

```java

logcat -s demo

```

全部命令选项如下

|选项|说明|
|---|----|
|`-s`|相当于过滤器表达式 `'*:S'` <br> 例如：`logcat -s demo`|
|`-f <file>`|`--file=<file>` <br>设置logcat 内容保存的位置，默认是stdout<br> 例如:  logcat -f sdcard/log.txt|
|`-r <kbytes>`|`--rotate-kbytes=<kbytes>` <br>每输出 `<kbytes>` 时轮替日志文件，默认是16 <br> 例如：`logcat -f sdcard/log.txt -r 1`|
|`-b <buffer>`|加载可供查看的备用日志缓冲区，例如 `events` 或 `radio`。<br>默认使用 `main`、`system` 和 `crash` 缓冲区集。<br>请参阅[查看备用日志缓冲区](https://developer.android.google.cn/tools/debugging/debugging-log#alternativeBuffers)|
|`-c`|`--clear` <br/>清除（清空）所选的缓冲区并退出。<br>默认缓冲区集为 `main`、`system` 和 `crash`。<br>要清除所有缓冲区，请使用 `-b all -c`。|
|`-e <expr>`|`--regex=<expr>` <br>只输出日志消息与 `` 匹配的行，其中 `` 是一个正则表达式。|
|`-m <count>`|`--max-count=<count>` <br>输出 `m` 行后退出。这样是为了与 `--regex` 配对，但可以独立运行。|
|`--pid=<pid> ...`|仅输出来自给定 PID 的日志。<br>例如：`logcat --pid=4355`|
|`-D`|`--dividers` <br>输出各个日志缓冲区之间的分隔线。|
|`-t <time>`|输出自指定时间以来的最新行。此选项包括 `-d` 功能。<br>要了解如何引用带有嵌入空格的参数，请参阅 [-P 选项](https://developer.android.google.cn/studio/command-line/logcat#quotes)。<br>例如：`adb logcat -t '01-26 20:52:41.820'`|
|`-v <format>`|设置日志消息的输出格式。默认格式为 `threadtime`。<br>有关支持的格式列表，请参阅介绍[控制日志输出格式](https://developer.android.google.cn/studio/command-line/logcat#outputFormat)的部分。|
|`-g`|输出指定日志缓冲区的大小并退出。|
|`-G <size>`|`--buffer-size=<size>` <br> 设置log缓冲区的大小，后缀可以是K或者M<br>例如：`logcat -G 2M`|
|`-S`|`--statistics` <br>在输出中包含统计信息，以识别和定位日志垃圾信息发送者。(注意，S是大写的)|
|`-c`|清空（清除）整个日志并退出。|
|`-t <count>`|仅输出最新的行数。此选项包括 `-d` 功能。|
|`-t <time>`|输出自指定时间以来的最新行。此选项包括 `-d` 功能。<br>要了解如何引用带有嵌入空格的参数，请参阅 [-P 选项](https://developer.android.google.cn/studio/command-line/logcat#quotes)。<br>例如：`adb logcat -t '01-26 20:52:41.820'`|


## 2、控制日志输出格式

可以修改log输出格式，来显示特定的元数据字段，您可以用`-v` 选项，并指定一下某一受支持的输出格式。

- `brief`：显示优先级、标记以及发出消息的进程的 PID。
- `long`：显示所有元数据字段，并使用空白行分隔消息。
- `process`：仅显示 PID。
- `raw`：显示不包含其他元数据字段的原始日志消息。
- `tag`：仅显示优先级和标记。
- `thread:`：旧版格式，显示优先级、PID 以及发出消息的线程的 TID。
- `threadtime`（默认值）：显示日期、调用时间、优先级、标记、PID 以及发出消息的线程的 TID。
- `time`：显示日期、调用时间、优先级、标记以及发出消息的进程的 PID。

例如：

```java

adb logcat -v time
adb logcat -v time -v tag // 可以指定多字段
 
```

您可以通过在命令行中输入 `logcat -v --help` 获取格式修饰符详细信息。

- `color`：使用不同的颜色来显示每个优先级。
- `descriptive`：显示日志缓冲区事件说明。此修饰符仅影响事件日志缓冲区消息，不会对其他非二进制文件缓冲区产生任何影响。事件说明取自 event-log-tags 数据库。
- `epoch`：显示自 1970 年 1 月 1 日以来的时间（以秒为单位）。
- `monotonic`：显示自上次启动以来的时间（以 CPU 秒为单位）。
- `printable`：确保所有二进制日志记录内容都进行了转义。
- `uid`：如果访问控制允许，则显示 UID 或记录的进程的 Android ID。
- `usec`：显示精确到微秒的时间。
- `UTC`：显示 UTC 时间。
- `year`：将年份添加到显示的时间。
- `zone`：将本地时区添加到显示的时间。

## 3、查看备用日志缓冲区

Android 日志记录系统为日志消息保留了多个环形缓冲区，而且并非所有的日志消息都会发送到默认的环形缓冲区。要查看其他日志消息，您可以使用 `-b` 选项运行 `logcat` 命令，以请求查看备用的环形缓冲区。您可以查看下列任意备用缓冲区：

- `radio`：查看包含无线装置/电话相关消息的缓冲区。
- `events`：查看已经过解译的二进制系统事件缓冲区消息。
- `main`：查看主日志缓冲区（默认），不包含系统和崩溃日志消息。
- `system`：查看系统日志缓冲区（默认）。
- `crash`：查看崩溃日志缓冲区（默认）。
- `all`：查看所有缓冲区。
- `default`：报告 `main`、`system` 和 `crash` 缓冲区。

例如：

```java

adb logcat -b crash // 查看crash 缓冲区

```

## 后记

把logcat仔细研究一番，发现还是有挺多实用的技巧，比如，有时候会遇到logcat报如下问题：

```java

logcat read unexpected eof
  
```

实际上是缓冲区不足导致的，如过有看过上面的参数，马上就知道对应的解决方案了——修改缓冲区大小就能解决

对应参数是`-G`

所以，我们还是很有必要好好研究平常常用到的命名，温故知新。

——Weiwq 后记于 2020.04 广州
---
layout:     post
title:      "Android基础性能监控"
subtitle:   " \"聊聊Android性能监控\""
date:       2020-08-20 22:30:00
author:     "Weiwq"
header-img: "img/background/post-bg-re-vs-ng2.jpg"
catalog: true
tags:
    - Android

---

> “这里有Android系统性能接口常用的命令“

### 1、系统资源：top 

top命令输出如下所示：

```java

Tasks: 950 total,   1 running, 949 sleeping,   0 stopped,   0 zombie
Mem:   1530632k total,  1291128k used,   239504k free,    37800k buffers
Swap:        0k total,        0k used,        0k free,   671992k cached
400%cpu   1%user   0%nice   2%sys 397%idle   0%iow   0%irq   0%sirq   0%host
  PID USER         PR  NI VIRT  RES  SHR S[%CPU] %MEM     TIME+ THREAD          PROCESS
20981 root         20   0 8.9M 4.7M 3.1M R  3.3   0.3   0:00.13 top             top
 1909 system       18  -2 1.6G 220M 163M S  1.0  14.7  34:58.90 system_server   system_server
 1645 system       RT   0  11M 4.7M 4.1M S  0.6   0.3   4:53.92 HwBinder:1645_1 android.hardware.sensors@1.0-service
     ......

```

对应的列含义

| 列    | 含义                                                         | 列     | 含义                                          |
| ----- | ------------------------------------------------------------ | ------ | --------------------------------------------- |
| PID   | 进程ID                                                       | PPID   | 父进程ID                                      |
| UID   | 进程所有者的用户ID                                           | USER   | 进程所有者的用户名                            |
| GROUP | 进程所有者的组名                                             | TTY    | 启动进程的终端名，不是终端启动的进程则显示"?" |
| PR    | 优先级                                                       | NInice | 负值表示高优先级 ，正值表示低优先级           |
| %CPU  | CPU时间占用百分比                                            | TIME   | 进程使用CPU时间总计，单位秒                   |
| TIME+ | 进程使用的CPU时间总计，单位1/100秒                           | %MEM   | 进程使用的物理内存百分比                      |
| VIRT  | 进程使用的虚拟内存总量，单位KB<br/>VIRT=SWAP+RES             | SWAP   | 进程使用的虚拟内存中，被换出的大小，单位KB    |
| RES   | 进程使用的，未被换出的物理内存大小，单位KB。<br/>RES=CODE+DATA | CODE   | 可执行代码占用的物理内存大小，单位KB          |
| DATA  | 可执行代码意外的部分（数据段+栈）占用的物理内存大小<br/>单位KB | SHR    | 共享内存大小，单位KB                          |
| nFLT  | 页面错误次数                                                 | nDRT   | 最后一次写入到现在，被修改过的页面            |

### 2、磁盘和CPU：vmstat

vmstat命令输出如下所示：

```java

127|generic_x86:/ # vmstat
procs -----------memory---------- ---swap-- -----io---- -system-- ----cpu----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa
 1  0      0 241892  37800 671380    0    0     6     7    0  115  0  0 100 0

```

对应列含义

| 列   | 含义                                                         | 列    | 含义                                                         |
| ---- | ------------------------------------------------------------ | ----- | ------------------------------------------------------------ |
| r    | 有多少个进程分配到CPU，超过CPU数目，就会出现CPU瓶颈          | b     | 阻塞的进程                                                   |
| swpd | 虚拟内存已使用的大小，如果大于0，表示物理内存不足            | free  | 空闲的物理内存大小                                           |
| buff | 目录里面有内容，权限等的缓存                                 | cache | 把空闲的物理内存的一部分作为文件和目录的缓存<br/>用来提高程序执行性能 |
| si   | 每秒从磁盘读入虚拟内存的大小，如果大于0，表示物理内存不够，或者内存泄漏 | so    | 每秒虚拟内存写入磁盘的大小，作用同`si`                       |
| bi   | 快设备每秒接收的快数量，快设备是指系统上所有的磁盘和其它块设备，默认快大小是1024byte | bo    | 快设备每秒发送的快数量，例如读取文件，bo值就会大于0，大于0有可能是IO频繁 |
| in   | 每秒CPU的中断次数，包括时间中断                              | cs    | 每秒上下文切换次数，尽量小                                   |
| us   | 用户CPU时间                                                  | sy    | 系统CPU时间                                                  |
| id   | 空闲CPU时间，一般的，id+us+sy=100                            | wa    | 等待I/O CPU 时间                                             |

### 3、内存使用：free

free命令可以快速查看内存使用情况，它是对 ` proc/meminfo` 搜集到的信息的一个概述，如下：

```java

generic_x86:/ # free
                total        used        free      shared     buffers
Mem:       1567367168  1254526976   312840192     1060864     6221824
-/+ buffers/cache:     1248305152   319062016
Swap:               0           0           0

```

其中第二行（Mem）的used/ free，相对于操作系统而言的内存使用情况，而第三行（-/+ buffers/cache）则是应用程序角度的使用情况。

### 4、磁盘使用：df

df命令可以查看各个分区的磁盘使用情况。如下：

```java

generic_x86:/ # df
Filesystem                    Size  Used Avail Use% Mounted on
/dev/root                     2.4G  1.6G  762M  70% /
tmpfs                         747M  468K  747M   1% /dev
tmpfs                         747M     0  747M   0% /mnt
/dev/block/vde1                91M   39M   52M  44% /vendor
/dev/block/dm-1               775M  127M  648M  17% /data

```

### 5、网络状态：netstat

netstat 命令可以输出当前的网络连接状态，如下：

```java

1|generic_x86:/ # netstat
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State
tcp6       0      1 fec0::5931:8711:6:56252 2001::d065:3c57:http    SYN_SENT
tcp6       0      1 ::ffff:192.168.23:47174 tsa01s09-in-f4.1e:https SYN_SENT
    ....
Active UNIX domain sockets (w/o servers)
Proto RefCnt Flags       Type       State           I-Node Path
unix  78     [ ]         DGRAM                        6236 /dev/socket/logdw
unix  4      [ ]         DGRAM                        7829 /dev/socket/statsdw

```

常见参数有：

| 参数 | 含义                     | 参数 | 含义                               |
| ---- | ------------------------ | ---- | ---------------------------------- |
| -a   | 显示所有选项             | -t   | 仅仅显示TCP选项                    |
| -u   | 仅仅显示UDP选项          | -n   | 拒绝显示别名，能显示数字的都转数字 |
| -l   | 列出处于listen状态的服务 | -p   | 显示建立相关链接程序名             |
| -r   | 显示路由信息，路由表     | -e   | 显示扩展名，例如uid等              |
| -s   | 按各个协议进行统计       | -c   | 每个固定时间执行netstat命令        |



——Weiwq  于 2020.08 广州
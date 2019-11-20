---
layout:     post
title:      "详细解剖adb的命令"
subtitle:   " \"你知道adb怎么用吗\""
date:       2019-11-03 21:30:00
author:     "Weiwq"
header-img: "img/background/home-bg-geek.jpg"
catalog: true
tags:
    - Android
---

###  写在前面
>  开始想将标题设置为“深度解剖adb命令，后来犹豫了一下，“深度”，何为 “深度” ？ 如同“精通”一词，不敢随意挥写。但是本文会尽力列举adb的相关命令和说明，那就“详细”一词吧！

> 本文会比较长，各位读者可以挑选自己感兴趣的部分阅读即可。

## 1、adb官方介绍

Android 调试桥 (adb) 是一种功能多样的命令行工具，可让您与设备进行通信。adb 命令便于执行各种设备操作（例如安装和调试应用），并提供对 Unix shell（可用来在设备上运行各种命令）的访问权限。它是一种客户端-服务器程序，包括以下三个组件：

- **客户端**：用于发送命令。客户端在开发计算机上运行。您可以通过发出 adb 命令从命令行终端调用客户端。
- **守护进程 (adbd)**：在设备上运行命令。守护进程在每个设备上作为后台进程运行。
- **服务器**：管理客户端和守护进程之间的通信。服务器在开发计算机上作为后台进程运行。

本文不会列出如何搭建，连接adb的步骤，详细可以见 [在硬件设备上运行应用](https://developer.android.com/studio/run/device?hl=zh-cn)


## 2、adb 基本用法

```java
adb devices -l // 查看已经连接的设备详细信息
adb shell // 进入串口
adb kill-sever // 关闭adb
adb start-server //启动adb 
adb install helloWorld.apk //安装apk
adb connect $device_ip_address //通过网络连接设备IPs
adb pull remote local // 从设备上的文件（夹） 复制到本地 例如： adb pull /sdcard/foo.txt  foo.txt 
adb push local remote // 从本地复制文件（夹）到设备 例如： adb push foo.txt /sdcard/foo.txt
adb root // 获取root权限
adb unroot //	在不使用 root 权限的情况下重新启动 adbd。
adb remount // 挂载，配合adb root使用

```

比如查看已经连接的设备

```java

➜  ~ adb devices
List of devices attached
172.19.135.22:5555	device
emulator-5554	device

```

最新版本的adb是带有port的。如果adb连接多个设备，需要对某一个设备执行操作怎么办？

比如adb shell 操作，可以用 “-s” 指定设备

```java

➜  ~ adb devices
List of devices attached
172.19.135.22:5555	device
emulator-5554	device

➜  ~ adb -s 172.19.135.22:5555 shell
root@r34a5:/ #

```

或者 用“-t” 指定设备对应的transport_id，如下

```java

//通过adb devices -l获取设备的transport_id，如emulator-5554 的transport_id是8
➜  ~ adb devices -l
List of devices attached
172.19.135.22:5555     device product:r34a5 model:SMART_TV device:r34a5 transport_id:7
emulator-5554          device product:sdk_google_atv_x86 model:sdk_google_atv_x86 device:generic_x86 transport_id:8

➜  ~ adb -t 8 shell
generic_x86:/ $

```

## 3、adb 其他命令解析

| 命令                       | 说明       |
| :----------- | :-------------------- |
| adb install [option]  `<apkFile>` | `-r`：替换现有的应用。<br>`-d`：降版本安装。 <br>`-g`：授权所有运行时候需要的权限。<br>`-s`：在sdcard上安装。<br>`--abi abi-identifier`：针对特定 [ABI](https://developer.android.com/ndk/guides/abis?hl=zh-cn#sa) 强制安装应用。<br>`-t`：允许安装测试软件包。如果使用开发者预览版 SDK<br>（如果 `targetSdkVersion` 是字母，而非数字）编译 APK，则安装测试 APK 时必须在 install 命令中包含 `-t` 选项。 |
| adb uninstall [-k] `<packageName>` | 从设备中移除此应用软件包。添加 `-k` 选项可保存数据和缓存目录。 |
| adb usb | 重新启动USB上的adbd监听 |
| adb tcpip `<PORT>` | 重新启动adbd侦听PORT上的TCP |
| adb reboot [bootloader] [recovery]  [sideload] [sideload-auto-reboot] | 重新启动设备； 默认启动系统映像，但是也支持引导加载程序和恢复。 侧载重启进入恢复状态并自动启动侧载模式，sideload-auto-reboot相同，但是在侧载后重新启动。 |
| adb reconnect | 从主机端剔除连接以强制重新连接 |
| adb reconnect device | 从设备强制重新连接。 |
|adb  reconnect offline | 重置离线/未经授权的设备以强制重新连接 |
|adb connect  HOST[:PORT]| 通过网络连接设备，默认端口是5555 |
|adb disconnect [HOST[:PORT]]|断开指定的网络连接，不设置host，port，则断开全部|
|adb backup [option] ... |可以通过该命令备份apk或者系统信息，<br>其中options包含：<br><br>`-f <file>`：指定备份的位置，如-f  backup.ab<br><br/>`-all`: 备份全部<br><br/>`-apk|-noapk`:是否在备份里包含apk或者仅仅只备份应用数据，默认的是-noapk<br><br/>`-all`: 包含全部应用<br><br/>`-system|-nosystem`:决定-all标签是否包含系统应用，默认的是-system<br><br/>`[PACKAGE..]`: 指定备份的应用包名，如：adb backup com.demo.package<br><br/>`-shared|-noshared`:决定是否备份设备共享的SD card内容，默认是-noshare<br><br/> `-obb-noobb` : 是否包含obb文件，默认是-noobb|
|adb restore <file> |恢复应用数据，例如：adb restore backup.ab|
| adb jdwp |列出可以调试的apk进程|

## 4、shell 命令解析

可以使用如下进入设备

```java

adb [-d | -e | -s serial_number] shell

```

或者在不进入设备的情况下发出shell命令

```java

adb [-d | -e | -s serial_number] shell shell_command

```

如果需要退出远程shell命令，可以使用 Ctrl + D 或输入 exit。

shell 命令二进制文件存储在设备的文件系统中，其路径为 /system/bin/。

## 5、am 命令解析

在shell下，您可以使用 Activity 管理器 (am) 工具发出命令以执行各种系统操作，如启动某项 Activity、强行停止某个进程、广播 intent、修改设备屏幕属性等等。例如发送一个action

```java

am start -a  android.intent.action.VIEW

```
或者无需进入shell命令

```java

adb shell am start -a  android.intent.action.VIEW

```

其中**intent**规范如下

| 命令                                          | 说明                                                         |
| :-------------------------------------------- | :----------------------------------------------------------- |
| `-a <action>`                                 | 指定 intent 操作，例如 `android.intent.action.VIEW`（只能声明一次）。 |
| `-d <data_uri>`                               | 指定 intent 数据 URI，例如 `content://contacts/people/1`（只能声明一次）。 |
| `-t  <mime_type>`                             | 指定 intent MIME 类型，例如 `image/png`（只能声明一次）。    |
| `-c <category>`                               | 指定 intent 类别，例如 `android.intent.category.APP_CONTACTS`。 |
| `-n  <component>`                             | 指定带有软件包名称前缀的组件名称以创建显式 intent，例如 `com.example.app/.ExampleActivity`。 |
| `-f <flags>`                                  | 将标记添加到 `setFlags()` 支持的 intent。                    |
| `-e  | --es <extra_key> <extra_string_value>` | 将字符串数据作为键值对添加进来。                             |
| `--ez <extra_key> <extra_boolean_value>`      | 将布尔型数据作为键值对添加进来。                             |
| `-ei <extra_key> <extra_int_value>`         | 将整型数据作为键值对添加进来。                               |
| `--el <extra_key> <extra_long_value>`          | 将长整型数据作为键值对添加进来。                             |
| `--ef <extra_key> <extra_float_value>`        | 将浮点型数据作为键值对添加进来。                             |
| `--eu <extra_key> <extra_uri_value>`        | 将 URI 数据作为键值对添加进来。                              |
| --activity-single-top                         | 包含标记 `FLAG_ACTIVITY_SINGLE_TOP`。                        |
| --activity-clear-task                         | 包含标记 `FLAG_ACTIVITY_CLEAR_TASK`。                        |
| URI component package                         | 如果不受上述某一选项的限制，您可以直接指定 URI 或软件包名称 或者组件名称。当某个参数不受限制时，如果该参数包含一个“:”（冒号），则该工具会假定参数是一个 URI；如果该参数包含一个“/”（正斜线），则该工具会假定参数是一个组件名称；如果并非这两种情况，则该工具会假定参数是一个软件包名称。 |

**am的命令如下**

| 命令                                                         | 说明                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `start [options] <intent>`                                   | 启动 `intent` 指定的 `Activity`。 请参阅intent规范<br>其中***options***包括<br><br/> `-D`：启用调试。<br><br/>`-N`: 打开native debug<br><br>`-W`: wait for launch to complete<br><br>`--start-profiler <FILE>`：启动分析器并将结果发送到 `file`。<br><br/>`-P <FILE>`：类似于 `--start-profiler`，但当应用进入空闲状态时分析停止。<br><br/>`-R <COUNT>`：重复启动 Activity `count` 次。在每次重复前，将完成顶层 Activity。<br><br/>`-S`：启动 Activity 前强行停止目标应用。<br><br/>`--opengl-trace`：启用对 OpenGL 函数的跟踪。<br><br> `--track-allocation`: 启用对象分配的跟踪 |
| `startservice <intent>`                                      | 启动 `intent` 指定的 `Service`。请参阅intent规范             |
| `stopservice <intent>`                                       | 停止intent指定的service。请参阅intent规范                    |
| `force-stop <package>`                                       | 强行停止与 `package`（应用的软件包名称）关联的所有进程。     |
| `kill-all`                                                   | 终止所有后台进程                                             |
| `broadcast [options] <intent>`                               | 发出广播 intent。请参阅intent规范                            |
| `instrument [options] <component>`                           | 使用 `Instrumentation` 实例启动监控。通常情况下，目标 `component` 是 `test_package/runner_class` 格式。<br>具体options选项包括：<br><br/>`-r`：输出原始结果（否则对 `report_key_streamresult` 进行解码）。与 `[-e perf true]` 结合使用以生成性能测量值的原始输出。<br><br/>`-e <NAME> <VALUE>`：将参数 `name` 设为 `value`。对于测试运行器，通用格式为 `-e testrunner_flag value[,value...]`。<br><br/>`-p <FILE>`：将分析数据写入 FILE。<br><br/>`-w`：先等待插桩完成，然后再返回。测试运行器需要使用此选项。 |
| `profile start process <file>`                               | 启动 `process` 的分析器，将结果写入 `file`。                 |
| `profile stop process`                                       | 停止 `process` 的分析器。                                    |
| `dumpheap [options] process  <file>`                         | 转储 `process` 的堆，写入 `file`。<br>具体options选项包括：<br>`-n`：转储原生堆，而非托管堆。<br>`--user [user_id | current] ` :提供进程名称时，指定要转储的进程的用户；如果未指定，则使用当前用户。 |
| `monitor [options]`                                          | 开始监控崩溃或 ANR。<br>具体options选项包括：<br>`--gdb`：在崩溃/ANR 时在指定端口上启动 gdbserv。 |
| `screen-compat [on | off ]  <package>`                       | 控制 `package` 的[屏幕兼容性](https://developer.android.com/guide/practices/screen-compat-mode.html?hl=zh-cn)模式。 |
| `set-debug-app [-w] [--persistent] <PACKAGE>`                | 将<PACKAGE>设置为debug模式。<br/>具体options选项包括：<br/>-w: 当应用起来的时候，等待调试<br>--persistent: 保留此值 |
| `clear-debug-app`                                            | 清除掉之前设置的debug应用包名                                |
| `display-size [reset | <width>x<height>]`                    | 替换设备显示尺寸。此命令支持使用大屏设备模仿小屏幕分辨率（反之亦然），对于在不同尺寸的屏幕上测试应用非常有用。示例:<br>` am display-size 1280x800` |
| `display-density <dpi>`                                      | 替换设备显示密度。此命令支持使用低密度屏幕在高密度屏幕环境上进行测试（反之亦然），对于在不同密度的屏幕上测试应用非常有用。示例：<br/>`am display-density 480` |
| `crash <PACKAGE | PID>`                                      | 在指定的程序包或进程中导致VM崩溃                             |
| `monitor [--gdb <port>]`                                     | 开始监视崩溃或ANR。<br>--gdb：在崩溃/ ANR的给定端口上启动gdbserv |
| `restart`                                                    | 重新启动用户空间系统                                         |
| `to-uri [INTENT]`                                            | 将指定的intent规范打印为uri                                  |
| `stack [COMMAND] [...]`                                      | 用于在活动堆栈上运行的子命令。<br>具体command选项如下：<br>`list`:列举所有activity的stack和他们的大小 <br><br>`resize <STACK_ID> <LEFT,TOP,RIGHT,BOTTOM>`:改变 STACK_ID的大小到<LEFT,TOP,RIGHT,BOTTOM>，例如：am stack  resize 1 0 0 500 500 // STACK_ID可以通过`am stack list` 获取<br><br>`remove <STACK_ID>`:  从stack list中删除 <STACK_ID>. |
| `task [COMMAND] [...]`                                       | 用于操作activity task的子命令。<br>具体command如下：<br><br>`lock <TASK_ID>` : 将TASK_ID置为前台，并且不让其他stask工作（转为前台）,即锁上TASK_ID<br><br>`lock stop` : 将当前的task 锁解锁 <br><br>`resizeable <TASK_ID> [0|1|2|3]`: 改变TASK_ID大小可调模式，模式对应如下：<br>0: 不可调节大小。1:裁剪window。2:大小可调。3:大小可调并且可移植。<br><br>`resize <TASK_ID> <LEFT,TOP,RIGHT,BOTTOM>` :确保<TASK_ID>在具有指定范围的堆栈中。强制将TASK_ID设置为大小可调，并且如果在现有stasks中不具有指定范围，会创建一个stask <br> |
| update-appinfo `<USER_ID>` `<PACKAGE_NAME>` [<PACKAGE_NAME>...] | 为USER_ID更新列举包名的应用信息对象，不需要重新启动包名对应的任何进程 |

**备注**

- USER_ID：可以通过/data/system/users/userlist.xml获取 ，当手机使用者(即User)下载你(即开发者)的应用程序，在安装(Install)时，Android就会给予一个UID。
- STACK_ID，TASK_ID：可以通过`am stack list`命令获取

## 6、pm 命令解析

跟am命令一样，需要在shell环境下执行

```java

➜  ~ adb shell
generic_x86:/ # pm uninstall com.example.MyApp //卸载apk

```

pm详细命令如下：

| 命令                                     | 说明                                                         |
| :---------------------------------------- | :------------------------------------------------------------ |
| list packages [**options**] `<filter>` | 输出所有软件包，或者，仅输出软件包名称包含 `filter` 中的文本的软件包。<br>具体**options**选项：<br>`-f`：查看它们的关联文件。<br>`-d`：进行过滤以仅显示已停用的软件包。<br>`-e`：进行过滤以仅显示已启用的软件包。<br>`-s`：进行过滤以仅显示系统软件包。<br>`-3`：进行过滤以仅显示第三方软件包。<br>`-i`：查看软件包的安装程序。<br>`-u`：还包括卸载的软件包。<br>`--user user_id`：要查询的用户空间。<br>示例：<br>`pm list packages -3 demo` // 列举包名包含“demo”的第三方应用 |
| list permission-groups                   | 输出所有已知的权限组。                                       |
| list permissions [**options**] `<group>` | 输出所有已知的权限，或者，仅输出 `group` 中的权限。<br>具体options选项：<br>`-g`：按组进行整理。<br>`-f`：输出所有信息。<br>`-s`：简短摘要。<br>`-d`：仅列出危险权限。<br>`-u`：仅列出用户将看到的权限。<br>示例：<br>`pm list permissions -d` // 列举危险权限 |
| list features                            | 列举系统所有功能                                             |
| list libraries                           | 输出当前设备支持的所有库。                                   |
| path `<package>`                     | 输出指定 `package` 的 APK 的路径。<br>示例：<br>pm path com.demo.mydemo |
| install [**options**] `<apkPath>`   | 将软件包（通过 `path` 指定）安装到系统。<br>具体options选项参考`adb install`命令 |
| uninstall [**options**] `<package>`  | 从系统中移除软件包。具体选项：<br>`-k`：移除软件包后保留数据和缓存目录。 |
| clear `<package>`                    | 删除与软件包关联的所有数据。                                 |
| enable `<package_or_component>`      | 启用指定软件包或组件（表示为“package/class”，需要root权限）。示例：<br>`pm enable com.cvte.demotv/com.cvte.demotv.MainActivity` |
| disable `<package_or_component>`     | 停用指定软件包或组件（表示为“package/class”，需要root权限）。示例：<br>`pm disable com.cvte.demotv/com.cvte.demotv.MainActivity` |
| grant `<package_name>` `<permission>` | 向应用授予权限。在搭载 Android 6.0（API 级别 23）及更高版本的设备上，该权限可以是应用清单中声明的任何权限。在搭载 Android 5.1（API 级别 22）及更低版本的设备上，该权限必须是应用定义的可选权限。 |
| revoke `<package_name>` **permission** | 从应用中撤消权限。在搭载 Android 6.0（API 级别 23）及更高版本的设备上，该权限可以是应用清单中声明的任何权限。在搭载 Android 5.1（API 级别 22）及更低版本的设备上，该权限必须是应用定义的可选权限。 |
| trim-caches `<desired_free_space>`   | 减少缓存文件以达到指定的可用空间。                           |
|dump `<PACKAGE>`|打印与指定包名相关的应用信息|
|resolve-activity [--brief] [--components] [--user USER_ID] `<INTENT>`|通过指定的INTENT 打印对应的activity信息|
|query-activities [--brief] [--components] [--user USER_ID] `<INTENT>`|打印所有能处理INTENT 的activities信息|
|query-services [--brief] [--components] [--user USER_ID] `<INTENT>`|打印所有能处理INTENT 的service信息|
|query-receivers [--brief] [--components] [--user USER_ID] `<INTENT>`|打印所有能处理INTENT 的广播接收者信息|
|hide/unhide [--user USER_ID] `<PACKAGE_OR_COMPONENT>`|隐藏/显示  指定的包名（组件名）|
|reset-permissions|重置所有运行时的权限到默认状态|
|set-permission-enforced `<PERMISSION>` `[true | false]`|设置权限是否强制执行|
|uninstall-system-updates|删除系统应用的更新，并回滚应用版本|
|set-home-activity [--user USER_ID] `<TARGET-COMPONENT>`|设置home的activity|
|dump-profiles `<TARGET-PACKAGE>`|存储 method/class 信息到：<br/>    /data/misc/profman/TARGET-PACKAGE.txt|
|snapshot-profile `<TARGET-PACKAGE>` [--code-path path]|将指定包名的性能快照文件保存到：<br/>    /data/misc/profman/TARGET-PACKAGE[-code-path].prof<br/>如果 TARGET-PACKAGE=android，系统将保存boot 镜像的快照。|

## 7、dpm 命令解析

`dmp`	即调用设备政策管理器，为便于您开发和测试设备管理（或其他企业）应用，您可以向设备政策管理器 (`dpm`) 工具发出命令。使用该工具可控制 Active Admin 应用，或更改设备上的政策状态数据。在 shell 中，相应的语法为：

```java

dmp command

```

或者直接从 adb 发出设备政策管理器命令，无需进入远程 shell：

```java

adb shell dmp command

```

dmp详细命令如下：

| 命令                                            | 说明                                                         |
| :---------------------------------------------- | :----------------------------------------------------------- |
| set-active-admin [**options**] `<component>`    | 将组件设置为 Active Admin。具体options选项包括：<br>`--user user_id`：指定目标用户。您也可以传递 `--user current` 以选择当前用户。 |
| set-profile-owner [**options**] `<component>`   | 将组件设置为 Active Admin，并将其软件包设置为现有用户的配置文件所有者。具体options选项包括：<br>`--user user_id`：指定目标用户。您也可以传递 `--user current` 以选择当前用户。<br>`--name name`：指定简单易懂的组织名称。 |
| set-device-owner [**options**] `<component>`    | 将组件设置为 Active Admin，并将其软件包设置为设备所有者。具体options选项包括：<br>`--user user_id`：指定目标用户。您也可以传递 `--user current` 以选择当前用户。<br>`--name name`：指定简单易懂的组织名称。 |
| remove-active-admin [**options**] `<component>` | 停用 Active Admin。应用必须在清单中声明 `android:testOnly`。该命令还会移除设备所有者和配置文件所有者。具体选项包括：<br>`--user user_id`：指定目标用户。您也可以传递 `--user current` 以选择当前用户。 |

## 8、其他命令解析

### 1）截图

screencap` 命令是一个用于对设备显示屏进行屏幕截图的 shell 实用程序。相应的语法为

```java

screencap /sdcard/screen.png // 在shell使用
adb shell screencap /sdcard/screen.png // 通过adb使用

```

可以使用pull命令将截图文件拉到本地

### 2）录制视频

`screenrecord` 命令是一个用于录制设备（搭载 Android 4.4（API 级别 19）及更高版本）显示屏的 shell 实用程序。该实用程序将屏幕 Activity 录制为 MPEG-4 文件。您可以使用此文件创建宣传视频或培训视频，或将其用于调试或测试。

```java

screenrecord /sdcard/demo.mp4 // shell 环境下的命令
adb shell screenrecord /sdcard/demo.mp4 //adb命令

```

按 Ctrl + C（在 Mac 上为 Command+C）停止屏幕录制，否则，到三分钟或 `--time-limit` 设置的时间限制时，录制将自动停止。

同理，可以通过pull命令拉取文件

`screenrecord` 实用程序的局限性：

- 音频不与视频文件一起录制。
- 无法在搭载 Wear OS 的设备上录制视频。
- 某些设备可能无法以它们的本机显示分辨率进行录制。如果在录制屏幕时遇到问题，请尝试使用较低的屏幕分辨率。
- 不支持在录制时旋转屏幕。如果在录制期间屏幕发生了旋转，则部分屏幕内容在录制时将被切断。

screenrecord命令解析

| 命令                        | 说明                                                         |
| :--------------------------- | :------------------------------------------------------------ |
| --size `<width>x<height>` | 设置视频大小：`1280x720`。默认值是设备的本机显示分辨率（如果支持）；如果不支持，则使用 1280x720。为获得最佳效果，请使用设备的 Advanced Video Coding (AVC) 编码器支持的大小。 |
| --bit-rate `<rate>`     | 设置视频的视频比特率（以 MB/秒为单位）。默认值为 4Mbps。您可以增加比特率以提升视频品质，但这么做会导致视频文件变大。下面的示例将录制比特率设为 6Mbps：<br>`screenrecord --bit-rate 6000000 /sdcard/demo.mp4` |
| --time-limit `<time>`   | 置最大录制时长（以秒为单位）。默认值和最大值均为 180（3 分钟）。 |
| --rotate                    | 将输出旋转 90 度。此功能处于实验阶段。                       |
| --verbose                   | 在命令行屏幕显示日志信息。如果您不设置此选项，则该实用程序在运行时不会显示任何信息。 |

### 3)获取其他shell命令

可以通过以下命令获取当前系统支持的shell命令

```java

adb shell ls ls system/bin

```

其他常用shell命令列举

| shell命令                            | 说明                                                         |
| :------------------------------------ | :------------------------------------------------------------ |
| dumpsys                              | 将系统数据转储到屏幕。要详细了解此命令行工具，请阅读 [dumpsys](https://developer.android.com/studio/command-line/dumpsys.html?hl=zh-cn) |
| dumpstate                            | 将状态转储到文件。                                           |
| logcat [option]...  [filter-spec]... | 启用系统和应用日志记录，并将输出显示到屏幕上。另请参阅 [Logcat 命令行工具](https://developer.android.com/studio/command-line/logcat.html?hl=zh-cn)。 |
| dmesg                                | 将内核调试消息输出到屏幕。                                   |
| start                                | 启动（重启）设备。                                           |
| stop                                 | 停止执行设备。                                               |
| sqlite3                              | 启动 `sqlite3` 命令行程序。`sqlite3` 工具包含用于输出表格内容的 `.dump` 以及用于输出现有表格的 SQL CREATE 语句的 `.schema` 等命令。您也可以随时执行 SQLite 命令。SQLite3 数据库存储在文件夹 `/data/data/package_name/databases/` 中。例如：<br>adb -s emulator-5554 shell<br>sqlite3 /data/data/com.example.app/databases/rssitems.db<br>SQLite version 3.3.12<br>Enter ".help" for instructions |



# 9、后记

读者看到最后，心中不免会犯嘀咕——标题不是adb的命令解析吗，怎么会牵扯到am，pm甚至dpm这个听都没有听过的工具？

犯嘀咕是正常的，adb这个工具确实需要我们Android开发者好好学习学习，其本身提供了一些强大的命令，但是更多的命令工具是放在系统system/bin下的，那如何使用系统中的命令工具呢？笔者个人觉得adb shell更像是提供了与系统 交互的窗口，我们可以通过这个窗口，去执行系统中的命令，去让我们的系统更加完美（在我们自己的角度来看）。

—— Weiwq 于 2019.11 广州


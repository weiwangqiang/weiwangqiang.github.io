---
layout:     post
title:      "android调试——教你用aapt命令分析apk内容"
subtitle:   " \"aapt--你可能不知道的命令\""
date:       2020-03-20 13:30:00
author:     "Weiwq"
header-img: "img/background/post-bg-re-vs-ng2.jpg"
catalog: true
tags:
    - Android
---

> “为了查看apk信息，你还在反编译吗？“

### aapt

aapt 是Android sdk提供的一个apk分析工具，其路径是在 

```java

${androidSdk}/sdk/build-tools/${sdk版本}/aapt

```

通过这个工具，你可以看到 apk的包名，版本，权限，资源等

- **aapt d badging xxx/xx.apk**

 获取对应apk的信息，如下，该命令基本上就能获取到我们想要的apk信息

  ```java
  
  ➜ aapt d badging /D/demo.apk
  package: name='com.demo.package' versionCode='19' versionName='2.4.0.4'
  sdkVersion:'19'
  targetSdkVersion:'22'
  uses-permission: name='android.permission.INSTALL_PACKAGES'  // apk 权限
  ......
  // apk 名称 和图标
  application: label='xx助手' icon='res/mipmap-mdpi-v4/ic_launcher.png'
  // 主入口activity
  launchable-activity: name='com.demo.package.LauncherActivity'  label='' icon=''
  // 其他的组件
  main
  other-activities
  .....
  other-receivers
  ......
  other-services
  .....
  // 支持的分辨率
  supports-screens: 'small' 'normal' 'large' 'xlarge'
  // 支持的架构
  native-code: 'armeabi'
   
  ```

- **aapt d string /xxx/xxx.apk**

  获取apk全部的string资源

  ```java
  ~ aapt d string /xxx/xxx.apk
  String #5212: Repeat None
  String #5213: Repeat One
  ```

- **aapt d permissions /xxx/xxx.apk**

  获取apk 声明的权限

  ```java
  ~ aapt d permissions /xxx/xxx.apk
  package: com.package.demo
  uses-permission: name='android.permission.INSTALL_PACKAGES' .....
  ```
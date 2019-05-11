---
layout:     post
title:      "把Google的flutter demo跑起来"
subtitle:   " \"是时候了解一下flutter了\""
date:       2019-05-11 15:10:00
author:     "Weiwq"
header-img: "img/background/post-bg-re-vs-ng2.jpg"
catalog: true
tags:
    - flutter
---

> “其实我并不喜欢追求新技术。flutter是Google出的？真香～“


## 引言

其实跨平台的痛，我真的没有体会到，毕竟我司不做ios平台。但是如果，flutter有可能成为新系统的开发框架，还是值得学习一下的，尤其是看了官方的demo。

![](https://camo.githubusercontent.com/23d3c78b0a2b645567630468bd68d54c02c2076a/68747470733a2f2f63646e2e3264696d656e73696f6e732e636f6d2f315f53746172742e676966)

我们将会搭建flutter开发环境，来跑这个demo。

## 1、 开发环境搭建

其实有点恶心新环境的搭建，意味着，午休时间是没有了的。好在flutter提供了完整的搭建教程，
见[flutter环境搭建](https://flutterchina.club/get-started/install/)

如果只是体验一下flutter，建议只下载flutter sdk ，和对应的平台插件即可。

本篇文章将以Android studio为开发环境，ios的请绕步。

1）下载[flutter sdk](https://flutter.dev/docs/development/tools/sdk/releases?tab=windows#macos) 
请下载最新的sdk，Google的demo要求sdk至少为1.2。解压到指定目录。

将flutter加载到配置环境中，查看flutter版本

```java
flutter --version
```

注意：这里flutter sdk内置dart ，不需要我们再下载。

2）配置Android studio 环境

到插件市场，下载dart ，flutter插件，需要科学上网哦！如图：

![在这里插入图片描述](/img/blog_flutter_google_demo/1.jpg)

3）安装完，重新启动as，然后配置flutter的sdk路径

![在这里插入图片描述](/img/blog_flutter_google_demo/2.jpg)

以及dart sdk

![在这里插入图片描述](/img/blog_flutter_google_demo/8.jpg)

点击apply 和 ok即可。如果提示失败，请到flutter sdk目录下，使用git init一下

这样，我们flutter开发环境就搭建好了，需要new flutter工程，点击file -->> new ->> new Flutter project即可，如果没有改选项，请确保以下是勾选的状态

![在这里插入图片描述](/img/blog_flutter_google_demo/3.jpg)
  
## 2、导入flutter项目

 [项目地址](https://github.com/weiwangqiang/HistoryOfEverything)
 
 用Android studio clone到本地，会发现一大筐红线，别怕，一步一步来。
 
 1） 先看项目的结构：

![在这里插入图片描述](/img/blog_flutter_google_demo/4.jpg)


 app目录下赫然显示Android，ios。想必大家都知道其中的意思了，是的，这里存放两个平台编译后的资源。这也是flutter为什么能跨平台的原因。这里说flutter跨平台，其实不太严谨，准确的说是，由flutter框架，编译出两个平台的资源，从而实现一处编写，双平台运行。
 
 与Android项目不同的是，flutter需要指定main入口，不然，编译器不知道从哪里开始编译，这有点像eclipse开发Android项目一样。这里我们可以看到，运行的按钮是灰色的。  
 
2） 先确保该项目的flutter sdk已经配置，详情见**配置环境** 

进入configuration配置

![在这里插入图片描述](/img/blog_flutter_google_demo/5.jpg)

点击左上角的“+”，选择flutter

![在这里插入图片描述](/img/blog_flutter_google_demo/6.jpg)

接着配置dart的main函数

![在这里插入图片描述](/img/blog_flutter_google_demo/7.jpg)

然后我们的run button就可以点击了

接着打开Android模拟器，点击run图标即可。当然，如果搭建了ios的开发平台，也可以在ios模拟器上直接运行。

### 2.1 编译失败

1）如果出现gradle编译失败，请到android 目录下修改build.gradle的版本，这里使用的是

```java
    dependencies {
        classpath 'com.android.tools.build:gradle:3.2.1'
    }
```

在gradle/wrapper/gradle-wrapper.properties中，对应的URL是

```java
distributionUrl=https\://services.gradle.org/distributions/gradle-4.6-all.zip
```

2） 如果出现类型转换错误
将

```java
Int32List _indices;
....
_indices = new Int32List.fromList(triangles); 
........

```

修改为

```java
Uint16List _indices;
.....
_indices = new Uint16List.fromList(triangles);
......
```

## 3、打包编译资源

关于配置签名，buildType的方法，到Android目录下，就一目了然了。这与我们在编译正常的Android apk 使用时完全一样的

```java
signingConfigs {
    release {
        keyAlias keystoreProperties['keyAlias']
        keyPassword keystoreProperties['keyPassword']
        storeFile file(keystoreProperties['storeFile'])
        storePassword keystoreProperties['storePassword']
    }
}
buildTypes {
    release {
        signingConfig signingConfigs.release
    }
}
```

如何打包呢？在工程目录下执行

```java
flutter build apk
```

构建完成，在如下目录就可以找到对应的apk了

![在这里插入图片描述](/img/blog_flutter_google_demo/9.jpg)

## 4、番外篇

在项目的dependencies目录下有两个特别有意思的动画demo，先上图

![在这里插入图片描述](/img/blog_flutter_google_demo/9.gif)

入口函数在这里，有兴趣的同学可以看看，修改一下入口函数就可以运行了

![在这里插入图片描述](/img/blog_flutter_google_demo/11.jpg)
     
 ## 后记

简单把玩了一下flutter，给我个人的感受就是，flutter是一款跨平台的ui框架，其编译热更新确实很厉害，尤其是在大项目中很有用（编译的痛，大家都懂）。

与Android原生java+xml的框架不同的是，flutter是纯dart语音编写，即我们不能在flutter中任性的拖拖拖了，所有的ui需要用代码去实现，里面各种包裹，里三层，外三层，真心很迷。


—— Weiwq 后记于 2019.05 广州



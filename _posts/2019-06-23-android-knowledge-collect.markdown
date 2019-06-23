---
layout:     post
title:      "android知识大杂汇"
subtitle:   " \"这里汇聚各路英雄好汉有关Android的文章\""
date:       2019-06-23 16:10:00
author:     "Weiwq"
header-img: "img/background/post-bg-re-vs-ng2.jpg"
catalog: true
tags:
    - android
---

> “开发到一定的阶段，有些问题不是API代码就能够解决的，还需要硬件相关的知识“


## 引言
开发android也有一段时间了，感觉焦点不能再局限上层应用，需要从Android整体框架入手，全面掌握Android的设计艺术，这里特意整理了相关的知识

# 一、Android 相关

## 1、卡顿分析

- **[Android线程死锁监控与自动化分析实践](https://cloud.tencent.com/developer/article/1064396)**
- **[Java线程Dump分析](https://juejin.im/post/5b31b510e51d4558a426f7e9)**


## 2、网络优化

 - **[聊聊Linux 五种IO模型](https://www.jianshu.com/p/486b0965c296)**
 - **[微信网络请求框架mars](https://github.com/Tencent/mars)**
 - **[微信跨业务基础组件mars](https://github.com/Tencent/mars/wiki)**
- **[携程 App 的网络性能优化实践](https://www.infoq.cn/article/how-ctrip-improves-app-networking-performance)**
- **[阿里无线 11.11：手机淘宝移动端接入网关基础架构演进之路](https://www.infoq.cn/article/taobao-mobile-terminal-access-gateway-infrastructure)**
- **[蚂蚁金服亿级并发下的移动端到端网络接入架构解析](https://mp.weixin.qq.com/s/nz8Z3Uj9840KHluWjwyelw)**
- **[百度App网络深度优化系列《一》DNS优化](https://mp.weixin.qq.com/s/iaPtSF-twWz-AN66UJUBDg)**
- **[HTTP/2 头部压缩技术介绍](https://imququ.com/post/header-compression-in-http2.html)**
- **[腾讯社交网络图片带宽优化技术演进之路](https://mp.weixin.qq.com/s/JcBNT2aKTmLXRD9zIOPe6g)**
- **[
看得「深」、看得「清」—— 深度学习在图像超清化的应用](http://imgtec.eetrend.com/d6-imgtec/blog/2017-08/10143.html)**
- **[TLS协议分析 与 现代加密通信协议设计](https://blog.helong.info/blog/2015/09/07/tls-protocol-analysis-and-crypto-protocol-design/)**
- **[TLS1.3](https://zhuanlan.zhihu.com/p/44980381)**
- **[基于TLS1.3的微信安全通信协议mmtls介绍](https://mp.weixin.qq.com/s/tvngTp6NoTZ15Yc206v8fQ)**
- **[Facebook是如何大幅提升TLS连接效率的？](https://mp.weixin.qq.com/s?__biz=MzI4MTY5NTk4Ng==&mid=2247489465&idx=1&sn=a54e3fe78fc559458fa47104845e764b&source=41#wechat_redirect)**
- **[小米安全：证书锁定](https://sec.xiaomi.com/article/48)**
- **[CDN + P2P 在大规模直播 & 实时直播的技术实践](https://toutiao.io/posts/6gb8ih/preview)**
- **[P2P如何将视频直播带宽降低75%？](https://mp.weixin.qq.com/s?__biz=MzI4MTY5NTk4Ng==&mid=2247489182&idx=1&sn=e892855fd315ed2f1395f05b765f9c4e&source=41#wechat_redirect)**
- **[QUIC协议在腾讯的实践和优化.PDF](https://archstat.com/infoQ/archSummit/2018%E6%9E%B6%E6%9E%84%E5%B8%88%E5%90%88%E9%9B%86/AS%E6%B7%B1%E5%9C%B32018-%E3%80%8AQUIC%E5%8D%8F%E8%AE%AE%E5%9C%A8%E8%85%BE%E8%AE%AF%E7%9A%84%E5%AE%9E%E8%B7%B5%E5%92%8C%E4%BC%98%E5%8C%96%E3%80%8B-%E7%BD%97%E6%88%90.pdf)**
- **[QUIC在手机微博中的应用实践.pdf](https://github.com/thinkpiggy/qcon2018ppt/blob/master/QUIC%E5%9C%A8%E6%89%8B%E6%9C%BA%E5%BE%AE%E5%8D%9A%E4%B8%AD%E7%9A%84%E5%BA%94%E7%94%A8%E5%AE%9E%E8%B7%B5.pdf)**
- **[手机淘宝移动端接入网关基础架构演进之路](https://mp.weixin.qq.com/s/QhaFKuxTf3mrbF-eWIkZTw)**
- **[360开源又一力作——ArgusAPM移动性能监控平台](https://github.com/Qihoo360/ArgusAPM)**
- **[面向切面（AspectJ）编程](http://www.shouce.ren/api/spring2.5/ch06s02.html)**



## 3、Ui适配

- **[头条：反射实现的Android屏幕适配](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&mid=2247484502&idx=2&sn=a60ea223de4171dd2022bc2c71e09351&scene=21#wechat_redirect)**
- **[OLED和LCD的区别](https://www.zhihu.com/question/22263252)**
- **[Android 目前稳定高效的UI适配方案](https://www.jianshu.com/p/a4b8e4c5d9b0)**
- **[SmallestWidth 限定符适配方案](https://juejin.im/post/5ba197e46fb9a05d0b142c62)**
- **[Vulkan:低开销，高性能3D图形API](https://source.android.com/devices/graphics/arch-vulkan)**
- **[Android图形整体架构](https://source.android.com/devices/graphics)**
- **[BufferQueue:图形处理核心](https://source.android.com/devices/graphics/arch-bq-gralloc)**
- **[VSYNC:android 垂直同步概念](https://source.android.com/devices/graphics/implement-vsync)**
- **[Android Project Butter（黄油计划）分析](https://blog.csdn.net/innost/article/details/8272867)**
- **[检查GPU渲染速度和绘制过度](https://developer.android.com/studio/profile/inspect-gpu-rendering)**
- **[一颗像素的诞生](https://mp.weixin.qq.com/s/QoFrdmxdRJG5ETQp5Ua3-A)**
- **ui测试问题定位工具[graphics API Debugger](https://github.com/google/gapid)**
- **gfxinfo可以拿到包各个阶段动画以及帧相关信息，命令如下**

```java
adb shell dumpsys gfxinfo packageName

//以下命令可以拿到近120帧的每帧耗时
adb shell dumpsys gfxinfo 包名 framestats 
// 通过以下命令获取到SurfaceFlinger
adb shell dumpsys SurfaceFlinger

```

- **GPU分析工具[Profile GPU Rendering](https://developer.android.com/topic/performance/rendering/profile-gpu)**
- **[硬件加速](https://developer.android.com/guide/topics/graphics/hardware-accel#drawing-support)**
- **[(推荐)facebook减少ui层级开源框架：litho](https://github.com/facebook/litho)**
- **[美团：Litho的基本使用](https://tech.meituan.com/2019/03/14/litho-use-and-principle-analysis.html)**


# 4、系统源码

- **[深入Android源码系列（一）](https://mp.weixin.qq.com/s/VSVUbaEIfrmFZMB1k49fyA)**


# 二、数据库


## 1、数据库原理
- **[数据库索引原理](https://www.cnblogs.com/huahuahu/p/sqlite-suo-yin-de-yuan-li-ji-ying-yong.html)**
- **[数据库索引原理官方文档](https://www.sqlite.org/queryplanner.html#searching)**
- **[MySQL索引背后的数据结构及算法原理](http://blog.codinglabs.org/articles/theory-of-mysql-index.html)**


## 2、数据持久化方案

- **[sharePreferences解析](https://juejin.im/entry/597446ed6fb9a06bac5bc630)**
- **[微信MMKV的存储方案](https://github.com/Tencent/MMKV)**
- **[Twitter序列化方案Serial](https://github.com/twitter/Serial/blob/master/README-CHINESE.rst/)**
- **[微信WCDB数据库框架](https://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=2649286603&idx=1&sn=d243dd27f2c6614631241cd00570e853&chksm=8334c349b4434a5fd81809d656bfad6072f075d098cb5663a85823e94fc2363edd28758ab882&mpshare=1&scene=1&srcid=0609GLAeaGGmI4zCHTc2U9ZX#rd)**
- **[Android WCDB ORM框架](https://github.com/Tencent/wcdb/wiki/Android-WCDB-%E4%BD%BF%E7%94%A8-Room-ORM-%E4%B8%8E%E6%95%B0%E6%8D%AE%E7%BB%91%E5%AE%9A)**


## 3、数据库修复
- **[How To Corrupt An SQLite Database File](https://sqlite.org/howtocorrupt.html)**
- **[微信 SQLite 数据库修复实践](https://mp.weixin.qq.com/s/N1tuHTyg3xVfbaSd4du-tw)**
- **[微信移动端数据库组件WCDB系列（二） — 数据库修复三板斧](https://mp.weixin.qq.com/s/Ln7kNOn3zx589ACmn5ESQA)**
- **[WCDB Android 数据库修复](https://github.com/Tencent/wcdb/wiki/Android%E6%95%B0%E6%8D%AE%E5%BA%93%E4%BF%AE%E5%A4%8D)**

## 4、数据库优化

- **[微信全文搜索优化之路](https://mp.weixin.qq.com/s/AhYECT3HVyn1ikB0YQ-UVg)**
- **[移动客户端多音字搜索](https://mp.weixin.qq.com/s/GCznwCtjJ2XUszyMcbNz8Q)**
- **[SQLite FTS3 and FTS4 Extensions](https://sqlite.org/fts3.html)**
- **[SQL解析在美团的应用](https://tech.meituan.com/2018/05/20/sql-parser-used-in-mtdp.html)**
- **[美团点评SQL优化工具SQLAdvisor开源](https://tech.meituan.com/2017/03/09/sqladvisor-pr.html)**

## 5、SQLite进阶

- **[SQLite源码分析](http://huili.github.io/sqlite/sqliteintro.html)**

## 6、文件压缩方案
- **[google跨语言编码协议（提高压缩率）： protocol buffers](https://developers.google.com/protocol-buffers/docs/overview)**
- **[google压缩率更高的：FlatBuffer](https://www.race604.com/flatbuffers-intro/)**
- **[FileVisitor:文件遍历](https://developer.android.com/reference/java/nio/file/FileVisitor)**


## 三、Java相关

- **[字节码操纵技术探秘](https://www.infoq.cn/article/Living-Matrix-Bytecode-Manipulation)**
- **[ASM-操纵Java字节码的框架](https://asm.ow2.io/)**

# 后记

望君学有所成！

—— Weiwq 后记于 2019.06 广州



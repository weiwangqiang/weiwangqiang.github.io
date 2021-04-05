---
layout:     post
title:      "抓包教程详解"
subtitle:   " \"抓包很难？那是你没找对方法\""
date:       2021-04-05 23:30:00
author:     "Weiwq"
header-img: "img/background/post-bg-alitrip.jpg"
catalog: true
tags:
    - Network

---

> “这篇就聊聊如何抓包“

# 引言

最近在复习网络相关的内容，顺便就整理一下之前的抓包技巧，供大家参考。

# 1、浏览器抓包

## 开发者工具

我们先看一个最简单的抓包工具——浏览器。大多数浏览器都会提供开发者入口。以chrome浏览器为例。

在右上角有一个菜单入口，点击，找到对应的开发者工具（或者快捷键Ctrl+shift+i）

<img src="/img/blog_network_capture/1.png" width="50%" height="40%"> 

接着就会出现如下页面，该页面分为两个区域，这里用红框标注。其中上面的为网页元素，日志控制台，网络，资源，性能和内存等维度的tab；下面的就是对应维度的内容。

对于网络来说，里面有如下列：

- name: 请求的资源路径。
- status：请求结果返回的状态码
- type：请求的资源类型。
- initiator: 请求发起来源。
- size：资源大小。
- time：请求到相应所花费的时间。
- waterfall：请求过程各个节点花费的时间情况。

<img src="/img/blog_network_capture/2.png" width="100%" height="50%">

以Baidu的链接为例。先点击左边的会话链接，然后右边会出现此次连接的请求头和响应头、返回内容预览、返回的内容、对应的时间。

<img src="/img/blog_network_capture/3.png" width="100%" height="50%">

如果想获取某次请求的参数怎么办？接着看。

## 获取请求参数和结果

这里用百度搜索“网络”，右边就会出现对应的请求连接。在“Headers” 一栏，有Query String Parameter的分组，这个就是本次GET请求的参数，对应的response是json数据，大家可以尝试看看。

<img src="/img/blog_network_capture/4.png" width="100%" height="50%">

那post请求呢？类似的道理。

这里借用知乎的登录界面，可以获取到登录时候提交的表单数据，这里知乎有对数据进行加密，并且不是普通的key-value方式提交。所以，在网页端，即使用了post请求，也是不安全的，该获取到还是能获取，除非对数据进行加密。

<img src="/img/blog_network_capture/5.png" width="100%" height="50%">

查看返回结果就比较简单了，直接切到response 下即可

<img src="/img/blog_network_capture/6.png" width="100%" height="50%">

##  分析网页结构

题外话，如果想获取（学习）其他网站的前端结构，当然，也可以通过解析网页的方式拿到对应的数据（这个涉及到html的解析，这里不做讲解），比如知乎的登录页面是如何实现的？

先切换到Elements页面下，对应的内容是网页的源代码，接着点击鼠标icon（2），把鼠标移动到想看的元素上，左键即可在右侧（4）的位置查看对应的源码。

<img src="/img/blog_network_capture/7.png" width="100%" height="50%">

# 2、Fiddler抓包

如果只分析网页数据，用浏览器提供的工具就足够了，但是如果想扩展到其他平台，比如Android端，那就需要Fiddler出场了，[官网下载链接](https://www.telerik.com/download/fiddler)，这里用的是fiddler 4来举例。

与浏览器工具有些类似，左边是请求的会话列表，右边Inspectors页面中，分别是请求内容（2）和响应内容（3）。

<img src="/img/blog_network_capture/8.png" width="100%" height="50%">

## 获取请求参数和结果

那要怎么看请求参数和返回结果呢？如图，请求参数可以直接点击webforms、返回结果可以点击webView或json或raw。

查看get、post请求参数的方式是一样的，这里就不再用post请求举例。

<img src="/img/blog_network_capture/9.png" width="100%" height="50%">

关于Fiddler请求的数据乱码的问题 ，可以参考 [Fiddler抓包中文乱码问题](https://blog.csdn.net/qq_36279445/article/details/79448130)

## 抓APP的数据包

那怎么用Fiddler抓App的数据包呢？

先说下大概思路：电脑开一个代理，手机连接电脑代理后，手机上的流量都会经过电脑，那这个时候就可以在电脑端抓取app的数据。

Fiddler抓app也是这个原理。

首先需要在Fiddler上配置端口，便于手机连接。

点击 tools——> Options.... 就进入该页面，选择Connections 栏目，按照下面的配置（端口可以修改，建议用默认的）

<img src="/img/blog_network_capture/10.png" width="50%" height="50%">

其他选项保持默认的，比如Https栏目下的选项默认勾选，**然后重启Fiddler。**

<img src="/img/blog_network_capture/11.png" width="50%" height="50%">

在命令行输入 `ipconfig` 获取本机的局域网ip地址，比如笔者的ip是 ` 192.168.31.35`

<img src="/img/blog_network_capture/12.png" width="50%" height="50%">

然后手机与电脑要在同一局域网，以小米手机为例，进入WiFi设置页面，点击已连接WiFi右侧的更多，会出现WiFi详情页面，里面有代理设置，按照下图设置即可。

<img src="/img/blog_network_capture/13.png" width="50%" height="50%">



## 手机无法访问网站

如果配置代理后，手机无法正常访问网站，或者Fiddler无法正常截取数据。可以按照如下配置

1、在Fiddler下，点击**Tools > Fiddler Options > connections **勾选上 **Allow remote clients to connect** 选项，确保**Fiddler listens to port是8888**。

2、打开注册表，在**HKEY_CURRENT_USER\SOFTWARE\Microsoft\Fiddler2** （也可以在注册表中直接搜索Fiddler）下创建一个DWORD，值设置为80（十进制）

<img src="/img/blog_network_capture/14.png" width="50%" height="50%">

3、在Fiddler下，点击Rules > Customize Rules，在 `OnBeforeRequest` 函数内加添如下代码

```java
static function OnBeforeRequest(oSession: Session) {
    // 添加的代码
	if (oSession.host.toLowerCase() == "webserver:8888") {
	     oSession.host = "webserver:80";
	}
    // .....
}
```

4、重启Fiddler即可。如下为手机浏览器打开头条的网站记录。

<img src="/img/blog_network_capture/15.png" width="100%" height="50%">

# 3、WireShark

要想学习TCP的连接流程，WireShark是一个不错的选择。[WireShark下载链接](https://www.wireshark.org/download.html)

首页会列出网络连接列表，点击正在使用的即可进入会话列表。

<img src="/img/blog_network_capture/16.png" width="100%" height="50%">

下图是会话列表，其中1 处表示开始抓包；2处表示停止抓包；过滤器可以过滤想看的内容，比如过滤TCP，就会只列出TCP协议的会话；3处表示已抓取的会话列表；4处是对应会话的全部内容；5处是数据包原始的内容。其中第4处中：

- Frame：表示物理层数据帧的内容。
- Ethernet：表示数据链路层的内容。
- Internet Protocol ：表示网络层的内容。
- Transmission control protocol：表示传输层的内容。



<img src="/img/blog_network_capture/17.png" width="100%" height="50%">

## 流量图

WireShark提供了很多种流量统计的方式，其中流量图很直观的看出流量的传输过程。选择 统计——》流量图即可出现下图的内容。

其中第一个ip是本机，右侧的为目标IP地址，可以通过流量图查看TCP的三次握手。

<img src="/img/blog_network_capture/18.png" width="100%" height="50%">


如果需要查看指定握手的详细内容，直接点击对应的连接，即可在主面板上看到对应的数据

<img src="/img/blog_network_capture/19.png" width="100%" height="50%">

## RTT

在  统计——》TCP流图形——》往返时间 处可以查看每个数据包的往返时间，其中点代表数据包的往返时间，横坐标是点的序号，纵坐标是时间（ms），点击下图的点，就可以在主页面上查看对应的会话信息。还有TCP的吞吐量、窗口尺寸等，这里就不一一列举。

<img src="/img/blog_network_capture/20.png" width="100%" height="50%">

更多见 [Wireshark零基础入门到实战](https://www.bilibili.com/video/BV1B5411h7t4?p=1)



——Weiwq  于 2021.04 广州
---
layout:     post
title:      "Kotlin手册"
subtitle:   " \"kotlin关键字与字符大全\""
date:       2021-09-19 08:30:00
author:     "Weiwq"
header-img: "img/background/home-bg-o.jpg"
catalog:  true
top: false
tags:
    - Kotlin

---

> “你想要的kotlin关键字都在这里了“
# 依赖

```java
// 标准库，具体版本参考官网
implementation "org.jetbrains.kotlin:kotlin-stdlib:1.4.20"

```

# 1、关键字

## as
## as?
## actual 


#  2、协程

依赖条件

```java

// Android版本的协程
implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.4.2'

```

async 与 await 在 Kotlin 中并不是关键字，甚至都不是标准库的一部分。

kotlinx.coroutines 是由 JetBrains 开发的功能丰富的协程库。它包含本指南中涵盖的很多启用高级协程的原语，包括 launch、 async 等等。

## async/await 

async { …… } 启动⼀个协程，当我们使⽤ await() 时，挂起协程的执⾏，⽽执⾏正在等待的操作，并且在等待的操作完成时恢复（可能在不同的线程上）。

```java

    fun login() = runBlocking {
        fun asyncWait() = async {
            delay(1000)
            "hello"
        }
        // 通过await方法获取到asyncWait协程的返回值
        println("result: ${asyncWait().wait()}")
    }
```

## launch

开启协程，无返回值

```java
    @Test
    fun test() = runBlocking {
        launch {
            delay(2000)
            print("launch1 ")
        }
        // 或者指定线程
        launch(Dispatchers.IO) {
            delay(2000)
            print("launch2 ")
        }
        println("finish ")
    }
    
```


# 3、操作符





## 后记

到目前为止，Google一直为Android提供新的调试工具，从monitor（已经被打入冷宫）到profiler，从systrace到Perfetto，Android studio也一直在迭代更新。目的就是让开发者能开发出更优秀的产品，也愿各位大佬不辜负Google的期望！共勉！！

——Weiwq  于 2021.08 广州


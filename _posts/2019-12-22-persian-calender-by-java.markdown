---
layout:     post
title:      "波斯日历"
subtitle:   " \"波斯日历的Java实现\""
date:       2019-12-22 16:30:00
author:     "Weiwq"
header-img: "img/background/post-bg-os-metro.jpg"
catalog: true
tags:
    - Android
---

> “波斯日历和公历的互相转换，波斯日历的加减“

## 1、简介

伊朗历（又名波斯历或Jalaali历）是在伊朗和阿富汗使用的阳历。其月份的规律是：前6个月是每月个31天，下5个月是30天，最后一个月平年29天，闰年30天。在对伊朗做相关的软件设计时候，免不了要适配波斯日历。这里需要注意的是，将公历转为波斯日历的时候，仅仅需要转换日期，时间是不需要转的（全球统一用UTC时间计算），例如：

```java
北京时间是：2019/12/22 16:29:31
转为波斯日历就是：1405/01/01 16:29:31
```

## 2、icu4j

  说到软件的全球化，就不能不说说icu4j这个Java开源库，Android也有自己的icu库（估计是control c 的），见[ICU4J Android 框架 API](https://developer.android.google.cn/guide/topics/resources/icu4j-framework.html?hl=zh-cn)。这个库里面包含了波斯日历。用法如下：
 
  
 ```java
 
import com.ibm.icu.text.SimpleDateFormat;
import com.ibm.icu.util.Calendar;
import com.ibm.icu.util.ULocale;

public class Test {

    @org.junit.Test
    public void testIcu4j() {
    		// 输出阿拉伯数字
        ULocale locale = new ULocale("@calendar=persian");
        SimpleDateFormat simpleDateFormat = new SimpleDateFormat(format, locale);
        Calendar persianCalender = Calendar.getInstance(locale);
        System.out.println(simpleDateFormat.format(persianCalender.getTime()));
    }
    
    @org.junit.Test
    public void testIcu4jPersian() {
        // 输出波斯文数字
        ULocale locale = new ULocale("fa_IR@calendar=persian");
        SimpleDateFormat simpleDateFormat = new SimpleDateFormat(format, locale);
        Calendar persianCalender = Calendar.getInstance(locale);
        System.out.println(simpleDateFormat.format(persianCalender.getTime()));
    }
}
 ```
 
但是是需要50.1及以上版本才支持波斯日历，并且整个包在8M以上，如果您有足够的耐心将其calendar包抽离出来，也ok，但是大约也会在2M左右，对于一个apk来说，几M的jar包是相当庞大的库，而且如果只是用到了波斯日历这个功能，将是得不偿失的。最严重的，最近测试出来该转换库有bug。按照简介里面说的“最后一个月平年29天，闰年30天”的计算规则，在公历2015/3/25 这天的转换，用该库转换的结果是"1403/12/30", 显然1403是平年，12月份最多只有29天。可以用以下代码测试

```java
   @org.junit.Test
    public void testIcu4j() {
        java.util.Calendar calendar = java.util.Calendar.getInstance();
        calendar.set(Calendar.YEAR, 2025);
        calendar.set(Calendar.MONDAY, 2);
        calendar.set(Calendar.DAY_OF_MONTH, 20);
        ULocale locale = new ULocale("@calendar=persian");
        SimpleDateFormat simpleDateFormat = new SimpleDateFormat(format, locale);
        Calendar persianCalender = Calendar.getInstance(locale);
        persianCalender.setTimeInMillis(calendar.getTimeInMillis());

        System.out.println(gregorianSimpleDateFormat.format(calendar.getTime()));
        System.out.println(simpleDateFormat.format(persianCalender.getTime()));
    }

```

## 3、PersianCalender库

在多次Google后，没有找到令人满意的Java库（业务需要将公历和波斯日历互转，并且需要支持波斯日历加减年月日），笔者决定自己写一个试试看（icu4j开源库都有小问题，逼上梁山了）。这里感谢 [波斯日历转换](https://cn.calcuworld.com/%E6%B3%A2%E6%96%AF%E6%97%A5%E5%8E%86)提供的算法，按照业务的需求，支持波斯日历和公历的互相转换，波斯日历的加减。最终的实现笔者放在github库中，请自取[PersianCalender.java](https://github.com/weiwangqiang/csdnDemo/blob/master/utils/com.demo/PersianCalender.java)

该实现中，没有用到第三方库，全是jdk提供的接口，用法也很简单，如下：

```java
 /**
     * 将当前日期转为波斯日历
     */
    @org.junit.Test
    public void testPersianTime() {
        String format = "yyyy/MM/dd HH:mm:ss";
        PersianCalender persianCalender = new PersianCalender(format);
        System.out.println(persianCalender.formatPersianCalender());
        persianCalender.set(Calendar.YEAR, 2026);
        persianCalender.set(Calendar.MONTH, 2);
        persianCalender.set(Calendar.DAY_OF_MONTH, 21);
        System.out.println(persianCalender.formatPersianCalender());
    }

    /**
     * 测试加月份
     */
    @org.junit.Test
    public void testPersianTimeAdd() {
        String format = "yyyy/MM/dd HH:mm:ss";
        PersianCalender persianCalender = new PersianCalender(format);
        for (int i = 0; i < 10; i++) {
            persianCalender.add(Calendar.MONTH, 1);
            System.out.println(persianCalender.formatPersianCalender());
        }
    }
```

与icu4j库的转换比较

```java
  @org.junit.Test
    public void testIcu4j() {
        java.util.Calendar calendar = java.util.Calendar.getInstance();
        calendar.set(Calendar.YEAR, 2025);
        calendar.set(Calendar.MONDAY, 2);
        calendar.set(Calendar.DAY_OF_MONTH, 20);
        // 公历
        System.out.println("公历      ：" + gregorianSimpleDateFormat.format(calendar.getTime()));

        ULocale locale = new ULocale("@calendar=persian");
        SimpleDateFormat simpleDateFormat = new SimpleDateFormat(format, locale);
        Calendar icuCalender = Calendar.getInstance(locale);
        icuCalender.setTimeInMillis(calendar.getTimeInMillis());
        // icu4j库转换结果
        System.out.println("icu4j库转换结果： " + simpleDateFormat.format(icuCalender.getTime()));

        PersianCalender persianCalender = new PersianCalender(format);
        persianCalender.setTimeInMillis(calendar.getTimeInMillis());
        // PersianCalender转换结果
        System.out.println("Persian转换结果： " + persianCalender.formatPersianCalender());
    }

```
打印结果是：

```java
公历      ：2025/03/20 17:27:24
icu4j库转换结果： 1403/12/30 17:27:24
Persian转换结果： 1404/01/01 17:27:24

```


# 后记

与icu4j 8M的体积相比，PersianCalender只有13K，当然，icu4j 不仅仅提供了波斯日历，还有其他功能。目前为止，PersianCalender 看起来比icu4j中的更加可靠一些。需要注意的是，PersianCalender库始终以公历存储，只是在formatPersianCalender的时候才会做转换。希望能帮助大家。


—— Weiwq 后记于 2019.12 广州


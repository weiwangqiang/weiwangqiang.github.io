---
layout:     post
title:      "android格式化文件大小"
subtitle:   " \"适配不同版本\""
date:       2019-03-23 20:10:00
author:     "Weiwq"
header-img: "img/background/post-bg-re-vs-ng2.jpg"
catalog: true
tags:
    - android
---

> “老板，来两斤干货，不要货“


## 引言

在系统设置里面，有一个显示当前rom的容量，需要格式化rom大小和单位，之前是使用Android提供的Formatter.formatFileSize方法，在Android 7之前也一直相安无事。直到有一天。。。。。

测试报了一个bug，8G的内存显示出来有8.59G！！！！，多了0.59G～～，扩容不加价哦～

下意识的盘查了一下是不是统计的代码有问题，后来发现同一套代码，在不同Android版本上格式化的结果不一样，有意思。。。

## 1、 Formatter的源码解析
本地IDE依赖的版本是Android 28（Android P，就是这么作死～～），然后大概看了一下源码

```java
    public static BytesResult formatBytes(Resources res, long sizeBytes, int flags) {
        final int unit = ((flags & FLAG_IEC_UNITS) != 0) ? 1024 : 1000;
        final boolean isNegative = (sizeBytes < 0);
        float result = isNegative ? -sizeBytes : sizeBytes;
        int suffix = com.android.internal.R.string.byteShort;
        long mult = 1;
        if (result > 900) {
            suffix = com.android.internal.R.string.kilobyteShort;
            mult = unit;
            result = result / unit;
        }
        ......
     }

```

敏锐的我注意到，formatBytes这个方法最开始做了一个判断，来确定单位值，大概算了一下，8G = 8589934592L,如果单位是1000？不就是。。。。。8.59G了吗！
嗯，有问题,回到formatFileSize方法，

```java

    public static String formatFileSize(@Nullable Context context, long sizeBytes) {
        if (context == null) {
            return "";
        }
        final BytesResult res = formatBytes(context.getResources(), sizeBytes, FLAG_SI_UNITS);
        return bidiWrap(context, context.getString(com.android.internal.R.string.fileSizeSuffix,
                res.value, res.units));
    }
```

这里给formatBytes传入一个FLAG_SI_UNITS 与 FLAG_IEC_UNITS进行同位与操作，再看看他们的值

```java
    /** {@hide} */
    public static final int FLAG_SI_UNITS = 1 << 2;
    /** {@hide} */
    public static final int FLAG_IEC_UNITS = 1 << 3;
```

显然，这两个相与的结果为0，那么unit=1000，也就出现8.59G的结果，好，问题找到了，但是在android 7 上结果怎么就不一样呢？下意识的扫了一眼formatFileSize的注释（写注释是一个良好的习惯）

```java
	 * <p>As of O, the prefixes are used in their standard meanings in the SI system, so kB = 1000
     * bytes, MB = 1,000,000 bytes, etc.</p>
     *
     * <p class="note">In {@link android.os.Build.VERSION_CODES#N} and earlier, powers of 1024 are
     * used instead, with KB = 1024 bytes, MB = 1,048,576 bytes, etc.</p>
```

意思就是在Android 7 之后单位就变了,使用标准的单位制含义，即国际单位制，就像1km = 1000m一样

在Android 7 及更早的版本是1024，即1k = 1024B，这里就不贴代码了，感兴趣的同学可以去看源码。

## 2、解决方案

那么如果需要在高版本使用1024的单位呢？有两个方案，一个是反射设置FLAG_SI_UNITS的值，使得与FLAG_IEC_UNITS相与不为0
，另外一个方案重新封装一个类，如下,用法与Android提供的一样

```java

public class Formatter {
    /**
     * get file format size
     *
     * @param context      context
     * @param roundedBytes file size
     * @return file format size (like 2.12k)
     */
    public static String formatFileSize(Context context, long roundedBytes) {
        return formatFileSize(context, roundedBytes, false, Locale.US);
    }

    public static String formatFileSize(Context context, long roundedBytes, Locale locale) {
        return formatFileSize(context, roundedBytes, false, locale);
    }


    private static String formatFileSize(Context context, long roundedBytes, boolean shorter, Locale locale) {
        if (context == null) {
            return "";
        }
        float result = roundedBytes;
        String suffix = "B";
        if (result > 900) {
            suffix = "KB";
            result = result / 1024;
        }
        if (result > 900) {
            suffix = "MB";
            result = result / 1024;
        }
        if (result > 900) {
            suffix = "GB";
            result = result / 1024;
        }
        if (result > 900) {
            suffix = "TB";
            result = result / 1024;
        }
        if (result > 900) {
            suffix = "PB";
            result = result / 1024;
        }
        String value;
        if (result < 1) {
            value = String.format(locale, "%.2f", result);
        } else if (result < 10) {
            if (shorter) {
                value = String.format(locale, "%.1f", result);
            } else {
                value = String.format(locale, "%.2f", result);
            }
        } else if (result < 100) {
            if (shorter) {
                value = String.format(locale, "%.0f", result);
            } else {
                value = String.format(locale, "%.2f", result);
            }
        } else {
            value = String.format(locale, "%.0f", result);
        }
        return String.format("%s%s", value, suffix);
    }

}

```

## 后记

作为Android developer，需要尽可能的知道各个版本之间的差异，才能适配不同的版本。另外，源码真香！！

—— Weiwq 后记于 2019.03 广州



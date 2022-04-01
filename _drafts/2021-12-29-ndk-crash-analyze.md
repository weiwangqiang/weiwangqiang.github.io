---
layout:     post
title:      "Native crash日志分析"
subtitle:   " \"可能是最好的一篇日志分析\""
date:       2021-12-29 13:30:00
author:     "Weiwq"
header-img: "img/background/home-bg-o.jpg"
catalog:  true
top: false
tags:
    - NDK

---

> “如何分析Native crash问题“
## 1、addr2line

可以用addr2line分析native的异常堆栈，路径是

```cmd
$ %{NDK_PATH}/toolchains/llvm/prebuilt/darwin-x86_64/bin/aarch64-linux-android-addr2line
```

或者在NDK路径下搜索

```cmd
$ find . -name "**addr2line**"
```

用法如下：

```cmd
Usage: ./aarch64-linux-android-addr2line [option(s)] [addr(s)]
 Convert addresses into line number/file name pairs.
 If no addresses are specified on the command line, they will be read from stdin
 The options are:
  @<file>                Read options from <file>
  -a --addresses         Show addresses
  -b --target=<bfdname>  Set the binary file format
  -e --exe=<executable>  Set the input file name (default is a.out)
  -i --inlines           Unwind inlined functions
  -j --section=<name>    Read section-relative offsets instead of addresses
  -p --pretty-print      Make the output easier to read for humans
  -s --basenames         Strip directory names
  -f --functions         Show function names
  -C --demangle[=style]  Demangle function names
  -h --help              Display this information
  -v --version           Display the program's version
```

示例：

```cmd
$ aarch64-linux-android-addr2line -e path/to/so/demo.so 000000000000f423 000000000000f480
```

## 2、ndk-stack

```cmd
ndk-stack -sym obj/armeabi-v7a -dump obj/armeabi-v7a/crash.log
```

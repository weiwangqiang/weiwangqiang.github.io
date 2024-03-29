---
layout:     post
title:      "常用Linux命令"
subtitle:   " \"适用Android shell环境下\""
date:       2021-05-16 17:30:00
author:     "Weiwq"
header-img: "img/background/post-sample-image.jpg"
catalog: true
tags:
    - Linux

---



## 1、查找命令：grep

在当前目录和所有子目录下，查找包含`keyword` 的文件，输出包含文件和行号

```shell

$ grep -n "keyword" -r ./

```

Android下的指定路径，执行如下命令可以找到包含该`string`的apk或文件

```shell

$ grep -rn "string" .

```

过滤多个条件，需要配合cat命令使用

```shell

$ cat demo.txt | grep -E "keyword1|keyword2"

```

## 2、查找命令：find

查找指定文件的详细路径

```shell

$ find . -name "demo.txt" // 查找demo.txt
$ find . -name "demo*" // 查找文件名开头是"demo" 的文件

```


## 3、删除命令：rm

删除文件或文件夹

```java

$ rm -r demo.txt demo

```

强制删除文件或文件夹，*谨慎使用*

```shell

$ rm -rf demo.txt demo

```


## 4、环境变量

```bash
> env // 查看环境变量
> echo $ANDROID_NDK_HOME // 查看 ANDROID_NDK_HOME 的配置

```

## 5、软链接

我们可以通过软链接方式创建源文件的快捷方式

```cmd
// 格式为：ln -s 源文件绝对路径，目标绝对路径
// 比如：
sudo ln -s /Applications/Sublime\ Text.app/Contents/SharedSupport/bin/subl /usr/local/bin/subl
```

如果提示已经存在，则可以强制覆盖

```mcd
ln -fs  /Applications/Sublime\ Text.app/Contents/SharedSupport/bin/subl /usr/local/bin/sublime
```

查看对应的软链接

```cmd
 ls -l /usr/local/bin/subl  
```

查看进程信息
比如查看Java应用程序的进程

```cmd
ps -ef | grep java
```

## 6、文本

在最后一行追加文本
```cmd
echo 131qeqe >> /Users/Downloads/aa.txt
```

覆盖原来的文本
```cmd
echo 131qeqe > /Users/Downloads/aa.txt
```

## 7、selinux

- [理解Linux下的SELinux](https://zhuanlan.zhihu.com/p/86813709)

Security Enhanced Linux(SELinux) 为Linux 提供了一种增强的安全机制，可以通过如下方式关掉该安全机制，降低权限问题

```java
setenforce 0
```


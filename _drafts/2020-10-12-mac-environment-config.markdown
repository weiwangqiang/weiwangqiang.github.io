---
layout:     post
title:      "Mac环境配置"
subtitle:   " \"常用的mac环境配置\""
date:       2021-05-16 17:30:00
author:     "Weiwq"
header-img: "img/background/post-sample-image.jpg"
catalog: true
tags:
    - Linux

---



## 1、配置环境变量

首先执行（“$”表示命令行页面，执行的时候忽略掉即可）
```cmd
$ vim ~/.bash_profile
```

在文件里面添加如下配置

```cmd
# 定义Python环境路径
PYTHON_PATH="/Library/Frameworks/Python.framework/Versions/3.10/bin"
# 定义Java环境路径
JAVA_HOME="/Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home"
# 导入环境变量中，用“:”分开，“${PATH}”为系统默认配置路径，用户自定义的路径放在前面
export PATH="${JAVA_HOME}:${PYTHON_PATH}:${PATH}" 
# 修改Python命令的指向。默认Python命令指向Python2，如果需要指向Python3 ，则添加如下配置
alias python="/Library/Frameworks/Python.framework/Versions/3.10/bin/python3"
```

保存后执行如下命令，更新配置环境

```CMD
$ source ~/.bash_profile
```

查看是否生效

```cmd
$ echo $PATH
```

如果需要永久生效，则执行如下

```cmd
$ vim ~/.zshrc
```

在开头添加

```cmd
if [ -f ~/.bash_profile ]; then
   source ~/.bash_profile
fi
```

使用下面的命令使之立即生效

```cmd
source ~/.zshrc
```


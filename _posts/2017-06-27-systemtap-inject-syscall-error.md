---
layout: post
title: 使用SystemTap给系统调用注入错误
categories: [linux, tool, SystemTap]
description: linux SystemTap
keywords: linux, SystemTap
---

`SystemTap`是linux下一款很好的动态追踪工具，在很多debug/profile场景都很有用。该工具的相关介绍
在网络上已经有很多了，就不在此赘述了。以后写一些结合该工具的debug/profile应用的案例。

## 给syscall注入错误
为了程序的健壮性或者逻辑保护，通常会根据系统调用的错误做很多容错的逻辑，比如网络/磁盘IO错误。
但是如何验证这些容错逻辑又变成了一个问题。

每次测试时，模拟这些错误又是一个很费时费力的问题。我们可以借助SystemTap给syscall注入一些错误，
方便我们模拟各种场景。

## 概念
首先，我们先了解个概念

### Tapsets
Tapset是SystemTap内置的一些脚本，安装再`/usr[/local]/share/systemtap/tapset`下，它将一些probe
点抽象出来，方便我们简化SystemTap脚本。例如`syscall.*`/`process.*`/`socket.*`/`vm.*`等，这些探测
点都是它提供的。如果没有tapsets的话，我们要探测系统调用时，需要这么写：
```
probe kernel.function("sys_*") {
    syscalls[probefunc()]++
}
```

有了tapsets时，我们可以简化为:
```
probe syscall.* {
    syscalls[name]++
}
```

对于syscall，tapsets给我们提供了一下内置变量：
* `name` syscall的名称
* `argstr` 参数值的字符串
* `retstr` 返回值的字符串

同时Tapsets还提供了一些方法，例如：`execname()`/`pid()`等

### 准备工作
通常需要先安装kernel的符号表，跟kernel版本相关，先通过`uname -r`来查看版本，然后安装对应版本的符号表。
例如，当前机器的`uname -r`输出为`3.10.0-229.el7.x86_64`，则需要安装

* `kernel-debuginfo-common-x86_64-3.10.0-229.el7.x86_64.rpm`
* `kernel-debuginfo-3.10.0-229.el7.x86_64.rpm`

例如centos可以先尝试yum安装，或者从[这里](http://debuginfo.centos.org/7/x86_64/)下载。

### 错误注入
在进行错误注入时，我们需要修改一些数据，因此我们需要使用`Guru mode`，即运行脚本时，加入`-g`参数。

例如我们向进程注入一个磁盘没有空间的错误`ENOSPC(28)`：
```
# inject.stp
probe begin {
  printf("inject begin... \n")
}

probe syscall.write.return {
  # 当进程号为脚本后第一个参数时，修改系统调用write的返回值为-28
  if (pid() == $1) {
    $return = -28
  }
}

probe end {
  printf("inject end... \n")
}
```

然后执行命令：
```
stap -g inject.stp 18096
```

## 参考
* [systemtap tutorial](https://sourceware.org/systemtap/tutorial/)
* [SystemTap Language Reference](https://sourceware.org/systemtap/langref/)
* [SystemTap:Instrumenting the Linux Kernel for Analyzing Performance and Functional Problems](http://www.redbooks.ibm.com/redpapers/pdfs/redp4469.pdf)
* [LPC_2008](https://sourceware.org/systemtap/wiki/LPC2008SystemTapTutorial?action=AttachFile&do=view&target=LPC_2008_stap.pdf)
* [SystemTap-II](http://www.neependra.net/kernel/SystemTap-II.pdf)

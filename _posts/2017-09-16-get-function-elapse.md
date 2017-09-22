---
layout: post
title: 统计函数执行耗时
categories: [linux, tool, SystemTap, bcc]
description: linux SystemTap bcc
keywords: linux, SystemTap bcc
---

当我们需要对应用程序进行系能分析时，我们通常可以使用`perf`或者[`火焰图`](http://www.brendangregg.com/flamegraphs.html)。
但是这些工具通常只能定性问题，发现那些函数占用cpu较多，需要优化。但是给不出定量的数据，
比如这个函数的耗时情况，它耗时1ms还是5ms。

# SystemTap

因此在不在代码中加入统计耗时的代码的情况，我们可以使用`SystemTap`来统计应用程序的耗时情况。

`SystemTap`可以跟踪内核函数和用户态进程，当我们跟踪用户态进程时，需要使用其`process`模块。

## 查找函数符号
很多情况下，代码在执行时，其函数符号并不一定是代码中写的名称，因此我们可以使用以下脚本打印
出应用程序中在调用的函数符号。
```
probe process("/data0/app").function("*") {
  println(probefunc())
}
```
然后执行
```
stap echo.stp
```
其会打印出`/data0/app`这个程序运行时调用到的各个函数名，此处最好填绝对路径。我们可以从中找到
我们需要统计的函数名称。

## 统计函数耗时
我们可以使用`SystemTap`内置的直方图来展示耗时的分布。我们有两种直方图函数可以使用:
```
@hist_linear(v, start, stop, interval) # 打印start-stop区间interval间隔的直方图
@hist_log(v)                           # 打印以2为底指数分布的直方图
```

统计脚本：
```
global sends  # 声明全局的统计存储容器

probe process("/data0/app").function("git.intra.xx.send").return { # function中为函数名，同时支持通配符*等，在该函数return时计算耗时
  sends <<< gettimeofday_us() - @entry(gettimeofday_us()) # 以微秒精度来统计，entry方法将一个表达式放置于函数入口处
}

probe timer.s(10) { # 每10s打印一次直方图
  print(@hist_log(sends))
}
```

然后执行`stap elaspe.stp`即可获得每10秒统计的结果，如果希望每10秒清空重新统计的话，
可以将打印函数修改为：

```
probe timer.s(10) { # 每10s打印一次直方图
  print(@hist_log(sends))
  delete sends      # 清空数据
}
```

## 注意
目前`SystemTap`并不能很好的分析`go/golang`程序，会极大概率的引起`go/golang`程序的崩溃，因此推荐使用`bcc`来
分析`go/golang`程序。

# BCC
如果我们要分析`go/golang`程序时，我们就需要[`bcc`](https://github.com/iovisor/bcc)，该工具也是一众大神最近
几年搞出来的新工具，利用linux的`eBPF`功能，因此需要使用`linux 3.15+/4.1+`。

因此我们需要先升级内核再安装`bcc`。

## 安装
* [Ubuntu/Fedora/Arch/Gentoo/openSUSE](https://github.com/iovisor/bcc/blob/master/INSTALL.md)
* [CentOS](http://blog.csdn.net/orangleliu/article/details/54099528)

## 使用
安装`bcc`后，各个工具默认安装路径为`/usr/share/bcc/tools/`

分析函数耗时我们可以使用`/usr/share/bcc/tools/funclatency`，执行命令:
```
./funclatency '/data1/vintage_rd/vintage_new:git*storage.\(*store\).Get' -F -p 29878 -i 20
```

* `-F` 如果规则匹配到多个函数符号，则为每一个函数生成一个独立的直方图
* `-p` 进程号
* `-i 20` 每20秒输出一次结果
* `-u` 时间单位为微秒
* `'/data1/vintage_rd/vintage_new:github*storage.\(*store\).Get'` 为`'程序路径:需要分析的函数符号'`。

注意:此处有一个坑，":"后的函数符号其实为一个正则表达式，但是作者为了方便linux系下熟悉`glob`表达式
的人使用，做了一个转换`pattern = pattern.replace('*', '.*')`。通常情况下写`vfs_fstat*`会被转为`vfs_fstat.*`
然后进行正则匹配。但是`go/golang`程序里，函数符号通常包含`.*()`这些在正则里的特殊字符，会出现一些问题。
因此有两种解决方案：

1. 编辑使用的命令脚本(/usr/share/bcc/tools/funclatency)，删除其中的`pattern = pattern.replace('*', '.*')`这句，
然后函数符号使用正常的正则表达式，例如用`.*\(\*store\)\.Get`来匹配"(*store).Get"结尾的这个函数符号。
2. 根据原脚本中的转换规则来针对性的修改匹配规则，例如`.`这个通常不需要转义，因为在正则里`.`可以匹配到'.'，
`*`也不需要转义，因为`*`被转为`.*`也能匹配到'*'，`()`必须要转义，否则会出错。因此可以使用`*\(*store\).Get`匹配
".(*store).Get"结尾的函数符号，因为其会被转换为`.*\(.*store\).Get`。

其运行结果：
```
Function = git.xxx/storage.(*store).Get [29878]
     usecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 0        |                                        |
         4 -> 7          : 0        |                                        |
         8 -> 15         : 1        |****************************************|
        16 -> 31         : 1        |****************************************|



Function = git.xxx/storage.(*store).Get [29878]
     usecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 0        |                                        |
         4 -> 7          : 0        |                                        |
         8 -> 15         : 1        |****************************************|
        16 -> 31         : 1        |****************************************|
```

## 参考
* [1.148. systemtap](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/5/html/5.6_Technical_Notes/systemtap.html)
* [funclatency_example](https://github.com/iovisor/bcc/blob/master/tools/funclatency_example.txt)

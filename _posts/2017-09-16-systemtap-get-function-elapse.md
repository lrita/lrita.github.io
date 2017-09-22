---
layout: post
title: 使用SystemTap统计函数执行耗时
categories: [linux, tool, SystemTap]
description: linux SystemTap
keywords: linux, SystemTap
---

当我们需要对应用程序进行系能分析时，我们通常可以使用`perf`或者[`火焰图`](http://www.brendangregg.com/flamegraphs.html)。
但是这些工具通常只能定性问题，发现那些函数占用cpu较多，需要优化。但是给不出定量的数据，
比如这个函数的耗时情况，它耗时1ms还是5ms。

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

## 参考
* [1.148. systemtap](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/5/html/5.6_Technical_Notes/systemtap.html)

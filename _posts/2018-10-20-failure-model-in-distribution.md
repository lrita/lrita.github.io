---
layout: post
title: 分布式系统中的故障模型
categories: [distributed-programming]
description: distributed system
keywords: distributed system
---

在分布式系统的设计中，一个很重要的属性就是`容错`。要`容错`那一定要先知道在分布式系统中通常存在哪些错误模型，以及他们之间的关系[^1]。

# Byzantine or arbitrary failures
[`拜占庭问题`](https://zh.wikipedia.org/zh/拜占庭将军问题)是分布式系统中很难处理的问题。出现该问题的节点进入一种“混乱”的状态，对确定的请求`φ`，故障节点给一部分请求返回`R1`，而给另一部分请求返回`R2`。比如，机器的内存出现了故障，程序从内存对应位置读取`KEY`的值，一会儿是`R1`，一会儿是`R2`，则反应到API上就可能是这种问题。在一个可信的分布式系统中，如果没有特殊需要，我们可以不考虑这种极端场景。如果需要解决该问题，可以采用`BFT(Byzantine Fault Tolerance)`/`Practical Byzantine Fault Tolerance`[^2]等共识方法，其能够容忍1/3的节点出现`拜占庭问题`。

# Authentification detectable byzantine failures
`可检测拜占庭问题`是服务进程有时可能会出现`拜占庭问题`，但是它不会否认自己之前打成的共识。通常是服务进程崩溃后，马上被重新启动时，会出现该问题，因为在崩溃前会根据当前情况响应一些请求，但是崩溃重启后，一些未打成共识的状态可能被丢弃，再次进行响应时，可能会跟崩溃的响应不同。

# Performance failures
`性能故障`比较好理解，就是服务进程做出了正确的响应，但是这个响应在错误的时间抵达（过早或者过晚）。比如网络产生的拥堵、请求重试等。

# Omission failures
`失效故障`是`性能故障`的一个特例，就是对应的服务器的响应可能永远无法到达。比如产生了消息丢失。

# Crash failures
`崩溃故障`就是服务进程停止了任何响应，比如进程崩溃等。

# Fail-stop failures
当服务进程进入`崩溃故障`后，并且能够被别的正常服务进程检测到它的故障，则成它进入到`失败停止故障`。

# 故障模型指间的关系
在这些故障模型中，`拜占庭问题`更严重一些，`失败停止故障`轻微一些。当有`拜占庭问题`发生时，我们无法分辨出哪些服务进程是正确的，到底发生了什么。如果有服务进程发生了`失败停止故障`，则其他进程可以清楚的知道它出现了故障。形式上，`byzantine failures` ⊃ `authentification detectable byzantine failures` ⊃ `performance failures` ⊃ `omission failures` ⊃ `crash failures` ⊃ `fail-stop failures`.

在一般的分布式系统中，我们不太需要关注`拜占庭问题`，应该重点关注`performance failures`、`omission failures`、`crash failures`、`fail-stop failures`，因为他们更容易出现，也更容易影响我们的系统。

# 参考
* [FAILURE MODES IN DISTRIBUTED SYSTEMS](http://alvaro-videla.com/2013/12/failure-modes-in-distributed-systems.html)

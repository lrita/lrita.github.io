---
layout: post
title: go优雅升级/重启工具调研
categories: [go]
description: go golang graceful upgrade restart
keywords: go golang graceful upgrade restart
---

对于一个常驻、高访问量的网络服务来说，升级/重启时，一个难以忽视的问题是避免对正在通信的客户端造成影响。因此大家一直在寻求一种优雅、零宕机的升级/重启方案（`seamless reload/upgrade`）。在工程师们的日常实践中，尝试了不同的方案。各方案的核心都是`fork-exec`流程，其不同的区别就是在这个过程中，如何优雅的传递活跃的网络连接，如何避免新建连接失败，以及处理这个过程中的错误和如何回退。

# 方案选型
首先先简单介绍一些方案[^1]：

## SO_REUSEPORT 多进程
在HAProxy 1.5.11时，采用该方案。首先可以对监听 socket 启用`SO_REUSEPORT`，这样可以使得多个监听 socket 共享同一个地址，这样可以使得我们能同时启动多个进程来监听同一个地址。在升级或重启的过程中，我们启动多个进程。

![](/images/posts/com/1-fork.png)

然后向老的进程发送退出命令，然后老的进程停止`accept`然后关闭监听的 socket，然后服务完已有链接后退出进程，最终回归到单进程的状态。

![](/images/posts/com/2-lost-conns.png)

但是这样仍然存在拒绝服务的问题，因为启用`SO_REUSEPORT`的 socket 在内核中拥有不同的队列，在老进程停止`accept`并关闭监听 socket 的过程中（下图红字部分），内核仍然会给该 socket 分配新建的链接到队列中，当老进程关闭监听 socket 后，内核并不会将其队列中的 pending 链接转移另一个监听相同地址的 socket 的队列里去，这样就造成了，如果业务新建连接的QPS很高时，仍然会拒绝一些新建连接的请求。

![](/images/posts/com/3-accept-close.png)

![](/images/posts/com/seamless-reloads.png)

为了解决该问题，我们需要采用`iptable`、`tc`等工具在升级/重启过程中，拒绝或者阻塞SYN包的进入，避免在此过程中产生新建连接，但是会造成服务产生一定的延时[^2]。

## 继承监听SOCKET
其利用父子进程`fork-exec`继承文件描述符的特性，在父子进程之间维护传递监听 socket。在升级/重启的过程中，父进程将监听 socket 继承给子进程，使得整个过程没有监听 socket 被关闭，从而不产生拒绝服务的问题。但是这使得进程模型变得复杂。

因此有的项目，例如[einhorn](https://github.com/stripe/einhorn)，其将监听 socket 与业务逻辑分离成为独立进程，成为一个`SOCKET SERVER`，专门负责监听 socket 的传递，使得父子进程模型变得简单。

## UNIX SOCKET进程间传递监听SOCKET
HAProxy 1.8[^3]采用该方案，其采用`UNIX SOCKET`的[access ancillary data](https://linux.die.net/man/3/cmsg)中的`SCM_RIGHTS`在非同源父子进程间传递监听 socket 。这样也使得在升级/重启过程中产生关闭监听 socket的问题。

# 开源实现
下面简单分析几个go语言实现的相关功能的 package：

## `facebookarchive/grace`
[`facebookarchive/grace`](https://github.com/facebookarchive/grace)整体实现比较简单，其只提供了一个简单的`继承监听套接字`方案，并不具备处理子进程失败、已有连接的功能。

## `rcrowley/goagain`
[`rcrowley/goagain`](https://github.com/rcrowley/goagain)的实现比`facebookarchive/grace`还要简单，采用`继承监听套接字`方案，但是只能继承一个监听 socket，参考价值比较低。

## `jpillora/overseer`
[`jpillora/overseer`](https://github.com/jpillora/overseer)采用主从进程设计，有父进程创建监听 socket ，然后`fork-exec`派生出子进程，将全部监听 socket 继承给子进程，业务逻辑由子进程来运行。自带定时拉取新版本升级的功能，比较适合用来写App/Agent。由于框架设计的开发性不足，用户定制性差，比如动态增加端口等功能无法在该框架下实现。

## cloudflare/tableflip
[cloudflare/tableflip](https://github.com/cloudflare/tableflip)采用`继承监听套接字`方案[^4]，整体设计开放性足够，目前看起来是最好的一个实现。其提供在升级/重启过程中的父子进程之间同步功能，例如`Ready()`、`WaitForParent()`等。也能够灵活处理多个监听 socket和已存在的链接等。

# 参考

[^1]: [GLB part 2: HAProxy zero-downtime, zero-delay reloads with multibinder](https://github.blog/2016-12-01-glb-part-2-haproxy-zero-downtime-zero-delay-reloads-with-multibinder/)
[^2]: [True Zero Downtime HAProxy Reloads](https://engineeringblog.yelp.com/2015/04/true-zero-downtime-haproxy-reloads.html)
[^3]: [Truly Seamless Reloads with HAProxy – No More Hacks!](https://www.haproxy.com/blog/truly-seamless-reloads-with-haproxy-no-more-hacks/)
[^4]: [Graceful upgrades in Go](https://blog.cloudflare.com/graceful-upgrades-in-go/)


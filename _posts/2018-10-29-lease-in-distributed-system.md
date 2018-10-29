---
layout: post
title: 分布式系统中的 lease（租约）机制
categories: [distributed-programming]
description: distributed programming
keywords: distributed programming
---

_编：`lease`还是非常容易理解的一个概念，但是为了不全知识地图，还是有必要写一下。_

# 简介
`lease(租约)`机制在分布式系统中有着广泛的应用，比如：选举、分布式锁、维护缓存一致性等。其本质上是一种协调机制，使得在分布式环境中，让不同进程之间产生一种同步语义。

**`lease(租约)`就是一种合约，各个参与者授予获得`lease(租约)`的实例在一段“期限”内拥有“特权”。在这个“期限”内，各个参与者都承认其“特权”。如果要延续这个期限，就需要“续约”。维持这个“期限”是否过期的判断条件可以是物理时钟、逻辑时钟也可以是版本号等信息，但是手段通常是“心跳”。“特权”的具体内容通常是软件逻辑预设好的，比如拥有写操作、成为主节点、持有某个资源等**。

其组织结构可以是参与者之间直接连通，也可以通过一个“协调者”。

![](/images/posts/distribution/lease-1.png)

# 容错能力
由于`lease(租约)`拥有“期限”，可以非常好的容错网络异常。当“租期”内网络出现分区、异常，在“租期”内仍然不会影响进程的正常工作，只是不能进行“续租”而已，如果网络能在“租期”耗尽前恢复，则不会产生任何异常。

由于`lease(租约)`能够自动释放，可以很好的容错宕机问题，当获得`lease(租约)`的节点宕机后，“租期”耗尽时，自动释放，使得其他节点能够重新获取`lease(租约)`。

但是`lease(租约)`不能容忍`拜占庭式故障`[^1]，根据上面的组织结构，如果获得`lease(租约)`的节点为拜占庭节点，会使得各个节点间数据产生不一致的情况。

# 时间流逝速度问题
多数`lease(租约)`的实现都是基于时间的，我们需要**假设各个节点上时间流逝速度是相同的，或者时间漂移的范围是有界的并且在计算“租期”时考虑上这个漂移上界**。在现实的`半同步网络`[^2]中，这一点很容易得到保证。

# 参数平衡
跟`lease(租约)`相关的参数有很多，多数没有什么规定，主要根据业务场景进行一些平衡。

* “租期”的长短。通常情况下都选取比较短的“租期”。如果获得“租约”的节点宕机时，“租期”较短时，能够快速使其他节点获得“租期”，使得失效时间很短。但是越短的“租期”，造成的“续约”成本更高。不过通常不太需要考虑“续约”的成本。
* 何时进行“续租”。在“租期”到达前多久开始“续租”，要考虑通讯耗时、不同节点间的时钟漂移等。

# 实践
在[Fault-tolerant and decentralized lease coordination for distributed systems](/images/posts/distribution/flease-fault-tolerant-and-decentralized-lease-coordination-for-distributed-systems.pdf)讲解了一种基于PAXSO算法的，`时钟漂移`有上界的`部分同步网络`的`lease(租约)`算法。

# 参考
[^1]: [分布式系统中的故障模型](https://lrita.github.io/2018/10/20/failure-model-in-distribution/)
[^2]: [分布式系统中的通讯模型](https://lrita.github.io/2018/10/19/communication-model-in-distribution/)
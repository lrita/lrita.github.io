---
layout: post
title: 分布式系统中的 safety 和 liveness
categories: [distributed-programming]
description: distributed system
keywords: distributed system
---

在分布式系统的算法和设计中，`safety`和`liveness`是2个非常重要的属性，这个概念最早由`L. Lamport`[^1]提出。这2个属性是非常基础的属性，系统中的其他属性都可以被分解为`safety`和`liveness`。

通俗来讲，这2个属性的含义就是[^2]：
> * `safety` something "bad" will **never** happen
> * `liveness` something "good" will **must** happen (but we don’t know when)

我们可以举一些例子来说明：
> 1. 同一时刻，只有一个进程能够进入互斥临界区

**1**是一个关于`safety`的表述，它表述了，什么样的时间不应该被发生，那就是”同一时刻，不少于一个进程进入了互斥临界区“。

> 2. 进程`P2`不会永远停留在互斥临界区，以至于`P1`最终能够进入互斥临界区。

**2**是一个关于`liveness`的表述，它表述了什么样的事件最终应该被发送，那就是“`P1`最终会进入互斥临界区”。

> 3. gossip协议具有[最终一致性](https://en.wikipedia.org/wiki/Eventual_consistency)

**3**是关于`liveness`的表述，

更加形式化的解释我们可以参考`Safety & Liveness Properties`[^3]。

虽然`safety`和`liveness`是正交地两个属性，但是在设计一个分布式系统时，我们需要同时考虑这两个属性，只具备其中之一的系统是没有意义的[^4]。

避免`死锁`是保证`liveness`的一个充分条件，虽然这个约束比较弱。但是是比较好验证的，如果算法、逻辑中存在死锁，那么一定不能保证系统的`liveness`。避免`饥饿`也是保证`liveness`的一个充分条件，这个约束比避免`死锁`更强一些，因为避免`饥饿`的算法一定是避免`死锁`的。更强的约束是，保证算法、逻辑能在`有限步骤内完成`，这样系统一定是`liveness`的，但是这个约束条件很难被验证和证明[^5]。

关于`safety`和`liveness`的历史可以参考`afety and liveness properties: a survey`[^6][^7]

# 参考
[^1]: [Proving the Correctness of Multiprocess Programs](https://ieeexplore.ieee.org/document/1702415)
[^2]: [Safety and Liveness](/images/posts/distribution/Safety-and-liveness.pdf)
[^3]: [Safety & Liveness Properties](/images/posts/distribution/Safety-Liveness-Properties.pdf)
[^4]: [Safety and Liveness: Eventual Consistency Is Not Safe](http://www.bailis.org/blog/safety-and-liveness-eventual-consistency-is-not-safe/)
[^5]: [Wiki Liveness](https://en.wikipedia.org/wiki/Liveness)
[^6]: [safety and liveness properties: a survey](/images/posts/distribution/safety-and-liveness-properties-a-survey.pdf)
[^7]: [Recognizing safety and liveness". Distributed Computing](/images/posts/distribution/RecSafeLive.pdf)



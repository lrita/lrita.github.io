---
layout: post
title: 【转】并发编程的15条建议(译)
categories: [program]
description: program
keywords: program
---

本文转自[并发编程的15条建议(译)](https://www.cnblogs.com/Solstice/archive/2010/09/29/realworld_concurency.html)。

内核专家 Bryan Cantrill 和 Jeff Bonwick 在 2008 年 9 月的《ACM Queue》上发表了[《Real-world Concurrency》](/images/posts/com/Real-world-Concurrency.pdf) 一文，提出了 15 条并发编程的建议，这里简单摘录如下。

**1. Know your cold paths from your hot paths.弄清楚代码里的热门执行路径和冷门执行路径。**

对冷门路径，用粗粒度的锁即可。对热门路径——也就是那些必须高度并发才能实现所期望的高吞吐量的代码，应该更加小心，加锁的策略必须简单明了且细粒度。

**2. Intuition is frequently wrong—be data intensive. 直觉常常是错的，要靠数据说话。**

【陈硕】比如线程切换到底有多大开销，普通 mutex 加锁到底有多大代价，系统调用的开销如何，gettimeofday() 在 x86-64 Linux 是不是真的系统调用等等，都要靠数据说话。

**3. Know when—and when not—to break up a lock. 知道什么时候把一个锁拆成多个，并知道什么时候不必这样做。**

除了把全局锁拆成多个锁，另外一种常用的避免线程争用 (contention) 的办法是减少加锁的范围。比方说从共享的数据结构里移除(remove and delete)元素，其实 delete 这一步可以放到锁外面。

**4. Be wary of readers/writer locks. 警惕读写锁。**

初学者常犯的一个错误是，见到某个数据结构频繁读而很少写，那么就把 mutex 替换为 rwlock。这不见得是正确的。

【陈硕】这条深得我心，muduo thread lib 目前就没有提供读写锁的封装。另外，这一条也能鉴别另一篇关于线程争用的文章不靠谱。

**5. Consider per-CPU locking. 考虑用每个 CPU 用一个锁。**

**6. Know when to broadcast—and when to signal. 知道什么时候用单个唤醒，什么时候用广播唤醒。**

notifyAll() 通常表示状态变更，而 notify() 通常表示资源变得可用。滥用 notifyAll() 会导致惊群现象。

【陈硕】 muduo thread lib 的 ThreadPool 区分使用 notify() 和 notifyAll()，可作参考。

**7. Learn to debug postmortem. 学会验尸。**

【陈硕】 在程序中只使用 Scoped locking 来加锁的话，很容易从 call stack 查出死锁。参考《多线程服务器的常用编程模型》第 6 节 线程间同步。

**8. Design your systems to be composable. 设计系统，使之能扩充。**

【陈硕】 比方说，把对对象的修改操作都挪到同一个线程，这样就不必加锁。参考 muduo 的 EventLoop::runInLoop()。

**9. Don’t use a semaphore where a mutex would suffice. 如果 Mutex 就能解决问题的话，不要使用信号量 semaphore。**

【陈硕】muduo thread lib 有意识地不提供信号量的封装。

**10. Consider memory retiring to implement per-chain hash-table locks. 考虑用内存“退休”法来实现哈希表的按桶加锁。**

**11. Be aware of false sharing. 知道什么是伪共享。**

跟多 CPU 的 Cache 有关，值得了解。

**12. Consider using nonblocking synchronization routines to monitor contention. 考虑使用非阻塞的加锁来观察线程争用。**

**13. When reacquiring locks, consider using generation counts to detect state change. 在重新加锁时，考虑使用版本号来检测状态变更。**

**14. Use wait- and lock-free structures only if you absolutely must. 只在别无它法时才使用无锁数据结构。**

**15. Prepare for the thrill of victory—and the agony of defeat. 准备接受成功的喜悦和失败的痛苦。**

更详细的解释请看原文。

Bryan Cantrill 是 dtrace 的主要作者，Jeff Bonwick 是 ZFS 和 Slab allocator 的发明者。

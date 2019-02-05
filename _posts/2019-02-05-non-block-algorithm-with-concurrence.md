---
layout: post
title: 并发中的非阻塞算法
categories: [datastructure]
description: datastructure
keywords: datastructure 并发中的非阻塞算法
---

现在多核计算机已经成为了当今主流，因此我们在每个程序中都需要考虑到多核并发的问题，在有高性能计算需求的项目中更是重要。其中在数据结构中，我们需要对并发考虑的更多，这里就要涉及很多并发算法。

在高并发的场景下，经常需要一些非常高效、并发度高的数据结构。通常基于悲观并发会加入互斥锁、信号量等同步原语。这样是一种阻塞算法，当持有互斥锁的线程被挂起后，其他线程也会被阻塞住，是一种并发度比较低的算法。与之相比并发度更高的有一些非阻塞算法。总之，互斥原语存在很多缺点[^1]。但是使用互斥语义在代码上实现简单，适用于大多数场景。

下面我们就要介绍一些并发度更高的非阻塞算法模型，这里我们并不叙述某一具体类型算法的实现，而是将它们之间的分类、区别，主要从概念上出发，这样可以让我们接触到一些具体实现时，可以通过其实现的关键点将其迅速归类，从而分析出该实现的原理性边界。

# 基本概念

## 阻塞(block)

`阻塞(block)`这个概念是我们讨论下面问题的基石，其定义是：
```
A function is said to be Blocked if it is unable to progress in its execution until some other thread releases a resource.
```

我认为**同时具有**以下两点特征的都是`阻塞(block)`的：

* 互斥性（串行操作）
> 比如超市的结账通道，大家只能按照一定的顺序依次结账。牌堆同时有很多张扑克，荷官依次给每一名玩家发牌，而不是大家同时去牌堆抢牌。又如`mutex`保护一个变量，将所有修改变量的操作依次执行。
* 通过性（故障容忍）
> 比如超市的结账通道，如果正在结账的顾客与收银员发生了冲突，则后面的顾客也只能等待。如果此时没有任何机制接入，则是不能容忍故障的一种表现。如果此时有经理介入，将该名顾客请离结账通道单独处理，这样收银员就能继续服务其他顾客。又如持有`mutex`的线程如果发生的异常，没有释放`mutex`，则关于该`mutex`的全部操作都无法进行处理。
>
> 这里要特别指出一点，`阻塞(block)`是指处理一件事的全部过程，从头到尾的这个过程。还是上面那个例子，如果一个顾客与收银员正在发生冲突，后面等待的顾客比较聪明，他们离开了结账通道，再去做些其他事，比如上洗手间，待会再来查看结账通道是否恢复。这样整个过程仍然是`阻塞(block)`的，因为在这个过程中后续顾客仍然是没有完成**结账**这个目的，只不过是中间穿插进行了一些其他无关事件。因此`mutex`中的`trylock`仍然是`阻塞(block)`的，`trylock`只是可以帮助线程在被阻塞时能进行一些无关工作，如果一直`trylock`失败，则其目一直无法达成。在知乎上一些人认为`trylock`是`wait-free`的，这是极大的认知错误。

比较简单的例子，通过`CAS`循环来同步状态。
```java
AtomicInteger lock = new AtomicInteger(0);
public void funcBlocking() {
         while (!lock.compareAndSet(0, 1)) {
               Thread.yield();
        }
}
```

## 步进性(progress)

`步进性(progress)`，对这个翻译也不是太满意，后面都使用`progress`。`progress`是一个指标，用来衡量一个算法“非阻塞”的程度，因此`阻塞(block)`算法的`progress`最低。如果每个参与者的操作都能在有限时间内完成，这样的`progress`是最高的，意味着每个参与者都不会被`阻塞(block)`。

需要明确的一点是，`progress`高并不一定代表效率高。在一些场景下阻塞的实现可能比非阻塞的实现效率更高，比如一个算法，如果采用阻塞的实现，每个线程只需要10ms，但是采用非阻塞的实现可能就需要100ms。

# 讨论范畴
在讨论这些问题前，首要的是，我们明确我们讨论的范畴，通常会有很多人没有明确讨论的范畴导致将一些概念混淆在一起。在常规情形下，我们讨论这类问题从小到大分为：

* 数据结构，判断是否阻塞的范畴是线程，我们讨论的是线程操作的`progress`。
* 数据库事务，判断是否阻塞的范畴是事务，我们讨论的是事务操作的`progress`。
* 分布式编程，判断是否阻塞的范畴是进程，我们讨论的是进程操作的`progress`。

下面的讨论以*数据结构*的范畴为准，要扩展到其他范畴，需要做针对性的变化。

# 算法分类

通常情况下，将各种并发算法可以分为以下几类，通常一个数据结构的不同方法可能采用不同的算法实现。比如一个map结构，读方法可能是`lock-free`的而写方法是`deadlock-free`的。

## `deadlock-free`

`deadlock-free`是一个并发数据结构的最低要求，其实现不会产生循环依赖的资源。因为循环依赖的资源会造成死锁情况的发生。

## `starvation-free`

`starvation-free`有时也被叫做`lockout-free`，通常取决于系统底层的保证。其定义为：想进入临界区的线程最终也够进入。

通常一个完全公平调度的排他锁是`starvation-free`的，比如x86中依赖内存总线锁实现的操作`atomic_fetch_add()`。比如完全公平锁[`Ticket Lock`](https://github.com/pramalhe/ConcurrencyFreaks/blob/master/C11/locks/ticket_mutex.c)等实现。

## `obstruction-free`

`obstruction-free`是最弱的非阻塞`progress`保证，其要求：在任何一线程单独运行时（其他线程都休眠或等待时），它能够在有限步数内完成相关操作。所有的`lock-free`算法都是`obstruction-free`的。

`obstruction-free`通常只要求任何部分完成（未全部完成）的操作可以被中断且回滚。可以阻止系统在争抢资源时，进入`live-locking`状态。

一些`obstruction-free`算法在实现时引入"consistency markers"角色，在一些操作前后都读取"consistency markers"的状态进行比较，当"consistency markers"指示状态发生变化时，其进行回滚重试。

类似的实现有：《Obstruction-Free Synchronization: Double-Ended Queues as an Example》[^2]。

## `lock-free`

`lock-free`对部分线程提供`progress`保证，其可能造成部分线程处于饥饿状态，但是在整体上提供了较大的吞吐。其定义是：

> 当程序的全部线程运行的时间足够长时，至少有一个线程能确保完成操作，保证`progress`（能够容忍任意线程被挂起，简单来说，就是前面提到的`通过性`，整个算法不会受到某一线程故障的影响）。

则，有任意两个线程争抢同一`mutex`或`spinlock`的算法，都不是`lock-free`的。所有的`wait-free`算法都是`lock-free`的。

In general, a lock-free algorithm can run in four phases: completing one's own operation, assisting an obstructing operation, aborting an obstructing operation, and waiting. Completing one's own operation is complicated by the possibility of concurrent assistance and abortion, but is invariably the fastest path to completion.[^3]

如何正确的处理`when to assist, abort or wait`是非常复杂的，而且可能带来非常高的性能损耗，但是好歹还是保证了`progress`。

比如，用CAS循环来实现一个原子自增逻辑：
```java
AtomicInteger atomicVar = new AtomicInteger(0);
public void funcLockFree() {
        int localVar = atomicVar.get();
        while (!atomicVar.compareAndSet(localVar, localVar+1)) {
               localVar = atomicVar.get();
        }
}
```

## `wait-free`

`wait-free`是最强的`progress`保证，同时结合了最大的系统吞吐和避免饥饿。其保证所有参与操作的线程能在有限步数内完成，通常这个有限步数与参与的线程数有关。

理论上，所有的算法都可以被设计为`wait-free`的，但是不能保证其效率高于`block`的实现[^4]。

例如：《[Wait-Freedom vs. Bounded Wait-Freedom in Public Data Structures](http://www.cs.technion.ac.il/~moran/r/PS/bm94.ps)》[^5]

# 关联关系

虽然人们都把相关算法分为这五大类，但是他们之间分隔的维度并不统一，他们之间并不是简单的包涵、从属关系。下图可以简单的表示他们之间的属性关系，箭头指向的方向是算法具备的属性。例如`wait-free`同时具备`lock-free`和`starvation-free`属性[^6]。

![](/images/posts/datastructure/Relationships-between-Progress-Properties.png)

上图从中间分开，他们从不同方面定义了算法的属性。具备`wait-free`、`lock-free`或`obstruction-free`属性的可以被称为非阻塞的算法，他们从“保证参与者最终能完成操作”出发，区别在于保证的参与者数量和参与者之间是否需要协调。而具备`deadlock-free`或`starvation-free`不一定是非阻塞的。`deadlock-free`或`starvation-free`从参与者的公平性上出发，区分不同算法。

还有一种“大统一理论”[^7]的划分方式，将这些算法划分为二维区间：

![](/images/posts/datastructure/concurrent-algorithm-table.png)

在上图中，X轴的维度是`progress`，`progress`从左向右逐步降低；而Y轴方向的维度是`是否依赖系统对线程的调度`，上侧不依赖系统调度，下侧依赖系统调度。

# 参考
[^1]: [Disadvantages of locks](https://en.wikipedia.org/wiki/Lock_(computer_science)#Disadvantages)
[^2]: [Obstruction-Free Synchronization: Double-Ended Queues as an Example](/images/posts/datastructure/Obstruction-Free-Synchronization-Double-Ended-Queues-as-an-Example.pdf)
[^3]: [Non-blocking algorithm](https://en.wikipedia.org/wiki/Non-blocking_algorithm#Lock-freedom)
[^4]: [Impossibility and universality results for wait-free synchronization](https://dl.acm.org/citation.cfm?doid=62546.62593)
[^5]: [Lock-Free and Wait-Free, definition and examples](http://concurrencyfreaks.blogspot.com/2013/05/lock-free-and-wait-free-definition-and.html)
[^6]: [Characterizing Progress Properties of Concurrent Objects via Contextual Refinements](/images/posts/datastructure/characterizing-progress-properties-of-concurrent-objects-via-contextual-refinements.pdf)
[^7]: [On the Nature of Progress](/images/posts/datastructure/on-the-nature-of-progress.pdf)




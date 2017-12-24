---
layout: post
title: Non-Volatile RAM/非易失内存
categories: [system]
description: system
keywords: system
---

通常我们系统中使用的内存都是易失内存，当断电后，内存中的数据都会丢失。因此我们的程序一般都需要
启动时将数据加载进内存，退出时将数据持久化到磁盘。如果我们有一种内存，省去了这些问题，是不是会
使我们的程序变得更加简单？

实现`Non-Volatile RAM`的方式也很多:
* 有从硬件入手，实现这种物理介质
* 有从kernel入手，将SSD与内核内存映射相结合
* 有从文件系统入手，利用文件的`mmap`机制来实现


当然`Non-Volatile RAM`一直都是科研的先锋，一般用户还很难接触到它们，但是随着`SSD`的高速发展，
`SSD`的读写性能越来越高，我们一般可以借助`SSD`来实现`Non-Volatile RAM`，或者作为一种持久的内存
来看待。这样我们的程序就有了另一种写法，对于各种数据结构的处理都可以重新进行考虑。

![disk.png](/images/posts/memory/disk.png)

从上图来看，SSD的写入速度、延时已经与RAM差距很小了。

比如[pmdk](https://github.com/pmem/pmdk)，就是上面的第三种方案的实现，其使用`mmap`文件作为可持久化
的内存，然后依赖`jemalloc`进行内存管理。`Apache Kudu`也在尝试使用`pmdk`。在这方面给了我们一些新方案
的启示，以后可以在一些项目中进行尝试。

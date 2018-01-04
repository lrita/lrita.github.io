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
* 有从kernel入手，将SSD与内核内存映射相结合[^1]
* 有从文件系统入手，利用文件的`mmap`机制来实现


当然`Non-Volatile RAM`一直都是科研的先锋，一般用户还很难接触到它们，但是随着`SSD`的高速发展，
`SSD`的读写性能越来越高，我们一般可以借助`SSD`来实现`Non-Volatile RAM`，或者作为一种持久的内存
来看待。这样我们的程序就有了另一种写法，对于各种数据结构的处理都可以重新进行考虑。

![disk.png](/images/posts/memory/disk.png)

从上图来看，SSD的写入速度、延时已经与RAM差距很小了。

比如[pmdk](https://github.com/pmem/pmdk)，就是上面的第三种方案的实现，其使用`mmap`文件作为可持久化
的内存，然后依赖`jemalloc`进行内存管理。`Apache Kudu`也在尝试使用`pmdk`。在这方面给了我们一些新方案
的启示，以后可以在一些项目中进行尝试。

# Linux DAX
在高版本的linux中新增加了[DAX](https://www.kernel.org/doc/Documentation/filesystems/dax.txt)机制，使
得用户可以直接将块设备直接映射到内存空间，减少了拷贝到`page cache`额外开销（之前的文件系统，需要将设备
上的数据拷贝到`page cache`中，然后将`page cache`映射到用户的内存空间，中间的所有操作都需要额外的内存开
销，以及`page cache`的异步写入磁盘等步骤，对于`NVMe`/`FusionIO`/`NVDIMM`这种高速设备来说，`page cache`
这层是额外的开销），当然DAX也需要硬件自身的支持。

# 参考
[^1]: [How to emulate Persistent Memory](http://pmem.io/2016/02/22/pm-emulation.html)

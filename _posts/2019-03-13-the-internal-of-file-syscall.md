---
layout: post
title: 文件IO系统调用内幕
categories: [linux]
description: linux filesystem syscall mmap write read
keywords: linux filesystem syscall mmap write read
---

# 分层结构

Linux 文件系统采用分层设计，不同层抽象出来完成不同层次的逻辑。其中的复杂的设计，可以拿两张图在不同程度上描绘：

![](/images/posts/filesystem/Linux-storage-stack-diagram_v4.10.png)

![](/images/posts/filesystem/Linux.IO.stack_v1.0.png)

因此用户态的应用程序每一个 IO 请求产生的数据都会经过文件系统的不同抽象层，但是很多抽象层之间是异步交互的，这使我们理解文件系统的难度又进一步提升。当然，还可以用一个简单的抽象来表示其中每个抽象层之间的关系：

![](/images/posts/filesystem/io_path_simple.png)

下面我们会简单分析一些场景文件 IO 系统调用的内核流程，让我们了解其中的内幕。

# 虚拟文件系统

为了连接不同文件系统的设计，内核在各种文件系统之上增加一个抽象层`虚拟文件系统(VFS)`，很多通用设计都在其中，包括一个文件在内核中如何表示、元数据`struct file`、`struct inode`等的，具体分析可以参考：[《Linux 虚拟文件系统》](/images/posts/filesystem/Linux.Virtual.Filesystem.pdf)

每个文件`open`后，会得到一个`struct file`来，在用户进程中为一个`int fd`来表示，文件读写偏移等元数据存储在`struct file`中。

同时，每个文件有唯一的一个`struct inode`，文件权限、属性等存储在其中。

因此，多个`struct file`可以对应同一个`struct inode`。

# pagecache

`pagecache`是内核为文件创建的内存缓存，用以加速相关的文件操作。当应用程序需要读取文件中的数据时，操作系统先分配一些内存，将数据从存储设备读入到这些内存中，然后再将数据分发给应用程序；当需要往文件中写数据时，操作系统先分配内存接收用户数据，然后再将数据从内存写到磁盘上。

- `pagecache`的内存在内核中是匿名的物理页（不与用户进程的逻辑地址进行映射），由`struct page`表示，在内核中`pagecache`使用 LRU 管理，当用户进行`mmap`映射文件时，内核创建对应的`vma`，在访问到`mmap`的内存区域时，触发`page fault`，在`page fault`回调中`pagecache`内存所属的物理页与用户进程的虚拟地址`vma`进行映射。
- 每个文件的`pagecache`元数据存储于对应的`struct inode->address_space`中，因此进程之间可以共享同一个文件的`pagecache`，同一个文件多次`open`不会影响其`pagecache`。
- 文件的`pagecache`是延时分配的，当有读写命令时，才会按需创建缓存页。
- `pagecache`的脏页是单线程回写的，因此当一个文件大量写入时，写入的性能与单 CPU 的性能有相当的关系。

详细的分析可以参见：[《Linux 内核文件 Cache 机制》](/images/posts/filesystem/Linux.Kernel.Cache.pdf)、[《Linux 内核延迟写机制》](/images/posts/filesystem/Linux.Kernel.Delay.Write.pdf)

# 块设备层

块设备层在不同版本中有不同的划分方式，但是总的来说可以分为[《Linux 通用块设备层》](/images/posts/filesystem/Linux.Generic.Block.Layer.pdf)和[《Linux 内核 IO 调度层》](/images/posts/filesystem/Linux.Kernel.IO.Scheduler.pdf)。其中两部分可以详见链接中的讲解。

关于块设备层中的主要逻辑流程，可以参考下图：

![](/images/posts/filesystem/LinuxBlockIO.png)

# 系统调用

## open

`open`负责在内核生成与文件相对应的`struct file`元数据结构，并且与文件系统中该文件的`struct inode`进行关联，装载对应文件系统的操作回调函数，然后返回一个`int fd`给用户进程。后续用户对该文件的相关操作，会涉及到其相关的`struct file`、`struct inode`、`inode->i_op`、`inode->i_fop`和`inode->i_mapping->a_ops`等。

_注：文件操作对应的偏移存储于`struct file`中，每个`open`的文件单独维护一份，同一个文件的读写操作共享同一个偏移。_

其整个内核逻辑流程可以用下图来表示：

![](/images/posts/filesystem/syscall_open.png)

## write

`write`的写逻辑路径有好几条，最常使用的就是利用`pagecache`延迟写的这条路径，所以主要分析这个。在`write`调用的调用、返回之间，其负责分配新的`pagecache`，将数据写入`pagecache`，同时根据系统参数，判断`pagecache`中的脏数据占比来确定是否要触发回写逻辑。其详细的代码分析可以参考：[《Linux 内核写文件过程》](/images/posts/filesystem/Linux.Kernel.Write.Procedure.pdf)和[《Linux 内核延迟写机制》](/images/posts/filesystem/Linux.Kernel.Delay.Write.pdf)。

其整个内核逻辑流程可以用下图来表示：

![](/images/posts/filesystem/syscall_write.png)

## read

`read`的读逻辑中包含预期`readahead`的逻辑，其可以通过与`fadvise`的配合达到文件预取的效果。这部分的代码分析可以参考：[《Linux 内核读文件过程》](/images/posts/filesystem/Linux.Kernel.Read.Procedure.pdf)

其整个内核逻辑流程可以用下图来表示：

![](/images/posts/filesystem/syscall_read.png)

## fsync/fdatasync

`fsync`和`fdatasync`主要逻辑流程基本相同。其通过触发对应文件的`pagecache`脏页回写，并且阻塞等待到回写逻辑完成，以达到同步数据的目的。

其整个内核逻辑流程可以用下图来表示：
![](/images/posts/filesystem/syscall_fsync.png)

## mmap

用户调用`mmap`将文件映射到内存时，内核进行一系列的参数检查，然后创建对应的`vma`，然后给该`vma`绑定`vma_ops`。当用户访问到`mmap`对应的内存时，CPU 会触发`page fault`，在`page fault`回调中，将申请`pagecache`中的匿名页，读取文件到其物理内存中，然后将`pagecache`中所属的物理页与用户进程的`vma`进行映射。

其整个内核逻辑流程可以用下图来表示，其中`page fault`部分比较简略，可以参考[Linux Page Fault(缺页异常)](/2019/03/07/linux-page-fault/)：
![](/images/posts/filesystem/syscall_mmap.png)

## munmap

![](/images/posts/filesystem/syscall_munmap.png)

## msync

`msync`的实际实现与其手册中的描述有很大不同，其调用时，`flag=MS_SYNC`等同于对`mmap`对应的文件调用`fsync`；`flag=MS_ASYNC/MS_INVALIDATE`其实什么都不执行。

![](/images/posts/filesystem/syscall_msync.png)

## madvise

![](/images/posts/filesystem/syscall_madvise.png)

## fadvise

![](/images/posts/filesystem/syscall_fadvise.png)

## io_sumbit

![](/images/posts/filesystem/syscall_io_submit.png)

对于非`O_DIRECT`标记打开的文件，其内部逻辑与`write`流程基本一致，最终将数据拷贝到`pagecache`中，整个调用实际都是同步阻塞的。

对于`O_DIRECT`标记打开的文件，在文件系统层(`vfs/ext4`等)仍然是同步的，在一些文件系统日志、文件系统数据块与磁盘映射、[bio 请求队列满](https://zhuanlan.zhihu.com/p/100026388)等情况下，仍然会被同步阻塞。当经过文件系统层后，被封装成一个`bio`请求时，且 bio 请求队列未满时，该请求进入 bio 请求队列后即刻返回，从而形成一个异步写事件。

目前异步 IO 使用最多的是 `linux native aio`，不幸的是，其存在着诸多约束[^1]：

- 最大的限制无疑是仅支持 direct io。而 `O_DIRECT` 存在 bypass 缓存和 size 对齐等限制，直接影响了 aio 在很多场景的使用。而针对 buffered io，其表现为同步。
- 即使满足了所有异步 IO 的约束，有时候还是可能会被阻塞。例如，等待元数据 IO，或者等待 block 层 request 的分配等。
- 存在额外的拷贝开销，每个 IO 提交需要拷贝 64+8 字节(`iocb` 64 字节，`iocbpp` 指针 8 字节)，每个 IO 完成需要拷贝 32 字节，这 104 字节的拷贝在大量小 IO 的场景下影响很可观。同时，需要非常小心地使用完成事件以避免丢事件。
- IO 需要至少 2 个系统调用（submit + wait-for-completion)，这在 spectre/meltdown 开启的前提下性能下降非常严重。

# 参考

[^1]: [io_uring 新异步 IO 机制](https://mp.weixin.qq.com/s?__biz=MzUxNjE3MTcwMg==&mid=2247484448&idx=1&sn=29e791cf602b8614c9d288c1859407f7&chksm=f9aa36f9ceddbfef2a28f3593f69dffedb71a85342dc28764aa728a4ea3290bd86761abf6445&mpshare=1&scene=23&srcid=0708pSauI4c7ShmXbZrQlqUl&sharer_sharetime=1594175513526&sharer_shareid=0d25aaa0141cb845ff5dc57c13b23352%23rd)

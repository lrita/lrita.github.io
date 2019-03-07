---
layout: post
title: Linux Page Fault(缺页异常)
categories: [linux]
description: linux page fault page-fault
keywords: linux page fault page-fault
---

# 缺页异常

> Linux的内存管理采用分页管理，使用多级页表，动态地址转换机构与主存、辅存共同实现虚拟内存。即使每个进程有相同的逻辑地址空间，通过分页机制，相应的物理地址也是不同的，因此他们在物理上不会彼此重叠。
>
> 从内核角度来看，逻辑地址和物理地址都被划分成为固定大小的页面。每个合法的逻辑页面敲好处于一个物理页面中，方便MMU的地址转换。**当地址转换无法完成时(例如由于给定的逻辑地址不合法或由于逻辑页面没有对应的物理页面)，MMU将产生中断，向核心发出信号。Linux核心可以处理这种页面错误(`Page Fault`)问题**。
>
> MMU也负责增强内存保护，当一个应用程序试图在它的内存中队一个已标明是只读的页面进行写操作时，MMU也会产生中断错误，通知内核。在没有MMU的情况下，内核不能防止一个进程非法存取其他进程的内存空间。
>
> 每个进程都有一套自己的页目录与页表，其中页目录的基地址是关键，通过它才能查到逻辑所对应的物理地址。页目录的基地址是每个进程的私有资源，保存在该进程的`task_struct`对象的`mm_struct`结构变量`mm`中。
>
> 在进程切换时，CPU会把新进程的页目录基地址填入CPU的页目录寄存器，供MMU使用。当新进程有地址访问时，MMU会根据被访问地址的最高10位从页目录中找到对应的页表基地址，然后根据次高10位再从页表中找到对应的物理地址的页首，最后根据剩下的12位偏移量与页首地址找到逻辑地址对应的真正物理地址。
>
> 与直接访问物理内存不同，`Page Fault`过程大部分是由软件完成的，消耗时间比较久，所以是影响性能的一个关键指标。

# 触发原因

在linux中，用户空间的内存由`VMA`结构管理，具体细节可以参考[《linux的内存管理》](http://kernel.pursuitofcloud.org/531037)：

![](/images/posts/memory/linux-memory-vma.png)

当用户进程进程在一下情形时，会由`MMU`触发`Page Fault`中断：

![](/images/posts/memory/linux-memory-page-fault-reason.jpg)

1. 新申请的堆内存（`malloc`等），由于lazy机制，只建立页表而没有真实物理内存的映射，因此页表里的权限是`R`，发生`Page Fault`，在`Page Fault`回调中，linux会去申请一页内存，此时把页表权限设置为`R+W`。
2. 用户访问了非法的内存（例如引用野指针等），`MMU`就会触发`Page Fault`中断，回调中检查进程并没有对应这段内存的`VMA`，给用户进程发送`SIGSEGV`信号报段错误并终止。
3. 代码段在`VMA`中权限为`R+X`，如果程序中有野指针飞到此区域去写，则也会由`MMU`触发`Page Fault`中断，导致用户进程收到`SIGSEGV`信号。（另，`malloc`堆区在`VMA`中权限为`R+W`，如果程序的`PC`指针飞到此区域去执行，同样发生段错误。）
4. 在代码段区域运行执行操作时发生缺页，说明该段代码数据未从磁盘加载，则Linux申请一页内存，并从硬盘读取出代码段，此时产生了IO操作，为major主缺页。

# 触发流程

从上面的讲解很容易看出，`Page Fault`是一个由硬件中断触发的可以由软件逻辑纠正的错误。根据[《linux下的中断机制》](/2019/03/05/linux-interrupt-and-trap/)中的叙述，一个`Page Fault`的触发流程为：

![](/images/posts/memory/page-fault-interrupt.png)

# 参考

* [Linux任督二脉之内存管理1](/images/posts/memory/Linux任督二脉之内存管理1.pdf)
* [Linux任督二脉之内存管理2](/images/posts/memory/Linux任督二脉之内存管理2.pdf)
* [Linux任督二脉之内存管理3](/images/posts/memory/Linux任督二脉之内存管理3.pdf)
* [Linux任督二脉之内存管理4](/images/posts/memory/Linux任督二脉之内存管理4.pdf)
* [Linux任督二脉之内存管理5](/images/posts/memory/Linux任督二脉之内存管理5.pdf)
* [Linux任督二脉之内存管理6](/images/posts/memory/Linux任督二脉之内存管理6.pdf)

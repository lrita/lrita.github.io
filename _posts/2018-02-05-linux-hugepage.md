---
layout: post
title: Linux HugePages与Transparent HugePages
categories: [system, linux, memory]
description: system linux Transparent HugePages
keywords: system linux Transparent HugePages
---

# HugePages与Transparent HugePages
Linux下的大页分为两种类型[^1]：标准大页（`Huge Pages`）和透明大页（`Transparent Huge Pages`）。
`Huge Pages`有时候也翻译成大页/标准大页/传统大页，它们都是`Huge Pages`的不同中文翻译名而已，顺带提一
下这个，免得有人被这些名词给混淆、误导了。`Huge Pages`是从Linux Kernel 2.6后被引入的。目的是使用更大
的内存页面（memory page size）以适应越来越大的系统内存，让操作系统可以支持现代硬件架构的大页面容量功能。
透明大页（`Transparent Huge Pages`）缩写为THP，这个是RHEL 6（其它分支版本SUSE Linux Enterprise Server
11, and Oracle Linux 6 with earlier releases of Oracle Linux Unbreakable Enterprise Kernel 2 (UEK2)）
开始引入的一个功能。具体可以参考官方文档。

这两者有啥区别呢？

这两者的区别在于大页的分配机制，标准大页管理是预分配的方式，而透明大页管理则是动态分配的方式。相信有不
少人将`Huge Pages`和`Transparent Huge Pages`混为一谈。目前透明大页与传统`HugePages`联用会出现一些问题，
导致性能问题和系统重启。Oracle 建议禁用透明大页（`Transparent Huge Pages`）。在 Oracle Linux 6.5 版中，
已删除透明`HugePages`的支持。

标准大页（HuagePages）英文介绍：
```
HugePages is a feature integrated into the Linux kernel with release 2.6. It is a method to have larger
pages where it is useful for working with very large memory. It can be useful for both 32-bit and 64-bit
configurations. HugePage sizes vary from 2MB to 256MB, depending on the kernel version and the hardware
architecture. For Oracle Databases, using HugePages reduces the operating system maintenance of page 
states, and increases TLB (Translation Lookaside Buffer) hit ratio.
```

注意:
1. HugePages size的大小默认为2M，这个也是可以调整的。区间范围为2MB to 256MB。
2. 同时HuagePages是不可以被SWAP到磁盘的。
3. Hugepages在`/proc/meminfo`中是被独立统计的，与其它统计项不重叠，既不计入进程的`RSS/PSS`中，又不计入
`LRU Active/Inactive`，也不会计入`cache/buffer`。如果进程使用了Hugepages，它的`RSS/PSS`不会增加。[^6]

RHEL的官方文档对传统大页（`Huge Pages`）和透明大页（`Transparent Huge Pages`）这两者的描述[如下](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Performance_Tuning_Guide/s-memory-transhuge.html)
```
Huge pages can be difficult to manage manually, and often require significant changes to code in order
to be used effectively. As such, Red Hat Enterprise Linux 6 also implemented the use of transparent huge
pages(THP). THP is an abstraction layer that automates most aspects of creating, managing, and using huge
pages.

THP hides much of the complexity in using huge pages from system administrators and developers. As the
goal of THP is improving performance, its developers (both from the community and Red Hat) have tested
and optimized THP across a wide range of systems, configurations, applications, and workloads. This
allows the default settings of THP to improve the performance of most system configurations. However,
THP is not recommended for database workloads.
```

注：
1. THP 目前只能映射异步内存区域，比如堆和栈空间。THP是可以被SWAP到磁盘的。
2. `/proc/meminfo`里`AnonHugePages`统计的是Transparent HugePages (THP)。它与`/proc/meminfo`的其他统计项
有重叠，首先它被包含在`AnonPages`之中，而且在`/proc/<pid>/smaps`中也有单个进程的统计，与进程的`RSS/PSS`
是有重叠的，如果用户进程用到了THP，进程的`RSS/PSS`也会相应增加，这与Hugepages是不同的。[^6]

# 为什么要引入大页内存

### 减少TLB Miss
Linux系统中对于用户态程序可见的是`Virtual Address`，每一个程序都拥有自己进程的内存空间。而进程的每一个内
存的操作，都有可能被转化为对一个物理内存的操作。因此在程序运行过程中，需要将`虚拟内存`转换为`物理内存`，
因此有了一个`虚拟内存`与`物理内存`的关系表，Linux就是用`Page Table`来管理内存[^15]，每一次内存的操作都需
要一次查表的转换的操作。为了提供高效的系统，现代CPU中就出现了`TLB(Translation Lookaside Buffer) Cache`用
于缓存少量热点内存地址的映射关系，帮助系统来完成内存地址的转换。然而由于制造成本和工艺的限制，响应时间需
要控制在CPU Cycle级别的Cache容量只能存储几十个对象。那么`TLB Cache`在应对大量热点数据`Virual Address`转
换的时候就显得捉襟见肘了。通常CPU的`TLB Cache`只有64个元素，可以通过`x86info -c`命令来查看（如果服务器有
多个CPU，则能看到有多个`TLB Cache`）。这样在默认内存页为4K时，只能缓存`4K*64 = 256K`的热点数据的内存地址。
但是现在的服务器动辄几百G的内存，一个进程就可能用掉10G+的内存，如果程序的热点数据比较分散，可想而知，会
产生大量的`TLB Miss`。

随着现在硬件的升级，服务器的物理存储越来越大，动辄几百G内存的服务器，应用程序使用的内存也越来越多，特别是
存储类型和缓存类型的。从系统层面增加一个`TLB Cache`entry所能对应的物理内存大小，从而增加`TLB Cache`所能涵
盖的热点内存数据量。假设我们把`Linux Page Size`增加到16M，那么同样一个容纳64个元素的`TLB Cache`就能顾及
`64*16M = 1G`的内存热点数据[^2]。这样就很大程度上减小了`TLB Miss`的概率。

### 减少内核管理内存消耗的资源
同时Linux采用分页的内存管理机制。当内存的每个页(page)很小时，内核需要耗费大量内存来维护内存的页表结构。我
们可以通过命令来查看`PageTables`的数量：

```shell
>$ grep PageTables /proc/meminfo
PageTables:      1573080 kB
```

当我们提高每个内存页的大小后，相同内存下，需要维护的页的数量就大大减小。减少了资源消耗。每个页表条目可以高
达64字节，如果我们50GB的RAM保存在页表（page table）当中，那么页表（page table）大小大约为800MB，实际上对
于lowmem来说，考虑到lowmem的其他用途，880MB大小是不合适的（在2.4内核当中，page tabel在低于2.6的内核当中不
是必须的），lowmem中通过256MB的hugepages访问95％的内存时，可以使用大约40MB的页表[^3]。

### 减少页表查询的耗时
缩小`PageTables`大小的同时也就减少了查表的耗时。当`TLB Miss`之后，就会去查询页表，我们不可能保证每次都能命
中`TLB Cache`的，减少页表查询的耗时，就加速了程序访问虚拟内存的速度，从而提高整体性能。

# 查看是否开启HugePages与Transparent HugePages

### 查看HugePages的配置

```shell
# 查看标准大页（HugePages)的页面大小：
>$ grep Hugepagesize /proc/meminfo
Hugepagesize:     2048 kB

# 确认HugePages是否配置、并在使用的方法：
>$ cat /proc/sys/vm/nr_hugepages
0                                    # 0 意味着没有设置使用

>$ grep -i HugePages_Total /proc/meminfo
HugePages_Total:     0               # 0 意味着没有设置使用
```

### 启用HugePages
使用Hugepages有三种方式：

1. mount一个特殊的`hugetlbfs`文件系统[^4]，在上面创建文件，然后用`mmap()`进行访问，如果要用`read()`访问
也是可以的，但是`write()`不行。为了方便，可以直接使用[libhugetlbfs](https://github.com/libhugetlbfs/libhugetlbfs)，
其完成了这一系列操作，同时对`malloc/free`进行了重载，使用户可以直接在`hugetlbfs`上分配内存。
2. 通过`shmget/shmat`也可以使用Hugepages，调用`shmget`申请共享内存时要加上`SHM_HUGETLB`标志。
3. 通过`mmap()`，调用时指定`MAP_HUGETLB`标志也可以使用Huagepages[^5]。

### 查看Transparent Hugepages开启[^7]
```shell
>$ cat /sys/kernel/mm/transparent_hugepage/enabled
[always] madvise never
```
* `[always]` 表示已经开启
* `[never]` 表示透明大页禁用
* `[madvise]` 表示只在`MADV_HUGEPAGE`标志的VMA中使用THP

同时也可以在内核启动参数进行配置：

* `"transparent_hugepage=always"`
* `"transparent_hugepage=madvise"`
* `"transparent_hugepage=never"`

### 修改Transparent Hugepages配置[^7]
`THP`的开启、关闭只影响修改以后的程序行为，因此当修改`THP`配置后，应该重启相关程序，使其使用新的配置。

```shell
echo always > /sys/kernel/mm/transparent_hugepage/enabled
echo madvise > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

为了为用户提供更多的`THP`使用，内核会对内存进行碎片整理，将连续的普通`page`合并为`THP`。
当然碎片整理也有开关可以控制：
* `always`：意思是当用户分配`THP`内存时，当没有足够`THP`内存可用时，请求会阻塞住，然后进行内存回收、
压缩，然后尽最大努力分配出一个`THP`。使用这个选项，显然会给程序带来不确定的延时。
* `defer`：Linux4.6开始支持该项。意思是程序会唤醒内核进程`kswapd`异步回收内存，同时唤醒`kcompactd`异步压
缩合并内存，从而避免了当分配THP时，连续内存不足2m时，*同步*压缩内存带来的进程停顿[^15]。
* `defer+madvise`：Linux4.11开始支持该项。意思是当`THP`内存不足时，用户请求分配`THP`内存时会直接回收、合
并内存，就像`always`选项一样，但是只针对调用`madvise(MADV_HUGEPAGE)`的内存区域。其他区域的内存会像`defer`
配置一样运作。
* `madvise`：当用户分配`THP`内存失败时，只对调用`madvise(MADV_HUGEPAGE)`的内存区域进行内存回收、合并。
* `never`：关闭用户分配`THP`内存失败时的回收机制。

```shell
echo always > /sys/kernel/mm/transparent_hugepage/defrag
echo defer > /sys/kernel/mm/transparent_hugepage/defrag
echo defer+madvise > /sys/kernel/mm/transparent_hugepage/defrag
echo madvise > /sys/kernel/mm/transparent_hugepage/defrag
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```

`huge zero page`是内核为`THP`读请求时的一个优化[^8]，可以决定是否开启：

```shell
echo 0 > /sys/kernel/mm/transparent_hugepage/use_zero_page
echo 1 > /sys/kernel/mm/transparent_hugepage/use_zero_page
```

当`THP`被设置为`always`或者`madvise`时，`khugepaged`会自动开启，当`THP`被设置为`never`时，`khugepaged`
会被自动关闭。`khugepaged`周期性运行以回收、合并内存。用户不想在分配内存时回收、合并内存时，至少应该
开启`khugepaged`来回收、合并内存。当然`khugepaged`也可以被关闭：

```shell
echo 0 > /sys/kernel/mm/transparent_hugepage/khugepaged/defrag
echo 1 > /sys/kernel/mm/transparent_hugepage/khugepaged/defrag
```

同时也可以通过`/sys/kernel/mm/transparent_hugepage/khugepaged/pages_to_scan`控制`khugepaged`每次扫描多
少个`page`。

通过`/sys/kernel/mm/transparent_hugepage/khugepaged/scan_sleep_millisecs`控制`khugepaged`每次扫描的间隔，
单位是毫秒。当其被设置为0是，会使一个CPU核使用率达到100%。

通过`/sys/kernel/mm/transparent_hugepage/khugepaged/alloc_sleep_millisecs`控制`khugepaged`内部每次分配
失败时SLEEP多久再进行一下次尝试，通常不需要调整。

还有一些其他参数，就不一一细讲了。

# Transparent HugePages的缺点
当然使用`Transparent HugePages`也有一些潜在问题：

### 内存额外开销增加
当内存的一个`page`增加到2MB时，即使我们使用很小的一点内存时，也会消耗一个`page`，造成2MB的内存开销。
这样是一个`page`4k时的512倍。当然在现代服务器上，可以忽略不计。有时也会也会造成严重的影响，如果内存
使用的比较琐碎，造成大量2MB的`page`都无法真正释放，可能会造成进程使用内存过量，被`OOM Killer`干掉[^9] [^13]。

### 暂停以及CPU开销
1. 当`Transparent HugePages`的2MB的`page`被SWAP到磁盘时，需要被重新划分为4K的`page`，这时需要额外的
CPU开销，以及更高的IO延时。当然，在现代高能性服务器上，通常会选择禁用SWAP。
2. 通常Linux内核还会有一个叫做`khugepaged`的进程，它会一直扫描所有进程占用的内存，在可能的情况下会把
4K`page`交换为`Transparent HugePages`，在这个过程中，对于操作的内存的各种分配活动都需要各种内存锁，直
接影响程序的内存访问性能，并且，这个过程对于应用是透明的，在应用层面不可控制，对于专门为4K`page`优化
的程序来说，可能会造成随机的性能下降现象。幸好的是，我们可以通过`echo 0 > /sys/kernel/mm/transparent_hugepage/khugepaged/defrag`
和`echo never > /sys/kernel/mm/transparent_hugepage/defrag`来关闭这个功能。

# 是否需要开启`HugePages`/`Transparent HugePages`
既然开启`HugePages`/`Transparent HugePages`又有优点，同时又可能带来不确定的缺点。而且大多人对`HugePages`/
`Transparent HugePages`都带有负面看法，建议我们关闭`HugePages`/`Transparent HugePages`[^10] [^11] [^12]。
那我们到底是否需要开启`HugePages`或者`Transparent HugePages`么？

这个问题当然没有确定的答案，因此我们需要先各自的项目中进行测试、测量，拿数据说话，看开启`HugePages`或者
`Transparent HugePages`是否在该项目中是否能带来好处。

## 简单的方法
最简单的方法就是，在项目中分别开启、关闭`HugePages`/`Transparent HugePages`的情况下，进行压测，来判断该
项目能否受益于`HugePages`/`Transparent HugePages`。

如果没有明显的受益时，使用的是Linux4.6之前内核的场景最好还是关闭`HugePages`/`Transparent HugePages`，避
免其带来的不确定性。当系统内核高于Linux4.6，可以尝试启动`Transparent HugePages`，同时将`defrag`调为`defer`。

## 复杂的方法
复杂的方法当然就是刨根问底儿，通过各种工具来分析程序的实际运行过程，看`HugePages`/`Transparent HugePages`
是否对程序带来正收益。

#### 测量TLB MISS
我们可以使用`perf`来分析开启/关闭`HugePages`/`Transparent HugePages`时，TLB miss的情况是否有明显改变[^16]:

dTLB-load-misses(dTLB是数据转换后援缓存)和iTLB-load-misses(iTLB是指令转换后援缓存)等指标值，load表示读指
令，store表示写操作：
```
# 每秒钟输出一次dTLB情况
>$ perf stat -e dTLB-loads,dTLB-load-misses,dTLB-stores,dTLB-store-misses -a -I 1000
#           time             counts unit events
     1.000458338      2,404,619,383      dTLB-loads                [100.00%]
     1.000458338         12,025,384      dTLB-load-misses          [100.00%]
     1.000458338      1,429,855,652      dTLB-stores               [100.00%]
     1.000458338          2,294,918      dTLB-store-misses
     2.001339288      2,406,698,800      dTLB-loads
     2.001339288         11,644,332      dTLB-load-misses
     2.001339288      1,476,477,700      dTLB-stores
     2.001339288          4,200,652      dTLB-store-misses

# 查看指定进程的dTLB情况
>$ perf stat -e dTLB-loads,dTLB-load-misses,dTLB-stores,dTLB-store-misses -a -p <pid>
# CTRL-C退出，可以看到dTLB命中情况
 Performance counter stats for process id '4577':

     4,579,026,997      dTLB-loads
        22,869,795      dTLB-load-misses          #    0.50% of all dTLB cache hits
     2,773,838,918      dTLB-stores
         6,483,562      dTLB-store-misses

       2.113034900 seconds time elapsed

# 每秒钟输出一次iTLB情况
>$ perf stat -e iTLB-load,iTLB-load-misses -a -I 1000
#           time             counts unit events
     1.000272672         97,787,479      iTLB-load                 [100.00%]
     1.000272672          4,014,902      iTLB-load-misses
     2.000750667         92,962,955      iTLB-load
     2.000750667          3,707,801      iTLB-load-misses

# 查看指定进程的iTLB情况
>$ perf stat -e iTLB-load,iTLB-load-misses -a -p 4577
# CTRL-C退出，可以看到iTLB命中情况
 Performance counter stats for process id '4577':

     1,794,122,924      iTLB-load                                                    [100.00%]
        71,716,505      iTLB-load-misses          #    4.00% of all iTLB cache hits

      19.078375072 seconds time elapsed
```

#### 测量内核函数
同时，我们还可以使用`SystemTap`来测量内核函数，来判断THP等是否会带来影响[^16]：

首先我们感兴趣的函数是[\_\_alloc\_pages\_slowpath](http://elixir.free-electrons.com/linux/v3.10/source/mm/page_alloc.c#L2391)，
该函数会在我们分配内存时，没有连续2m内存也可用时被调用，其会调用内存页压缩/回收逻辑，可能会引起进程的停
顿。

第二个我们感兴趣的函数时[khugepaged\_scan\_mm\_slot](http://elixir.free-electrons.com/linux/v3.10/source/mm/huge_memory.c#L2466)，
其会被内核线程`khugepaged`调用，它会扫描内存，将常规页合并成hugepage，这个过程中会对内存页进行锁定，如果
其耗时较长的话，也可能引起程序的停顿。

因此我们可以使用一下脚本进行测量这2个函数的耗时：
```
#! /usr/bin/env stap
global start, intervals

probe $1 { start[tid()] = gettimeofday_us() }
probe $1.return
{
  t = gettimeofday_us()
  old_t = start[tid()]
  if (old_t) intervals <<< t - old_t
  delete start[tid()]
}

probe timer.ms($2)
{
    if (@count(intervals) > 0)
    {
        printf("%-25s:\n min:%dus avg:%dus max:%dus count:%d \n", tz_ctime(gettimeofday_s()),
             @min(intervals), @avg(intervals), @max(intervals), @count(intervals))
        print(@hist_log(intervals));
    }
}
```

然后执行：
```sh
>$ ./func_time_stats.stp 'kernel.function("__alloc_pages_slowpath")' 1000
```



[^1]: [Linux传统Huge Pages与Transparent Huge Pages再次学习总结](http://www.cnblogs.com/kerrycode/p/7760026.html)
[^2]: [Huge Page 是否是拯救性能的万能良药？](http://cenalulu.github.io/linux/huge-page-on-numa/)
[^3]: [Myths and common misconceptions about (transparent) huge pages for Oracle databases](https://blogs.sap.com/2013/06/17/oracle-myths-and-common-misconceptions-about-transparent-huge-pages-for-oracle-databases-on-linux-uncovered/)
[^4]: [kernel documentation hugetlbpage](https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt)
[^5]: [Linux下试验大页面映射](http://blog.yufeng.info/archives/2118)
[^6]: [/PROC/MEMINFO之谜](http://linuxperf.com/?p=142)
[^7]: [Transparent Hugepage Support](https://www.kernel.org/doc/Documentation/vm/transhuge.txt)
[^8]: [Adding a huge zero page](https://lwn.net/Articles/517465/)
[^9]: [Transparent Huge Pages and Alternative Memory Allocators: A Cautionary Tale](https://blog.digitalocean.com/transparent-huge-pages-and-alternative-memory-allocators/)
[^10]: [Performance Issues with Transparent Huge Pages](https://blogs.oracle.com/linux/performance-issues-with-transparent-huge-pages-thp)
[^11]: [MemSQL: Disable Transparent Huge Pages](https://help.memsql.com/hc/en-us/articles/115002948663-Disable-Transparent-Huge-Pages)
[^12]: [mongodb: Disable Transparent Huge Pages](https://docs.mongodb.com/manual/tutorial/transparent-huge-pages/)
[^13]: [Linux Transparent Huge Pages, JEMalloc and NuoDB](https://www.nuodb.com/techblog/linux-transparent-huge-pages-jemalloc-and-nuodb)
[^14]: [failing to understand the issues with transparent huge paging](https://groups.google.com/forum/#!topic/mechanical-sympathy/sljzehnCNZU)
[^15]: [20 years of Linux Virtual Memory](/images/posts/memory/VM.pdf)
[^16]: [Transparent Hugepages: measuring the performance impact](https://alexandrnikitin.github.io/blog/transparent-hugepages-measuring-the-performance-impact/)

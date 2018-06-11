---
layout: post
title: Linux Kernel VM 参数
categories: [linux]
description: linux sysctl
keywords: linux sysctl
---

在linux内核中有许多参数可以有用户进行配置。可以通过`sysctl -a`命令来查看。本文主要讲一些与内存相关的参数，会根据不断需要进行补充。

关于内存相关的参数可以通过命令`sysctl -a | grep "vm\."`进行查看，其中各个参数在官方文档[^1]中也有详细描述。

# pagecache
```
# 当系统脏页的比例或者所占内存数量超过 dirty_background_ratio(百分数)/dirty_background_bytes(字节) 设定的
# 阈值时，启动相关内核线程(pdflush/flush/kdmflush)开始将脏页写入磁盘。
# 如果该值过大，同时又有进程大量写磁盘(未使用DIRECT_IO)时，会使pagecache占用对应比例的系统内存。
# 注意：dirty_background_bytes参数和dirty_background_ratio参数是相对的，只能指定其中一个。
#      当其中一个参数文件被写入时，会立即开始计算脏页限制，并且会将另一个参数的值清零。
vm.dirty_background_bytes = 0
vm.dirty_background_ratio = 10

# 当系统pagecache的脏页达到系统内存 dirty_ratio(百分数)/dirty_bytes字节)阈值时，系统就会阻塞新的写请求，
# 直到脏页被回写到磁盘，此值过低时，遇到写入突增时，会造成短时间内pagecache脏页快速上升，造成写请求耗时增加。
# 但是此值也不能设置的太高，当该值太高时，会造成内核flush脏页时，超过内核限制的120s导致进程挂起或中断。
# 注意：dirty_bytes 和 dirty_ratio 是相对的，只能指定其中一个。
# 当其中一个参数文件被写入时，会立即开始计算脏页限制，并且会将另一个参数的值清零。
vm.dirty_bytes = 0
vm.dirty_ratio = 20

# 声明Linux内核写缓冲区里面的数据多“旧”了之后，pdflush/flush/kdmflush进程就开始考虑写到磁盘中去。
# 单位是 1/100秒。缺省是3000，也就是30秒的数据就算旧了，将会刷新磁盘。
# 对于特别重载的写操作来说，这个值适当缩小也是好的，但也不能缩小太多，因为缩小太多也会导致IO提高太快。
vm.dirty_expire_centisecs = 100

# 表示内核线程(pdflush/flush/kdmflush)多久唤醒一次来检查是否需要将cache中的数据写入磁盘，单位1/100秒。
# 如果设置为0，则禁止周期性地唤醒回写线程。
vm.dirty_writeback_centisecs = 100

# 该文件表示内核回收用于directory和inode cache内存的倾向
# 缺省值100表示内核将根据pagecache和swapcache，把directory和inode cache保持在一个合理的百分比；
# 降低该值低于100，将导致内核倾向于保留directory和inode cache；
# 增加该值超过100，将导致内核倾向于回收directory和inode cache。
vm.vfs_cache_pressure = 100

# 释放cache，该参数每修改一次，触发一次释放操作。
# 可以设置的值分别为1、2、3。它们所表示的含义为：
#       1 = 表示清除pagecache
#       2 = 表示清除回收slab分配器中的对象（包括目录项缓存和inode缓存）。
#           slab分配器是内核中管理内存的一种机制，其中很多缓存数据实现都是用的pagecache。
#       3 = 表示清除pagecache和slab分配器中的缓存对象。
vm.drop_caches = 0
```

# OOM
```
# 启动OOM信息打印
# 当OOM-KILLER启动时，将会将各进程的
# pid, uid, tgid, vm size, rss, pgtables_bytes, swapents, oom_score_adj, 名称
# 等信息打印到dmesg里，管理员事后可以在dmesg信息中查看，定位问题。
vm.oom_dump_tasks = 1

# 控制是否杀死触发OOM的进程。
# 设置为0时，OOM-KILLER会扫描进程列表，选择一个进程来杀死。通常都会选择消耗内存内存最多
# 的进程，杀死这样的进程后可以释放大量的内存。
# 设置为非0时，OOM-KILLER，只会简单地将触发OOM的进程杀死，避免遍历进程列表，减少了决策开销。
# 通常建议设置为0。
# 为了避免特殊的进程被OOM-KILLER干掉，可以修改echo -1000 > /proc/$pid/oom_score_adj，禁止OOM-KILLER杀掉该进程
vm.oom_kill_allocating_task = 0
```

# SWAP
```
# 控制是否允许超额申请内存。可以设置的值有0/1/2
# 当用户空间请求更多的的内存时：
#   0 = 内核尝试估算出剩余可用的内存
#   1 = 内核假装总是足够，直到用完为止
#   2 = 内核会使用一个决不过量使用内存的算法，即系统整个内存地址空间不能超过SWAP+50%的物理RAM值，
#       50%参数的设定是在overcommit_ratio中设定
vm.overcommit_memory = 0

# 当 overcommit_memory = 2 时，系统内存空间不会超过 SWAP + (overcommit_ratio% * 物理RAM值)
# 该配置与overcommit_kbytes只有一个能设置生效
vm.overcommit_ratio = 50

# 当 overcommit_memory = 2 时，系统内存空间不会超过 SWAP + overcommit_kbytes
# 该配置与overcommit_ratio只有一个能设置生效
vm.overcommit_kbytes = 0

# 控制一次从SWAP中连续读取多少内存页，当使用SWAP时，较低的该值，能减少一次读取SWAP的数据，而从减少延时。
# 注意：这个配置不是指多少页，而是2的多少次幂，当值为0时，是1页，值为1时，是2页，默认值是3，是8页。
vm.page-cluster = 3

# 控制内核在OOM时是否panic
# 设置为0时，内核会启动OOM-KILLER杀死内存占用过多的进程。通常杀死内存占用最多的进程，系统就会恢复。
# 设置为1时，内核会panic。如果一个进程通过内存策略或进程绑定限制了可以使用的节点，并且这些节点的内存已经
# 耗尽，OOM-KILLER可能会杀死一个进程来释放内存。在这种情况下，内核不会panic，因为其他节点的内存可能还有
# 空闲，这意味着整个系统的内存状 况还没有处于崩溃状态。
vm.panic_on_oom = 0

# VM统计信息更新的时间间隔，默认值1s
vm.stat_interval = 1

# 控制是否使用SWAP分区，以及使用的比例。
# 设置的值越大，内核会越倾向于使用SWAP。
# 设置为0，内核只有在空闲页的和文件相关的内存页数量小于内存区域的高水位线时才开始SWAP，并不是像其他文章所
# 说的禁用SWAP。
vm.swappiness = 0

# 水位线等级系数，这个系数控制了kswapd进程的激进程度，控制了kswapd进程从唤醒到休眠，需要给系统（NUMA节点）
# 是释放出多少内存。
# 该值的单位是万分几。默认值是10，意思是0.1%的系统内存（NUMA节点内存）。该值的最大值是1000，意思是10%的系
# 统内存（NUMA节点内存）。
vm.watermark_scale_factor = 10

# 该参数只有在启用CONFIG_NUMA选项时才有效。
# 用来控制在NUMA节点上内存区域OOM时，如何来回收内存。
# 如果设置为0，则禁止内存域回收，从系统中其他内存域或NUMA节点来分配内存。0是默认值。
# 该值还以用1/2/4三个数或运算组合起来(类似文件权限一样)，来表示多种组合策略：
#           1      = 启用内存区域回收
#           2      = 刷脏页回收内存
#           4      = 通过SWAP回收内存
# 对于一般的服务应该设置为0，因为它们通常能从cache中获益，将数据cache住远比数据NUMA节点内存本地化重要。
# 当用户的服务确定能很好的利用NUMA节点内存本地化带来的好处时，再启动该参数。
vm.zone_reclaim_mode = 0
```

# hugepages
```
# hugepage池大小，单位是page，默认是0，等于关闭hugepage功能。
# 设置大于0的值，开启hugepage功能，开启后，内核会预分配hugepage内存到池子中去，然后其他程序就
# 可以使用hugepage功能了。
# 也可以通过"cat /proc/meminfo |grep HugePages_Total"来查看到。
vm.nr_hugepages = 0

# 设置hugepage可以超额使用的page，hugepage池的实际大小为 nr_hugepages + nr_overcommit_hugepages，
# 当vm.nr_overcommit_hugepages > 0时，可能会使用到SWAP。
vm.nr_overcommit_hugepages = 0
```

# numa
```
# 设置numa运行时的数据统计，当内存紧张时，可以设置为0来减少统计计数的精度。
vm.numa_stat = 1

# 该配置已经废弃，除了"Node"其他设置都会失败。
vm.numa_zonelist_order = Node

# 该参数只在NUMA内核中才有效，精通再调整
# 如果一个内存区域中可以回收的slab页面所占的百分比（应该是相对于当前内存域的所有页面）超过
# min_slab_ratio，在回收区的slabs会被回收。这样可以确保即使在很少执行全局回收的NUMA系统中，
# slab的增长也是可控的。
vm.min_slab_ratio = 5

# 该参数只在NUMA内核中才有效，精通再调整
# 只有在当前内存域中处于zone_reclaim_mode允许回收状态的内存页所占的百分比超过min_unmapped_ratio
# 时，内存域才会执行回收操作。
vm.min_unmapped_ratio = 1
```

# 不太需要关心的参数
```
# 给admin预留的内存，用于紧急登录、执行命令等，避免系统内存紧张时，admin用户无法登录，无法查看问题。
vm.admin_reserve_kbytes = 8192

# 当overcommit_memory设置为2，"never overcommit"模式时，预留"3%的进程内存大小"和"user_reserve_kbytes"
# 两者之间最小的数量的内存。
# 该值的作用是，避免内核将剩余空闲内存一次性分配给一个新启动的进程，造成系统空闲内存耗尽。
# 将该值减少到0，将会允许内核将剩余空闲内存一次性分配给一个申请内存的进程，但是可能带来负面影响。
vm.user_reserve_kbytes = 131072

# 非0值开启block I/O debug，详情Documentation/laptops/laptop-mode.txt
vm.block_dump = 0
# 开启laptop模式，详情Documentation/laptops/laptop-mode.txt
vm.laptop_mode = 0

# 该参数影响内核分配高端内存时，内存不足时是采用压缩还是回收策略。该值接近0时，倾向于缺少内存时直接
# 失败，该值接近1000，倾向于压缩，当碎片过多时，会导致失败。
vm.extfrag_threshold = 500

# 开启旧的mmap布局
vm.legacy_va_layout = 0

# 低端内存主要用于DMA、SYSCALL等，设置预留比例。如非精通，不要调节。
vm.lowmem_reserve_ratio = 256    256    32

# 每个进程内存拥有的VMA(虚拟内存区域)的数量。虚拟内存区域是一个连续的虚拟地址空间区域。在进程的生命
# 周期中，每当程序尝试在内存中映射文件，链接到共享内存段，或者分配堆空间的时候，这些区域将被创建。
# 进程加载的动态库、分配的内存、mmap的内存都会增加VMA的数量。通常一个进程会有小于1K个VMA，如果进程有
# 特殊逻辑，可能会超过该限制。
# 调优这个值将限制进程可拥有VMA的数量。限制一个进程拥有VMA的总数可能导致应用程序出错，因为当进程达到
# 了VMA上线但又只能释放少量的内存给其他的内核进程使用时，操作系统会抛出内存不足的错误。如果你的操作系
# 统在NORMAL区域仅占用少量的内存，那么调低这个值可以帮助释放内存给内核用。
# https://www.elastic.co/guide/en/elasticsearch/reference/current/vm-max-map-count.html
# 可以使用命令 cat /proc/${pid}/maps 来查看指定进程拥有的VMA。
vm.max_map_count = 65530

# 控制硬件检测出内存错误时，如何处理进程。
# 1 杀死全部进程
# 0 解除内存错误页对应的映射，杀死尝试访问这些内存的进程
vm.memory_failure_early_kill = 0

# 启动内存故障恢复
vm.memory_failure_recovery = 1

# 内核最小free内存阈值，用于计算内存水位线，使得各低端内存区域按照一定比例，预留内存。
# 预留内存避免内核，在紧急时刻/内部逻辑分配不出内存而导致死锁等问题。
# 也不可调的过高，否则容易导致系统判定内存不足而出发OOM逻辑。
vm.min_free_kbytes = 90112

# 该参数定义了用户进程能够映射的最低内存地址。由于最开始的几个内存页面用于处理内核空引用错误，
# 这些页面不允许写入。
# 该参数的默认值是0，表示安全模块不需要强制保护最开始的页面。
# 如果设置为64K，可以保证大多数的程序运行正常，避免比较隐蔽的内核BUG。
vm.mmap_min_addr = 4096
```

[^1]: [Documentation/sysctl/vm.txt](https://www.kernel.org/doc/Documentation/sysctl/vm.txt)

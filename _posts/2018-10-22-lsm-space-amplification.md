---
layout: post
title: LSM-TREE 存储结构的空间放大
categories: [database]
description: database lsm space amplification
keywords: lsm 空间放大
---

`log structured merge tree`[^1]已经是现代数据库常用的一种数据结构，其优点就是能够将全部操作都转化为写入，从而使磁盘的连续写入优势发挥最大等。但是会带来`写放大`和`空间放大`。但是`读放大`、`写放大`和`空间放大`是一对矛盾体[^2] [^3] [^4]。因此，由于有着不同的取舍，就有了不同的`compaction`算法。使用不同的`compaction`算法，导致的`空间放大`会有不同，但是BigTable, HBase、Cassandra、LevelDB、RocksDB等都使用了`leveled compaction`[^5]，则我们根据`leveled compaction`算法来谈谈`空间放大`。

_`leveled compaction`拥有最小的`空间放大`，但是带来很大的`写放大`和`读放大`。_

# 空间放大产生的原因

根据`LSM`的定义，其将所以的操作都转化为一个新的`op`写入存储结构，增删改查都是一个对应的`op`，当查找对应的KEY时，就查找对应的最新的`op`，如果未找到或者最新的`op`是“删除”，则该KEY不存在，否则返回最新`op`对应的值。因此一个KEY在存储中会对应多个`op`，从而导致实际使用的磁盘空间大于存储中的数据量大小，也就是`空间放大`。

![](/images/posts/database/level_structure_0.png)

其中省略一些细节，为了节省存储空间，将这些`op`按层级组织起来，然后从上到下，每层的存储容量依次增加。通常规定$$L _ {n+1}$$的容量是$$L _ n$$的`增长系数`倍，这个值通常是`10`：

![](/images/posts/database/level_targets_1.png)

按照`op`的写入时间顺序，逐层安排。最新写入的在内存中，然后到达`写缓存`容量后，生成一个存储文件放置到L0层，当每一层的文件到达给层容量限制后，就会开启一个`compaction`工作：

![](/images/posts/database/pre_l0_compaction.png)

当L1的文件总大小超过该层限制后，会继续进行`compaction`工作，会挑选适当的文件合并入L2层：

![](/images/posts/database/pre_l1_compaction.png)

如果`compaction`后，使得L2层的总大小超过该层的限制，则继续向下`compaction`：

![](/images/posts/database/post_l1_compaction.png)

以此类推，最终将每层的文件大小控制在要求内。

那么我们就可以分析其中产生的空间放大：

首先，我们**忽略`WAL`占用的额外空间**，因为这个占用的磁盘空间大小是一个常数，通常这个值远小于磁盘容量，所以不太需要关注。

然后，然后分析每个场景的`空间放大`：
1. 每层的数据都不重叠，很明显此时`空间放大`为1
2. 每层的数据都重叠，而且每层数据都填充慢，此时我们按照`增长系数 = 10`，来计算，总数据容量为最后一层的大小，则`空间放大`为`1.111...`
3. 每层的数据都重叠，但是最后一层数据没有被填充慢，只略微比上一层大一点点，此时我们按照`增长系数 = 10`，来计算，总数据容量为最后一层的大小，则`空间放大`为`2.011...`[^6]

# 实验
在`ScyllaDB`[^3]进行了两个实验，测试不同场景下`leveled compaction`的`空间放大`的实际表现

## 连续写入新KEY
在这个场景中，连续写入不重复的新KEY。可以看出，`空间放大`几乎为**1**。

![](/images/posts/database/leveled-compaction-2.png)

## 反复修改KEY
在这个场景中，反复修改1.2 GB数据，重复15次。可以看出，`空间放大`的范围为**1.1-2**。

![](/images/posts/database/leveled-compaction-3.png)

## 附录
`rocksdb`目前支持4中`compaction`算法，在`Options::compaction_style`可以选择：
* [`kCompactionStyleLevel`](https://github.com/facebook/rocksdb/wiki/Leveled-Compaction)，如上文所述。
* [`kCompactionStyleUniversal`](https://github.com/facebook/rocksdb/wiki/Universal-Compaction)，本质上是`tiered compaction`算法，`写放大`比较低，但是`读放大`和`空间放大`比较高。
* [`kCompactionStyleFIFO`](https://github.com/facebook/rocksdb/wiki/FIFO-compaction-style)，通常用于几乎不修改的场景，比如消息队列、时序数据库等。
* [`kCompactionStyleNone`](https://github.com/facebook/rocksdb/wiki/Manual-Compaction)，关闭自动`compaction`，可以手动发起。

# 参考
_注_： **5**中间讲了很多不同的`compaction`算法，中间会引用一个概念`run`/`runs`，在**2**中得到了这个概念的解释：
> A run is a log-structured-merge (LSM) term for a large sorted file split into several smaller files. In other words, a run is a collection of sstables with non-overlapping token ranges.

[^1]: [Log-structured merge-tree](https://en.wikipedia.org/wiki/Log-structured_merge-tree)
[^2]: [Scylla’s Compaction Strategies Series: Space Amplification in Size-Tiered Compaction](https://www.scylladb.com/2018/01/17/compaction-series-space-amplification/)
[^3]: [Scylla’s Compaction Strategies Series: Write Amplification in Leveled Compaction](https://www.scylladb.com/2018/01/31/compaction-series-leveled-compaction/)
[^4]: [Rocksdb Compaction](https://github.com/facebook/rocksdb/wiki/Compaction)
[^5]: [Name that compaction algorithm](https://smalldatum.blogspot.com/2018/08/name-that-compaction-algorithm.html)
[^6]: [Optimizing Space Amplification in RocksDB](/images/posts/database/Optimizing-Space-Amplification-in-RocksDB.pdf)
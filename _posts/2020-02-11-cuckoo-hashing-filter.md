---
layout: post
title: Cuckoo Hashing and Filter
categories: [datastructure]
description: datastructure Cuckoo-Hashing Cuckoo-Filter
keywords: datastructure Cuckoo-Hashing Cuckoo-Filter
---

# 介绍

Cuckoo 哈希[^1]是一种查找开销十分稳定(_constant time worst-case complexity for lookups_)的哈希算法，这是其相较于开放地址法、链表法的显著特征。

![](/images/posts/datastructure/cuckoo-dense-hash.png)

![](/images/posts/datastructure/cuckoo-chain-hash.png)

Cuckoo 哈希可以视作是开放地址法的一种进阶，其基础算法描述为：在一个由一维数组构成的哈希表$$T$$中，采用两个哈希函数($$h_1,h_2$$)，使得任意一个元素在哈希表中存在 2 个存储位置：

- 当查询哈希表中元素$$x$$时，我们分别通过两个哈希函数($$h_1,h_2$$)计算出其 2 个可能存在的位置，然后进行(并行)查询，如果在$$T[h_1(x)]$$或$$T[h_2(x)]$$位置存在元素$$x$$，则存在。否则，不存在。
- 当在哈希表中插入元素$$x$$时，我们先进行查询，如果$$x$$已经存在，直接返回。否则，我们按照 2 个哈希函数顺序，依次计算其位置$$i = h_1(x), j = h_2(x)$$，如果$$T[i]/T[j]$$任一位置空闲，则将$$x$$放置在其中一个位置上；如果 2 个位置都不空闲，则可以将$$T[i]$$位置的元素$$y$$拿出，然后将$$x$$放置在$$T[i]$$的位置上，然后将$$y$$放置在$$h_1(y),h_2(y)$$中的另一个位置上去，以此递归，如果$$y$$所属的另一个位置仍被占用，则再将其拿出，再次放置，细节如下图所示；当递归踢元素的此处到达一定次数，则我们可以判定当前哈希表已经满了，需要重构哈希表大小。
- 当在哈希表中删除元素$$x$$时，按照查询的逻辑找到$$x$$所属位置，将其直接删除即可。
- 在一维空间上，Cuckoo 哈希的填充率上限差不多时 50%，要提高填充率需要增加哈希表关联的维度，后面会提到。

![](/images/posts/datastructure/cuckoo-hashing-insert.png)

# 变种

## 2-array 实现

如图所示，是一种比较常见的变种实现，其采用两个数组分别关联一个哈希函数，然后放置元素、踢出元素在这两个数组之间进行，元素$$x$$在第一个数组中的位置是$$h_1(x)$$，在第二个数组中的位置是$$h_2(x)$$。该种实现有在线可视化的操作动画[^2]。

![](/images/posts/datastructure/cuckoo-double-array.png)

## D-Cuckoo Hashing

将常见的 2 数组实现扩展到多层实现，同时搭配多种哈希函数，如下图扩展到 4 层数组，这样能一定程度提高整体的填充率，减少*rehash*的次数。*RocksDB*实现了基于多层*Cuckoo Hashing*的 SST 文件结构[^3]，最多支持 64 层 hash 表。充分利用其高效的查找优势。当*Cuckoo Hash*表层数增多时，哈希冲突时，找到适合踢出的元素就是一件复杂的事情了，*RocksDB*采用[广度优先搜索](https://github.com/facebook/rocksdb/blob/29e24434fec91cbeae1deb6cd96319af1b308716/table/cuckoo/cuckoo_table_builder.cc#L430-L439)来搜索最合适踢出的元素。

![](/images/posts/datastructure/cuckoo-hashing-4-layers.png)

## Cuckoo Filter

_cuckoo filter_[^4]是基于*Cuckoo Hashing*延伸出来的判断存在性的数据结构，功能性上与*bloom filter*一样。为了提高空间填充率，其也增加了 hash 数组空间，但是采用了和*RocksDB*不同的策略：

![](/images/posts/datastructure/cuckoo-filter-arch.png)

其中采用单层*bucket*数组的原始方案，但是每个*bucket*能够容纳的元素扩展为*b*个(如图*b=4*)，这样使得元素被踢出后，可以用快速、简单的方法找到其另一个*bucket*位置。由于只判断存在性，允许假阳性，为了节省空间，*bucket*中每个*entry*只存储对应元素的*fingerprint*(一种普通哈希值即可，通常为 1byte 信息)，然后定位所属*bucket*位置时，在根据如下两种哈希公式计算(一共三种 hash 值)：

$$
\begin{aligned}
& i = h_1(x) = hash(x) \\
& j = h_2(x) = h_1(x) \oplus hash(x's fingerprint) \\
\end{aligned}
$$

这样设计的必要性是，由于*cuckoo filter*中存储的是元素的*fingerprint*，已经丢失了大量信息，当该元素被踢出时，无法再次通过 hash 计算其对应对应的另一个*bucket*的位置，因此其中利用了异或($$\oplus$$)的特性，_i_、*j*可以分别又对方异或($$\oplus$$)上$$hash(fingerprint)$$计算得出：

$$
\begin{aligned}
& i = j \oplus hash(fingerprint) \\
& j = i \oplus hash(fingerprint) \\
\end{aligned}
$$

其增删改查方法也比较基本：

![](/images/posts/datastructure/cuckoo-filter-algorithm.png)

一些需要注意的点：

- 假如出现了三个 hash 值完全相等的两个元素($$fingerprint/h_1/h_2$$)，仍然按照算法正常运行，后果仅仅是对应的*bucket*中的*entry*存在两个相同的元素(_fingerprint_)而已，并不影响该数据结构的增删改查；删除其中之一时，删除其中一个元素(_fingerprint_)即可；当每个*bucket*有*b*个*entry*时，可以支持最多*2b*个三个 hash 值完全相等的元素，可以由相关计算，如果选用比较均匀的 hash 函数，一个数据集中数据能够重复到达*2b*的概率是十分的低的；
- 不建议元素重复添加，添加元素前先采用其他结构去重根据这套算法，否则，人为增加了三个 hash 值完全相等的概率；
- 由于哈希碰撞存在的可能性，不支持删除不在集合中的元素。

但是其相比*bloom filter*优势在于：

- CPU cache line 亲和性更高，更能够利用 CPU 硬件特性进行加速。可以通过一维数组实现多*entry*的*bucket*数组，由于 CPU cache line 的特性，使得查询*bucket*的每个*entry*速度更快，而*bloom filter*要访问分布在多个位置的 bit 信息，完全不能利用 CPU cache line。
- 查询时间复杂度更低，永远只查询固个数的数据；
- 占用存储空间更低，填充率高，如下图，当每个*bucket*有 4 个*entry*时，每个最总填充率能到达 95%，而且每个*entry*大于 8bit 后，填充率提升不大，因此每个*entry*只需要存储 8bit 的*fingerprint*数据，总和计算下来，*cuckoo filter*每个元素只需要占用 12bit 空间，而*bloom filter*需要用到 13bit；
  > ![](/images/posts/datastructure/cuckoo-filter-space.png) ![](/images/posts/datastructure/cuckoo-filter-cmp.png)
- 假阳性概率更低，从上图可以看出；
- 支持删除元素。

其一个开源实现[libcuckoo](https://github.com/efficient/libcuckoo)。

# 局限性/适用场景

*Cuckoo Hash*在填充率过高时，添加元素变得困难，通常适用于一些读多写少的场景。而且*Cuckoo Hash*的读写操作都需要访问 2 个以上的*bucket*，使得读写方法的难以并发执行，而且*rehash*也难以开销分摊，无论是无锁算法、或者减小锁粒度都是很困难的一件事：

- 死锁风险，*Cuckoo Hash*的读写操作都需要访问 2 个以上的*bucket*，而且两个*bucket*的上锁顺序也按照特定顺序，这就大大增加了多个元素并发访问时，死锁的风险；
- 假丢失数据，当某个元素被踢出并且还未插入另一个对应位置时，如果同时有查询该元素的请求到达时，可能会误判数据丢失。

[MemC3](https://github.com/efficient/memc3)[^5]实现了*concurrent cuckoo hashing*，通过很多努力，实现了多读单写并发的(_multiple-reader/single writer concurrent access_)的*cuckoo hashing*：

![](/images/posts/datastructure/cuckoo-memc3-arch.png)

- 其基础结构与前面*cuckoo filter*的设计类似，每个*bucket*可以存储四个元素，使得填充率到达 90% 以上；
- 为了充分发挥 CPU cache line 的特性，在每个 entry 上增加了一个 1byte 的 tag，减少指针解引用的次数，平均每次调用 `lookup` 方法时，只有 0.03 次指针解引用(97% 的概率被 tag 比较拦截)；
- 采用原子操作变更数据，放弃了排他锁。为了避免在读写并发时读到不新鲜的数据，采用数据版本检测的方法。写方法一开始会更新对应元素的版本，读方法开始和结束时，会分别读取一次版本，如果版本一直，则表示，读取过程中数据没有发生并发修改，否则，发生了改变，重新读取一次。这样就使得每个元素都需要一个内存存储版本，为了提高内存填充率，这里预分配了 8K 个`版本计数器`，将全部元素经过 hash 分配到不同`版本计数器`上，即每个`版本计数器`负责管理一批 hash 相同的元素。经过检验，在正常使用中只有`0.01%`的概率会触发读取重试的逻辑；
- 为了避免假丢失数据，其会先搜索出需要踢出操作的元素顺序，然后按照顺序反向踢出。如下图，其搜索出需要踢出的顺序(_a=>b=>c_)后，然后先将*c*的*entry*数据复制到另一个所属*bucket*后，再将*b*的*entry*数据复制到原来*c*的位置，这样就保证在踢出过程中，没有数据会出现丢失假象。
  > ![](/images/posts/datastructure/cuckoo-hash-path.png)

# 参考

[^1]: [Cuckoo Hashing for Undergraduates](/images/posts/datastructure/cuckoo-undergrad.pdf) 简介版没有太复杂的公式表达
[^2]: [Cuckoo Hashing 可视化](http://www.lkozma.net/cuckoo_hashing_visualization/)
[^3]: [RocksDB: Cuckoo Hashing Table Format](https://rocksdb.org/blog/2014/09/12/cuckoo.html)
[^4]: [Cuckoo Filter: Practically Better Than Bloom](/images/posts/datastructure/conext14_cuckoofilter.pdf)
[^5]: [MemC3: Compact and Concurrent MemCache with Dumber Caching and Smarter Hashing](/images/posts/datastructure/MemC3-Compact-and-Concurrent-MemCache-with-Dumber-Caching-and-Smarter-Hashing.pdf)

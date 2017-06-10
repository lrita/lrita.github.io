---
layout: post
title: boltdb 源码分析-MVCC/持久化-3
categories: [database, bolt]
description: boltdb
keywords: boltdb, database
---

# boltdb 持久化
在前面简介部分已经描述了一部分持久化相关的[内容](https://lrita.github.io/2017/05/21/boltdb-overview-0/#持久化)

`boltdb`采用单个文件来将数据存储在磁盘上，该文件的前4个`page`是固定的:

1. 第1个page为meta
2. 第2个page为meta
3. 第3个page是freelist，存储了一个int数组，
4. 第4个page是leaf page

## page
`page`是`boltdb`持久化时，与磁盘相关的数据结构。page的大小采用操作系统内存页的大小，即`getpagesize`系统调用
的返回值，通常是4k大小。

每个page开始的几个字节存储的是[`page`](https://github.com/boltdb/bolt/blob/e9cf4fae01b5a8ff89d0ec6b32f0d9c9f79aefdd/page.go#L30-L36)
的raw data:

```go
type page struct {
    id       pgid         // page的序号
    flags    uint16       // page的类型，有branchPageFlag/leafPageFlag/metaPageFlag/freelistPageFlag几种
    count    uint16       // 当page是freelistPageFlag类型时，存储的是freelist中pgid数组中元素的个数；
                          // 当page时其他类型时，存储的是inode的个数
    overflow uint32       // 当写操作数据量大于1个page大小时，该字段记录超出的page数，例如：写入12k的数据，
                          // page大小为4k，则page.overflow=2，标明page header后的2个page大小区域也是该page的
                          // 区域。
    ptr      uintptr
}
```

每个`page`对应对应一个磁盘上的数据块。这个数据块的layout为:

```
| page struct data | page element items | k-v pairs |
```

其分为3个部分：
* 第一部分`page struct data`是该`page`的header，存储的就是`page`struct的数据。
* 第二部分`page element items`其实就是`node`(下面会讲)的里`inode`的持久化部分数据。
* 第三部分`k-v pairs`存储的是`inode`里具体的key-value数据。

```go
type meta struct {
    magic    uint32  // 存储魔数0xED0CDAED
    version  uint32  // 标明存储格式的版本，现在是2
    pageSize uint32  // 标明每个page的大小
    flags    uint32  // 当前已无用
    root     bucket  // 根Bucket
    freelist pgid    // 标明当前freelist数据存在哪个page中
    pgid     pgid    //
    txid     txid    //
    checksum uint64  // 以上数据的校验和，校验数据是否损坏
}
```

## freelist
根据前面[简介](https://lrita.github.io/2017/05/21/boltdb-overview-0/#持久化)中的描述，`boltdb`
是不会释放空间的磁盘空间的，因此需要一个机制来实现磁盘空间的重复利用，`freelist`就是实现该机制的
文件page缓存。其数据结构为：

```go
type freelist struct {
    ids     []pgid          // all free and available free page ids.
    pending map[txid][]pgid // mapping of soon-to-be free page ids by tx.
    cache   map[pgid]bool   // fast lookup of all free and pending page ids.
}
```

其有三部分组成，`ids`记录了当前缓存着的空闲`page`的pgid，`cache`中记录的也是这些pgid，采用map记录
方便快速查找。

当用户需要`page`时，调用`freelist.allocate(n int) pgid`，其中n为需要的`page`数量，其会遍历`ids`，从中
挑选出连续n个空闲的`page`，然后将其从缓存中剔除，然后将其实的page-id返回给调用者。当不存在满足需求的
`page`时，返回0，因为文件的起始2个`page`固定为meta page，因此有效的page-id不可能为0。

当某个写事务产生无用`page`时，将调用`freelist.free(txid txid, p *page)`将指定`page` p放入`pending`池和
`cache`中。当下一个写事务开启时，会将没有`Tx`引用的`pending`中的`page`搬移到`ids`缓存中。之所以这样做，
是为了支持事务的回滚和并发读事务，从而实现MVCC。

当发起一个读事务时，`Tx`单独复制一份`meta`信息，从这份独有的`meta`作为入口，可以读出该`meta`指向的数据，
此时即使有一个写事务修改了相关key的数据，新修改的数据只会被写入新的`page`，读事务持有的`page`会进入`pending`
池，因此该读事务相关的数据并不会被修改。只有该`page`相关的读事务都结束时，才会从`pending`池进入到`cache`池
中，从而被复用修改。

当写事务更新数据时，并不直接覆盖老数据，而且分配一个新的`page`将更新后的数据写入，然后将老数据占用的`page`
放入`pending`池，建立新的索引。当事务需要回滚时，只需要将`pending`池中的`page`释放，将索引回滚即完成数据的
回滚。这样加速了事务的回滚。减少了事务缓存的内存使用，同时避免了对正在读的事务的干扰。


## B-Tree 索引

`Cursor`是遍历`Bucket`的迭代器，其声明是:
```
type elemRef struct {
    page  *page
    node  *node
    index int
}

type Cursor struct {
	bucket *Bucket     // parent Bucket
	stack  []elemRef   // 遍历过程中记录走过的page-id或者node，elemRef中的page、node同时只能有一个存在
}

type Bucket struct {
    *bucket
    tx       *Tx                // the associated transaction
    buckets  map[string]*Bucket // subbucket cache
    page     *page              // inline page reference
    rootNode *node              // materialized node for the root page.
    nodes    map[pgid]*node     // node cache
    FillPercent float64
}

type node struct {
    bucket     *Bucket
    isLeaf     bool	// 标记该节点是否为叶子节点，决定inode中记录的是什么
    unbalanced bool	// 当该node上有删除操作时，标记为true，当Tx执行Commit时，会执行rebalance，将inode重新排列
    spilled    bool
    key        []byte	// 当加载page变成node缓存时，将该node下边界inode[0]的key缓存在node上，用于在parent node
                        // 查找本身时使用
    pgid       pgid
    parent     *node
    children   nodes
    inodes     inodes
}

type inode struct {
    flags uint32   // 当所在node为叶子节点时，记录key的flag
    pgid  pgid     // 当所在node为叶子节点时，不使用，当所在node为分支节点时，记录所指向的page-id
    key   []byte   // 当所在node为叶子节点时，记录的为拥有的key；当所在node为分支节点时，记录的是子
                   // 节点的上key的下边界。例如，当当前node为分支节点，拥有3个分支，[1,2,3][5,6,7][10,11,12]
                   // 这该node上可能有3个inode，记录的key为[1,5,10]。当进行查找时2时，2<5,则去第0个子分支上继
                   // 续搜索，当进行查找4时，1<4<5,则去第1个分支上去继续查找。
    value []byte   // 当所在node为叶子节点时，记录的为拥有的value
}

type bucket struct {
    root     pgid   // page id of the bucket's root-level page
    sequence uint64 // monotonically incrementing, used by NextSequence()
}
```

`boltdb`通过B-Tree来构建数据的索引，其B-Tree的根为`Bucket`，其数上的每个节点为`node`和`inode`或
`branchPageElements`和`leafPageElement`；B-Tree的上全部的Key-Value数据都记录在叶子节点上。

当`Bucket`的数据还没有commit写入磁盘时，在内存中以`node`和`inode`来记录，当数据被写入磁盘时，以
`branchPageElements`和`leafPageElement`来记录。

正式这样的混合组织方式，因此在搜索这个B-Tree的时候，遇见的数据可以是磁盘上的数据也可能是内存中的数据，
因此采用`Cursor`这样的迭代器。`Cursor`的`stack []elemRef`中几率的每次迭代操作是走过的路径。其路径可能是
磁盘中的某个`page`也可能是还未刷入磁盘的`node`。`elemRef`中的`index`用来记录search时命中的index。

这棵大B-Tree上总共有2中节点，一种是`Bucket`，一种是`node`，这两种不同是节点都存储在B-Tree的K-V对上，只是
flag不同。`Bucket`被当做树或者子树的根节点，`node`是B-Tree上的普通节点，依负在某一个`Bucket`上。`Bucket`
当做一个子树看待，所以不会跟同级的`node`一同`rebalance`。`Bucket`在树上记录的value为`bucket`，即根节点的
page-id和sequence。

### Bucket
因此，很好理解`Bucket`的嵌套关系。子`Bucket`就是在父`Bucket`上创建一个`Bucket`节点。
关于`Bucket`的描述可以参考[BoltDB之Bucket(一)](http://www.d-kai.me/boltdb之bucket一/)/[BoltDB之Bucket(二)](http://www.d-kai.me/boltdb之bucket二/)
两篇文章的描述。看这2篇文章时，先忽略掉`inline bucket`部分的内容，不然不容易理解。`inline bucket`只不过是
一个对于磁盘空间的优化，通常`Bucket`的信息在磁盘上的记录很小，如果直接占用一个`page`有些浪费，则将`Bucket`
对应`page`的剩余部分当做`Bucket`可以使用的一个`page`，则当数据量较小时，可以节省一个`page`。

`node`并不是B-Tree上的一个节点，并不是最总存储数据的K-V对，在`node`上的`inode`才是最终存储数据的K-V对。
每个`node`对应唯一一个`page`，是`page`在内存中的缓存对象。在`Bucket`下会有`node`的缓存集合。当需要访问
某个`page`时，会先去缓存中查找其`node`，只有`node`不存在时，才去加载`page`。

### MVCC
`bolt`的MVCC实现主要依赖COW和`meta`副本来实现。

每当一个`Tx`被创建时，就复制一份当前最新的`meta`。在`Tx`中的每个操作都会将变化后的B-Tree上的`node`缓存
`Tx`的`Bucket`副本中，这个变化只对该`Tx`可见。当有删除操作时，`Tx`会将要释放的page-id暂存在`freelist`的
`pending`池中，当`Tx`调用Commit时，`freelist`会将`pending`的page-id真正标记为free状态，如果`Tx`调用`Rollback`
则会将`pending`池中的page-id移除，则使`Tx`的删除操作回滚。

当`Tx`调用`Commit`时，

会触发`Bucket`的`rebalance`，`rebalance`会根据阈值条件，尽量提高每个修改过的`node`的
使用率(即`Bucket`缓存的`node`，只有`put`/`del`过的`page`才会加载成`node`缓存在`Bucket`下)，经过剪去空数据分
枝、合并相邻且填充率较低的分支，最后通过数据的搬移，释放更多的`page`。

然后再触发`Bucket`的`spill`，`spill`会再将`rebalance`聚拢后的`node`根据`Bucket.FillPercent`将每个`node`所持
有的数据将到`pagesize*Bucket.FillPercent`之下。然后获取新的`page`(获取新的`page`时，会先去freelist查找能复用
page-id，如果找不到就新分配一个，当新分配的`page`使文件大小超过只读mmap的大小时，会重新mmap一次)，然后将`node`
写入`page`。

然后更新`freelist`的持久化记录。然后将上面操作的`page`刷新到磁盘上。最后更新`meta`的在磁盘上的记录、释放`Tx`
的资源。

因此在没有更新`meta`之前，写入的数据对于读事务都是不可见的。

### 文件映射增长策略
当`boltdb`文件小于1GB时，每次重新映射时，映射大小翻倍，当文件大于1GB时，每次增长1GB，且与pagesize对齐。

## 参考
* [持久化文件增长和复用](https://github.com/boltdb/bolt/issues/308#issuecomment-74811638)

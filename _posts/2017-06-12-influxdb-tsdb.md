---
layout: post
title: influxdb 源码解析-tsdb
categories: [database, influxdb]
description: influxdb tsdb
keywords: influxdb tsdb
---

`tsdb`是`influxdb`的存储引擎，主要用于持久化时序数据。在分析`tsdb`之前，我们先要了解`influxdb`在使用上关于
存储的一些概念。

# concept
对于`influxdb`中涉及到的各种概念，官方已经提供了一个[词汇表](https://docs.influxdata.com/influxdb/v1.2/concepts/glossary)，
以供用户查阅。

`influxdb`将数据按不同的`database`和`measurements`来存储。`database`可以类比为`mysql`中的`database`概念。
`measurements`可以类比为`mysql`中`table`的概念。写入一条数据时，需要制定其`database`和`measurements`。`influxdb`
提供一套`influxql`的类似`sql`的语法来使用，具体的操作可以参考[官方文档](https://docs.influxdata.com/influxdb/v1.2/query_language/schema_exploration/)

然后`influxdb`存储的数据是schemaless的，可以包含任意维度(列)，每个维度为一个K-V结构，其中维度划分为`tag`和
`field`2种。

#### tag
`tag`是可以建立索引的，在查询时有更好的性能，是可选的字段。通常用来存储一些meta字段。`field`不能建立索引，在查
询某个`field`的值时，需要遍历该时间区间内的全部数据。`tag`跟`field`的区别主要在是否建立索引上，因此写入数据时，
选取`tag`还是`field`主要依据查询语句而定，官方有一个[指导文档](https://docs.influxdata.com/influxdb/v1.2/concepts/schema_and_data_layout/)
可以进行参考。

#### shard
`shard`是一个存储单元，用于存储一段时间范围内的数据，同时`shard`与`retention policy`直接相关，不同的过期策略
下不同的`shard`，每个`shard`底层对应一个`TSM`存储引擎。当过期策略检查到某一个`shard`过期后，会释放其对应的资源。

#### point
每一次写入的数据称之为`point`，其包含`timestamp`、`measurements`、`retention policy`、`tag`和`field`等信息，其中`point`的`key`[由
`measurement`和`tag`hash后构成](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/models/points.go#L1481-L1485)。
然后根据`point.HashID`的值和`retention policy`选取合适的`shard`，将该`point`写入该`shard`。

`tsdb`对外提供的2个最重要的接口是：
* `CreateShard(database, retentionPolicy string, shardID uint64, enabled bool) error` 创建`database`对应的的`shard`
* `WriteToShard(shardID uint64, points []models.Point) error` 向指定`shard`写入数据

#### series
series 相当于是InfluxDB中一些数据的集合，在同一个database中，retention policy、measurement、tag sets 完全
相同的数据同属于一个series，同一个series的数据在物理上会按照时间顺序排列存储在一起。

series的key为measurement加所有tags的序列化字符串，这个key在之后会经常用到。

#### store
`store`是`influxdb`的存储模块，全局只有一个该实例。主要负责将数据按一定格式写入磁盘，并且维护`influxdb`相关的
存储概念。例如：创建/删除`Shard`、创建/删除`retention policy`、调度`shard`的compaction、以及最重要的`WriteToShard`
等等。在`store`内部又包含`index`和`engine`2个抽象概念，`index`是对应`shard`的索引，`engine`是对应`shard`的存储实现，
不同的`engine`采用不同的存储格式和策略。后面要讲的`tsdb`其实就是一个`engine`的实现。在`influxdb`启动时，会创建
一个`store`实例，然后`Open`它，初始化时，它会加载已经存在的[`Shard`](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/tsdb/store.go#L154-L276)
，并启动一个`Shard`[监控任务](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/tsdb/store.go#L1047-L1116)，
监控任务负责调度每个`Shard`的`Compaction`和对使用`inmem`索引的`Shard`计算每种`Tag`拥有的数值的基数(与配置中`max-values-per-tag`有关)。

# tsdb
我们可以先阅读以下对于`tsdb`的[官方文档](https://docs.influxdata.com/influxdb/v1.2/concepts/storage_engine/)。
其采用的存储模型是`LSM-Tree`模型，对其进行了一定的改造。将其称之为`Time-Structured Merge Tree (TSM)`

当一个`point`写入时，`influxdb`根据其所属的`database`、`measurements`和`timestamp`选取一个对应的`shard`。每个
`shard`对应一个`TSM`存储引擎。每个`shard`对应一段时间范围的存储。

一个`TSM`存储引擎包含：
* `In-Memory Index` 在`shard`之间共享，提供`measurements`，`tags`，和`series`的索引
* `WAL` 同其他database的binlog一样，当`WAL`的大小达到一定大小后，会重启开启一个`WAL`文件。
* `Cache` 内存中缓存的`WAL`，加速查找
* `TSM Files` 压缩后的`series`数据
* `FileStore` `TSM Files`的封装
* `Compactor` 存储数据的比较器
* `Compaction Planner` 用来确定哪些`TSM`文件需要`compaction`，同时避免并发`compaction`之间的相互干扰
* `Compression` 用于压缩持久化文件
* `Writers/Readers` 用于访问文件

`shard`通过[`CreateShard`](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/tsdb/store.go#L358-L407)
来创建。可以看出其依次创建了所需的文件目录，然后创建[`Index`](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/tsdb/index/inmem/inmem.go#L46-L58)
和[`Shard`](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/tsdb/shard.go#L104-L134)
数据结构。

## 文件结构
在`influxdb`指定的存储目录下的文件结构为:

```
.
├── data  # 配置中[data]下dir配置的路径
│   ├── _internal
│   │   └── monitor
│   │       ├── 73
│   │       │   └── 000000003-000000002.tsm
│   │       ├── 74
│   │       │   └── 000000003-000000002.tsm
│   │       └── 75
│   │           └── 000000001-000000001.tsm
│   └── testing
│       └── autogen
│           └── 2
│               └── 000000002-000000002.tsm
├── meta
│   └── meta.db
└── wal	# 配置文件[data]下wal-dir配置
    ├── _internal
    │   └── monitor
    │       ├── 73
    │       │   └── _00012.wal
    │       ├── 74
    │       │   └── _00012.wal
    │       └── 75
    │           ├── _00005.wal
    │           ├── _00006.wal
    │           └── _00007.wal
    └── testing
        └── autogen
            └── 2
                └── _00003.wal
```

每一个`shard`为目录，其目录的命名格式为：

* `$(ROOT)/data/$(Database)/$(RetentionPolicy)/$(ShardID)`
* `$(ROOT)/wal/$(Database)/$(RetentionPolicy)/$(ShardID)`

_注：当相关`shard`不使用`In-Memory Index(inmem)`时，会使用文件型`index`，默认类型为`tsi1`，会在创建
`$(ROOT)/data/$(Database)/$(RetentionPolicy)/$(ShardID)/index`文件。_

## 存储结构
每个`shard`由一个`Index`和一个`Engine`组成，`Index`负责进行反向索引数据，`Engine`为具体的存储模型实现
，即上面的`TSM`结构。

### [Engine](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/tsdb/engine.go#L30-L81)

```go
type Engine interface {
    Open() error
    Close() error
    SetEnabled(enabled bool)
    SetCompactionsEnabled(enabled bool)

    WithLogger(zap.Logger)

    LoadMetadataIndex(shardID uint64, index Index) error

    CreateSnapshot() (string, error)
    Backup(w io.Writer, basePath string, since time.Time) error
    Restore(r io.Reader, basePath string) error
    Import(r io.Reader, basePath string) error

    CreateIterator(measurement string, opt query.IteratorOptions) (query.Iterator, error)
    IteratorCost(measurement string, opt query.IteratorOptions) (query.IteratorCost, error)
    WritePoints(points []models.Point) error

    CreateSeriesIfNotExists(key, name []byte, tags models.Tags) error
    CreateSeriesListIfNotExists(keys, names [][]byte, tags []models.Tags) error
    DeleteSeriesRange(keys [][]byte, min, max int64) error

    SeriesSketches() (estimator.Sketch, estimator.Sketch, error)
    MeasurementsSketches() (estimator.Sketch, estimator.Sketch, error)
    SeriesN() int64

    MeasurementExists(name []byte) (bool, error)
    MeasurementNamesByExpr(expr influxql.Expr) ([][]byte, error)
    MeasurementNamesByRegex(re *regexp.Regexp) ([][]byte, error)
    MeasurementFields(measurement []byte) *MeasurementFields
    ForEachMeasurementName(fn func(name []byte) error) error
    DeleteMeasurement(name []byte) error

    // TagKeys(name []byte) ([][]byte, error)
    HasTagKey(name, key []byte) (bool, error)
    MeasurementTagKeysByExpr(name []byte, expr influxql.Expr) (map[string]struct{}, error)
    MeasurementTagKeyValuesByExpr(auth query.Authorizer, name []byte, key []string, expr influxql.Expr, keysSorted bool) ([][]string, error)
    ForEachMeasurementTagKey(name []byte, fn func(key []byte) error) error
    TagKeyCardinality(name, key []byte) int

    // InfluxQL iterators
    MeasurementSeriesKeysByExpr(name []byte, condition influxql.Expr) ([][]byte, error)
    SeriesPointIterator(opt query.IteratorOptions) (query.Iterator, error)

    // Statistics will return statistics relevant to this engine.
    Statistics(tags map[string]string) []models.Statistic
    LastModified() time.Time
    DiskSize() int64
    IsIdle() bool
    Free() error

    io.WriterTo
}
```
当`influxd`的API收到数据后，最终会根据所属的`shard`而找到对应的`Engine`，然后调用[`WritePoints`](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/tsdb/engine/tsm1/engine.go#L799-L856)
写入该`Engine`。在`Engine`中会将`models.Point`写入[`Cache`](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/tsdb/engine/tsm1/cache.go#L182-L204)
和[`WAL`](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/tsdb/engine/tsm1/wal.go#L81-L115)。
`Cache`和`WAL`中存有相同的数据，`Cache`主要用于查询时的读取操作，当用户查询时，可能会读取`Cache`和`TSM`
文件中的数据，当数据有重复时，`Cache`中的数据优先级高。`WAL`中数据主要用于启动时的数据加载。

可以从`WritePoints`看出其将`models.Point`拆分为多个[`Value`](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/tsdb/engine/tsm1/encoding.go#L94-L110)。
其拆分规则为：

当在database`tt`执行`insert disk_free,hostname=server01 value1=1000i,value2=1001i 1435362189575692182`
后，该条会被解析为一个`models.Point`，然后在写入`Engine`后，因为有2个`Field`，所以会被拆解为2个`Value`。
第一个`Value`的Key为`disk_free#!~#hostname=server01value1`的`IntegerValue`，`IntegerValue`中包含数据1000和
时间戳1435362189575692182，第二个`Value`为Key为`disk_free#!~#hostname=server01value2`的`IntegerValue`，
`IntegerValue`中包含数据1001和时间戳1435362189575692182。

然后会将分解为的`Value`写入`Cache`和`WAL`。`Cache`在内存中的组织形式为一个两级的map，在这里就不细说了。

将分解成的`Value`以[`Values`](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/tsdb/engine/tsm1/engine.go#L854)
的形式写入`WAL`[0](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/tsdb/engine/tsm1/wal.go#L397)/[1](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/tsdb/engine/tsm1/wal.go#L1013-L1029):

```
| type(0x01) 1 byte | data length 4 byte | data (snappy compressed Values encoding data) |
```

[Values encoding的格式为](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/tsdb/engine/tsm1/wal.go#L631-L727):
```
    // ┌────────────────────────────────────────────────────────────────────┐
    // │                           WriteWALEntry                            │
    // ├──────┬─────────┬────────┬───────┬─────────┬─────────┬───┬──────┬───┤
    // │ Type │ Key Len │   Key  │ Count │  Time   │  Value  │...│ Type │...│
    // │1 byte│ 2 bytes │ N bytes│4 bytes│ 8 bytes │ N bytes │   │1 byte│   │
    // └──────┴─────────┴────────┴───────┴─────────┴─────────┴───┴──────┴───┘
```
在写`WAL`时，如果`WAL`文件大于[10M](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/tsdb/engine/tsm1/wal.go#L28)
时，会发生滚动，[生成一个新的`WAL`文件](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/tsdb/engine/tsm1/wal.go#L545-L571)。

### Compaction
当数据不断写入，`WAL`文件的数量和`Cache`的大小会不断增长，因此需要一个`Compaction`来对数据进行压缩、文
件清理。在`Engine`被创建后，会启动3个与`Compaction`相关的任务[`compactCache`](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/tsdb/engine/tsm1/engine.go#L1134-L1159)
、[`compactTSMFull`](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/tsdb/engine/tsm1/engine.go#L1195-L1214)
和[`compactTSMLevel`](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/tsdb/engine/tsm1/engine.go#L1174-L1193)
分别对`Cache`和`TSM`进行`Compaction`。

* `compactCache` 主要是将内存中的`Cache`的数据变为TSM文件，然后清除对应的`WAL`文件。
* `compactTSMFull`/`compactTSMLevel` 主要是进行`TSM`文件的Compaction，

#### Cache Compaction
当`Cache`使用内存的大小大于配置文件中`cache-snapshot-memory-size`的大小，或者空闲时间大于配置文件中
`cache-snapshot-write-cold-duration`的时间时，会触发`Cache`的compaction。其会将`Cache`中的`Value`数据
[写入新的`TSM`文件](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/tsdb/engine/tsm1/compact.go#L650-L672)
，然后删除与该`Cache`对应的全部`WAL`文件。

#### TSM 文件
`TSM`文件有4块组成，分别为`Header`、`blocks`、`Index`和`Footer`。
```
┌────────┬────────────────────────────────────┬─────────────┬──────────────┐
│ Header │               Blocks               │    Index    │    Footer    │
│5 bytes │              N bytes               │   N bytes   │   4 bytes    │
└────────┴────────────────────────────────────┴─────────────┴──────────────┘
```

其中`Header`的前4字节为Magic Number(大端的0x16D116D1)，第5个字节为版本号(目前为1)。
```
┌───────────────────┐
│      Header       │
├─────────┬─────────┤
│  Magic  │ Version │
│ 4 bytes │ 1 byte  │
└─────────┴─────────┘
```

`Blocks`存储的是很多组CRC32和Data组成的Block，通常每个Block中有[1000](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/tsdb/config.go#L42)
个`Value`。每个Block中存储的都是相同Key的`Value`，该Block数据序列化后的长度、`Value`的类型、最大时间、最小
时间、在文件中的偏移地址等都存储在与之对应的`Index`块中。
```
┌───────────────────────────────────────────────────────────┐
│                          Blocks                           │
├───────────────────┬───────────────────┬───────────────────┤
│      Block 1      │      Block 2      │      Block N      │
├─────────┬─────────┼─────────┬─────────┼─────────┬─────────┤
│  CRC    │  Data   │  CRC    │  Data   │  CRC    │  Data   │
│ 4 bytes │ N bytes │ 4 bytes │ N bytes │ 4 bytes │ N bytes │
└─────────┴─────────┴─────────┴─────────┴─────────┴─────────┘
```

Blocks后面是Index块，Index中存储的是按`Value`的Key的字典序和时间戳排列的Block的元数据。
每个Block的元数据称为一个[`IndexEntry`](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/tsdb/engine/tsm1/writer.go#L172-L181)
。同一个Key可能拥有多个Block的数据，因为每个Block最多1000个`Value`，因此其`IndexEntry`
为一个数组。
```
┌────────────────────────────────────────────────────────────────────────────┐
│                                   Index                                    │
├─────────┬─────────┬──────┬───────┬─────────┬─────────┬────────┬────────┬───┤
│ Key Len │   Key   │ Type │ Count │Min Time │Max Time │ Offset │  Size  │...│
│ 2 bytes │ N bytes │1 byte│2 bytes│ 8 bytes │ 8 bytes │8 bytes │4 bytes │   │
├─────────┼─────────┼──────┼───────┼─────────┼─────────┼────────┼────────┼───┤
│ Key Len │   Key   │ Type │ Count │Min Time │Max Time │ Offset │  Size  │...│
│ 2 bytes │ N bytes │1 byte│2 bytes│ 8 bytes │ 8 bytes │8 bytes │4 bytes │   │
└─────────┴─────────┴──────┴───────┴─────────┴─────────┴────────┴────────┴───┘
```
根据`IndexEntry`中记录的时间戳范围和文件偏移地址，我们能很高效的找到我们所需数据在TSM文件中
位置。

The last section is the footer that stores the offset of the start of the index.

最后一个是Footer块，占用8个字节，存储Index块的偏移位置，Footer相对于文件末尾的位置是固定的，
读取TSM文件时，可以从文件末尾先速度出Index块的偏移量，再加载读取Index内容。
```
┌─────────┐
│ Footer  │
├─────────┤
│Index Ofs│
│ 8 bytes │
└─────────┘
```

#### Level Compaction
在Cache Compaction的同时，还会进行[Level Compaction](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/tsdb/engine/tsm1/engine.go#L215-L238)。


### `Index`

```go
type Index interface {
    Open() error
    Close() error
    WithLogger(zap.Logger)

    MeasurementExists(name []byte) (bool, error)
    MeasurementNamesByExpr(expr influxql.Expr) ([][]byte, error)
    MeasurementNamesByRegex(re *regexp.Regexp) ([][]byte, error)
    DropMeasurement(name []byte) error
    ForEachMeasurementName(fn func(name []byte) error) error

    InitializeSeries(key, name []byte, tags models.Tags) error
    CreateSeriesIfNotExists(key, name []byte, tags models.Tags) error
    CreateSeriesListIfNotExists(keys, names [][]byte, tags []models.Tags) error
    DropSeries(key []byte) error

    SeriesSketches() (estimator.Sketch, estimator.Sketch, error)
    MeasurementsSketches() (estimator.Sketch, estimator.Sketch, error)
    SeriesN() int64

    HasTagKey(name, key []byte) (bool, error)
    TagSets(name []byte, options query.IteratorOptions) ([]*query.TagSet, error)
    MeasurementTagKeysByExpr(name []byte, expr influxql.Expr) (map[string]struct{}, error)
    MeasurementTagKeyValuesByExpr(auth query.Authorizer, name []byte, keys []string, expr influxql.Expr, keysSorted bool) ([][]string, error)

    ForEachMeasurementTagKey(name []byte, fn func(key []byte) error) error
    TagKeyCardinality(name, key []byte) int

    // InfluxQL system iterators
    MeasurementSeriesKeysByExpr(name []byte, condition influxql.Expr) ([][]byte, error)
    SeriesPointIterator(opt query.IteratorOptions) (query.Iterator, error)

    // Sets a shared fieldset from the engine.
    SetFieldSet(fs *MeasurementFieldSet)

    // Creates hard links inside path for snapshotting.
    SnapshotTo(path string) error

    // To be removed w/ tsi1.
    SetFieldName(measurement []byte, name string)
    AssignShard(k string, shardID uint64)
    UnassignShard(k string, shardID uint64) error
    RemoveShard(shardID uint64)

    Type() string

    Rebuild()
}
```

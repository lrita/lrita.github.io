---
layout: post
title:  leveldb源码分析-6-SSTable
date:   2017-01-11 20:00:11
category: "leveldb"
---

# SSTable

  SSTable是leveldb中持久化部分的核心模块。
  在源码文档的[doc/table_format.txt](https://github.com/google/leveldb/blob/master/doc/table_format.txt)
  中对SSTable持久化的格式进行了描述。

  ![SST](/images/posts/leveldb/leveldb-sst.jpg)

  如上图所示，SST文件逻辑上可分为两大块，数据存储区Data Block，以及各种Meta信息。

* 文件中的K/V对是有序存储的，并且被划分到连续排列的Data Block里面，这些Data Block从文件头开始顺序存储，Data Block的存储格式代码在block_builder.cc中
* 紧跟在Data Block之后的是Meta Block，其格式代码也在block_builder.cc中；Meta Block存储的是Filter生成的信息，比如Bloom过滤器生成的bitmap，用于快速定位key是否在data block中
* MetaIndex Block是对Meta Block的索引，它只有一条记录，key是meta index的名字（也就是Filter的名字），value为指向meta index的BlockHandle；BlockHandle是一个结构体，成员offset_是Block在文件中的偏移，成员size_是block的大小
* Index block是对Data Block的索引，对于其中的每个记录，其key >= Data Block最后一条记录的key，同时 < 其后Data Block的第一条记录的key；value是指向data index的BlockHandle
* Footer，文件的最后，大小固定

## 文件

* `table/format.h`，`table/format.cc`: `BlockHandle`、`Footer`定义实现
* `table/block.h`，`table/block.cc` `Block`定义实现

## BlockHandle

  BlockHandle存储了2个varint64，BlockHandle的最大长度为`kMaxEncodedLength`(20)。

## Block

  ![block](/images/posts/leveldb/leveldb-sst-block.jpg)

  如图所示，每个Block有三部分构成：block data, type, crc32。

* type指明使用的是哪种压缩方式，当前支持`kNoCompression`和`kSnappyCompression`
* 虽然block有好几种，但是Block Data都是有序的K/V对，因此写入、读取BlockData的接口都是统一的，对于Block Data的管理也都是相同的。
* crc32部分是该Block的校验码

### Block格式

  BlockBuilder对key的存储是前缀压缩的，对于有序的字符串来讲，这能极大的减少存储空间。
  但是却增加了查找的时间复杂度，为了兼顾查找效率，每隔K个key，leveldb就不使用前缀压
  缩，而是存储整个key，这就是重启点（restartpoint）。
  在构建Block时，有参数`Options::block_restart_interval`定每隔几个key就直接存储一个
  重启点key。

  Block在结尾记录所有重启点的偏移，可以二分查找指定的key。Value直接存储在key的后面，
  无压缩。
  对于重启点，shared_bytes = 0。

  对于一个k/v对，其在block中的存储格式为：

```
  共享前缀长度         shared_bytes:    varint32
  前缀之后的字符串长度 unshared_bytes:  varint32
  值的长度             value_length:    varint32
  前缀之后的字符串     key_delta:       char[unshared_bytes]
  值                   value:           char[value_length]
```

  例如第一条记录为key:`hello world` value:`11`，第二条记录为key:`hello you` value:`9`，
  那么第一条记录存储格式为:0+11+2+`hello world`+`11`，首先说明下，第一条是重启点，因此
  记录共享长度为0，因为它没有上条记录，所以就没有共享。那么第二条记录存储格式为：6+3+1+you+9。

  Block的结尾段格式是：

```
  restarts:       uint32[num_restarts]
  num_restarts:   uint32 // 重启点个数
```

  元素restarts[i]存储的是block的第i个重启点的偏移。很明显第一个k/v对，总是第一个重启
  点，也就是restarts[0] = 0;

  ![block-layout](/images/posts/leveldb/leveldb-sst-block-layout.jpg)

### DataBlock

  DataBlock按照Block的布局存储相关的K/V数据。
### MetaBlock/FilterBlock

  该Block不存储K/V，因此不按照上面K/V的内存布局。
  当`DB::Open`传入的`options.filter_policy == NULL`时，没有meta信息。

  该Block存储的是filter data；数据布局如图所示：
  ![filter-block](/images/posts/leveldb/leveldb-sst-filter-block.jpg)

  filter< N >偏移为1个fixed32，记录filter< N > data 的offset。

### MetaBlockIndex

  当`DB::Open`传入的`options.filter_policy == NULL`时，没有meta信息。

  MetaBlockIndex存储的是一个meta信息的K/V
  
  目前只有一个key:

  key `filter.<options.filter_policy->Name()>` 存储的是MetaBlock(filterblock)的`BlockHandle`。

### IndexBlock

  `IndexBlock` 根据Block的K/V存储形式，存储index数据。

  `key`第i块最大的键值，但是必须比第i+1个data block最小键值小。因为block数据是有序
  的（skiplist数据为有序），所以有最大键值，就可以知道这个块存储的数据的键值范围。
  `value`是第i个data block的`BlockHandle`，表明偏移量和大小。

  例如第i个data block最小键值为hello，最大键值为world；第i+1个data block最小键值为www，
  最大键值为yellow，所以第i个data block的键值字段为world。

  ![index-block](/images/posts/leveldb/leveldb-sst-index-block.jpg)

### Footer

  Footer在文件的最后，大小固定，格式如：
  ![footer](/images/posts/leveldb/leveldb-sst-footer.jpg)
  Footer长度为固定2个BlockHandle的最大长度(20)和1个64bit魔数长度。这样方便解析。
  当BlockHandle实际长度和最大长度之间的空隙填充为0。

  成员metaindex_handle指出了meta index block的起始位置和大小；
  成员index_handle指出了index block的起始地址和大小；
  这两个字段都是BlockHandle对象，可以理解为索引的索引，通过Footer可以直接定位
  到metaindex和index block。再后面是一个填充区和魔数(0xdb4775248b80fb57)。


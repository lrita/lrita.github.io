---
layout: post
title:  leveldb源码分析-4-MemTable
date:   2017-01-09 20:00:11
category: "leveldb"
---

## MemTable

    在讲memtable之前，有必要先讲讲leveldb模型，当向leveldb写入数据时，首先将数据写入log文件，
  然后在写入memtable内存中。log文件主要是用在当断电时，内存中数据会丢失，数据可以从log文件中
  恢复。当memtable数据达到一定大小即（`options.write_buffer_size`大小，默认`4<<20`)，会变为
  immemtable，然后log文件也生成一个新的log文件。immemtable数据将会被持久化到磁盘中。模型图如下：

  ![memtable](/images/posts/leveldb/leveldb-memtable.jpg)

## MemTable实现

  `MemTable`的实现为非线程安全的。
  `MemTable`是基于引用计数的，每次使用需要首先调用`Ref()`，使用完毕时调用`Unref()`。

  `MemTable`内部持有一个内存池`Arena`，其相关内存都有该内存池分配。
  `MemTable`其实有一个`SkipList`实现的。`SkipList`持有的元素为`const char *`，也是由其内存池所分配的。
  `SkipList`持有的元素的内存分布为：

```
  | n (varint,key部分内存,含sequence) | key data (n-8 bytes) | sequence (8 bytes) | m (varint,val data) | val data (m bytes) |
```

  可以看出key部分的内存结构为一个`LookupKey`的内存结构，其`Get`方法使用的key就是`LookupKey`。

  拥有`Add`、`Get`和`NewIterator`方法。

* `Add`将一组k-v存储在`MemTable`里；
* `Get`获取指定key的vale，当key的value type为`kTypeDeletion`时，返回`NotFound`；
* `NewIterator`返回一个iteration，遍历`SkipList`。

## 源码分析

  [memtable.h](https://github.com/lrita/leveldb/blob/master/db/memtable.h)<br/>
  [memtable.cc](https://github.com/lrita/leveldb/blob/master/db/memtable.cc)

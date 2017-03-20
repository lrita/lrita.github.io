---
layout: post
title:  leveldb源码分析-9-operation
date:   2017-03-21 20:00:00
category: "leveldb"
---

# leveldb operation
  leveldb的操作分别为`Put`、`Delete`、`Write`、`Get`、`CompactRange`、`NewIterator`、`GetSnapshot`、
  `ReleaseSnapshot`等。

  其中`Put`、`Delete`都是`Write`操作的简单封装，`Delete`就是写入一条删除记录。

## Write
  `Status Write(const WriteOptions& options, WriteBatch* my_batch)`写入一批记录到leveldb中。Write
  操作会先分配一个Writer，然后将这个Writer添加到Writer队列中，该队列将全部Writer的操作串行化（Write
  操作可能在多个线程中被调用，但是其相关的Writer会通过Writer队列串行化，最总在其中的一个Write调用线
  程中被批量执行）。

  Writer被调用时，leveldb先调用`MakeRoomForWrite`。`MakeRoomForWrite`会判断写缓存的空间，当空间不够
  时会触发compaction，或者当有compaction进行是，会进行限速。
  
  然后把Writer中的内容构建一个WriteBatch，填充这些记录的sequence值，然后把WriteBatch写入binlog和memtable。

  最后唤醒等待中的Write调用。

## Get
  `Status Get(const ReadOptions& options, const Slice& key, std::string* value)`被用户调用时，会按照
  memtable、immutable、SSTable的查询顺序查询，只要查询到结果就返回给用户。当查询到SSTable时，会记录一
  个state，当用户的本次查询操作需要seek的文件超过阈值时会触发一次compaction。

## Snapshot
  `GetSnapshot`获取的是一个隐含leveldb当前sequence的数据结构，对外用户不可见。snapshot用于用户调用
  `Get`操作时使用，leveldb在存储用户的key时，扩展了8byte，用于填充记录的type和sequence值。用户调用
  `Get`操作时，会比较sequence，当用于传入一个snapshot时，大于该snapshot的sequence的记录会被隐藏。
  因此用于只能得到该早于sequence的记录。

  用户调用`GetSnapshot`时，会分配一个Snapshot数据结构，中记录当前的sequence，然后添加到leveldb的
  VersionSet中一个链表上。之所以记录全部分配的snapshot，防止compaction时，删除早先的记录。

  `ReleaseSnapshot`会是否掉VersionSet链表上的资源。

## CompactRange
  `void CompactRange(const Slice* begin, const Slice* end)`被调用时，先遍历每个level，找到与`begin`
  和`end`重叠的最大level(`max_level`)，然调用一次`Write(WriteOptions(), NULL)`，触发一次compaction。
  然后在遍历level-0到`max_level`，每个level在触发一次手动compaction。

  可见CompactRange是一个特别重的操作，因此调用时要慎之又慎。通常情况下，不需要手动调用，依赖leveldb
  自动触发的compaction即可。

## NewIterator
  调用`NewIterator`时，先调用`Iterator* NewInternalIterator(const ReadOptions& options, SequenceNumber* latest_snapshot, uint32_t* seed)`，
  锁定当前sequence作为遍历时的snapshot，选取memtable、immutable和version。从memtable、immutable中各
  生成一个iteration，从每层SSTable中生成文件的iteration，然后将这些iteration合并成一个MergingIterator。

  然后用这个MergingIterator生成DBIterator返回给用户。

  在使用DBIterator时会对遍历的数据进行采样，当遍历时seek文件达到阈值时，会触发自动的compaction。

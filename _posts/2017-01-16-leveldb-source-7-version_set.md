---
layout: post
title:  leveldb源码分析-7-Version/VersionEdit/VersionSet
date:   2017-01-16 20:00:11
category: "leveldb"
---

# Version/VersionEdit/VersionSet
  当执行一次compaction后，leveldb将在当前版本基础上创建一个新版本，当前版本就变成了
  历史版本。还有，如果你创建了一个Iterator，那么该Iterator所依附的版本将不会被leveldb
  删除。在leveldb中，Version就代表了一个版本，它包括当前磁盘及内存中的所有文件信息。
  在所有的Version中，只有一个是CURRENT。

  每个版本的sstable文件管理，主要集中在Version。Version不会修改其管理的sstable文件，
  只有读取操作。

  `VersionEdit`记录每个`Version`的相关元数据("Comparator","LogNumber","PrevLogNumber",
  "NextFile","LastSeq","DeleteFile","AddFile"等)，然后将`VersionEdit`encode后存储在`MANIFEST-XXXXX`
  文件中。

  VersionSet是所有Version的集合，用来操作全部Version。
前面讲过的VersionEdit记录了Version之间的变化，相当于delta增量，表示又增加了多少文件，删除了文件。也就是说：Version0 + VersionEdit --> Version1。
每次文件有变动时，leveldb就把变动记录到一个VersionEdit变量中，然后通过VersionEdit把变动应用到current version上，并把current version的快照，也就是db元信息保存到MANIFEST文件中。
另外，MANIFEST文件组织是以VersionEdit的形式写入的，它本身是一个log文件格式，采用log::Writer/Reader的方式读写，一个VersionEdit就是一条log record。
  leveldb 写操作包括`DB::Put()`，`DB::Delete()`，`DB::Write()`。
  其中`DB::Put()`、`DB::Delete()`都是对`DB::Write()`简化封装。

  `DB::Write()`是一个批量操作的实现。下面先分析这个`WriteBatch`。

## WriteBatch
  `WriteBatch`默认是可以拷贝构造的，定义为:

  ```
namespace leveldb {
class WriteBatch {
 public:
  WriteBatch();
  ~WriteBatch();

  // Store the mapping "key->value" in the database.
  void Put(const Slice& key, const Slice& value);

  // If the database contains a mapping for "key", erase it.  Else do nothing.
  void Delete(const Slice& key);

  // Clear all updates buffered in this batch.
  void Clear();

  // Support for iterating over the contents of a batch.
  class Handler {
   public:
    virtual ~Handler();
    virtual void Put(const Slice& key, const Slice& value) = 0;
    virtual void Delete(const Slice& key) = 0;
  };
  // 用一个handler callback遍历batch, batch中的PUT会传递给handler->Put(key, value),
  // DELETE会传递handler->Delete(key)
  Status Iterate(Handler* handler) const;

 private:
  friend class WriteBatchInternal;

  std::string rep_;  // See comment in write_batch.cc for the format of rep_
};
}  // namespace leveldb
  ```

  其使用`std::string rep_`来存储batch里的操作record，其内存分布为:

  ```
  | sequence (uint64,8bytes) | count of record (uint32,4bytes) | records... |
  ```
  首部为12bytes，存储`sequence`和record的个数n，其后是n个record。

  record的内存分布为:<br/>
    Put record为:
  ```
  | type (kTypeValue,1byte) | key_size (varint,1-5bytes) | key_data (key_size bytes) | value_size (varint,1-5bytes) | value_data (value_size bytes) |
  ```
    Delete record为:
  ```
  | type (kTypeDeletion,1byte) | key_size (varint,1-5bytes) | key_data (key_size bytes) |
  ```

  在调用`DB::Put()`，`DB::Delete()`时先填充好`WriteBatch`的data，然后调用`DB::Write()`

## DB::Write()
  在调用`DB::Write()`时会获取`DB`的全局锁。

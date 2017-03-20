---
layout: post
title:  leveldb源码分析-0-概况
date:   2016-12-29 20:00:11
category: "leveldb"
---

# leveldb 写操作
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

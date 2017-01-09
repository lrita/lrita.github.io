---
layout: post
title:  leveldb源码分析-2-comparator
date:   2017-01-06 20:00:00
category: "leveldb"
---

# Comparator
  `Comparator` 是leveldb内部对key、value进行比较排序的实现。

## 定义
  `Comparator`的定义为:

```
class Comparator {
 public:
  virtual ~Comparator();

  // Three-way comparison.  Returns value:
  //   < 0 iff "a" < "b",
  //   == 0 iff "a" == "b",
  //   > 0 iff "a" > "b"
  virtual int Compare(const Slice& a, const Slice& b) const = 0;

  // The name of the comparator.  Used to check for comparator
  // mismatches (i.e., a DB created with one comparator is
  // accessed using a different comparator.
  //
  // The client of this package should switch to a new name whenever
  // the comparator implementation changes in a way that will cause
  // the relative ordering of any two keys to change.
  //
  // Names starting with "leveldb." are reserved and should not be used
  // by any clients of this package.
  virtual const char* Name() const = 0;

  // Advanced functions: these are used to reduce the space requirements
  // for internal data structures like index blocks.

  // If *start < limit, changes *start to a short string in [start,limit).
  // Simple comparator implementations may return with *start unchanged,
  // i.e., an implementation of this method that does nothing is correct.
  virtual void FindShortestSeparator(
      std::string* start,
      const Slice& limit) const = 0;

  // Changes *key to a short string >= *key.
  // Simple comparator implementations may return with *key unchanged,
  // i.e., an implementation of this method that does nothing is correct.
  virtual void FindShortSuccessor(std::string* key) const = 0;
};
```

## 用法
  `Comparator`由`DB::Open()`时的参数`Options`传入。默认实现为`BytewiseComparator`。如果需要自定义实现，则
  继承`Comparator`自己实现一个，然后覆盖掉`Options`中的comparator即可:

```
  class TwoPartComparator : public leveldb::Comparator {
    public:
    // Three-way comparison function:
    //   if a < b: negative result
    //   if a > b: positive result
    //   else: zero result
    int Compare(const leveldb::Slice& a, const leveldb::Slice& b) const {
      int a1, a2, b1, b2;
      ParseKey(a, &a1, &a2);
      ParseKey(b, &b1, &b2);
      if (a1 < b1) return -1;
      if (a1 > b1) return +1;
      if (a2 < b2) return -1;
      if (a2 > b2) return +1;
      return 0;
    }

    // Ignore the following methods for now:
    const char* Name() const { return "TwoPartComparator"; }
    void FindShortestSeparator(std::string*, const leveldb::Slice&) const { }
    void FindShortSuccessor(std::string*) const { }
  };

  ...

  TwoPartComparator cmp;
  leveldb::DB* db;
  leveldb::Options options;
  options.create_if_missing = true;
  options.comparator = &cmp;
  leveldb::Status status = leveldb::DB::Open(options, "/tmp/testdb", &db);
```

## 实现
  leveldb内部comparator的默认实现为`BytewiseComparatorImpl`，即字典序的排序：

```
  Slice a, b;
  int Compare(const Slice& a, const Slice& b) {
    const size_t min_len = (a.size_ < b.size_) ? a.size_ : b.size_;
    int r = memcmp(a.data_, b.data_, min_len);
    if (r == 0) {
      if (a.size_ < b.size_) r = -1;
      else if (a.size_ > b.size_) r = +1;
    }
    return r;
  }
```

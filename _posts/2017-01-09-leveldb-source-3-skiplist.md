---
layout: post
title:  leveldb源码分析-3-skiplist
date:   2016-12-29 20:00:11
category: "leveldb"
---

# SkipList 跳跃列表(跳表)
  `跳跃列表`（也称跳表）是一种随机化数据结构，基于并联的链表，其效率可比拟于二叉查找树（对于大多数操作
  需要O(log n)平均时间）。
  是一种空间换时间的方法。
## 定义
  [SkipList 跳跃列表 wiki](https://zh.wikipedia.org/wiki/跳跃列表)

## 参考资料
  SkipList实现方面的参考资料还是很多，就不在赘述，具体参见一下资料:
  [跳表SkipList](http://www.cnblogs.com/xuqiang/archive/2011/05/22/2053516.html)

---
layout: post
title: boltdb 源码分析-数据结构-1
categories: [database, bolt]
description: boltdb
keywords: boltdb, database
---

# boltdb 数据结构

`boltdb`暴露给用户的数据概念较少，只有以下:
* [`Options`](https://github.com/boltdb/bolt/blob/e9cf4fae01b5a8ff89d0ec6b32f0d9c9f79aefdd/db.go#L897-L922)
初始化`boltdb`时的相关配置选择；
* [`DB`](https://github.com/boltdb/bolt/blob/e9cf4fae01b5a8ff89d0ec6b32f0d9c9f79aefdd/db.go#L45-L130)
整个`boltdb`的持有者，跟`boltdb`相关操作都要通过调用其方法发起，是`boltdb`的一个抽象；
* [`Stats`](https://github.com/boltdb/bolt/blob/e9cf4fae01b5a8ff89d0ec6b32f0d9c9f79aefdd/db.go#L932-L944)
调用`DB.Stats()`方法返回的数据结构，内包含一些`boltdb`内部的计数信息，可以供用户查看；
* [`Bucket`](https://github.com/boltdb/bolt/blob/e9cf4fae01b5a8ff89d0ec6b32f0d9c9f79aefdd/bucket.go#L36-L50)
类似于`表`的一个概念，在boltdb相关数据必须存在一个`Bucket`下，不同`Bucket`下的数据相互隔离，每个`Bucket`下
有一个单调递增的`Sequence`，类似于自增主键；
* [`BucketStats`](https://github.com/boltdb/bolt/blob/e9cf4fae01b5a8ff89d0ec6b32f0d9c9f79aefdd/bucket.go#L730-L751)
`Bucket`的一些统计信息；
* [`Tx`](https://github.com/boltdb/bolt/blob/e9cf4fae01b5a8ff89d0ec6b32f0d9c9f79aefdd/tx.go#L24-L41)
`boltdb`的事务数据结构，在`boltdb`中，全部的对数据的操作必须发生在一个事务之内，`boltdb`的并发读取都在此实现；
* [`Cursor`](https://github.com/boltdb/bolt/blob/e9cf4fae01b5a8ff89d0ec6b32f0d9c9f79aefdd/cursor.go#L18-L21)
是内部B-TREE的一个迭代器，用于遍历数据库，提供`First`/`Last`/`Seek`等操作；

还有一些内部数据结构，帮助实现一些内部逻辑:
* [`node`](https://github.com/boltdb/bolt/blob/e9cf4fae01b5a8ff89d0ec6b32f0d9c9f79aefdd/node.go#L11-L21)
用来存储每个`Bucket`中的一部分Key-Value，每个`Bucket`中的`node`组织成一个B-TREE；
* [`inode`](https://github.com/boltdb/bolt/blob/e9cf4fae01b5a8ff89d0ec6b32f0d9c9f79aefdd/node.go#L597-L602)
记录Key-Value对的数据结构，每个`node`内包含一个`inode`数组，`inode`是K-V在内存中缓存的记录，该记录落到磁盘上
时，记录为`leafPageElement`；
* [`leafPageElement`](https://github.com/boltdb/bolt/blob/e9cf4fae01b5a8ff89d0ec6b32f0d9c9f79aefdd/page.go#L110-L115)
磁盘上记录具体Key-Value的索引；
* [`page`](https://github.com/boltdb/bolt/blob/e9cf4fae01b5a8ff89d0ec6b32f0d9c9f79aefdd/page.go#L30-L36)
用户记录持久化文件中每个区域的重要信息，同时`page`分为很多种类，不同`page`存储不同的数据信息；

---
layout: post
title: boltdb 源码分析-启动和初始化-2
categories: [database, bolt]
description: boltdb
keywords: boltdb, database
---

# boltdb 启动和初始化
根据前面的简述，`boltdb`只使用单个文件进行持久化，在`boltdb`启动时，会先打开对应的
持久化文件，然后根据是否是只读模式来获取该文件的文件锁。当不是只读模式时，`boltdb`
会采用排他文件锁来保证只有一个进程能够操作该文件，避免多个进程同时操作，造成数据毁坏。

当持久化文件大小为0时，`boltdb`会认为该数据库为新创建的，然后对其进行初始化(写入4个初
始page)，然后将文件映射到内存空间，最后从映射的内存中读取`meta0`、`meta1`数据。

然后从从meta数据中标识的当前freelist页来读取freelist pgid数据，然后freelist根据这些
数据构建free page cache。即用一个map来记录那些page是空闲的。

此时即完成启动和初始化。

由于`boltdb`更新Key-Value操作时，直接通过B-TREE索引直接将数据写到指定的文件page中，因此
仅有一次写入并且不需要log来维护`boltdb`的crash一致性。

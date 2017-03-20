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

  `VersionEdit`记录每个`Version`的相关修改元数据("Comparator","LogNumber","PrevLogNumber",
  "NextFile","LastSeq","DeleteFile","AddFile"等)，相当于增量信息，然后将`VersionEdit`encode
  后存储在`MANIFEST-XXXXX`文件中。

  VersionSet是所有Version的集合，用来操作全部Version。

  每次文件有变动时，leveldb就把变动记录到一个VersionEdit变量中，然后通过VersionEdit把变动
  应用到current version上，并把current version的快照，也就是db元信息保存到MANIFEST文件中。

## VersionEdit
  每一次compaction，都好比是生成了一个新的db版本，对应的Menifest则保存着这个版本的db元信息。
  VersionEdit并不操作文件，只是为Manifest文件读写准备好数据、从读取的数据中解析出DB元信息。

## Version
  `Version`就是一个sstable文件集合，`Version`记录在`VersionSet`内的一个双向链表上。

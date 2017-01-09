---
layout: post
title:  leveldb源码分析-0-概况
date:   2016-12-29 20:00:11
category: "leveldb"
---

# 简介
  `leveldb`是一款google开发的高性能，单机嵌入式k-v存储，广泛被用作各种database engine。

# database文件
  `leveldb`创建后，`dbname`目录下会产生一系列的小文件，由这些小文件共同存储database中的数据。

  总共拥有的文件名称有:

```
    dbname/CURRENT
    dbname/LOCK
    dbname/LOG
    dbname/LOG.old
    dbname/MANIFEST-[0-9]+
    dbname/[0-9]+.(log|sst|ldb)
```

  不同的文件根据后缀来区分功能。

* `LOCK` database文件锁，`leveldb`通过文件锁来避免同一个db被多次打开操作。调用`DB::Open()`时会先获取该文件锁。
* `MANIFEST-XXXXX`，描述文件。
* `CURRENT` 表明当前正在使用哪个`MANIFEST`文件
* `LOG`，`LOG.old` 运行日志文件，当未指定`options.info_log`时，默认会将错误日志输出到该文件，每一次`DB::Open()`时，会将`LOG`重命名为`LOG.old`
* `.log` 日志文件
* `.dbtmp` 变更文件内容，中间过程中的临时文件。

![LevelDB架构](/images/posts/leveldb/leveldb.jpg)

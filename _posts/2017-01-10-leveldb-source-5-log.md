---
layout: post
title:  leveldb源码分析-5-log
date:   2017-01-10 20:00:11
category: "leveldb"
---

## log

  通常来说解析操作语句、执行语句相关的动作会是一个耗时较长(相对写log来说)的操作，为了保证数据在极
  端情况下的安全，多数database都实现了binlog的机制，即先将每次操作的记录写到binlog里，然后再进行
  相关的其他db相关的操作。
  在每次重启或恢复时进行校验、读取log等操作保证操作数据的一致性。通常来说成功写到binlog里的操作就
  不会发生丢失。

  leveldb也实现的类似的log机制，db目录下于此相关的文件为`dbname/[0-9]+.log`。

## log format

  在leveldb源码目录中有[`doc/log_format.txt`](https://github.com/google/leveldb/blob/master/doc/log_format.txt)
  详细描述了log文件的格式。

  log文件按`block`来划分，默认每个`block`大小为32KB。block由连续的log record组成。每个record的格式为:

```
  | crc32 (4 bytes) | length (2 bytes) | log type (1 byte) | log data |
```

  log type有:

* `kZeroType` = 0    enum 边界标识，正常情况下不应该出现
* `kFullType` = 1    表明该log record包含了完整的user record
*                    以下3个出现时，说明user record可能内容很多，超过了block的可用大小，就需要分成几条log record。
* `kFirstType` = 2   说明是record分片的开头
* `kMiddleType` = 3  说明是record分片的中间部分
* `kLastType` = 4    说明是record分片的结尾

  参看官方文档上的例子：

```
  有3个连续的user record:
    A: length 1000
    B: length 97270
    C: length 8000
```

*  A应该作为`kFullType`类型的record存储在第1个block中
*  B由于超过了32KB，会被拆分为3条log record，分别存储在第1、2、3个block中，这时第三个block还有6 bytes，将被填充为0(record写入的最小要求为`kHeaderSize`(7))
*  C将作为`kFullType`类型的record存储在第4个block中

![leveldb-log-block](/images/posts/leveldb/leveldb-log-block.jpg)

## user record
  (待完成)

## `log::Writer`
  用于将log record写入文件，非常简单，具体参见源码分析。

## `log::Reader`
  用于将log record从文件读出，非常简单，具体参见源码分析。

## 源码分析

  [db/log_format.h](https://github.com/lrita/leveldb/blob/master/db/log_format.h)<br/>
  [db/log_writer.h](https://github.com/lrita/leveldb/blob/master/db/log_writer.h)<br/>
  [db/log_writer.cc](https://github.com/lrita/leveldb/blob/master/db/log_writer.cc)<br/>
  [db/log_reader.h](https://github.com/lrita/leveldb/blob/master/db/log_reader.h)<br/>
  [db/log_reader.cc](https://github.com/lrita/leveldb/blob/master/db/log_reader.cc)

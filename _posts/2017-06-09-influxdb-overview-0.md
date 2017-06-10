---
layout: post
title: influxdb 源码分析-概述-0
categories: [database, influxdb]
description: database influxdb
keywords: database influxdb
---

# influxdb
[influxdb](https://github.com/influxdata/influxdb) 是个比较有名的时序数据库，相关的简介
在网上已经比较多了，再次就不在赘述了。相关的部署、操作相关的可以通过搜索找到。

今后会花一些时间来研究该时序数据库，主要研究其`tsdb`部分。`influxdb`该工程总共逻辑代码量
有7万多行，连同单元测试一共有不到12万行。相对来说还是算中等量级的一个工程。由于其分布式
集群部分的逻辑不再开源，用于商业化。因此目前版本的集群部分的逻辑还是比较简单的。

# influxdb startup
`influxd`是`influxdb`对外提供服务的进程，与该database主体相关的逻辑，可以通过该进程的入口
作为切入点来分析。

当用户运行`influxd run`时可以启动一个`influxdb`实例，可以通过[`run`](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/cmd/influxd/main.go#L80-L114)
作为切入点来进入。

首先是打印logo、解析(合并)配置等一些动作，在校验配置正确后，实例化[`Server`](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/cmd/influxd/run/server.go#L102-L209)
，再[启动](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/cmd/influxd/run/server.go#L348-L437)`Server`。

## Server.Run
在`Server`启动时，会创建其TCP服务，然后在该TCP端口上启动一个复用器，使得多种数据协议都使用同一个端口来通信。
其实并没有很多模块复用这个端口。

然后启动`meta`、`monitor`、`precreator`、`snapshotter`、`continuous_querier`、`retention`、`subscriber`、
`httpd`、多协议接收等模块。后面会讲到这些模块。

`influxdb`为了争夺市场，因此兼容了很多其他竞品的数据接收格式，如`OpenTSDB`/`Graphite`/`Statsd`等。

### tcp复用器
tcp复用器的实现有很多种，通常是根据新建连接的开头几个字节来猜测/或直接判断该链接的协议，然后根据相关协议使用
不同的处理回调。相关类似实现有：

* [github.com/influxdata/influxdb/tcp/mux](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/tcp/mux.go)
* [github.com/hashicorp/yamux](https://github.com/hashicorp/yamux)
* [github.com/soheilhy/cmux](https://github.com/soheilhy/cmux)

### meta
`influxdb`将meta数据库，包装成一个`MetaClient`对外提供数据，需要meta的模块都引用这个`MetaCient`。该`meta.db`
直接使用protobuf格式的数据作为持久化文件。meta加载持久化文件后，会将全部内容缓存在内存中。当有meta改写时，
`MetaClient`会将更新后的数据序列化然后写入磁盘中。

`MetaClient`一部分数据已slice的形式存储，很多api都会将该slice返回给调用方，从而脱离了其锁的保护，有数据并发竞争
访问的问题存在。

`meta.db`中存储每个`database`的元数据(名称、过期策略、ContinuousQuery)和用户信息。

### coordinator
`coordinator`是`tsdb`的一个封装，原本是提供分布式存储、保证数据一致性等功能，但是现在由于集群功能不再开源，其中
这部分逻辑也已经被移除，现在仅仅是`tsdb`的代理。

### monitor
`monitor`模块是`influxd`用来给用户提供一些统计、诊断数据的模块。`influxd`有[相关文档](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/monitor/README.md)
专门介绍了该项功能，需要给对外暴露相关信息的模块可以调用`Monitor.RegisterDiagnosticsClient()`方法来注册到该
模块中来。当相关接口被调用时，会读取注册模块的数据。

### precreator
`influxd`将时序数据按时间区间来切分文件，被切分的块称之为`shard`。当`influxdb`组成一个集群时，`shard`写时创建
可能会在触发切换文件的一瞬间影响进程的吞吐量，因此采用预先创建`shard`的方式，使得文件总是预先被创建的，写请求
不会被切换文件的动作阻塞。

`precreator`模块就是创建一个定时函数，每隔一定时间间隔，检查并按需预先创建一次`shard`。

### snapshotter
`snapshotter`会复用Server的端口，接受并处理跟snapshot相关的命令。相关的命令有：

* `RequestShardBackup` 读取通过shard-id和time标识的一段tsdb的数据，返回给调用方。备份命令通常通过该接口读取数据。
* `RequestMetastoreBackup` 读取meta数据
* `RequestDatabaseInfo` 读取指定`database`(就是influxdb操作中的database的概念，如use database xxx)的shards的路径，
* `RequestRetentionPolicyInfo` 读取指定`database`和`retention policy`的shard的路径

### continuous_querier
`continuous_query`是`influxdb`中非常重要的一个概念，通常用来压缩/聚合数据。该模块就运行这些逻辑。

### retention
`influxdb`通常存储一些监控、统计数据，通常来说，这些数据并不需要一直存储着，当数据超过一定期限时，这些数据
就失去了保留的意义。同时`influxdb`的存储空间也不是无限的。因此`influxdb`就内置了数据过期自动删除的逻辑。对于
`influxdb`上的每个`database`可以设置一些过期策略，满足这些策略时，`influxdb`就是删除满足条件的shard。

`retention`就是执行这些过期策略的模块。该模块定期检查每个`database`的过期策略，将满足过期条件的ShardGroup删除，
然后再将与该ShardGroup相关的shard删除。

### subscriber
`subscriber`处理`influxdb`中`SUBSCRIPTIONS`部分的逻辑，将内部通过`coordinator.PointsWriter`写入的数据发送到
`database`中配置的`SUBSCRIPTIONS`指定的URI上，采用推的模式。有点类似`Watch`的机制。

各个数据接收模块都会使用到`coordinator.PointsWriter`。

### httpd
`httpd`是`influxdb`对外提供服务的主要协议，用户可以通过http协议查询数据、数据写入、修改database、增加用户等操作。对外
提供的了一些[API](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/services/httpd/handler.go#L121-L156)
和`/debug/pprof`、`/debug/expvar`、`/debug/requests`接口。

### 其他数据接收模块
`influxdb`兼容了主流的多种监控数据接收协议有`graphit`/`opentsdb`/`collectd`。这些模块接收到相关数据后，将数据
解析成统一格式[Point](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/models/points.go#L47-L120)
然后通过[`coordinator.PointsWriter.WritePointsPrivileged()`](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/coordinator/points_writer.go#L293-L355)
/[`coordinator.PointsWriter.WritePoints()`](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/coordinator/points_writer.go#L288-L290)
/[`coordinator.PointsWriter.WritePointsInto()`](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/coordinator/points_writer.go#L283-L285)
写入`tsdb`。

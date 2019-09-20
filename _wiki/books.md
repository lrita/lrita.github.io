---
layout: wiki
title: 编程书籍/文档
categories: program books
description: 编程书籍/文档
keywords: 编程书籍/文档
---

# assembly
* [X86汇编简介](/images/posts/tools/CS356Unit4_x86_ISA.pdf)

# cpp
* [cmake实践](/images/posts/cplusplus/CMake-Practice.pdf)
* [精通cmake](/images/posts/cplusplus/mastering-cmake.pdf)
* [如何使用identity元函数禁止模板自动推导](/images/posts/cplusplus/the-identity-metafunction.pdf)

# go
* [go tool trace](/images/posts/go/Rhys-Hiltner-go-tool-trace-GopherCon-2017.pdf)

# rust
* [在 Mac 下面调优 TiKV](https://www.jianshu.com/p/a80010878def) MacOS下调优RUST的方法

# linux
* [深度分析Linux下双网卡绑定七种模式](/images/posts/linux/深度分析Linux下双网卡绑定七种模式.pdf)

# 文件系统
* [Linux文件空洞与稀疏文件](/images/posts/filesystem/Linux_File_Hole_And_Sparse_Files.pdf) 讲解linux文件系统下文件空洞与稀疏文件

# 调优诊断
* [看懂vmstat/mpstat数据诊断](/images/posts/filesystem/Extreme-Linux-Performance-Monitoring-and-Tuning.pdf)
* [Linux CPU 占用率原理与 精确度分析](/images/posts/linux/Linux_CPU_Usage_Analysis.pdf)
* [震惊，用了这么多年的 CPU 利用率，其实是错的](https://mp.weixin.qq.com/s/KaDJ1EF5Y-ndjRv2iUO3cA)：如何正确理解CPU使用率，找到系能问题所在。
* [Linux Systems Performance in 50 mins](/images/posts/linux/Percona2016_LinuxSystemsPerf.pdf)
* [Linux Performance Tools](/images/posts/debug/Linux.Performance.Tools.Oct.2014.pdf) 主要讲Linux系统各方面都有哪些debug工具（观察、系能压测、调优、统计）、debug方法论
* [Linux Performance Tools - 2015](/images/posts/debug/velocity2015linuxperftools-150527215912-lva1-app6891.pdf) 主要讲Linux系统各方面都有哪些debug工具，其中内容比👆更丰富一点。

## 火焰图
* [Blazing Performance with Flame Graphs](/images/posts/debug/LISA13_Flame_Graphs.pdf) 火焰图入门，教如何看图、生成图
* [Linux CPI FlameGraph](http://oliveryang.net/2018/03/linux-CPI-flamegraph/) 识别负载类型(CPU/MEMORY Bound) 当CPU使用率高时，CPU行为分析，哪些stall行为在浪费CPU。如何生成CPI火焰图，如何看图。
* [Linux Profiling At Netflix](/images/posts/debug/Linux.Profiling.at.Netflix.Feb.2015.pdf) 如何使用`perf`，可能会遇到哪些问题（栈损坏\符号丢失），如何修复它们。

## SystemTap
* [Using SystemTap with Linux](/images/posts/systemtap/Using-SystemTap-with-Linux.pdf) SystemTap 入门，将一些基本的语法、数据结构

# database
* [基于HLC的分布式事务实现深度剖析](/images/posts/database/基于HLC的分布式事务实现深度剖析.pdf)
* [数据库系统的优化与调优：从理论到实践](/images/posts/database/数据库系统的优化与调优：从理论到实践.pdf)
* [利用新硬件提升数据库性能.pptx](/images/posts/database/利用新硬件提升数据库性能.pptx)

### mysql
* [深入解析MySQL replication协议](https://www.jianshu.com/p/5e6b33d8945f)

### canssandra
* [CASSANDRA实战](/images/posts/database/canssandra/CASSANDRA实战[白色].pdf)
* [Cassandra权威指南](/images/posts/database/canssandra/Cassandra权威指南.pdf)

### clickhouse
* [MySQL-DBA解锁数据分析的新姿势-ClickHouse-新浪](/images/posts/database/clickhouse/MySQL-DBA解锁数据分析的新姿势-ClickHouse-新浪.pdf)

# 数据结构
* [Spin](http://spinroot.com/) 一个形式化验证多线程数据结构的工具
* [优化的LRU算法：Outperforming LRU with an Adaptive Replacement Cache Algorithm](/images/posts/datastructure/ARC.pdf)
* [各种数据结构的可视化展示](https://www.cs.usfca.edu/~galles/visualization/Algorithms.html)

# API 设计
* [GRPC API Design Guide](https://cloud.google.com/apis/design/)

# 数学理论
* [排队论及其应用浅析](/images/posts/math/排队论及其应用浅析.pdf)
* [排队论](http://netedu.xauat.edu.cn/jpkc/netedu/jpkc/ycx/kcjy/kejian/pdf/09.pdf)

# 工具
* [latex入门-简版-刘海洋](/images/wiki/latex入门-简版-刘海洋.pdf)
* [SRE-Google运维解密](/images/blog/SRE-Google.pdf)
* [SRE工作手册（google开源书）](/images/posts/com/the-site-reliability-workbook-next18.pdf)
* [给 TiKV 开发 Grafana 的 datasource](https://www.jianshu.com/p/057fe9e57274) 资源提供API配合Grafana，提供监控的新思路

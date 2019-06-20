---
layout: post
title: 提升go编译器内联程度
categories: [go]
description: go golang inline aggressiveness
keywords: go golang inline aggressiveness
---

根据[Proposal: Mid-stack inlining in the Go compiler](https://go.googlesource.com/proposal/+/master/design/19348-midstack-inlining.md)，当前`go1.9+`已经支持用户调整编译器的内联程度。

可以使用编译参数`go build -gcflags="-l=4"`增加内联程度，从而提升9%的运行速度，代价是生成的二进制程序大小提升11-15%，看起来是很不错的一个交换。

我们可以通过添加`-m=2`参数来对比效果。我们可以使用下列命令来对比前后内联的程度，数据结果越少，表示内联的函数越多。
```shell
# 优化前
go build -gcflags="-m=2" 2>&1 | grep "too complex"
# 优化后
go build -gcflags="-m=2 -l=4" 2>&1 | grep "too complex"
```

根据[cmd/compile/internal/gc/inl.go](https://github.com/golang/go/blob/71a6a44428feb844b9dd3c4c8e16be8dee2fd8fa/src/cmd/compile/internal/gc/inl.go#L10-L17)源码显示，`-l`的取值为：
```go
// The debug['l'] flag controls the aggressiveness. Note that main() swaps level 0 and 1,
// making 1 the default and -l disable. Additional levels (beyond -l) may be buggy and
// are not supported.
//      0: disabled
//      1: 80-nodes leaf functions, oneliners, panic, lazy typechecking (default)
//      2: (unassigned)
//      3: (unassigned)
//      4: allow non-leaf functions
```

当然，一味的吹求函数内联并不总能带来性能的提高，因为随着代码段大小的提升，CPU运行时`instruction cache miss`会随之增加，过高时会[适得其反](https://www.scylladb.com/2017/07/06/scyllas-approach-improve-performance-cpu-bound-workloads/)，所以是否采用还要以实际的PROFILE为准绳。
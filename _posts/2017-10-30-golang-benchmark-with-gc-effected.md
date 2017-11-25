---
layout: post
title: golang gc/arch 对 benchmark 的影响
categories: [golang go]
description: golang go
keywords: golang go
---
最近在同事提出了一个疑问：
在对一个slice进行遍历时，将`for`循环条件中的`len`提出到循环外是否会比golang编译器的优化结果更加好。

即：
```go
func g0(a []int) int {
    l := len(a)
    for i := 0; i < l; i++ {
    }
    return 1
}
```
是否会比
```go
func g1(a []int) int {
    for i := 0; i < len(a); i++ {
    }
    return 1
}
```
的结果更加优化(目前golang的编译器并不会对这个空循环进行消除)。

那为了证明这个问题，那就上 benchmark 证明啊。

```go
import "testing"

var a = make([]int, 1<<25)

func BenchmarkG0(b *testing.B) {
    for i := 0; i < b.N; i++ {
        g0(a)
    }
}

func BenchmarkG1(b *testing.B) {
    for i := 0; i < b.N; i++ {
        g1(a)
    }
}
```

然后执行
```shell
go test -c .
./len.test -test.bench=. -test.count=2
```

得到输出结果：
```
goos: darwin
goarch: amd64
BenchmarkG0-4            100      11784627 ns/op
BenchmarkG0-4            100      11841061 ns/op
BenchmarkG1-4            100      18623122 ns/op
BenchmarkG1-4            100      17790754 ns/op
PASS
```

果然`g0`比`g1`速度快很多，但是这个有点反常识啊，不能这样轻易下定结论。那我们来看看`g0`的编译结果
是否就比`g1`优化很多：

我们来执行
```shell
go tool objdump ./len.test > main.s
```

我们来看得到的结果：
```
TEXT _/test/go/len.g0(SB) /test/go/len/main.go
  main.go:4             0x10ef150               488b442410              MOVQ 0x10(SP), AX
  main.go:4             0x10ef155               31c9                    XORL CX, CX
  main.go:6             0x10ef157               eb03                    JMP 0x10ef15c
  main.go:6             0x10ef159               48ffc1                  INCQ CX
  main.go:6             0x10ef15c               4839c1                  CMPQ AX, CX
  main.go:6             0x10ef15f               7cf8                    JL 0x10ef159
  main.go:8             0x10ef161               48c744242001000000      MOVQ $0x1, 0x20(SP)
  main.go:8             0x10ef16a               c3                      RET
  :-1                   0x10ef16b               cc                      INT $0x3
  :-1                   0x10ef16c               cc                      INT $0x3
  :-1                   0x10ef16d               cc                      INT $0x3
  :-1                   0x10ef16e               cc                      INT $0x3
  :-1                   0x10ef16f               cc                      INT $0x3

TEXT _/test/go/len.g1(SB) /test/go/len/main.go
  main.go:12            0x10ef170               488b442410              MOVQ 0x10(SP), AX
  main.go:12            0x10ef175               31c9                    XORL CX, CX
  main.go:13            0x10ef177               eb03                    JMP 0x10ef17c
  main.go:13            0x10ef179               48ffc1                  INCQ CX
  main.go:13            0x10ef17c               4839c1                  CMPQ AX, CX
  main.go:13            0x10ef17f               7cf8                    JL 0x10ef179
  main.go:15            0x10ef181               48c744242001000000      MOVQ $0x1, 0x20(SP)
  main.go:15            0x10ef18a               c3                      RET
  :-1                   0x10ef18b               cc                      INT $0x3
  :-1                   0x10ef18c               cc                      INT $0x3
  :-1                   0x10ef18d               cc                      INT $0x3
  :-1                   0x10ef18e               cc                      INT $0x3
  :-1                   0x10ef18f               cc                      INT $0x3
```

我们可以看到编译器生成的中间代码完全是相同的，那为什么在运行起来会有不同的结果呢？
那我们就要考虑了，除了代码以外，还有什么会影响代码执行？那就是：

* 运行环境
* runtime

那我们就分别对这两个因素进行验证。

首先是运行环境，我们换到`linux`上再进行一次验证：
```shell
GOOS=linux GOARCH=amd64 go test -c .
## copy to linux
./test.len -test.bench=. -test.count=2
```
得到输出结果：
```
goos: linux
goarch: amd64
BenchmarkG0-32             100      10824437 ns/op
BenchmarkG0-32             100      10743979 ns/op
BenchmarkG1-32             100      10740347 ns/op
BenchmarkG1-32             100      10898047 ns/op
PASS
```
在`linux`上`g0`/`g1`的表现是相同的。那我们就要考虑了`linux`和`darwin`有哪些不同？这个可就多了，
没有办法去一一对比了。但是这些区别会很大程度反映到`runtime`上。

那我们就对`runtime`进行比较。那`runtime`中的什么会影响到程序的运行？可能有：（未列举全）

* 函数栈空间的扩展
* goroutine的调度
* io/syscall/cgo
* gc

我们从上面的objdump的结果来看，生成的代码应该跟前3个因素都无关。那我们就尝试关闭gc，再进行一次比较:

```go
import (
    "runtime/debug"
    "testing"
)

func init() {
    debug.SetGCPercent(-1)
}

var a = make([]int, 1<<25)

func BenchmarkG0(b *testing.B) {
    for i := 0; i < b.N; i++ {
        g0(a)
    }
}

func BenchmarkG1(b *testing.B) {
    for i := 0; i < b.N; i++ {
        g1(a)
    }
}
```

然后
```
go test -c .
./len.test -test.bench=. -test.count=2
```
得到结果：
```
goos: darwin
goarch: amd64
BenchmarkG0-4            100      11521770 ns/op
BenchmarkG0-4            100      11310217 ns/op
BenchmarkG1-4            100      11562763 ns/op
BenchmarkG1-4            100      11590019 ns/op
PASS
```

可以看出关闭`gc`后，`g0`/`g1`表现相同，那是因为`g1`在运行过程中会动态分配内存么？上面objdump的结果来看，
明显不是。那只有可能是gc运行的时机问题，为什么gc偏巧要在`g1`运行时启动呢？这个就比较微妙了，`runtime`的
表现跟系统很多因素有关系，相同的代码在不同的操作系统上也有微妙的差异，或许有某种伪随机的因素在`runtime`中？
还未求证。

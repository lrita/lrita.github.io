---
layout: post
title: golang 汇编
categories: [golang, go]
description: golang go
keywords: golang go
---

在某些场景下，我们需要进行一些特殊优化，因此我们可能需要用到golang汇编，golang汇编源于plan9，此方面的
介绍很多，就不进行展开了。我们WHY和HOW开始讲起。

golang汇编相关的内容还是很少的，而且多数都语焉不详，而且缺乏细节。对于之前没有汇编经验的人来说，是很难
理解的。而且很多资料都过时了，包括官方文档的一些细节也未及时更新。因此需要掌握该知识的人需要仔细揣摩，
反复实验。

## WHY
我们为什么需要用到golang的汇编，基本出于以下场景。

* [算法加速](https://github.com/minio/sha256-simd)，golang编译器生成的机器码基本上都是通用代码，而且
优化程度一般，远比不上C/C++的`gcc/clang`生成的优化程度高，毕竟时间沉淀在那里。因此通常我们需要用到特
殊优化逻辑、特殊的CPU指令让我们的算法运行速度更快，如`sse4_2/avx/avx2/avx-512`等。
* 摆脱golang编译器的一些约束，如[通过汇编调用其他package的私有函数](https://sitano.github.io/2016/04/28/golang-private/)。
* 进行一些hack的事，如[通过汇编适配其他语言的ABI来直接调用其他语言的函数](https://github.com/petermattis/fastcgo)。
* 利用`//go:noescape`进行内存分配优化，golang编译器拥有逃逸分析，用于决定每一个变量是分配在堆内存上
还是函数栈上。但是有时逃逸分析的结果并不是总让人满意，一些变量完全可以分配在函数栈上，但是逃逸分析将其
移动到堆上，因此我们需要使用golang编译器的[`go:noescape`](https://golang.org/cmd/compile/#hdr-Compiler_Directives)
将其转换，强制分配在函数栈上。当然也可以强制让对象分配在堆上，可以参见[这段实现](https://github.com/golang/go/blob/d1fa58719e171afedfbcdf3646ee574afc08086c/src/reflect/value.go#L2585-L2597)。

## HOW
使用到golang会汇编时，golang的对象类型、buildin对象、语法糖还有一些特殊机制就都不见了，全部底层实现
暴露在我们面前，就像你拆开一台电脑，暴露在你面前的是一堆PCB、电阻、电容等元器件。因此我们必须掌握一些
go ABI的机制才能进行golang汇编编程。

### go汇编简介
这部分内容可以参考:

* [1](https://golang.org/doc/asm)
* [2](https://github.com/yangyuqian/technical-articles/blob/master/asm/golang-plan9-assembly-cn.md)

go 汇编中有4个核心的伪寄存器，这4个集群器是编译器用来维护上下文、特殊标识等作用的：
* FP(Frame pointer): arguments and locals
* PC(Program counter): jumps and branches
* SB(Static base pointer): global symbols
* SP(Stack pointer): top of stack

所有用户空间的数据都可以通过FP(局部数据、输入参数、返回值)或SB(全局数据)访问。
通常情况下，不会对`SB`/`FP`寄存器进行运算操作，通常情况以会以`SB`/`FP`作为基准地址，进行偏移解引用
等操作。

而且在某些情况下`SB`更像一些声明标识，其标识语句的作用。例如：

1. `TEXT runtime·_divu(SB), NOSPLIT, $16-0` 在这种情况下，`TEXT`、`·`、`SB`共同作用声明了一个函数
`runtime._divu`，这种情况下，不能对`SB`进行解引用。
2. `GLOBL fast_udiv_tab<>(SB), RODATA, $64` 在这种情况下，`GLOBL`、`fast_udiv_tab`、`SB`共同作用，
在RODATA段声明了一个私有全局变量`fast_udiv_tab`，大小为64byte，此时可以对`SB`进行偏移、解引用。
3. `CALL    runtime·callbackasm1(SB)` 在这种情况下，`CALL`、`runtime·callbackasm1`、`SB`共同标识，
标识调用了一个函数`runtime·callbackasm1`。
4. `MOVW    $fast_udiv_tab<>-64(SB), RM` 在这种情况下，与2类似，但不是声明，是解引用全局变量
`fast_udiv_tab`。

`FP`伪集群器用来标识函数参数、返回值。例如`0(FP)`表示函数参数其实的位置，`8(FP)`表示函数参数偏移8byte
的位置。如果操作命令是`MOVQ arg+8(FP), AX`的话，`MOVQ`表示对8byte长的内存进行移动，其实位置是函数参数偏移8byte
的位置，目的是寄存器`AX`，因此此命令为将一个参数赋值给寄存器`AX`，参数长度是8byte，可能是一个uint64，`FP`
前面的`arg+`是标记。至于`FP`的偏移怎么计算，会在后面的[go函数调用](#go函数调用)中进行表述。同时我们
还可以在命令中对`FP`的解引用进行标记，例如`first_arg+0(FP)`将`FP`的起始标记为参数`first_arg`，但是
`first_arg`只是一个标记，在汇编中`first_arg`是不存在的，不能直接引用`first_arg`。但是go汇编编译器强制
要求我们为每一次`FP`解引用加上一个标记，可能是为了可读性。

`SP`是栈指针寄存器，指向当前函数栈的栈顶，可以向`+`方向解引用，即向高地址偏移，可以获取到`FP`指向的范围
(函数参数、返回值)，例如`p+32(SP)`。也可以向`-`方向解引用，即向低地址偏移，访问函数栈上的局部变量，例如
`p-16(SP)`。由于可以对`SP`进行赋值运算，通常接触到的代码不会向`-`方向解引用，而是使用命令将`SP`的值减少
，例如`SUBQ    $24, SP`将`SP`减少24，则此时的`p+0(SP)`等于减之前的`p-24(SP)`。

注意，当`SP`寄存器操作时，如果前面没有指示参数时，则代表的是硬件栈帧寄存器`SP`，此处需要注意。

对于函数控制流的跳转，是用label来实现的，label只在函数内可见，类似`goto`语句：

```asm
next:
  MOVW $0, R1
  JMP  next
```

#### 文件命名
使用到汇编时，即表明了所写的代码不能够跨平台使用，因此需要针对不同的平台使用不同的汇编
代码。go编译器采用文件名中加入平台名后缀进行区分。

比如`sqrt_386.s  sqrt_amd64p32.s  sqrt_amd64.s  sqrt_arm.s`

或者使用`+build tag`也可以，详情可以参考[go/build](https://golang.org/pkg/go/build/)。

#### 函数声明
首先我们先需要对go汇编代码有一个抽象的认识，因此我们可以先看一段go汇编代码：
```asm
TEXT runtime·profileloop(SB),NOSPLIT,$8
  MOVQ    $runtime·profileloop1(SB), CX
  MOVQ    CX, 0(SP)
  CALL    runtime·externalthreadhandler(SB)
  RET
```

此处声明了一个函数`profileloop`，函数的声明以`TEXT`标识开头，以`${package}·${function}`为函数名。
如何函数属于本package时，通常可以不写`${package}`，只留`·${function}`即可。`·`在mac上可以用`shift+option+9`
打出。`$8`表示该函数栈大小为8byte。当有`NOSPLIT`标识时，可以不写输入参数、返回值的大小。

那我们再看一个函数：

```asm
TEXT ·add(SB),$0-24
  MOVQ x+0(FP), BX
  MOVQ y+8(FP), BP
  ADDQ BP, BX
  MOVQ BX, ret+16(FP)
  RET
```

该函数等同于：
```go
func add(x, y int64) int {
    return x + y
}
```

该函数没有局部变量，故`$`后第一个数为0，但其有2个输入参数，一个返回值，则第二个数为24(3\*8byte)。

#### 全局变量声明
以下就是一个私有全局变量的声明，`<>`表示该变量只在该文件内全局可见。
全局变量的数据部分采用`DATA    symbol+offset(SB)/width, value`格式进行声明。

```asm
DATA divtab<>+0x00(SB)/4, $0xf4f8fcff  // divtab的前4个byte为0xf4f8fcff
DATA divtab<>+0x04(SB)/4, $0xe6eaedf0  // divtab的4-7个byte为0xe6eaedf0
...
DATA divtab<>+0x3c(SB)/4, $0x81828384  // divtab的最后4个byte为0x81828384
GLOBL divtab<>(SB), RODATA, $64        // 全局变量名声明，以及数据所在的段"RODATA"，数据的长度64byte
```

```
GLOBL runtime·tlsoffset(SB), NOPTR, $4 // 声明一个全局变量tlsoffset，4byte，没有DATA部分，因其值为0。
                                       // NOPTR 表示这个变量数据中不存在指针，GC不需要扫描。
```

类似`RODATA`/`NOPTR`的特殊声明还有：
* NOPROF = 1  (For TEXT items.) Don't profile the marked function. This flag is deprecated.
* DUPOK = 2  It is legal to have multiple instances of this symbol in a single binary. The linker will choose one of the duplicates to use.
* NOSPLIT = 4 (For TEXT items.) Don't insert the preamble to check if the stack must be split. The frame for the routine, plus anything it calls, must fit in the spare space at the top of the stack segment. Used to protect routines such as the stack splitting code itself.
* RODATA = 8 (For DATA and GLOBL items.) Put this data in a read-only section.
* NOPTR = 16 (For DATA and GLOBL items.) This data contains no pointers and therefore does not need to be scanned by the garbage collector.
* WRAPPER = 32 (For TEXT items.) This is a wrapper function and should not count as disabling recover.
* NEEDCTXT = 64 (For TEXT items.) This function is a closure so it uses its incoming context register.

#### 局部变量声明
局部变量存储在函数栈上，因此不需要额外进行声明，在函数栈上预留出空间，使用命令操作这些内存即可。因此这些
局部变量没有标识，操作时，牢记局部变量的分布、内存偏移即可。

### buildin类型
在golang汇编中，没有`struct/slice/string/map/chan/interface{}`等类型，有的只是寄存器、内存。因此我们需要了解这些
类型对象在汇编中是如何表达的。

#### `(u)int??/float??`
`uint32`就是32bit长的一段内存，`float64`就是64bit长的一段内存，其他相似类型可以以此类推。

#### `int/unsafe.Pointer/unint`
在32bit系统中`int`等同于`int32`，`uintptr`等同于`uint32`，`unsafe.Pointer`长度32bit。

在64bit系统中`int`等同于`int64`，`uintptr`等同于`uint64`，`unsafe.Pointer`长度64bit。

`byte`等同于`uint8`。`rune`等同于`int32`。

`string`底层是[`StringHeader`](https://github.com/golang/go/blob/d1fa58719e171afedfbcdf3646ee574afc08086c/src/reflect/value.go#L1783-L1786)
这样一个结构体，`slice`底层是[`SliceHeader`](https://github.com/golang/go/blob/d1fa58719e171afedfbcdf3646ee574afc08086c/src/reflect/value.go#L1800-L1804)
这样一个结构体。

#### `map`
`map`是指向[`hmap`](https://github.com/golang/go/blob/d1fa58719e171afedfbcdf3646ee574afc08086c/src/runtime/hashmap.go#L107-L122)
的一个`unsafe.Pointer`

#### `chan`
`chan`是指向[`hchan`](https://github.com/golang/go/blob/d1fa58719e171afedfbcdf3646ee574afc08086c/src/runtime/chan.go#L31-L50)
的一个`unsafe.Pointer`

#### `interface{}`
`interface{}`是[`eface`](https://github.com/golang/go/blob/d1fa58719e171afedfbcdf3646ee574afc08086c/src/runtime/runtime2.go#L148-L151)
这样一个结构体。详细可以参考[深入解析GO](https://tiancaiamao.gitbooks.io/go-internals/content/zh/07.2.html)

### go函数调用
通常函数会有输入输出，我们要进行编程就需要掌握其ABI，了解其如何传递输入参数、返回值、调用函数。

go汇编使用的是`caller-save`模式，因此被调用函数的参数、返回值、栈位置都需要由调用者维护、准备。因此
当你需要调用一个函数时，需要先将这些工作准备好，方能调用下一个函数，另外这些都需要进行内存对其，对其
的大小是`sizeof(uintptr)`。

我们将结合一些函数来进行说明：

#### 无局部变量的函数

```go
func xxx(a, b, c int) (e, f, g int) {
	e, f, g = a, b, c
	return
}
```

该函数有3个输入参数、3个返回值，假设我们使用x86\_64平台，因此一个int占用8byte。则其函数栈空间为：
```
高地址位
                ┼───────────┼
                │  返回值g  │
                ┼───────────┼
                │  返回值f  │
                ┼───────────┼
                │  返回值e  │
                ┼───────────┼
                │   参数c   │
                ┼───────────┼
                │   参数b   │
                ┼───────────┼
                │   参数a   │     <-- FP
                ┼───────────┼
                │    PC     │     <-- SP
                ┼───────────┼
低地址位
```
各个输入参数和返回值将以倒序的方式从高地址位分布于栈空间上，由于没有局部变量，则xxx的函数栈空间为
0，根据前面的描述，该函数应该为：
```
#include "textflag.h"

TEXT ·xxx(SB),NOSPLIT,$0-48
   MOVQ a+0(FP), AX           // FP+0  为参数a，将其值拷贝到寄存器AX中
   MOVQ AX, e+24(FP)          // FP+24 为返回值e，将寄存器AX赋值给返回值e
   MOVQ b+8(FP), AX           // FP+0  为参数b
   MOVQ AX, f+32(FP)          // FP+24 为返回值f
   MOVQ c+16(FP), AX          // FP+0  为参数c
   MOVQ AX, g+40(FP)          // FP+24 为返回值g
   RET                        // return
```

然后在一个go源文件(.go)中声明该函数即可
```go
func xxx(a, b, c int) (e, f, g int)
```

#### 有局部变量的函数

当函数中有局部变量时，函数的栈空间就应该留出足够的空间：
```go
func zzz(a, b, c int) [3]int{
	var d [3]int
	d[0], d[1], d[2] = a, b, c
	return d
}
```

当函数中有局部变量时，我们就需要移动函数栈帧来进行栈内存分配，因此我们就需要了解相关平台计算机体系
的一些设计问题，在此我们只讲解x86平台的相关要求，我们先需要参考：

1. [Where the top of the stack is on x86](https://eli.thegreenplace.net/2011/02/04/where-the-top-of-the-stack-is-on-x86/)
2. [Stack frame layout on x86-64](https://eli.thegreenplace.net/2011/09/06/stack-frame-layout-on-x86-64)
3. [x86 Assembly Guide](http://www.cs.virginia.edu/~evans/cs216/guides/x86.html)

其中讲到x86平台上`BP`寄存器，通常用来指示函数栈的起始位置，仅仅其一个指示作用，现代编译器生成的代码
通常不会用到`BP`寄存器，但是可能某些debug工具会用到该寄存器来寻找函数参数、局部变量等。因此我们写汇编
代码时，也最好将栈起始位置存储在`BP`寄存器中。

因此在函数被调用之后，输入参数、返回值、`FP`和`SP`的布局仍然跟上面的程序一样，然后我们需要手动分配栈空间
和存储`BP`寄存器，因此需要在栈空间分配的内存为`SIZE_OF(局部变量)+8`，即`3*8+8=32`。

此处需要注意，go编译器会将函数栈空间自动加8，用于存储BP寄存器，跳过这8字节后才是函数栈上局部变量的内存。
逻辑上的FP/SP位置就是我们在写汇编代码时，计算偏移量时，FP/SP的基准位置，因此局部变量的内存在逻辑SP的低地
址侧，因此我们访问时，需要向负方向偏移。

实际上，在该函数被调用后，编译器会在函数进入时和RET是添加上扩容栈空间、缩容栈空间的代码，SP指向的位置在
函数栈空间顶部(顶部的意思是地址最低端)，并不是代码中逻辑上的SP位置，这个偏移量差距，在编译器会在编译时自
动加在所有的FP/SP的相关运算上，不需要我们处理。

当我们使用反编译命令反编译go二进制程序时得到的汇编代码是寄存器添加相关附加代码后的最终版本，并不是我们
所写的汇编代码版本，其中的寄存器偏移量都是计算栈空间偏移后的最终值，与我们缩写的汇编代码中的偏移量不同，
因此我们需要特别注意这一点。

```
高地址位
          ┼───────────┼
          │  返回值g  │
          ┼───────────┼
          │  返回值f  │
          ┼───────────┼
          │  返回值e  │
          ┼───────────┼
          │   参数c   │
          ┼───────────┼
          │   参数b   │
          ┼───────────┼
          │   参数a   │    <-- 逻辑上FP位置
          ┼───────────┼
          │    PC     │    <-- 逻辑上SP位置
          ┼───────────┼
          │ Saved BP  │
          ┼───────────┼
          │ 局部变量3 │
          ┼───────────┼
          │ 局部变量2 │
          ┼───────────┼
          │ 局部变量1 │     <-- 实际上SP位置
          ┼───────────┼
低地址位
```

我们实现该函数的汇编代码：

```
#include "textflag.h"

TEXT ·zzz(SB),NOSPLIT,$24-48    // $24值栈空间24byte，- 后面的48跟上面的含义一样，
                                // 在编译后，栈空间会被+8用于存储BP寄存器，这步骤由编译器自动添加
   MOVQ    $0, d-24(SP)         // 初始化d[0]
   MOVQ    $0, d-16(SP)         // 初始化d[1]
   MOVQ    $0, d-8(SP)          // 初始化d[2]
   MOVQ    a+0(FP), AX          // d[0] = a
   MOVQ    AX, d-24(SP)         //
   MOVQ    b+8(FP), AX          // d[1] = b
   MOVQ    AX, d-16(SP)         //
   MOVQ    c+16(FP), AX         // d[2] = c
   MOVQ    AX, d-8(SP)          //
   MOVQ    d-24(SP), AX         // d[0] = return [0]
   MOVQ    AX, r+24(FP)         //
   MOVQ    d-16(SP), AX         // d[1] = return [1]
   MOVQ    AX, r+32(FP)         //
   MOVQ    d-8(SP), AX          // d[2] = return [2]
   MOVQ    AX, r+40(FP)         //
   RET                          // return
```

然后我们go源码中声明该函数：
```go
func zzz(a, b, c int) [3]int
```

#### 汇编中调用其他函数
在汇编中调用其他函数通常可以使用2中方式：

* `JMP` 含义为跳转，直接跳转时，与函数栈空间相关的几个寄存器`SP`/`FP`不会发生变化，可以理解为被调用函数
复用调用者的栈空间，此时，参数传递采用寄存器传递，调用者和被调用者协商好使用那些寄存传递参数，调用者将
参数写入这些寄存器，然后跳转到被调用者，被调用者从相关寄存器读出参数。具体实践可以参考[1](https://github.com/golang/go/blob/d1be0fd910758852584ab53d2c92c4caac3f5b7e/src/runtime/asm_amd64.s#L1516-L1522)。
* `CALL` 通过`CALL`命令来调用其他函数时，栈空间会发生响应的变化，传递参数时，我们需要输入参数、返回值按
之前将的栈布局安排在调用者的栈顶(低地址段)，然后再调用`CALL`命令来调用其函数，调用`CALL`命令后，`SP`寄存
器会下移一个`WORD`(x86\_64上是8byte)，然后进入新函数的栈空间运行。

下面演示一个`CALL`方法调用的例子：
```go
func yyy(a, b, c int) [3]int {
    return zzz(a, b, c)
}
```

在这个例子中我们计划做一些优化，其中中间的临时变量，让`zzz`的输入参数、返回值复用`yyy`的输入参数、返回值
这部分空间，其代码为：

```asm
TEXT ·yyy(SB),NOSPLIT,$0-48
   MOVQ pc+0(SP),          AX            // 将PC寄存器中的值暂时保存在最后一个返回值的位置，因为在
                                         // 调用zzz时，该位置不会参与计算
   MOVQ AX,                ret_2+40(FP)  //
   MOVQ a+0(FP),           AX            // 将输入参数a，放置在栈顶
   MOVQ AX,                z_a+0(SP)     //
   MOVQ b+8(FP),           AX            // 将输入参数b，放置在栈顶+8
   MOVQ AX,                z_b+8(SP)     //
   MOVQ c+16(FP),          AX            // 将输入参数c，放置在栈顶+16
   MOVQ AX,                z_c+16(SP)    //
   CALL ·zzz(SB)                         // 调用函数zzz
   MOVQ ret_2+40(FP),      AX            // 将PC寄存器恢复
   MOVQ AX,                pc+0(SP)      //
   MOVQ z_ret_2+40(SP),    AX            // 将zzz的返回值[2]防止在yyy返回值[2]的位置
   MOVQ AX,                ret_2+40(FP)  //
   MOVQ z_ret_1+32(SP),    AX            // 将zzz的返回值[1]防止在yyy返回值[1]的位置
   MOVQ AX,                ret_1+32(FP)  //
   MOVQ z_ret_0+24(SP),    AX            // 将zzz的返回值[0]防止在yyy返回值[0]的位置
   MOVQ AX,                ret_0+24(FP)  //
   RET                                   // return
```

整个函数调用过程为：
```
高地址位
            ┼───────────┼           ┼────────────┼          ┼────────────┼
            │ 返回值[2] │           │     PC     │          │     PC     │
            ┼───────────┼           ┼────────────┼          ┼────────────┼
            │ 返回值[1] │           │zzz返回值[2]│          │zzz返回值[2]│
            ┼───────────┼           ┼────────────┼          ┼────────────┼
            │ 返回值[0] │           │zzz返回值[1]│          │zzz返回值[1]│
            ┼───────────┼  =调整=>  ┼────────────┼  =调用=> ┼────────────┼
            │   参数c   │           │zzz返回值[0]│          │zzz返回值[0]│
            ┼───────────┼           ┼────────────┼          ┼────────────┼
            │   参数b   │           │    参数c   │          │    参数c   │
            ┼───────────┼           ┼────────────┼          ┼────────────┼
            │   参数a   │  <-- FP   │    参数b   │  <-- FP  │    参数b   │
            ┼───────────┼           ┼────────────┼  <-- SP  ┼────────────┼
            │    PC     │  <-- SP   │    参数a   │          │    参数a   │  <--FP
            ┼───────────┼           ┼────────────┼          ┼────────────┼
                                                            │     PC     │  <--SP  zzz函数栈空间
                                                            ┼────────────┼
                                                            │   Saved BP │
                                                            ┼────────────┼
                                                            │zzz局部变量2│
                                                            ┼────────────┼
                                                            │zzz局部变量1│
                                                            ┼────────────┼
                                                            │zzz局部变量0│
                                                            ┼────────────┼
低地址位
```

然后我们可以使用反汇编来对比我们自己实现的汇编代码版本和go源码版本生成的汇编代码的区别：

我们自己汇编的版本：
```asm
TEXT main.yyy(SB) go/asm/xx.s
  xx.s:31               0x104f6b0               488b0424                MOVQ 0(SP), AX
  xx.s:32               0x104f6b4               4889442430              MOVQ AX, 0x30(SP)
  xx.s:33               0x104f6b9               488b442408              MOVQ 0x8(SP), AX
  xx.s:34               0x104f6be               48890424                MOVQ AX, 0(SP)
  xx.s:35               0x104f6c2               488b442410              MOVQ 0x10(SP), AX
  xx.s:36               0x104f6c7               4889442408              MOVQ AX, 0x8(SP)
  xx.s:37               0x104f6cc               488b442418              MOVQ 0x18(SP), AX
  xx.s:38               0x104f6d1               4889442410              MOVQ AX, 0x10(SP)
  xx.s:39               0x104f6d6               e865ffffff              CALL main.zzz(SB)
  xx.s:40               0x104f6db               488b442430              MOVQ 0x30(SP), AX
  xx.s:41               0x104f6e0               48890424                MOVQ AX, 0(SP)
  xx.s:42               0x104f6e4               488b442428              MOVQ 0x28(SP), AX
  xx.s:43               0x104f6e9               4889442430              MOVQ AX, 0x30(SP)
  xx.s:44               0x104f6ee               488b442420              MOVQ 0x20(SP), AX
  xx.s:45               0x104f6f3               4889442428              MOVQ AX, 0x28(SP)
  xx.s:46               0x104f6f8               488b442418              MOVQ 0x18(SP), AX
  xx.s:47               0x104f6fd               4889442420              MOVQ AX, 0x20(SP)
  xx.s:48               0x104f702               c3                      RET
```

go源码版本生成的汇编：
```asm
TEXT main.yyy(SB) go/asm/main.go
  main.go:20            0x104f360               4883ec50                        SUBQ $0x50, SP
  main.go:20            0x104f364               48896c2448                      MOVQ BP, 0x48(SP)
  main.go:20            0x104f369               488d6c2448                      LEAQ 0x48(SP), BP
  main.go:20            0x104f36e               48c744247000000000              MOVQ $0x0, 0x70(SP)
  main.go:20            0x104f377               48c744247800000000              MOVQ $0x0, 0x78(SP)
  main.go:20            0x104f380               48c784248000000000000000        MOVQ $0x0, 0x80(SP)
  main.go:20            0x104f38c               488b442458                      MOVQ 0x58(SP), AX
  main.go:21            0x104f391               48890424                        MOVQ AX, 0(SP)
  main.go:20            0x104f395               488b442460                      MOVQ 0x60(SP), AX
  main.go:21            0x104f39a               4889442408                      MOVQ AX, 0x8(SP)
  main.go:20            0x104f39f               488b442468                      MOVQ 0x68(SP), AX
  main.go:21            0x104f3a4               4889442410                      MOVQ AX, 0x10(SP)
  main.go:21            0x104f3a9               e892020000                      CALL main.zzz(SB)
  main.go:21            0x104f3ae               488b442418                      MOVQ 0x18(SP), AX
  main.go:21            0x104f3b3               4889442430                      MOVQ AX, 0x30(SP)
  main.go:21            0x104f3b8               0f10442420                      MOVUPS 0x20(SP), X0
  main.go:21            0x104f3bd               0f11442438                      MOVUPS X0, 0x38(SP)
  main.go:22            0x104f3c2               488b442430                      MOVQ 0x30(SP), AX
  main.go:22            0x104f3c7               4889442470                      MOVQ AX, 0x70(SP)
  main.go:22            0x104f3cc               0f10442438                      MOVUPS 0x38(SP), X0
  main.go:22            0x104f3d1               0f11442478                      MOVUPS X0, 0x78(SP)
  main.go:22            0x104f3d6               488b6c2448                      MOVQ 0x48(SP), BP
  main.go:22            0x104f3db               4883c450                        ADDQ $0x50, SP
  main.go:22            0x104f3df               c3                              RET
```

经过对比可以看出我们的优点:
* 没有额外分配栈空间
* 没有中间变量，减少了拷贝次数
* 没有中间变量的初始化，节省操作

go源码版本的优点：
* 对于连续内存使用了`MOVUPS`命令优化，（此处不一定是优化，有时还会劣化，因为X86\_64不同
指令集混用时，会产生[额外开销](://software.intel.com/en-us/articles/intel-avx-state-transitions-migrating-sse-code-to-avx)）

我们可以运行一下`go benchmark`来比较一下两个版本，可以看出自己的汇编版本速度上明显快于go源码版本。
```
go test -bench=. -v -count=3
goos: darwin
goarch: amd64
BenchmarkYyyGoVersion-4        100000000            16.9 ns/op
BenchmarkYyyGoVersion-4        100000000            17.0 ns/op
BenchmarkYyyGoVersion-4        100000000            17.1 ns/op
BenchmarkYyyAsmVersion-4       200000000            10.1 ns/op
BenchmarkYyyAsmVersion-4       200000000             7.90 ns/op
BenchmarkYyyAsmVersion-4       200000000             8.01 ns/op
PASS
ok      go/asm    13.005s
```

### 编译/反编译
因为go汇编的资料很少，所以我们需要通过编译、反汇编来学习。

```shell
// 编译
go build -gcflags="-S"
go tool compile -S hello.go
go tool compile -N -S hello.go // 禁止优化
// 反编译
go tool objdump <binary>
```

## 总结
了解go汇编并不是一定要去手写它，因为汇编总是不可移植和难懂的。但是它能够帮助我们了解go的一些底层机制，
了解计算机结构体系，同时我们需要做一些hack的事时可以用得到。

比如，我们可以使用`go:noescape`来减少内存的分配：

很多时候，我们可以使函数内计算过程使用栈上的空间做缓存，这样可以减少对内存的使用，并且是计算速度更快：
```
func xxx() int{
	var buf [1024]byte
	data := buf[:]
	// do something in data
}
```
但是，很多时候，go编译器的逃逸分析并不让人满意，经常会使`buf`移动到堆内存上，造成不必要的内存分配。
这是我们可以使用`sync.Pool`，但是总让人不爽。因此我们使用汇编完成一个`noescape`函数，绕过go编译器的
逃逸检测，使`buf`不会移动到堆内存上。

```asm
// asm_amd64.s
#include "textflag.h"

TEXT ·noescape(SB),NOSPLIT,$0-48
        MOVQ    d_base+0(FP),   AX
        MOVQ    AX,     b_base+24(FP)
        MOVQ    d_len+8(FP),    AX
        MOVQ    AX,     b_len+32(FP)
        MOVQ    d_cap+16(FP),AX
        MOVQ    AX,     b_cap+40(FP)
        RET
```

```go
//此处使用go编译器的指示
//go:noescape
func noescape(d []byte) (b []byte)

func xxx() int {
	var buf [1024]byte
	data := noescape(buf[:])
	// do something in data
	// 这样可以确保buf一定分配在xxx的函数栈上
}
```

## 参考
* [A Quick Guide to Go's Assembler](https://golang.org/doc/asm)
* [解析 Go 中的函数调用](https://juejin.im/post/58f579b58d6d81006491c7c0/)
* [A Manual for the Plan 9 assembler](http://doc.cat-v.org/plan_9/4th_edition/papers/asm)
* [Golang中的Plan9汇编器](https://github.com/yangyuqian/technical-articles/blob/master/asm/golang-plan9-assembly-cn.md)

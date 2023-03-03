---
layout: wiki
title: linux stack canary 机制
categories: linux
description: linux 安全机制
keywords: linux stack canary
---

# [canary分析](https://hardenedlinux.github.io/2016/11/27/canary.html)

## 导引
由于stack overflow而引发的攻击非常普遍也非常古老, 相应地一种叫做canary的 mitigation技术很早就出现在gcc/glibc里, 直到现在也作为系统安全的第一道防线存在.

canary不管是实现还是设计思想都比较简单高效, 就是插入一个值, 在stack overflow发生的 高危区域的尾部, 当函数返回之时检测canary的值是否经过了改变, 以此来判断stack/buffer overflow是否发生.

canary的实现大约是1998年就在gcc里出现出现了第一个合并入upstream的patch, 只不过后 期的实现将主要的功能移入了Glibc, 并且调整了局部数据和canary的位置, 以扩大数据的受 保护范围. 因为canary的出现的很早, 所以本文涉及到的GNU toolchains源代码版本来说选择 非常自由, 我使用的是我熟悉的版本: GCC-4.8.2 & eglibc-2.19 & linux-4.8.

## 示例
在gcc的调用参数里有以下跟canary有关系:

- `-fstack-protector` 对包含有malloc族系和内部的buffer大于8字节的函数使能canary.
- `-fstack-protector-all` 对所有函数使能canary.
- `-fstack-protector-strong` 对包含有malloc族系或者内部的buffer大于8字节的或者包含局部数组的或者包含对local frame地址引用的函数使能canary.
- `-fstack-protector-explicit` 只对有明确stack_protect attribute的函数使能canary.
- `-fno-stack-protector` 禁用canary.

下面首先给出一个例子, 展示一下canary在应用程序级别对代码的直接影响.

```c
#include <stdio.h>

int main () {
        int local = 1;
        char buffer[8];
        buffer[7] = 2;
        puts("42\n");

        return 0;
}
```

```asm
$gcc -m32 -fstack-protector-all -g -o canary canary.c

/// gdb查看运行时的反汇编代码
///
   // classical prologue.
   |0x804846d <main at canary.c:3>          push   %ebp
   |0x804846e <main+1 at canary.c:3>        mov    %esp,%ebp
   |0x8048470 <main+3 at canary.c:3>        and    $0xfffffff0,%esp
   |0x8048473 <main+6 at canary.c:3>        sub    $0x20,%esp
   // %gs:0x14里面存储的就是canary值, 并将其插入在地址esp+1c处.
   |0x8048476 <main+9 at canary.c:3>        mov    %gs:0x14,%eax
   |0x804847c <main+15 at canary.c:3>       mov    %eax,0x1c(%esp)
   |0x8048480 <main+19 at canary.c:3>       xor    %eax,%eax
   // 下面的两行代码可以看出gcc对buffer和local的位置进行了重排, 按照一般情况来说
   // 局部变量的声明先出现的先分配, 也就是地址更大, 也就是说应该是被赋值为1的局部
   // 变量local的地址更大. 但是这里进行了调整.
   |0x8048482 <main+21 at canary.c:5>       movl   $0x1,0x10(%esp)
   |0x804848a <main+29 at canary.c:6>       movb   $0x2,0x1b(%esp)
   // 入参准备puts()的调用
   |0x804848f <main+34 at canary.c:7>       movl   $0x8048550,(%esp)
   |0x8048496 <main+41 at canary.c:7>       call   0x8048340 <puts@plt>
   |0x804849b <main+46 at canary.c:9>       mov    $0x0,%eax
   // 取出插入的canary值与(%gs:0x14)处的原值做比较.
   |0x80484a0 <main+51 at canary.c:10>      mov    0x1c(%esp),%edx
   |0x80484a4 <main+55 at canary.c:10>      xor    %gs:0x14,%edx
   |0x80484ab <main+62 at canary.c:10>      je     0x80484b2 <main+69 at canary.c:10>
   // 如果canary的值发生了篡改, 那么将调用__stack_chk_fail.
   |0x80484ad <main+64 at canary.c:10>      call   0x8048330 <__stack_chk_fail@plt>
   |0x80484b2 <main+69 at canary.c:10>      leave
   |0x80484b3 <main+70 at canary.c:10>      ret

```

根据上面的代码可以画出如下的stack结构.

```
        High
        Address |                 |
                +-----------------+
                | arg1            |
                +-----------------+
                | return address  |
                +-----------------+
        ebp =>  | old ebp         |
                +-----------------+
     esp+1c =>  | canary value    |
                +-----------------+
     esp+1b =>  | char[7]         |
                | ...             |
     esp+14 =>  | char[0]         |
                +-----------------+
     esp+10 =>  | local           |
                +-----------------+
        Low     |                 |
        Address
```

上面的分析可以清楚地看出canary的原理, 下面将分析canary在gcc和glibc里的实现细节.

## 细节
canary的实现分为两部分, gcc编译时选择canary的插入位置, 以及生成含有canary的汇编代 码, glibc产生实际的canary值, 以及提供错误捕捉函数和报错函数. 也就是gcc使用glibc提供 的组件, gcc本身并不定义. 这样会让canary的值会是一个运行时才动态知道的值, 而不能通过 查看静态的bianry得到.

### gcc

在gcc里跟canary的实现分为两部分: 引入canary值的比较的代码, 这是通过对外部变量 (__stack_chk_guard)引用来实现, 以及插入canary比较出错时输出异常的代码, 这是通过对 外部定义的异常函数(__stack_chk_fail)的调用来实现的.

也就是说gcc只是声明和使用了__stack_chk_guard/__stack_chk_fail, 并没有定义. 定义是 在glibc里.

__stack_chk_guard和__stack_chk_fail的插入是在gcc将GIMPLE转换为RTL的pass里分别通 过函数default_stack_protect_guard()和ix86_stack_protect_fail()构建手动的tree, 然 后调用expand_normal()自动转换为RTL再插入待分析的用户代码来进行的.

在gcc中这部分的实现代码的调用栈如下:

```
// 注意由下至上的调用序.
#0  ix86_stack_protect_fail () at ../../gcc/config/i386/i386.c:37607
#0  default_stack_protect_guard () at ../../gcc/targhooks.c:635
#1  stack_protect_prologue () at ../../gcc/function.c:4646
#2  gimple_expand_cfg () at ../../gcc/cfgexpand.c:4641
#3  execute_one_pass () at ../../gcc/passes.c:2333
#4  execute_pass_list () at ../../gcc/passes.c:2381
#5  expand_function () at ../../gcc/cgraphunit.c:1640
#6  output_in_order () at ../../gcc/cgraphunit.c:1833
#7  compile () at ../../gcc/cgraphunit.c:2037
#8  finalize_compilation_unit () at ../../gcc/cgraphunit.c:2119
#9  c_write_global_declarations () at ../../gcc/c/c-decl.c:10118
#10 compile_file () at ../../gcc/toplev.c:557
#11 do_compile () at ../../gcc/toplev.c:1864
#12 toplev_main () at ../../gcc/toplev.c:1940
#13 main () at ../../gcc/main.c:36
```

gcc对__stack_chk_guard的使用其实还涉及到跟linux kernel的相关的部分.

```
gcc-4.8.2/gcc/config/i386/gnu-user.h
```

```
#line 153
// 也就是其实__stack_chk_guard的值是由kernel提供的一个随机数.
// 其实这也是最基本的情况, glibc还实现了不由kernel提供的另外的代码.
// gs寄存器是一个跟TLS(Thread Local Storage)有关系的寄存器, 一般情况下TLS肯定开启使用的,
// 所以最普遍的情况就是__stack_chk_guard由kernel提供.
#ifdef TARGET_LIBC_PROVIDES_SSP
/* i386 glibc provides __stack_chk_guard in %gs:0x14.  */
#define TARGET_THREAD_SSP_OFFSET    0x14
```

gcc使用了__stack_chk_guard和__stack_chk_fail之后就完成了canary实现的协议的编译器 的那部分, 接着查看glibc的细节.

### glibc

首先查看用户代码canary出错时的glibc错误输出代码:

```
eglibc-2.19/debug/stack_chk_fail.c
```

```
#line 24
void
__attribute__ ((noreturn))
__stack_chk_fail (void)
{
  __fortify_fail ("stack smashing detected");
}
```

上面的代码非常简单就是输出出错信息.

接着我们将分析glibc对于整个canary机制的实现过程的代码.

首先给出跟canary相关的调用栈:

```
#0  security_init () at rtld.c:854
// 相当于glibc/dynamic-linker的main
#1  dl_main () at rtld.c:1818
#2  _dl_sysdep_start () at ../elf/dl-sysdep.c:249
#3  _dl_start_final () at rtld.c:331
#4  _dl_start () at rtld.c:557
// glibc/dynamic-linker入口
#5  _start () from /lib/ld-linux.so.2

// security_init初始化canary的值到%gs:0x14, 所以这个函数是真正的关键所在. 我们在
// 下面的代码里会添加注释详细解释.
static void
security_init (void)
{
  /* Set up the stack checker's canary.  */
  // _dl_random的值其实在进入这个函数的时候就已经由kernel写入了. 也就是说glibc直
  // 接使用了_dl_random的值并没有给赋值, 进入下面的函数会看到其实如果不是采用TLS
  // 这种模式支持, glibc是可以自己产生随机数的. 但是做为普遍情况来说, _dl_random就
  // 是由kernel写入的. 所以_dl_setup_stack_chk_guard()的行为就是将_dl_random的最
  // 后一个字节设置为0x00.
  uintptr_t stack_chk_guard = _dl_setup_stack_chk_guard (_dl_random);
#ifdef THREAD_SET_STACK_GUARD
  // TLS会进入这里. macro的定义及其展开详见下面.
  THREAD_SET_STACK_GUARD (stack_chk_guard);
  // 删掉部分代码
  // ...
  /* We do not need the _dl_random value anymore.  The less
     information we leave behind, the better, so clear the
     variable.  */
  _dl_random = NULL;
}

// 这个宏只是一个过渡, header的定义见下面.
//
/* Set the stack guard field in TCB head.  */
#define THREAD_SET_STACK_GUARD(value) \
  THREAD_SETMEM (THREAD_SELF, header.stack_guard, value)

// header的定义.
// TLS相关的数据结构, 注意元素stack_guard的偏移是20也就是0x14.
//
typedef struct
{
  void *tcb;        /* Pointer to the TCB.  Not necessarily the
                       thread descriptor used by libpthread.  */
  dtv_t *dtv;
  void *self;       /* Pointer to the thread descriptor.  */
  int multiple_threads;
  uintptr_t sysinfo;
  uintptr_t stack_guard;
  uintptr_t pointer_guard;
  int gscope_flag;
#ifndef __ASSUME_PRIVATE_FUTEX
  int private_futex;
#else
  int __unused1;
#endif
  /* Reservation of some values for the TM ABI.  */
  void *__private_tm[4];
  /* GCC split stack support.  */
  void *__private_ss;
} tcbhead_t;

// 这个是会将canary的值写入%gs:0x14. 进行了删减之保留主要部分.
//
/* Same as THREAD_SETMEM, but the member offset can be non-constant.  */
# define THREAD_SETMEM(descr, member, value) \
  ({if (sizeof (descr->member) == 4)                              \
       asm volatile ("movl %0,%%gs:%P1" :                         \
                     : "ir" (value),                              \
                       "i" (offsetof (struct pthread, member)));  \
   })
```

由此就完成了glibc对canary的值的写入工作.

### linux kernel

linux初始化gs, 就是跟TLS相关的寄存器, TLS相关的部分i386比较复杂, 由于跟canary没有太 大关系, 具体其他细节可以参考下面的源文件的注释部分描述.

```
linux-4.8/arch/x86/include/asm/stackprotector.h
```

```
#line 99
static inline void load_stack_canary_segment(void)1
{
#ifdef CONFIG_X86_32
    asm("mov %0, %%gs" : : "r" (__KERNEL_STACK_CANARY) : "memory");
#endif
}
```

到这里我们就基本上解释清楚了canary的这条代码线.

# [Stack buffer overflow protection 學習筆記 – Stack canaries mechanism in User space](https://szlin.me/2017/12/09/stack-buffer-overflow-stack-canaries/)

緩衝區溢位 (Stack buffer overflow) 是傳統且常見的資訊安全攻擊手法. 主要透過程式漏洞造成輸入緩衝區溢位, 從而破壞原本程式執行、並取得系統的控制權。

而透過 buffer overflow 攻擊手法以及攻擊範例的相關資訊可瞭解如何使用 buffer overflow 來蓋掉 return address 內的資料以執行 shell code, 最終得到系統最高權限.

## Stack buffer overflow 防護機制
可透過下列幾種方法來避免或降低 stack buffer overflow 造成的系統安全危害

- KALSR & ALSR (Address space Layout Randomization)
- Executable Stack Protection
- Position Independent Executables
- Fortify Source
- _Stack Canaries_

## Stack Canaries

stack canaries 又稱為 stack protector, 此方法會在程式中加入 “canary value"。之所以會稱之為金絲雀 (canary) 是因為早期煤礦工人常會在工作時發生急性一氧化碳(煤氣)中毒事件。但以前沒有先進裝備來偵測一氧化碳(煤氣), 所以煤礦工人都會帶金絲雀 (canary) 進去礦坑工作。原因是金絲雀對一氧化碳(煤氣)非常敏感, 些許煤氣就會讓金絲雀啼叫甚至死亡。因此礦工會將金絲雀放在礦坑中作為是否需要逃生的依據。

同理可知, 若我們先在程式中加入 “canary value", 若 buffer overflow 異常情況發生, 則會修改 “canary value"。所以我們可透過檢查 “canary value" 是否遭修改來決定程式是否繼續往下執行或者中止結束。

![](/images/wiki/stack_canary_01.png)

### User space stack protector

在 user space 底下可使用 toolchain 來達到 stack protector 的目的. 以 gcc 7.1 為例, 共支援 4 種類型

- `-fstack-protector` 在使用到動態配置記憶體 (alloc) 或者 buffer > 8 bytes 的函數時加入以及檢查 canary 值
- `-fstack-protector-all` 對所有 function 都加入以及檢查 canary 值
- `-fstack-protector-strong` (gcc 版本需 >= 4.9) 除了 `-fstack-protector` 的條件外, 符合下列條件皆會加入以及檢查 canary 值
    1. 若 local 變數位址用來賦值或者當作函式參數
    2. local 變數為陣列類型, 或者 union 內含陣列
    3. 以 register 類型宣告的 local 變數
- `-fstack-protector-explicit` 只對以 __attribute__((stack_protect)) 宣告的 function 加入以及檢查 canary 值
    > e.g.  __attribute__((stack_protect)) test()
- `-fno-stack-protector` 不啟用 stack protector

### 運作原理剖析

以下列明顯有漏洞的程式為例

![](/images/wiki/stack_canary_02.png)

在 32-bit 作業系統環境下使用 gcc 7.1 的編譯結果如下:

![](/images/wiki/stack_canary_03.png)

note : 若在 64-bit 作業系統下則會差 8 bytes

由上表可知, 若啟用 stack protector 選項, 編譯器會做一些處理以至讓 binary 增加大小.

讓 binary 增加的部分, 勢必就是 Stack protector 施展魔法之處. 我們來看究竟在 binary 發生甚麼事

```sh
readelf -s hello-stack-protector
```

- 讀取 binary symbol table, 可以發現 binary 被 gcc 插入兩個 glibc 函式, 分別位於以及用來處理 overflow 時的行為

![](/images/wiki/stack_canary_04.png)

那甚麼時候會執行 __stack_chk_fail_locak 呢?  我們可以使用 objdump 工具來反組譯 binary 以獲得更多訊息

```sh
objdump -d hello-stack-protector
```

- 我們可以看到, 裡面將 %gs:0x14 的值 (canary value) 放進暫存器, 然後存放到 address (ebp –0xc) 的地方, 以進行防護

![](/images/wiki/stack_canary_05.png)

- #60c 執行 strcpy 後, 會將 address (ebp –0xc) 的 canary值取出, 並和 %gs:0x14 裡的值做比較
- 若比較結果一樣 (jump equal) 則跳到 #62a, 如果canary的值不一致, 代表偵測到 overflow, 執行 #625 的__stack_chk_fail.
- `%gs:0x14` 的值為 kernel space所賦予之亂數, 詳情可參考

![](/images/wiki/stack_canary_06.png)
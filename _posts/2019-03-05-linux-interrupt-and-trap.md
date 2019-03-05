---
layout: post
title: linux下的中断机制
categories: [linux]
description: linux interrupt 中断
keywords: linux interrupt 中断
---

# 中断的定义与种类

Linux中的中断异常机制[^1]：

> 中断（interrupt）被定义为一个事件，该事件改变处理器执行的指令顺序，这样的事件与CPU芯片内外部硬件电路产生的电信号相对应。**中断通常分为同步（synchronous）中断和异步（asynchronous）中断**。
>
> * 同步中断指的是当指令执行时由CPU控制单元产生的，之所以称为同步，是因为只有在一条指令终止执行后CPU才会发出中断。
> * 异步中断是由其他硬件设备依照CPU时钟信号随机产生的。
>
> 在Intel处理器中，**把同步中断和异步中断分别称为异常（exception）和中断（interrupt）**。我们这里也采用这种分类。我们也使用中断信号来统称中断和异常。
>
> 中断是由间隔定时器和I/O设备产生的，例如，用户的一次按键会引起一个中断，虽然用户没有感觉，但是按键这个过程到下一次按键之间的间隔对于计算机指令时间来说是非常长的。
>
> 另一方面，异常是由程序的错误产生的，或者由内核必须处理的异常条件产生的。第一种情况下，内核通过发送一个每个Unix程序员都熟悉的信号来处理异常，第二种情况下，内核执行恢复异常需要的所有步骤，例如缺页异常。
>
> 中断信号提供了一种特殊的方式，使处理器转而去运行正常的控制流之外的代码。当一个中断信号到达时，CPU必须停止它当前正在做的事情，保留上下文，并切换产生中断后的一个空间。
>
> 虽然进程切换和中断都会导致内核保存上下文并且切换到另一空间，但中断处理程序和进程切换有一个明显的差异，由中断或异常处理程序执行的代码不是一个进程，更确切的说，它是一个内核执行路径，代表中断发生时正在运行的进程执行。作为一个内核控制路径，中断处理程序要比一个进程更轻量。
>
> 中断处理是由内核执行的最敏感的任务之一，因此它必须满足下面的约束：
>
> 当内核正打算去完成一些别的事情时，中断会随时到来。因此，内核的目标就是让中断尽可能快的处理完，尽其所能把更多的处理向后推迟。例如一个数据块已经到达了网线，当硬件中断内核时，内核只简单的标志数据到来了，让处理器恢复到它以前的运行状态，其余的处理稍后再进行。因此，内核响应中断后需要进行的操作氛围两部分，关键而紧急的部分内核立即执行，其他的推迟的部分内核随后会执行。
>
> 因为中断随时到来，所以内核可能正在处理其中一个中断的时候，另一个中断又会到来，应该尽可能多的允许这样的情况发生，因为这能维持更多的I/O设备处于忙状态，提高I/O设备的吞吐量。因此中断处理程序必须便写成使相应的内核控制路径能以嵌套的方式执行。当最后一个内核控制路径终止时，内核必须能恢复被中断执行的进程。
>
> 尽管内核在处理前一个中断时可以接受一个新的中断，但在内核代码中还是存在一些临界区，在临界区中，中断必须被禁止。必须尽可能的限制这样的临界区，因为根据以前的要求，内核，尤其时中断处理程序，应该在大部分时间内以开中断的方式运行。
>
> Intel文档把中断和异常分为以下几类：
>
> 中断：
>
> * 可屏蔽中断，I/O设备发出的所有中断请求（IRQ）都产生可屏蔽中断，一个屏蔽的中断只要还是屏蔽的，控制单元就可以忽略它。
> * 非屏蔽中断，有一些危险的事件才能引起非屏蔽中断，例如硬件故障，非屏蔽中断总是由CPU辨认。
>
> 异常：
>
> 当CPU执行指令时探测到一个异常，会产生一个处理器探测异常（processor-detected exception），可以进一步区分，这取决于CPU控制单元产生异常时保存在内核堆栈eip寄存器的值。
>
> * 故障（fault），通常可以纠正，一旦纠正，程序就可以重新开始，保存在eip寄存器中的值是引起故障的指令地址。
> * 陷阱（trap）在陷阱指令执行后立即报告，内核把控制权烦给程序后就可以继续它的执行而不失连续性。保存在eip中的值是一个随后要执行的指令地址。陷阱的主要作用是为了调试程序。
> * 异常中止（abort），发生一个严重的错误，控制单元出了问题，不能在eip寄存器中保存引起异常的指令所在的确切位置。异常中止用于报告严重的错误，例如硬件故障或系统表中无效的值或者不一致的值。这种异常会强制中止进程。
> * 编程异常（programmed exception），在编程者发出的请求时发送，是由int或int3指令触发的。
>
> 每个中断和异常是由0～255之间的一个数来标识的，Intel把这个8位无符号整数叫做一个向量（vector）。非屏蔽中断的向量和异常的向量是固定的，而可屏蔽中断的向量是可以通过对中断控制器的编程来改变。

在X86中，分为实模式和保护模式，实模式通常是CPU启动到BIOS再到操作系统启动前的这段时间，操作启动初始化完成进入到保护模式。在不同模式下中断的处理机制同步，在这里只**简单探讨一下保护模式下的中断机制**。

# 中断的初始化

Intel X86位处理器有256个硬中断号，用一个8位的无符号整数表示，被叫做一个向量（vector）。

> 中断描述符表（Interrupt Descriptor Table，`IDT`）是一个系统表，它与每一个中断或异常向量相联系，每一个向量在表中有相应地中断或异常处理程序地入口地址，内核在允许中断发生前，必须适当地初始化IDT。

在`IDT`中可以存储以下三种`Gate Descriptor`（门描述符）：用于描述和控制`Interrupt Service Routine`（每个中断对应的回调函数）的访问，它们分别是：
* Interrupt Gate Descriptor（中断门描述符），包含段选择符和中断或异常处理程序地段内偏移量，当控制权转移到一个适当地段时，处理器清IF标志，从而关闭将来会发生的可屏蔽中断。
* Trap Gate Descriptor（陷井门描述符），和中断门类似，只是控制权传递到一个适当地段处理器不修改IF标志。
* Task Gate Descriptor（任务门描述符），Linux中未使用该类型，当中断信号发生时，必须取代当前进程地那个进程地TSS选择符存放在任务门中。

> `IDT`表中地每一项`Gate Descriptor`由8个字节组成，因此，最多需要256x8=2048字节来存放IDT（64位时扩展到16个字节）。因为需要增加权限控制等因素，在`IDT`中并不直接关联到`Interrupt Service Routine`。

`IDTR`是一个CPU寄存器，用以存储`IDT`的基地址与段大小，用以快速访问。机器指令`LLDT`和`SLDT`是分别用来设置和保存`LDTR`寄存器的值[^2]。

![](/images/posts/linux/062111_1451_x86x8664CPU11_IDTR.png)

在内核初始化的时候，会为每一个中断向量注册一个对应的回调函数（`Interrupt Service Routine`），初始化`IDT`与`IDTR`寄存器：
```c
// init/main.c
asmlinkage void __init start_kernel(void)
{
    ...
    setup_arch(&command_line); // 初始化架构相关的，其中包含一些与架构相关的中断，
                               // Page Fault相关的中断就在其内部注册
    ...
    trap_init();    // 初始化异常处理
    ...
    init_IRQ();     // 初始化外部中断
    ...
    init_timers();  // 初始化定时器模块，同时，会注册定时器的软中断处理函数
    ...
    softirq_init(); // 初始化软中断
}

// 在setup_arch中会调用early_trap_pf_init/early_trap_init，其中会注册
// Page Fault异常对应的中断函数page_fault
/* Set of traps needed for early debugging. */
void __init early_trap_init(void)
{
    set_intr_gate_ist(X86_TRAP_DB, &debug, DEBUG_STACK);
    /* int3 can be called from all */
    set_system_intr_gate_ist(X86_TRAP_BP, &int3, DEBUG_STACK);
#ifdef CONFIG_X86_32
    set_intr_gate(X86_TRAP_PF, &page_fault);
#endif
    load_idt(&idt_descr);
}

void __init early_trap_pf_init(void)
{
#ifdef CONFIG_X86_64
    set_intr_gate(X86_TRAP_PF, &page_fault);
#endif
}
```

更加详细的流程可以参考下面几张图，出自[^3]。

![](/images/posts/linux/interrupt-0.png)

![](/images/posts/linux/interrupt-1.jpg)

![](/images/posts/linux/interrupt-2.jpg)

![](/images/posts/linux/interrupt-3.jpg)

在初始化完成后，每个中断向量都会有一个对应的回调函数（`Interrupt Service Routine`），不同的中断向量对应不同的软件逻辑，如果系统不关心，则置为`ignore_int`：

| 中断向量号 | 异常事件 | Linux的函数 |
| -------- | ------ | ---------- |
| 0 | 除法错误 | divide_error |
| 1 | 调试异常 | debug |
| 2 | NMI中断 | nmi |
| 3	| 单字节，int 3 | int3 |
| 4 | 溢出 | overflow |
| 5 | 边界监测中断 | bounds |
| 6 | 无效操作码 | invalid_op |
| 7 | 设备不可用 | device_not_available |
| 8 | 双重故障 | double_fault |
| 9 | 协处理器段溢出 | coprocessor_segment_overrun |
| 10 | 无效TSS | invalid_TSS |
| 11 | 缺段中断 | segment_not_present |
| 12 | 堆栈异常 | stack_segment |
| 13 | 一般保护异常 | general_protection |
| 14 | 页异常 | page_fault |
| 15 | | spurious_interrupt_bug |
| 16 | 协处理器出错 | coprocessor_error |
| 17 | 对齐检查中断 | alignment_check |
| 0x80 | 系统调用 | ia32_syscall |
| 0xf9 | 内核调试 | call_debug |

上面表格中的这些对应函数，都只是由汇编实现的跳转函数，他们的实现在`arch/x86/kernel/entry_64.S`，基本上都是处理一下必要的上下文，然后跳转到相关的C实现的函数`do_xxx`中[^4]。

| 中断向量号 | 异常事件 | Linux汇编 | 调用c函数 | 处理结果 |
| 0 | 除法错误 | divide_error | do_divide_error | 发送SIGFPE信号 |
| 1	| 调试异常 | debug | do_debug | 发送SIGTRAP信号 |
| 2 | NMI中断 | nmi | do_nmi | |
| 3 | 单字节，int 3 | int3 | do_int3 | 发送SIGTRAP信号 |
| 4 | 溢出 | overflow | do_overflow | 发送SIGSEGV信号 |
| 5 | 边界监测中断 | bounds | do_bounds | 发送SIGSEGV信号 |
| 6	| 无效操作码 | invalid_op | do_invalid_op | 发送SIGILL信号 |
| 7 | 设备不可用 | device_not_available | do_device_not_available | 发送SIGSEGV信号 |
| 8	| 双重故障 | double_fault | do_double_fault |
| 9	| 协处理器段溢出 | coprocessor_segment_overrun | do_coprocessor_segment_overrun | 发送SIGFPE信号 |
| 10 | 无效TSS | invalid_TSS | do_invalid_TSS | 发送SIGSEGV信号 |
| 11 | 缺段中断 | segment_not_present | do_segment_not_present | 发送SIGBUS信号 |
| 12 | 堆栈异常 | stack_segment | do_stack_segment |
| 13 | 一般保护异常 | general_protection | do_general_protection |
| 14 | 页异常 | page_fault | do_page_fault | 处理缺页中断 |
| 15 | | spurious_interrupt_bug | do_spurious_interrupt_bug |
| 16 | 协处理器出错 | coprocessor_error | do_coprocessor_error | 发送SIGFPE信号 |
| 17 | 对齐检查中断 | alignment_check | do_alignment_check | 发送SIGBUS信号 |
| 0x80 | 系统调用 | ia32_syscall |
| 0xf9 | 内核调试 | call_debug | do_call_debug |

# 中断和异常的硬件处理[^5]

## 中断触发

> 假定内核已经被初始化，因此，CPU在保护模式下运行，Linux只有在刚刚启动的时候是在实模式，之后便进入保护模式。
>
> 在执行了一条指令之后，cs和eip这对寄存器包含下一条将要执行的指令的逻辑地址，在处理了那条指令之后，控制单元会检查在运行前一条指令时是否已经发生了一个中断异或异常。如果发生了一个中断或者异常，那么控制单元执行下列操作：
>
> 1. 确定与中断或异常的关联向量i。
> 2. 读由`IDTR`寄存器指向的`IDT`表中的第i项门描述符。
> 3. 从`GDTR`寄存器获得`GDT`的基地址，并在`GDT`中查找，以读取`IDT`表项中的选择符所标识的段描述符，这个描述符指定中断或异常处理程序所在的段的基地址。
> 4. 确定中断是由授权的中断发生源发出的。(Why？因为INT指令允许用户态的进程产生中断信号，其向量值可以为0到255的任一值，为了避免用户通过INT指令产生非法中断，在初始化的时候，将向量值为80H的门描述符（系统调用使用该门）的DPL设为3，将其他需要避免访问的门描述符的DPL值设为0，这样在做权限检查的时候就可以检查出来非法的情况。)
> 5. 检查是否发生了特权等级变化。如果是由用户态陷入了内核态，控制单元必须开始使用与新的特权级相关的堆栈
>   5.1. 读`TR`寄存器，访问运行进程的`TSS`段。（why？因为任何进程从用户态陷入内核态都必须从`TSS`获得内核堆栈指针。)
>   5.2. 用与新特权级相关的栈段和栈指针装载ss和esp寄存器。这些值可以在进程的`TSS`段中找到。
>   5.3. 在新的栈（内核栈）中保存用户态的ss和esp，这些值指明了用户态相关栈的逻辑地址。
> 6. 如果故障已经发生，用引起异常的指令地址装载cs和eip寄存器，从而使这条指令能够再次被执行。
> 7. 在栈中保存eflags、cs以及eip的内容。
> 8. 如果异常产生了一个硬件出错码，则保存在栈中。
> 9. 装载cs和eip寄存器，其值分别是`IDT`表中的第i项门描述符的段选择符和偏移量，这些值给出了中断或者异常处理程序的第一条指令的逻辑地址（开始执行中断回调函数）。

## 中断返回

> 中断或异常被处理完毕后，相应的处理程序必须产生一条`iret`指令，把控制权转交给被中断的进程，这样控制单元就会产生以下操作：
>
> 1. 用保存在栈中的值装载cs、eip或eflags寄存器，如果一个硬件出错码曾被押入栈中，并且在eip内容上面，那么执行iret指令必须先弹出这个硬件出错码。
> 2. 检查处理程序的CPL是否等于cs中的最低两位的值，如果是，说明在同一特权级，iret中止执行，否则转入下一步。
> 3. 从栈中装载ss和esp寄存器，返回到与旧特权级相关的栈。
> 4. 检查ds、es、fs以及gs段寄存器的内容，如果其中一个寄存器包含的选择符是个段描述符，并且其DPL值小于CPL，那么就清除相应的段寄存器。

# 参考

[^1]: [中断和异常](http://guojing.me/linux-kernel-architecture/posts/interrupt/)
[^2]: [x86/x86_64 CPU内存管理寄存器](http://ilinuxkernel.com/?p=610)
[^3]: [LINUX-内核-中断分析-中断向量表](https://blog.csdn.net/haolianglh/article/details/51946687)
[^4]: [Linux x86_64内核中断初始化](https://www.cnblogs.com/stonehat/p/8681639.html)
[^5]: [中断描述符表](http://guojing.me/linux-kernel-architecture/posts/interrupt-descriptor-table/)
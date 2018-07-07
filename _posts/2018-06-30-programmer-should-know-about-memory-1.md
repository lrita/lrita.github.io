---
layout: post
title: 每个程序员都应该了解的内存知识-1
categories: [linux, memory]
description: linux memory
keywords: linux memory
---

### 注
这个系列文章源于`What Every Programmer Should Know About Memory`[^1]，粗读下来觉得很不错，最好能留存下来。同时发现这个系列的文章已经有一部分被人翻译了。故在此转发留存一份，毕竟存在自己收留的才是最可靠的，经常发现很多不错的文章链接失效的情况。

本文转载自[^2]，翻译自[^3]。本人进行了轻微的修改，感觉更符合原义。

现在的CPU比25年前要精密得多了。在那个年代，CPU的频率与内存总线的频率基本在同一层面上。内存的访问速度仅比寄存器慢那么一点点。但是，这一局面在上世纪90年代被打破了。CPU的频率大大提升，但内存总线的频率与内存芯片的性能却没有得到成比例的提升。并不是因为造不出更快的内存，只是因为太贵了。内存如果要达到目前CPU那样的速度，那么它的造价恐怕要贵上好几个数量级。

如果有两个选项让你选择，一个是速度非常快、但容量很小的内存，一个是速度还算快、但容量很多的内存，如果你的工作集比较大，超过了前一种情况，那么人们总是会选择第二个选项。原因在于辅助存储介质(一般为磁盘)的速度。由于工作集超过主存，那么必须用辅存来保存交换出去的那部分数据，而辅存的速度往往要比主存慢上好几个数量级。

好在这问题也并不全然是非黑即白的选择。在配置大量DRAM的同时，我们还可以配置少量SRAM。将地址空间的某个部分划给SRAM，剩下的部分划给DRAM。一般来说，SRAM可以当作扩展的寄存器来使用。

上面的做法看起来似乎可以，但实际上并不可行。首先，将SRAM内存映射到进程的虚拟地址空间就是个非常复杂的工作，而且，在这种做法中，每个进程都需要管理这个SRAM区内存的分配。每个进程可能有大小完全不同的SRAM区，而组成程序的每个模块也需要索取属于自身的SRAM，更引入了额外的同步需求。简而言之，快速内存带来的好处完全被额外的管理开销给抵消了。

因此，SRAM是作为CPU自动使用和管理的一个资源，而不是由OS或者用户管理的。在这种模式下，SRAM用来复制保存（或者叫缓存）主内存中有可能即将被CPU使用的数据。这意味着，在较短时间内，CPU很有可能重复运行某一段代码，或者重复使用某部分数据。从代码上看，这意味着CPU执行了一个循环，所以相同的代码一次又一次地执行（空间局部性的绝佳例子）。数据访问也相对局限在一个小的区间内。即使程序使用的物理内存不是相连的，在短期内程序仍然很有可能使用同样的数据（时间局部性）。这个在代码上表现为，程序在一个循环体内调用了入口一个位于另外的物理地址的函数。这个函数可能与当前指令的物理位置相距甚远，但是调用的时间差不大。在数据上表现为，程序使用的内存是有限的（相当于工作集的大小）。但是实际上由于RAM的随机访问特性，程序使用的物理内存并不是连续的。正是由于空间局部性和时间局部性的存在，我们才提炼出今天的CPU缓存概念。

我们先用一个简单的计算来展示一下高速缓存的效率。假设，访问主存需要200个周期，而访问高速缓存需要15个周期。如果使用100个数据元素100次，那么在没有高速缓存的情况下，需要2000000个周期，而在有高速缓存、而且所有数据都已被缓存的情况下，只需要168500个周期。节约了91.5%的时间。

用作高速缓存的SRAM容量比主存小得多。以我的经验来说，高速缓存的大小一般是主存的千分之一左右(目前一般是4GB主存、4MB缓存)。这一点本身并不是什么问题。只是，计算机一般都会有比较大的主存，因此工作集的大小总是会大于缓存。特别是那些运行多进程的系统，它的工作集大小是所有进程加上内核的总和。

处理高速缓存大小的限制需要制定一套很好的策略来决定在给定的时间内什么数据应该被缓存。由于不是所有工作集的数据都是在完全相同的时间段内被使用，因此我们可以用一些技术手段将需要用到的数据临时替换那些当前并未使用的缓存数据。这种预取将会减少部分访问主存的成本，因为它与程序的执行是异步的。所有的这些技术将会使高速缓存在使用的时候看起来比实际更大。我们将在3.3节讨论这些问题。 我们将在第6章讨论如何让这些技术能很好地帮助程序员，让处理器更高效地工作。

# 3.1 高速缓存的位置

在深入介绍高速缓存的技术细节之前，有必要说明一下它在现代计算机系统中所处的位置。

![](/images/posts/memory/cpumemory.1.png)

图3.1: 最简单的高速缓存配置图

图3.1展示了最简单的高速缓存配置。早期的一些系统就是类似的架构。在这种架构中，CPU核心不再直连到主存。{在一些更早的系统中，高速缓存像CPU与主存一样连到系统总线上。那种做法更像是一种hack，而不是真正的解决方案。}数据的读取和存储都经过高速缓存。CPU核心与高速缓存之间是一条特殊的快速通道。在简化的表示法中，主存与高速缓存都连到系统总线上，这条总线同时还用于与其它组件通信。我们管这条总线叫“FSB”——就是现在称呼它的术语，参见第2.2节。在这一节里，我们将忽略北桥。

在过去的几十年，经验表明使用了冯诺伊曼结构的计算机，将用于代码和数据的高速缓存分开是存在巨大优势的。自1993年以来，Intel 并且一直坚持使用独立的代码和数据高速缓存。由于所需的代码和数据的内存区域是几乎相互独立的，这就是为什么独立缓存工作得更完美的原因。近年来，独立缓存的另一个优势慢慢显现出来：常见处理器解码 指令的步骤是缓慢的，尤其当管线为空的时候，往往会伴随着错误的预测或无法预测的分支的出现， 将高速缓存技术用于 指令 解码可以加快其执行速度。

在高速缓存出现后不久，系统变得更加复杂。高速缓存与主存之间的速度差异进一步拉大，直到加入了另一级缓存。新加入的这一级缓存比第一级缓存更大，但是更慢。由于加大一级缓存的做法从经济上考虑是行不通的，所以有了二级缓存，甚至现在的有些系统拥有三级缓存，如图3.2所示。随着单个CPU中核数的增加，未来甚至可能会出现更多层级的缓存。

![](/images/posts/memory/cpumemory.2.png)

图3.2: 三级缓存的处理器

图3.2展示了三级缓存，并介绍了本文将使用的一些术语。`L1d`是一级数据缓存，`L1i`是一级指令缓存，等等。请注意，这只是示意图，真正的数据流并不需要流经上级缓存。CPU的设计者们在设计高速缓存的接口时拥有很大的自由。而程序员是看不到这些设计的。

另外，我们有多核CPU，每个核心可以有多个“线程”。核心与线程的不同之处在于，核心拥有独立的硬件资源({早期的多核CPU甚至有独立的二级缓存。})。在不同时使用相同资源(比如，通往外界的连接)的情况下，核心可以完全独立地运行。而线程只是共享资源。Intel的线程只有独立的寄存器，而且还有限制——不是所有寄存器都独立，有些是共享的。综上，现代CPU的结构就像图3.3所示。

![](/images/posts/memory/cpumemory.3.png)

在上图中，有两个处理器，每个处理器有两个核心，每个核心有两个线程。线程们共享一级缓存。核心(以深灰色表示)有独立的一级缓存，同时共享二级缓存。处理器(淡灰色)之间不共享任何缓存。这些信息很重要，特别是在讨论多进程和多线程情况下缓存的影响时尤为重要。

# 3.2 高级的缓存操作

了解成本和节约使用缓存，我们必须结合在第二节中讲到的关于计算机体系结构和RAM技术，以及前一节讲到的缓存描述来探讨。

默认情况下，CPU核心所有的数据的读或写都存储在缓存中。当然，也有内存区域不能被缓存的，但是这种情况只发生在操作系统的实现者对数据考虑的前提下；对程序实现者来说，这是不可见的。这也说明，程序设计者可以故意绕过某些缓存，不过这将是第六节中讨论的内容了。

如果CPU需要访问某个字(word)，先检索缓存。很显然，缓存不可能容纳主存所有内容(否则还需要主存干嘛)。系统用字的内存地址来对缓存条目进行标记。如果需要读写某个地址的字，那么根据标签来检索缓存即可。这里用到的地址可以是虚拟地址，也可以是物理地址，取决于缓存的具体实现。

缓存的标签是需要额外空间的，用字(word)作为缓存的粒度显然毫无效率。比如，在x86机器上，32位字的标签可能需要32位，甚至更长。另一方面，由于空间局部性的存在，与当前地址相邻的地址有很大可能会被一起访问。再回忆下2.2.1节——内存模块在传输位于同一行上的多份数据时，由于不需要发送新 $$\overline{CAS}$$ 信号，甚至不需要发送 $$\overline{RAS}$$ 信号，因此可以实现很高的效率。基于以上的原因，缓存条目并不存储单个字(word)，而是存储若干连续字组成的“线”(cache line)。在早期的缓存中，线长是32字节，现在一般是64字节。对于64位宽的内存总线，每条线需要8次传输。而DDR对于这种传输模式的支持更为高效。

当处理器需要内存中的某块数据时，整条cache line被装入`L1d`。cache line的地址通过对内存地址进行掩码操作生成。对于64字节的cache line，地址的低6 bit为0，这些不使用的bit位（低6bit）作为线内偏移量。其它的位作为标签，并用于在缓存内定位。在实践中，我们将地址分为三个部分。32位地址的情况如下:

![](/images/posts/memory/cpumemory.12.png)

如果cache line长度为 $$2^O$$，那么地址的低`O`位用作线内偏移量，接着的`S`位选择“缓存集”。后面我们会说明使用缓存集的原因。现在只需要明白一共有 $$2^S$$ 个缓存集就够了。剩下的`32 - S - O` = `T`位组成缓存标签，它们用来区分在缓存集中别名相同的各条线{有相同`S`部分的cache line被称为有相同的别名}。`S`位用于定位缓存集，不需要存储，因为属于同一缓存集的所有线的`S`部分都是相同的。

当某条指令修改内存时，仍然要先装入cache line，因为任何指令都不可能同时修改整条线(只有一个例外——第6.1节中将会介绍的写合并(write-combine))。因此需要在写操作前先把cache line装载进来。如果cache line被写入，但还没有写回主存，那就是所谓的“脏了”。脏了的线一旦写回主存，脏标记即被清除。

为了加载新数据，基本上总是要先在缓存中清理出位置。`L1d`将内容逐出`L1d`，推入`L2`(线长相同)。当然，`L2`也需要清理位置。于是`L2`将内容推入`L3`，最后`L3`将它推入主存。这种逐出操作一级比一级昂贵。这里所说的是现代AMD和VIA处理器所采用的独占型缓存(exclusive cache)。而Intel采用的是包容型缓存(inclusive cache)，{并不完全正确，Intel有些缓存是独占型的，还有一些缓存具有独占型缓存的特点。}`L1d`的每条线同时存在于`L2`里。对这种缓存，逐出操作就很快了。如果有足够`L2`，对于相同内容存在不同地方造成内存浪费的缺点可以降到最低，而且在逐出时非常有利。而独占型缓存在装载新数据时只需要操作`L1d`，不需要碰`L2`，因此会比较快。

处理器体系结构中定义的作为存储器的模型只要还没有改变，那就允许多CPU按照自己的方式来管理高速缓存。这表示，例如，设计优良的处理器，利用很少或根本没有内存总线活动，并主动写回主内存脏高速缓存行。这种高速缓存架构在如x86和x86-64各种各样的处理器间存在。制造商之间，即使在同一制造商生产的产品中，证明了的内存模型抽象的力量。

在对称多处理器（SMP）架构的系统中，CPU的高速缓存不能独立的工作。在任何时候，所有的处理器都应该拥有相同的内存内容。保证这样的统一的内存视图被称为“缓存一致性”。如果在其自己的高速缓存和主内存间，处理器设计简单，它将不会看到在其他处理器上的脏高速缓存行的内容。从一个处理器直接访问另一个处理器的高速缓存这种模型设计代价将是非常昂贵的，它是一个相当大的瓶颈。相反，当另一个处理器要读取或写入到高速cache line上时，处理器会去检测。

如果CPU检测到一个写访问，而且该CPU的缓存中已经缓存了一个cache line的原始副本，那么这个cache line将被标记为无效的cache line。接下来在引用这个cache line之前，需要重新加载该cache line。需要注意的是读访问并不会导致cache line被标记为无效的。

更精确的缓存实现需要考虑到其他更多的可能性，比如第二个CPU在读或者写它的cache line时，发现该cache line在第一个CPU的缓存中被标记为脏数据了，此时我们就需要做进一步的处理。在这种情况下，主存储器已经失效，第二个CPU需要读取第一个CPU的cache line。通过测试，我们知道在这种情况下第一个CPU会将自己的cache line数据自动发送给第二个CPU。这种操作是绕过主存储器的，但是有时候存储控制器是可以直接将第一个CPU中的cache line数据存储到主存储器中。对第一个CPU的缓存的写访问会导致本地cache line的所有拷贝被标记为无效。

随着时间的推移，一大批缓存一致性协议已经开发。其中，最重要的是MESI,我们将在第3.3.4节进行介绍。以上结论可以概括为几个简单的规则: 

* 一个脏cache line不存在于任何其他处理器的缓存之中。
* 同一cache line中的干净拷贝可以驻留在任意多个其他缓存之中。

如果遵守这些规则，处理器甚至可以在多处理器系统中更加有效的使用它们的缓存。所有的处理器需要做的就是监控其他每一个写访问和比较本地缓存中的地址。在下一节中，我们将介绍更多细节方面的实现，尤其是存储开销方面的细节。

最后，我们至少应该关注高速缓存命中或未命中带来的消耗。下面是英特尔`Pentium M`的数据：

| To Where | Cycles |
| -- | -- |
| `Register` | <= 1 |
| `L1d` | ~3 |
| `L2` | ~14 |
| Main Memory | ~240 |

这是在CPU周期中的实际访问时间。有趣的是，对于L2高速缓存的访问时间很大一部分（甚至是大部分）是由线路的延迟引起的。这是一个限制，增加高速缓存的大小变得更糟。只有当减小时（例如，从60纳米的Merom到45纳米Penryn处理器），可以提高这些数据。

表格中的数字看起来很高，但是，幸运的是，整个成本不必须负担每次出现的缓存加载和缓存失效。某些部分的成本可以被隐藏。现在的处理器都使用不同长度的内部管道，在管道内指令被解码，并为准备执行。如果数据要传送到一个寄存器，那么部分的准备工作是从存储器（或高速缓存）加载数据。如果内存加载操作在管道中足够早的进行，它可以与其他操作并行发生，那么加载的全部发销可能会被隐藏。对`L1d`常常可能如此；某些有长管道的处理器的`L2`也可以。 

提早启动内存的读取有许多障碍。它可能只是简单的不具有足够资源供内存访问，或者地址从另一个指令获取，然后加载的最终地址才变得可用。在这种情况下，加载成本是不能隐藏的（完全的）。 

对于写操作，CPU并不需要等待数据被安全地放入内存。只要指令具有类似的效果，就没有什么东西可以阻止CPU走捷径了。它可以早早地执行下一条指令，甚至可以在影子寄存器(shadow register)的帮助下，更改这个写操作将要存储的数据。

![](/images/posts/memory/cpumemory.21.png)

图3.4: 随机写操作的访问时间

图3.4展示了缓存的效果。关于产生图中数据的程序，我们会在稍后讨论。这里大致说下，这个程序是连续随机地访问某块大小可配的内存区域。每个数据项的大小是固定的。数据项的多少取决于选择的工作集大小。Y轴表示处理每个元素平均需要多少个CPU周期，注意它是对数刻度。X轴也是同样，工作集的大小都以2的n次方表示。

图中有三个比较明显的不同阶段。很正常，这个处理器有`L1d`和`L2`，没有`L3`。根据经验可以推测出，`L1d`有$$2^{13}$$字节，而`L2`有$$2^{20}$$字节。因为，如果整个工作集都可以放入`L1d`，那么只需不到10个周期就可以完成操作。如果工作集超过`L1d`，处理器不得不从`L2`获取数据，于是时间飘升到28个周期左右。如果工作集更大，超过了`L2`，那么时间进一步暴涨到480个周期以上。这时候，许多操作将不得不从主存中获取数据。更糟糕的是，如果修改了数据，还需要将这些脏了的cache line写回内存。

看了这个图，大家应该会有足够的动力去检查代码、改进缓存的利用方式了吧？这里的性能改善可不只是微不足道的几个百分点，而是几个数量级呀。在第6节中，我们将介绍一些编写高效代码的技巧。而下一节将进一步深入缓存的设计。虽然精彩，但并不是必修课，大家可以选择性地跳过。

# 3.3 CPU缓存实现的细节

缓存的实现者们都要面对一个问题：主存中每一个单元都可能需被缓存。如果程序的工作集很大，就会有许多内存位置为了缓存而打架。前面我们曾经提过缓存与主存的容量比，1:1000也十分常见。

## 3.3.1 关联性

我们可以让缓存的每条线能存放任何内存地址的数据。这就是所谓的全关联缓存(fully associative cache)。对于这种缓存，处理器为了访问某条线，将不得不检索所有线的标签。而标签则包含了整个地址，而不仅仅只是线内偏移量(也就意味着，图3.2中的`S`为0)。

高速缓存有类似这样的实现，但是，看看在今天使用的`L2`的数目，表明这是不切实际的。给定4MB的高速缓存和64B的高速缓存段，高速缓存将有65,536个项。为了达到足够的性能，缓存逻辑必须能够在短短的几个时钟周期内，从所有这些项中，挑一个匹配给定的标签。实现这一点的工作将是巨大的。

![](/images/posts/memory/cpumemory.70.png)

Figure 3.5: 全关联高速缓存原理图

对于每个高速缓存行，比较器是需要比较大标签（注意，`S`是零）。每个连接旁边的字母表示位的宽度。如果没有给出，它是一个单比特线。每个比较器都要比较两个`T`位宽的值。然后，基于该结果，适当的高速缓存行的内容被选中，并使其可用。这需要合并多套`O`数据线，因为他们是缓存桶（译注：这里类似把`O`输出接入多选器，所以需要合并）。实现仅仅一个比较器，需要晶体管的数量就非常大，特别是因为它必须非常快。没有迭代比较器是可用的。节省比较器的数目的唯一途径是通过反复比较标签，以减少它们的数目。这是不适合的，出于同样的原因，迭代比较器不可用：它消耗的时间太长。

全关联高速缓存对 小缓存是实用的（例如，在某些Intel处理器的TLB缓存是全关联的），但这些缓存都很小，非常小的。我们正在谈论的最多几十项。 

对于`L1i`，`L1d`和更高级别的缓存，需要采用不同的方法。可以做的就是是限制搜索。最极端的限制是，每个标签映射到一个明确的缓存条目。计算很简单：给定的4MB/64B缓存有65536项，我们可以使用地址的bit6到bit21（16位）来直接寻址高速缓存的每一个项。地址的低6位作为高速缓存段的索引。 

![](/images/posts/memory/cpumemory.71.png)

图3.6: 直接映射缓存原理图

在图3.6中可以看出，这种直接映射的高速缓存，速度快，比较容易实现。它只是需要一个比较器，一个多路复用器（在这个图中有两个，标记和数据是分离的，但是对于设计这不是一个硬性要求），和一些逻辑来选择只是有效的高速缓存行的内容。由于速度的要求，比较器是复杂的，但是现在只需要一个，结果是可以花更多的精力让其变得快速。这种方法的复杂性在于在多路复用器。一个简单的多路转换器中的晶体管的数量增速是`O(log N)`的，其中`N`是高速cache line的数目。这是可以容忍的，但可能会很慢，在某种情况下，速度可提升，通过增加多路复用器晶体管数量，来并行化的一些工作和自身增速。晶体管的总数只是随着快速增长的高速缓存缓慢的增加，这使得这种解决方案非常有吸引力。但它有一个缺点：只有用于直接映射地址的相关的地址位均匀分布，程序才能很好工作。如果分布的不均匀，而且这是常态，一些缓存项频繁的使用，并因此多次被换出，而另一些则几乎不被使用或一直是空的。

![](/images/posts/memory/cpumemory.73.png)

图3.7: 组关联高速缓存原理图

可以通过使高速缓存的组关联来解决此问题。组关联结合高速缓存的全关联和直接映射高速缓存特点，在很大程度上避免这些设计的弱点。图3.7显示了一个组关联高速缓存的设计。标签和数据存储分成不同的组并可以通过地址选择。这类似直接映射高速缓存。但是，少量个数的元素可以在同一个高速缓存组缓存，而不是一个缓存组只有一个元素，用于在高速缓存中的每个设定值是相同的一组值的缓存。所有组的成员的标签可以并行比较，这类似全关联缓存的功能。

其结果是高速缓存不容易被不幸地或故意被分配到同一缓存组中，同时高速缓存的大小并不限于由比较器的数目，可以以并行的方式实现。如果高速缓存增长，只（在该图中）增加列的数目，而不增加行数。只有高速缓存之间的关联性增加，行数才会增加。今天，处理器的`L2`高速缓存或更高的高速缓存，使用的关联性高达16路。 L1高速缓存通常使用8路。

![](/images/posts/memory/cpumem-table-0.png)

Table 3.1: 高速缓存大小，关联行，段大小的影响

给定我们4MB/64B高速缓存，8路组关联，相关的缓存留给我们的有8192组，只用标签的13位，就可以寻址缓集。要确定哪些（如果有的话）的缓存组设置中的条目包含寻址的高速缓存行，8个标签都要进行比较。在很短的时间内做出来是可行的。通过一个实验，我们可以看到，这是有意义的。

表3.1显示一个程序在改变缓存大小，缓存段大小和关联集大小时的`L2`高速缓存的缓存失效数量（根据Linux内核相关的方面人的说法，GCC在这种情况下，是他们所有中最重要的指标）。在7.2节中，我们将介绍工具来模拟此测试要求的高速缓存。

万一这还不是很明显，所有这些值之间的关系是高速缓存的大小为：

```
cache line size × associativity × number of sets
```

地址被映射到高速缓存使用

$$
O = log _ 2 (cache\ line\ size) \\
S = log _ 2 (number\ of\ sets)
$$

在第3.2节中的图显示的方式

![](/images/posts/memory/cpumemory.33.png)

图3.8: 缓存段大小 vs 关联行 (CL=32)

图3.8表中的数据更易于理解。它显示一个固定的32个字节大小的cache line，对于一个给定的高速缓存大小，我们可以看出，关联行的增加，的确可以帮助明显减少高速缓存未命中的数量。对于8MB的缓存，从直接映射到2路组相联，可以减少近44％的高速缓存未命中。组相联高速缓存和直接映射缓存相比，该处理器可以把更多的工作集保持在缓存中。

在文献中，偶尔可以读到，引入关联行，和加倍高速缓存的大小具有相同的效果。在从4M缓存跃升到8MB缓存的极端的情况下，这是正确的。关联行再提高一倍那就肯定不正确啦。正如我们所看到的数据，后面的收益要小得多。我们不应该完全低估它的效果，虽然。在示例程序中的内存使用的峰值是5.6M。因此，具有8MB缓存不太可能有很多（两个以上）使用相同的高速缓存集。从较小的缓存的关联性的巨大收益可以看出，较大工作集可以节省更多。

在一般情况下，增加8以上的高速缓存之间的关联性似乎对只有一个单线程工作量影响不大。随着介绍一个使用共享`L2`的多核处理器，形势发生了变化。现在你基本上有两个程序命中相同的缓存，实际上导致高速缓存减半（对于四核处理器是1/4）。因此，可以预期，随着核的数目的增加，共享高速缓存的相关性也应增长。一旦这种方法不再可行（16 路组关联性已经很难）处理器设计者不得不开始使用共享的三级高速缓存和更高级别的，而`L2`高速缓存只被核的一个子集共享。

从图3.8中，我们还可以研究缓存大小对性能的影响。这一数据需要了解工作集的大小才能进行解读。很显然，与主存相同的缓存比小缓存能产生更好的结果，因此，缓存通常是越大越好。

上文已经说过，示例中最大的工作集为5.6M。它并没有给出最佳缓存大小值，但我们可以估算出来。问题主要在于内存的使用并不连续，因此，即使是16M的缓存，在处理5.6M的工作集时也会出现冲突(参见2路集合关联式16MB缓存vs直接映射式缓存的优点)。不管怎样，我们可以有把握地说，在同样5.6M的负载下，缓存从16MB升到32MB基本已没有多少提高的余地。但是，工作集是会变的。如果工作集不断增大，缓存也需要随之增大。在购买计算机时，如果需要选择缓存大小，一定要先衡量工作集的大小。原因可以参见图3.10。

![](/images/posts/memory/cpumemory.75.png)

图3.9: 测试的内存分布情况

我们执行两项测试。第一项测试是按顺序地访问所有元素。测试程序循着指针n进行访问，而所有元素是链接在一起的，从而使它们的被访问顺序与在内存中排布的顺序一致，如图3.9的下半部分所示，末尾的元素有一个指向首元素的引用。而第二项测试(见图3.9的上半部分)则是按随机顺序访问所有元素。在上述两个测试中，所有元素都构成一个单向循环链表。

## 3.3.2 Cache的性能测试

用于测试程序的数据可以模拟一个任意大小的工作集：包括读、写访问，随机、连续访问。在图3.4中我们可以看到，程序为工作集创建了一个与其大小和元素类型相同的数组：

```c
struct l {
    struct l *n;
    long int pad[NPAD];
};
```

n字段将所有节点随机得或者顺序的加入到环形链表中，用指针从当前节点进入到下一个节点。pad字段用来存储数据，其可以是任意大小。在一些测试程序中，pad字段是可以修改的, 在其他程序中，pad字段只可以进行读操作。

在性能测试中，我们谈到工作集大小的问题，工作集使用结构体`l`定义的元素表示的。$$2^N$$字节的工作集包含$$\frac{2^N}{sizeof(struct\ l)}$$个元素. 显然`sizeof(struct l)`的值取决于`NPAD`的大小。在32位系统上，`NPAD=7`意味着数组的每个元素的大小为32字节，在64位系统上，`NPAD=7`意味着数组的每个元素的大小为64字节。

### 单线程顺序访问

最简单的情况就是遍历链表中顺序存储的节点。无论是从前向后处理，还是从后向前，对于处理器来说没有什么区别。下面的测试中，我们需要得到处理链表中一个元素所需要的时间，以CPU时钟周期最为计时单元。图3.10显示了测试结构。除非有特殊说明, 所有的测试都是在`Pentium 4`64-bit平台上进行的，因此结构体`l`中`NPAD=0`，大小为8字节。

![](/images/posts/memory/cpumemory.22.png)

图 3.10: 顺序读访问`NPAD=0`

![](/images/posts/memory/cpumemory.23.png)

图 3.11: 顺序读多个字节

一开始的两个测试数据收到了噪音的污染。由于它们的工作负荷太小，无法过滤掉系统内其它进程对它们的影响。我们可以认为它们都是4个周期以内的。这样一来，整个图可以划分为比较明显的三个部分:

* 工作集小于$$2^{14}$$字节的
* 工作集从$$2^{15}$$字节到$$2^{20}$$字节的
* 工作集大于$$2^{21}$$字节的

这样的结果很容易解释——是因为处理器有16KB的`L1d`和1MB的`L2`。而在这三个部分之间，并没有非常锐利的边缘，这是因为系统的其它部分也在使用缓存，我们的测试程序并不能独占缓存的使用。尤其是L2，它是统一式的缓存，处理器的指令也会使用它(注: Intel使用的是包容式缓存)。

测试的实际耗时可能会出乎大家的意料。`L1d`的部分跟我们预想的差不多，在一台P4上耗时为4个周期左右。但`L2`的结果则出乎意料。大家可能觉得需要14个周期以上，但实际只用了9个周期。这要归功于处理器先进的处理逻辑，当它使用连续的内存区时，会**预取**下一条缓存线的数据。这样一来，当真正使用下一条线的时候，其实已经早已读完一半了，于是真正的等待耗时会比`L2`的访问时间少很多。

在工作集超过`L2`的大小之后，**预取**的效果更明显了。前面我们说过，主存的访问需要耗时200个周期以上。但在预取的帮助下，实际耗时保持在9个周期左右。200 vs 9，效果非常不错。

我们可以观察到**预取**的行为，至少可以间接地观察到。图3.11中有4条线，它们表示处理不同大小结构时的耗时情况。随着结构的变大，元素间的距离变大了。图中4条线对应的元素距离分别是0、56、120和248字节。

图中最下面的这一条线来自前一个图，但在这里更像是一条直线。其它三条线的耗时情况比较差。图中这些线也有比较明显的三个阶段，同时，在小工作集的情况下也有比较大的错误(请再次忽略这些错误)。在只使用`L1d`的阶段，这些线条基本重合。因为这时候还不需要**预取**，只需要访问`L1d`就行。

在`L2`阶段，三条新加的线基本重合，而且耗时比老的那条线高很多，大约在28个周期左右，差不多就是`L2`的访问时间。这表明，从`L2`到`L1d`的**预取**并没有生效。这是因为，对于最下面的线(`NPAD=0`)，由于结构小，8次循环后才需要访问一条新缓存线，而上面三条线对应的结构比较大，拿相对最小的`NPAD=7`来说，光是一次循环就需要访问一条新线，更不用说更大的`NPAD=15`和`NPAD=31`了。而预取逻辑是无法在每个周期装载新线的，因此每次循环都需要从`L2`读取，我们看到的就是从L2读取的时延。

更有趣的是工作集超过`L2`容量后的阶段。4条线远远地拉开了。元素的大小变成了主角，左右了性能。处理器应能识别每一步(stride)的大小，不去为`NPAD=15`和`NPAD=31`等获取那些实际并不需要的cache line(参见6.3.1)。元素大小对预取的约束是根源于硬件预取的限制：它无法跨越页边界。如果允许预取器跨越页边界，而下一页不存在或无效，那么OS还得去寻找它。这意味着，程序需要遭遇一次并非由它自己产生的页错误，这是完全不能接受的。在`NPAD=7`或者更大的时候，由于每个元素都至少需要一条cache line，预取器已经帮不上忙了，它没有足够的时间去从内存装载数据。

另一个导致慢下来的原因是TLB缓存的未命中。TLB是存储虚实地址映射的缓存，参见第4节。为了保持快速，TLB只有很小的容量。如果有大量页被反复访问，超出了TLB缓存容量，就会导致反复地进行地址翻译，这会耗费大量时间。TLB查找的代价分摊到所有元素上，如果元素越大，那么元素的数量越少，每个元素承担的那一份就越多。

为了观察TLB的影响，我们可以进行另两项测试。第一项：我们还是顺序存储列表中的元素，使`NPAD=7`，让每个元素占满整个cache line，第二项：我们将列表的每个元素存储在一个单独的页上，忽略每个页没有使用的部分以用来计算工作集的大小。（这样做可能不太一致，因为在前面的测试中，我计算了结构体中每个元素没有使用的部分，从而用来定义NPAD的大小，因此每个元素占满了整个页，这样以来工作集的大小将会有所不同。但是这不是这项测试的重点，预取的低效率多少使其有点不同）。结果表明，第一项测试中，每次列表的迭代都需要一个新的cache line，而且每64个元素就需要一个新的页。第二项测试中，每次迭代都会在一个新的页中加载一个新的cache line。

![](/images/posts/memory/cpumemory.59.png)

图 3.12: TLB 对顺序读的影响

结果见图3.12。该测试与图3.11是在同一台机器上进行的。基于可用RAM空间的有限性，测试设置容量空间大小为$$2^{24}$$字节，这就需要1GB的容量将对象放置在分页上。图3.12中下方的红色曲线正好对应了图3.11中`NPAD=7`的曲线。我们看到不同的步长显示了高速缓存`L1d`和`L2`的大小。第二条曲线看上去完全不同，其最重要的特点是当工作容量到达$$2^{13}$$字节时开始大幅度增长。这就是TLB缓存溢出的时候。我们能计算出一个64字节大小的元素的TLB缓存有64个输入。成本不会受页面错误影响，因为程序锁定了存储器以防止内存被换出。

可以看出，计算物理地址并把它存储在TLB中所花费的周期数量级是非常高的。图3.12的表格显示了一个极端的例子，但从中可以清楚的得到：TLB缓存效率降低的一个重要因素是NPAD值，效率随着NPAD值增大而降低。由于物理地址必须在cache line能被`L2`或主存读取之前计算出来，地址转换这个不利因素就增加了内存访问时间。这一点部分解释了为什么`NPAD=31`时每个列表元素的总花费比理论上的RAM访问时间要高。

![](/images/posts/memory/cpumemory.24.png)

图3.13 `NPAD=1`时的顺序读和写

通过查看链表元素被修改时测试数据的运行情况，我们可以窥见一些更详细的预取实现细节。图3.13显示了三条曲线。所有情况下元素宽度都为16个字节。第一条曲线“Follow”是熟悉的链表遍历在这里作为基线。第二条曲线，标记为“Inc”，仅仅在当前元素进入下一个前给其增加`pad[0]`成员。第三条曲线，标记为"Addnext0"，取出下一个元素的`pad[0]`链表元素并把它加和到当前链表元素的`pad[0]`成员。

在没运行时，大家可能会以为"Addnext0"更慢，因为它要做的事情更多：在没前进到下个元素之前就需要加载它的值。但实际的运行结果令人惊讶——在某些小工作集下，"Addnext0"比"Inc"更快。这是为什么呢？原因在于，系统一般会对下一个元素进行强制性**预取**。当程序前进到下个元素时，这个元素其实早已被预取在`L1d`里。因此，只要工作集比`L2`小，"Addnext0"的性能基本就能与"Follow"测试媲美。

但是，"Addnext0"比"Inc"更快超出`L2`，这是因为它需要从主存装载更多的数据。而在工作集达到$$2^{21}$$字节时，"Addnext0"的耗时达到了28个周期，是同期"Follow"14周期的两倍。这个两倍也很好解释。"Addnext0"和"Inc"涉及对内存的修改，因此`L2`的逐出操作不能简单地把数据一扔了事，而必须将它们回写入内存。因此FSB的可用带宽变成了一半，传输等量数据的耗时也就变成了原来的两倍。

![](/images/posts/memory/cpumemory.25.png)

图3.14: 更大L2/L3缓存的优势

决定顺序式缓存处理性能的另一个重要因素是缓存容量。虽然这一点比较明显，但还是值得一说。图3.14展示了128字节长元素的测试结果(64位机，`NPAD=15`)。这次我们比较三台不同计算机的曲线，两台P4，一台Core 2。两台P4的区别是缓存容量不同，一台是32k的`L1d`和1M的`L2`，一台是16K的`L1d`、512k的`L2`和2M的`L3`。Core 2那台则是32k的`L1d`和4M的`L2`。

图中最有趣的地方，并不是Core 2如何大胜两台P4，而是工作集开始增长到连末级缓存也放不下，需要主存被大量调用时部分（图中最右侧部分）。

![](/images/posts/memory/cpumem-table-3.2.png)

表3.2: 顺序访问与随机访问时L2命中与未命中的情况，NPAD=0

与我们预计的相似，最末级缓存越大，曲线停留在`L2`访问耗时区的时间越长。在$$2^{20}$$字节的工作集时，第二台P4(更老一些)比第一台P4要快上一倍，这要完全归功于更大的末级缓存。而Core 2拜它巨大的4M `L2`所赐，表现更为卓越。

对于随机的工作负荷而言，可能没有这么惊人的效果，但是，如果我们能将工作负荷进行一些裁剪，让它匹配末级缓存的容量，就完全可以得到非常大的性能提升。也是由于这个原因，有时候我们需要多花一些钱，买一个拥有更大缓存的处理器。

### 单线程随机访问模式的测量

前面我们已经看到，处理器能够利用`L1d`到`L2`之间的预取消除访问主存、甚至是访问`L2`的时延。

![](/images/posts/memory/cpumemory.26.png)

图3.15: 顺序读取vs随机读取，NPAD=0

但是，如果换成随机访问或者不可预测的访问，情况就大不相同了。图3.15比较了顺序读取与随机读取的耗时情况。换成随机之后，处理器无法再有效地预取数据，只有少数情况下靠运气刚好碰到先后访问的两个元素挨在一起的情形。

图3.15中有两个需要关注的地方。首先，在大的工作集下需要非常多的周期。这台机器访问主存的时间大约为200-300个周期，但图中的耗时甚至超过了450个周期。我们前面已经观察到过类似现象(对比图3.11)。这说明，处理器的自动预取在这里起到了反效果。

其次，代表随机访问的曲线在各个阶段不像顺序访问那样保持平坦，而是不断攀升。为了解释这个问题，我们测量了程序在不同工作集下对`L2`的访问情况。结果如图3.16和表3.2。

从图中可以看出，当工作集大小超过`L2`时，未命中率(`L2`未命中次数/`L2`访问次数)开始上升。整条曲线的走向与图3.15有些相似: 先急速爬升，随后缓缓下滑，最后再度爬升。它与耗时图有紧密的关联。`L2`未命中率会一直爬升到100%为止。只要工作集足够大(并且内存也足够大)，就可以将L2中cache line随机选取或加载的可能性降到非常低。

缓存未命中率的攀升已经可以解释一部分的开销。除此以外，还有一个因素。观察表3.2的`L2/#Iter`列，可以看到每个循环对`L2`的使用次数在增长。由于工作集每次为上一次的两倍，如果没有缓存的话，内存的访问次数也将是上一次的两倍。在按顺序访问时，由于缓存的帮助及完美的预见性，对`L2`使用的增长比较平缓，完全取决于工作集的增长速度。

![](/images/posts/memory/cpumemory.32.png)

图3.16: L2d未命中率

![](/images/posts/memory/cpumemory.67.png)

图3.17: 页意义上(Page-Wise)的随机化，NPAD=7

而换成随机访问后，单位耗时的增长超过了工作集的增长，根源是TLB未命中率的上升。图3.17描绘的是`NPAD=7`时随机访问的耗时情况。这一次，我们修改了随机访问的方式。正常情况下是把整个列表作为一个块进行随机(以∞表示)，而其它11条线则是在小一些的块里进行随机。例如，标签为'60'的线表示以60页(245760字节)为单位进行随机。先遍历完这个块里的所有元素，再访问另一个块。这样一来，可以保证任意时刻使用的TLB条目数都是有限的。

`NPAD=7`对应于64字节，正好等于cache line的长度。由于元素顺序随机，硬件预取不可能有任何效果，特别是在元素较多的情况下。这意味着，分块随机时的`L2`未命中率与整个列表随机时的未命中率没有本质的差别。随着块的增大，曲线逐渐逼近整个列表随机对应的曲线。这说明，在这个测试里，性能受到TLB命中率的影响很大，如果我们能提高TLB命中率，就能大幅度地提升性能(在后面的一个例子里，性能提升了38%之多)。

## 3.3.3 写入时的行为

在我们开始研究多个线程或进程同时使用相同内存之前，先来看一下缓存实现的一些细节。我们要求缓存是一致的，而且这种一致性必须对用户级代码完全透明。而内核代码则有所不同，它有时候需要对缓存进行转储(flush)。

这意味着，如果对cache line进行了修改，那么在这个时间点之后，系统的结果应该是与没有缓存的情况下是相同的，即主存的对应位置也已经被修改的状态。这种要求可以通过两种方式或策略实现：

* 写穿透(write-through)
* 回写(write-back)

写穿透比较简单。当修改cache line时，处理器立即将它写入主存。这样可以保证主存与缓存的内容永远保持一致。当cache line被替代时，只需要简单地将它丢弃即可。这种策略很简单，但是速度比较慢。如果某个程序反复修改一个本地变量，可能导致FSB上产生大量数据流，而不管这个变量是不是有人在用，或者是不是短期变量。

回写比较复杂。当修改cache line时，处理器不再马上将它写入主存，而是打上已弄脏(dirty)的标记。当以后某个时间点cache line被丢弃时，这个已弄脏标记会通知处理器把数据回写到主存中，而不是简单地扔掉。

回写有时候会有非常不错的性能，因此较好的系统大多采用这种方式。采用回写时，处理器们甚至可以利用FSB的空闲容量来存储cache line。这样一来，当需要缓存空间时，处理器只需清除脏标记，丢弃cache line即可。

但回写也有一个很大的问题。当有多个处理器(或核心、超线程)访问同一块内存时，必须确保它们在任何时候看到的都是相同的内容。如果cache line在其中一个处理器上弄脏了(修改了，但还没回写主存)，而第二个处理器刚好要读取同一个内存地址，那么这个读操作不能去读主存，而需要读第一个处理器的cache line。在下一节中，我们将研究如何实现这种需求。

在此之前，还有其它两种缓存策略需要提一下:

* 写入合并
* 不可缓存

这两种策略用于真实内存不支持的特殊地址区，内核为地址区设置这些策略(x86处理器利用内存类型范围寄存器MTRR)，余下的部分自动进行。MTRR还可用于写穿透和回写策略的选择。

写入合并是一种有限的缓存优化策略，更多地用于显卡等设备之上的内存。由于设备的传输开销比本地内存要高的多，因此避免进行过多的传输显得尤为重要。如果仅仅因为修改了cache line上的一个字，就传输整条cache line，而下个操作刚好是修改cache line上的下一个字，那么这次传输就过于浪费了。而这恰恰对于显卡来说是比较常见的情形——屏幕上水平邻接的像素往往在内存中也是靠在一起的。顾名思义，写入合并是在写出cache line前，先将多个写入访问合并起来。在理想的情况下，cache line被逐字逐字地修改，只有当写入最后一个字时，才将整条cache line写入内存，从而极大地加速内存的访问。

最后来讲一下不可缓存的内存。一般指的是不被RAM支持的内存位置，它可以是硬编码的特殊地址，承担CPU以外的某些功能。对于商用硬件来说，比较常见的是映射到外部卡或设备的地址。在嵌入式主板上，有时也有类似的地址，用来开关LED。对这些地址进行缓存显然没有什么意义。比如上述的LED，一般是用来调试或报告状态，显然应该尽快点亮或关闭。而对于那些PCI卡上的内存，由于不需要CPU的干涉即可更改，也不该缓存。

## 3.3.4 多处理器支持

在上节中我们已经指出当多处理器开始发挥作用的时候所遇到的问题。甚至对于那些不共享的高速级别的缓存（至少在`L1d`级别）的多核处理器也有问题。

直接提供从一个处理器到另一处理器的高速访问，这是完全不切实际的。从一开始，连接速度根本就不够快。实际的选择是，在其需要的情况下，转移cache中的内容到其他处理器。需要注意的是，这同样应用在相同处理器上无需共享的高速缓存。

现在的问题是，当该cache line转移的时候会发生什么？这个问题回答起来相当容易：当一个处理器需要在另一个处理器的高速缓存中读或者写的脏的cache line的时候。但怎样处理器怎样确定在另一个处理器的缓存中的cache line是脏的？假设它仅仅是因为一个cache line被另一个处理器加载将是次优的（最好的）。通常情况下，大多数的内存访问是只读cache line，从而并不弄脏cache line。在cache line上处理器的操作非常频繁，因此意味着每一次写访问后，要广播关于cache line的改变是不切实际的。

多年来，人们开发除了MESI缓存一致性协议(MESI=Modified, Exclusive, Shared, Invalid，变更的、独占的、共享的、无效的)。协议的名称来自协议中cache line可以进入的四种状态:

* **变更的**: 本地处理器修改了cache line。同时暗示，它是所有缓存中唯一的拷贝。
* **独占的**: cache line没有被修改，而且没有被装入其它处理器的缓存。
* **共享的**: cache line没有被修改，但可能已被装入其它处理器的缓存。
* **无效的**: cache line无效，即，未被使用。

MESI协议开发了很多年，最初的版本比较简单，但是效率也比较差。现在的版本通过以上4个状态可以有效地实现回写式缓存，同时支持不同处理器对只读数据的并发访问。

![](/images/posts/memory/cpumemory.13.png)

图3.18: MESI协议的状态跃迁图

在协议中，通过处理器监听其它处理器的活动，不需太多努力即可实现状态变更。处理器将操作发布在外部引脚上，使外部可以了解到处理过程。目标的cache line地址则可以在地址总线上看到。在下文讲述状态时，我们将介绍总线参与的时机（图3.18）。

一开始，所有cache line都是空的，缓存为无效(Invalid)状态。当有数据装进缓存供写入时，缓存变为变更(Modified)状态。如果有数据装进缓存供读取，那么新状态取决于其它处理器是否已经加载了同一条cache line。如果是，那么新状态变成共享(Shared)状态，否则变成独占(Exclusive)状态。

如果本地处理器对某条Modified状态的cache line进行读写，那么直接使用缓存内容，状态保持不变。如果第二个处理器希望读它，那么第一个处理器将内容发给第二个处理器，然后可以将缓存状态置为Shared。而发给第二个处理器的数据由内存控制器接收，并回写入内存。如果这一步没有发生，就不能将这条线置为Shared。如果第二个处理器希望的是写，那么第一个处理器将内容发给它后，将缓存置为Invalid。这就是臭名昭著的"请求所有权(Request For Ownership,RFO)"操作。在末级缓存执行RFO操作的代价比较高。如果是写穿透缓存，还要加上将内容写入上一层缓存或主存的时间，进一步提升了代价。

对于Shared状态的cache line，本地处理器的读取操作并不需要修改状态，而且可以直接从缓存满足。而本地处理器的写入操作则需要将状态置为Modified，而且需要将cache line在其它处理器的所有拷贝置为Invalid。因此，这个写入操作需要通过RFO消息发通知其它处理器。如果第二个处理器请求读取，无事发生。因为主存已经包含了当前数据，而且状态已经为Shared。如果第二个处理器需要写入，则将cache line置为Invalid。不需要总线操作。

Exclusive状态与Shared状态很像，只有一个不同之处: 在Exclusive状态时，本地写入操作不需要在总线上通知，因为本地的缓存是系统中唯一的拷贝。这是一个巨大的优势，所以处理器会尽量将cache line保留在Exclusive状态，而不是Shared状态。只有在信息不可用时，才退而求其次选择shared。放弃Exclusive不会引起任何功能缺失，但会导致性能下降，因为E→M要远远快于S→M。

从以上的描述中应该已经可以看出，在多处理器环境下，哪一步的代价比较大了。填充缓存的代价当然还是很高，但我们还需要留意RFO消息。一旦涉及RFO，操作就快不起来了。

RFO消息在两种情况下是必需的:

* 线程从一个处理器迁移到另一个处理器，需要将所有cache line移到新处理器。
* 某条缓存线确实需要被两个处理器使用。{对于同一处理器的两个核心，也有同样的情况，只是代价稍低。RFO消息可能会被发送多次。}

多线程或多进程的程序总是需要同步，而这种同步依赖内存来实现。因此，有些RFO消息是合理的，但仍然需要尽量降低发送频率。除此以外，还有其它来源的RFO。在第6节中，我们将解释这些场景。缓存一致性协议的消息必须发给系统中所有处理器。只有当协议确定已经给过所有处理器响应机会之后，才能进行状态跃迁。也就是说，协议的速度取决于最长响应时间。*这也是现在能看到三插槽AMD Opteron系统的原因。这类系统只有三个超级链路(hyperlink)，其中一个连接南桥，每个处理器之间都只有一跳的距离*。总线上可能会发生冲突，NUMA系统的延时很大，突发的流量会拖慢通信。这些都是让我们避免无谓流量的充足理由。

此外，关于多处理器还有一个问题。虽然它的影响与具体机器密切相关，但根源是唯一的：FSB是共享的。在大多数情况下，所有处理器通过唯一的总线连接到内存控制器(参见图2.1)。如果一个处理器就能占满总线(十分常见)，那么共享总线的两个或四个处理器显然只会得到更有限的带宽。

即使每个处理器有自己连接内存控制器的总线，如图2.2，但还需要通往内存模块的总线。一般情况下，这种总线只有一条。退一步说，即使像图2.2那样不止一条，对同一个内存模块的并发访问也会限制它的带宽。

对于每个处理器拥有本地内存的AMD模型来说，也是同样的问题。的确，所有处理器可以非常快速地同时访问它们自己的内存。但是，多线程呢？多进程呢？它们仍然需要通过访问同一块内存来进行同步。

对同步来说，有限的带宽严重地制约着并发度。程序需要更加谨慎的设计，将不同处理器访问同一块内存的机会降到最低。以下的测试展示了这一点，还展示了与多线程代码相关的其它效果。

# 多线程测量

为了帮助大家理解问题的严重性，我们来看一些曲线图，主角也是前文的那个程序。只不过这一次，我们运行多个线程，并测量这些线程中最快那个的运行时间。也就是说，等它们全部运行完是需要更长时间的。我们用的机器有4个处理器，而测试是做多跑4个线程。所有处理器共享同一条通往内存控制器的总线，另外，通往内存模块的总线也只有一条。

![](/images/posts/memory/cpumemory.30.png)

图3.19: 顺序读操作，多线程

图3.19展示了顺序读访问时的性能，元素为128字节长(64位计算机，NPAD=15)。对于单线程的曲线，我们预计是与图3.11相似，只不过是换了一台机器，所以实际的数字会有些小差别。

更重要的部分当然是多线程的环节。由于是只读，不会去修改内存，不会尝试同步。但即使不需要RFO，而且所有缓存线都可共享，性能仍然分别下降了18%(双线程)和34%(四线程)。由于不需要在处理器之间传输缓存，因此这里的性能下降完全由以下两个瓶颈之一或同时引起: 一是从处理器到内存控制器的共享总线，二是从内存控制器到内存模块的共享总线。当工作集超过L3后，三种情况下都要预取新元素，而即使是双线程，可用的带宽也无法满足线性扩展(无惩罚)。

当加入修改之后，场面更加难看了。图3.20展示了顺序递增测试的结果。

![](/images/posts/memory/cpumemory.29.png)

图3.20: 顺序递增，多线程

图中Y轴采用的是对数刻度，不要被看起来很小的差值欺骗了。现在，双线程的性能惩罚仍然是18%，但四线程的惩罚飙升到了93%！原因在于，采用四线程时，预取的流量与回写的流量加在一起，占满了整个总线。

我们用对数刻度(Y轴)来展示`L1d`范围的结果。可以发现，当超过一个线程后，`L1d`就无力了。单线程时，仅当工作集超过`L1d`时访问时间才会超过20个周期，而多线程时，即使在很小的工作集情况下，访问时间也达到了那个水平。

这里并没有揭示问题的另一方面，主要是用这个程序很难进行测量。问题是这样的，我们的测试程序修改了内存，所以本应看到RFO的影响，但在结果中，我们并没有在`L2`阶段看到更大的开销。原因在于，要看到RFO的影响，程序必须使用大量内存，而且所有线程必须同时访问同一块内存。如果没有大量的同步，这是很难实现的，而如果加入同步，则会占满执行时间。

![](/images/posts/memory/cpumemory.31.png)

图3.21: 随机的Addnextlast，多线程

最后，在图3.21中，我们展示了随机访问的Addnextlast测试的结果。这里主要是为了让大家感受一下这些巨大到爆的数字。极端情况下，甚至用了1500个周期才处理完一个元素。如果加入更多线程，真是不可想象哪。我们把多线程的效能总结了一下:

![](/images/posts/memory/cpumem-table-3.3.png)

表3.3: 多线程的效能

这个表展示了图3.21中多线程运行大工作集时的效能。表中的数字表示测试程序在使用多线程处理大工作集时可能达到的最大加速倍数。双线程和四线程的理论最大加速倍数分别是2和4。从表中数据来看，双线程的结果还能接受，但四线程的结果表明，扩展到双线程以上是没有什么意义的，带来的收益可以忽略不计。只要我们把图3.21换个方式呈现，就可以很容易看清这一点。

![](/images/posts/memory/cpumemory.28.png)

图3.22: 通过并行化实现的加速倍数

图3.22中的曲线展示了加速倍数，即多线程相对于单线程所能获取的性能加成值。测量值的精确度有限，因此我们需要忽略比较小的那些数字。可以看到，在`L2`与`L3`范围内，多线程基本可以做到线性加速，双线程和四线程分别达到了2和4的加速倍数。但是，一旦工作集的大小超出`L3`，曲线就崩塌了，双线程和四线程降到了基本相同的数值(参见表3.3中第4列)。也是部分由于这个原因，我们很少看到4CPU以上的主板共享同一个内存控制器。如果需要配置更多处理器，我们只能选择其它的实现方式(参见第5节)。

可惜，上图中的数据并不是普遍情况。在某些情况下，即使工作集能够放入末级缓存，也无法实现线性加速。实际上，这反而是正常的，因为普通的线程都有一定的耦合关系，不会像我们的测试程序这样完全独立。而反过来说，即使是很大的工作集，即使是两个以上的线程，也是可以通过并行化受益的，但是需要程序员的聪明才智。我们会在第6节进行一些介绍。

### 特例: 超线程

由CPU实现的超线程(有时又叫对称多线程，SMT)是一种比较特殊的情况，每个线程并不能真正并发地运行。它们共享着除寄存器外的绝大多数处理资源。每个核心和CPU仍然是并行工作的，但核心上的线程则受到这个限制。理论上，每个核心可以有大量线程，不过到目前为止，Intel的CPU最多只有两个线程。CPU负责对各线程进行时分复用，但这种复用本身并没有多少厉害。它真正的优势在于，CPU可以在当前运行的超线程发生延迟时，调度另一个线程。这种延迟一般由内存访问引起。

如果两个线程运行在一个超线程核心上，那么只有当两个线程合起来的运行时间少于单线程运行时间时，效率才会比较高。我们可以将通常先后发生的内存访问叠合在一起，以实现这个目标。有一个简单的计算公式，可以帮助我们计算如果需要某个加速倍数，最少需要多少的缓存命中率。

程序的执行时间可以通过一个只有一级缓存的简单模型来进行估算(参见[htimpact]):

$$
T_{exe} = N[(1-F_{mem})T_{proc} + F_{mem}(G_{hit}T_{cache} + (1-G_{hit})T_{miss})]
$$

各变量的含义如下:

$$
N = 指令数 \\
F_{mem} = N中访问内存的比例 \\
G_{hit} = 命中缓存的比例 \\
T_{proc} = 每条指令所用的周期数 \\
T_{cache} = 缓存命中所用的周期数 \\
T_{miss} = 缓冲未命中所用的周期数 \\
T_{exe} = 程序的执行时间
$$

为了让任何判读使用双线程，两个线程之中任一线程的执行时间最多为单线程指令的一半。两者都有一个唯一的变量缓存命中数。 如果我们要解决最小缓存命中率相等的问题需要使我们获得的线程的执行率不少于50%或更多，如图 3.23.

For it to make any sense to use two threads the execution time of each of the two threads must be at most half of that of the single-threaded code. The only variable on either side is the number of cache hits. If we solve the equation for the minimum cache hit rate required to not slow down the thread execution by 50% or more we get the graph in Figure 3.23.

![](/images/posts/memory/cpumemory.14.png)

图 3.23: 最小缓存命中率-加速

X轴表示单线程指令的缓存命中率Ghit，Y轴表示多线程指令所需的缓存命中率。这个值永远不能高于单线程命中率，否则，单线程指令也会使用改良的指令。为了使单线程的命中率在低于55%的所有情况下优于使用多线程，cup要或多或少的足够空闲因为缓存丢失会运行另外一个超线程。

The X–axis represents the cache hit rate Ghit of the single-thread code. The Y–axis shows the required cache hit rate for the multi-threaded code. This value can never be higher than the single-threaded hit rate since, otherwise, the single-threaded code would use that improved code, too. For single-threaded hit rates below 55% the program can in all cases benefit from using threads. The CPU is more or less idle enough due to cache misses to enable running a second hyper-thread.

绿色区域是我们的目标。如果线程的速度没有慢过50%，而每个线程的工作量只有原来的一半，那么它们合起来的耗时应该会少于单线程的耗时。对我们用的示例系统来说(使用超线程的P4机器)，如果单线程代码的命中率为60%，那么多线程代码至少要达到10%才能获得收益。这个要求一般来说还是可以做到的。但是，如果单线程代码的命中率达到了95%，那么多线程代码要做到80%才行。这就很难了。而且，这里还涉及到超线程，在两个超线程的情况下，每个超线程只能分到一半的有效缓存。因为所有超线程是使用同一个缓存来装载数据的，如果两个超线程的工作集没有重叠，那么原始的95%也会被打对折——47%，远低于80%。

The green area is the target. If the slowdown for the thread is less than 50% and the workload of each thread is halved the combined runtime might be less than the single-thread runtime. For the modeled system here (numbers for a P4 with hyper-threads were used) a program with a hit rate of 60% for the single-threaded code requires a hit rate of at least 10% for the dual-threaded program. That is usually doable. But if the single-threaded code has a hit rate of 95% then the multi-threaded code needs a hit rate of at least 80%. That is harder. Especially, and this is the problem with hyper-threads, because now the effective cache size (L1d here, in practice also L2 and so on) available to each hyper-thread is cut in half. Both hyper-threads use the same cache to load their data. If the working set of the two threads is non-overlapping the original 95% hit rate could also be cut in half and is therefore much lower than the required 80%.

因此，超线程只在某些情况下才比较有用。单线程代码的缓存命中率必须低到一定程度，从而使缓存容量变小时新的命中率仍能满足要求。只有在这种情况下，超线程才是有意义的。在实践中，采用超线程能否获得更快的结果，取决于处理器能否有效地将每个进程的等待时间与其它进程的执行时间重叠在一起。并行化也需要一定的开销，需要加到总的运行时间里，这个开销往往是不能忽略的。

在6.3.4节中，我们会介绍一种技术，它将多个线程通过公用缓存紧密地耦合起来。这种技术适用于许多场合，前提是程序员们乐意花费时间和精力扩展自己的代码。

Hyper-Threads are therefore only useful in a limited range of situations. The cache hit rate of the single-threaded code must be low enough that given the equations above and reduced cache size the new hit rate still meets the goal. Then and only then can it make any sense at all to use hyper-threads. Whether the result is faster in practice depends on whether the processor is sufficiently able to overlap the wait times in one thread with execution times in the other threads. The overhead of parallelizing the code must be added to the new total runtime and this additional cost often cannot be neglected.

In Section 6.3.4 we will see a technique where threads collaborate closely and the tight coupling through the common cache is actually an advantage. This technique can be applicable to many situations if only the programmers are willing to put in the time and energy to extend their code.

如果两个超线程执行完全不同的代码(两个线程就像被当成两个处理器，分别执行不同进程)，那么缓存容量就真的会降为一半，导致缓冲未命中率大为攀升，这一点应该是很清楚的。这样的调度机制是很有问题的，除非你的缓存足够大。所以，除非程序的工作集设计得比较合理，能够确实从超线程获益，否则还是建议在BIOS中把超线程功能关掉。{我们可能会因为另一个原因 开启 超线程，那就是调试，因为SMT在查找并行代码的问题方面真的非常好用。}

What should be clear is that if the two hyper-threads execute completely different code (i.e., the two threads are treated like separate processors by the OS to execute separate processes) the cache size is indeed cut in half which means a significant increase in cache misses. Such OS scheduling practices are questionable unless the caches are sufficiently large. Unless the workload for the machine consists of processes which, through their design, can indeed benefit from hyper-threads it might be best to turn off hyper-threads in the computer's BIOS. {Another reason to keep hyper-threads enabled is debugging. SMT is astonishingly good at finding some sets of problems in parallel code.}

## 3.3.5 其它细节

我们已经介绍了地址的组成，即标签、集合索引和偏移三个部分。那么，实际会用到什么样的地址呢？目前，处理器一般都向进程提供虚拟地址空间，意味着我们有两种不同的地址: 虚拟地址和物理地址。

虚拟地址有个问题——并不唯一。随着时间的变化，虚拟地址可以变化，指向不同的物理地址。同一个地址在不同的进程里也可以表示不同的物理地址。那么，是不是用物理地址会比较好呢？

So far we talked about the address as consisting of three parts, tag, set index, and cache line offset. But what address is actually used? All relevant processors today provide virtual address spaces to processes, which means that there are two different kinds of addresses: virtual and physical.

The problem with virtual addresses is that they are not unique. A virtual address can, over time, refer to different physical memory addresses. The same address in different process also likely refers to different physical addresses. So it is always better to use the physical memory address, right?

问题是，处理器指令用的虚拟地址，而且需要在内存管理单元(MMU)的协助下将它们翻译成物理地址。这并不是一个很小的操作。在执行指令的管线(pipeline)中，物理地址只能在很后面的阶段才能得到。这意味着，缓存逻辑需要在很短的时间里判断地址是否已被缓存过。而如果可以使用虚拟地址，缓存查找操作就可以更早地发生，一旦命中，就可以马上使用内存的内容。结果就是，使用虚拟内存后，可以让管线把更多内存访问的开销隐藏起来。

The problem here is that instructions use virtual addresses and these have to be translated with the help of the Memory Management Unit (MMU) into physical addresses. This is a non-trivial operation. In the pipeline to execute an instruction the physical address might only be available at a later stage. This means that the cache logic has to be very quick in determining whether the memory location is cached. If virtual addresses could be used the cache lookup can happen much earlier in the pipeline and in case of a cache hit the memory content can be made available. The result is that more of the memory access costs could be hidden by the pipeline.

处理器的设计人员们现在使用虚拟地址来标记第一级缓存。这些缓存很小，很容易被清空。在进程页表树发生变更的情况下，至少是需要清空部分缓存的。如果处理器拥有指定变更地址范围的指令，那么可以避免缓存的完全刷新。由于一级缓存L1i及L1d的时延都很小(~3周期)，基本上必须使用虚拟地址。

对于更大的缓存，包括L2和L3等，则需要以物理地址作为标签。因为这些缓存的时延比较大，虚拟到物理地址的映射可以在允许的时间里完成，而且由于主存时延的存在，重新填充这些缓存会消耗比较长的时间，刷新的代价比较昂贵。

Processor designers are currently using virtual address tagging for the first level caches. These caches are rather small and can be cleared without too much pain. At least partial clearing the cache is necessary if the page table tree of a process changes. It might be possible to avoid a complete flush if the processor has an instruction which specifies the virtual address range which has changed. Given the low latency of L1i and L1d caches (~3 cycles) using virtual addresses is almost mandatory.

For larger caches including L2, L3, ... caches physical address tagging is needed. These caches have a higher latency and the virtual→physical address translation can finish in time. Because these caches are larger (i.e., a lot of information is lost when they are flushed) and refilling them takes a long time due to the main memory access latency, flushing them often would be costly.

一般来说，我们并不需要了解这些缓存处理地址的细节。我们不能更改它们，而那些可能影响性能的因素，要么是应该避免的，要么是有很高代价的。填满缓存是不好的行为，缓存线都落入同一个集合，也会让缓存早早地出问题。对于后一个问题，可以通过缓存虚拟地址来避免，但作为一个用户级程序，是不可能避免缓存物理地址的。我们唯一可以做的，是尽最大努力不要在同一个进程里用多个虚拟地址映射同一个物理地址。

It should, in general, not be necessary to know about the details of the address handling in those caches. They cannot be changed and all the factors which would influence the performance are normally something which should be avoided or is associated with high cost. Overflowing the cache capacity is bad and all caches run into problems early if the majority of the used cache lines fall into the same set. The latter can be avoided with virtually addressed caches but is impossible for user-level processes to avoid for caches addressed using physical addresses. The only detail one might want to keep in mind is to not map the same physical memory location to two or more virtual addresses in the same process, if at all possible.

另一个细节对程序员们来说比较乏味，那就是缓存的替换策略。大多数缓存会优先逐出最近最少使用(Least Recently Used,LRU)的元素。这往往是一个效果比较好的策略。在关联性很大的情况下(随着以后核心数的增加，关联性势必会变得越来越大)，维护LRU列表变得越来越昂贵，于是我们开始看到其它的一些策略。

在缓存的替换策略方面，程序员可以做的事情不多。如果缓存使用物理地址作为标签，我们是无法找出虚拟地址与缓存集之间关联的。有可能会出现这样的情形: 所有逻辑页中的缓存线都映射到同一个缓存集，而其它大部分缓存却空闲着。即使有这种情况，也只能依靠OS进行合理安排，避免频繁出现。

Another detail of the caches which is rather uninteresting to programmers is the cache replacement strategy. Most caches evict the Least Recently Used (LRU) element first. This is always a good default strategy. With larger associativity (and associativity might indeed grow further in the coming years due to the addition of more cores) maintaining the LRU list becomes more and more expensive and we might see different strategies adopted.

As for the cache replacement there is not much a programmer can do. If the cache is using physical address tags there is no way to find out how the virtual addresses correlate with the cache sets. It might be that cache lines in all logical pages are mapped to the same cache sets, leaving much of the cache unused. If anything, it is the job of the OS to arrange that this does not happen too often.

虚拟化的出现使得这一切变得更加复杂。现在不仅操作系统可以控制物理内存的分配。虚拟机监视器（VMM，也称为 hypervisor）也负责分配内存。

对程序员来说，最好 a) 完全使用逻辑内存页面 b) 在有意义的情况下，使用尽可能大的页面大小来分散物理地址。更大的页面大小也有其他好处，不过这是另一个话题（见第4节）。

With the advent of virtualization things get even more complicated. Now not even the OS has control over the assignment of physical memory. The Virtual Machine Monitor (VMM, aka hypervisor) is responsible for the physical memory assignment.

The best a programmer can do is to a) use logical memory pages completely and b) use page sizes as large as meaningful to diversify the physical addresses as much as possible. Larger page sizes have other benefits, too, but this is another topic (see Section 4).

# 3.4 指令缓存

其实，不光处理器使用的数据被缓存，它们执行的指令也是被缓存的。只不过，指令缓存的问题相对来说要少得多，因为:

* 执行的代码量取决于代码大小。而代码大小通常取决于问题复杂度。问题复杂度则是固定的。
* 程序的数据处理逻辑是程序员设计的，而程序的指令却是编译器生成的。编译器的作者知道如何生成优良的代码。
* 程序的流向比数据访问模式更容易预测。现如今的CPU很擅长模式检测，对预取很有利。
* 代码永远都有良好的时间局部性和空间局部性。

有一些准则是需要程序员们遵守的，但大都是关于如何使用工具的，我们会在第6节介绍它们。而在这里我们只介绍一下指令缓存的技术细节。

Not just the data used by the processor is cached; the instructions executed by the processor are also cached. However, this cache is much less problematic than the data cache. There are several reasons:

The quantity of code which is executed depends on the size of the code that is needed. The size of the code in general depends on the complexity of the problem. The complexity of the problem is fixed.
While the program's data handling is designed by the programmer the program's instructions are usually generated by a compiler. The compiler writers know about the rules for good code generation.
Program flow is much more predictable than data access patterns. Today's CPUs are very good at detecting patterns. This helps with prefetching.
Code always has quite good spatial and temporal locality.
There are a few rules programmers should follow but these mainly consist of rules on how to use the tools. We will discuss them in Section 6. Here we talk only about the technical details of the instruction cache.

随着CPU的核心频率大幅上升，缓存与核心的速度差越拉越大，CPU的处理开始管线化。也就是说，指令的执行分成若干阶段。首先，对指令进行解码，随后，准备参数，最后，执行它。这样的管线可以很长(例如，Intel的Netburst架构超过了20个阶段)。在管线很长的情况下，一旦发生延误(即指令流中断)，需要很长时间才能恢复速度。管线延误发生在这样的情况下: 下一条指令未能正确预测，或者装载下一条指令耗时过长(例如，需要从内存读取时)。

Ever since the core clock of CPUs increased dramatically and the difference in speed between cache (even first level cache) and core grew, CPUs have been pipelined. That means the execution of an instruction happens in stages. First an instruction is decoded, then the parameters are prepared, and finally it is executed. Such a pipeline can be quite long (> 20 stages for Intel's Netburst architecture). A long pipeline means that if the pipeline stalls (i.e., the instruction flow through it is interrupted) it takes a while to get up to speed again. Pipeline stalls happen, for instance, if the location of the next instruction cannot be correctly predicted or if it takes too long to load the next instruction (e.g., when it has to be read from memory).

为了解决这个问题，CPU的设计人员们在分支预测上投入大量时间和芯片资产(chip real estate)，以降低管线延误的出现频率。

在CISC处理器上，指令的解码阶段也需要一些时间。x86及x86-64处理器尤为严重。近年来，这些处理器不再将指令的原始字节序列存入L1i，而是缓存解码后的版本。这样的L1i被叫做“追踪缓存(trace cache)”。追踪缓存可以在命中的情况下让处理器跳过管线最初的几个阶段，在管线发生延误时尤其有用。

As a result CPU designers spend a lot of time and chip real estate on branch prediction so that pipeline stalls happen as infrequently as possible.

On CISC processors the decoding stage can also take some time. The x86 and x86-64 processors are especially affected. In recent years these processors therefore do not cache the raw byte sequence of the instructions in L1i but instead they cache the decoded instructions. L1i in this case is called the “trace cache”. Trace caching allows the processor to skip over the first steps of the pipeline in case of a cache hit which is especially good if the pipeline stalled.

前面说过，L2以上的缓存是统一缓存，既保存代码，也保存数据。显然，这里保存的代码是原始字节序列，而不是解码后的形式。

* 在提高性能方面，与指令缓存相关的只有很少的几条准则:
* 生成尽量少的代码。也有一些例外，如出于管线化的目的需要更多的代码，或使用小代码会带来过高的额外开销。

尽量帮助处理器作出良好的预取决策，可以通过代码布局或显式预取来实现。
这些准则一般会由编译器的代码生成阶段强制执行。至于程序员可以参与的部分，我们会在第6节介绍。

As said before, the caches from L2 on are unified caches which contain both code and data. Obviously here the code is cached in the byte sequence form and not decoded.

To achieve the best performance there are only a few rules related to the instruction cache:

Generate code which is as small as possible. There are exceptions when software pipelining for the sake of using pipelines requires creating more code or where the overhead of using small code is too high.
Whenever possible, help the processor to make good prefetching decisions. This can be done through code layout or with explicit prefetching.
These rules are usually enforced by the code generation of a compiler. There are a few things the programmer can do and we will talk about them in Section 6.

## 3.4.1 自修改的代码

在计算机的早期岁月里，内存十分昂贵。人们想尽千方百计，只为了尽量压缩程序容量，给数据多留一些空间。其中，有一种方法是修改程序自身，称为自修改代码(SMC)。现在，有时候我们还能看到它，一般是出于提高性能的目的，也有的是为了攻击安全漏洞。

一般情况下，应该避免SMC。虽然一般情况下没有问题，但有时会由于执行错误而出现性能问题。显然，发生改变的代码是无法放入追踪缓存(追踪缓存放的是解码后的指令)的。即使没有使用追踪缓存(代码还没被执行或有段时间没执行)，处理器也可能会遇到问题。如果某个进入管线的指令发生了变化，处理器只能扔掉目前的成果，重新开始。在某些情况下，甚至需要丢弃处理器的大部分状态。

In early computer days memory was a premium. People went to great lengths to reduce the size of the program to make more room for program data. One trick frequently deployed was to change the program itself over time. Such Self Modifying Code (SMC) is occasionally still found, these days mostly for performance reasons or in security exploits.

SMC should in general be avoided. Though it is generally executed correctly there are boundary cases in which it is not and it creates performance problems if not done correctly. Obviously, code which is changed cannot be kept in the trace cache which contains the decoded instructions. But even if the trace cache is not used because the code has not been executed at all (or for some time) the processor might have problems. If an upcoming instruction is changed while it already entered the pipeline the processor has to throw away a lot of work and start all over again. There are even situations where most of the state of the processor has to be tossed away.

最后，由于处理器认为代码页是不可修改的(这是出于简单化的考虑，而且在99.9999999%情况下确实是正确的)，L1i用到并不是MESI协议，而是一种简化后的SI协议。这样一来，如果万一检测到修改的情况，就需要作出大量悲观的假设。

因此，对于SMC，强烈建议能不用就不用。现在内存已经不再是一种那么稀缺的资源了。最好是写多个函数，而不要根据需要把一个函数改来改去。也许有一天可以把SMC变成可选项，我们就能通过这种方式检测入侵代码。如果一定要用SMC，应该让写操作越过缓存，以免由于L1i需要L1d里的数据而产生问题。更多细节，请参见6.1节。

Finally, since the processor assumes - for simplicity reasons and because it is true in 99.9999999% of all cases - that the code pages are immutable, the L1i implementation does not use the MESI protocol but instead a simplified SI protocol. This means if modifications are detected a lot of pessimistic assumptions have to be made.

It is highly advised to avoid SMC whenever possible. Memory is not such a scarce resource anymore. It is better to write separate functions instead of modifying one function according to specific needs. Maybe one day SMC support can be made optional and we can detect exploit code trying to modify code this way. If SMC absolutely has to be used, the write operations should bypass the cache as to not create problems with data in L1d needed in L1i. See Section 6.1 for more information on these instructions.

在Linux上，判断程序是否包含SMC是很容易的。利用正常工具链(toolchain)构建的程序代码都是写保护(write-protected)的。程序员需要在链接时施展某些关键的魔术才能生成可写的代码页。现代的Intel x86和x86-64处理器都有统计SMC使用情况的专用计数器。通过这些计数器，我们可以很容易判断程序是否包含SMC，即使它被准许运行。

Normally on Linux it is easy to recognize programs which contain SMC. All program code is write-protected when built with the regular toolchain. The programmer has to perform significant magic at link time to create an executable where the code pages are writable. When this happens, modern Intel x86 and x86-64 processors have dedicated performance counters which count uses of self-modifying code. With the help of these counters it is quite easily possible to recognize programs with SMC even if the program will succeed due to relaxed permissions.

# 3.5 缓存未命中的因素

我们已经看过内存访问没有命中缓存时，那陡然猛涨的高昂代价。但是有时候，这种情况又是无法避免的，因此我们需要对真正的代价有所认识，并学习如何缓解这种局面。

We have already seen that when memory accesses miss the caches, the costs skyrocket. Sometimes this is not avoidable and it is important to understand the actual costs and what can be done to mitigate the problem.

## 3.5.1 缓存与内存带宽

为了更好地理解处理器的能力，我们测量了各种理想环境下能够达到的带宽值。由于不同处理器的版本差别很大，所以这个测试比较有趣，也因为如此，这一节都快被测试数据灌满了。我们使用了x86和x86-64处理器的SSE指令来装载和存储数据，每次16字节。工作集则与其它测试一样，从1kB增加到512MB，测量的具体对象是每个周期所处理的字节数。

To get a better understanding of the capabilities of the processors we measure the bandwidth available in optimal circumstances. This measurement is especially interesting since different processor versions vary widely. This is why this section is filled with the data of several different machines. The program to measure performance uses the SSE instructions of the x86 and x86-64 processors to load or store 16 bytes at once. The working set is increased from 1kB to 512MB just as in our other tests and it is measured how many bytes per cycle can be loaded or stored.

![](/images/posts/memory/cpumemory.60.png)

图3.24: P4的带宽

图3.24展示了一颗64位Intel Netburst处理器的性能图表。当工作集能够完全放入L1d时，处理器的每个周期可以读取完整的16字节数据，即每个周期执行一条装载指令(moveaps指令，每次移动16字节的数据)。测试程序并不对数据进行任何处理，只是测试读取指令本身。当工作集增大，无法再完全放入L1d时，性能开始急剧下降，跌至每周期6字节。在218工作集处出现的台阶是由于DTLB缓存耗尽，因此需要对每个新页施加额外处理。由于这里的读取是按顺序的，预取机制可以完美地工作，而FSB能以5.3字节/周期的速度传输内容。但预取的数据并不进入L1d。当然，真实世界的程序永远无法达到以上的数字，但我们可以将它们看作一系列实际上的极限值。

Figure 3.24 shows the performance on a 64-bit Intel Netburst processor. For working set sizes which fit into L1d the processor is able to read the full 16 bytes per cycle, i.e., one load instruction is performed per cycle (the movaps instruction moves 16 bytes at once). The test does not do anything with the read data, we test only the read instructions themselves. As soon as the L1d is not sufficient anymore the performance goes down dramatically to less than 6 bytes per cycle. The step at 218 bytes is due to the exhaustion of the DTLB cache which means additional work for each new page. Since the reading is sequential prefetching can predict the accesses perfectly and the FSB can stream the memory content at about 5.3 bytes per cycle for all sizes of the working set. The prefetched data is not propagated into L1d, though. These are of course numbers which will never be achievable in a real program. Think of them as practical limits.

更令人惊讶的是写操作和复制操作的性能。即使是在很小的工作集下，写操作也始终无法达到4字节/周期的速度。这意味着，Intel为Netburst处理器的L1d选择了写通(write-through)模式，所以写入性能受到L2速度的限制。同时，这也意味着，复制测试的性能不会比写入测试差太多(复制测试是将某块内存的数据拷贝到另一块不重叠的内存区)，因为读操作很快，可以与写操作实现部分重叠。最值得关注的地方是，两个操作在工作集无法完全放入L2后出现了严重的性能滑坡，降到了0.5字节/周期！比读操作慢了10倍！显然，如果要提高程序性能，优化这两个操作更为重要。

What is more astonishing than the read performance is the write and copy performance. The write performance, even for small working set sizes, does not ever rise above 4 bytes per cycle. This indicates that, in these Netburst processors, Intel elected to use a Write-Through mode for L1d where the performance is obviously limited by the L2 speed. This also means that the performance of the copy test, which copies from one memory region into a second, non-overlapping memory region, is not significantly worse. The necessary read operations are so much faster and can partially overlap with the write operations. The most noteworthy detail of the write and copy measurements is the low performance once the L2 cache is not sufficient anymore. The performance drops to 0.5 bytes per cycle! That means write operations are by a factor of ten slower than the read operations. This means optimizing those operations is even more important for the performance of the program.

再来看图3.25，它来自同一颗处理器，只是运行双线程，每个线程分别运行在处理器的一个超线程上。

In Figure 3.25 we see the results on the same processor but with two threads running, one pinned to each of the two hyper-threads of the processor.

![](/images/posts/memory/cpumemory.61.png)

图3.25: P4开启两个超线程时的带宽表现

图3.25采用了与图3.24相同的刻度，以方便比较两者的差异。图3.25中的曲线抖动更多，是由于采用双线程的缘故。结果正如我们预期，由于超线程共享着几乎所有资源(仅除寄存器外)，所以每个超线程只能得到一半的缓存和带宽。所以，即使每个线程都要花上许多时间等待内存，从而把执行时间让给另一个线程，也是无济于事——因为另一个线程也同样需要等待。这里恰恰展示了使用超线程时可能出现的最坏情况。

The graph is shown at the same scale as the previous one to illustrate the differences and the curves are a bit jittery simply because of the problem of measuring two concurrent threads. The results are as expected. Since the hyper-threads share all the resources except the registers each thread has only half the cache and bandwidth available. That means even though each thread has to wait a lot and could award the other thread with execution time this does not make any difference since the other thread also has to wait for the memory. This truly shows the worst possible use of hyper-threads.

![](/images/posts/memory/cpumemory.62.png)

图3.26: Core 2的带宽表现

再来看Core 2处理器的情况。看看图3.26和图3.27，再对比下P4的图3.24和3.25，可以看出不小的差异。Core 2是一颗双核处理器，有着共享的L2，容量是P4 L2的4倍。但更大的L2只能解释写操作的性能下降出现较晚的现象。

当然还有更大的不同。可以看到，读操作的性能在整个工作集范围内一直稳定在16字节/周期左右，在220处的下降同样是由于DTLB的耗尽引起。能够达到这么高的数字，不但表明处理器能够预取数据，并且按时完成传输，而且还意味着，预取的数据是被装入L1d的。

Compared to Figures 3.24 and 3.25 the results in Figures 3.26 and 3.27 look quite different for an Intel Core 2 processor. This is a dual-core processor with shared L2 which is four times as big as the L2 on the P4 machine. This only explains the delayed drop-off of the write and copy performance, though.
There are much bigger differences. The read performance throughout the working set range hovers around the optimal 16 bytes per cycle. The drop-off in the read performance after 220 bytes is again due to the working set being too big for the DTLB. Achieving these high numbers means the processor is not only able to prefetch the data and transport the data in time. It also means the data is prefetched into L1d.

写/复制操作的性能与P4相比，也有很大差异。处理器没有采用写通策略，写入的数据留在L1d中，只在必要时才逐出。这使得写操作的速度可以逼近16字节/周期。一旦工作集超过L1d，性能即飞速下降。由于Core 2读操作的性能非常好，所以两者的差值显得特别大。当工作集超过L2时，两者的差值甚至超过20倍！但这并不表示Core 2的性能不好，相反，Core 2永远都比Netburst强。

The write and copy performance is dramatically different, too. The processor does not have a Write-Through policy; written data is stored in L1d and only evicted when necessary. This allows for write speeds close to the optimal 16 bytes per cycle. Once L1d is not sufficient anymore the performance drops significantly. As with the Netburst processor, the write performance is significantly lower. Due to the high read performance the difference is even higher here. In fact, when even the L2 is not sufficient anymore the speed difference increases to a factor of 20! This does not mean the Core 2 processors perform poorly. To the contrary, their performance is always better than the Netburst core's.

![](/images/posts/memory/cpumemory.63.png)

图3.27: Core 2运行双线程时的带宽表现

在图3.27中，启动双线程，各自运行在Core 2的一个核心上。它们访问相同的内存，但不需要完美同步。从结果上看，读操作的性能与单线程并无区别，只是多了一些多线程情况下常见的抖动。

In Figure 3.27, the test runs two threads, one on each of the two cores of the Core 2 processor. Both threads access the same memory, not necessarily perfectly in sync, though. The results for the read performance are not different from the single-threaded case. A few more jitters are visible which is to be expected in any multi-threaded test case.

有趣的地方来了——当工作集小于L1d时，写操作与复制操作的性能很差，就好像数据需要从内存读取一样。两个线程彼此竞争着同一个内存位置，于是不得不频频发送RFO消息。问题的根源在于，虽然两个核心共享着L2，但无法以L2的速度处理RFO请求。而当工作集超过L1d后，性能出现了迅猛提升。这是因为，由于L1d容量不足，于是将被修改的条目刷新到共享的L2。由于L1d的未命中可以由L2满足，只有那些尚未刷新的数据才需要RFO，所以出现了这样的现象。这也是这些工作集情况下速度下降一半的原因。这种渐进式的行为也与我们期待的一致: 由于每个核心共享着同一条FSB，每个核心只能得到一半的FSB带宽，因此对于较大的工作集来说，每个线程的性能大致相当于单线程时的一半。

The interesting point is the write and copy performance for working set sizes which would fit into L1d. As can be seen in the figure, the performance is the same as if the data had to be read from the main memory. Both threads compete for the same memory location and RFO messages for the cache lines have to be sent. The problematic point is that these requests are not handled at the speed of the L2 cache, even though both cores share the cache. Once the L1d cache is not sufficient anymore modified entries are flushed from each core's L1d into the shared L2. At that point the performance increases significantly since now the L1d misses are satisfied by the L2 cache and RFO messages are only needed when the data has not yet been flushed. This is why we see a 50% reduction in speed for these sizes of the working set. The asymptotic behavior is as expected: since both cores share the same FSB each core gets half the FSB bandwidth which means for large working sets each thread's performance is about half that of the single threaded case.

由于同一个厂商的不同处理器之间都存在着巨大差异，我们没有理由不去研究一下其它厂商处理器的性能。图3.28展示了AMD家族10h Opteron处理器的性能。这颗处理器有64kB的L1d、512kB的L2和2MB的L3，其中L3缓存由所有核心所共享。

Because there are significant differences between the processor versions of one vendor it is certainly worthwhile looking at the performance of other vendors' processors, too. Figure 3.28 shows the performance of an AMD family 10h Opteron processor. This processor has 64kB L1d, 512kB L2, and 2MB of L3. The L3 cache is shared between all cores of the processor. The results of the performance test can be seen in Figure 3.28.

![](/images/posts/memory/cpumemory.64.png)

图3.28: AMD家族10h Opteron的带宽表现

大家首先应该会注意到，在L1d缓存足够的情况下，这个处理器每个周期能处理两条指令。读操作的性能超过了32字节/周期，写操作也达到了18.7字节/周期。但是，不久，读操作的曲线就急速下降，跌到2.3字节/周期，非常差。处理器在这个测试中并没有预取数据，或者说，没有有效地预取数据。

The first detail one notices about the numbers is that the processor is capable of handling two instructions per cycle if the L1d cache is sufficient. The read performance exceeds 32 bytes per cycle and even the write performance is, with 18.7 bytes per cycle, high. The read curve flattens quickly, though, and is, with 2.3 bytes per cycle, pretty low. The processor for this test does not prefetch any data, at least not efficiently.

另一方面，写操作的曲线随几级缓存的容量而流转。在L1d阶段达到最高性能，随后在L2阶段下降到6字节/周期，在L3阶段进一步下降到2.8字节/周期，最后，在工作集超过L3后，降到0.5字节/周期。它在L1d阶段超过了Core 2，在L2阶段基本相当(Core 2的L2更大一些)，在L3及主存阶段比Core 2慢。

复制的性能既无法超越读操作的性能，也无法超越写操作的性能。因此，它的曲线先是被读性能压制，随后又被写性能压制。

The write curve on the other hand performs according to the sizes of the various caches. The peak performance is achieved for the full size of the L1d, going down to 6 bytes per cycle for L2, to 2.8 bytes per cycle for L3, and finally .5 bytes per cycle if not even L3 can hold all the data. The performance for the L1d cache exceeds that of the (older) Core 2 processor, the L2 access is equally fast (with the Core 2 having a larger cache), and the L3 and main memory access is slower.

The copy performance cannot be better than either the read or write performance. This is why we see the curve initially dominated by the read performance and later by the write performance.

图3.29显示的是Opteron处理器在多线程时的性能表现。

The multi-thread performance of the Opteron processor is shown in Figure 3.29.

![](/images/posts/memory/cpumemory.65.png)

图3.29: AMD Fam 10h在双线程时的带宽表现

读操作的性能没有受到很大的影响。每个线程的L1d和L2表现与单线程下相仿，L3的预取也依然表现不佳。两个线程并没有过渡争抢L3。问题比较大的是写操作的性能。两个线程共享的所有数据都需要经过L3，而这种共享看起来却效率很差。即使是在L3足够容纳整个工作集的情况下，所需要的开销仍然远高于L3的访问时间。再来看图3.27，可以发现，在一定的工作集范围内，Core 2处理器能以共享的L2缓存的速度进行处理。而Opteron处理器只能在很小的一个范围内实现相似的性能，而且，它仅仅只能达到L3的速度，无法与Core 2的L2相比。

The read performance is largely unaffected. Each thread's L1d and L2 works as before and the L3 cache is in this case not prefetched very well either. The two threads do not unduly stress the L3 for their purpose. The big problem in this test is the write performance. All data the threads share has to go through the L3 cache. This sharing seems to be quite inefficient since even if the L3 cache size is sufficient to hold the entire working set the cost is significantly higher than an L3 access. Comparing this graph with Figure 3.27 we see that the two threads of the Core 2 processor operate at the speed of the shared L2 cache for the appropriate range of working set sizes. This level of performance is achieved for the Opteron processor only for a very small range of the working set sizes and even here it approaches only the speed of the L3 which is slower than the Core 2's L2.

3.5.2 关键字加载

内存以比缓存线还小的块从主存储器向缓存传送。如今64位可一次性传送，缓存线的大小为64或128比特。这意味着每个缓存线需要8或16次传送。

DRAM芯片可以以触发模式传送这些64位的块。这使得不需要内存控制器的进一步指令和可能伴随的延迟，就可以将缓存线充满。如果处理器预取了缓存，这有可能是最好的操作方式。

Memory is transferred from the main memory into the caches in blocks which are smaller than the cache line size. Today 64 bits are transferred at once and the cache line size is 64 or 128 bytes. This means 8 or 16 transfers per cache line are needed.

The DRAM chips can transfer those 64-bit blocks in burst mode. This can fill the cache line without any further commands from the memory controller and the possibly associated delays. If the processor prefetches cache lines this is probably the best way to operate.

如果程序在访问数据或指令缓存时没有命中(这可能是强制性未命中或容量性未命中，前者是由于数据第一次被使用，后者是由于容量限制而将缓存线逐出)，情况就不一样了。程序需要的并不总是缓存线中的第一个字，而数据块的到达是有先后顺序的，即使是在突发模式和双倍传输率下，也会有明显的时间差，一半在4个CPU周期以上。举例来说，如果程序需要缓存线中的第8个字，那么在首字抵达后它还需要额外等待30个周期以上。

当然，这样的等待并不是必需的。事实上，内存控制器可以按不同顺序去请求缓存线中的字。当处理器告诉它，程序需要缓存中具体某个字，即「关键字(critical word)」时，内存控制器就会先请求这个字。一旦请求的字抵达，虽然缓存线的剩余部分还在传输中，缓存的状态还没有达成一致，但程序已经可以继续运行。这种技术叫做关键字优先及较早重启(Critical Word First & Early Restart)。

If a program's cache access of the data or instruction caches misses (that means, it is a compulsory cache miss, because the data is used for the first time, or a capacity cache miss, because the limited cache size requires eviction of the cache line) the situation is different. The word inside the cache line which is required for the program to continue might not be the first word in the cache line. Even in burst mode and with double data rate transfer the individual 64-bit blocks arrive at noticeably different times. Each block arrives 4 CPU cycles or more later than the previous one. If the word the program needs to continue is the eighth of the cache line, the program has to wait an additional 30 cycles or more after the first word arrives.

Things do not necessarily have to be like this. The memory controller is free to request the words of the cache line in a different order. The processor can communicate which word the program is waiting on, the critical word, and the memory controller can request this word first. Once the word arrives the program can continue while the rest of the cache line arrives and the cache is not yet in a consistent state. This technique is called Critical Word First & Early Restart.

现在的处理器都已经实现了这一技术，但有时无法运用。比如，预取操作的时候，并不知道哪个是关键字。如果在预取的中途请求某条缓存线，处理器只能等待，并不能更改请求的顺序。

Processors nowadays implement this technique but there are situations when that is not possible. If the processor prefetches data the critical word is not known. Should the processor request the cache line during the time the prefetch operation is in flight it will have to wait until the critical word arrives without being able to influence the order.

![](/images/posts/memory/cpumemory.72.png)

图3.30: 关键字位于缓存线尾时的表现

在关键字优先技术生效的情况下，关键字的位置也会影响结果。图3.30展示了下一个测试的结果，图中表示的是关键字分别在线首和线尾时的性能对比情况。元素大小为64字节，等于缓存线的长度。图中的噪声比较多，但仍然可以看出，当工作集超过L2后，关键字处于线尾情况下的性能要比线首情况下低0.7%左右。而顺序访问时受到的影响更大一些。这与我们前面提到的预取下条线时可能遇到的问题是相符的。

Even with these optimizations in place the position of the critical word on a cache line matters. Figure 3.30 shows the Follow test for sequential and random access. Shown is the slowdown of running the test with the pointer which is chased in the first word versus the case when the pointer is in the last word. The element size is 64 bytes, corresponding the cache line size. The numbers are quite noisy but it can be seen that, as soon as the L2 is not sufficient to hold the working set size, the performance of the case where the critical word is at the end is about 0.7% slower. The sequential access appears to be affected a bit more. This would be consistent with the aforementioned problem when prefetching the next cache line.

## 3.5.3 缓存设定

缓存放置的位置与超线程，内核和处理器之间的关系，不在程序员的控制范围之内。但是程序员可以决定线程执行的位置，接着高速缓存与使用的CPU的关系将变得非常重要。

这里我们将不会深入（探讨）什么时候选择什么样的内核以运行线程的细节。我们仅仅描述了在设置关联线程的时候，程序员需要考虑的系统结构的细节。

Where the caches are placed in relationship to the hyper-threads, cores, and processors is not under control of the programmer. But programmers can determine where the threads are executed and then it becomes important how the caches relate to the used CPUs.

Here we will not go into details of when to select what cores to run the threads. We will only describe architecture details which the programmer has to take into account when setting the affinity of the threads.

超线程，通过定义，共享除去寄存器集以外的所有数据。包括 L1 缓存。这里没有什么可以多说的。多核处理器的独立核心带来了一些乐趣。每个核心都至少拥有自己的 L1 缓存。除此之外，下面列出了一些不同的特性：

* 早期多核心处理器有独立的 L2 缓存且没有更高层级的缓存。
* 之后英特尔的双核心处理器模型拥有共享的L2 缓存。对四核处理器，则分对拥有独立的L2 缓存，且没有更高层级的缓存。
* AMD 家族的 10h 处理器有独立的 L2 缓存以及一个统一的L3 缓存。

Hyper-threads, by definition share everything but the register set. This includes the L1 caches. There is not much more to say here. The fun starts with the individual cores of a processor. Each core has at least its own L1 caches. Aside from this there are today not many details in common:

Early multi-core processors had separate L2 caches and no higher caches.
Later Intel models had shared L2 caches for dual-core processors. For quad-core processors we have to deal with separate L2 caches for each pair of two cores. There are no higher level caches.
AMD's family 10h processors have separate L2 caches and a unified L3 cache.

关于各种处理器模型的优点，已经在它们各自的宣传手册里写得够多了。在每个核心的工作集互不重叠的情况下，独立的L2拥有一定的优势，单线程的程序可以表现优良。考虑到目前实际环境中仍然存在大量类似的情况，这种方法的表现并不会太差。不过，不管怎样，我们总会遇到工作集重叠的情况。如果每个缓存都保存着某些通用运行库的常用部分，那么很显然是一种浪费。

如果像Intel的双核处理器那样，共享除L1外的所有缓存，则会有一个很大的优点。如果两个核心的工作集重叠的部分较多，那么综合起来的可用缓存容量会变大，从而允许容纳更大的工作集而不导致性能的下降。如果两者的工作集并不重叠，那么则是由Intel的高级智能缓存管理(Advanced Smart Cache management)发挥功用，防止其中一个核心垄断整个缓存。

A lot has been written in the propaganda material of the processor vendors about the advantage of their respective models. Separate L2 caches have an advantage if the working sets handled by the cores do not overlap. This works well for single-threaded programs. Since this is still often the reality today this approach does not perform too badly. But there is always some overlap. The caches all contain the most actively used parts of the common runtime libraries which means some cache space is wasted.

Completely sharing all caches beside L1 as Intel's dual-core processors do can have a big advantage. If the working set of the threads working on the two cores overlaps significantly the total available cache memory is increased and working sets can be bigger without performance degradation. If the working sets do not overlap Intel's Advanced Smart Cache management is supposed to prevent any one core from monopolizing the entire cache.

即使每个核心只使用一半的缓存，也会有一些摩擦。缓存需要不断衡量每个核心的用量，在进行逐出操作时可能会作出一些比较差的决定。我们来看另一个测试程序的结果。

If both cores use about half the cache for their respective working sets there is some friction, though. The cache constantly has to weigh the two cores' cache use and the evictions performed as part of this rebalancing might be chosen poorly. To see the problems we look at the results of yet another test program.

![](/images/posts/memory/cpumemory.74.png)

这次，测试程序两个进程，第一个进程不断用SSE指令读/写2MB的内存数据块，选择2MB，是因为它正好是Core 2处理器L2缓存的一半，第二个进程则是读/写大小变化的内存区域，我们把这两个进程分别固定在处理器的两个核心上。图中显示的是每个周期读/写的字节数，共有4条曲线，分别表示不同的读写搭配情况。例如，标记为读/写(read/write)的曲线代表的是后台进程进行写操作(固定2MB工作集)，而被测量进程进行读操作(工作集从小到大)。

The test program has one process constantly reading or writing, using SSE instructions, a 2MB block of memory. 2MB was chosen because this is half the size of the L2 cache of this Core 2 processor. The process is pinned to one core while a second process is pinned to the other core. This second process reads and writes a memory region of variable size. The graph shows the number of bytes per cycle which are read or written. Four different graphs are shown, one for each combination of the processes reading and writing. The read/write graph is for the background process, which always uses a 2MB working set to write, and the measured process with variable working set to read.

图中最有趣的是220到223之间的部分。如果两个核心的L2是完全独立的，那么所有4种情况下的性能下降均应发生在221到222之间，也就是L2缓存耗尽的时候。但从图上来看，实际情况并不是这样，特别是背景进程进行写操作时尤为明显。当工作集达到1MB(220)时，性能即出现恶化，两个进程并没有共享内存，因此并不会产生RFO消息。所以，完全是缓存逐出操作引起的问题。目前这种智能的缓存处理机制有一个问题，每个核心能实际用到的缓存更接近1MB，而不是理论上的2MB。如果未来的处理器仍然保留这种多核共享缓存模式的话，我们唯有希望厂商会把这个问题解决掉。

The interesting part of the graph is the part between 220 and 223 bytes. If the L2 cache of the two cores were completely separate we could expect that the performance of all four tests would drop between 221 and 222 bytes, that means, once the L2 cache is exhausted. As we can see in Figure 3.31 this is not the case. For the cases where the background process is writing this is most visible. The performance starts to deteriorate before the working set size reaches 1MB. The two processes do not share memory and therefore the processes do not cause RFO messages to be generated. These are pure cache eviction problems. The smart cache handling has its problems with the effect that the experienced cache size per core is closer to 1MB than the 2MB per core which are available. One can only hope that, if caches shared between cores remain a feature of upcoming processors, the algorithm used for the smart cache handling will be fixed.

推出拥有双L2缓存的4核处理器仅仅只是一种临时措施，是开发更高级缓存之前的替代方案。与独立插槽及双核处理器相比，这种设计并没有带来多少性能提升。两个核心是通过同一条总线(被外界看作FSB)进行通信，并没有什么特别快的数据交换通道。

未来，针对多核处理器的缓存将会包含更多层次。AMD的10h家族是一个开始，至于会不会有更低级共享缓存的出现，还需要我们拭目以待。我们有必要引入更多级别的缓存，因为频繁使用的高速缓存不可能被许多核心共用，否则会对性能造成很大的影响。我们也需要更大的高关联性缓存，它们的数量、容量和关联性都应该随着共享核心数的增长而增长。巨大的L3和适度的L2应该是一种比较合理的选择。L3虽然速度较慢，但也较少使用。

对于程序员来说，不同的缓存设计就意味着调度决策时的复杂性。为了达到最高的性能，我们必须掌握工作负载的情况，必须了解机器架构的细节。好在我们在判断机器架构时还是有一些支援力量的，我们会在后面的章节介绍这些接口。

Having a quad-core processor with two L2 caches was just a stop-gap solution before higher-level caches could be introduced. This design provides no significant performance advantage over separate sockets and dual-core processors. The two cores communicate via the same bus which is, at the outside, visible as the FSB. There is no special fast-track data exchange.

The future of cache design for multi-core processors will lie in more layers. AMD's 10h family of processors make the start. Whether we will continue to see lower level caches be shared by a subset of the cores of a processor remains to be seen. The extra levels of cache are necessary since the high-speed and frequently used caches cannot be shared among many cores. The performance would be impacted. It would also require very large caches with high associativity. Both numbers, the cache size and the associativity, must scale with the number of cores sharing the cache. Using a large L3 cache and reasonably-sized L2 caches is a reasonable trade-off. The L3 cache is slower but it is ideally not as frequently used as the L2 cache.

For programmers all these different designs mean complexity when making scheduling decisions. One has to know the workloads and the details of the machine architecture to achieve the best performance. Fortunately we have support to determine the machine architecture. The interfaces will be introduced in later sections.

## 3.5.4 FSB的影响

FSB在性能中扮演了核心角色。缓存数据的存取速度受制于内存通道的速度。我们做一个测试，在两台机器上分别跑同一个程序，这两台机器除了内存模块的速度有所差异，其它完全相同。图3.32展示了Addnext0测试(将下一个元素的pad[0]加到当前元素的pad[0]上)在这两台机器上的结果(NPAD=7，64位机器)。两台机器都采用Core 2处理器，一台使用667MHz的DDR2内存，另一台使用800MHz的DDR2内存(比前一台增长20%)。

The FSB plays a central role in the performance of the machine. Cache content can only be stored and loaded as quickly as the connection to the memory allows. We can show how much so by running a program on two machines which only differ in the speed of their memory modules. Figure 3.32 shows the results of the Addnext0 test (adding the content of the next elements pad[0] element to the own pad[0] element) for NPAD=7 on a 64-bit machine. Both machines have Intel Core 2 processors, the first uses 667MHz DDR2 modules, the second 800MHz modules (a 20% increase).

![](/images/posts/memory/cpumemory.27.png)

图3.32: FSB速度的影响

图上的数字表明，当工作集大到对FSB造成压力的程度时，高速FSB确实会带来巨大的优势。在我们的测试中，性能的提升达到了18.5%，接近理论上的极限。而当工作集比较小，可以完全纳入缓存时，FSB的作用并不大。当然，这里我们只测试了一个程序的情况，在实际环境中，系统往往运行多个进程，工作集是很容易超过缓存容量的。

The numbers show that, when the FSB is really stressed for large working set sizes, we indeed see a large benefit. The maximum performance increase measured in this test is 18.2%, close to the theoretical maximum. What this shows is that a faster FSB indeed can pay off big time. It is not critical when the working set fits into the caches (and these processors have a 4MB L2). It must be kept in mind that we are measuring one program here. The working set of a system comprises the memory needed by all concurrently running processes. This way it is easily possible to exceed 4MB memory or more with much smaller programs.

如今，一些英特尔的处理器，支持前端总线(FSB)的速度高达1,333 MHz，这意味着速度有另外60％的提升。将来还会出现更高的速度。速度是很重要的，工作集会更大，快速的RAM和高FSB速度的内存肯定是值得投资的。我们必须小心使用它，因为即使处理器可以支持更高的前端总线速度，但是主板的北桥芯片可能不会。使用时，检查它的规范是至关重要的。

Today some of Intel's processors support FSB speeds up to 1,333MHz which would mean another 60% increase. The future is going to see even higher speeds. If speed is important and the working set sizes are larger, fast RAM and high FSB speeds are certainly worth the money. One has to be careful, though, since even though the processor might support higher FSB speeds the motherboard/Northbridge might not. It is critical to check the specifications.

# 参考
[^1]: [What-Every-Programmer-Should-Know-About-Memory](/images/posts/memory/What-Every-Programmer-Should-Know-About-Memory.pdf)
[^2]: [每个程序员都应该了解的 CPU 高速缓存](https://www.oschina.net/translate/what-every-programmer-should-know-about-cpu-cache-part2)
[^3]: [Memory part 2: CPU caches](https://lwn.net/Articles/252125/)

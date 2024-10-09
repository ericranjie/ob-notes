# [蜗窝科技](http://www.wowotech.net/)

### 慢下来，享受技术。

[![](http://www.wowotech.net/content/uploadfile/201401/top-1389777175.jpg)](http://www.wowotech.net/)

- [博客](http://www.wowotech.net/)
- [项目](http://www.wowotech.net/sort/project)
- [关于蜗窝](http://www.wowotech.net/about.html)
- [联系我们](http://www.wowotech.net/contact_us.html)
- [支持与合作](http://www.wowotech.net/support_us.html)
- [登录](http://www.wowotech.net/admin)

﻿

## 

作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2016-8-12 18:50 分类：[内存管理](http://www.wowotech.net/sort/memory_management)

一、前言

本文主要分析linux-4.4.6/arch/arm64/include/asm目录下的若干和地址翻译相关的头文件（例如page.h、pgtable.h、pgtable-hwdef.h、pgtable-prot.h等文件）中的各种宏定义以及相关的ARM64硬件知识。硬肯ARM ARM文档有时候太费劲，结合linux源代码会让学习变得简单一些。

二、ARM64地址翻译概述

ARM64定义了若干个translation regimes，下面的每一行地址翻译过程都是一个translation regimes：

Secure EL3 VA         -----------    Secure EL3 stage 1     -----------> PA, Secure or Non-secure

Secure EL1&EL0 VA ----------- Secure EL1&EL0 stage 1 -----------> PA, Secure or Non-secure

Non-Secure EL2 VA  -----------  Non-Secure EL2 stage 1 -----------> PA, Non-secure only

Non-Secure EL1&0 VA  ------ stage 1 -----> IPA ------ stage 2 -----> PA, Non-secure only

translation regime可能包括一个stage，也可能包括两个stage。每个exception level都有自己的独立一套的地址翻译的机制，使用不同的页表基地址寄存器和控制寄存器。地址翻译可以细分到stage，大部分的EL包括一个stage的地址翻译过程， Non-Secure EL1&0包括了2个stage的地址翻译过程。每个stage都有自己独立的一系列Translation tables，每个stage都能独立的enable或者disable。每个stage都是将输入地址（可能是虚拟地址或者IPA）翻译成输出地址（可能是物理地址或者IPA）。

对于一个stage的地址翻译而言，也不是一蹴而就的，而是分成若干个level，每个level都是根据内存中的Translation table（我们将PGD，PUD，PMD和Page table统称Translation table）将整个VA中的部分地址翻译出来，一个简单的示意图如下（来自ULK3）：

![](http://www.wowotech.net/content/uploadfile/201608/f0651470999247.gif)

当然，上图是基于x86，对于ARM64，cr3寄存器应该是TTBRx寄存器，PGD也就是Level 0 translation table、PUD相当于Level 1 translation table、PMD对应Level 2 translation table、Page Table对应Level 3 translation table。

Linear Address是X86特有的地址类型，对于ARM64，Linear Address应该被修改为VA或者IPA，具体ARM64地址翻译单元要处理的地址类型如下：

（1）虚拟地址（Virtual Address，简称VA）。MMU的存在可以让各个进程生活在自己的虚拟地址空间中，因此，在进程在进行数据操作的时候（内存操作指令），在进程逻辑跳转的时候（指令地址）都是使用的虚拟地址。更具体的说，进程正文段程序位于虚拟地址之上，进程的数据段、stack和heap都是位于虚拟地址之上，因此CPU的PC、LR、SP、ELR等都是指向了虚拟地址。

（2）Intermediate Physical Address，简称IPA。IPA是计算机虚拟化之后引入的一个概念，传统的计算机系统都是虚拟地址直接映射到物理地址，但是，对于支持虚拟化的计算机系统，每一个guest OS不能直接将虚拟地址映射到整个系统的物理地址，而是映射到一个受限的物理地址空间上，而且映射是随着VM的创建而创建，一旦VM销毁，这些物理内存还需要返回给Hypervisor。因此在支持虚拟化的计算机系统中，guest OS负责将VA映射到IPA，而Hypervisor负责将IPA映射到具体的物理地址上去。

（3）物理地址（Physical Address，简称PA）。物理地址是实实在在的地址，体现在实际的电路中。如果CPU和memory chip是通过bus连接，那么物理地址就是bus上的地址信号线。实际的物理地址空间被分成了两个世界，一个是Normal world，另外一个是Secure world，虽然物理地址一样，但是却通往不同的memory cell。

三、如何确定page table level？

定义如下：

> #define ARM64_HW_PGTABLE_LEVELS(va_bits) (((va_bits) - 4) / (PAGE_SHIFT - 3))

从这个宏定义可以看出，两个输入参数就可以确定page table level。其中一个参数就是PAGE_SHIFT（也就是page size了），另外一个va_bits，也就是虚拟地址的bit数目。为何公式这么怪异呢？3和4又是什么鬼？为了回答这个问题，我们先看看一些具体的例子好了。我们还是先用最经典的4K page为例来描述。一旦确定了4K的page size，那么页偏移所占用的bit数就确定了，即\[11:0\]这12个地位bit用来确定页内偏移。此外，一个Translation table往往占用一个page的size，由于ARM64中，一个page table中的描述符是8个字节，因此4K中有512个描述符，因此各级index（指向PGD/PUD/PMD/PT）占用的bit数都是9个，综上所述，48bit的虚拟地址在4K page的情况下，4级映射的地址分配如下：

|   |   |   |   |   |   |
|---|---|---|---|---|---|
|页表基地址|Global DIR|Upper DIR|Middle DIR|Page Table|Offset|
|16|9|9|9|9|12|

高16bit用来确定选用TTBR0或者TTBR1的Translation tables。中间的bit被平均分成4个9-bit的段，分别用来索引到PGD/PUD/PMD/PT中的具体的描述符。

如果page size是64K的话，那么offset需要16个bit，一个Translation table占有64K，每个描述符8B，总共有8K个entry，需要13个bit做为index，这时候使用3级映射，地址分配如下：

|   |   |   |   |   |   |
|---|---|---|---|---|---|
|页表基地址|Global DIR|Upper DIR|Middle DIR|Page Table|Offset|
|16|N/A|6|13|13|16|

这时候，由于是3级映射，PGD和PUD是重合的。此外需要说明的是PGD（或者PUD）中只有64个entry，其他的都是无效的。

OK，搞定了4K和64K page size之后，我们来看看16K size的情况。16K的页对应的offset的bit数目是14个bit，而index需要的bit是11个bit（即每个16K的Translation table中有2K个entry）。这时候使用4级映射，地址分配如下：

|   |   |   |   |   |   |
|---|---|---|---|---|---|
|页表基地址|Global DIR|Upper DIR|Middle DIR|Page Table|Offset|
|16|1|11|11|11|14|

因此，经过上面的三个例子之后，我们可以回头看看ARM64_HW_PGTABLE_LEVELS这个宏定义了，该宏定义中的3其实是和描述符的size相关的，对于ARM64，描述符的size是8个字节，因此PAGE_SHIFT - 3实际上描述了中间level的translation table所占据的虚拟地址的bit数目。也就是说，从虚拟地址的LSB开始，先切掉 PAGE_SHIFT 个bit做为页内偏移，然后按照PAGE_SHIFT - 3来切其他的中间的各个中间level的translation table所占据的虚拟地址的bit数目。因此，page table levels 应该等于 DIV_ROUND_UP((va_bits - PAGE_SHIFT), (PAGE_SHIFT - 3))，也就是((((va_bits) - PAGE_SHIFT) + (PAGE_SHIFT - 3) - 1) / (PAGE_SHIFT - 3))，简单的数学运算之后就得到了(((va_bits) - 4) / (PAGE_SHIFT - 3))。

在linux kernel中，具体支持的配置请参考下面的表格：

|   |   |   |
|---|---|---|
|虚拟地址的数目|page size|page table level|
|48|64K|3|
|48|4K or 16K|4|
|47|16K|3|
|47|64K or 16K|无效|
|42|64K|2|
|42|16K or 4K|无效|
|39|4K|3|
|39|64K or 16K|无效|
|36|16K|2|
|36|64K or 4K|无效|

四、如何定义各个level的地址bit偏移

在linux paging model中，PGD_SHIFT、PUD_SHIFT和PMD_SHIFT用来定义定义各个Translation table在虚拟地址上的偏移量，这些定义都借用了一个ARM64_HW_PGTABLE_LEVEL_SHIFT的宏定义，如下：

> #define ARM64_HW_PGTABLE_LEVEL_SHIFT(n)    ((PAGE_SHIFT - 3) * (4 - (n)) + 3)

我们用下面的表格来表述虚拟地址偏移的定义（4 level page table）：

|   |   |   |   |   |
|---|---|---|---|---|
|Global Dir bits（L0）|Upper Dir bits（L1）|Middle Dir bits（L2）|table bits（L3）|offset bits|

|   |   |
|---|---|
||\<----page_shift----->|

|   |   |
|---|---|
||\<-------------------pmd_shift--------------------------->|

|   |   |
|---|---|
||\<----------------------------------------pud_shift----------------------------------------->|

|   |   |
|---|---|
||\<-------------------------------------------------------pgd_shift------------------------------------------------------------->|

从上面的表格可以看出，如果是4级映射都在，对于L0而言，该level的SHIFT值应该是3 x (PAGE_SHIFT - 3) + PAGE_SHIFT。对于L1而言，该level的SHIFT值应该是2 x (PAGE_SHIFT - 3) + PAGE_SHIFT。以此类推L2和L3的。假设用n来表示level，那么Ln的SHIFT值应该是((4 - n) - 1) * (PAGE_SHIFT - 3) + PAGE_SHIFT，经过简单的数学运算之后可以得到(4 - n) * (PAGE_SHIFT - 3) + 3。当然，如果page table的level没有那么多，那么情况也类似，只不过少了PUD和PMD而已。我们用下面的表格来表述3 level page table中虚拟地址偏移的定义：

|   |   |   |   |   |
|---|---|---|---|---|
||Global Dir bits（L1）|Middle Dir bits（L2）|table bits（L3）|offset bits|

|   |   |
|---|---|
||\<----page_shift----->|

|   |   |
|---|---|
||\<-------------------pmd_shift--------------------------->|

|   |   |
|---|---|
||\<----------------------------------------pud_shift----------------------------------------->|

|   |   |
|---|---|
||\<----------------------------------------pgd_shift---------------------------------------->|

2 level page table的情况请自行补脑。

五、Translation table中的一个描述符对应多大的地址空间呢？

对于Level 3 translation table而言（也就是linux paging model中的page table），一个描述符就指向一个page，因此相关定义如下：

> #ifdef CONFIG_ARM64_64K_PAGES\
> #define PAGE_SHIFT        16\
> #elif defined(CONFIG_ARM64_16K_PAGES)\
> #define PAGE_SHIFT        14\
> #else\
> #define PAGE_SHIFT        12\
> #endif\
> #define PAGE_SIZE        (\_AC(1, UL) \<\< PAGE_SHIFT)\
> #define PAGE_MASK        (~(PAGE_SIZE-1))

PAGE_SIZE定义了一个page的大小，可能是4K，16K或者64K。PAGE_MASK是地址掩码，可以屏蔽掉虚拟地址offset域的那些地址bits。

对于Level 2 translation table而言（也就是PMD），当page level大于2的时候（也就是说page level是3或者4的时候），定义如下：

> #if CONFIG_PGTABLE_LEVELS > 2\
> #define PMD_SHIFT        ARM64_HW_PGTABLE_LEVEL_SHIFT(2)\
> #define PMD_SIZE        (\_AC(1, UL) \<\< PMD_SHIFT)\
> #define PMD_MASK        (~(PMD_SIZE-1))\
> #define PTRS_PER_PMD        PTRS_PER_PTE\
> #endif

PMD_SHIFT包括了虚拟地址中的offset域和table域，PMD_SIZE定义了PMD中的一个描述能mapping的地址空间的size。PMD_MASK是地址掩码，可以mask掉虚拟地址level 3 以及offset域的那些地址bits。

page level小于等于2的时候（也就是说page level是1或者2的时候，当然，目前ARM64中不支持page level等于1的情况），相关定义如下：

> #define PMD_SHIFT    PUD_SHIFT\
> #define PTRS_PER_PMD    1\
> #define PMD_SIZE      (1UL \<\< PMD_SHIFT)\
> #define PMD_MASK      (~(PMD_SIZE-1))

也就是说，PMD_SHIFT跟随PUD_SHIFT（当然，这时候其实PUD也不存在，是说L2变成了PGD）。对于Level 1 translation table而言（也就是PUD），当page level大于3的时候（也就是说page level是4的时候），定义如下：

> #if CONFIG_PGTABLE_LEVELS > 3\
> #define PUD_SHIFT        ARM64_HW_PGTABLE_LEVEL_SHIFT(1)\
> #define PUD_SIZE        (\_AC(1, UL) \<\< PUD_SHIFT)\
> #define PUD_MASK        (~(PUD_SIZE-1))\
> #define PTRS_PER_PUD        PTRS_PER_PTE\
> #endif

当page level小于等于3的时候（也就是说page level是3或者2的时候），相关定义如下：

> #define PUD_SHIFT    PGDIR_SHIFT\
> #define PTRS_PER_PUD    1\
> #define PUD_SIZE      (1UL \<\< PUD_SHIFT)\
> #define PUD_MASK      (~(PUD_SIZE-1))

也就是说，PUD_SHIFT跟随PGDIR_SHIFT。而PGD相关的定义如下：

> #define PGDIR_SHIFT      ARM64_HW_PGTABLE_LEVEL_SHIFT(4 - CONFIG_PGTABLE_LEVELS)\
> #define PGDIR_SIZE        (\_AC(1, UL) \<\< PGDIR_SHIFT)\
> #define PGDIR_MASK        (~(PGDIR_SIZE-1))\
> #define PTRS_PER_PGD        (1 \<\< (VA_BITS - PGDIR_SHIFT))

PGDIR_SHIFT其实可以确定最高层那个level的Translation table（TTBR指向的那个Translation table）中的一个描述符能够mapping多大size的地址空间（参考PGDIR_SIZE的定义）。具体这个top level的Translation table是那个呢？根据硬件手册的描述，可能是L0，可能是L1，也可能是L2，和page level的配置有关。具体的关系是4 - CONFIG_PGTABLE_LEVELS，也就是说，如果CONFIG_PGTABLE_LEVELS等于4，那么top level的Translation table就是L0，如果CONFIG_PGTABLE_LEVELS等于3，那么top level的Translation table就是L1。

OK，我们一起来整理一下ARM64上的具体映射关系，如下图所示：

![](http://www.wowotech.net/content/uploadfile/201608/7fc21470999301.gif)

当page level等于4的时候，映射关系是PGD（L0）--->PUD（L1）--->PMD（L2）--->Page table（L3）--->page。当page level等于3的时候，没有PUD，这时候映射关系是PGD（L1）--->PMD（L2）--->Page table（L3）--->page。当page level等于2的时候，没有PUD和PMD，映射关系是PGD（L2）--->Page table（L3）--->page。

最后，闲聊一下PTRS_PER_PTE、PTRS_PER_PMD、PTRS_PER_PUD和PTRS_PER_PGD。PTRS_PER_PGD定义了PGD中有多少个描述符，当然，有多少个描述符是和Global Dir bits的长度相关，也就是VA_BITS - PGDIR_SHIFT。而其他的PTRS_PER_XXX都是描述相应translation table中描述符的个数，对于ARM64，PTRS_PER_PMD和PTRS_PER_PUD（如果存在的话）中的描述符个数基本是是固定的，和PTRS_PER_PTE保持一致，定义如下：

> #define PTRS_PER_PTE        (1 \<\< (PAGE_SHIFT - 3))

六、Translation table相关的宏定义

在内核中，pte_t，pmd_t，pud_t和pgd_t分别描述了page table（L3），PMD（L2），PUD（L1）和PGD（L0）中一个描述符的数据结构，对于ARM64而言其类型就是U64。pgprot_t也是U64，描述了各个Translation table中一个描述符中的flag域。根据ARM硬件手册，Translation table中的描述符可能有四种情况：

（1）是table descriptor，指向下一级的Translation table

（2）是page descriptor，指向一个PAGE size的地址区域

（3）是block descriptor，指向一个Block size的地址区域

（4）无效描述符

对于L0，L1和L2级别的tranlation table而言，其描述符的定义如下：

![](http://www.wowotech.net/content/uploadfile/201608/a4ad1470999344.gif)

我们选择L1为例子来看看内核代码（其他的定义大家自己看source code吧，灰常类似）：

> #define PUD_TYPE_TABLE        (\_AT(pudval_t, 3) \<\< 0)\
> #define PUD_TABLE_BIT        (\_AT(pgdval_t, 1) \<\< 1)\
> #define PUD_TYPE_MASK        (\_AT(pgdval_t, 3) \<\< 0)\
> #define PUD_TYPE_SECT        (\_AT(pgdval_t, 1) \<\< 0)

描述符中，bit 0是表示该描述符是否是一个有效描述符，bit 1表示该描述符的类型。0b01是block descriptor，0b11是table描述符。L0，L1和L2的描述符不可能指向page descripotr。

这里我们还要具体讲讲什么是block。根据上文的描述，VA到PA的映射的最小力度就是page size（4K，16K或者64K），不过在有些情况下，例如kernel image对应地址空间的翻译，即便64K的page size也显得比较小（更大的mapping粒度意味着更高的TLB hit，从而提高性能），这时候，有一个叫做section的概念被提出了（ARM手册的术语是block），具体定义如下：

> #define SECTION_SHIFT        PMD_SHIFT\
> #define SECTION_SIZE        (\_AC(1, UL) \<\< SECTION_SHIFT)\
> #define SECTION_MASK        (~(SECTION_SIZE-1))

什么是section呢？我用一个实际的例子来描述好了。假设VA是48 bit，page size是4K，那么，在地址映射过程中，地址被分成9（level 0） ＋ 9（level 1） ＋ 9（level 2） ＋ 9（level 3） ＋ 12（page offset），对于kernel image这样的big block memory region，使用4K的page来mapping有点得不偿失，在这种情况下，可以考虑让level 2的Translation table entry指向一个2M 的memory region，而不是下一级的Translation table。所谓的section map就是指使用2M的为单位进行映射。对于其他的VA配置和page size的情况请大家自行理解吧，都是类似的。

对于L3级别的tranlation table而言，其描述符的定义和上图非常类似，只不过0b01是reserved，0b11是page描述符。L3的描述符不可能指向block descripotr或者table descriptor。具体定义如下：

> #define PTE_TYPE_MASK        (\_AT(pteval_t, 3) \<\< 0)\
> #define PTE_TYPE_FAULT        (\_AT(pteval_t, 0) \<\< 0)\
> #define PTE_TYPE_PAGE        (\_AT(pteval_t, 3) \<\< 0)\
> #define PTE_TABLE_BIT        (\_AT(pteval_t, 1) \<\< 1)

（还有一些其他相关的宏定义以后会慢慢补充）

参考文献：

1、ULK3 第二章

2、ARM Architecture Reference Manual(ARMv8, for ARMv8-A architecture profile)

标签: [arm64](http://www.wowotech.net/tag/arm64) [地址翻译](http://www.wowotech.net/tag/%E5%9C%B0%E5%9D%80%E7%BF%BB%E8%AF%91)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Linux kernel内核配置解析(5)\_Boot options(基于ARM64架构)](http://www.wowotech.net/linux_kenrel/kernel_config_boot_option.html) | [Linux kernel内核配置解析(1)\_概述(基于ARM64架构)](http://www.wowotech.net/linux_kenrel/kernel_config_overview.html)»

**评论：**

**[春天大cc](http://https//blog.csdn.net/leesagacious)**\
2018-09-07 21:22

开头的x64图错了，

[回复](http://www.wowotech.net/memory_management/arm64-memory-addressing.html#comment-6938)

**[春天大cc](http://https//blog.csdn.net/leesagacious)**\
2018-09-07 21:21

开头的那个x64的图错了， CR3 指向pml4表，持有pml4表的基地址

[回复](http://www.wowotech.net/memory_management/arm64-memory-addressing.html#comment-6937)

**Aiden**\
2018-09-14 09:04

@春天大cc：仔细看，基于X86的图，对应的是arm64的寄存器

[回复](http://www.wowotech.net/memory_management/arm64-memory-addressing.html#comment-6951)

**tree**\
2017-06-09 15:32

对32位机器来说， 共4G的地址空间， 用户空间占用0-3G， 内核空间占用3-4G.\
而对1G的内核空间由划分为三部分， ZONE_DMA, ZONE_NORMAL, ZONE_HIGHMEM,\
其中3G~3G+896M的空间是直接映射到物理地址的，内核中可以使用\_\_virt_to_phys/\_\_phys_to_virt相互转化。\
而对0-3G的用户空间虚拟地址转化为物理地址是通过页表转化进行，这一切都是通过硬件MMU去转化的？

[回复](http://www.wowotech.net/memory_management/arm64-memory-addressing.html#comment-5646)

**kk**\
2017-07-19 16:59

@tree：3G这个点是linux定义的，1G的内核虚拟地址空间在物理内存比较小（小于1G）时会把全部物理地址映射到内核空间，物理内存比较大时在内核的虚拟空间就保留了一部分区域(HIGHMEM)可以动态映射，CPU输出的是虚拟地址，经过段管理（i386）转换为线性地址（linux中虚拟地址等于线性地址），再经过页管理转换为物理地址。整个过程页表是关键，操作系统软件设置好页表，硬件MMU是页表的使用者。内核空间的页表是虚拟地址3G到物理地址0的映射，这个在系统启动时就设置好了，用户空间的页表在malloc等申请内存时不会分配，只是记录虚拟空间占用，在更新时内核才会建立页表并关联物理内存

[回复](http://www.wowotech.net/memory_management/arm64-memory-addressing.html#comment-5823)

**发表评论：**

昵称

邮件地址 (选填)

个人主页 (选填)

![](http://www.wowotech.net/include/lib/checkcode.php)

- ### 站内搜索

  蜗窝站内  互联网

- ### 功能

  [留言板\
  ](http://www.wowotech.net/message_board.html)[评论列表\
  ](http://www.wowotech.net/?plugin=commentlist)[支持者列表\
  ](http://www.wowotech.net/support_list)

- ### 最新评论

  - Shiina\
    [一个电路（circuit）中，由于是回路，所以用电势差的概念...](http://www.wowotech.net/basic_subject/voltage.html#8926)
  - Shiina\
    [其中比较关键的点是相对位置概念和点电荷的静电势能计算。](http://www.wowotech.net/basic_subject/voltage.html#8925)
  - leelockhey\
    [你这是哪个内核版本](http://www.wowotech.net/pm_subsystem/generic_pm_architecture.html#8924)
  - ja\
    [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
  - 元神高手\
    [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
  - 十七\
    [内核空间的映射在系统启动时就已经设定好，并且在所有进程的页表...](http://www.wowotech.net/process_management/context-switch-arch.html#8920)

- ### 文章分类

  - [Linux内核分析(25)](http://www.wowotech.net/sort/linux_kenrel) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=4)
    - [统一设备模型(15)](http://www.wowotech.net/sort/device_model) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=12)
    - [电源管理子系统(43)](http://www.wowotech.net/sort/pm_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=13)
    - [中断子系统(15)](http://www.wowotech.net/sort/irq_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=14)
    - [进程管理(31)](http://www.wowotech.net/sort/process_management) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=15)
    - [内核同步机制(26)](http://www.wowotech.net/sort/kernel_synchronization) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=16)
    - [GPIO子系统(5)](http://www.wowotech.net/sort/gpio_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=17)
    - [时间子系统(14)](http://www.wowotech.net/sort/timer_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=18)
    - [通信类协议(7)](http://www.wowotech.net/sort/comm) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=20)
    - [内存管理(31)](http://www.wowotech.net/sort/memory_management) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=21)
    - [图形子系统(2)](http://www.wowotech.net/sort/graphic_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=23)
    - [文件系统(5)](http://www.wowotech.net/sort/filesystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=26)
    - [TTY子系统(6)](http://www.wowotech.net/sort/tty_framework) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=27)
  - [u-boot分析(3)](http://www.wowotech.net/sort/u-boot) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=25)
  - [Linux应用技巧(13)](http://www.wowotech.net/sort/linux_application) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=3)
  - [软件开发(6)](http://www.wowotech.net/sort/soft) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=1)
  - [基础技术(13)](http://www.wowotech.net/sort/basic_tech) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=6)
    - [蓝牙(16)](http://www.wowotech.net/sort/bluetooth) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=10)
    - [ARMv8A Arch(15)](http://www.wowotech.net/sort/armv8a_arch) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=19)
    - [显示(3)](http://www.wowotech.net/sort/display) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=22)
    - [USB(1)](http://www.wowotech.net/sort/usb) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=28)
  - [基础学科(10)](http://www.wowotech.net/sort/basic_subject) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=7)
  - [技术漫谈(12)](http://www.wowotech.net/sort/tech_discuss) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=8)
  - [项目专区(0)](http://www.wowotech.net/sort/project) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=9)
    - [X Project(28)](http://www.wowotech.net/sort/x_project) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=24)

- ### 随机文章

  - [實作 spinlock on raspberry pi 2](http://www.wowotech.net/231.html)
  - [/proc/meminfo分析（一）](http://www.wowotech.net/memory_management/meminfo_1.html)
  - [统一设备模型：kobj、kset分析](http://www.wowotech.net/device_model/421.html)
  - [CFS调度器（5）-带宽控制](http://www.wowotech.net/process_management/451.html)
  - [DRAM 原理 3 ：DRAM Device](http://www.wowotech.net/basic_tech/321.html)

- ### 文章存档

  - [2024年2月(1)](http://www.wowotech.net/record/202402)
  - [2023年5月(1)](http://www.wowotech.net/record/202305)
  - [2022年10月(1)](http://www.wowotech.net/record/202210)
  - [2022年8月(1)](http://www.wowotech.net/record/202208)
  - [2022年6月(1)](http://www.wowotech.net/record/202206)
  - [2022年5月(1)](http://www.wowotech.net/record/202205)
  - [2022年4月(2)](http://www.wowotech.net/record/202204)
  - [2022年2月(2)](http://www.wowotech.net/record/202202)
  - [2021年12月(1)](http://www.wowotech.net/record/202112)
  - [2021年11月(5)](http://www.wowotech.net/record/202111)
  - [2021年7月(1)](http://www.wowotech.net/record/202107)
  - [2021年6月(1)](http://www.wowotech.net/record/202106)
  - [2021年5月(3)](http://www.wowotech.net/record/202105)
  - [2020年3月(3)](http://www.wowotech.net/record/202003)
  - [2020年2月(2)](http://www.wowotech.net/record/202002)
  - [2020年1月(3)](http://www.wowotech.net/record/202001)
  - [2019年12月(3)](http://www.wowotech.net/record/201912)
  - [2019年5月(4)](http://www.wowotech.net/record/201905)
  - [2019年3月(1)](http://www.wowotech.net/record/201903)
  - [2019年1月(3)](http://www.wowotech.net/record/201901)
  - [2018年12月(2)](http://www.wowotech.net/record/201812)
  - [2018年11月(1)](http://www.wowotech.net/record/201811)
  - [2018年10月(2)](http://www.wowotech.net/record/201810)
  - [2018年8月(1)](http://www.wowotech.net/record/201808)
  - [2018年6月(1)](http://www.wowotech.net/record/201806)
  - [2018年5月(1)](http://www.wowotech.net/record/201805)
  - [2018年4月(7)](http://www.wowotech.net/record/201804)
  - [2018年2月(4)](http://www.wowotech.net/record/201802)
  - [2018年1月(5)](http://www.wowotech.net/record/201801)
  - [2017年12月(2)](http://www.wowotech.net/record/201712)
  - [2017年11月(2)](http://www.wowotech.net/record/201711)
  - [2017年10月(1)](http://www.wowotech.net/record/201710)
  - [2017年9月(5)](http://www.wowotech.net/record/201709)
  - [2017年8月(4)](http://www.wowotech.net/record/201708)
  - [2017年7月(4)](http://www.wowotech.net/record/201707)
  - [2017年6月(3)](http://www.wowotech.net/record/201706)
  - [2017年5月(3)](http://www.wowotech.net/record/201705)
  - [2017年4月(1)](http://www.wowotech.net/record/201704)
  - [2017年3月(8)](http://www.wowotech.net/record/201703)
  - [2017年2月(6)](http://www.wowotech.net/record/201702)
  - [2017年1月(5)](http://www.wowotech.net/record/201701)
  - [2016年12月(6)](http://www.wowotech.net/record/201612)
  - [2016年11月(11)](http://www.wowotech.net/record/201611)
  - [2016年10月(9)](http://www.wowotech.net/record/201610)
  - [2016年9月(6)](http://www.wowotech.net/record/201609)
  - [2016年8月(9)](http://www.wowotech.net/record/201608)
  - [2016年7月(5)](http://www.wowotech.net/record/201607)
  - [2016年6月(8)](http://www.wowotech.net/record/201606)
  - [2016年5月(8)](http://www.wowotech.net/record/201605)
  - [2016年4月(7)](http://www.wowotech.net/record/201604)
  - [2016年3月(5)](http://www.wowotech.net/record/201603)
  - [2016年2月(5)](http://www.wowotech.net/record/201602)
  - [2016年1月(6)](http://www.wowotech.net/record/201601)
  - [2015年12月(6)](http://www.wowotech.net/record/201512)
  - [2015年11月(9)](http://www.wowotech.net/record/201511)
  - [2015年10月(9)](http://www.wowotech.net/record/201510)
  - [2015年9月(4)](http://www.wowotech.net/record/201509)
  - [2015年8月(3)](http://www.wowotech.net/record/201508)
  - [2015年7月(7)](http://www.wowotech.net/record/201507)
  - [2015年6月(3)](http://www.wowotech.net/record/201506)
  - [2015年5月(6)](http://www.wowotech.net/record/201505)
  - [2015年4月(9)](http://www.wowotech.net/record/201504)
  - [2015年3月(9)](http://www.wowotech.net/record/201503)
  - [2015年2月(6)](http://www.wowotech.net/record/201502)
  - [2015年1月(6)](http://www.wowotech.net/record/201501)
  - [2014年12月(17)](http://www.wowotech.net/record/201412)
  - [2014年11月(8)](http://www.wowotech.net/record/201411)
  - [2014年10月(9)](http://www.wowotech.net/record/201410)
  - [2014年9月(7)](http://www.wowotech.net/record/201409)
  - [2014年8月(12)](http://www.wowotech.net/record/201408)
  - [2014年7月(6)](http://www.wowotech.net/record/201407)
  - [2014年6月(6)](http://www.wowotech.net/record/201406)
  - [2014年5月(9)](http://www.wowotech.net/record/201405)
  - [2014年4月(9)](http://www.wowotech.net/record/201404)
  - [2014年3月(7)](http://www.wowotech.net/record/201403)
  - [2014年2月(3)](http://www.wowotech.net/record/201402)
  - [2014年1月(4)](http://www.wowotech.net/record/201401)

[![订阅Rss](http://www.wowotech.net/content/templates/default/images/rss.gif)](http://www.wowotech.net/rss.php "RSS订阅")

Copyright @ 2013-2015 [蜗窝科技](http://www.wowotech.net/ "wowotech") All rights reserved. Powered by [emlog](http://www.emlog.net/ "采用emlog系统")

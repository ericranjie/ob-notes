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

作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2016-11-10 19:07 分类：[内存管理](http://www.wowotech.net/sort/memory_management)

一、前言

本文没有什么框架性的东西，就是按照__create_page_tables代码的执行路径走读一遍，记录在初始化阶段，内核是如何创建内核运行需要的页表过程。想要了解一些概述性的、框架性的东西可以参考[内存初始化](http://www.wowotech.net/memory_management/mm-init-1.html)文档。

本文的代码来自ARM64，内核版本是4.4.6，此外，阅读本文最好熟悉ARMv8中翻译表描述符的格式。

二、create_table_entry

这个宏定义主要是用来创建一个中间level的translation table中的描述符。如果用linux的术语，就是创建PGD、PUD或者PMD的描述符。如果用ARM64术语，就是创建L0、L1或者L2的描述符。具体创建哪一个level的Translation table descriptor是由tbl参数指定的，tbl指向了该translation table的内存。virt参数给出了要创建地址映射的那个虚拟地址，shift参数以及ptrs参数是和具体在哪一个entry中写入描述符有关。我们知道，在定位页表描述的时候，我们需要截取虚拟地址中的一部分做为offset（index）来定位描述符，实际上，虚拟地址右移shift，然后截取ptrs大小的bit field就可以得到entry index了。tmp1和tmp2是临时变量。create_table_entry的代码如下：

> .macro    create_table_entry, tbl, virt, shift, ptrs, tmp1, tmp2  
> lsr    \tmp1, \virt, #\shift  
> and    \tmp1, \tmp1, #\ptrs - 1    // table index－－－－－－－－－－－－－－－－－－－（1）  
> add    \tmp2, \tbl, #PAGE_SIZE－－－－－－－－－－－－－－－－－－－－－－－－－（2）  
> orr    \tmp2, \tmp2, #PMD_TYPE_TABLE－－－－－－－－－－－－－－－－－－－－－（3）  
> str    \tmp2, [\tbl, \tmp1, lsl #3]－－－－－－－－－－－－－－－－－－－－－－－－－－（4）  
> add    \tbl, \tbl, #PAGE_SIZE－－－－－－－－－－－－－－－－－－－－－－－－－－－（5）  
> .endm

（1）tmp1中保存virt地址对应在Translation table中的entry index。

（2）初始阶段的页表页表定义在链接脚本中，如下：

> BSS_SECTION(0, 0, 0)
> 
> . = ALIGN(PAGE_SIZE);  
> idmap_pg_dir = .;  
> . += IDMAP_DIR_SIZE;  
> swapper_pg_dir = .;  
> . += SWAPPER_DIR_SIZE;

初始阶段的页表（PGD/PUD/PMD/PTE）都是排列在一起的，每一个占用一个page。也就是说，如果create_table_entry当前操作的是PGD，那么tmp2这时候保存了下一个level的页表，也就是PUD了。

（3）这一步是合成描述符的数值。光有下一级translation table的地址不行，还要告知该描述符是否有效（bit 0），该描述符的类型是哪一种类型（bit 1）。对于中间level的页表，该描述符不可能是block entry，只能是table type的描述符，因此该描述符的最低两位是0b11。

> #define PMD_TYPE_TABLE        (_AT(pmdval_t, 3) << 0)

（4）这是最关键的一步，将描述符写入页表中。之所以有“lsl #3”操作，是因为一个描述符占据8个Byte。

（5）将translation table的地址移到next level，以便进行下一步设定。

三、create_pgd_entry

从字面上看，create_pgd_entry似乎是用来在PGD中创建一个描述符，但是，实际上该函数不仅仅创建PGD中的描述符，如果需要下一级的translation table，例如PUD、PMD，也需要同时建立，最终的要求是能够完成所有中间level的translation table的建立（其实每个table中都是只建立了一个描述符），仅仅留下PTE，由其他代码来完成。该函数需要四个参数：tbl是pgd translation table的地址，具体要创建哪一个地址的描述符由virt指定，tmp1和tmp2是临时变量，create_pgd_entry具体代码如下：

>     .macro    create_pgd_entry, tbl, virt, tmp1, tmp2  
>     create_table_entry \tbl, \virt, PGDIR_SHIFT, PTRS_PER_PGD, \tmp1, \tmp2－－－－－－－（1）  
> #if SWAPPER_PGTABLE_LEVELS > 3－－－－－－－－－－－－－－－－－－－－－－－－（2）  
>     create_table_entry \tbl, \virt, PUD_SHIFT, PTRS_PER_PUD, \tmp1, \tmp2－－－－－－－－（3）  
> #endif  
> #if SWAPPER_PGTABLE_LEVELS > 2－－－－－－－－－－－－－－－－－－－－－－－－（4）  
>     create_table_entry \tbl, \virt, SWAPPER_TABLE_SHIFT, PTRS_PER_PTE, \tmp1, \tmp2  
> #endif  
>     .endm

（1）create_table_entry 在上一节已经描述了，这里通过调用该函数在PGD中为虚拟地址virt创建一个table type的描述符。

（2）SWAPPER_PGTABLE_LEVELS这个宏定义和ARM64_SWAPPER_USES_SECTION_MAPS相关，而这个宏在蜗窝已经[有一篇文章](http://www.wowotech.net/linux_kenrel/create_page_tables.html)描述，这里就不说了。SWAPPER_PGTABLE_LEVELS其实定义了swapper进程地址空间的页表的级数，可能3，也可能是2，具体中间的Translation table有多少个level是和配置相关的，如果是section mapping，那么中间level包括PGD和PUD就OK了，PMD是最后一个level。如果是page mapping，那么需要PGD、PUD和PMD这三个中间level，PTE是最后一个level。当然，如果整个page level是3或者2的时候，也有可能不存在PUD或者PMD这个level。

（3）当SWAPPER_PGTABLE_LEVELS > 3的时候，需要创建PUD这一级的Translation table。

（4）当SWAPPER_PGTABLE_LEVELS > 2的时候，需要创建PMD这一级的Translation table。

上面太枯燥了，我们给出一些实例：

例子1：当虚拟地址是48个bit，4k page size，这时候page level等于4，映射关系是PGD（L0）--->PUD（L1）--->PMD（L2）--->Page table（L3）--->page，但是如果采用了section mapping（4k的page一定会采用section mapping），映射关系是PGD（L0）--->PUD（L1）--->PMD（L2）--->section。在create_pgd_entry函数中将创建PGD和PUD这两个中间level。

例子2：当虚拟地址是48个bit，16k page size（不能采用section mapping），这时候page level等于4，映射关系是PGD（L0）--->PUD（L1）--->PMD（L2）--->Page table（L3）--->page。在create_pgd_entry函数中将创建PGD、PUD和PMD这三个中间level。

例子3：当虚拟地址是39个bit，4k page size，这时候page level等于3，映射关系是PGD（L1）--->PMD（L2）--->Page table（L3）--->page。由于是4k page，因此采用section mapping，映射关系是PGD（L1）--->PMD（L2）--->section。在create_pgd_entry函数中将创建PGD这一个中间level。

四、create_block_map

create_block_map的名字起得不错，该函数就是在tbl指定的Translation table中建立block descriptor以便完成address mapping。具体mapping的内容是将start 到 end这一段VA mapping到phys开始的PA上去，代码如下：

>     .macro    create_block_map, tbl, flags, phys, start, end  
>     lsr    \phys, \phys, #SWAPPER_BLOCK_SHIFT  
>     lsr    \start, \start, #SWAPPER_BLOCK_SHIFT  
>     and    \start, \start, #PTRS_PER_PTE - 1    // table index  
>     orr    \phys, \flags, \phys, lsl #SWAPPER_BLOCK_SHIFT    // table entry  
>     lsr    \end, \end, #SWAPPER_BLOCK_SHIFT  
>     and    \end, \end, #PTRS_PER_PTE - 1        // table end index  
> 9999:    str    \phys, [\tbl, \start, lsl #3]        // store the entry  
>     add    \start, \start, #1            // next entry  
>     add    \phys, \phys, #SWAPPER_BLOCK_SIZE        // next block  
>     cmp    \start, \end  
>     b.ls    9999b  
>     .endm

五、__create_page_tables

1、准备阶段

> __create_page_tables:  
>     adrp    x25, idmap_pg_dir－－－－－－－－－－－－－－－－－－－－－－－－（1）  
>     adrp    x26, swapper_pg_dir  
>     mov    x27, lr
> 
>     mov    x0, x25－－－－－－－－－－－－－－－－－－－－－－－－－－－－－（2）  
>     add    x1, x26, #SWAPPER_DIR_SIZE  
>     bl    __inval_cache_range
> 
>     mov    x0, x25－－－－－－－－－－－－－－－－－－－－－－－－－－－－－（3）  
>     add    x6, x26, #SWAPPER_DIR_SIZE  
> 1:    stp    xzr, xzr, [x0], #16  
>     stp    xzr, xzr, [x0], #16  
>     stp    xzr, xzr, [x0], #16  
>     stp    xzr, xzr, [x0], #16  
>     cmp    x0, x6  
>     b.lo    1b
> 
>     ldr    x7, =SWAPPER_MM_MMUFLAGS－－－－－－－－－－－－－－－－－（4）

（1）取idmap_pg_dir这个符号的物理地址，保存到x25。取swapper_pg_dir这个符号的物理地址，保存到x26。这段代码没有什么特别要说明的，除了adrp这条指令。adrp是计算指定的符号地址到run time PC值的相对偏移（不过，这个offset没有那么精确，是以4K为单位，或者说，低12个bit是0）。在指令编码的时候，立即数（也就是 offset）占据21个bit，此外，由于偏移计算是按照4K进行的，因此最后计算出来的符号地址必须要在该指令的－4G和4G之间。由于执行该指令的 时候，还没有打开MMU，因此通过adrp获取的都是物理地址，当然该物理地址的低12个bit是全零的。此外，由于在链接脚本中 idmap_pg_dir和swapper_pg_dir是page size aligned，因此使用adrp指令也是OK的。

（2）这段代码是要进行invalid cache的操作了，具体要操作的范围就是identity mapping和kernel image mapping所对应的页表区域，起始地址是idmap_pg_dir，结束地址是swapper_pg_dir＋SWAPPER_DIR_SIZE。

为什么要调用__inval_cache_range来invalidate idmap_pg_dir和swapper_pg_dir对应页表空间的cache呢？根据boot protocol，代码执行到此，对于cache的要求是kernel image对应的那段空间的cache line是clean到PoC的，不过idmap_pg_dir和swapper_pg_dir对应页表空间不属于kernel image的一部分，因此其对应的cacheline很可能有一些旧的，无效的数据，必须要清理掉。

（3）将idmap和swapper页表内容设定为0是有意义的。实际上这些translation table中的大部分entry都是没有使用的，PGD和PUD都是只有一个entry是有用的，而PMD中有效的entry数目是和mapping的地 址size有关。将页表内容清零也就是意味着将页表中所有的描述符设定为invalid（描述符的bit 0指示是否有效，等于0表示无效描述符）。

（4）要创建mapping除了需要VA和PA，还需要memory attribute的参数，这个参数定义如下：

> #if ARM64_SWAPPER_USES_SECTION_MAPS  
> #define SWAPPER_MM_MMUFLAGS    (PMD_ATTRINDX(MT_NORMAL) | SWAPPER_PMD_FLAGS)  
> #else  
> #define SWAPPER_MM_MMUFLAGS    (PTE_ATTRINDX(MT_NORMAL) | SWAPPER_PTE_FLAGS)  
> #endif

为了理解这些定义，需要理解block type和page type的描述符的格式，大家自行对照ARMv8文档，这里就不贴图了。SWAPPER_MM_MMUFLAGS这个flag其实定义了要映射地址的memory attribut。对于kernel image这一段内存，当然是普通内存，因此其中的MT_NORMAL就是表示后续的地址映射都是为normal memory而创建的。其他的flag定义如下：

> #define SWAPPER_PTE_FLAGS    (PTE_TYPE_PAGE | PTE_AF | PTE_SHARED)  
> #define SWAPPER_PMD_FLAGS    (PMD_TYPE_SECT | PMD_SECT_AF | PMD_SECT_S)

PMD_SECT_AF（PTE_AF）中的AF是access flag的缩写，这个bit用来表示该entry是否第一次使用（当程序访问对应的page或者section的时候，就会使用该entry，如果从来没有被访问过，那么其值等于0，否者等于1）。该bit主要被操作系统用来跟踪一个page是否被使用过（最近是否被访问），当该page首次被创建的时候，AF等于0，当代码第一次访问该page的时候，会产生MMU fault，这时候，异常处理函数应该设定AF等于1，从而阻止下一次访问该page的时候产生MMU Fault。在这里，kernel image对应的page，其描述符的AF bit都设定为1，表示该page当前状态是actived（最近被访问），因为只有用户空间进程的page才会根据AF bit来确定哪些page被swap out，而kernel image对应的page是always actived的。

PMD_SECT_S（PTE_SHARED）对应shareable attribute bits，这个两个bits定义了该page的shareable attribute。那么是shareable attribute呢？shareable attribute定义了memory location被多个系统中的bus master共享的属性。具体定义如下：

|   |   |
|---|---|
|SH[1:0]|Normal memory|
|00|Non-shareable|
|01|无效|
|10|outer shareable|
|11|inner shareble|

这里memory attribute中SH被设定为0b11，即inner shareable。如果一个page被标注为inner shareable，那么在inner shareable domain中，所有的bus mast访问该page中的memory都是coherent的（HW会处理cache coherence问题），软件不需要考虑cache。一般而言，所有cpu core组成了inner shareable domain，也就是说，对于kernel direct mapping而言，其对应的内存对所有的cpu core的访问都是coherent的。

memory attribute中其他的flag都没有显式指定，也就是说它们的值都是0，我们可以简单过一下。AP的值是0，表示该page对kernel mode（EL1）是read/write的，对于userspace（EL0），是不允许访问的。nG bit是0，表示该地址翻译是全局的，不是process-specific的，这也合理，内核page的映射当然是全局的了。

2、建立identity mapping

>     mov    x0, x25                －－－－－－－－－－－－－－－－－－－－－－－－－（1）  
>     adrp    x3, __idmap_text_start        －－－－－－－－－－－－－－－－－－－－（2）
> 
> #ifndef CONFIG_ARM64_VA_BITS_48－－－－－－－－－－－－－－－－－－－－－（3）  
> #define EXTRA_SHIFT    (PGDIR_SHIFT + PAGE_SHIFT - 3)－－－－－－－－－－－（4）  
> #define EXTRA_PTRS    (1 << (48 - EXTRA_SHIFT)) －－－－－－－－－－－－－－－（5）
> 
>   
> #if VA_BITS != EXTRA_SHIFT－－－－－－－－－－－－－－－－－－－－－－－－－（6）  
> #error "Mismatch between VA_BITS and page size/number of translation levels"  
> #endif
> 
>     adrp    x5, __idmap_text_end－－－－－－－－－－－－－－－－－－－－－－－－－（7）  
>     clz    x5, x5  
>     cmp    x5, TCR_T0SZ(VA_BITS)    －－－－－－－－－－－－－－－－－－－－－－（8）  
>     b.ge    1f
> 
>     adr_l    x6, idmap_t0sz－－－－－－－－－－－－－－－－－－－－－－－－－－－（9）  
>     str    x5, [x6]  
>     dmb    sy  
>     dc    ivac, x6
> 
>     create_table_entry x0, x3, EXTRA_SHIFT, EXTRA_PTRS, x5, x6－－－－－－－－（10）  
> 1:  
> #endif
> 
>     create_pgd_entry x0, x3, x5, x6－－－－－－－－－－－－－－－－－－－－－－－（11）  
>     mov    x5, x3                // __pa(__idmap_text_start)  
>     adr_l    x6, __idmap_text_end        // __pa(__idmap_text_end)  
>     create_block_map x0, x7, x3, x5, x6－－－－－－－－－－－－－－－－－－－－－（12）

（1）x0保存了idmap_pg_dir变量的物理地址，也就是identity mapping的PGD。

（2）x3保存了__idmap_text_start的物理地址，对于identity mapping而言，x3也保存了虚拟地址，因为虚拟地址是等于物理地址的。

（3）基本上创建identity mapping是没有什么大问题的，但是，如果物理内存的地址位于非常高的位置，那么在进行identity mapping就有问题了，因为有可能你配置的VA_BITS不够大，超出了虚拟地址的范围。这时候，就需要扩展virtual address range了。当然，如果配置了48bits的VA_BITS就不存在这样的问题了，因为ARMv8最大支持的VA BITS就是48个，根本不可能扩展了。

（4）在虚拟地址地址不是48 bit，而系统内存的物理地址又放到了非常非常高的位置，这时候，为了完成identity mapping，我们必须要扩展虚拟地址，那么扩展多少呢？扩展到48个bit。扩展之后，增加了一个EXTRA的level，地址映射关系是EXTRA--->PGD--->……，其中EXTRA_SHIFT等于(PGDIR_SHIFT + PAGE_SHIFT - 3)。

（5）扩展之后，地址映射多个一个level，我们称之EXTRA level，该level的Translation table中有多少个entry呢？EXTRA_PTRS给出了答案。

（6）其实现行的linux kernel中，对地址映射是有要求的，即要求PGD是满的。例如：48 bit的虚拟地址，4k的page size，对应的映射关系是PGD(9-bit）＋PUD(9-bit）＋PMD(9-bit）＋PTE(9-bit）＋page offset(12-bit），对于42bit的虚拟地址，64k的page size，对应的映射关系是PGD(13-bit）＋ PTE(13-bit）＋ page offset(16-bit）。这两种例子有一个共同的特点就是PGD中的entry数目都是满的，也就是说索引到PGD的bit数目都是PAGE_SIZE-3。如果不满足这个关系，linux kernel会认为你的配置是有问题的。注意：这是内核的要求，实际上ARM64的硬件没有这么要求。

正因为正确的配置下，PGD都是满的，因此扩展之后EXTRA_SHIFT一定是等于VA_BITS的，否则一定是你的配置有问题。我们延续上一个实例来说明如何扩展虚拟地址的bit数目。对于42bit的虚拟地址，64k的page size，扩展之后，虚拟地址是48个bit，地址映射关系是EXTRA（6-bit）＋ PGD(13-bit）＋ PTE(13-bit）＋ page offset(16-bit）。

（7）x5保存了__idmap_text_end的物理地址，之所以这么做是因为需要确定identity mapping的最高的物理地址，计算该物理地址的前导0有多少个，从而可以判断该地址是否是位于物理地址空间中比较高的位置。

（8）宏定义TCR_T0SZ可以计算给定虚拟地址数目下，前导0的个数。如果虚拟地址是48的话，那么前导0是16个。如果当前物理地址的前导0的个数（x5的值）还有小于当前配置虚拟地址的前导0的个数，那么就需要扩展。

（9）OK，现在进入需要扩展的分支，当然，具体要配置虚拟地址是通过TCR_EL1寄存器中的T0SZ域进行的，现在还不是时候（具体的设定在__cpu_setup函数中），这里，我们只要设定idmap_t0sz这个变量值就OK了，在__cpu_setup函数中会从该变量取值并设定到TCR_EL1寄存器中的。代码中，x6是idmap_t0sz变量的物理地址，x5是物理地址前导0的个数，将其保存到idmap_t0sz变量中。

（10）创建extra translation table的entry。具体传递的参数如下：

x0：页表地址idmap_pg_dir

x3：准备映射的虚拟地址（虽然x3保存的是物理地址，但是identity mapping嘛，VA和PA都是一样的）

EXTRA_SHIFT：正常建立最高level mapping的时候， shift是PGDIR_SHIFT，但是，由于物理地址位置太高，需要额外的映射，因此这里需要再加上一个level的mapping，因此shift需要PGDIR_SHIFT ＋ （PAGE_SHIFT - 3）。

EXTRA_PTRS：增加了一个level的Translation table，我们需要确定这个增加level的Translation table中包含的描述符的数目，EXTRA_PTRS给出了这个参数。

（11）create_pgd_entry这个函数上面解释过了，建立各个中间level的table描述符。

（12）创建最后一个level translation table的entry。该entry可能是page descriptor，也可能是block descriptor，具体传递的参数如下：

x0：指向最后一个level的translation table

x7：要创建映射的memory attribute

x3：物理地址

x5：虚拟地址的起始地址（其实和x3一样）

x6：虚拟地址的结束地址

3、创建kernel direct mapping

> mov    x0, x26                －－－－－－－－－－－－－－－－－－－－－－－－（1）  
> mov    x5, #PAGE_OFFSET－－－－－－－－－－－－－－－－－－－－－－－（2）  
> create_pgd_entry x0, x5, x3, x6－－－－－－－－－－－－－－－－－－－－－（3）  
> ldr    x6, =KERNEL_END            // __va(KERNEL_END)  
> mov    x3, x24                // phys offset  
> create_block_map x0, x7, x3, x5, x6 －－－－－－－－－－－－－－－－－－－（4）
> 
>   
> mov    x0, x25  
> add    x1, x26, #SWAPPER_DIR_SIZE  
> dmb    sy  
> bl    __inval_cache_range
> 
> mov    lr, x27  
> ret

（1）swapper_pg_dir其实就是swapper进程（pid等于0的那个，其实就是idle进程）的地址空间，这时候，x0指向了内核地址空间的PGD的基地址。

（2）PAGE_OFFSET是kernel image的首地址，对于48bit的VA而言，该地址是0xffff8000-00000000

（3）创建PAGE_OFFSET（即kernel image首地址，虚拟地址）对应中间level的table描述符。

（4）创建PAGE_OFFSET～KERNEL_END之间地址映射的最后一个level的描述符。

参考文献：

1、ARMv8技术手册

2、Linux 4.4.6内核源代码

_原创文章，转发请注明出处。蜗窝科技_

标签: [__create_page_tables](http://www.wowotech.net/tag/__create_page_tables)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [X-016-KERNEL-串口驱动开发之驱动框架](http://www.wowotech.net/x_project/serial_driver_porting_1.html) | [蓝牙协议分析(8)_BLE安全机制之白名单](http://www.wowotech.net/bluetooth/ble_white_list.html)»

**评论：**

**see**  
2019-09-29 10:18

记得之前有一篇文档介绍identity mapping和kernel image mapping的内存映射图，怎么找不到那篇文章了？

[回复](http://www.wowotech.net/memory_management/__create_page_tables_code_analysis.html#comment-7673)

**rikeyone**  
2018-04-12 11:57

mov    x0, x26                －－－－－－－－－－－－－－－－－－－－－－－－（1）  
mov    x5, #PAGE_OFFSET－－－－－－－－－－－－－－－－－－－－－－－（2）  
create_pgd_entry x0, x5, x3, x6－－－－－－－－－－－－－－－－－－－－－（3）  
ldr    x6, =KERNEL_END            // __va(KERNEL_END)  
mov    x3, x24                // phys offset  
create_block_map x0, x7, x3, x5, x6 －－－－－－－－－－－－－－－－－－－（4）  
  
  
关于这段 kernel direct map，有个问题想请教，上面只执行一次create pgd entry函数其实是在各个level的页表中只创建了一个entry。那么它创建出来的映射地址范围也才4K ， kernel image的大小不会超过这个界限吗。

[回复](http://www.wowotech.net/memory_management/__create_page_tables_code_analysis.html#comment-6657)

**[smcdef](http://www.wowotech.net/)**  
2018-04-12 22:56

@rikeyone：如果是4k页面，create_pgd_entry就是为PGD页表的某一个表项填充PUD页表，继续填充PUD页表的页表项指向PMD页表，最后create_block_map创建对应的PMD页表的表项。使用section mapping，也就是PMD页表的表项填充block descriptor，也就是2M大小。

[回复](http://www.wowotech.net/memory_management/__create_page_tables_code_analysis.html#comment-6658)

**xuan**  
2021-01-23 16:17

@rikeyone：create_block_map内部是一个循环语句，在设置swapper_pg_dir时，是将虚拟地址段PAGE_OFFSET到KERNEL_END映射到__pa(__PHYS_OFFSET)起始的连续物理地址段上，设置的页表entry是在PMD上，如果是4KB的page，那么最多可以达到512个entry，每个entry对应的设置大小是2MB，最大也就是1GB，没有哪个Image的大小是超过1GB的吧？

[回复](http://www.wowotech.net/memory_management/__create_page_tables_code_analysis.html#comment-8187)

**rikeyone**  
2018-04-11 15:09

Hi,linuxer：  
我想请教一下，PGD的入口地址最终会被设置到CP15的TTBR寄存器，我想了解的是，这个地址后续会由MMU去访问并去解析地址，那么MMU能访问到的PGD空间有多大？  
换句话说，PGD这一个level的页表大小是否有一个限制。比如4KB或者其他？

[回复](http://www.wowotech.net/memory_management/__create_page_tables_code_analysis.html#comment-6655)

**eastl**  
2017-12-11 17:55

你好，有個小問題想請教一下  
這邊__create_page_tables的return是透過自己呼叫ret來return回去  
但是在preserve_boot_args是在__inval_cache_range裡面做ret  
為何這邊不要一樣在__inval_cache_range使用b不要用bl  
來讓__inval_cache_rangeret回去就好呢

[回复](http://www.wowotech.net/memory_management/__create_page_tables_code_analysis.html#comment-6340)

**[electrlife](http://www.wowotech.net/)**  
2016-12-21 14:59

初始阶段的页表（PGD/PUD/PMD/PTE）都是排列在一起的，每一个占用一个page。  
===>> 请问下，每一个页表占用一个page，且排列在一起，为什么每个页表使用一个page，是合适的？比如：pgd 页表，如果4k page，那么最多保存4k/8=512 entry。对于0xC0000000会不会落到其512entry之外，或者说使用一个页来保存pgd页表在任何配置下都会使用其kernel的虚拟地址落在一个页内。

[回复](http://www.wowotech.net/memory_management/__create_page_tables_code_analysis.html#comment-5045)

**[tigger](http://www.wowotech.net/)**  
2016-12-07 19:00

tlb参数里面都保存着什么？只是指向该translation talbe的内存吗？

[回复](http://www.wowotech.net/memory_management/__create_page_tables_code_analysis.html#comment-4988)

**[linuxer](http://www.wowotech.net/)**  
2016-12-08 10:16

@tigger：tbl指向某个level的translation table。  
BTW，是tbl（表示table）而不是tlb

[回复](http://www.wowotech.net/memory_management/__create_page_tables_code_analysis.html#comment-4992)

**[tigger](http://www.wowotech.net/)**  
2016-12-07 18:58

hi linuxer  
这个宏定义主要是用来创建一个中间level的translation table中的描述符  
==============================================================  
那第一个level的translation table在哪里？放在TTBR0_ELX中吗？  
  
如果是这样的话，第一level的translation table怎么找到你说的这个中间级别的？

[回复](http://www.wowotech.net/memory_management/__create_page_tables_code_analysis.html#comment-4987)

**[linuxer](http://www.wowotech.net/)**  
2016-12-08 10:19

@tigger：我是这样定义中间level的页表：  
TTBR-->（中间level的页表）--->PTE--->page  
  
你说的第一level的translation table也是中间level的页表，也就是说中间level的页表就是那些描述符是table描述的level，只有最后一个translation table不是中间level

[回复](http://www.wowotech.net/memory_management/__create_page_tables_code_analysis.html#comment-4993)

**发表评论：**

 昵称

 邮件地址 (选填)

 个人主页 (选填)

![](http://www.wowotech.net/include/lib/checkcode.php) 

- ### 站内搜索
    
       
     蜗窝站内  互联网
    
- ### 功能
    
    [留言板  
    ](http://www.wowotech.net/message_board.html)[评论列表  
    ](http://www.wowotech.net/?plugin=commentlist)[支持者列表  
    ](http://www.wowotech.net/support_list)
- ### 最新评论
    
    - Shiina  
        [一个电路（circuit）中，由于是回路，所以用电势差的概念...](http://www.wowotech.net/basic_subject/voltage.html#8926)
    - Shiina  
        [其中比较关键的点是相对位置概念和点电荷的静电势能计算。](http://www.wowotech.net/basic_subject/voltage.html#8925)
    - leelockhey  
        [你这是哪个内核版本](http://www.wowotech.net/pm_subsystem/generic_pm_architecture.html#8924)
    - ja  
        [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
    - 元神高手  
        [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
    - 十七  
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
    
    - [Concurrency Managed Workqueue之（一）：workqueue的基本概念](http://www.wowotech.net/irq_subsystem/workqueue.html)
    - [Linux cpuidle framework(4)_menu governor](http://www.wowotech.net/pm_subsystem/cpuidle_menu_governor.html)
    - [关于java单线程经常占用cpu100%分析](http://www.wowotech.net/linux_kenrel/483.html)
    - [load_balance函数代码详解](http://www.wowotech.net/process_management/load_balance_function.html)
    - [Linux读写锁逻辑解析](http://www.wowotech.net/kernel_synchronization/rwsem.html)
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
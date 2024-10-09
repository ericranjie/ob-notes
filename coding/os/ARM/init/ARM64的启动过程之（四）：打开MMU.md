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

作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2015-10-24 12:35 分类：[ARMv8A Arch](http://www.wowotech.net/sort/armv8a_arch)

一、前言

经过漫长的前戏，我们终于迎来了打开MMU的时刻，本文主要描述打开MMU以及跳转到start_kernel之前的代码逻辑。这一节完成之后，我们就会离开痛苦的汇编，进入人民群众喜闻乐见的c代码了。

二、打开MMU前后的概述

对CPU以及其执行的程序而言，打开MMU是一件很有意思的事情，好象从现实世界一下子走进了奇妙的虚幻世界，本节，我们一起来看看内核是如何“穿越”的。下面这张图描述了两个不同的世界：

[![mmu-on-off](http://www.wowotech.net/content/uploadfile/201512/b8d2febaa80339bcb73e255061e7aada20151202015400.gif "mmu-on-off")](http://www.wowotech.net/content/uploadfile/201512/60448863bf7d21ce6ac493a4b2d92de920151202015359.gif)

当没有打开MMU的时候，cpu在进行取指以及数据访问的时候是直接访问物理内存或者IO memory。虽然64bit的CPU理论上拥有非常大的address space，但是实际上用于存储kernel image的物理main memory并没有那么大，一般而言，系统的main memory在低端的一小段物理地址空间中，如上图右侧的图片所示。当打开MMU的时候，cpu对memory系统的访问不能直接触及物理空间，而是需要通过一系列的Translation table进行翻译。虚拟地址空间分成三段，低端是0x00000000_00000000～0x0000FFFF_FFFFFFFF，用于user space。高端是0xFFFF0000_00000000～0xFFFFFFFF_FFFFFFFF，用于kernel space。中间的一段地址是无效地址，对其访问会产生MMU fault。虚拟地址空间如上图右侧的图片所示。

Linker感知的是虚拟地址，在将内核的各个object文件链接成一个kernel image的时候，kernel image binary code中访问的都是虚拟地址，也就是说kernel image应该运行在Linker指定的虚拟地址空间上。问题来了，kernel image运行在那个地址上呢？实际上，将kernel image放到kernel space的首地址运行是一个最直观的想法，不过由于各种原因，具体的arch在编译内核的时候，可以指定一个offset（TEXT_OFFSET），对于ARM64而言是512KB（0x00080000），因此，编译后的内核运行在0xFFFF8000_00080000的地址上。系统启动后，bootloader会将kernel image copy到main memory，当然，和虚拟地址空间类似，kernel image并没有copy到main memory的首地址，也保持了一个同样size的offset。现在，问题又来了：在kernel的开始运行阶段，MMU是OFF的，也就是说kernel image是直接运行在物理地址上的，但是实际上kernel是被linker链接到了虚拟地址上去的，在这种情况下，在没有turn on MMU之前，kernel能正常运行吗？可以的，如果kernel在turn on MMU之前的代码都是PIC的，那么代码实际上是可以在任意地址上运行的。你可以仔细观察turn on MMU之前的代码，都是位置无关的代码。

OK，解决了MMU turn on之前的问题，现在我们可以准备“穿越”了。真正打开MMU就是一条指令而已，就是将某个system register的某个bit设定为1之类的操作。这样我们可以把相关指令分成两组，turn on mmu之前的绿色指令和之后的橘色指令，如下图所示：

[![mmu-1](http://www.wowotech.net/content/uploadfile/201510/34e38e720d5cc7776246207c6cdc92bd20151024043506.gif "mmu-1")](http://www.wowotech.net/content/uploadfile/201510/a4baf59a65665a921228cf031606bbe320151024043505.gif)

由于现代CPU的设计引入了pipe， super scalar，out-of-order execution，分支预测等等特性，实际上在turn on MMU的指令执行的那个时刻，该指令附近的指令的具体状态有些混乱，可能绿色指令执行的数据加载在实际在总线上发起bus transaction的时候已经启动了MMU，本来它是应该访问physical address space的。而也有可能橘色的指令提前执行，导致其发起的memory操作在MMU turn on之前就完成。为了解决这些混乱，可以采取一种投机取巧的办法，就是建立一致性映射：假设kernel image对应的物理地址段是A～B这一段，那么在建立页表的时候就把A～B这段虚拟地址段映射到A～B这一段的物理地址。这样，在turn on MMU附近的指令是毫无压力的，无论你是通过虚拟地址还是物理地址，访问的都是同样的物理memory。

还有一种方法，就是清楚的隔离turn on MMU前后的指令，那就是使用指令同步工具，如下：

[![mmu-2](http://www.wowotech.net/content/uploadfile/201510/1814f128c7e1cf94cacff66e8824056220151024043507.gif "mmu-2")](http://www.wowotech.net/content/uploadfile/201510/4e9eff9c8d6e4d68316b168c76299b3620151024043506.gif)

指令屏障可以清清楚楚的把指令的执行划分成三段，第一段是绿色指令，在执行turn on mmu指令执行之前全部完成，随后启动turn on MMU的指令，随后的指令屏障可以确保turn on MMU的指令完全执行完毕（整个计算机系统的视图切换到了虚拟世界），这时候才启动橘色指令的取指、译码、执行等操作。

三、打开MMU的代码

具体打开MMU的代码在\_\_enable_mmu函数中如下：

> \_\_enable_mmu:\
> ldr    x5, =vectors\
> msr    vbar_el1, x5 －－－－－－－－－－－－－－－－－－－－－－－－－－－（1）\
> msr    ttbr0_el1, x25            // load TTBR0 －－－－－－－－－－－－－－－－－（2）\
> msr    ttbr1_el1, x26            // load TTBR1\
> isb\
> msr    sctlr_el1, x0 －－－－－－－－－－－－－－－－－－－－－－－－－－－（3）\
> isb\
> br    x27 －－－－－－－－－－－－－跳转到\_\_mmap_switched执行，不设定lr寄存器\
> ENDPROC(\_\_enable_mmu)

传入该函数的参数有四个，一个是x0寄存器，该寄存器中保存了打开MMU时候要设定的SCTLR_EL1的值（在\_\_cpu_setup函数中设定），第二个是个是x25寄存器，保存了idmap_pg_dir的值。第三个参数是x26寄存器，保存了swapper_pg_dir的值。最后一个参数是x27，是执行完毕该函数之后，跳转到哪里去执行（\_\_mmap_switched）。

（1）VBAR_EL1, Vector Base Address Register (EL1)，该寄存器保存了EL1状态的异常向量表。在ARMv8中，发生了一个exception，首先需要确定的是该异常将送达哪一个exception level。如果一个exception最终送达EL1，那么cpu会跳转到这里向量表来执行。具体异常的处理过程由其他文档描述，这里就不说了。

（2）idmap_pg_dir是为turn on MMU准备的一致性映射，物理地址的高16bit都是0，因此identity mapping必定是选择TTBR0_EL1指向的各级地址翻译表。后续当系统运行之后，在进程切换的时候，会修改TTBR0的值，切换到真实的进程地址空间上去。TTBR1用于kernel space，所有的内核线程都是共享一个空间就是swapper_pg_dir。

（3）打开MMU。实际上在这条指令的上下都有isb指令，理论上已经可以turn on MMU之前之后的代码执行顺序严格的定义下来，其实我感觉不必要再启用idmap_pg_dir的那些页表了，当然，这只是猜测。

四、通向start_kernel

我痛恨汇编，如果能不使用汇编那绝对不要使用汇编，还好我们马上就要投奔start_kernel：

> \_\_mmap_switched:\
> adr_l    x6, \_\_bss_start\
> adr_l    x7, \_\_bss_stop
>
> 1:    cmp    x6, x7\
> b.hs    2f\
> str    xzr, \[x6\], #8 －－－－－－－－－－－－－－－clear BSS\
> b    1b\
> 2:\
> adr_l    sp, initial_sp, x4 －－－－－－－－－－－建立和swapper进程的链接\
> str_l    x21, \_\_fdt_pointer, x5        // Save FDT pointer\
> str_l    x24, memstart_addr, x6        // Save PHYS_OFFSET\
> mov    x29, #0\
> b    start_kernel\
> ENDPROC(\_\_mmap_switched)

这段代码分成两个部分，一部分是清BSS，另外一部分是为进入c代码做准备（主要是stack）。clear BSS段就是把未初始化的全局变量设定为0的初值，没有什么可说的。要进入start_kernel这样的c代码，没有stack可不行，那么如何设定stack呢？熟悉kernel的人都知道，用户空间的进程当陷入内核态的时候，stack切换到内核栈，实际上就是该进程的thread info内存段（4K或者8K）的顶部。对于swapper进程，原理是类似的：

> .set    initial_sp, init_thread_union + THREAD_START_SP

如果说之前的代码执行都处于一个孤魂野鬼的状态，“adr_l    sp, initial_sp, x4”指令执行之后，初始化代码终于找到了归宿，初始化代码有了自己的thread info，有了自己的task struct，有了自己的pid，有了进程（内核线程）应该拥有的一切，从此之后的代码归属idle进程，pid等于0的那个进程。

为了方便后面的代码的访问，这里还初始化了两个变量，分别是\_\_fdt_pointer（设备树信息，物理地址）和memstart_addr（kernel image所在的物理地址，一般而言是main memory的首地址）。 memstart_addr主要用于main memory中物理地址和虚拟地址的转换，具体可以参考\_\_virt_to_phys和\_\_phys_to_virt的实现。

五、参考文献

1、ARM Architecture Reference Manual

change log：

1、2015-11-30，强调了初始化代码和idle进程的连接

2、2015-12-2，修改了物理空间和虚拟空间的视图

3、2016-9-21，修改对一致性映射的描述

_原创文章，转发请注明出处。蜗窝科技_

标签: [打开MMU](http://www.wowotech.net/tag/%E6%89%93%E5%BC%80MMU)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [作業系統之前的程式 for rpi2 (1) - mmu (0) : 位址轉換](http://www.wowotech.net/linux_kenrel/address-translation.html) | [ARM64的启动过程之（三）：为打开MMU而进行的CPU初始化](http://www.wowotech.net/armv8a_arch/__cpu_setup.html)»

**评论：**

**daiyinger**\
2020-01-15 10:35

博主你好，我目前遇到个开启mmu之后处理器变慢的问题，请问可能是什么原因呢\
ldr    x1, =0xf0000000\
str    x0,\[x1\]

br    x27\
ENDPROC(\_\_enable_mmu)\
我是4核A53处理器，我在用其中一个核来引导另一个内核，如上我打算在开启mmu之后往0xf0000000物理地址写特征值表征mmu已开启，发现要等很久(几十秒)才发现0xf0000000的值被改写，如果代码放到开启mmu之前就不会。

[回复](http://www.wowotech.net/armv8a_arch/turn-on-mmu.html#comment-7839)

**daiyinger**\
2020-01-17 11:05

@daiyinger：最后发现是cache不一致的问题导致的

[回复](http://www.wowotech.net/armv8a_arch/turn-on-mmu.html#comment-7848)

**llp**\
2022-04-19 09:56

@daiyinger：请问你这个物理地址在mmu enable 之后，是如何映射的

[回复](http://www.wowotech.net/armv8a_arch/turn-on-mmu.html#comment-8589)

**see**\
2019-06-05 10:39

请教下，文档里说编译后的内核运行在0xFFFF8000_00080000?  但是通过readelf查看内核，入口点地址是0xffff000008080000，怎么和文档说法不一致？

[回复](http://www.wowotech.net/armv8a_arch/turn-on-mmu.html#comment-7457)

**[smcdef](http://www.wowotech.net/)**\
2019-06-05 17:58

@see：可能你的内核代码已经把kernel从linear mapping区域搬移到vmalloc区域了。所以造成了这种现象。详情可以看下另外一篇文档：ARM64 Kernel Image Mapping的变化：http://www.wowotech.net/memory_management/436.html

[回复](http://www.wowotech.net/armv8a_arch/turn-on-mmu.html#comment-7458)

**rikeyone**\
2018-02-28 09:29

你好，linuxer：\
我想请教一下，在start kernel之前创建的页表应该只是临时页表，那么在start kernel中的setup arch中会重新创建内核页表的映射，那么已经运行在虚拟空间上了，这个映射是怎么切换的呢？

[回复](http://www.wowotech.net/armv8a_arch/turn-on-mmu.html#comment-6571)

**rikeyone**\
2018-02-28 11:41

@rikeyone：我看的是arm32的内核操作，里面会调用pmd clear清除一级页表的内容

[回复](http://www.wowotech.net/armv8a_arch/turn-on-mmu.html#comment-6573)

**leba**\
2017-12-12 09:13

实际上在这条指令的上下都有isb指令，理论上已经可以turn on MMU之前之后的代码执行顺序严格的定义下来，其实我感觉不必要再启用idmap_pg_dir的那些页表了，当然，这只是猜测。\
------ 应该还与 pipeline 有关系，所以，那个 idmap 还是需要的。

[回复](http://www.wowotech.net/armv8a_arch/turn-on-mmu.html#comment-6342)

**[linuxer](http://www.wowotech.net/)**\
2017-12-16 09:38

@leba：你说得有道理，看来还需要再深入理解一下CPU的架构。

[回复](http://www.wowotech.net/armv8a_arch/turn-on-mmu.html#comment-6372)

**[2freeman](http://2freeman.github.io/)**\
2018-05-15 11:28

@leba：idmpa_pg_dir在MMU开启后应该还是需要的，无论是否跟pipeline有关，在MMU开启后，VA=PA=xxxx，所以，对于这样的VA，MMU的翻译还是需要使用idmap_pg_dir页表，只有在VA不再等于PA，也就是不再是一致性映射时，idmap_pg_dir页表才真正失效。而VA不等于PA的点，应该在\_\_primary_switch中：ldr x8, =\_\_primary_witched；br  x8，x8里保存的是真正的VA（！=PA），在br指令执行后，完成了VA（=PA）--> VA（！=PA）的转换，这是才是idmap_pg_dir不再起作用的时间点，这也许是这个函数为什么叫\_\_primary_witched的原因。

[回复](http://www.wowotech.net/armv8a_arch/turn-on-mmu.html#comment-6747)

**breeze**\
2017-07-26 14:19

@linuxer:

本来我以为memblock是在arm64_memblock_init()中初始化的，但我发现kernel在call arm64_memblock_init()之前就已经初始化过memblock一次了. 如：我在arm64_memblock_init()的入口处调用memblock_dump_all(), 输出如下:\
MEMBLOCK configuration:\
memory size = 0xf0000000 reserved size = 0x0\
memory.cnt  = 0x1\
memory\[0x0\]    \[0x00000010000000-0x000000ffffffff\], 0xf0000000 bytes flags: 0x0\
reserved.cnt  = 0x1\
reserved\[0x0\]    \[0x00000000000000-0xffffffffffffffff\], 0x0 bytes flags: 0x0

至少memblock.memory已经初始化过了，region为: \[0x00000010000000-0x000000ffffffff\].\
请问楼主您知道是在哪里初始化的吗?我百思不得其姐．

谢谢．

[回复](http://www.wowotech.net/armv8a_arch/turn-on-mmu.html#comment-5852)

**[wowo](http://www.wowotech.net/)**\
2017-07-28 08:56

@breeze：你可以检查一下是不是这里初始化的（dts文件中指定的memblock）：\
setup_arch\
setup_machine_fdt\
early_init_dt_scan_memory\
early_init_dt_add_memory_arch\
memblock_add

[回复](http://www.wowotech.net/armv8a_arch/turn-on-mmu.html#comment-5853)

**macooma**\
2016-10-10 14:21

博主的文章对于我这样初学armv8的晚辈来讲真是提供了极大的参考啊，感谢博主的无私精神！\
有个问题想请教一下，如下：

## idmap_pg_dir是为turn on MMU准备的一致性映射，物理地址的高16bit都是0，因此identity mapping必定是选择TTBR0_EL1指向的各级地址翻译表。

请问这里的“必定”是因何而来呢？ 谢谢！

[回复](http://www.wowotech.net/armv8a_arch/turn-on-mmu.html#comment-4683)

**[linuxer](http://www.wowotech.net/)**\
2016-10-10 21:49

@macooma：identity mapping实际上是把 物理地址映射到物理地址上去，ARMv8支持的物理地址最大是48个bit，因此高16bit必定等于0

[回复](http://www.wowotech.net/armv8a_arch/turn-on-mmu.html#comment-4687)

**macooma**\
2016-11-08 17:12

@linuxer：意思是CPU会主动判断要转换的地址前16位来决定使用TTBR0还是TTBR1吧？

[回复](http://www.wowotech.net/armv8a_arch/turn-on-mmu.html#comment-4852)

**thatman**\
2019-03-21 14:49

@macooma：请问下。这个问题有答案了吗？

[回复](http://www.wowotech.net/armv8a_arch/turn-on-mmu.html#comment-7331)

**[archer](http://www.wowotech.net/)**\
2016-07-31 14:25

linuxer，你好，请教一个问题：\
PAGE_OFFSET开始的虚拟地址 会 映射到PHYS_OFFSET的物理地址 （线性区），\
PHYS_OFFSET的定义：\
#define PHYS_OFFSET             ({ VM_BUG_ON(memstart_addr & 1); memstart_addr; })

这个memstart_addr的全局变量是在arm64_memblock_init这个函数中计算确定的。\
所以线性区的起始物理地址是在这里才确定的？

我的问题：因为内核的虚拟地址在PAGE_OFFSET的0x80000偏移处，所以Image物理地址也应该在memstart_addr的0x80000偏移处？ 但是memstart_addr是在这里才计算确定的，bootloader是怎么知道要把Image加载到这里的呢？

[回复](http://www.wowotech.net/armv8a_arch/turn-on-mmu.html#comment-4353)

**[linuxer](http://www.wowotech.net/)**\
2016-08-01 10:23

@archer：这个memstart_addr的全局变量是在arm64_memblock_init这个函数中计算确定的。\
所以线性区的起始物理地址是在这里才确定的？\
－－－－－－－－－－－－－－－－－－－\
我看的是4.4.6内核，memstart_addr是在内核启动的初始阶段被设定的：\
ENTRY(stext)\
......\
adrp    x24, \_\_PHYS_OFFSET\
......\
之后，在\_\_mmap_switched函数中：    \
str_l    x24, memstart_addr, x6        // Save PHYS_OFFSET\
mov    x29, #0

因为内核的虚拟地址在PAGE_OFFSET的0x80000偏移处，所以Image物理地址也应该在memstart_addr的0x80000偏移处？\
－－－－－－－－－－－－－－\
是的

但是memstart_addr是在这里才计算确定的，bootloader是怎么知道要把Image加载到这里的呢？\
－－－－－－－－－－－－－－－\
这些内容本来就是bootloader和kernel之间接口的一部分，bootloader本来就知道应该copy kernel image到哪里的，如果不知道，系统就没有办法运作了。

[回复](http://www.wowotech.net/armv8a_arch/turn-on-mmu.html#comment-4356)

**[ericzhou](http://www.wowotech.net/)**\
2016-08-01 22:05

@linuxer：linuxer，thanks for your reply.\
我看得是4.6的内核，memstart_addr的计算确定过程有点变化。\
是在arm64_memblock_init这个函数里确定的。\
不过你的意思我明白了。谢谢。

[回复](http://www.wowotech.net/armv8a_arch/turn-on-mmu.html#comment-4364)

**tgn**\
2016-05-21 01:09

请教下博主，内核正式页表中的PXN位一般在哪里设置呀，我看arm64代码里面默认是没有的，能直接在create_mapping函数调用的时候直接把PAGE_KERNEL_EXEC给或上一个PXN吗？\
我是用的4.5的内核

[回复](http://www.wowotech.net/armv8a_arch/turn-on-mmu.html#comment-3963)

**[linuxer](http://www.wowotech.net/)**\
2016-05-23 17:23

@tgn：为何要给PAGE_KERNEL_EXEC加上一个PXN的flag？PAGE_KERNEL_EXEC应该对应内核可执行代码的那些memory，因此创建页表的时候没有PXN是合适的，如果置位PXN，那么你的意思是准备让它在特权模式下永远不执行（Privileged execute-never）？

[回复](http://www.wowotech.net/armv8a_arch/turn-on-mmu.html#comment-3965)

**[amusion](http://www.wowotech.net/)**\
2015-10-28 11:25

无意中看到了这个网站，拜读了几篇文章后，真是很钦佩啊，文章分析的很深入，不知能否写些和SMP先关的分析

[回复](http://www.wowotech.net/armv8a_arch/turn-on-mmu.html#comment-2873)

**[linuxer](http://www.wowotech.net/)**\
2015-10-29 08:59

@amusion：这位客官，本站暂时不接受“点菜”，呵呵～～～开玩笑的，大家工作都很忙，业余时间写写文章，让自己爽一下，所以，想到哪里写到哪里，SMP的代码分布在各个内核的子系统中，其实不是很好写的。

[回复](http://www.wowotech.net/armv8a_arch/turn-on-mmu.html#comment-2887)

**[kitty](http://blog.csdn.net/chongyang198999)**\
2015-10-27 17:31

博主真的是辛苦了，像博主这样静下心搞钻研，如此认真隐忍的，太少了。

[回复](http://www.wowotech.net/armv8a_arch/turn-on-mmu.html#comment-2861)

**[linuxer](http://www.wowotech.net/)**\
2015-10-27 18:30

@kitty：不辛苦，如果真心热爱的话就不辛苦，^\_^

喜欢钻研的人很多，只是没有聚合在一起，蜗窝这个网站就是因此而设立的，欢迎每一个沉醉于技术的人。

[回复](http://www.wowotech.net/armv8a_arch/turn-on-mmu.html#comment-2864)

**[kitty](http://www.wowotech.net/linux_kenrel/turn-on-mmu.html#comment-2864)**\
2015-10-30 10:38

@linuxer：hi linuxer，看你写的power相关文章，写的非常详细。但power相关架构，很多要进行调整了，在今年年底，linuro要发布一个新的调度器架构EAS，会把DVFS和cpu idle添加CFS调度器中，而themal也会被IPA机制取代，这是新的研究方向，感兴趣的话，可以一起学习。

[回复](http://www.wowotech.net/armv8a_arch/turn-on-mmu.html#comment-2895)

1 [2](http://www.wowotech.net/armv8a_arch/turn-on-mmu.html/comment-page-2#comments)

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

  - [X-006-UBOOT-pinctrl driver移植(Bubblegum-96平台)](http://www.wowotech.net/x_project/bubblegum_uboot_pinctrl.html)
  - [Linux kernel内核配置解析(5)\_Boot options(基于ARM64架构)](http://www.wowotech.net/linux_kenrel/kernel_config_boot_option.html)
  - [u-boot FIT image介绍](http://www.wowotech.net/u-boot/fit_image_overview.html)
  - [eMMC框架及其初始化](http://www.wowotech.net/filesystem/329.html)
  - [X-007-UBOOT-DDR的初始化(Bubblegum-96平台)](http://www.wowotech.net/x_project/bubblegum_uboot_ddr.html)

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

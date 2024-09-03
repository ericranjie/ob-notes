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

作者：[smcdef](http://www.wowotech.net/author/531) 发布于：2018-5-6 10:00 分类：[内存管理](http://www.wowotech.net/sort/memory_management)

## 引言

什么是KASLR？KASLR是kernel address space layout randomization的缩写，直译过来就是内核地址空间布局随机化。KASLR技术允许kernel image加载到VMALLOC区域的任何位置。当KASLR关闭的时候，kernel image都会映射到一个固定的链接地址。对于黑客来说是透明的，因此安全性得不到保证。KASLR技术可以让kernel image映射的地址相对于链接地址有个偏移。偏移地址可以通过dts设置。如果bootloader支持每次开机随机生成偏移数值，那么可以做到每次开机kernel image映射的虚拟地址都不一样。因此，对于开启KASLR的kernel来说，不同的产品的kernel image映射的地址几乎都不一样。因此在安全性上有一定的提升。

> 注：文章代码分析基于linux-4.15，架构基于aarch64（ARM64）。

## 如何使用

打开KASLR功能非常简单，在支持KASLR的内核配置选项添加选项CONFIG_RANDOMIZE_BASE=y。同时还需要告知kernel映射的偏移地址，通过dts传递。在chosen节点下添加kaslr-seed属性。属性值就是偏移地址。另外command line不要带nokaslr，否则KASLR还是关闭。dts信息举例如下。顺便说一下，在dts中`<>`符号中是一个32 bit的值。但是在ARM64平台，这里的`kaslr-seed`属性是一个特例，他就是一个64 bit的值。

1. / {
2.     chosen {
3.     	kaslr-seed = <0x10000000>;
4.     };
5. }; 

## 如何获取偏移

kaslr-seed属性的解析在kaslr_early_init函数完成。该函数根据输入参数dtb首地址（物理地址）解析dtb，找到偏移地址，最后返回。kaslr_early_init实现如下。

1. u64 __init kaslr_early_init(u64 dt_phys)
2. {
3. 	void *fdt;
4. 	u64 seed, offset, mask, module_range;
5. 	const u8 *cmdline, *str;
6. 	int size;

8. 	early_fixmap_init();                                         /* 1 */
9. 	fdt = __fixmap_remap_fdt(dt_phys, &size, PAGE_KERNEL);       /* 1 */

11. 	seed = get_kaslr_seed(fdt);                                  /* 2 */
12. 	if (!seed)
13. 		return 0;

15. 	cmdline = get_cmdline(fdt);
16. 	str = strstr(cmdline, "nokaslr");                            /* 3 */
17. 	if (str == cmdline || (str > cmdline && *(str - 1) == ' '))
18. 		return 0;

20. 	mask = ((1UL << (VA_BITS - 2)) - 1) & ~(SZ_2M - 1);          /* 4 */
21. 	offset = seed & mask;

23. 	/* use the top 16 bits to randomize the linear region */
24. 	memstart_offset_seed = seed >> 48;                           /* 5 */

26. 	if ((((u64)_text + offset) >> SWAPPER_TABLE_SHIFT) !=
27. 	    (((u64)_end + offset) >> SWAPPER_TABLE_SHIFT))
28. 		offset = round_down(offset, 1 << SWAPPER_TABLE_SHIFT);   /* 6 */

30. 	return offset;
31. } 

> 1. 由于dtb的地址是物理地址，因此第一步先为dtb区域建立映射。
> 2. 从dtb文件获取kaslr-seed属性的值。
> 3. 确保command line没有传递nokaslr参数，如果传递nokaslr则关闭KASLR。
> 4. 保证传递的偏移地址2M地址对齐，并且保证kernel位于VMALLOC区域大小的一半地址空间以下 (VA_BITS - 2)。当VA_BITS=48时，mask=0x0000_3fff_ffe0_0000。
> 5. 线性映射区地址也会随机化。
> 6. kernel启动初期只有一个PUD页表，因此希望kernel映射在1G（1 << SWAPPER_TABLE_SHIFT）大小范围内，这样就不用两个PUD页表。如果kernel加上偏移offset后不满足这点，自然要重新对齐。

## 如何创建映射

kernel启动初期在汇编阶段创建映射关系。代码位于head.S文件。在__primary_switched函数中会调用kaslr_early_init得到偏移地址。保存在x23寄存器中。然后重新创建kernel image的映射。

1. __primary_switched:
2. 	tst	x23, ~(MIN_KIMG_ALIGN - 1)	// already running randomized?
3. 	b.ne	0f
4. 	mov	x0, x21				        // pass FDT address in x0
5. 	bl	kaslr_early_init		    // parse FDT for KASLR options
6. 	cbz	x0, 0f				        // KASLR disabled? just proceed
7. 	orr	x23, x23, x0			    // record KASLR offset
8. 	ldp	x29, x30, [sp], #16		    // we must enable KASLR, return
9. 	ret					            // to __primary_switch() 

创建映射的函数是__create_page_tables。

```c
__create_page_tables:    /*     * Map the kernel image.     */    adrp	x0, swapper_pg_dir    mov_q	x5, KIMAGE_VADDR + TEXT_OFFSET  // compile time __va(_text)    add	x5, x5, x23                         // add KASLR displacement    create_pgd_entry x0, x5, x3, x6    adrp	x6, _end                        // runtime __pa(_end)    adrp	x3, _text                       // runtime __pa(_text)    sub	x6, x6, x3                          // _end - _text    add	x6, x6, x5                          // runtime __va(_end)    create_block_map x0, x7, x3, x5, x6
```

这段代码在我的另一篇文章[《ARM64 Kernel Image Mapping的变化》](http://www.wowotech.net/memory_management/436.html)已经有分析过，这里就略过了。注意第7行，kernel image映射的虚拟地址加上了一个偏移地址x23。还有一点需要说明，就是对重定位段进行重定位。这部分代码在`arch/arm64/kernel/head.S`文件`__relocate_kernel`函数实现。大概说下`__relocate_kernel`有什么用呢！例如链接脚本中常见的几个变量`_text`、`_etext`、`_end`。这几个你应该很熟悉，他们是一个地址并且他们的值是链接的时候确定下来，那么现在使能kaslr的情况下，代码中再访问`_text`的值就很明显不是运行时的虚拟地址，而是链接时候的值。因此，`__relocate_kernel`函数可以负责重定位这些变量。保证访问这些变量的值依然是正确的值。这部分涉及编译和链接，有兴趣的可以研究一下《程序员的自我修养》这本书（我不太熟悉）。

## addr2line怎么办

KASLR在技术上增加了OS安全性，但是对于调试或许增加了些难度。何以见得呢？首先，我们知道编译kernel的时候链接地址和最终运行地址是不一样的，因此如果发生oops，栈的回溯信息中的函数地址其实都是运行地址。如果你使用过addr2line工具的话，应该不会陌生`addr2line -e vmlinux 0xffffff8000080000`这条命令获取某个地址在代码中的哪一行。那么现在问题是oops中的地址和链接地址有一个偏差，并且这个偏差随着bootloader传递的值而变化。现在摆在我们眼前的是addr2line工具还怎么用？下面就为你答疑解惑。kernel开机log中会打印Virtual kernel memory layout。举例如下。

```c
 
Virtual kernel memory layout:  modules : 0xffffff8000000000 - 0xffffff8008000000   (   128 MB)  vmalloc : 0xffffff8008000000 - 0xffffffbebfff0000   (   250 GB)    .text : 0xffffff80ae280000 - 0xffffff80af2e0000   ( 16768 KB)  .rodata : 0xffffff80af300000 - 0xffffff80afb60000   (  8576 KB)    .init : 0xffffff80afb60000 - 0xffffff80b01e0000   (  6656 KB)    .data : 0xffffff80b01e0000 - 0xffffff80b044f200   (  2493 KB)     .bss : 0xffffff80b044f200 - 0xffffff80b0e18538   ( 10021 KB)  fixed   : 0xffffffbefe7fb000 - 0xffffffbefec00000   (  4116 KB)  PCI I/O : 0xffffffbefee00000 - 0xffffffbeffe00000   (    16 MB)  vmemmap : 0xffffffbf00000000 - 0xffffffc000000000   (     4 GB maximum)            0xffffffbf00000000 - 0xffffffbf03000000   (    48 MB actual)  memory  : 0xffffffc000000000 - 0xffffffc0c0000000   (  3072 MB)
```

注意看以上.text区域（kernel代码段）起始地址和结束地址是不是位于VMALLOC区域。如果发生oops，log中函数的地址必然是一个位于.text段的地址，假设是addr_run，但是链接地址应该是KIMAGE_VADDR + TEXT_OFFSET，这两个宏定义参考这篇文章[《ARM64 Kernel Image Mapping的变化》](http://www.wowotech.net/memory_management/436.html)。在这个例子中，KIMAGE_VADDR = 0xffffff8008000000，TEXT_OFFSET = 0x80000。addr2line工具使用的必须是链接地址，因此需要将addr_run转换成链接地址。公式很容易推导出来，addr_link = addr_run - .text_start + vmalloc_start + TEXT_OFFSET。在这个例子中就是addr_link = addr_run - 0xffffff80ae280000 + 0xffffff8008000000 + 0x80000。计算的addr_link就可以使用addr2line工具解析了。Have fun！

标签: [kaslr](http://www.wowotech.net/tag/kaslr)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Linux中常见同步机制设计原理](http://www.wowotech.net/kernel_synchronization/445.html) | [fixmap addresses原理](http://www.wowotech.net/memory_management/440.html)»

**评论：**

**Enlin**  
2023-11-14 18:38

在4.15中，我觉得节点信息应该是：kaslr-seed = <0x10000000 0x10000000>;这类格式的。  
  
因为下面有如下的判断：  
if (!prop || len != sizeof(u64))  
  
节点长度不够，直接return 0；

[回复](http://www.wowotech.net/memory_management/441.html#comment-8846)

**悟实**  
2021-10-14 11:24

请教wowo大佬:  
关于线性映射区地址也会随机化。当前调试kernel-4.14 arm64, 驱动里面kmalloc申请的buf, buf前后连续地址一直是随机的, 关闭CONFIG_RANDOMIZE_BASE之后还是随机的, 如何分析这个随机化, 感谢

[回复](http://www.wowotech.net/memory_management/441.html#comment-8333)

**zxq**  
2022-01-13 19:15

@悟实：你这个主要原因是因为slub使能了CONFIG_SLAB_FREELIST_RANDOM，跟CONFIG_RANDOMIZE_BASE没关系。

[回复](http://www.wowotech.net/memory_management/441.html#comment-8488)

**[蜗牛](http://na/)**  
2021-04-21 09:47

您好，窝窝  
  
请问arm平台有对kaslr机制的处理的，kernel414上好像没有

[回复](http://www.wowotech.net/memory_management/441.html#comment-8220)

**chenlifu2015**  
2020-05-28 16:23

博主，您好  
  请教一个问题，就算开启了KASLR（没有设置kaslr_seed。偏移应该为0）和内核代码段被映射到vmalloc区域，内核代码段的hash值应该还是不会改变的吧，我现在通过sha256去计算[_stext, _etext]范围的hash值，在内核的加载过程中会一直变化，难道内核代码段也会发生缺页中断么，我的内核版本和博主您的是一样的：4.14 ARM64

[回复](http://www.wowotech.net/memory_management/441.html#comment-7999)

**[smcdef](http://www.wowotech.net/)**  
2020-05-28 17:13

@chenlifu2015：我不明白你提出缺页异常和代码段变化有什么关系。当然内核的代码也不会发生缺页异常。你好奇的问题应该是代码段为什么hash值变化了？代码段发生变化，也和KASLR没有关系。我这里给你提供一点思路证明代码段确实可以发生变化。  
  
1. aarch64会有一些CPU的feature，这些feature会在系统boot的时候动态监测。如果某些feature在当前CPU是支持的，此时会有指令替换的过程，把新feature的指令替换编译期的指令。  
2. 类似kprobe技术，会修改代码段达到hook函数的目的。  
  
这些都可能导致代码段的hash值变化。

[回复](http://www.wowotech.net/memory_management/441.html#comment-8000)

**chenlifu2015**  
2020-05-29 16:24

@smcdef：感谢博主的回复，我今天去了解了一下kprobe机制，确实可能会导致内核代码段被篡改，但您说的第一点我确实没找到相关资料，能方便一点一下么，比如说一个具体的机制我能够去熟悉，或许这个动态监测的机制能够通过配置宏关闭

[回复](http://www.wowotech.net/memory_management/441.html#comment-8002)

**[smcdef](http://www.wowotech.net/)**  
2020-05-29 21:15

@chenlifu2015：例如atomic实现有2种方式，LSE和LL/SC。印象中LSE是armv8.2以后支持的特性。LL/SC是老的实现方式。你看看atomic实现的2个文件吧。

[回复](http://www.wowotech.net/memory_management/441.html#comment-8003)

**chenlifu2015**  
2020-06-01 11:05

@smcdef：我看了下我使用的SDK中内核关于atomic的代码实现，可以确定这部分是通过编译宏来决定的，并且kprobe特性在编译时也没有被打开，因此可以排除kprobe和LSE的原因导致内核代码段hash发生变化，博主，还有其它的思路么，因为我在4.12的arm32位上计算内核hash是不会发生变化的，也就是说是可以关闭掉cpu feature带来的影响，博主，我猜测了一下，这种动态检测的特性能否通过配置宏来关闭呢

[回复](http://www.wowotech.net/memory_management/441.html#comment-8007)

**[smcdef](http://www.wowotech.net/)**  
2020-06-01 14:32

@chenlifu2015：我让你看的目的不是为了让你确认是不是宏确认编译。而是让你看怎么根据CPU feature检测，动态替换代码的。默认情况下本身就是宏唯一确定了使用LL/SC的方式，只不过boot的时候会检测feature，如果支持LSE，就会把LSE的指令替换了LL/SC的方式。你通过编译是无法确定的。  
另外你在arm32上测试有什么意义呢？CPU feature检测替换这块是arm64实现的，arm32是否实现了类似机制都不确定，你的测试能说明什么呢？  
你应该去看看这个宏定义的作用  
#define ALTERNATIVE(oldinstr, newinstr, ...)

[回复](http://www.wowotech.net/memory_management/441.html#comment-8008)

**TTTT**  
2019-09-25 16:02

@smcdef，你好，感谢分享。有问题请教：  
  
kaslr_early_init返回了offset，__create_page_tables根据offset去重新创建映射。我没搞明白，如何实现的随机化。  
  
多谢！

[回复](http://www.wowotech.net/memory_management/441.html#comment-7665)

**[smcdef](http://www.wowotech.net/)**  
2019-09-25 21:05

@TTTT：kaslr_early_init() 返回的 offset 得值是 BootLoader 通过 dts 传递过来的。所以，随机产生 offset 的工作由 BootLoader 完成。

[回复](http://www.wowotech.net/memory_management/441.html#comment-7666)

**[yiyeguzhou100](http://blog.csdn.net/yiyeguzhou100)**  
2019-02-19 19:20

感谢作者写了这么好的文档， 我在们的产品上用的arm64架构，  我发现CONFIG_RANDOMIZE_BASE配置了（CONFIG_RANDOMIZE_BASE=y），CONFIG_ARM64_RANDOMIZE_TEXT_OFFSET没有配置（is not set）， 我编译个若干次内核进行测试， 发现kernel image会被load到固定的位置（0xffff000010080000~0xffff000010c95000）, 其中vmalloc区域的地址是0xffff000010000000~0xffff7bffbfff0000。  
这样看来即使配置了CONFIG_RANDOMIZE_BASE也没有实现kernel image会被随机copy到不同的位置。 反倒时我觉得配置了CONFIG_ARM64_RANDOMIZE_TEXT_OFFSET才能实现kernel image被随机copy到不同的位置。 因CONFIG_ARM64_RANDOMIZE_TEXT_OFFSET会决定TEXT_OFFSET的值。

[回复](http://www.wowotech.net/memory_management/441.html#comment-7192)

**[smcdef](http://www.wowotech.net/)**  
2019-02-21 11:58

@yiyeguzhou100：当你配置CONFIG_ARM64_RANDOMIZE_TEXT_OFFSET后，你就会发现每次开机后，kernel image 映射开始地址是固定不变的。你看到现象虽然变了，但是这并不是kaslr ，kaslr 的目的是每次开机都不一样。实际上这个选项是配置编译时期使TEXT_OFFSET随机值。不配置该选项时，默认TEXT_OFFSET = 0x00080000。所以这个选项其实和kaslr 无关。该配置选项实际上还是链接地址等于最终的运行地址。但是kaslr 却不想等。  
你配置CONFIG_RANDOMIZE_BASE后，kaslr 没有起效。可能原因文章提到的dts kaslr-seed没有。

[回复](http://www.wowotech.net/memory_management/441.html#comment-7198)

**qifeifly**  
2019-04-27 19:14

@smcdef：请问开启kaslr后，kernel image被随机映射在vmalloc区，链接地址和运行地址不相等，若遇到位置相关码如何处理？

[回复](http://www.wowotech.net/memory_management/441.html#comment-7378)

**[smcdef](http://www.wowotech.net/)**  
2019-04-29 16:12

@qifeifly：所以打开kaslr后，会做重定位操作，否则不能运行啊

[回复](http://www.wowotech.net/memory_management/441.html#comment-7383)

**nicole**  
2019-06-11 15:29

@smcdef：大师您好  
    请问一下内核如果配置成了可重定位内核，那么内核是如何通过vmlinux.relocs这个重定位表进行重定位的？  
  
我的理解是内核会遍历vmlinux.relocs ，在每个需要修订的位置，加上内核实际加载地址和理论加载地址的差值，实现重定位；但是这只是改变了vmlinux.relocs,并没有改变位置相关码呀？  
  
请大师指点一二，谢谢。

[回复](http://www.wowotech.net/memory_management/441.html#comment-7464)

**[smcdef](http://www.wowotech.net/)**  
2019-06-14 09:48

@nicole：以ARM64架构为例，relocate 实现代码位于：arch/arm64/kernel/head.S __relocate_kernel() 函数。  
  
__relocate_kernel:  
    /*  
     * Iterate over each entry in the relocation table, and apply the  
     * relocations in place.  
     */  
    ldr    w9, =__rela_offset        // offset to reloc table  
    ldr    w10, =__rela_size        // size of reloc table  
  
    mov_q    x11, KIMAGE_VADDR        // default virtual offset  
    add    x11, x11, x23            // actual virtual offset  
    add    x9, x9, x11            // __va(.rela)  
    add    x10, x9, x10            // __va(.rela) + sizeof(.rela)  
  
0:    cmp    x9, x10  
    b.hs    1f  
    ldp    x11, x12, [x9], #24  
    ldr    x13, [x9, #-8]  
    cmp    w12, #R_AARCH64_RELATIVE  
    b.ne    0b  
    add    x13, x13, x23            // relocate  
    str    x13, [x11, x23]  
    b    0b  
1:    ret  
ENDPROC(__relocate_kernel)  
  
__rela_offset和__rela_size在链接脚本(arch/arm64/kernel/vmlinux.lds.S)中可以找到。你可以看下这部分实现。

[回复](http://www.wowotech.net/memory_management/441.html#comment-7469)

**[zhangge0409](http://www.wowotech.net/)**  
2018-11-03 21:52

请教下博主，我想查看一个程序在运行时的符号表信息，比如程序为app1，  
使用nm app1查看的符号表和app1在运行时的符号表地址信息不一致（应该是链接脚本重定位地址）  
如何才能看到程序的符号表和运行时一致呢？？

[回复](http://www.wowotech.net/memory_management/441.html#comment-7012)

**天津网站建设**  
2018-09-06 22:09

感谢博主，写得真好

[回复](http://www.wowotech.net/memory_management/441.html#comment-6934)

**litao6169**  
2018-08-06 16:15

addr_link = addr_run - 0xffffff80ae280000 + 0xffffff8008000000 + 0x80000 等同于：  
addr_link = addr_run - （0xffffff80ae280000 - 0xffffff8008000000 - 0x80000）  
而（0xffffff80ae280000 - 0xffffff8008000000 - 0x80000）实际就是karsl-seed 偏移地址。

[回复](http://www.wowotech.net/memory_management/441.html#comment-6870)

**[hymmsx](http://www.wowotech.net/)**  
2023-03-08 19:48

@litao6169：你这个结果计算出来是A6200000和他设的A6200000不一样？

[回复](http://www.wowotech.net/memory_management/441.html#comment-8749)

**[hymmsx](http://www.wowotech.net/)**  
2023-03-08 19:49

@litao6169：你这个结果计算出来是A6200000和他设的A6200000不一样？

[回复](http://www.wowotech.net/memory_management/441.html#comment-8750)

**hzm**  
2018-05-18 11:28

请问下，开了KASLR后， kernel image 加载到VMALLOC是随机变化的， 加载到物理地址还是固定的吗？  
因为如果不是固定，bootloader那边加载image也需要偏移的信息  
  
谢谢。

[回复](http://www.wowotech.net/memory_management/441.html#comment-6754)

**[smcdef](http://www.wowotech.net/)**  
2018-05-18 19:25

@hzm：物理地址还是以前一样，改变的只是虚拟地址而已，物理地址在哪其实没什么影响，创建的是虚拟地址到物理地址的映射关系

[回复](http://www.wowotech.net/memory_management/441.html#comment-6756)

**hzm**  
2018-05-19 11:24

@smcdef：多谢解答。。另外这个过程可以理解成静态地址重定位？  
还有的是不大理解 为什么要0x80000这个偏移，没KASLR的版本是为了预留一段空间用来fix mapping吗? 那打开了KASLR似乎不需要这个偏移?

[回复](http://www.wowotech.net/memory_management/441.html#comment-6757)

**[smcdef](http://www.wowotech.net/)**  
2018-05-19 16:15

@hzm：针对ARM32架构，0x80000偏移之前的物理地址是特殊用途，例如页表。但是针对ARM64架构，好像没什么作用。你说的fix mapping，可以参考另一篇fix mapping的文章。

[回复](http://www.wowotech.net/memory_management/441.html#comment-6758)

**hzm**  
2018-05-20 23:08

@smcdef：TEXT_OFFSET放的是哪个页表？看了窝窝内存初始化【上】这篇，说swapper进程放在bss段以后

[回复](http://www.wowotech.net/memory_management/441.html#comment-6760)

**[smcdef](http://www.wowotech.net/)**  
2018-05-24 22:27

@hzm：注意区分ARM32和ARM64的区别

[回复](http://www.wowotech.net/memory_management/441.html#comment-6763)

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
    
    - ja  
        [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
    - 元神高手  
        [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
    - 十七  
        [内核空间的映射在系统启动时就已经设定好，并且在所有进程的页表...](http://www.wowotech.net/process_management/context-switch-arch.html#8920)
    - lw  
        [sparse模型和disconti模型没看出来有什么本质区别...](http://www.wowotech.net/memory_management/memory_model.html#8919)
    - 肥饶  
        [一个没设置好就出错](http://www.wowotech.net/linux_kenrel/516.html#8918)
    - orange  
        [点赞点赞，对linuxer的文章总结到位](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html#8917)
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
    
    - [开源的RF硬件平台](http://www.wowotech.net/75.html)
    - [Linux TTY framework(2)_软件架构](http://www.wowotech.net/tty_framework/tty_architecture.html)
    - [Linux时间子系统之（十七）：ARM generic timer驱动代码分析](http://www.wowotech.net/timer_subsystem/armgeneraltimer.html)
    - [linux kernel内存碎片防治技术](http://www.wowotech.net/memory_management/memory-fragment.html)
    - [蓝牙协议分析(6)_BLE地址类型](http://www.wowotech.net/bluetooth/ble_address_type.html)
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
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

作者：[smcdef](http://www.wowotech.net/author/531) 发布于：2018-4-21 20:25 分类：[内存管理](http://www.wowotech.net/sort/memory_management)

## 引言

随着linux的代码更新，阅读linux-4.15代码，从中发现很多与众不同的地方。之所以与众不同，就是因为和我之前从网上博客或者书籍中看到的内容有所差异。当然了，并不是为了表明书上或者博客的观点是错误的。而是因为linux代码更新的太快，网上的博客和书籍跟不上linux的步伐而已。究竟是哪些发生了差异了？例如：kernel image映射区域从原来的linear mapping region（线性映射区域）搬移到VMALLOC区域。因此，我希望通过本篇文章揭晓这些差异。当然，我相信不久的将来这篇文章也将会成为一段历史。

> 注：文章代码分析基于linux-4.15，架构基于aarch64（ARM64）。涉及页表代码分析部分，假设页表映射层级是4，即配置CONFIG_ARM64_PGTABLE_LEVELS=4。地址宽度是48，即配置CONFIG_ARM64_VA_BITS=48。

## kernel启动页表在哪里

在ARM64架构上，汇编代码初始化阶段会创建两次地址映射。第一次是为了打开MMU操作的准备。因为再打开MMU之前，当前代码运行在物理地址之上，而打开MMU之后代码运行在虚拟地址之上。为了从物理地址（Physical Address，简称PA）转换到虚拟地址（Virtual Address，简称VA）的平滑过渡，ARM推荐创建VA和PA相等的一段映射（例如：虚拟地址addr通过页表查询映射的物理地址也是addr）。这段映射在linux中称为identity mapping。第二次是kernel image映射。而这段映射在linux-4.15代码上映射区域是VMALLOC区域。

kernel启动开始首先就会打开MMU，但是打开MMU之前，我们需要填充页表。也就是告诉MMU虚拟地址和物理地址的对应关系。系统启动初期使用section mapping，因此需要3个页面存储页表项。但是我们有identity mapping和kernel image mapping，因此总需要6个页面。那么这6个页面内存是在哪里分配的呢？可以从vmlinux.lds.S中找到答案。

1. BSS_SECTION(0, 0, 0)

3. . = ALIGN(PAGE_SIZE);
4. idmap_pg_dir = .;
5. . += IDMAP_DIR_SIZE;
6. swapper_pg_dir = .;
7. . += SWAPPER_DIR_SIZE; 

从链接脚本中可以看到预留6个页面存储页表项。紧跟在bss段后面。idmap_pg_dir是identity mapping使用的页表。swapper_pg_dir是kernel image mapping初始阶段使用的页表。请注意，这里的内存是一段连续内存。也就是说页表（PGD/PUD/PMD）都是连在一起的，地址相差PAGE_SIZE（4k）。

## 如何填充页表的页表项

从链接脚本vmlinux.lds.S文件中可以找到kernel代码起始代码段是".head.text"段，因此kernel的代码起始位置位于arch/arm64/kernel/head.S文件`_head`标号。在head.S文件中有三个宏定义和创建地址映射相关。分别是：`create_table_entry`、`create_pgd_entry`和`create_block_map`。

create_table_entry实现如下。

1. /*
2.  * Macro to create a table entry to the next page.
3.  *
4.  *	tbl:	页表基地址
5.  *	virt:	需要创建地址映射的虚拟地址
6.  *	shift:	#imm page table shift
7.  *	ptrs:	#imm pointers per table page
8.  *
9.  * Preserves:	virt
10.  * Corrupts:	tmp1, tmp2
11.  * Returns:	tbl -> next level table page address
12.  */
13. 	.macro	create_table_entry, tbl, virt, shift, ptrs, tmp1, tmp2
14. 	lsr	\tmp1, \virt, #\shift
15. 	and	\tmp1, \tmp1, #\ptrs - 1	    // table index
16. 	add	\tmp2, \tbl, #PAGE_SIZE
17. 	orr	\tmp2, \tmp2, #PMD_TYPE_TABLE	// address of next table and entry type
18. 	str	\tmp2, [\tbl, \tmp1, lsl #3]
19. 	add	\tbl, \tbl, #PAGE_SIZE		    // next level table page
20. 	.endm 

这里是汇编中的宏定义。汇编中宏定义是以`.macro`开头，以`.endm`结尾。宏定义中以`\x`来引用宏定义中的参数`x`。该宏定义的作用是创建一个level的页表项（PGD/PUD/PMD）。具体是哪个level是由virt、shift和ptrs参数决定。我总是喜欢帮你翻译成C语言的形式。C语言如果不懂的话，我也没办法了。既然汇编你不熟悉，没关系，下面帮你转换成C语言的宏定义。

1. #define PAGE_SIZE            (1 << 12)
2. #define PMD_TYPE_TABLE       (3 << 0)

4. #define create_table_entry(tbl, virt, shift, ptrs, tmp1, tmp2) do { \
5. 		tmp1 = virt >> shift;                     /* 1 */           \
6. 		tmp1 &= ptrs - 1;                         /* 1 */           \
7. 		tmp2 = tbl + PAGE_SIZE;                   /* 2 */           \
8. 		tmp2 |= PMD_TYPE_TABLE;                   /* 3 */           \
9. 		*((long *)(tbl + (tmp1 << 3))) = tmp2;    /* 4 */           \
10. 		tbl += PAGE_SIZE;                         /* 5 */           \
11. 	} while (0) 

> 1. 根据virt和ptrs参数计算该虚拟地址virt的页表项在页表中的index。例如计算virt地址在PGD也表中的indedx，可以传递shift = PGDIR_SHIFT，ptrs = PTRS_PER_PGD，tbl传递PGD页表基地址。所以，宏定义是一个创建中间level的页表项。
> 2. 既然要填充当前level的页表项就需要告知下一个level页表的基地址，这里就是计算下一个页表的基地址。还记得上面说的idmap_pg_dir和swapper_pg_dir吗？页表（PGD/PUD/PMD）都是连在一起的，地址相差PAGE_SIZE。
> 3. 告知MMU这是一个中间level页表并且是有效的。
> 4. 页表项的真正填充操作，tmp1 << 3是因为ARM64的地址占用8bytes。
> 5. 更新tbl，也就只指向下一个level页表的地址，可以方便再一次调用create_table_entry填充下一个level页表项而不用自己更新tbl。

create_pgd_entry的实现如下。

1. /*
2.  * Macro to populate the PGD (and possibily PUD) for the corresponding
3.  * block entry in the next level (tbl) for the given virtual address.
4.  *
5.  * Preserves:	tbl, next, virt
6.  * Corrupts:	tmp1, tmp2
7.  */
8. 	.macro	create_pgd_entry, tbl, virt, tmp1, tmp2
9. 	create_table_entry \tbl, \virt, PGDIR_SHIFT, PTRS_PER_PGD, \tmp1, \tmp2
10. 	create_table_entry \tbl, \virt, SWAPPER_TABLE_SHIFT, PTRS_PER_PTE, \tmp1, \tmp2
11. 	.endm 

create_pgd_entry可以用来填充PGD、PUD、PMD等中间level页表对应页表项。虽然名字是创建PGD的描述符，但是实际上是一级一级的创建页表项，最终只留下最后一级页表没有填充页表项。老规矩转换成C语言分析。

  

1. #define SWAPPER_TABLE_SHIFT	PUD_SHIFT

3. #define create_pgd_entry(tbl, virt, tmp1, tmp2) do {                                          \
4. 		create_table_entry(tbl, virt, PGDIR_SHIFT, PTRS_PER_PGD, tmp1, tmp2);         /* 1 */ \
5. 		create_table_entry(tbl, virt, SWAPPER_TABLE_SHIFT, PTRS_PER_PTE, tmp1, tmp2); /* 2 */ \
6. 	} while (0) 

  

> 1. 这里的tbl参数相当于PGD页表地址，调用create_table_entry创建PGD页表中virt地址对应的页表项。
> 2. 填充下一个level的页表项。这里是PUD页表。由于使用了ARM64初期使用section mapping，因此PUD页表就是最后一个中间level的页表，所以只剩下PMD页表的页表项没有填充，virt地址对应的PMD页表项最终会填充block descriptor。假设这里使用4级页表，那么下面还会创建PMD页表的页表项，也就是只留下PTE页表。所以，宏定义是创建所有中间level的页表项，只留下最后一级页表。

在经过create_pgd_entry宏的调用后，就填充好了从PGD开始的所有中间level的页表的页表项的填充操作。现在是不是只剩下PTE页表的页表项没有填充呢？所以最后一个create_block_map就是完成这个操作的。

1. /*
2.  * Macro to populate block entries in the page table for the start..end
3.  * virtual range (inclusive).
4.  *
5.  * Preserves:	tbl, flags
6.  * Corrupts:	phys, start, end, pstate
7.  */
8. 	.macro	create_block_map, tbl, flags, phys, start, end
9. 	lsr	\phys, \phys, #SWAPPER_BLOCK_SHIFT
10. 	lsr	\start, \start, #SWAPPER_BLOCK_SHIFT
11. 	and	\start, \start, #PTRS_PER_PTE - 1               // table index
12. 	orr	\phys, \flags, \phys, lsl #SWAPPER_BLOCK_SHIFT  // table entry
13. 	lsr	\end, \end, #SWAPPER_BLOCK_SHIFT
14. 	and	\end, \end, #PTRS_PER_PTE - 1                   // table end index
15. 9999:	str	\phys, [\tbl, \start, lsl #3]               // store the entry
16. 	add	\start, \start, #1                              // next entry
17. 	add	\phys, \phys, #SWAPPER_BLOCK_SIZE               // next block
18. 	cmp	\start, \end
19. 	b.ls	9999b
20. 	.endm 

create_block_map宏的作用是创建虚拟地址（从start到end）区域映射到到phys物理地址。传入5个参数，分别如下意思。

> - tbl：页表基地址
> - flags：将要填充页表项的flags
> - phys：创建映射的物理地址
> - start：创建映射的虚拟地址起始地址
> - end：创建映射的虚拟地址结束地址

我们还是依然翻译成C语言分析。

1. #define SWAPPER_BLOCK_SHIFT	PMD_SHIFT
2. #define SWAPPER_BLOCK_SIZE	(1 << PMD_SHIFT)

4. #define create_block_map(tbl, flags, phys, start, end) do {  \
5. 		phys >>= SWAPPER_BLOCK_SHIFT;                /* 1 */ \
6. 		start >>= SWAPPER_BLOCK_SHIFT;               /* 2 */ \
7. 		start &= PTRS_PER_PTE - 1;                   /* 2 */ \
8. 		phys = flags | (phys << SWAPPER_BLOCK_SHIFT);/* 3 */ \
9. 		end >>= SWAPPER_BLOCK_SHIFT;                 /* 4 */ \
10. 		end &= PTRS_PER_PTE - 1;                     /* 4 */ \
11.                                                              \
12. 		while (start != end) {                       /* 5 */ \
13. 			*((long *)(tbl + (start << 3))) = phys;  /* 6 */ \
14. 			start++;                                 /* 7 */ \
15. 			phys += SWAPPER_BLOCK_SIZE;              /* 8 */ \
16. 		}                                                    \
17. 	} while (0) 

> 1. 针对phys的低SWAPPER_BLOCK_SHIFT位进行清零，和第三步骤的phys << SWAPPER_BLOCK_SHIFT收尾呼应。相当于对齐（这里的情况是2M对齐）。
> 2. 计算起始地址start的页目录项的index。
> 3. 构造描述符。
> 4. 计算结束地址end的页目录项的index。
> 5. 循环填充start到end的页目录项。
> 6. 根据页表基地址tbl和当前的start变量填充对应的页表项。start << 3是因为ARM64地址占用8 bytes。
> 7. 更新下一个页表项。
> 8. 更新下一个block的物理地址。

如何使用上述三个接口创建映射关系呢？其实很简单，首先我们需要先调用create_pgd_entry宏填充PGD以及所有中间level的页表项。最后的PMD页表的填充可以调用create_block_map宏来完成操作。

## 如何创建页表

在汇编代码阶段的head.S文件中，负责创建映射关系的函数是create_page_tables。create_page_tables函数负责identity mapping和kernel image mapping。前文提到identity mapping主要是打开MMU的过度阶段，因此对于identity mapping不需要映射整个kernel，只需要映射操作MMU代码相关的部分。如何区分这部分代码呢？当然是利用linux中常用手段自定义代码段。自定义的代码段的名称是".idmap.text"。除此之外，肯定还需要在链接脚本中声明两个标量，用来标记代码段的开始和结束。可以从vmlinux.lds.S中找到答案。

1. #define IDMAP_TEXT                             \
2. 	. = ALIGN(SZ_4K);                          \
3. 	VMLINUX_SYMBOL(__idmap_text_start) = .;    \
4. 	*(.idmap.text)                             \
5. 	VMLINUX_SYMBOL(__idmap_text_end) = .; 

从链接脚本中可以看出idmap_text_start和idmap_text_end分别是.idmap.text段的起始和结束地址。在创建identity mapping的时候会使用。另外我们同样从链接脚本中得到_text和_end两个变量，分别是kernel代码链接的开始和结束地址。编译器的链接地址实际上就是最后代码期望运行的地址。在KASLR关闭的情况下就是kernel image需要映射的虚拟地址。当我们编译kernel后，可以根据符号表System.map文件查看哪些函数被放在".idmap.text"段。当然你也可以看代码，但是我觉得没有这种方法简单。

1. ffff200008fbc000 T __idmap_text_start
2. ffff200008fbc000 T kimage_vaddr
3. ffff200008fbc008 T el2_setup
4. ffff200008fbc054 t set_hcr
5. ffff200008fbc118 t install_el2_stub
6. ffff200008fbc16c t set_cpu_boot_mode_flag
7. ffff200008fbc190 T secondary_holding_pen
8. ffff200008fbc1b4 t pen
9. ffff200008fbc1c8 T secondary_entry
10. ffff200008fbc1d4 t secondary_startup
11. ffff200008fbc1e4 t __secondary_switched
12. ffff200008fbc218 T __enable_mmu
13. ffff200008fbc26c t __no_granule_support
14. ffff200008fbc290 t __primary_switch
15. ffff200008fbc2b0 T cpu_resume
16. ffff200008fbc2d0 T cpu_do_resume
17. ffff200008fbc340 T idmap_cpu_replace_ttbr1
18. ffff200008fbc370 T __cpu_setup
19. ffff200008fbc3f0 t crval
20. ffff200008fbc408 T __idmap_text_end 

create_page_tables的汇编代码比较简单，就不转换成C语言讲解了。create_page_tables实现如下。

1. __create_page_tables:
2. 	mov	x7, SWAPPER_MM_MMUFLAGS
3. 	/*
4. 	 * Create the identity mapping.
5. 	 */
6. 	adrp	x0, idmap_pg_dir                                             /* 1 */
7. 	adrp	x3, __idmap_text_start          // __pa(__idmap_text_start)  /* 2 */
8. 	create_pgd_entry x0, x3, x5, x6                                      /* 3 */
9. 	mov	x5, x3                              // __pa(__idmap_text_start)  /* 4 */
10. 	adr_l	x6, __idmap_text_end            // __pa(__idmap_text_end)
11. 	create_block_map x0, x7, x3, x5, x6                                  /* 5 */

13. 	/*
14. 	 * Map the kernel image.
15. 	 */
16. 	adrp	x0, swapper_pg_dir                                           /* 6 */
17. 	mov_q	x5, KIMAGE_VADDR + TEXT_OFFSET  // compile time __va(_text)
18. 	add	x5, x5, x23                         // add KASLR displacement    /* 7 */
19. 	create_pgd_entry x0, x5, x3, x6                                      /* 8 */
20. 	adrp	x6, _end                        // runtime __pa(_end)
21. 	adrp	x3, _text                       // runtime __pa(_text)
22. 	sub	x6, x6, x3                          // _end - _text
23. 	add	x6, x6, x5                          // runtime __va(_end)
24. 	create_block_map x0, x7, x3, x5, x6                                  /* 9 */ 

> 1. x0寄存器PGD页表基地址，这里是idmap_pg_dir，是为了创建identity mapping。
> 2. adrp指令可以获取__idmap_text_start符号的实际运行物理地址。
> 3. 填充PGD及中间level页表的页表项。
> 4. 因为我们为了创建虚拟地址和物理地址相等的映射，因此这里的x5和x3值相等。
> 5. 调用create_block_map创建identity mapping，注意这里传递的参数物理地址（x3）和虚拟地址（x5）相等。
> 6. 创建kernel image mapping，PGD页表基地址是swapper_pg_dir。
> 7. KASLR默认关闭的情况下，x23的值为0。
> 8. 填充PGD及中间level页表的页表项。
> 9. 填充PMD页表项。因为采用的是section mapping，所以一个页表项对应2M大小。注意汇编中的注释，va()代表得到的事虚拟地址，pa()得到的是物理地址。

经过以上初始化，页表就算是初始化完成。kernel映射区域从先行映射区域迁移到VMALLOC区域在哪里体现呢？答案就是KIMAGE_VADDR宏定义。KIMAGE_VADDR是kernel的虚拟地址。其定义在arch/arm64/mm/memory.h文件。

  

1. #define VA_BITS         (CONFIG_ARM64_VA_BITS)
2. #define VA_START        (UL(0xffffffffffffffff) - (UL(1) << VA_BITS) + 1)
3. #define PAGE_OFFSET     (UL(0xffffffffffffffff) - (UL(1) << (VA_BITS - 1)) + 1)
4. #define KIMAGE_VADDR    (MODULES_END)
5. #define VMALLOC_START   (MODULES_END)
6. #define VMALLOC_END     (PAGE_OFFSET - PUD_SIZE - VMEMMAP_SIZE - SZ_64K)
7. #define MODULES_END     (MODULES_VADDR + MODULES_VSIZE)
8. #define MODULES_VADDR   (VA_START + KASAN_SHADOW_SIZE)
9. #define MODULES_VSIZE   (SZ_128M)
10. #define VMEMMAP_START   (PAGE_OFFSET - VMEMMAP_SIZE)
11. #define PCI_IO_END      (VMEMMAP_START - SZ_2M)
12. #define PCI_IO_START    (PCI_IO_END - PCI_IO_SIZE)
13. #define FIXADDR_TOP     (PCI_IO_START - SZ_2M)
14. #define TASK_SIZE_64    (UL(1) << VA_BITS) 

  

上面的宏定义显得不够直观，画张图表示现阶段kernel的地址空间分布情况。可以看出KIMAGE_VADDR正好处在VMALLOC区域，因此kernnel的运行地址位于VMALLOC区域。

[![kernel地址空间分布.png](http://www.wowotech.net/content/uploadfile/201804/b8e81524313791.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201804/b8e81524313791.png)

## virt_to_phys和phys_to_virt怎么办

通过上面的介绍，你应该有所了解kernel image和linear mapping region不在一个区域。virt_to_phys宏的作用是将内核虚拟地址转换成物理地址（针对线性映射区域）。在kernel image还在线性映射区域的时候，virt_to_phys宏可以将kernel代码中的一个地址转换成物理地址，因为线性映射区域，物理地址和虚拟地址只有一个偏移。因此两者很容易转换。那么现在kernel image和线性映射区域分开了，virt_to_phys宏又该如何实现呢？virt_to_phys宏实现如下。

1. #define PHYS_OFFSET              ({ VM_BUG_ON(memstart_addr & 1); memstart_addr; })

3. #define __is_lm_address(addr)    (!!((addr) & BIT(VA_BITS - 1)))
4. #define __lm_to_phys(addr)       (((addr) & ~PAGE_OFFSET) + PHYS_OFFSET)
5. #define __kimg_to_phys(addr)     ((addr) - kimage_voffset)

7. #define __virt_to_phys_nodebug(x) ({              \
8. 	phys_addr_t __x = (phys_addr_t)(x);           \
9. 	__is_lm_address(__x) ? __lm_to_phys(__x) :    \
10. 			       __kimg_to_phys(__x);           \
11. #define __virt_to_phys(x)	__virt_to_phys_nodebug(x)

13. static inline phys_addr_t virt_to_phys(const volatile void *x)
14. {
15. 	return __virt_to_phys((unsigned long)(x));
16. } 

  

从__virt_to_phys_nodebug宏可以看出其中的奥秘。通过addr地址的(VA_BITS - 1)位是否为1判断addr是位于kernel image区域还是线性映射区域（因为线性映射区域大小正好是kernel虚拟地址空间大小的一半）。针对线性映射区域，虚拟地址和物理地址的偏差是memstart_addr。针对kernel image区域，虚拟地址和物理地址的偏差是kimage_voffset。kimage_voffset和memstart_addr是如何计算的呢？先看看kimage_voffset的计算。

1. #define KERNEL_START    _text
2. #define __PHYS_OFFSET	(KERNEL_START - TEXT_OFFSET)
3. ENTRY(kimage_vaddr)
4. 	.quad		_text - TEXT_OFFSET
5. /*
6.  * The following fragment of code is executed with the MMU enabled.
7.  *
8.  *   x0 = __PHYS_OFFSET
9.  */
10. __primary_switched:
11. 	ldr_l	x4, kimage_vaddr        // Save the offset between      /* 2 */
12. 	sub	x4, x4, x0                  // the kernel virtual and       /* 3 */
13. 	str_l	x4, kimage_voffset, x5  // physical mappings            /* 4 */

15. 	b	start_kernel

17. __primary_switch:
18. 	bl	__enable_mmu
19. 	ldr	x8, =__primary_switched
20. 	adrp	x0, __PHYS_OFFSET                                       /* 1 */
21. 	br	x8 

  

> 1. x0是__primary_switch函数中设置。x0寄存器通过adrp指令可以获取运行时的地址。也就是实际运行的物理地址。你是不是好奇此时不是已经打开MMU了嘛！为什么adrp得到的运行地址就是物理地址呢？请往上翻看看__primary_switch函数是不是位于".idmap.text"段，那么该段是identity mapping。因此获取的运行地址虽然是虚拟地址，但是它和实际运行的物理地址相等。
> 2. x4寄存器保存的是kernel image的运行的虚拟地址。你是不是又好奇这个地方为什么获取的运行地址和物理地址不相等呢？其实是因为__primary_switched函数映射在kernel image mapping区域。
> 3. 计算虚拟地址和物理地址的偏移。
> 4. 将偏移写入kimage_voffset全局变量。

memstart_addr是kernel选取的物理基地址，memstart_addr在arm64_memblock_init函数中设置。arm64_memblock_init函数实现如下（截取部分代码）。

1. void __init arm64_memblock_init(void)
2. {
3. 	const s64 linear_region_size = -(s64)PAGE_OFFSET;

5. 	/*
6. 	 * Ensure that the linear region takes up exactly half of the kernel
7. 	 * virtual address space. This way, we can distinguish a linear address
8. 	 * from a kernel/module/vmalloc address by testing a single bit.
9. 	 */
10. 	BUILD_BUG_ON(linear_region_size != BIT(VA_BITS - 1));           /* 1 */

12. 	/*
13. 	 * Select a suitable value for the base of physical memory.
14. 	 */
15. 	memstart_addr = round_down(memblock_start_of_DRAM(),            /* 2 */
16. 				   ARM64_MEMSTART_ALIGN);

18. 	memblock_remove(max_t(u64, memstart_addr + linear_region_size,  /* 3 */
19. 			__pa_symbol(_end)), ULLONG_MAX);
20. 	if (memstart_addr + linear_region_size < memblock_end_of_DRAM()) {
21. 		/* ensure that memstart_addr remains sufficiently aligned */
22. 		memstart_addr = round_up(memblock_end_of_DRAM() - linear_region_size,
23. 					 ARM64_MEMSTART_ALIGN);                         /* 4 */
24. 		memblock_remove(0, memstart_addr);                          /* 5 */
25. 	}
26. } 

  

> 1. 从注释以及代码皆可以看出，PAGE_OFFSET是线性区域的开始虚拟地址。线性区域大小是整个kernel虚拟地址空间的一半。
> 2. 选取一个合适的物理基地址，根据RAM的起始地址按照1G对齐。
> 3. memstart_addr是选取的物理基地址。kernel虚拟地址空间一半大小作为线性映射区域。因此最大支持的内存范围是memstart_addr + linear_region_size。所以告诉memblock，超过这个区域的范围都是非法的。
> 4. 如果memstart_addr + linear_region_size的值小于RAM的结束地址，说明[memstart_addr, memstart_addr + linear_region_size]地址空间范围的区域无法覆盖整个RAM地址范围。这时候就需要从RAM结束地址减去linear_region_size的值作为memstart_addr。什么时候会出现这种情况呢？当物理内存足够大时，if语句就可能满足条件了。
> 5. 既然4满足，自然这里[0, memstart_addr]的地址空间需要从memblock中remove。

memstart_addr的值定下来之后，虚拟地址和物理地址以memstart_addr为偏差创建线性映射区域。在map_mem函数中完成。phys_to_virt宏的实现就不用介绍了，就是virt_to_phys的反操作。

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [一次触摸屏中断调试引发的深入探究](http://www.wowotech.net/linux_kenrel/437.html) | [tty驱动分析](http://www.wowotech.net/tty_framework/435.html)»

**评论：**

**xiaowu**  
2020-12-27 16:38

您好，有个疑惑的地方，这个汇编翻译成C的，id_map段创建block_map时，大小小于2M，start和end地址相同，走不进while循环，那岂不是无法填充tbl中的单元？  
#define create_block_map(tbl, flags, phys, start, end) do {  \  
        phys >>= SWAPPER_BLOCK_SHIFT;                /* 1 */ \  
        start >>= SWAPPER_BLOCK_SHIFT;               /* 2 */ \  
        start &= PTRS_PER_PTE - 1;                   /* 2 */ \  
        phys = flags | (phys << SWAPPER_BLOCK_SHIFT);/* 3 */ \  
        end >>= SWAPPER_BLOCK_SHIFT;                 /* 4 */ \  
        end &= PTRS_PER_PTE - 1;                     /* 4 */ \  
                                                             \  
        while (start != end) {                       /* 5 */ \  
            *((long *)(tbl + (start << 3))) = phys;  /* 6 */ \  
            start++;                                 /* 7 */ \  
            phys += SWAPPER_BLOCK_SIZE;              /* 8 */ \  
        }                                                    \  
    } while (0)

[回复](http://www.wowotech.net/memory_management/436.html#comment-8168)

**orangeboyye**  
2019-11-17 22:31

[ kernel image映射区域从原来的linear mapping region（线性映射区域）搬移到VMALLOC区域 ]  
你确定对吗，我咋感觉肯定不对呢

[回复](http://www.wowotech.net/memory_management/436.html#comment-7754)

**[smcdef](http://www.wowotech.net/)**  
2019-12-09 18:24

@orangeboyye：这里的代码分析都是基于ARM64 arch。你可以瞅瞅最新的代码是不是这样的

[回复](http://www.wowotech.net/memory_management/436.html#comment-7779)

**misaka**  
2019-10-12 10:44

您好，请问 memstart_addr = round_down(memblock_start_of_DRAM(),ARM64_MEMSTART_ALIGN); 这条语句为什么不是 round_up 而是 round_down 呢？ round_down 了地址不是可能会比 memblock_start_of_DRAM() 小吗，可是 memblock_start_of_DRAM() 不是最低可用的物理地址吗？thx

[回复](http://www.wowotech.net/memory_management/436.html#comment-7693)

**威点零**  
2019-06-24 11:33

hi,@smcdef,深入浅出，强大。另外请教一下，VA转PA都是固定的offset，那么针对Kernel中的动态分配的内存，有两条路，一条是slub一条是大块的走buddy system。kernel是怎么保证这个VA和PA一定是一个offset的？

[回复](http://www.wowotech.net/memory_management/436.html#comment-7485)

**威点零**  
2019-06-24 16:12

@威点零：我想了一下，是不是分配kmalloc的时候，va是通过pa转过来的，所以全部都差offset。补充一个问题，vmalloc分配的va是连续的，但是PA拿到的是不连续的，看起来这样就没办法直接做到va/pa转换直接加offset？

[回复](http://www.wowotech.net/memory_management/436.html#comment-7486)

**[smcdef](http://www.wowotech.net/)**  
2019-07-09 15:47

@威点零：VA和PA偏差是固定offset仅仅针对linear mapping区域，VMALLOC是不保证这个特征的。

[回复](http://www.wowotech.net/memory_management/436.html#comment-7519)

**[smcdef](http://www.wowotech.net/)**  
2019-07-09 15:46

@威点零：分配的内存给用户，用户拿到的都是VA。不关注PA。VA和PA偏差offset是映射的时候决定的。slub和buddy的使用已经是在映射建立后了，此时就不需要关注PA了。

[回复](http://www.wowotech.net/memory_management/436.html#comment-7518)

**randyqiu**  
2018-08-08 19:57

问一个问题，内核把kernel image也放到vmalloc区域，进行这样的修改是为了解决什么问题？内核之前kernel区域是线性映射的，个人认为是内核一直常驻内存，不会swap出去，所以用section方式的线性映射关系还可以减少页表数目，减少内存开销和MMU映射时的MIPS开销，现在又放到vmalloc这个可以被释放出去的区域里面干啥？

[回复](http://www.wowotech.net/memory_management/436.html#comment-6873)

**[smcdef](http://www.wowotech.net/)**  
2018-08-09 22:48

@randyqiu：其实并kernel并不会释放，就是仅仅换一个区域而已，本质来说和之前差别不大（虽然再VMALLOC区域，但是kernel的部分其实物理地址还是连续的）。迁移到VMALLOC区域可能是和KASLR有关（猜测，没有去看修改提交记录）。关于KASLR部分也可以参考本网站的一篇文章。

[回复](http://www.wowotech.net/memory_management/436.html#comment-6878)

**nzg**  
2018-08-03 15:01

您好，请教一下PAGE_OFFSET这个宏定义为： PAGE_OFFSET - the virtual address of the start of the kernel image (top(VA_BITS - 1))  
按照这个注释的意思，kernel image 的起始虚拟地址应该是PAGE_OFFSET+TEXT_OFFSET, 但实际上_text和 _end 这两个symbol却不在此范围，这是为什么呢？ PAGE_OFFSET到底是用来做什么的呢？

[回复](http://www.wowotech.net/memory_management/436.html#comment-6865)

**[smcdef](http://www.wowotech.net/)**  
2018-08-04 13:05

@nzg：文章的目的就是为了介绍kernel日新月异的变化！你贴的注释应该是以前的代码，但是这篇文章是基于4.13内核代码分析变化。我给你看看4.13内核对于PAGE_OFFSET的解释  
PAGE_OFFSET - the virtual address of the start of the linear map

[回复](http://www.wowotech.net/memory_management/436.html#comment-6868)

**nzg**  
2018-08-06 13:23

@smcdef：您好，我看的代码版本是4.4.83,PAGE_OFFSET就是我贴出的那样注释，实际上印出_text & _end并不在PAGE_OFFSET往后的范围中..

[回复](http://www.wowotech.net/memory_management/436.html#comment-6869)

**[smcdef](http://www.wowotech.net/)**  
2018-08-06 22:27

@nzg：4.4.83链接脚本：  
. = PAGE_OFFSET + TEXT_OFFSET;  
  
.head.text : {  
    _text = .;  
}  
_text = PAGE_OFFSET + TEXT_OFFSET

[回复](http://www.wowotech.net/memory_management/436.html#comment-6871)

**cqy**  
2018-07-31 12:36

嗨，@smcdef ，感谢详细的分析。  
我有点疑问。  
在预处理阶段，_text符号的地址应该还没有分配吧？  
那么预处理之后，KERNEL_START和__PHYS_OFFSET的值应该是多少呢？

[回复](http://www.wowotech.net/memory_management/436.html#comment-6857)

**[smcdef](http://www.wowotech.net/)**  
2018-07-31 22:18

@cqy：#define KERNEL_START      _text  
_text在连接脚本中定义，你可以看作是一个变量，预处理就是把KERNEL_START替换成_text而已，在链接的时候_text会有一个确定的值。

[回复](http://www.wowotech.net/memory_management/436.html#comment-6861)

**Tw**  
2018-04-24 16:44

if (memstart_addr + linear_region_size < memblock_end_of_DRAM())  
  
这里是不是因为(memstart_addr + linear_region_size) overflow了，可能是因为memstart_addr选取的太大了...

[回复](http://www.wowotech.net/memory_management/436.html#comment-6692)

**[smcdef](http://www.wowotech.net/)**  
2018-04-30 10:48

@Tw：或许真的是这样

[回复](http://www.wowotech.net/memory_management/436.html#comment-6723)

**kuroky**  
2018-06-12 11:36

@smcdef：有可能，毕竟128TB的内存不多

[回复](http://www.wowotech.net/memory_management/436.html#comment-6796)

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
    
    - [Process Creation（二）](http://www.wowotech.net/process_management/process-creation-2.html)
    - [RCU（2）- 使用方法](http://www.wowotech.net/kernel_synchronization/462.html)
    - [Dynamic DMA mapping Guide](http://www.wowotech.net/memory_management/DMA-Mapping-api.html)
    - [计算机科学基础知识（三）:静态库和静态链接](http://www.wowotech.net/basic_subject/static-link.html)
    - [Linux内核同步机制之（七）：RCU基础](http://www.wowotech.net/kernel_synchronization/rcu_fundamentals.html)
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
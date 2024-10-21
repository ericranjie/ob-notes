
作者：[smcdef](http://www.wowotech.net/author/531) 发布于：2018-4-29 20:35 分类：[内存管理](http://www.wowotech.net/sort/memory_management)

# 引言

fixmap是一段固定地址映射。kernel预留一段虚拟地址空间。因此虚拟地址是在编译的时候确定。fixmap可以用来做什么？kernel启动初期，由于此时的kernel已经运行在虚拟地址上。因此我们访问具体的物理地址是不行的，必须建立虚拟地址和物理地址的映射，然后通过虚拟地址访问才可以。例如：dtb中包含bootloader传递过来的内存信息，我们需要解析dtb，但是我们得到的是dtb的物理地址。因此访问之前必须创建映射，创建映射又需要内存。但是由于所有的内存管理子系统还没有ready。因此我们不能使用ioremap接口创建映射。为此kernel提出fixmap的解决方案。

> 注：文章代码分析基于linux-4.15，架构基于aarch64（ARM64）。

# fixmap空间分配

fixmap虚拟地址空间又被平均分成两个部分permanent fixed addresses和temporary fixed addresses。permanent fixed addresses是永久映射，temporary fixed addresses是临时映射。永久映射是指在建立的映射关系在kernel阶段不会改变，仅供特定模块一直使用。临时映射就是模块使用前创建映射，使用后解除映射。

fixmap区域又被继续细分，分配给不同模块使用。kernel中定义枚举类型作为index，根据index可以计算该模拟在fixmap区域的虚拟地址。

```cpp
1. enum fixed_addresses {
2. 	FIX_HOLE,
3. #define FIX_FDT_SIZE		(MAX_FDT_SIZE + SZ_2M)
4. 	FIX_FDT_END,
5. 	FIX_FDT = FIX_FDT_END + FIX_FDT_SIZE / PAGE_SIZE - 1,    /* 1 */
6. 	FIX_EARLYCON_MEM_BASE,                                   /* 2 */
7. 	FIX_TEXT_POKE0,
8. 	__end_of_permanent_fixed_addresses,
9. #define NR_FIX_BTMAPS		(SZ_256K / PAGE_SIZE)
10. #define FIX_BTMAPS_SLOTS	7
11. #define TOTAL_FIX_BTMAPS	(NR_FIX_BTMAPS * FIX_BTMAPS_SLOTS)
12. 	FIX_BTMAP_END = __end_of_permanent_fixed_addresses,
13. 	FIX_BTMAP_BEGIN = FIX_BTMAP_END + TOTAL_FIX_BTMAPS - 1,  /* 3 */
14. 	FIX_PTE,
15. 	FIX_PMD,
16. 	FIX_PUD,
17. 	FIX_PGD,
18. 	__end_of_fixed_addresses
19. };
20. #define FIXADDR_SIZE	(__end_of_permanent_fixed_addresses << PAGE_SHIFT)
21. #define FIXADDR_START	(FIXADDR_TOP - FIXADDR_SIZE) 
```

> 1. FIX_FDT映射设备树使用。在ARM64架构，大小是4M。
> 1. early console使用，大小1页。1页虚拟地址空间完全够了，毕竟串口操作相关寄存器没有几个。
> 1. early_ioremap()接口使用，这部分属于动态映射。大小是7 × 256KB。

fixmap区域是地址空间范围从FIXADDR_START到FIXADDR_TOP结束。FIXADDR_SIZE是permanent fixed addresses区域的大小。我对这个地方很奇怪，为什么不包括temporary fixed addresses区域的大小呢？如果你知道可以告诉我。fixmap区域可以想象成一块内存以页为单位被平均分成\_\_end_of_permanent_fixed_addresses块。而这些枚举值就是这块内存的index。因此虚拟地址可以根据index进行计算。计算方法如下。

```cpp
1. #define __fix_to_virt(x)	(FIXADDR_TOP - ((x) << PAGE_SHIFT))
2. #define __virt_to_fix(x)	((FIXADDR_TOP - ((x) & PAGE_MASK)) >> PAGE_SHIFT) 
```

FIX_FDT是给dtb创建映射的区域。例如需要得到FDT的虚拟地址，即可以利用\_\_fix_to_virt(FIX_FDT)得到虚拟地址。之所以FIX_FDT放在枚举的最前面，是因为我们针对dtb映射采用section mapping要求虚拟地址2M对齐。FIXADDR_TOP地址本身是2M对齐的，因此FIXADDR_TOP - (FIX_FDT \<\< PAGE_SHIFT)可以很容易2M对齐。

# fixmap初始化

fixmap初始化操作在early_fixmap_init函数中完成。主要是建立PGD/PUD/PMD页表。early_fixmap_init实现如下（简化部分代码逻辑）。

```cpp
1. static pte_t bm_pte[PTRS_PER_PTE] __page_aligned_bss;               /* 1 */
2. static pmd_t bm_pmd[PTRS_PER_PMD] __page_aligned_bss __maybe_unused;/* 1 */
3. static pud_t bm_pud[PTRS_PER_PUD] __page_aligned_bss __maybe_unused;/* 1 */

5. void __init early_fixmap_init(void)
6. {
7. 	pgd_t *pgd;
8. 	pud_t *pud;
9. 	pmd_t *pmd;
10. 	unsigned long addr = FIXADDR_START;                             /* 2 */

12. 	pgd = pgd_offset_k(addr);
13. 	if (pgd_none(*pgd))
14. 		__pgd_populate(pgd, __pa_symbol(bm_pud), PUD_TYPE_TABLE);   /* 2 */
15. 	pud = fixmap_pud(addr);
16. 	if (pud_none(*pud))
17. 		__pud_populate(pud, __pa_symbol(bm_pmd), PMD_TYPE_TABLE);   /* 2 */
18. 	pmd = fixmap_pmd(addr);
19. 	__pmd_populate(pmd, __pa_symbol(bm_pte), PMD_TYPE_TABLE);       /* 2 */

21. 	BUILD_BUG_ON((__fix_to_virt(FIX_BTMAP_BEGIN) >> PMD_SHIFT)
22. 		     != (__fix_to_virt(FIX_BTMAP_END) >> PMD_SHIFT));       /* 3 */

24. 	if ((pmd != fixmap_pmd(fix_to_virt(FIX_BTMAP_BEGIN)))
25. 			|| pmd != fixmap_pmd(fix_to_virt(FIX_BTMAP_END))) {     /* 3 */
26. 		WARN_ON(1);
27. 	}
28. } 
```

> 1. 静态定义数组作为页表（PUD/PMD/PTE）使用。PGD页表使用swapper_pg_dir。
> 1. 以FIXADDR_START为虚拟地址，建立页表映射关系。假设计算FDT的虚拟地址为addr = \_\_fix_to_virt(FIX_FDT)，必然addr是2M对齐的一个地址。由上面的分析可知，FIXADDR_START的值位于\[addr - 2M, addr\]之间。因此访问\[addr - 2M, addr\]之间的虚拟地址不需要再建立PUD/PMD页表项。只需要设置PTE页表对应的页表项即可。
> 1. \[FIX_BTMAP_BEGIN, FIX_BTMAP_END\]区域给动态映射使用，保证该区域正好位于\[addr - 2M, addr\]之间，必须检查动态映射区域小于2M。定义的PTE页表数组其实是给动态映射使用的。当我们需要访问物理地址A，从\[addr - 2M, addr\]区域找到一个合适的虚拟地址B，填充B地址对应的PTE页表项即可访问。这也是early_ioremap的实现原理。

经过early_fixmap_init函数的探究，我们也可以得到一个结论：为了以page为单位进行映射，必须保证FIX_FDT和\_\_end_of_fixed_addresses之间的虚拟地址空间必须小于2M。如果有超过2M的部分就要使用section mapping（因为只有一个PTE页表），以2M为单位映射。

# early ioremap初始化

如果你希望kernel启动早期使用ioremap操作，其实是不行的。我们必须借助early ioremap接口。early ioremap是基于fixmap实现。初始化在early_ioremap_init完成。简化部分代码如下。

```cpp
1. static void __iomem *prev_map[FIX_BTMAPS_SLOTS] __initdata;
2. static unsigned long prev_size[FIX_BTMAPS_SLOTS] __initdata;
3. static unsigned long slot_virt[FIX_BTMAPS_SLOTS] __initdata;

5. void __init early_ioremap_setup(void)
6. {
7. 	int i;

9. 	for (i = 0; i < FIX_BTMAPS_SLOTS; i++)
10. 		slot_virt[i] = __fix_to_virt(FIX_BTMAP_BEGIN - NR_FIX_BTMAPS*i);
11. }

13. void __init early_ioremap_init(void)
14. {
15. 	early_ioremap_setup();
16. } 
```

> early ioremap利用slot管理映射，最多支持FIX_BTMAPS_SLOTS个映射，每个映射最大支持映射256KB。slot_virt数组存储每个slot的虚拟地址首地址。prev_map数组用来记录已经分配出去的虚拟地址，数组值为0代表没有分配。prev_size记录映射的size。

# 创建FDT映射

创建FDT映射的函数是\_\_fixmap_remap_fdt，实现如下。

```cpp
1. void *__init __fixmap_remap_fdt(phys_addr_t dt_phys,
2. 		int *size, pgprot_t prot)
3. {
4. 	const u64 dt_virt_base = __fix_to_virt(FIX_FDT);                 /* 1 */
5. 	int offset;
6. 	void *dt_virt;

8. 	BUILD_BUG_ON(dt_virt_base % SZ_2M);                              /* 2 */

10. 	BUILD_BUG_ON(__fix_to_virt(FIX_FDT_END) >> SWAPPER_TABLE_SHIFT !=
11. 		     __fix_to_virt(FIX_BTMAP_BEGIN) >> SWAPPER_TABLE_SHIFT); /* 3 */

13. 	offset = dt_phys % SWAPPER_BLOCK_SIZE;
14. 	dt_virt = (void *)dt_virt_base + offset;

16. 	create_mapping_noalloc(round_down(dt_phys, SWAPPER_BLOCK_SIZE),  /* 4 */
17. 			dt_virt_base, SWAPPER_BLOCK_SIZE, prot);

19. 	*size = fdt_totalsize(dt_virt);
20. 	if (offset + *size > SWAPPER_BLOCK_SIZE)
21. 		create_mapping_noalloc(round_down(dt_phys, SWAPPER_BLOCK_SIZE), dt_virt_base,
22. 			   round_up(offset + *size, SWAPPER_BLOCK_SIZE), prot);  /* 5 */

24. 	return dt_virt;
25. } 
```

> 1. 根据\_\_fix_to_virt计算fdt虚拟地址。
> 1. 虚拟地址必须2M对齐。主要是为了section mapping。
> 1. 在early_fixmap_init函数中，已经建立了PUD/PMD页表。为了让映射过程不用分配额外的PMD页表内存，这里必须保证FDT所在的虚拟地址范围落在early_fixmap_init函数建立的PMD页表范围内。
> 1. 创建页表。这里映射2M空间。
> 1. 如果dtb文件结尾地址超过上一个建立映射的地址范围，就必须紧接着再映射2M空间。

______________________________________________________________________

« [KASLR](http://www.wowotech.net/memory_management/441.html) | [文件系统和裸块设备的page cache问题](http://www.wowotech.net/filesystem/439.html)»

**评论：**

**rikeyone**\
2018-12-12 15:33

ffc00000    ffefffff    Fixmap mapping region.  Addresses provided\
by fix_to_virt() will be located here.

PKMAP_BASE  PAGE_OFFSET-1   Permanent kernel mappings\
One way of mapping HIGHMEM pages into kernel\
space.

我看arm的内存分布，fixmap仅仅代表的是临时映射，永久映射区的定义在PKMAP那里。

PKMAP也就是Permanent kernel mappings 永久映射。

我的理解看来和楼主有点不同

[回复](http://www.wowotech.net/memory_management/440.html#comment-7079)

**[smcdef](http://www.wowotech.net/)**\
2018-12-16 00:03

@rikeyone：ARM的设计我不是很了解，其实文章的引言部分已经注明了架构是ARM64。所以我说的ARM64平台。

[回复](http://www.wowotech.net/memory_management/440.html#comment-7085)

**win.lin**\
2018-07-09 11:28

恩，我理解错了。这里面有几个词要分清，permanent-fix-map(永久固定映射)/temporary-fix-map(临时固定映射)。还有一个是persisent-kernel-mapping(持久内核映射)。要翻译过来都很接近。

[回复](http://www.wowotech.net/memory_management/440.html#comment-6841)

**rikeyone**\
2018-12-12 15:44

@win.lin：可以去看一下代码，kmap就是永久内存映射，最后映射到的地址就是跟PKMAP_BASE有关，所以我认为这个理解上，博主可能出现点差错。

当然如果是我有错，请大家指正。

[回复](http://www.wowotech.net/memory_management/440.html#comment-7080)

**win.lin**\
2018-07-09 11:11

对的。fix-map分:fix-map与temporary fix_map。和Permanent不是一回事。

[回复](http://www.wowotech.net/memory_management/440.html#comment-6840)

**蓮之物**\
2018-07-06 16:44

(  )5.關於fixmap address,下列何者正確\
(1) 在kernel啟動初期,我們藉著使用ioremap接口創建虛擬地址和物理地址的映射\
(2) fixmap區域是地址空間範圍從FIXADDR_START到FIXADDR_TOP結束, FIXADDR_SIZE是temporary fixed addresses區域的大小\
(3) kernel正常運作時,運作temporary fixed addresses的資料必須是特權指令的\
(4) FIX_FDT是給dtb創建映射的區域。例如需要得到FDT的虛擬地址，即可以利用\_\_virt_to_fix(x)得到虛擬地址,也就是運作(FIXADDR_TOP - ((x) \<\< PAGE_SHIFT))\
(5) Device Tree Blob中包含bootloader傳遞過來的內存信息,而我們得到的是Device Tree Blob的物理地址

[回复](http://www.wowotech.net/memory_management/440.html#comment-6837)

**蓮之物**\
2018-07-06 16:46

@蓮之物：up寫得太好\
情不自禁的做了個單選題

[回复](http://www.wowotech.net/memory_management/440.html#comment-6838)

**gnx**\
2018-05-06 16:51

固定映射空间分为两部分：永久映射和临时映射，其中临时映射和kmap（持久映射）在32位系统上有一些纠葛。在32位系统上，如果使能了高端内存，虚拟地址的最后两部分分别是持久映射和固定映射，这个持久映射不要和永久固定映射搞混淆了。持久映射是可以通过kmap和kunmap建立和解除映射的，持久映射可以理解为是一种将高端内存映射到内核虚拟地址空间的一种方式。但是kmap函数有个问题，他可能导致阻塞因而不能用在中断上下文中，所以内核搞了一个临时映射kmap_atomic，从固定映射空间拿了一块虚拟内存出来，这部门内存空间就处于固定映射的临时映射区域。所以内核在计算固定映射大小时不会把临时映射区域计算在内，当然在64位系统上没有高端内存，也就没有所谓的持久映射，但是代码为了向前兼容，就依然保持这样的计算方式，而且这也并不影响固定映射的使用。以上是我的个人理解，不对之处请指正！

[回复](http://www.wowotech.net/memory_management/440.html#comment-6735)

**[smcdef](http://www.wowotech.net/)**\
2018-05-06 17:57

@gnx：关于文章中“FIXADDR_SIZE是permanent fixed addresses区域的大小。我对这个地方很奇怪，为什么不包括temporary fixed addresses区域的大小呢？”这个疑问，我也有自己的想法。\
我是这么想的，临时映射仅仅是系统启动初期，ioremap等子系统还没有ready的时候使用，当内存管理等各个子系统ready之后，临时映射在ARM64中实际上就没有用武之地了，当然此时也不会有相关的模块还是用临时映射区域。所以临时映射区域的虚拟地址实际上并没有占用（内存管理系统没有ready的时候才占用），因此不需要显示的宏定义告诉你这部分地址被分配了。\
这位兄弟，你觉得呢？

[回复](http://www.wowotech.net/memory_management/440.html#comment-6736)

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

  - ja\
    [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
  - 元神高手\
    [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
  - 十七\
    [内核空间的映射在系统启动时就已经设定好，并且在所有进程的页表...](http://www.wowotech.net/process_management/context-switch-arch.html#8920)
  - lw\
    [sparse模型和disconti模型没看出来有什么本质区别...](http://www.wowotech.net/memory_management/memory_model.html#8919)
  - 肥饶\
    [一个没设置好就出错](http://www.wowotech.net/linux_kenrel/516.html#8918)
  - orange\
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

  - [Linux DMA Engine framework(3)\_dma controller驱动](http://www.wowotech.net/linux_kenrel/dma_controller_driver.html)
  - [在PowerShell中使用Vim](http://www.wowotech.net/soft/vim_in_powershell.html)
  - [CFS调度器（2）-源码解析](http://www.wowotech.net/process_management/448.html)
  - [Linux内核同步机制之（四）：spin lock](http://www.wowotech.net/kernel_synchronization/spinlock.html)
  - [irq wakeup in linux](http://www.wowotech.net/pm_subsystem/491.html)

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

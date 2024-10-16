
作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2016-11-24 12:08 分类：[内存管理](http://www.wowotech.net/sort/memory_management)

# 一、前言

经过[内存初始化代码分析（一）](http://www.wowotech.net/memory_management/__create_page_tables_code_analysis.html)和[内存初始化代码分析（二）](http://www.wowotech.net/memory_management/memory-layout.html)的过渡，我们终于来到了内存初始化的核心部分：paging_init。当然本文不能全部解析完该函数（那需要的篇幅太长了），我们只关注创建系统内存地址映射这部分代码实现，也就是解析paging_init中的map_mem函数。

同样的，我们选择的是4.4.6的内核代码，体系结构相关的代码来自ARM64。

# 二、准备阶段

在进入实际的代码分析之前，我们首先回头看看目前内存的状态。偌大的物理地址空间中，系统内存占据了一段或者几段地址空间，这些信息被保存在了memblock模块中的memory type类型的数组中，数组中每一个memory region描述了一段系统内存信息（base、size以及node id）。OK，系统内存就这么多，但是也并非所有的memory type类型的数组中区域都是free的，实际上，有些已经被使用或者保留使用的内存区域需要从memory type类型的一段或者几段地址空间中摘取下来，并定义在reserved type类型的数组中。实际上，在整个系统初始化过程中（更具体的说是内存管理模块完成初始化之前），我们都需要暂时使用memblock这样一个booting阶段的内存管理模块来暂时进行内存的管理（收集内存布局信息只是它的副业），每次分配的内存都是在reserved type数组中增加一个新的item或者扩展其中一个memory region的size而已。

通过memblock模块，我们已经收集了内存布局的信息（memory type类型的数组），也知道了free memory资源的信息（memory type类型的数组减去reserved type类型的数组，从集合论的角度看，reserved type类型的数组是memory type类型的数组的真子集），但是，要管理这些珍贵的系统内存，首先要能够访问他们啊（顺便说一句：memblock中的那些数组定义的地址都是物理地址），通过前面的分析文章，我们知道有两段内存已经是可见的（完成了地址映射），一个是kernel image段，另外一个是fdt段。而广大的系统内存区域仍然在黑暗之中，等待我们去拯救（进行地址映射）。

最后，我们思考这样一个问题：是否memory type类型的数组代表了整个的系统内存的地址空间呢？当然不是，有些驱动可能会保留一段系统内存区域为自己使用，同时也不希望OS管理这段内存（或者说对OS不可见），而是自己创建该段内存的地址映射。如果你对dts中的memory reserve节点比较熟悉的话，那么实际上这样的reserved memory region是有no-map属性的。这时候，内核初始化过程中，在解析该reserved-memory节点的时候，会将该段地址从memblock模块中移除。而在map_mem函数中，为所有memory type类型的数组创建地址映射的时候，有no-map属性的那段内存地址将不会创建地址映射，也就不在OS的控制范围内了。

# 三、概览

创建系统内存地址映射的代码在map_mem中，如下：

```cpp
static void init map_mem(void)  {  
struct memblock_region reg;  
phys_addr_t limit;

limit = PHYS_OFFSET + SWAPPER_INIT_MAP_SIZE;－－－－－－－－－－－－－－－（1）  
memblock_set_current_limit(limit);

for_each_memblock(memory, reg) {－－－－－－－－－－－－－－－－－－－－－－－－（2）  
phys_addr_t start = reg->base;――确定该region的起始地址  
phys_addr_t end = start + reg->size; ――确定该region的结束地址

if (start >= end)－－参数检查  
break;

if (ARM64_SWAPPER_USES_SECTION_MAPS) {－－－－－－－－－－－－－－－－（3）  
if (start < limit)  
start = ALIGN(start, SECTION_SIZE);  
if (end < limit) {  
limit = end & SECTION_MASK;  
memblock_set_current_limit(limit);  
}  
}  
__map_memblock(start, end);－－－－－－－－－－－－－－－－－－－－－－－－－（4）  
}

memblock_set_current_limit(MEMBLOCK_ALLOC_ANYWHERE);－－－－－－－－－－（5）  
}
```

（1）首先限制了当前memblock的上限。之所以这么做是因为在进行mapping的时候，如果任何一级的Translation table不存在的话都需要进行页表内存的分配。而在这个时间点上，伙伴系统没有ready，无法动态分配。当然，这时候memblock已经ready了，但是如果分配的内存都还没有创建地址映射（整个物理内存布局已知并且保存在了memblock模块中的memblock模块中，但是并非所有系统内存的地址映射都已经建立好的，而我们map_mem函数的本意就是要创建所有系统内存的mapping），内核一旦访问memblock_alloc分配的物理内存，悲剧就会发生了。怎么破？这里采用了限定memblock上限的方法。一旦设定了上限，那么memblock_alloc分配的物理内存不会高于这个上限。

设定怎样的上限呢？基本思路就是在map_mem的调用过程中，不需要分配translation table，怎么做到呢？当然是尽量利用已经静态定义好的那些页表了。PHYS_OFFSET是物理内存的起始地址，SWAPPER_INIT_MAP_SIZE 是启动阶段kernel direct mapping的size。也就是说，从PHYS_OFFSET到PHYS_OFFSET + SWAPPER_INIT_MAP_SIZE的区域，所有的页表（各个level的translation table）都已经OK，不需要分配，只需要把描述符写入页表即可。因此，如果将当前memblock分配的上限设定在这里将不会产生内存分配的动作（因为页表都已经ready）。

（2）对系统中所有的memory type的region建立对应的地址映射。由于reserved type的memory region是memory type的region的真子集，因此reserved memory 的地址映射也就一并建立了。

（3）如果不使用section map，那么我们在kernel direct mapping区域静态分配了PGD~PTE的页表，通过起始地址对齐以及对memblock limit的设定就可以保证在create_mapping（）的时候不分配页表内存。但是在下面的情况下：

（A）Memory block的start或者end地址并没有对齐在2M上

（B）使用section map

在这种情况下，调用create_mapping（）的时候会分配pte页表内存（没有对齐2M，无法进行section mapping）。怎么破？还好第一个memory block（也就是kernel image所在的block）的start address是必定对齐在2M地址上的，所以只要考虑end地址，这时候需要适当的缩小limit到end & SECTION_MASK就可以保证分配的页表内存是已经建立地址映射的了。

（4）\_\_map_memblock代码如下：

```cpp
static void init __map_memblock(phys_addr_t start, phys_addr_t end) {  
create_mapping(start, __phys_to_virt(start), end - start,  
PAGE_KERNEL_EXEC);  
}
```

需要说明的是，在map_mem之后，所有之前通过\_\_create_page_tables创建的描述符都被覆盖了，取而代之的是新的映射，并且memory attribute如下：

> #define PAGE_KERNEL_EXEC    \_\_pgprot(\_PAGE_DEFAULT | PTE_UXN | PTE_DIRTY | PTE_WRITE)

大部分memory attribute保持不变（例如MT_NORMAL、PTE_AF 、 PTE_SHARED等），有几个bit需要说明一下：PTE_UXN，Unprivileged Execute-never bit，也就是限制userspace从这里取指执行。PTE_DIRTY是一个软件设定的bit，硬件并不操作这个bit，OS软件用这个bit标识该entry是clean or dirty，如果是dirty的话，说明该page的数据已经被写入过，如果该page需要被swapped out，那么还需要保存dirty的数据才能回收该page。关于PTE_WRITE的解释todo。

（5）所有的系统内存的地址映射已经建立完毕，取消之前的上限，让memblock模块可以自由的分配内存。

# 四、填充PGD中的描述符

create_mapping实际上是调用底层的\_\_create_mapping函数完成地址映射的，具体代码如下：

```cpp
static void init create_mapping(phys_addr_t phys, unsigned long virt,  
phys_addr_t size, pgprot_t prot)  
{  
if (virt < VMALLOC_START) {  
pr_warn("BUG: not creating mapping for %pa at 0x%016lx - outside kernel range\n",  
&phys, virt);  
return;  
}  
__create_mapping(&init_mm, pgd_offset_k(virt & PAGE_MASK), phys, virt,  
size, prot, early_alloc);  
}
```

create_mapping的作用是将起始物理地址等于phys，大小是size的这一段物理内存mapping到起始虚拟地址是virt的虚拟地址空间，映射的memory attribute是prot。内核的虚拟地址空间从VMALLOC_START开始，低于这个地址就不对了，验证完虚拟地址，底层是调用\_\_create_mapping函数，传递的参数情况是这样的，init_mm是内核空间的内存描述符，pgd_offset_k是根据给定的虚拟地址，在kernel space的pgd中找到对应的描述符的位置，early_alloc是在mapping过程中，如果需要分配内存的话（页表需要内存），调用该函数进行内存的分配。\_\_create_mapping函数具体代码如下：

```cpp
static void  create_mapping(struct mm_struct mm, pgd_t *pgd,  
phys_addr_t phys, unsigned long virt,  
phys_addr_t size, pgprot_t prot,  
void *(*alloc)(unsigned long size))  
{  
unsigned long addr, length, end, next;

addr = virt & PAGE_MASK;－－－－－－－－－－－－－－－－－－－－－－－－（1）  
length = PAGE_ALIGN(size + (virt & ~PAGE_MASK));

end = addr + length;  
do {－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－（2）  
next = pgd_addr_end(addr, end);－－－－－－－－－－－－－－－－－－－－（3）  
alloc_init_pud(mm, pgd, addr, next, phys, prot, alloc);－－－－－－－－－－（4）  
phys += next - addr;  
} while (pgd++, addr = next, addr != end);  
}
```

创建地址映射熟悉要明确地址空间，不同的进程有不同的地址空间，struct mm_struct就是描述一个进程的虚拟地址空间，当然，我们这里的场景是为内核虚拟地址空间而创建地址映射，因此传递的参数是init_mm。需要创建地址映射的起始虚拟地址是virt，该虚拟地址对应的PUD中的描述符是一个8B的内存，pgd就是指向这个描述符内存的指针。

（1）因为地址映射的最小单位是page，因此这里进行mapping的虚拟地址需要对齐到page size，同样的，长度也需要对齐到page size。经过对齐运算，（addr，length）定义的地址范围应该是囊括（virt，size）定义的地址范围，并且是对齐到page的。
（2）（addr，length）这个虚拟地址范围可能需要占据多个PGD entry，因此这里我们需要一个循环，不断的调用alloc_init_pud函数来完成（addr，length）这个虚拟地址范围的映射，当然，alloc_init_pud函数其实也会建立下游（例如PUD、PMD、PTE）翻译表的entry。
（3）pgd中的一个描述符只能mapping有限区域的虚拟地址（PGDIR_SIZE），pgd_addr_end的宏就是计算addr所在区域的end address。如果计算出来的end address小于传入的end地址参数，那么返回end参数值。也就是说，如果（addr，length）这个虚拟地址范围的mapping需要跨越多个pgd entry，那么next变量保存了下一个pgd entry的起始虚拟地址。
（4）这个函数有两个作用，一是填充pgd entry，二是创建后续的pud translation table（如果需要的话）并进行下游Translation table的建立。

# 五、分配PUD页表内存并填充相应的描述符

alloc_init_pud并非只是操作pud，实际上它是操作pgd的一个entry，并分配初始pud以及后续translation table的。填充PGD的entry需要给出对应PUD translation table的内存地址，如果PUD不存在，那么alloc_init_pud还需要分配PUD translation table（page size），只有得到PUD翻译表的物理内存地址，我们才能填充PGD entry。具体代码如下：

```cpp
static void alloc_init_pud(struct mm_struct mm, pgd_t *pgd,  
unsigned long addr, unsigned long end,  
phys_addr_t phys, pgprot_t prot,  
void *(*alloc)(unsigned long size))  
{  
pud_t *pud;  
unsigned long next;

if (pgd_none(*pgd)) {－－－－－－－－－－－－－－－－－－－－－－－－－－（1）  
pud = alloc(PTRS_PER_PUD * sizeof(pud_t));  
pgd_populate(mm, pgd, pud);  
}   
pud = pud_offset(pgd, addr); －－－－－－－－－－－－－－－－－－－－－（2）  
do { －－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－（3）  
next = pud_addr_end(addr, end);

if (use_1G_block(addr, next, phys)) { －－－－－－－－－－－－－－－－（4）  
pud_t old_pud = *pud;  
set_pud(pud, pud(phys | pgprot_val(mk_sect_prot(prot)))); －－－－－（5）

if (!pud_none(old_pud)) { －－－－－－－－－－－－－－－－－－－－－（6）  
flush_tlb_all(); －－－－－－－－－－－－－－－－－－－－－－－－（7）  
if (pud_table(old_pud)) {  
phys_addr_t table = __pa(pmd_offset(&old_pud, 0));  
if (!WARN_ON_ONCE(slab_is_available()))  
memblock_free(table, PAGE_SIZE); －－－－－－－－－－－－（8）  
}  
}  
} else {  
alloc_init_pmd(mm, pud, addr, next, phys, prot, alloc);  
}  
phys += next - addr;  
} while (pud++, addr = next, addr != end);  
}
```

（1）如果当前pgd entry是全0的话，说明还没有对应的下级PUD页表内存，因此需要进行PUD页表内存的分配。需要说明的是这时候，伙伴系统没有ready，分配内存仍然使用memblock模块，pgd_populate用来建立pgd entry和PUD 页表内存的关系。
（2）至此，pud的页表内存已经有了，但是addr对应PUD中的哪一个描述符呢？pud_offset给出了答案，其返回的指针指向传入参数addr地址对应的pud 描述符内存，而我们随后的任务就是填充pud entry了。
（3）虽然（addr，end）之间的虚拟地址范围共享一个pgd entry，但是这个地址范围对应的pud entry可能有多个，通过循环，逐一填充pud entry，同时分配并初始化下一阶页表。
（4）如果没有可能存在的1G block地址映射，这里的代码逻辑和上一节中的类似，只不过不断的循环调用alloc_init_pud改成alloc_init_pmd即可。然而，ARM64的MMU硬件提供了灰常强大的功能，系统支持1G size的block mapping，如果能够应用会获取非常多的好处：不需要分配下级的translation table节省了内存，更重要的是大大降低了TLB miss，提高了性能。既然这么好，当然要使用，不过有什么条件呢？首先系统配置必须是4k的page size，这种配置下，一个PUD entry可以覆盖1G的memory block。此外，起止虚拟地址以及映射到的物理地址都必须要对齐在1G size上。
（5）填写一个PUD描述符，一次搞定1G size的address mapping，没有PMD和PTE的页表内存，没有对PMD 和PTE描述符的访问，多么简单，多么美妙啊。假设系统内存4G，并且物理地址对齐在1G上（虚拟地址PAGE_OFFSET本来就是对齐在1G的），那么4个PUD的描述符就搞定了内核空间的线性地址映射区间。
（6）如果pud entry是非空的，那么就说明之前已经有了对该段地址的mapping（也许是只映射了部分）。一个简单的例子就是起始阶段的kernel image mapping，在\_\_create_page_tables创建pud 以及pmd中entry。如果不能进行section mapping，那么还建立了PTE中的描述符，现在这些描述符都没有用了，我们可以丢弃它们了。
（7）虽然建立了新的页表，但是旧的页表还残留在了TLB中，必须将其“赶尽杀绝”，清除出TLB。
（8）如果pud指向了一个table描述符，也就是说明该entry指向一个PMD table，那么需要释放其memory。

# 六、分配PMD页表内存并填充相应的描述符

1G block mapping虽好，不一定适合所有的系统，下面我一起看看PUD entry中填充的是block descriptor的情况（描述符指向PMD translation table）：

> static void alloc_init_pmd(struct mm_struct \*mm, pud_t \*pud,\
> unsigned long addr, unsigned long end,\
> phys_addr_t phys, pgprot_t prot,\
> void \*(\*alloc)(unsigned long size))\
> {\
> pmd_t \*pmd;\
> unsigned long next;
>
> if (pud_none(\*pud) || pud_sect(\*pud)) {－－－－－－－－－－－－－－－－－－－（1）\
> pmd = alloc(PTRS_PER_PMD * sizeof(pmd_t));－－－分配pmd页表内存\
> if (pud_sect(\*pud)) {－－－－－－－－－－－－－－－－－－－－－－－－－－（2）\
> split_pud(pud, pmd);\
> }\
> pud_populate(mm, pud, pmd);－－－－－－－－－－－－－－－－－－－－－（3）\
> flush_tlb_all();\
> }\
> BUG_ON(pud_bad(\*pud));
>
> pmd = pmd_offset(pud, addr);－－－－－－－－－－－－－－－－－－－－－－－（4）\
> do {\
> next = pmd_addr_end(addr, end);\
> if (((addr | next | phys) & ~SECTION_MASK) == 0) {－－－－－－－－－－－－（5）\
> pmd_t old_pmd =\*pmd;\
> set_pmd(pmd, \_\_pmd(phys | pgprot_val(mk_sect_prot(prot))));
>
> if (!pmd_none(old_pmd)) {－－－－－－－－－－－－－－－－－－－－－－（6）\
> flush_tlb_all();\
> if (pmd_table(old_pmd)) {\
> phys_addr_t table = \_\_pa(pte_offset_map(&old_pmd, 0));\
> if (!WARN_ON_ONCE(slab_is_available()))\
> memblock_free(table, PAGE_SIZE);\
> }\
> }\
> } else {\
> alloc_init_pte(pmd, addr, next, \_\_phys_to_pfn(phys),\
> prot, alloc);\
> }\
> phys += next - addr;\
> } while (pmd++, addr = next, addr != end);\
> }

（1）有两个场景需要分配PMD的页表内存，一个是该pud entry是空的，我们需要分配后续的PMD页表内存。另外一个是旧的pud entry是section 描述符，映射了1G的address block。但是现在由于种种原因，我们需要修改它，故需要remove这个1G block的section mapping。

（2）虽然是建立新的mapping，但是原来旧的1G mapping也要保留的，也许这次我们只是想更新部分地址映射呢。在这种情况下，我们先通过split_pud函数调用把一个1G block mapping转换成通过pmd进行mapping的形式（一个pud的section mapping描述符（1G size）变成了512个pmd中的section mapping描述符（2M size）。形式变了，味道不变，加量不加价，仍然是1G block的地址映射。

（3）修正pud entry，让其指向新的pmd页表内存，同时flush tlb的内容。

（4）下面这段代码的逻辑起始是和alloc_init_pud是类似的。如果不能进行2M的section mapping，那么就循环调用alloc_init_pte进行地址的mapping，这里我们就不多说了，重点看看2M section mapping的处理。

（5）如果满足2M section的要求，那么就调用set_pmd填充pmd entry。

（6）如果有旧的section mapping，并且指向一个PTE table，那么还需要释放这些不需要的PTE页表描述符占用的内存。

七、分配PTE页表内存并填充相应的描述符

> static void alloc_init_pte(pmd_t \*pmd, unsigned long addr,\
> unsigned long end, unsigned long pfn,\
> pgprot_t prot,\
> void \*(\*alloc)(unsigned long size))\
> {\
> pte_t \*pte;
>
> if (pmd_none(\*pmd) || pmd_sect(\*pmd)) {－－－－－－－－－－－－－－－－（1）\
> pte = alloc(PTRS_PER_PTE * sizeof(pte_t));\
> if (pmd_sect(\*pmd))\
> split_pmd(pmd, pte);－－－－－－－－－－－－－－－－－－－－－－（2）\
> \_\_pmd_populate(pmd, \_\_pa(pte), PMD_TYPE_TABLE);－－－－－－－－（3）\
> flush_tlb_all();\
> }\
> BUG_ON(pmd_bad(\*pmd));
>
> pte = pte_offset_kernel(pmd, addr);\
> do {\
> set_pte(pte, pfn_pte(pfn, prot));－－－－－－－－－－－－－－－－－－－（4）\
> pfn++;\
> } while (pte++, addr += PAGE_SIZE, addr != end);\
> }

（1）走到这个函数，说明后续需要建立PTE这一个level的页表描述符，因此，需要分配PTE页表内存，场景有两个，一个是从来没有进行过映射，另外一个是已经建立映射，但是是section mapping，不符合要求。

（2）如果之前有section mapping，那么我们需要将其分裂成512个pte中的page descriptor。

（3）让pmd entry指向新的pte页表内存。需要说明的是：如果之前pmd entry是空的，那么新的pte页表中有512个invalid descriptor，如果之前有section mapping，那么实际上这个新的PTE页表已经通过split_pmd填充了512个page descritor。

（4）循环设定（addr，end）这段地址区域的pte中的page descriptor。

参考文献：

1、ARMv8技术手册

2、Linux 4.4.6内核源代码

_原创文章，转发请注明出处。蜗窝科技_

标签: [create_mapping](http://www.wowotech.net/tag/create_mapping)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [蓝牙协议分析(9)\_BLE安全机制之LL Privacy](http://www.wowotech.net/bluetooth/ble_ll_privacy.html) | [X-018-KERNEL-串口驱动开发之serial console](http://www.wowotech.net/x_project/serial_driver_porting_3.html)»

**评论：**

**kidesire**\
2020-05-07 19:50

文章很好, 受益匪浅. 从研究内核的角度出发, 有一处细节不知是否还有更精确的描述: "从集合论的角度看，reserved type类型的数组是memory type类型的数组的真子集", 有两个例子(如有考虑不全的地方, 希望指出):

一. 如果reserved-memory中有内除区A带有no-map标签, 那么这段地址会从memory region中移除. 这种情况下A在reserved-memory region中, A不在memory region中, 这时"reserved type类型的数组是memory type类型的数组的真子集"说法不符合当前情况.

二. 以下是源码https://elixir.bootlin.com/linux/latest/source/arch/arm64/boot/dts/hisilicon/hi6220-hikey.dts

memory@0 {\
device_type = "memory";\
reg = \<0x00000000 0x00000000 0x00000000 0x05e00000>,\
\<0x00000000 0x05f00000 0x00000000 0x00001000>,\
\<0x00000000 0x05f02000 0x00000000 0x00efd000>,\
\<0x00000000 0x06e00000 0x00000000 0x0060f000>,\
\<0x00000000 0x07410000 0x00000000 0x1aaf0000>,\
\<0x00000000 0x22000000 0x00000000 0x1c000000>;\
};

reserved-memory {\
#address-cells = \<2>;\
#size-cells = \<2>;\
ranges;

ramoops@21f00000 {\
compatible = "ramoops";\
reg = \<0x0 0x21f00000 0x0 0x00100000>;\
record-size    = \<0x00020000>;\
console-size    = \<0x00020000>;\
ftrace-size    = \<0x00020000>;\
};

/\* global autoconfigured region for contiguous allocations \*/\
linux,cma {\
compatible = "shared-dma-pool";\
reusable;\
size = \<0x00000000 0x08000000>;\
linux,cma-default;\
};\
};\
此源码中ramoops的保留内存区域不在memory表达的范围.

以上是我的观察, 期待指导.

[回复](http://www.wowotech.net/memory_management/mem_init_3.html#comment-7986)

**zxq**\
2021-10-13 14:33

@kidesire：我也发现了这个问题，别的不说，你可以看一下这个函数int \_\_init \_\_weak early_init_dt_alloc_reserved_memory_arch(phys_addr_t size,\
phys_addr_t align, phys_addr_t start, phys_addr_t end, bool nomap,\
phys_addr_t *res_base)\
{\
phys_addr_t base;\
/*\
\* We use \_\_memblock_alloc_base() because memblock_alloc_base()\
\* panic()s on allocation failure.\
\*/\
end = !end ? MEMBLOCK_ALLOC_ANYWHERE : end;\
base = \_\_memblock_alloc_base(size, align, end);\
if (!base)\
return -ENOMEM;

/\*\
\* Check if the allocated region fits in to start..end window\
\*/\
if (base \< start) { //\_\_memblock_alloc_base不保证返回的base大于start，这里再次检查\
memblock_free(base, size);\
return -ENOMEM;\
}

*res_base = base;\
/*\
\* 如果nomap属性，则将其从memory中移除，但是这里并没有从reserve区域移除，\
\* 那么reserve不就不是memory的真子集了吗？\
\*/\
if (nomap)\
return memblock_remove(base, size);\
return 0;\
}

\_\_memblock_alloc_base调用将找到的base区域添加到了reserve区域，但是在函数最后，如果有nomap属性，则将其从memory区域移除了，这样看reserve区域并不满足是memory区域真子集的说法了。

[回复](http://www.wowotech.net/memory_management/mem_init_3.html#comment-8332)

**kidesire**\
2020-05-07 19:16

由于我一直以来认为mmu是通过页表的虚拟地址来访问页表的,这个想法可能是错的.由于一直以来的可能错误的想法给我阅读源码和博客时造成困惑, 因此重置固有想法,再进行思考.\
假设mmu访问页表是通过物理地址直接访问的而不通过虚拟地址访问页表,也即ttbr0,ttbr1存储的是页表的物理地址, 以及各个pgd/pud/pmd/pte的entry中存储的也是物理地址相关值, 那么此问题就迎刃而解了.\
因此我再次思考, mmu可以直接读取物理地址, 不需要通过虚拟地址转换. 并且重新查看\_\_enable_mmu代码, 发现,\
ENTRY(\_\_enable_mmu)\
...\
adrp    x2, idmap_pg_dir\
phys_to_ttbr x1, x1\
phys_to_ttbr x2, x2\
msr    ttbr0_el1, x2            // load TTBR0\
msr    ttbr1_el1, x1            // load TTBR1\
...\
ENDPROC(\_\_enable_mmu)

#ifdef CONFIG_ARM64_PA_BITS_52\
#define phys_to_ttbr(addr)    (((addr) | ((addr) >> 46)) & TTBR_BADDR_MASK_52)\
#else\
#define phys_to_ttbr(addr)    (addr)\
#endif

ttbr中确实为物理地址. 同时在paging_init实现中各级页表的entry存放的也是物理地址相关值.

不知以上描述是否正确或者说是否准确.

[回复](http://www.wowotech.net/memory_management/mem_init_3.html#comment-7985)

**[smcdef](http://www.wowotech.net/)**\
2020-05-12 11:52

@kidesire：正确，如果MMU通过虚拟地址访问的话，岂不是陷入了“先有鸡还是先有蛋”的问题。

[回复](http://www.wowotech.net/memory_management/mem_init_3.html#comment-7989)

**[smcdef](http://www.wowotech.net/)**\
2020-05-12 11:52

@kidesire：正确，如果MMU通过虚拟地址访问的话，岂不是陷入了“先有鸡还是先有蛋”的问题。

[回复](http://www.wowotech.net/memory_management/mem_init_3.html#comment-7990)

**kidesire**\
2020-05-07 17:02

你好，感谢分享。有些疑问求教下,我原本的理解是,假设内核现在需要访问虚拟地址为virt对应的物理地址为phys上的内容, 那么MMU通过存放内核页表的寄存器得的内核页表swapper_pg_dir的虚拟地址即PGD页表基地址,然后根据virt计算对应的PGD entry(PGD页表基地址 + virt计算得到的offset, 后面PUD/PMD类似),从PGD entry中得到PUD页表基地址的虚拟地址,然后根据virt计算对应的PUD entry, 从PUD entry中得到PMD页表基地址的虚拟地址,然后根据virt计算对应的PMD entry,从PMD entry中得到PTE页表基地址的虚拟地址,然后根据virt计算PTE entry,从PTE entry中得到phys所在的物理页帧地址,加上virt计算得到的偏移后得到virt对应的物理地址,执行访问指令.经过上述描述,我一直有一个问题不解,PGD/PUD/PMD 的各个entry中存放的应该是下级页表基地址的虚拟地址才对,文章中和源码中却是 phys|port.不知我的思考哪里出了问题, 还望指点,万分感谢.

[回复](http://www.wowotech.net/memory_management/mem_init_3.html#comment-7981)

**casio8040**\
2019-03-14 23:04

有点疑问，如果是三级映射在create_mapping中也要alloc_init_pud,分配PUD页表内存并填充相应的描述符，这有用吗，三级映射不是没有pud吗？

[回复](http://www.wowotech.net/memory_management/mem_init_3.html#comment-7282)

**cs350203**\
2019-03-08 17:23

hell  linuxer

一直不是很明白memblock_reserve 和memblock_remove      memblock_reserve是被操作系统保留了，但是还会在系统中建立映射关系，而memblock_remove表示这段memory会从系统中之间删除，mam_mem不会建立映射关系，换句话税就是被memblock_remove 的内存是不是就没办法访问了。但是我实际测试发现用memblock_remove溢出一段物理内存后，在系统起来之后用/dev/mem还能访问，难道是我用/dev/mem访问重新建立了页表关系吗！

[回复](http://www.wowotech.net/memory_management/mem_init_3.html#comment-7255)

**[linuxer](http://www.wowotech.net/)**\
2019-03-12 09:14

@cs350203：其实你可以自己在内核代码中找到答案的......

[回复](http://www.wowotech.net/memory_management/mem_init_3.html#comment-7270)

**神品江湖**\
2017-11-06 17:29

# hello linuxer

# 如果当前pgd entry是全0的话，说明还没有对应的下级PUD页表内存，因此需要进行PUD页表内存的分配。需要说明的是这时候，伙伴系统没有ready，分配内存仍然使用memblock模块，pgd_populate用来建立pgd entry和PUD 页表内存的关系。

上文说分配内存仍然使用memblock模块，memblock模块分配的是物理内存，这个内存是没有建立页表映射的，那如何访问它呢？

[回复](http://www.wowotech.net/memory_management/mem_init_3.html#comment-6176)

**vedic**\
2018-11-02 16:43

@神品江湖：页表存储的就是物理地址， memblock里面记录的也是物理地址， 所以刚好

[回复](http://www.wowotech.net/memory_management/mem_init_3.html#comment-7011)

**明天**\
2021-07-16 16:21

@神品江湖：我理解memblock申请的内存是物理地址，要将该地址填充到页表中，页表就是需要物理地址的，所以该地址也就可以通过页表访问了

[回复](http://www.wowotech.net/memory_management/mem_init_3.html#comment-8252)

**时间为贵**\
2017-10-31 09:19

@wowo\
如此，前面文章中的部分表述是否不太合适？\
==>\
内存初始化代码分析（二）：内存布局\
......\
（3）size等于0的memory region表示这是一个动态分配region，base address尚未定义，因此我们需要通过\_\_reserved_mem_alloc_size函数对节点进行分析（size、alignment等属性），然后调用memblock的alloc接口函数进行memory block的分配，最终的结果是确定base address和size，并将这段memory region从memory type的数组中移到reserved type的数组中。当然，如果定义了no-map属性，那么这段memory会从系统中之间删除（memory type和reserved type数组中都没有这段memory的定义）。

[回复](http://www.wowotech.net/memory_management/mem_init_3.html#comment-6153)

**时间为贵**\
2017-10-12 07:35

@linuxer\
memory type类型的数组减去reserved type类型的数组，从集合论的角度看，reserved type类型的数组是memory type类型的数组的真子集\
==》为什么会是真子集？在membloc_reserve的时候不会将对应region从memory type中移除吗？

[回复](http://www.wowotech.net/memory_management/mem_init_3.html#comment-6103)

**[wowo](http://www.wowotech.net/)**\
2017-10-16 16:49

@时间为贵：至于是不是子集，可以先从debugfs中看看结论，例如：\
flo:/ $ cat /sys/kernel/debug/memblock/memory\
0: 0x80200000..0x88dfffff\
1: 0x8a000000..0x8d9fffff\
2: 0x8ec00000..0x8effffff\
3: 0x8f700000..0x8fdfffff\
4: 0x90000000..0x9fdfffff\
5: 0xa5d00000..0xfe9fefff\
flo:/ $ cat /sys/kernel/debug/memblock/reserved\
0: 0x80204000..0x80207fff\
1: 0x80208180..0x81580efb\
2: 0x82200000..0x823c42c4\
3: 0x88d00000..0x88dfffff\
4: 0xa9fdb000..0xa9ffefff\
5: 0xa9fff8c0..0xaaffffff\
至于为什么不移除，也很好理解，memory就是所有的memory，reserve就是从所有的memory里面reserve一点。

[回复](http://www.wowotech.net/memory_management/mem_init_3.html#comment-6115)

**Murphy**\
2017-04-28 17:43

Hi linuxer，\
在ARM的页表中有一个domain区域，在我的理解中用户进程的0~3G区间的全局页目录中的domain区域应该是 DOMAIN_USER ,但我在内核中没找到设定用户进程页目录domain区域的地方，不知能否告知下？

[回复](http://www.wowotech.net/memory_management/mem_init_3.html#comment-5500)

**langyijiang**\
2017-03-15 09:28

hello linuxer\
目前项目中DDR是512M的，设备树中使用448M作为系统内存，留下64M做特殊用处，当在内核中特殊处理完成以后，我想将64M空间返还给内核，请问有什么策略吗？\
望回复！！！！！！！！！

[回复](http://www.wowotech.net/memory_management/mem_init_3.html#comment-5317)

**[linuxer](http://www.wowotech.net/)**\
2017-03-16 09:18

@langyijiang：使用free_reserved_area这个接口函数可以将reserved page归还给伙伴系统。

[回复](http://www.wowotech.net/memory_management/mem_init_3.html#comment-5325)

**langyijiang**\
2017-04-01 09:51

@linuxer：不好意思，这段时间有事耽搁了，没看到回复哈。\
这64M在系统起来的时候，并没有被MMU管理，使用这个接口释放内存不会是系统崩溃么？

[回复](http://www.wowotech.net/memory_management/mem_init_3.html#comment-5408)

**Murphy**\
2017-04-28 17:47

@langyijiang：不会，前提是要将这64M内存在内核中设为保留

[回复](http://www.wowotech.net/memory_management/mem_init_3.html#comment-5501)

1 [2](http://www.wowotech.net/memory_management/mem_init_3.html/comment-page-2#comments)

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

  - [X-009-KERNEL-Linux kernel的移植(Bubblegum-96平台)](http://www.wowotech.net/x_project/bubblegum_kernel_porting.html)
  - [蜗窝流量地域统计](http://www.wowotech.net/168.html)
  - [Unix的历史](http://www.wowotech.net/tech_discuss/Unix%E7%9A%84%E5%8E%86%E5%8F%B2.html)
  - [进程切换分析（3）：同步处理](http://www.wowotech.net/process_management/scheudle-sync.html)
  - [Linux common clock framework(2)\_clock provider](http://www.wowotech.net/pm_subsystem/clock_provider.html)

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

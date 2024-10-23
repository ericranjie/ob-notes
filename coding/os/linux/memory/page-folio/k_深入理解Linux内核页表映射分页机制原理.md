
Original 徐琪 Linux内核之旅 _2024年09月23日 21:30_ _陕西_

徐琪，西安邮电大学研一在读，操作系统和Linux内核爱好者，热衷于探索操作系统底层工作原理和内核编程。

# 一、寻址机制

最初8086处理器使用的是实地址，后来Intel为解决地址宽度不足的问题从而引入分段机制，再后来为进一步保护数据又引入分页机制，从而衍生出MMU、CRn等寄存器和物理单元，演变为至今的分段加分页的寻址系统

![[Pasted image 20241007092117.png]]

## 1.1 段式寻址

当起初8086寻址范围64K太小，于是Intel将其扩展到1MB，即20位地址宽度。为此Intel发明了一种巧妙的方法，即分段。在CPU中设置了四个段寄存器：CS、DS、SS、ES，用于访问指令、数据、堆栈和其他。将内存对应划分为多段，用段寄存器配合偏移量来完成寻址。

### **1.1.1 实模式：**

8086处理器上写汇编语言时，访问一个内存的两步：1.将段地址写入DS寄存器，将偏移量写入BX寄存器。2.使用\[DS：BX\]组合完成寻址，这就是段式寻址。此时DS：BX的组合称为逻辑地址，经过分段单元的硬件电路转化为线性地址。

缺点：
这种寻址方式存在安全风险，任何进程都能访问所有地址空间（实模式）

### **1.1.2 保护模式：**

![[Pasted image 20241006090124.png]]

- 保护模式下段寄存器中不再是段地址，而是一个段选择子
- 逻辑地址由16位段选择符和32位偏移量组成，段选择符存放段寄存器中。有六个段寄存器，分别是cs,ss,ds,es,fs和gs。每个段选择符有一个TI位表示是哪个描述符表，RPL位表示访问权限，13位索引号字段表示是段描述符表中的哪一个。
- 有两类段描述符表：GDT和LDT；GDT的地址和大小再gdtr控制寄存器定义，LDT在ldtr控制寄存器中定义
- 此时访问一个地址，先将该地址所在段的段选择子放入段寄存器，根据索引字段找到段描述符，找到段基址，再加上偏移量，就转换成了线性地址；在这个过程中，通过段访问权限，可以控制进程无法访问非法地址

![[Pasted image 20241006090134.png]]

## 1.2 页式寻址

### 1.2.1 分页机制作用

- 将线性地址转换成物理地址
- 将大小不同的大内存段拆分为大小相等的小内存块

### 1.2.2 一级页表

32位地址表示4GB空间，CPU采用的页大小定为4KB，那么4GB地址空间被划分为4GB/4KB=1M个页；那么4GB地址空间可以将32位地址分为高低两部分；虚拟地址高20位用来索引一个页，低12位用来页内寻址。

![[Pasted image 20241006090147.png]]
**线性地址转换为物理地址**

线性地址高20位作为页表项的索引，每个页表项占用4字节大小，故高20位的索引乘以4后才是该页表项相对于页表物理地址的字节偏移量。用CR3寄存器中的页表物理地址加上此偏移量便是该页表项的物理地址，从该页表项中得到映射的物理地址，然后再用线性地址的低12位与该物理页地址相加，所得地址之和便是最终要访问的物理地址。

![[Pasted image 20241006090157.png]]

### 1.2.3 二级页表

- 将每个页表的物理地址存放在页目录表中都以页目录项（PDE）的形式存储，页目录项大小同页表项一样都为4KB，PDE用来描述一个物理页的物理地址

- 页目录表和所有页表都放在物理内存中
  ![[Pasted image 20241006090204.png]]

- 页目录项和页表项
  页目录项和页表项都是4字节大小，用来存储物理页地址。具体结构如下图所示：
  ![[Pasted image 20241006090227.png]]

**二级页表的地址转换原理**

二级页表地址转换原理是将32位虚拟地址拆分为高10位、中间10位、低12位三部分；

- 高10位作为页表的索引，用于在页目录表中定为一个页目录项PDE，页目录项中有页表的物理地址，也就是定位到了某个页表
- 中间10位作为物理页的索引，用于在页表内定位到某个页表项篇TE，页表项中有分配的物理页地址，也就是定位到了某个物理页
- 低12位作为页内偏移量用于在已经定位到的物理页寻址

![[Pasted image 20241006090234.png]]

### 1.2.4 启动分页机制的3个步骤：

1）准备好页目录表及页表
2）将页表地址写入控制寄存器CR3
3）寄存器CR0的PG位置1

# 二、分页管理代码分析

## 2.1 X86架构下的分页管理分析

以下是内核Linux5.6.4版本四级分页模型介绍：

### 2.1.1 PGDIR_SHIFT及相关的宏

- 表示线性地址中的offset字段，Table字段，Middle Dir字段和Upper Dir 字段，PGDIR_SIZE用于计算页全局目录中一个表项能映射区域的大小。PGDIR_MASK用于屏蔽线性地址中Middle Dir字段、Table字段和offset字段所在位。
- 在四级分页模型中，PGDIR_SHIFT占据39位，即9位页上级目录、9位页中间目录、9位页表和12位偏移。页全局目录同样占线性地址的9位，因此PTRS_PER_PGD（表示的是PGD对应的页表中有多少个表项）为512。

```cpp
arch/x86/include/asm/pgtable_64_types.h      
#define PGDIR_SHIFT 39      #define PTRS_PER_PGD 512      #define PGDIR_SIZE (_AC(1, UL) << PGDIR_SHIFT)      #define PGDIR_MASK (~(PGDIR_SIZE - 1))      
```

**pgd_offset**该函数返回线性地址address在页全局目录中对应表项的线性地址。mm为指向一个内存描述符的指针，address为要转换的线性地址。该宏最终返回address在页全局目录中相应表项的线性地址。

```
#define pgd_index(address)	(((address) >> PGDIR_SHIFT) & (PTRS_PER_PGD-1))   
#define pgd_offset(mm, address)	((mm)->pgd+pgd_index(address)) 
```

### 2.1.2 PUD_SHIFT及相关的宏

- 表示线性地址中offset字段、Table字段和Middle Dir字段的位数。PUD_SIZE用于计算页上级目录一个表项映射的区域大小，PUD_MASK用于屏蔽线性地址中Middle Dir字段、Table字段和offset字段所在位。
- 在64位系统四级分页模型下，PUD_SHIFT的大小为30，包括12位的offset字段、9位Table字段和9位Middle Dir字段。由于页上级目录在线性地址中占9位，因此页上级目录的表项数为512。

```cpp
arch/x86/include/asm/pgtable_64_types.h      
#define PUD_SHIFT 30      #define PTRS_PER_PUD 512      #define PUD_SIZE        (_AC(1, UL) << PUD_SHIFT)      #define PUD_MASK        (~(PUD_SIZE - 1))    
```

**pud_offset**

该函数与pgd_offset类似，最终得到address对应的页上级目录项的线性地址。

```cpp
#define pud_offset(dir,addr) \      	((pud_t *) pgd_page_vaddr(*(dir)) + (((addr) >> PUD_SHIFT) & (PTRS_PER_PUD - 1)))      
#endif   
```

### 2.1.3 PMD_SHIFT及相关宏

- 表示线性地址中offset字段和Table字段的位数，2的PMD_SHIFT次幂表示一个页中间目录项可以映射的内存区域大小。PMD_SIZE用于计算这个区域的大小，PMD_MASK用来屏蔽offset字段和Table字段的所有位。PTRS_PER_PMD表示页中间目录中表项的个数。

- 在64位系统中，Linux采用四级分页模型。线性地址包含页全局目录、页上级目录、页中间目录、页表和偏移量五部分。在这两种模型中PMD_SHIFT占21位，即包括Table字段的9位和offset字段的12位。PTRS_PER_PMD的值为512，即2的9次幂，表示页中间目录包含的表项个数。

```
#define PMD_SHIFT 21      #define PTRS_PER_PMD 512      #define PMD_SIZE (_AC(1, UL) << PMD_SHIFT)      #define PMD_MASK (~(PMD_SIZE - 1))   
```

**pmd_offset**

该函数返回address在页中间目录中对应表项的线性地址。

### 2.1.4 PAGE_SHIFT及相关宏

- 表示线性地址offset字段的位数。该宏的值被定义为12位，即页的大小为4KB。与它对应的宏有PAGE_SIZE，它返回一个页的大小；PAGE_MASK用来屏蔽offset字段，其值为oxfffff000。PTRS_PER_PTE表明页表在线性地址中占据9位。
- 通过上面的分析可知，在x86-64架构下64位的线性地址被划分为五部分，每部分占据的位数分别为9，9，9，9，12，实际上只用了64位中的48位。对于四级页表而言，级别从高到底每级页表中表项的个数为512，512，512，512。

## 2.2 ARM架构下的分页管理分析

### 2.2.1 虚拟地址到物理地址的转换

ARMv8中，Kernel Space的页表基地址存放在`TTBR1_EL1`寄存器中，User Space页表基地址存放在`TTBR0_EL0`寄存器中，其中内核地址空间的高位为全1，(0xFFFF0000_00000000 ~ 0xFFFFFFFF_FFFFFFFF)，用户地址空间的高位为全0，(0x00000000_00000000 ~ 0x0000FFFF_FFFFFFFF)

结合有效虚拟地址位， 页面大小，页表的级数，可以组合成不同的页表映射方式。以下以内核配置为：39位有效位，4KB大小页面，3级页表来介绍

![[Pasted image 20241006090437.png]]

1. 虚拟地址\[63:39\]用于区分内核空间与用户空间，从而选择不同的TTBRn寄存器来获取Level 1页表基地址；
1. 虚拟地址\[38:30\]放置Level 1页表中的索引，从而找到对应的描述符地址并获取描述符内容，根据描述符中的内容获取Level 2页表基地址;
1. 虚拟地址\[29:21\]Level 2页表中的索引，从而找到对应的描述符地址并获取描述符内容，根据描述符中的内容获取Level 3页表基地址;
1. 虚拟地址\[20:12\]Level 3页表中的索引，从而找到对应的描述符地址并获取描述符内容，根据描述符中的内容获取物理地址的高36位，以4K地址对齐；
1. 虚拟地址\[11:0\]放置的是物理地址的偏移，结合获取的物理地址高位，最终得到物理地址。

### 2.2.2 Linux页表映射

内核中关于页表的操作如图所示：

![[Pasted image 20241006090451.png]]

代码路径：

arch/arm64/include/asm/pgtable-types.h：定义`pgd_t, pud_t, pmd_t, pte_t`等类型；
arch/arm64/include/asm/pgtable-prot.h：针对页表中entry中的权限内容设置；
arch/arm64/include/asm/pgtable-hwdef.h：主要包括虚拟地址中PGD/PMD/PUD等的划分，这个与虚拟地址的有效位及分页大小有关，此外还包括硬件页表的定义， TCR寄存器中的设置等；arch/arm64/include/asm/pgtable.h：页表设置相关；

在这些代码中可以看到，

- 当`CONFIG_PGTABLE_LEVELS=4`时：`pgd-->pud-->pmd-->pte`;
- 当`CONFIG_PGTABLE_LEVELS=3`时，没有`PUD`页表：`pgd(pud)-->pmd-->pte`;
- 当`CONFIG_PGTABLE_LEVELS=2`时，没有`PUD`和`PMD`页表：`pgd(pud, pmd)-->pte`

常用的宏定义

![[Pasted image 20241006090503.png]]

页表处理

```cpp
/*描述各级页表中的页表项*/   
typedef struct { pteval_t pte; } pte_t;   typedef struct { pmdval_t pmd; } pmd_t;   typedef struct { pudval_t pud; } pud_t;   typedef struct { pgdval_t pgd; } pgd_t;      /*  将页表项类型转换成无符号类型 */   #define pte_val(x) ((x).pte)   #define pmd_val(x) ((x).pmd)   #define pud_val(x) ((x).pud)   #define pgd_val(x) ((x).pgd)      /*  将无符号类型转换成页表项类型 */   #define __pte(x) ((pte_t) { (x) } )   #define __pmd(x) ((pmd_t) { (x) } )   #define __pud(x) ((pud_t) { (x) } )   #define __pgd(x) ((pgd_t) { (x) } )      /* 获取页表项的索引值 */   #define pgd_index(addr)  (((addr) >> PGDIR_SHIFT) & (PTRS_PER_PGD - 1))   #define pud_index(addr)  (((addr) >> PUD_SHIFT) & (PTRS_PER_PUD - 1))   #define pmd_index(addr)  (((addr) >> PMD_SHIFT) & (PTRS_PER_PMD - 1))   #define pte_index(addr)  (((addr) >> PAGE_SHIFT) & (PTRS_PER_PTE - 1))      /*  获取页表中entry的偏移值 */   #define pgd_offset(mm, addr) (pgd_offset_raw((mm)->pgd, (addr)))   #define pgd_offset_k(addr) pgd_offset(&init_mm, addr)   #define pud_offset_phys(dir, addr) (pgd_page_paddr(*(dir)) + pud_index(addr) * sizeof(pud_t))   #define pud_offset(dir, addr)  ((pud_t *)__va(pud_offset_phys((dir), (addr))))   #define pmd_offset_phys(dir, addr) (pud_page_paddr(*(dir)) + pmd_index(addr) * sizeof(pmd_t))   #define pmd_offset(dir, addr)  ((pmd_t *)__va(pmd_offset_phys((dir), (addr))))   #define pte_offset_phys(dir,addr) (pmd_page_paddr(READ_ONCE(*(dir))) + pte_index(addr) * sizeof(pte_t))   #define pte_offset_kernel(dir,addr) ((pte_t *)__va(pte_offset_phys((dir), (addr)))) 
```

### 2.2.3 head.S中的页表映射

下面来介绍页表的创建过程，代码路径：arch/arm64/kernel/head.S
在`head.S`中，创建页表相关的有三个宏：

1. `create_pgd_entry`

```cpp
/*    * Macro to populate the PGD (and possibily PUD) for the corresponding    * block entry in the next level (tbl) for the given virtual address.    *    * Preserves: tbl, next, virt    * Corrupts: tmp1, tmp2    */    .macro create_pgd_entry, tbl, virt, tmp1, tmp2    create_table_entry \tbl, \virt, PGDIR_SHIFT, PTRS_PER_PGD, \tmp1, \tmp2   #if SWAPPER_PGTABLE_LEVELS > 3    create_table_entry \tbl, \virt, PUD_SHIFT, PTRS_PER_PUD, \tmp1, \tmp2   #endif   #if SWAPPER_PGTABLE_LEVELS > 2    create_table_entry \tbl, \virt, SWAPPER_TABLE_SHIFT, PTRS_PER_PTE, \tmp1, \tmp2   #endif    .endm      
```

上述函数主要是调用`create_table_entry`，由于`SWAPPER_PGTABLES`配置为3，因此相当于创建了`pgd和pmd`两级页表，此处需要注意一点，`create_table_entry`函数执行后，`tbl`参数会自动加上`PAGE_SIZE`，也就是说`pgd和pmd`两级页表是物理连续的。

2. `create_block_map`

```cpp
/*    * Macro to populate block entries in the page table for the start..end    * virtual range (inclusive).    *    * Preserves: tbl, flags    * Corrupts: phys, start, end, pstate    */    .macro create_block_map, tbl, flags, phys, start, end    lsr \phys, \phys, #SWAPPER_BLOCK_SHIFT    lsr \start, \start, #SWAPPER_BLOCK_SHIFT    and \start, \start, #PTRS_PER_PTE - 1 // table index
 orr \phys, \flags, \phys, lsl #SWAPPER_BLOCK_SHIFT // table entry
 lsr \end, \end, #SWAPPER_BLOCK_SHIFT    and \end, \end, #PTRS_PER_PTE - 1  // table end index   9999: str \phys, [\tbl, \start, lsl #3]  // store the entry    add \start, \start, #1   // next entry    add \phys, \phys, #SWAPPER_BLOCK_SIZE  // next block    cmp \start, \end    b.ls 9999b    .endm      
```

3. `create_table_entry`

```cpp
/*    * Macro to create a table entry to the next page.    *    * tbl: page table address    * virt: virtual address    * shift: #imm page table shift    * ptrs: #imm pointers per table page    *    * Preserves: virt    * Corrupts: tmp1, tmp2    * Returns: tbl -> next level table page address    */    .macro create_table_entry, tbl, virt, shift, ptrs, tmp1, tmp2    lsr \tmp1, \virt, #\shift    and \tmp1, \tmp1, #\ptrs - 1 // table index    add \tmp2, \tbl, #PAGE_SIZE    orr \tmp2, \tmp2, #PMD_TYPE_TABLE // address of next table and entry type    str \tmp2, [\tbl, \tmp1, lsl #3]    add \tbl, \tbl, #PAGE_SIZE  // next level table page    .endm      
```

上述三个函数创建页表项，并且返回下一个Level的页表地址

![[Pasted image 20241006090617.png]]

# 三、动手实践

基于上面的分析，编写内核模块，获取一个线性地址对应的物理地址
首先写一个测试程序获取其虚拟地址

```cpp
#include <stdio.h>   
#include <stdlib.h>   
int main(void)   {    char *p = NULL;    p = malloc(10);    printf("address = 0x%x\n",p);    while(1);    return 0;   }   
```

以下是内核模块的整个代码：

```cpp
#include  <linux/module.h>    
#include <linux/kernel.h>    
#include <linux/init.h>    
#include <linux/sched.h>    
#include <linux/pid.h>    
#include <linux/mm.h>    
#include <asm/pgtable.h>    
#include <asm/page.h>       
MODULE_AUTHOR("wang.com");   MODULE_DESCRIPTION("vitual address to physics address");      static int pid;    static unsigned long va;       module_param(pid,int,0644); //从命令行传递参数（变量，类型，权限）   
module_param(va,ulong,0644); //va表示的是虚拟地址      
static int find_pgd_init(void)    {
unsigned long pa = 0; //pa表示的物理地址           
struct task_struct *pcb_tmp = NULL;            pgd_t *pgd_tmp = NULL;            pud_t *pud_tmp = NULL;            pmd_t *pmd_tmp = NULL;            pte_t *pte_tmp = NULL;               printk(KERN_INFO"PAGE_OFFSET = 0x%lx\n",PAGE_OFFSET);  //页表中有多少个项   		/*pud和pmd等等  在线性地址中占据多少位*/           
printk(KERN_INFO"PGDIR_SHIFT = %d\n",PGDIR_SHIFT);    		//注意：在32位系统中  PGD和PUD是相同的           
printk(KERN_INFO"PUD_SHIFT = %d\n",PUD_SHIFT);            printk(KERN_INFO"PMD_SHIFT = %d\n",PMD_SHIFT);            printk(KERN_INFO"PAGE_SHIFT = %d\n",PAGE_SHIFT);               printk(KERN_INFO"PTRS_PER_PGD = %d\n",PTRS_PER_PGD); //每个PGD里面有多少个ptrs           
printk(KERN_INFO"PTRS_PER_PUD = %d\n",PTRS_PER_PUD);            printk(KERN_INFO"PTRS_PER_PMD = %d\n",PTRS_PER_PMD); //PMD中有多少个项           
printk(KERN_INFO"PTRS_PER_PTE = %d\n",PTRS_PER_PTE);               printk(KERN_INFO"PAGE_MASK = 0x%lx\n",PAGE_MASK); //页的掩码      	
struct pid *p = NULL;   	p = find_vpid(pid); //通过进程的pid号数字找到
struct pid的结构体   	pcb_tmp = pid_task(p,PIDTYPE_PID); //通过pid的结构体找到进程的task  struct           
printk(KERN_INFO"pgd = 0x%p\n",pcb_tmp->mm->pgd);                   // 判断给出的地址va是否合法(va&lt;vm_end)     	if(!find_vma(pcb_tmp->mm,va)){                    printk(KERN_INFO"virt_addr 0x%lx not available.\n",va);                    return 0;            }            pgd_tmp = pgd_offset(pcb_tmp->mm,va);  //返回线性地址va，在页全局目录中对应表项的线性地址           printk(KERN_INFO"pgd_tmp = 0x%p\n",pgd_tmp);    		//pgd_val获得pgd_tmp所指的页全局目录项   		//pgd_val是将pgd_tmp中的值打印出来           printk(KERN_INFO"pgd_val(*pgd_tmp) = 0x%lx\n",pgd_val(*pgd_tmp));            if(pgd_none(*pgd_tmp)){  //判断pgd有没有映射                   printk(KERN_INFO"Not mapped in pgd.\n");                            return 0;            }            pud_tmp = pud_offset(pgd_tmp,va); //返回va对应的页上级目录项的线性地址           printk(KERN_INFO"pud_tmp = 0x%p\n",pud_tmp);            printk(KERN_INFO"pud_val(*pud_tmp) = 0x%lx\n",pud_val(*pud_tmp));            if(pud_none(*pud_tmp)){                    printk(KERN_INFO"Not mapped in pud.\n");                    return 0;            }            pmd_tmp = pmd_offset(pud_tmp,va); //返回va在页中间目录中对应表项的线性地址           printk(KERN_INFO"pmd_tmp = 0x%p\n",pmd_tmp);            printk(KERN_INFO"pmd_val(*pmd_tmp) = 0x%lx\n",pmd_val(*pmd_tmp));            if(pmd_none(*pmd_tmp)){                    printk(KERN_INFO"Not mapped in pmd.\n");                    return 0;            }            //在这里，把原来的pte_offset_map()改成了pte_offset_kernel           pte_tmp = pte_offset_kernel(pmd_tmp,va);  //pte指的是  找到表              printk(KERN_INFO"pte_tmp = 0x%p\n",pte_tmp);            printk(KERN_INFO"pte_val(*pte_tmp) = 0x%lx\n",pte_val(*pte_tmp));            if(pte_none(*pte_tmp)){ //判断有没有映射                   printk(KERN_INFO"Not mapped in pte.\n");                    return 0;            }            if(!pte_present(*pte_tmp)){                    printk(KERN_INFO"pte not in RAM.\n");                    return 0;            }            pa = (pte_val(*pte_tmp) & PAGE_MASK) ;//物理地址的计算方法           printk(KERN_INFO"virt_addr 0x%lx in RAM Page is 0x%lx .\n",va,pa);            //printk(KERN_INFO"contect in 0x%lx is 0x%lx\n",pa,*(unsigned long *)((char *)pa + PAGE_OFFSET));                                                                      return 0;       }       static void __exit  find_pgd_exit(void)    {            printk(KERN_INFO"Goodbye!\n");       }       module_init(find_pgd_init);    module_exit(find_pgd_exit);      MODULE_LICENSE("GPL");         
```

**Makefile**

```cpp
# If KERNELRELEASE is defined, we've been invoked from the   # # kernel build system and can use its language.   ifneq ($(KERNELRELEASE),)    obj-m := lab3.o   #         # Otherwise we were called directly from the command   # line; invoke the kernel build system.   else            KERNELDIR ?= /lib/modules/$(shell uname -r)/build           PWD := $(shell pwd)          default:    $(MAKE) -C $(KERNELDIR) M=$(PWD) modules   endif         clean:    rm -rf *.o *~ core .depend .*.cmd *.ko *.mod.c .tmp_versions *.order *.symvers *.unsigned         
```

- 执行命令：insmod lab3.ko pid=2630 va=0xa87010
- 通过dmesg查看打印的信息：

![[Pasted image 20241006090723.png]]

可以看到相关的宏，以及线性地址对应的物理地址

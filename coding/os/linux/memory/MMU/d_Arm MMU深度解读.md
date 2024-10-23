
# 思考

1、为什么要用虚拟地址？为什么要用MMU？
2、MMU硬件完成了地址翻译，我们软件还需要做什么？
3、MMU在哪里？MMU和SMMU是什么关系？
本文转自 周贺贺，baron，代码改变世界ctw，Arm精选， armv8/armv9，trustzone/tee，secureboot，资深安全架构专家，11年手机安全/SOC底层安全开发经验。擅长trustzone/tee安全产品的设计和开发。文章有感而发。

# 一、MMU概念介绍

MMU分为两个部分: TLB maintenance 和 address translation

MMU的作用，主要是完成地址的翻译，即虚拟地址到物理地址的转换，无论是main-memory地址(DDR地址)，还是IO地址(设备device地址)，在开启了MMU的系统中，CPU发起的指令读取、数据读写都是虚拟地址，在ARM Core内部，会先经过MMU将该虚拟地址自动转换成物理地址，然后在将物理地址发送到AXI总线上，完成真正的物理内存、物理设备的读写访问.

那么为什么要用MMU？为什么要用虚拟地址？ 以下总结了三点：

多个程序独立执行 — 不需要知道具体物理地址

虚拟地址是连续的 — 程序可以在多个分段的物理内存运行

允许操作系统管理内存 — 哪些是可见的，哪些是允许读写的，哪些是cacheable的……

既然MMU开启后，硬件会自动的将虚拟地址转换成物理地址，那么还需要我们软件做什么事情呢？ 即创建一个页表翻译都需要做哪些事情呢？ 或者说启用一个MMU需要软件做什么事情呢？

设置页表基地址TTBR(Specify the location of the translation table)
初始化MAIR_EL3 (Memory Attribute Indirection Register)
配置TCR_EL3 (Configure the translation regime)
创建页表 (Generate the translation tables)
Enable the MMU

# 二、虚拟地址空间和物理地址空间

## 2.1 (虚拟/物理)地址空间的范围
内核虚拟地址空间的范围是什么？应用程序的虚拟地址空间的范围是什么？
以前我们在学习操作系统时，最常看到的一句话是：内核的虚拟地址空间范围是3G-4G地址空间，应用程序的虚拟地址空间的范围是0-3G地址空间； 到了aarch64上，则为 ： 内核的虚拟地址空间是0xffff_0000_0000_0000 - 0xffff_ffff_ffff_ffff , 应用程序的虚拟地址空间是: 0x0000_0000_0000_0000 - 0x0000_ffff_ffff_ffff.
做为一名杠精，必需告诉你这句话是错误的。错误主要有两点：

(1) arm处理器，并没有规定你的内核必需要使用哪套地址空间，以上这是Linux Kernel自己的设计，它设计了让Linux Kernel使用0xffff_0000_0000_0000 - 0xffff_ffff_ffff_ffff地址区间，Userspace使用0x0000_0000_0000_0000 - 0x0000_ffff_ffff_ffff地址区间，这里正好可以举一个反例，比如optee os，它的kernel mode和user mode使用的都是高位的虚拟地址空间。
(2) 高位是有几个F（几个1）是根据你操作系统使用的有效虚拟地址位来决定的，也并非固定的。比如optee中的mode和user mode的虚拟地址空间范围都是： 0x0000_0000_0000_0000 - 0x0000_0000_ffff_ffff
其实arm文档中有一句标准的描述 :

高位是1的虚拟地址空间，使用TTBR1_ELx基地址寄存器进行页表翻译；高位是0的虚拟地址空间，使用TTBR0_ELx基地址寄存器页表翻译。 所以不应该说，因为你使用了哪个寄存器(TTBR0/TTBR1)，然后决定了你使用的哪套虚拟地址空间；应该说，你操作系统(或userspace软件)使用了哪套虚拟地址空间，决定了使用哪个哪个基地址寄存器(TTBR0/TTBR1)进行翻译。

如下便是两套虚拟地址空间和TTBRn_ELx的对应关系，其中高位的位数不是固定的16(即T1SZ和T0SZ不一定等于16)

![[Pasted image 20240909130008.png]]

以下摘自ARM文档的官方描述：

As Figure shows, for 48-bit VAs:
• The address range translated using TTBR0_ELx is 0x0000000000000000 to 0x0000FFFFFFFFFFFF.
• The address range translated using TTBR1_ELx is 0xFFFF000000000000 to 0xFFFFFFFFFFFFFFFF.
In an implementation that includes ARMv8.2-LVA and is using Secure EL3 the 64KB translation granule, for 52-bit VAs:
• The address range translated using TTBR0_ELx is 0x0000000000000000 to 0x000FFFFFFFFFFFFF.
• The address range translated using TTBR1_ELx is 0xFFF0000000000000 to 0xFFFFFFFFFFFFFFFF.
Which TTBR_ELx is used depends only on the VA presented for translation. The most significant bits of the VA must all be the same value and:
• If the most significant bits of the VA are zero, then TTBR0_ELx is used.
• If the most significant bits of the VA are one, then TTBR1_ELx is used.

## 2.2 物理地址空间有效位(范围)

具体每一个core的物理地址是多少位，其实都是定死的，虚拟地址是多少位，是编译或开发的时候根据自己的需要自己配置的。如下表格摘出了部分arm core的物理地址有效位，所以你具体使用多少有效位的物理地址，可以查询core TRM手册。

![[Pasted image 20240909130025.png]]

页表翻译相关寄存器的配置

ID_AA64MMFR0_EL1.PARange : Physical address size : 读取arm寄存器，得到当前系统支持的有效物理地址是多少位
![[Pasted image 20240909130036.png]]

TCR_EL1.IPS : Output address size : 告诉mmu，你需要给我输出多少位的物理地址
![[Pasted image 20240909130041.png]]

TCR_EL1.T0SZ和TCR_EL1.T1SZ : Input address size : 告诉mmu，我输入的是多少有效位的虚拟地址
![[Pasted image 20240909130047.png]]

# 三、Translation regimes

内存管理单元 (MMU) 执行地址翻译。MMU 包含以下内容：

The table walk unit : 它从内存中读取页表，并完成地址转换
Translation Lookaside Buffers (TLBs) ： 缓存，相当于cache

软件看到的所有内存地址都是虚拟的。 这些内存地址被传递到 MMU，它检查最近使用的缓存转换的 TLB。 如果 TLB没有找到最近缓存的翻译，那么翻译单元将从内存中读取适当的一个或多个表项目进行地址翻译，如下所示：

![[Pasted image 20240909130056.png]]

Translation tables 的工作原理是将虚拟地址空间划分为大小相等的块，并在表中为每个块提供一个entry。
Translation tables 中的entry 0 提供block 0 的映射，entry 1 提供block 1 的映射，依此类推。 每个entry都包含相应物理内存块的地址以及访问物理地址时要使用的属性。

![[Pasted image 20240909130102.png]]

在当前的ARMV8/ARMV9体系中(暂不考虑armv9的RME扩展), 至少存在以下9类Translation regime：

Secure EL1&0 translation regime, when EL2 is disabled
Non-secure EL1&0 translation regime, when EL2 is disabled
Secure EL1&0 translation regime, when EL2 is enabled
Non-secure EL1&0 translation regime, when EL2 is enabled
Secure EL2&0 translation regime
Non-secure EL2&0 translation regime
Secure EL2 translation regime
Non-secure EL2 translation regime
Secure EL3 translation regime
这9类Translation regime的地址翻译的场景如下图所示：

![[Pasted image 20240909130109.png]]

Secure and Non-secure地址空间
在REE(linux)和TEE(optee)双系统的环境下，可同时开启两个系统的MMU.
在secure和non-secure中使用不同的页表.secure的页表可以映射non-secure的内存，而non-secure的页表不能去映射secure的内存，否则在转换时会发生错误

![[Pasted image 20240909130115.png]]

Two Stage Translations
EL1&0 Translation regime处于VM(Virtual Machine)或SP(Secure Partition)时，EL2 enabled的情况下，是需要stage2转换的。对于EL2 Translation regime 和 EL3 Translation regime是没用stage2 转换的。

![[Pasted image 20240909130123.png]]

# 四、地址翻译/几级页表？

4.1、思考：页表到底有几级？
从以下图来看，有的页表从L2开始，有得从L1开始，有的从L0开始，还有从L-1开始的，都是到L3终止。
那么我们的页表到底有几级呢？

![[Pasted image 20240909130130.png]]

4.2、以4KB granule为例，页表的组成方式

![[Pasted image 20240909130140.png]]

除了第一级index(这里是leve 0 table中的index)，每一个查找table/page的index都是9个bit，也就是说除了第一级页表，后面的每一级table都是有512个offset

如果VA_BIT = 39，那么leve 0 table用BIT\[38:39\]表示，只有1个offset

如果VA_BIT = 48，那么leve 0 table用BIT\[47:39\]表示，有512个offset

如果VA_BIT > 48，那是不存在的，因为arm规定，大于48的，只有一个，那就是VA_BIT=52，并且规定该情况下的最小granue size=64KB，而我们这里讲述的是granue size=4KB的情况

如果VA_BIT = 32，那么leve 0 table就不用了，TTBR_ELx指向Level 1 table

另外我们还需注意一点，在Level 0 table中，他只能指向D_Table，不能指向D_Block

以下针对虚拟地址是48有效位的情形做了一个总结：

![[Pasted image 20240909130148.png]]

4.3、optee实际使用的示例
32位有效虚拟地址、，3级页表查询(L1、L2、L3)，颗粒的位4KB

![[Pasted image 20240909130158.png]]

如下展示是optee os的页表结构，TTBR0_EL1指向L1 Table，L1 Table中有4个表项，但只用了3个 , 也就对应着3张L2 Table.

![[Pasted image 20240909130225.png]]

配置相关的代码如下：

![[Pasted image 20240909130232.png]]

# 五、页表格式（Descriptor format）

## 5.1、ARMV8支持的3种页表格式

AArch64 Long Descriptor : 我们只学习这个
Armv7-A Long Descriptor ： for Large Physical Address Extension (LPAE)
Armv7-A Short Descriptor

## 5.2、AArch64 Long Descriptor支持的四种entry

对于AArch64 Long Descriptor，又分为下面四种entry：

An invalid or fault entry.
A table entry, that points to the next-level translation table.
A block entry, that defines the memory properties for the access.
A reserved format
注意：entry\[1:0\] 表示该entry属于哪类entry， Block Descriptor和Page Descriptor是一个意思。在当前架构中，reserved也是invalid。

![[Pasted image 20240909130238.png]]

## 5.3、页表的属性位介绍（ Block Descriptor/Page Descriptor ）

### 5.3.1、stage1的页表属性

![[Pasted image 20240909130245.png]]

（Attribute fields in stage 1 VMSAv8-64 Block and Page descriptors）

PBHA, bits\[62:59\] ：for FEAT_HPDS2
XN or UXN, bit\[54\] ： Execute-never or Unprivileged execute-never
PXN, bit\[53\] ：Privileged execute-never
Contiguous, bit\[52\] ： translation table entry 是连续的，可以存在一个TLB Entry中
DBM, bit\[51\] ：Dirty Bit Modifier
GP, bit\[50\] ：for FEAT_BTI
nT, bit\[16\] ：for FEAT_BBM
nG, bit\[11\] ：缓存在TLB中的翻译是否使用ASID标识
AF, bit\[10\] ： Access flag, AF=0后，第一次访问该页面时，会将该标志置为1. 即暗示第一次访问
SH, bits\[9:8\] ：shareable属性
AP\[2:1\], bits\[7:6\] ：Data Access Permissions bits,
NS, bit\[5\] ：Non-secure bit
AttrIndx\[2:0\], bits\[4:2\] ：

### 5.3.2、stage2的页表属性

（Attribute fields in stage 2 VMSAv8-64 Block and Page descriptors）

![[Pasted image 20240909130254.png]]

PBHA\[3:1\], bits\[62:60\] ：for FEAT_HPDS2

PBHA\[0\], bit\[59\] ：for FEAT_HPDS2

XN\[1:0\], bits\[54:53\] ：Execute-never

Contiguous, bit\[52\] ：translation table entry 是连续的，可以存在一个TLB Entry中

DBM, bit\[51\] ：Dirty Bit Modifier

nT, bit\[16\] ：for FEAT_BBM

FnXS, bit\[11\] ：for FEAT_XS

AF, bit\[10\] ：Access flag

SH, bits\[9:8\] ：shareable属性

S2AP, bits\[7:6\] ：Stage 2 data Access Permissions

MemAttr, bits\[5:2\] ：

### 5.3.3、其它标志位的详细介绍

（1）、MemAttr
指向MAIR_ELx寄存器中的attrn属性域，表示内存的缓存属性，如cachable、shareable等

（2）、NS
Non-secure比特 表示转换后的物理地址是secure的还是non-secure的。

（3）、AP
Data access permissions 数据访问权限

![[Pasted image 20240909130300.png]]

（4）、SH
shareable属性

![[Pasted image 20240909130305.png]]

（5）、AF
Access flag, AF=0后，第一次访问该页面时，会将该标志置为1. 即暗示第一次访问
（6）、nG
对于 EL0/EL1 虚拟地址空间，Page Descriptor属性字段中的 nG 位将转换标记为Gloabl(G) 或non-Gloabl(nG)。例如，内核映射是Gloabl(G)翻译，应用程序映射是non-Gloabl翻译。Gloabl翻译适用于当前正在运的任何应用程序。非全局翻译仅适用于特定应用程序

non-Gloabl映射在 TLB 中使用 ASID进行标记。在 TLB 查找时，将 TLB 条目中的 ASID 与当前选择的 ASID 进行比较。如果它们不匹配，则不使用TLB 条目。下图显示了内核空间中没有 ASID 标记的全局映射和用户空间中具有 ASID 标记的非全局映射

![[Pasted image 20240909130314.png]]

（7）、XN or UXN
特权和非特权不可从该memory-region中执行指令的标志位：
Execute-never
Unprivileged execute-never

# 六、地址翻译指令介绍

address translation的指令大约14个：

![[Pasted image 20240909130320.png]]

总结一下：

![[Pasted image 20240909130328.png]]

# 七、地址翻译相关的系统寄存器总结

地址转换由系统寄存器的组合控制：

## 7.1 SCTLR_ELx

![[Pasted image 20240909130339.png]]

系统控制寄存器，控制着MMU、I-cache、D-cache的打开与关闭，也控制着translation table walks访问内存的大小端。

M - Enable Memory Management Unit (MMU).

C - Enable for data and unified caches.

EE - Endianness of translation table walks.

## 7.2 TTBRn_ELx

![[Pasted image 20240909130344.png]]

BADDR : 基地址
ASID ：TLB entry区分user程序所用的ASID

## 7.3 TCR_ELx

在ARM Core中(aarch64)，有三个Translation Control Register 寄存器:

![[Pasted image 20240909130349.png]]

![[Pasted image 20240909130354.png]]

比特位 功能 说明
ORGN1、IRGN1、ORGN0、IRGN0 cacheable属性 outer/inner cableability的属性(如直写模式、回写模式)
SH1、SH0 shareable属性 cache的共享属性配置(如non-shareable, outer/inner shareable)
TG0/TG1 Granule size Granule size(其实就是页面的大小,4k/16k/64k)
IPS 物理地址size 物理地址size,如32bit/36bit/40bit
EPD1、EPD0 - TTBR_EL1/TTBR_EL0的enable和disable
TBI1、TBI0 - top addr是ignore，还是用于MTE的计算
A1 - ASID的选择，是使用TTBR_EL1中的，还是使用TTBR_EL0中的
AS - ASID是使用8bit，还是使用16bit

## 7.3 MAIR_ELx

内存属性寄存器，分为8个Attrn，所以一个core，最多只支持8中内存属性。
页表中的每一个entry，都会指向一个Attr域。

![[Pasted image 20240909130400.png]]

推荐
ARMv8/ARMv9架构从入门到精通 --博客专栏
《Armv8/Armv9架构从入门到精通 第二期》 --大课程
8天入门ARM架构 --入门课程
————————————————

版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。

原文链接：https://blog.csdn.net/a1657054242/article/details/136613964

# 

原创 baron 人人极客社区

 _2022年02月23日 08:28_

## MMU概念介绍

MMU分为两个部分: TLB maintenance 和 address translation

![图片](https://mmbiz.qpic.cn/mmbiz_png/9sNwsXcN68pmJMZMVvVZQibRyr8sXQDTiaOiaT8cmsAbZW18nCKFPvibj6ENaibhWlHr7kOicZY7RT1wic0LKGDWiabzoQ/640?wx_fmt=png&wxfrom=13&tp=wxpic)

MMU的作用，主要是完成地址的翻译，无论是main-memory地址(DDR地址)，还是IO地址(设备device地址)，在开启了MMU的系统中，CPU发起的指令读取、数据读写都是虚拟地址，在ARM Core内部，会先经过MMU将该虚拟地址自动转换成物理地址，然后在将物理地址发送到AXI总线上，完成真正的物理内存、物理设备的读写访问。

下图是一个linux kernel系统中宏观的虚拟地址到物理地址转换的视图，可以看出在MMU进行地址转换时，会依赖TTBRx_EL1寄存器指向的一个页表基地址。其中，TTBR1_EL1指向特权模式的页表基地址，用于特权模式下的地址空间转换；TTBR0_EL0指向非特权模式的页表基地址，用于非特权模式下的地址空间转换。刚刚我们也提到，CPU发出读写后, MMU会自动的将虚拟地址转换为为例地址，那么我们软件需要做什么呢？我们软件需要做的其实就是管理这个页表，按照ARM的技术要求去创建一个这样的页表，然后再将其基地址写入到TTBR1_EL1或TTBR0_EL1。当然，根据实际的场景和需要，这个基地址和页表中的内容都会发生动态变化。例如，两个user进程进行切换时，TTBR0_EL1是要从一个user的页表地址，切换到另外一个user的页表地址。

![图片](https://mmbiz.qpic.cn/mmbiz_png/9sNwsXcN68pmJMZMVvVZQibRyr8sXQDTialkKoodOOHu0cDcjC3UxsRP6ibVpXsj77FgSs3U1rvWV0PhLPL2wtxMg/640?wx_fmt=png&wxfrom=13&tp=wxpic)

## MMU地址翻译的过程

using a 64KB granule with a 42-bit virtual address space,地址翻译的过程(只用到一级页表的情况)：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

使用二级页表的情况举例：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 在secure和non-secure中使用MMU

TTBRx_EL1是banked的，在linux和optee双系统的环境下，可同时开启两个系统的MMU。在secure和non-secure中使用不同的页表.secure的页表可以映射non-secure的内存，而non-secure的页表不能去映射secure的内存，否则在转换时会发生错误。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 在不同异常等级中使用MMU

在ARMV8-aarch64架构下，页表基地址寄存器有:

- TTBR0_EL1 – banked
    
- TTBR1_EL1 – banked
    
- TTBR1_EL2
    
- TTBR1_EL3
    

在EL0/EL1的系统中，MMU地址转换时，如果虚拟地址在0x00000000_ffffffff - 0x0000ffff_ffffffff范围，MMU会自动使用TTBR0_EL1指向的页表，进行地址转换；如果虚拟地址在0xffff0000_ffffffff - 0xffffffff_ffffffff范围，MMU会自动使用TTBR1_EL1指向的页表，进行地址转换。

在EL2系统中，MMU地址转换时，会自动使用TTBR2_EL1指向的页表。

在EL3系统中，MMU地址转换时，会自动使用TTBR3_EL1指向的页表

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## memory attributes介绍

translation tables为每一块region(entry)都定义了一个memory attributes条目,如同下面这个样子（以linux kernel为例，在创建页表的时候，应该会设置这个memory attributes，有待看代码去验证）:

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

• Unprivileged eXecute Never (UXN) and Privileged eXecute Never (PXN) are execution permissions. • AF is the access flag. • SH is the shareable attribute. • AP is the access permission. • NS is the security bit, but only at EL3 and Secure EL1. ---- secure权限配置 • Indx is the index into the MAIR_ELn

在这块region(entry)的memory attributes条目中的BIT4:2(index)指向了系统寄存器MAIR_ELn中的attr，MAIR_ELn共有8中attr选择。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

而每一个attr都有一种配置:

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## memory tagging介绍

> When tagged addressing support is enabled, the top eight bits [63:56] of the virtual address are ignored by the processor. It internally sets bit [55] to sign-extend the address to 64-bit format. The top 8 bits can then be used to pass data. These bits are ignored for addressing and translation faults. The TCR_EL1 has separate enable bits for EL0 and EL1

如果使用memory tagging, 虚拟地址的[63:56]用于传输签名数据，bit[55]表示是否需要签名.TCR_EL1也会有一个bit区分是给EL0用的还是给EL1用的

## 启用hypervisor

如果启用了hypervisor那么虚拟地址转换的过程将有VA—>PA变成了VA—>IPA—>PA, 也就是要经过两次转换.在guestos(如linux kernel)中转换的物理地址，其实不是真实的物理地址(假物理地址)，然后在EL2通过VTTBR0_EL2基地址的页表转换后的物理地址，才是真实的硬件地址。

> IPA : intermediate physical address

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## Access permissions

> Access permissions are controlled through translation table entries. Access permissions control whether a region is readable or writeable, or both, and can be set separately to EL0 for unprivileged and access to EL1, EL2, and EL3 for privileged accesses, as shown in the following table

参考 ：SCTLR_EL1.WXN、SCTLR.UWXN![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)执行权限:

- non-executable (Execute Never (XN))
    
- The Unprivileged Execute Never (UXN)
    
- Privileged Execute Never (PXN)
    

## MMU/cache相关的寄存器总结

MMU(address translation /TLB maintenance)、cache maintenance相关的寄存器

1. address translation
    

address translation 共计14个寄存器

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

2. TLB maintenance
    

TLB maintenance数十个寄存器

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

3. cache maintenance
    

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

4. Base system registers
    

系统寄存器中, 和MMU/Cache相关的寄存器有:

**TTBR0_ELx TTBR1_ELx**

(aarch64)

- TTBR0_EL1
    
- TTBR0_EL2
    
- TTBR0_EL3
    
- TTBR1_EL1
    
- VTTBR_EL2
    

(aarch32)

- TTBR0
    
- TTBR1
    
- HTTBR
    
- VTTBR
    

**TCR_ELx**

(aarch64)

- TCR_EL1
    
- TCR_EL2
    
- TCR_EL3
    
- VTCR_EL2
    

(aarch32)

- TTBCR(NS)
    
- HTCR
    
- TTBCR(S)
    
- VTCR
    

**MAIR_ELx**

- MAIR_EL1
    
- MAIR_EL2
    
- MAIR_EL3
    

## 系统寄存器 — TCR寄存器介绍

在ARM Core中(aarch64)，还有几个相关的系统寄存器:

- TCR_EL1 banked
    
- TCR_EL2
    
- TCR_EL3
    

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

1. T1SZ、T0SZ
    

- T1SZ, bits [21:16] 通过TTBR1寻址的内存区域的大小偏移量，也就是TTBR1基地址下的一级页表的大小
    
- T0SZ, bits [5:0]
    

2. ORGN1、IRGN1、ORGN0、IRGN0
    

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

其实可以总结为这样：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

3. SH1、SH0
    

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

其实可以总结为这样：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Shareable的很容易理解，就是某个地址的可能被别人使用。我们在定义某个页属性的时候会给出。Non-Shareable就是只有自己使用。当然，定义成Non-Shareable不表示别人不可以用。某个地址A如果在核1上映射成Shareable，核2映射成Non-Shareable，并且两个核通过CCI400相连。那么核1在访问A的时候，总线会去监听核2，而核2访问A的时候，总线直接访问内存，不监听核1。显然这种做法是错误的。

对于Inner和Outer Shareable，有个简单的的理解，就是认为他们都是一个东西。在最近的ARM A系列处理器上上，配置处理器RTL的时候，会选择是不是把inner的传输送到ACE口上。当存在多个处理器簇或者需要双向一致性的GPU时，就需要设成送到ACE端口。这样，内部的操作，无论inner shareable还是outershareable，都会经由CCI广播到别的ACE口上。

4. TG0/TG1 - Granule size
    

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

5. IPS
    

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

6. EPD1、EPD0
    

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

7. TBI1、TBI0
    

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

8. A1
    

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

9. AS
    

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

除了以上介绍的bit之外，剩余的bit都是特有功能使用或reserved的

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 代码使用示例展

设置inner/outer cache的属性(只写模式/回写模式/write allocate/No-write allocate)

`#define TCR_IRGN_WBWA  ((UL(1) << 8) | (UL(1) << 24))   //使用TTBR0和使用TTBR1时后的inner cache的属性设置      #define TCR_ORGN_WBWA  ((UL(1) << 10) | (UL(1) << 26))   //使用TTBR0和使用TTBR1时后的outer cache的属性设置      #define TCR_CACHE_FLAGS TCR_IRGN_WBWA | TCR_ORGN_WBWA   // inner + outer cache的属性值         ENTRY(__cpu_setup)   ......    /*     * Set/prepare TCR and TTBR. We use 512GB (39-bit) address range for     * both user and kernel.     */    ldr x10, =TCR_TxSZ(VA_BITS) | TCR_CACHE_FLAGS | TCR_SMP_FLAGS | \      TCR_TG_FLAGS | TCR_ASID16 | TCR_TBI0 | TCR_A1    tcr_set_idmap_t0sz x10, x9      ......    msr tcr_el1, x10    ret     // return to head.S   ENDPROC(__cpu_setup)   `

属性设置了1，也就是回写模式、write allocate模式。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "虚线阴影分割线")

![](https://res.wx.qq.com/t/fed_upload/b39ef69e-c4d6-4169-8612-5f00a84860e7/wx-avatar-default.svg)

公众号

5T技术资源大放送！包括但不限于：C/C++，Arm, Linux，Android，人工智能，单片机，树莓派，等等。在上面的【人人都是极客】公众号内回复「peter」，即可免费获取！！  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E) **记得点击****分享****、****赞****和****在看****，给我充点儿电吧**

ARM6

ARM · 目录

上一篇ARM SMMU学习笔记下一篇ARM64 的多核启动流程分析

阅读 3726

​

写留言

**留言 6**

- 程序员Peter
    
    2022年2月23日
    
    赞4
    
    大家可以加我微信：rrjike，一起学习，共同进步，还可以进群认识更多朋友![[玫瑰]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[玫瑰]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[玫瑰]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 王小二°C
    
    2022年2月23日
    
    赞2
    
    说实话，看的一脸懵逼
    
    人人极客社区
    
    作者2022年2月23日
    
    赞
    
    安卓才是大佬的强项
    
- 锐rui
    
    上海2022年5月5日
    
    赞
    
    有没有电子档PDF，或者其他途径查看这些寄存器。
    
    人人极客社区
    
    作者2022年5月5日
    
    赞
    
    有网站 后面发下
    
- Tariq-Cao
    
    2022年2月23日
    
    赞
    
    不知道是不是笔误，基地址ttbr0_el0还是ttbr0_el1
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/9sNwsXcN68pL55XIyzTrCHZTbIUdTibQcuzuCaYeGTXNMyn6ACmicUrpoDC0xZSap46XJ59sKysPg9Rg379f32cA/300?wx_fmt=png&wxfrom=18)

人人极客社区

18914

6

写留言

**留言 6**

- 程序员Peter
    
    2022年2月23日
    
    赞4
    
    大家可以加我微信：rrjike，一起学习，共同进步，还可以进群认识更多朋友![[玫瑰]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[玫瑰]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[玫瑰]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 王小二°C
    
    2022年2月23日
    
    赞2
    
    说实话，看的一脸懵逼
    
    人人极客社区
    
    作者2022年2月23日
    
    赞
    
    安卓才是大佬的强项
    
- 锐rui
    
    上海2022年5月5日
    
    赞
    
    有没有电子档PDF，或者其他途径查看这些寄存器。
    
    人人极客社区
    
    作者2022年5月5日
    
    赞
    
    有网站 后面发下
    
- Tariq-Cao
    
    2022年2月23日
    
    赞
    
    不知道是不是笔误，基地址ttbr0_el0还是ttbr0_el1
    

已无更多数据
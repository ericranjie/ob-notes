# 

张健 Linux内核远航者

 _2021年09月27日 14:24_

  

# 

作者按：在上个月的 [os2atc会议](https://mp.weixin.qq.com/s?__biz=MzU1ODkxNDA4Mw==&mid=2247484900&idx=1&sn=7b1a8c273f778a5786d9afede3160cb7&scene=21#wechat_redirect) 上，笔者作为Linux阅码场高级顾问分享了RISC-V对Linux对支持情况。会议后对分享内容再次做了迭代，期待和大家一起交流，进步。

  

从2010年开始的RISC-V 项目，已经有10年的时间，RISC-V基金会先后批准了RISC-V Base ISA， Privileged Architecture，Processor Trace等规范。RISC-V对Linux的基本支持也已经完成。本文尝试通俗易懂的介绍RISC-V对于Linux的基本支持，包括指令集和异常处理。内存管理，迁移到RISC-V，UEFI，KVM等支持，欢迎继续关注本公众号。

  

# 

**ISA**

眼见为实，下面就是RISC-V的汇编语言了。从笔者代码中反汇编得来，功能是把传入的字符c，通过RISC-V提供的标准接口（此处指OpenSBI，见 下文 ）输出到终端。

![Image](https://mmbiz.qpic.cn/mmbiz_png/Ass1lsY6bytxQ5R4bLCiaUsicHQt9nZnkGAs0Yiamx85z2BCBuR2DaMTHPJGGMN51cWkHia7icgavjGSZWff8lPXTPQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

  

# 

**名正才能言顺，RISC-V指令集规范**

想做好一个生态，需要大家对齐目标，RISC-V的规范（ Specifications，参考链接1）就起了这样的作用，目前的规范分成两部分，第1卷是非特权指令，第2卷是特权指令。在第一卷中，RISC-V已经定义了RV32I和RV64I两个基础整数运算，并有如下扩展。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

现在问题来了，这么多规范，大家如果用的指令集不一致，岂不是没法互操作了？别急，RISC-V还定义了下面指令集组合。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

为了提高指令密度，更节省存储空间，RISC-V还有上述的C扩展（压缩指令），例如RV32GC表示使用压缩指令的RV32G指令集，RV64GC表示使用压缩指令的RV64G指令集。根据Andrew Waterman的测试，在Spec2006（一个测试cpu性能的商用测试套）中，RV32GC和RV64GC分别比RV32G和RV64G节省30%+的空间，而性能变化不大，见 参考资料2 。

  

除了非特权指令，RISC-V的规范还包括特权指令。Privileged Spec里面Machine ISA和Supervisor ISA已经release了1.11版本。而虚拟化Virtualization ISA目前是0.6，还在讨论中。

  

# 

**ISA简述**

了解指令集有助于我们了解这个架构。RISC-V是一个RISC架构。所有的运算都在寄存器之间进行，通过单独的load和store指令，把数据从内存中读出或写回。整体的指令集架构方面，包云岗老师带领团队已经做了很好的中文翻译（参考链接3） ，我这边就不再详细的展开讲，仅仅举两个例子

“Addi    sp,sp,-32”是把sp寄存器的值减32并保存到sp寄存器中，这条指令在准备本函数自己的栈空间。

“Sd      ra,24(sp)”是把本返回地址（ra）保存到栈上，24(sp)表示相对+24的位置，这是RISC-V二进制调用规范定义的。

  

### 

**伪汇编**

平时读代码的时候，除了架构中定义的汇编指令还会遇到伪汇编。伪汇编是一些帮助我们平时手写汇编提高效率的东西。比如说寄存器的赋值，下面的一条li伪指令会被翻译为lui和addiw两条指令。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

再举个例子，csrw用于写入csr寄存器。其中csr的全称是Control and Status Register，主要是和特权管理相关的寄存器。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

# 

**异常处理**

了解了基本的汇编语言，我们就可以进一步的了解RISC-V的异常，这是操作系统的职责之一（另一个重要职责是虚拟内存的管理，在下一篇文章介绍）。

# 

  

为了便于理解，我们与ARM和X86对比下。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

大约40年前，x86架构有了如上图的保护模式。其中Level0跑操作系统，Level3跑应用。为了支持虚拟化，x86引入了VMX operation（如下图），Guest操作系统和应用运行在non-root模式，Hypervisor运行在root模式。在这样的设计下，支持Type-1和Type-2的虚拟机技术都比较方便，并且原有的操作系统不需要任何修改就可以作为Guest操作系统运行。不过早期的x86虚拟化也有缺点，例如不支持二级页表转换，需要用shadow page table，这样效率很低，直到EPT的引入解决这一问题。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

相比之下，ARM架构采取了不同的方式。由于ARM架构下已经有了如下图的Normal和Secure world设计（这里指的是Normal world的操作系统，例如Linux，可以不加修改的运行在Secure world）。没有用类似x86添加VMX root和non-root的operation的形式。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

而是如下图添加了新的一个异常级别EL2（下图的Hypervisor），很容易理解的是EL2比EL1有更多的级别。问题在于EL2并不是EL1的复制，也就是说Linux kernel没法直接运行在EL2上。对于Xen这种典型的Type-1虚拟化机制没问题，Xen hypervisor可以很开心的运行在EL2。但是对于KVM，KVM作为Linux kernel的一个模块，就比较尴尬：KVM需要EL2的一些权限，但是Linux又只能运行在EL1。于是原本在x86上完整的KVM被拆成了high-visor和low-visor（需要EL2特权能力的部分）两部分。平时KVM的high-visor愉快和Linux kernel一起运行在EL1，当需要虚拟化管理的特权操作时，KVM从high-visor陷入到low-visor处理。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

ARM的虚拟化技术比x86的晚了很多年，有个好处是可以完成x86多次迭代得到的状态，例如前文提到的x86为了避免shadow page table引入的EPT，在ARM虚拟化扩展时是原生支持的。同时，ARM的虚拟化扩展在32位和64位架构下是完全一样的，早期的虚拟化工作，不论是xen还是KVM的工作都是在32位的ARMv7a架构的Cortex-A15和Cortex-A7上完成的。这样ARM64推出后，虚拟化这部分工作不需要重新做。至于ARM虚拟化上更多异常处理导致的性能问题，从ARMv8.1开始，有了VHE模式，支持把EL1下沉到EL2运行，这样KVM ARM就没有了前述的开销。

  

从上述历史可以看出，软硬件的协同，灵活可扩展的设计非常重要。RISC-V的设计中也体现了这一点。在没有虚拟化特性情况下，RISC-V最多支持三个特权级别。通常来说，为了支持Linux这样的Rich OS，需要同时支持这三个模式。每一层有不同的权限。Bootloader/BIOS/UEFI运行系统的最高级别machine mode，Linux kernel运行在supervisor mode，应用运行在user mode。默认情况下，所有的异常都在machine mode处理。在有Linux kernel时，这样明显降低了效率：所有原本可以由Linux kernel处理的异常，例如应用的缺页异常，都需要先陷入到machine mode再转发给kernel。为了允许软件系统更灵活的管理异常，RISC-V引入了delegation机制，可以选择把一部分异常和中断由硬件直接交给supervisor mode的kernel处理。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

现在问题来了，RISC-V的虚拟化是如何设计的呢？很明显，虚拟化的特权级别需要支持Linux kernel这种Rich OS。所以RISC-V没有像早期的ARM虚拟化一样把虚拟化异常直接直接加到supervisor mode和machine mode之间，而是定义了独立的virtualization mode，这个mode再与user和supervisor mode组合，于是有了下面的表格。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

_（表格来自The RISC-V Instruction Set Manual, Volume II: Privileged Architecture, Document Version 1.12-draft Table 5.1)_

这么说有点抽象，用RISC-V kVM作者之一的Anup Patel画的图表示（图片已获得作者授权， 原图见参考链接4）。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

备注：RISC-V虚拟化规范目前处于0.6草稿状态，未来可能还会有些小的变化。

  

# 

**SBI**

了解了RISC-V的特权模式，不同层次的软件调用遵循什么样的规范呢？RISC-V的设计中，下层（硬件/软件）对上层透明，规范会定义二进制接口，对具体如何实现没有要求。例如Linux kernel在supervisor mode，对下面的特权级别，通过SBI（Supervisor Binary Interface）访问，SBI访问的软件称为SEE（Supervisor Execution Environment），SEE可以是bootloader，BIOS，也可以Hypervisor。和SEE类似的还有支持应用的运行环境AEE。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

_(图片来自The RISC-V Instruction Set Manual, Volume II: Privileged Architecture, Document Version 1.12-draft Figure 1.1)_

  

SBI的规范见参考链接5，规范定义了SBI的能力，例如获得SBI规范的版本，发送或接收一个字符，remote fence，设置timer，发送IPI中断，管理RISC-V处理器（RISC-V中称为hart）等，以及SBI的二进制调用规范。截止这篇文章，SBI是0.3 draft，这个版本主要是增加了用于系统复位的SBI接口。既然SBI是个规范，那就有各种实现，OpenSBI就是其中一个实现，这个实现支持generic（用于支持qemu的RISC-V virt machine），sifive和k210等芯片。

这么说有点抽象，咱们举个简单的例子。如果想写一个简单的从supervisor mode调用SBI接口打印字符的代码，要怎么做呢？

  

首先，假设，我们以及有了c语言的运行环境，那我们需要根据SBI定义的二进制调用规范，使用寄存器a7传递指定的extension ID。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

_(图片来自 RISC-V Supervisor Binary Interface Specification Version 0.3-rc0 p6)_

  

从下图可以看到，extension ID是1。同时我们看到函数原型是通过第一个参数传入字符ch。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

_(图片来自 RISC-V Supervisor Binary Interface Specification Version 0.3-rc0 p6)_

RISC-V使用哪个寄存器保存第一个参数呢？根据RISC-V ELF psABI

specification的整数寄存器调用约定（ 参考链接6 ），我们可以看到寄存器a0用于传递第一个参数。发送一个字符的对应的代码是这个样子

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

写了SBI调用接口，还没有万事大吉，如果希望bootloader直接加载我们的代码，我们还需要自己准备c语言运行环境。加上下面几行汇编即可。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

cpu_enter里面会打印字符串。我们选择使用OpenSBI的fw_jump从固定的0x80200000加载我们的二进制，启动效果如下。最后一行“Hello XU Xiake“是上面代码打印的。希望我们像徐霞客一样，通过编写代码，游览RISC-V的各种特性。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

[](https://mp.weixin.qq.com/s?__biz=MzI5MzcwODYxMQ==&mid=2247485102&idx=1&sn=655d7479dd919faf0b0295be939fc32b&scene=21#wechat_redirect)

# 

参考链接

[1] RISC-V 规范: https://riscv.org/technical/specifications/

[2] Design of the RISC-V Instruction Set Architecture https://www2.eecs.berkeley.edu/Pubs/TechRpts/2016/EECS-2016-1.pdf

[3] RISC-V 架构简述：http://riscvbook.com/chinese/

[4] RISC-V虚拟化扩展：https://static.sched.com/hosted_files/osseu19/4e/Xvisor_Embedded_Hypervisor_for_RISCV_v5.pdf

[5] SBI规范：https://github.com/riscv/riscv-sbi-doc

[6] RISC-V Integer Register Convention https://github.com/riscv/riscv-elf-psabi-doc/blob/master/riscv-elf.md#integer-register-convention-

[7] RISC-V软件状态 https://github.com/riscv/riscv-software-list

  

作者其他文章链接：

- [了解技术趋势助力面试和职业规划](https://mp.weixin.qq.com/s?__biz=MzI5MzcwODYxMQ==&mid=2247485224&idx=1&sn=89f768bae77de11c604e9a7d829a8b2d&scene=21#wechat_redirect)
    
- [Being02: 时间，精力与自身的平衡](https://mp.weixin.qq.com/s?__biz=MzI5MzcwODYxMQ==&mid=2247485132&idx=1&sn=5488d87e1b0f4141d7638fe2e3ee02fc&scene=21#wechat_redirect)
    
- [35岁的变与不变](https://mp.weixin.qq.com/s?__biz=MzI5MzcwODYxMQ==&mid=2247485102&idx=1&sn=655d7479dd919faf0b0295be939fc32b&scene=21#wechat_redirect)
    

（END）

  

更多精彩，尽在"Linux阅码场"，扫描下方二维码关注

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Reads 773

​
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

作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2015-11-24 18:22 分类：[ARMv8A Arch](http://www.wowotech.net/sort/armv8a_arch)

一、前言

本文主要描述了4.1.10内核初始化过程中如何初始化异常向量表。当然，首先需要准备一些异常的基础知识，这主要在第二章，如果你非常熟悉ARM64的异常，那么可以忽略这个章节。 第三章描述了ARM64上各种形形色色的异常，第四章描述了ARM64上硬件提供的协助，最后一章描述了代码过程。

为了简化，本文对所描述的异常进行了限制：

1、所有的exception level的运行状态都是AArch64，不考虑异常发生在AArch32 excution state的时候

2、不考虑支持security extension，也就是说EL3状态的异常处理也不在本文描述

3、不考虑virtualization的支持，也就是说EL2的异常处理不会在本文描述

一句话总结，本文主要描述EL0和EL1这两个exception level下的异常向量表的设定。

二、 异常（exception）的基础知识

1、什么是异常（exception）？

对于ARM64而言，exception是指cpu的某些异常状态或者一些系统的事件（可能来自外部，也可能来自内部），这些状态或者事件可以导致cpu去执行一些预先设定的，具有更高执行权利的软件（也叫exception handler）。执行exception handler可以进行异常的处理，从而让系统平滑的运行。exception handler执行完毕之后，需要返回发生异常的现场。上面这段话非常的拗口，但是不要紧，当进入细节的时候一切会慢慢清晰起来，下面我们逐一介绍各种异常：中断（interrupt），abort，reset，异常指令……

2、exception level

上一节中，我们在定义异常的时候说道：一旦异常发生，系统（包括硬件和软件）将切换到具有更高执行权利的状态，对于cpu而言，就是exception level了，ARM64最大支持EL0～EL3四个exception level，EL0的execution privilege最低，EL3的execution privilege最高。当发生异常的时候，系统的exception会迁移到更高的exception level或者维持不变，但是绝不会降低。此外，不会有任何的异常会去到EL0。

对比是一个不错的学习方法，我们先看看ARM32的情况。对于ARM32而言，cpu可以处在各种processor mode下，例如User、FIQ、IRQ、Abort、Undefined、System，这些不同的mode又对应两种privilege level，non-privilege（user processor mode）和privilege（其他processor mode）。

来到ARM64，processor mode这个概念已经消失了，取而代之的是exception level，如果没有支持两个security state（但是支持虚拟化），那么ARM64有3个exception level，分别是：EL0（对应user mode的application），EL1（guest OS）和EL2（Hypervisor）。如果支持两个security state（但是不支持虚拟化），ARM64还是有3个exception level，分别是：EL0（对应trusted service），EL1（trusted OS kernel）和EL3（Secure monitor）。如果支持了虚拟化并且同时支持两种security state，那么ARM64的处理器可以处于4种exception level，具体如下（摘自wowo同学的文章里面的图片，^\_^）：

![](http://www.wowotech.net/content/uploadfile/201507/7ffa71881f6356c740ef254b01cecf1520150707143059.gif)

由于EL3是和安全相关的，目前linux kernel并不涉及这部分的内容，因此本文将不考虑EL3这个exception level。

2、异步异常（asynchronous exception）和同步异常（synchronous exception）

虽然异常各具形态，但是基本可以分成两类，一类是asynchronous exception，另外一类是synchronous exception。

asynchronous exception基本上可以类似大家平常说的中断，它是毫无预警的，丝毫不考虑cpu core感受的外部事件（需要注意的是：外部并不是表示外设，这里的外部是针对cpu core而言，有些中断是来自SOC的其他HW block，例如GIC，这时候，对于processor或者cpu（指soc）而言，这些事件是内部的），这些事件打断了cpu core对当前软件的执行，因此称之interrupt。interrupt或者说asynchronous exception有下面的特点：

（1）异常和CPU执行的指令无关。

（2）返回地址是硬件保存下来并提供给handler，以便进行异常返回现场的处理。这个返回地址并非产生异常时的指令

根据这个定义IRQ、FIQ和SError interrupt属于asynchronous exception。

synchronous exception和asynchronous exception相反，其特点是：

（1）异常的产生是和cpu core执行的指令或者试图执行执行相关

（2）硬件提供给handler的返回地址就是产生异常的那一条指令所在的地址

synchronous exception又可以细分成两个类别，一种我们称之为synchronous abort，例如未定义的指令、data abort、prefetch instruction abort、SP未对齐异常，debug exception等等。还有一种是正常指令执行造成的，包括SVC/HVC/SMC指令，这些指令的使命就是产生异常。

3、precise exception

什么是precise exception呢？现代的cpu设计越来越复杂，各种pipe技术，各种cache技术，分支预测，multi-issue，out-of-order-execution等等，这些都让cpu core执行指令处于一个混沌的状态（主要是指不象sequential cpu 那么清晰），因此，当发生一个exception的时候，需要一个快刀来斩断那些处于各种纠缠中的乱麻一样的各种cpu以及memory的状态。这时候，我们需要选定一条指令，该指令之前的指令（不包括该指令）都已经全部执行完毕，修改了硬件上下文（cpu状态寄存器，各种通用寄存器，系统寄存器等等，memory hierarchy状态）。而该指令之后（包括该指令）的指令都没有执行（各种HW状态保持该指令之前状态），如果有执行的需要回退（exception的检测发生在流水线的write back stage，比较靠后，我们选定的那条指令应该在流水线的execute stage，是不是需要回退我也不知道，反正HW设计的时候需要考虑如何处理execute stage之后那些在流水线中的指令，确保硬件状态的一致性）。OK，快刀已经斩在了两条指令之间，让hardware context定格在这一刻，并把控制权转交给exception handler。

为何要定义precise exception呢？有什么用呢？无它，让生活变得简单一些（当然这里的生活不是去超市买东西，是指软件debug之类的程序员生活）。例如：这样exception handler返回现场就比较容易了。

在ARM64中，除了SError interrupt这种exception，其他的exception都是precise exception。

三、异常的分类

1、中断

中断主要有两种，physical interrupt和virtual interrupt（不在本文中描述）。physical interrupt是来自cpu core（或者叫做PE）外部一种信号，它包括下面三种类型：

（1）IRQ

（2）FIQ

（3）System error，简称SError

IRQ和FIQ是广大ARM嵌入式工程师的老朋友了，大家常说的中断实际上特指IRQ和FIQ，当然，实际上SError也是一种中断类型。IRQ和FIQ分别和cpu core的nIRQ和nFIQ这两根实际的信号线相关，interrupt controller收集各种来自外设的（或者来自其他CPU core的）中断请求，通过nIRQ和nFIQ的电平信号（active low）来通知cpu core某些外设的异步事件（或者来自其他CPU core的异步事件）。其中IRQ是普通中断，而FIQ是快速中断。由于中断来自cpu core的外部，可以在任何的时间点插入，和cpu core上执行的指令没有任何的关系，因此中断这种exception被归入asynchronous exception类别。

要想理解SError interrupt这个概念，我们需要先看看external abort这个术语。external abort来自memory system，当然不是所有的来自memory system的abort都是external abort，例如来自MMU的abort就不是external abort（这里的external是针对processor（而非cpu core）而言，因此MMU实际上是internal的）。external abort发生在processor通过bus访问memory的时候（可能是直接对某个地址的读或者写，也可能是取指令导致的memory access），processor在bus上发起一次transaction，在这个过程中发生了abort，abort来自processor之外的memory block、device block或者interconnection block，用来告知processor，搞不定了，你自己看着办吧。external abort可以被实现成synchronous exception（precise exception），也可以实现成asynchronous exception（imprecise exception）。如果external abort是asynchronous的，那么它可以通过SError interrupt来通知cpu core。

我的习惯是搞清楚一个术语的定义之后，下一个问题就是why，不过大部分的资料都不会讲why，因此在回答why的时候，往往没有那么权威，多半是自己的思考，不保证是正确的。为何要定义SError interrupt？当一个abort发生的时候，如果HW如果能够通过寄存器提供一些信息的话，那么事情还没有那么糟糕，例如return address给出了发生exception的具体的位置。从这个角度看，软件工程师更喜欢precise exception，毕竟知道导致异常的指令的位置，从而找到一些蛛丝马迹，让系统恢复。从这个角度看，根本就不应该存在SError interrupt这样的imprecise exception的鬼东西，HW没有提供有意义的信息，就只是两手一摊，反正在memory access的时候发生了system error，而且我也不知道是哪一条指令干的，这让软件工程师情何以堪呐。氮素，这只是事情的一个方面，从设计CPU的IC工程师角度看，他们更喜欢设计最快的处理器来完成自己的价值。SError interrupt是imprecise exception，允许更大的指令并发，因此性能更好。

2、reset

reset是一种优先级最高的异常，无法mask。系统首次上电，watchdog以及软件控制都可以让cpu core历经一次reset。reset有两种，一种是cold reset，另外一种是warm reset，它们之间唯一的不同是是否reset cpu core build-in的debug HW block。

3、abort

abort有两种，一种是和指令的执行有关，进入异常状态时候保持的返回地址精准的反应了发生异常的那一条指令，我们称之synchronoud abort。有同步就有异步，asynchronous abort（也就是上面描述的SError interrupt）和执行的指令无关，返回地址也不会提供abort时候的执行指令相关的精准信息。asynchronous abort和中断类似，只不过中断多半是来自外部（外设或者其他cpu core），而asynchronous abort来自外部memory system，源于bus上的一些错误，例如不可恢复的ECC error。

synchronoud abort有可能在cpu执行指令过程中的任何一步发生。例如在取指阶段失败，在译码阶段失败，在指令执行阶段等等。synchronoud abort和指令的执行过程有关，abort有可能在很早的阶段就被感知到，例如cpu core在将保存在memory系统中的指令读取到cpu core内部准备译码执行的时候就发现了错误，怎么办呢？由于指令还没有进入执行阶段（正在执行的是pipeline中的其他指令），因此不能触发exception，而是仅仅mark这个abort，一旦该指令在pipeline中到底execute stage，一次synchronoud abort被触发了。如果还没有执行cpu core就flush了这个pipeline（例如该指令的上一条指令是跳转指令），那么这次abort不会触发。

4、异常指令

对于ARM而言，system call被运行在non-privilege的软件用来向运行在privilege mode（例如linux kernel）的软件申请服务。来到ARM64，privilege包括了EL1/EL2/EL3，随之而来的是system call这个概念的扩展。从low privilege level都可以通过系统调用来申请higer privilege level的服务，因此，在ARM64中，能正常产生异常（不是abort）并申请拥有更高权利的软件服务的指令有三条：

（1）Supervisor Call (SVC)指令。类似于ARM时代的SWI指令，主要被EL0（user mode）的软件用来申请EL1（OS service）软件的服务。

（2）Hypervisor Call (HVC) 指令。主要被guest OS用来请求hypervisor的服务。

（3）Secure monitor Call (SMC) 指令，用来切换不同的世界，normal world或是secure world。

毫无疑问，SVC/HVC/SMC指令产生的异常属于synchronous exception。

四、ARM64为了异常处理提供了哪些帮助？

1、一旦发生了异常，处理器会切换到哪一个exception level？

对于ARM处理器，发生了不同的异常会进入不同的processor mode，例如发生了IRQ中断，处理器进入IRQ mode（privilege mode的一种）。对于ARM64，这个问题变成了处理器会切换到哪一个exception level？我们首先看reset和异常指令，因为这两个是最简单的。我们先看最简单的reset exception，当发生reset之后，处理处于那个EL呢？这个是和实现相关（实现的最高的EL是哪一个），一般而言，ARM64处理在reset的时候缺省会进入最高的那个exception level，例如如果处理器最高支持的EL是EL2，那么reset之后，系统就是处于EL2。对于那些正常通过system call产生的异常，处理器会切换到哪一个exception level这个问题也很好回答，SVC、HVC和SMC都有自己target exception level。

对于中断和abort，稍微有些复杂，这是通过各种寄存器可以进行配置的，具体参考ARM ARM文档。对于本文的场景，我们不支持EL2和EL3，因此exception的路由配置比较简单，大部分的异常都被路由到EL1来处理。

2、为了方便软件工程师处理各个exception level 的exception handler，PE提供了哪些寄存器的支持？

不同的exception level使用相同的general purpose registers，按64bit访问的话，寄存器是x0～x30共计31个寄存器，如果按照32bit访问的话，寄存器是w0～w30。

每个exception level都有自己的stack pointer register，名字是SP_ELx，当然，前提是处理器支持该EL。对于EL0，只能使用SP_EL0，对于其他的exception level，除了可以使用自己的SP_ELx，还可以选择使用SP_EL0。

当发生异常的时候，PE总是把当前cpu的状态保存在SPSR寄存器中，该寄存器有三个，分别是SPSR_EL1，SPSR_EL2，SPSR_EL3，异常迁移到哪一个exception level就使用哪一个SPSR。由于不会有异常把系统状态迁移到EL0，因此也就不存在SPSR_EL0了。在返回异常现场的时候，可以使用SPSR_ELx来恢复PE的状态。

当发生异常的时候，PE把一个适当的返回地址保存在ELR（Exception Link Register）寄存器中，该寄存器有三个，分别是ELR_EL1，ELR_EL2，ELR_EL3，异常迁移到哪一个exception level就使用哪一个ELR。同样的，由于不会有异常把系统状态迁移到EL0，因此也就不存在ELR_EL0了。在返回异常现场的时候，可以使用ELR_ELx来恢复PC值。

对于abort（包括synchronous exception和SError interrupt），ESR寄存器（ Exception Syndrome Register）保存了更详细的异常信息。ESR寄存器有三个，分别是ESR_EL1，ESR_EL2，ESR_EL3。

3、如何定义exception handler？

系统有那么多异常，不同的异常有可以将处理器状态迁移到不同的exception level中，如何组织这些exception handler呢？第一阶是各个exception level的Vector Base Address Register (VBAR)寄存器，该寄存器保存了各个exception level的异常向量表的基地址。该寄存器有三个，分别是VBAR_EL1，VBAR_EL2，VBAR_EL3。

具体的exception handler是通过vector base address ＋ offset得到，offset的定义如下表所示：

|   |   |   |   |   |
|---|---|---|---|---|
|exception level迁移情况|Synchronous exception的offset值|IRQ和vIRQ exception的offset值|FIQ和vFIQ exception的offset值|SError和vSError exception的offset值|
|同级exception level迁移，使用SP_EL0。例如EL1迁移到EL1|0x000|0x080|0x100|0x180|
|同级exception level迁移，使用SP_ELx。例如EL1迁移到EL1|0x200|0x280|0x300|0x380|
|ELx迁移到ELy，其中y>x并且ELx处于AArch64状态|0x400|0x480|0x500|0x580|
|ELx迁移到ELy，其中y>x并且ELx处于AArch32状态|0x600|0x680|0x700|0x780|

五、代码分析

对于linux kernel而言，它可以有两种情况：一种是做Hypervisor，另外一种是做Guest OS。linux kernel一般不会做Trusted OS，一般Trusted OS都是size比较小，比较轻盈的OS。因此，对于linux kernel而言，我们只要设定两个异常向量表：一个是做为guest OS（或者普通OS）的EL1 EL0的异常向量表，另外一个是for Hypervisor的EL2的异常向量表。

1、EL1的异常向量表。当发生一个异常，并且处理器迁移到了EL1这个exception level，那么该异常由EL1的异常向量表来决定如果跳转到相应的exception handler。具体代码如下：

> .align    11－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－（1）\
> ENTRY(vectors)－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－（2）\
> ventry    el1_sync_invalid        // Synchronous EL1t－－－－－－－－－－－－－－（3）\
> ventry    el1_irq_invalid            // IRQ EL1t\
> ventry    el1_fiq_invalid            // FIQ EL1t\
> ventry    el1_error_invalid        // Error EL1t
>
> ventry    el1_sync            // Synchronous EL1h －－－－－－－－－－－－－－－－（4）\
> ventry    el1_irq                // IRQ EL1h\
> ventry    el1_fiq_invalid            // FIQ EL1h\
> ventry    el1_error_invalid        // Error EL1h
>
> ventry    el0_sync            // Synchronous 64-bit EL0 －－－－－－－－－－－－－－（5）\
> ventry    el0_irq                // IRQ 64-bit EL0\
> ventry    el0_fiq_invalid            // FIQ 64-bit EL0\
> ventry    el0_error_invalid        // Error 64-bit EL0 \
> ventry    el0_sync_invalid        // Synchronous 32-bit EL0 －－－－－－－－－－－－（6）\
> ventry    el0_irq_invalid            // IRQ 32-bit EL0\
> ventry    el0_fiq_invalid            // FIQ 32-bit EL0\
> ventry    el0_error_invalid        // Error 32-bit EL0\
> END(vectors)

（1）EL1的异常向量表保存在VBAR_EL1寄存器中（Vector Base Address Register (EL1)），该寄存器的低11bit是reserve的，11～63表示了Vector Base Address，因此这里的异常向量表是2K对齐的。

（2）exception vector table中有很多entry（否则也不会叫做vector table了），发生了异常具体选择哪一个entry需要考虑下面的这些因素：该exception从何而来？（对于本场景，exception只能来自EL0或者EL1），使用哪一个stack pointer？（SP_EL0还是SP_EL1），哪一种类型的异常？（synchronous exception、IRQ、FIQ还是SError interrupt）。

（3）异常向量表是分组的，每一组都包括四种类型的exception，分别对应synchronous exception（elx_sync or elx_sync_invalid），irq中断（elx_irq or elx_irq_invalid），fiq中断（elx_fiq or elx_fiq_invalid）以及SError中断（elx_error or elx_error_invalid）。这一组异常对应异常状态的迁移是EL1到EL1的迁移，并且选择使用了SP_EL0。对于linux kernel而言，这类exception vector实际上就是Invalid mode handlers。

（4）如果用大家熟悉的语言，其实这一段exception vectors可以这样表述：这些是异常发生在内核态（EL1）并且系统配置为内核处理这些异常（这些异常导致PE迁移到EL1）时候的异常向量。目前版本的ARM64代码还没有fiq和SError这两种异常的支持。fiq在ARM ARM文档中建议是在secure state世界中处理，因此没有出现在linux中是合理的，但是为何SError为何不处理呢？估计是因为SError是一种异步异常，硬件没有提供足够的信息恢复，因此linux只能是按照invalid mode来处理。

（5）异常发生在了用户态（EL0）并且该异常需要在内核态（EL1）中处理。

（6）同（5）不过发生异常的现场处于AArch32状态。

2、EL2的异常向量表。在本文的场景下（不支持虚拟化），实际上EL2的异常向量表没有存在的意义，如果硬件支持EL2，这时候，linux kernel可以做为hypervisor存在。启动的时候，在el2_setup函数中会设定一个dummy的EL2异常向量表（\_\_hyp_stub_vectors）。当然，真正的EL2的异常向量表会在kvm初始化的时候完成设定。

六、参考文献

1、ARM ARM

2、Computer Architecture, A Quantitative Approach, 5th

3、DEN0024A_v8_architecture_PG.pdf

_原创文章，转发请注明出处。蜗窝科技_

标签: [arm64](http://www.wowotech.net/tag/arm64) [exception](http://www.wowotech.net/tag/exception)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [显示技术介绍(2)\_电子显示的前世今生](http://www.wowotech.net/display/display_tech_intro.html) | [Linux进程冻结技术](http://www.wowotech.net/pm_subsystem/237.html)»

**评论：**

**伊斯科明**\
2020-07-30 17:16

大佬，你这昵称很危险啊

[回复](http://www.wowotech.net/armv8a_arch/238.html#comment-8075)

**renameit**\
2020-07-28 16:49

我很想知道，对于System error，这个V8独有的中断向量定义，对应的v7里面是怎么处理的

[回复](http://www.wowotech.net/armv8a_arch/238.html#comment-8071)

**[bsp](http://www.wowotech.net/)**\
2018-11-28 11:31

arm32的中断向量表地址一般是0xffff0000，然后中断或异常发生时 由硬件跳转到对应的地址，比如\
ffff1004 t vector_rst\
ffff1020 t vector_irq\
ffff10a0 t vector_dabt\
ffff1120 t vector_pabt\
ffff11a0 t vector_und\
ffff1220 t vector_addrexcptn\
ffff1240 T vector_fiq\
那么，Arm-v8a在硬件层是怎么跳转的呢？

[回复](http://www.wowotech.net/armv8a_arch/238.html#comment-7059)

**[bsp](http://www.wowotech.net/)**\
2018-11-28 11:39

@bsp：sorry，看玩了文章，已经有所了解，多谢楼主！

[回复](http://www.wowotech.net/armv8a_arch/238.html#comment-7060)

**[badao](http://cole.sciecne/)**\
2018-07-04 13:50

十分感谢，最近在看arm的异常分发流程，看到您的博客真的是省了很多时间

[回复](http://www.wowotech.net/armv8a_arch/238.html#comment-6836)

**CHENYAN**\
2016-09-27 16:45

看了你的文章深受启发,最近一直在研究ARM64 Linux的系统调用的HOOK方法.记得以前X86平台下可以通过IDT寄存器获取系统调用表的基地址.我看了ARM64的内核代码,貌似没有类似的寄存器.想问一下ARM64由办法获取系统调用表的基地址吗?感谢楼主

[回复](http://www.wowotech.net/armv8a_arch/238.html#comment-4617)

**[linuxer](http://www.wowotech.net/)**\
2016-09-27 19:12

@CHENYAN：对于ARM64，VBAR_ELx寄存器保存中各个exception level的异常向量表的基地址，应该类似与X86的IDT寄存器。

你问题中的系统调用表是指什么？是指保存各个系统调用跳转函数的入口表格（sys_call_table）吗？这个表是软件定义，和硬件无关啊。

[回复](http://www.wowotech.net/armv8a_arch/238.html#comment-4619)

**CHENYAN**\
2016-09-28 10:14

@linuxer：@linuxer 多谢你的回复!

我的需求是要能够在自己编写的内核模块获取sys_call_table,然后通过sys_call_table获取和修改系统调用表中的跳转地址. 比如 :\
read函数指针类型 old_read = sys_call_table\[\_NR_read\];\
sys_call_table\[\_\_NR_read\] = xxxx;

我看ARM64平台的sys_call_table是在sys.c文件中定义的,\
void \*sys_call_table\[\_\_NR_syscalls\] \_\_aligned(4096) = {\
\[0 ... \_\_NR_syscalls - 1\] = sys_ni_syscall,\
#include \<asm/unistd.h>\
};\
在entry.S文件中直接访问 : adrp  stbl, sys_call_table.\
这样我的理解是, 自己编写的内核模块代码根本没办法访问sys_call_table, 因为sys_call_table入口既和寄存器没有半毛钱关系, 又没有哪个头文件有extern const unsigned long sys_call_table\[\] 这样的语句.

请问我的理解正确吗? 还有其他的思路能获取sys_call_table吗?

[回复](http://www.wowotech.net/armv8a_arch/238.html#comment-4622)

**[linuxer](http://www.wowotech.net/)**\
2016-09-28 11:16

@CHENYAN：sys_call_table本身是一个内核符号，如果你的内核模块编译进入kernel image，那么其实你可以直接访问这个符号。如果你的内核模块是以模块的形式插入内核，那么就有点麻烦了，因为sys_call_table不是导出符号（EXPORT_SYMBOL），你的模块应该是无法访问的。

[回复](http://www.wowotech.net/armv8a_arch/238.html#comment-4624)

**CHENYAN**\
2016-09-28 13:51

@linuxer：感谢你的回复! 这样说来我基本都明白了!

[回复](http://www.wowotech.net/armv8a_arch/238.html#comment-4626)

**tchvsps**\
2016-12-05 11:29

@CHENYAN：请问你看懂\
\[0 ... \_\_NR_syscalls - 1\] = sys_ni_syscall\
这行代码了吗？

[回复](http://www.wowotech.net/armv8a_arch/238.html#comment-4966)

**[wowo](http://www.wowotech.net/)**\
2016-12-05 16:36

@tchvsps：定义一个指针数组，并将其中的每个元素都初始化为sys_ni_syscall，效果和下面的类似：\
char sa\[8\] = {                                                                  \
\[0 ... 7\] = 0xaa,                                                      \
};

[回复](http://www.wowotech.net/armv8a_arch/238.html#comment-4967)

**[madang](http://www.wowotech.net/)**\
2015-12-30 11:53

这一组异常对应异常状态的迁移是EL1到EL1的迁移，并且选择使用了SP_EL0。对于linux kernel而言，这类exception vector实际上就是Invalid mode handlers\
//------------------------------\
请教一下：\
这是什么场景下要用的异常向量表？没看明白。\
同时Synchronous EL1t ，Synchronous EL1h 分表代表什么意思？

[回复](http://www.wowotech.net/armv8a_arch/238.html#comment-3308)

**[linuxer](http://www.wowotech.net/)**\
2015-12-30 12:35

@madang：不同的exception level可以选择其使用的栈指针：\
1、对于EL0，不能选择只能使用EL0_SP，我们简称为EL0t\
2、对于EL1，可以选择使用EL0_SP，我们简称为EL1t，或者选择EL1_SP，我们简称EL1h\
3、对于EL2，可以选择使用EL0_SP，我们简称为EL2t，或者选择EL2_SP，我们简称EL1h\
4、对于EL3，可以选择使用EL0_SP，我们简称为EL3t，或者选择EL3_SP，我们简称EL1h

t表示thread，h表示handler

当发生一次异常之后，系统可以迁移到某个exception level（可能是t，也可能是h）。

因此这一组异常向量对应系统迁移到EL1t这个状态。

[回复](http://www.wowotech.net/armv8a_arch/238.html#comment-3309)

**[tigger](http://www.wowotech.net/)**\
2015-11-26 14:04

soga 哈哈

[回复](http://www.wowotech.net/armv8a_arch/238.html#comment-3145)

**[tigger](http://www.wowotech.net/)**\
2015-11-26 11:18

上面那张讲exception level的图，secure firmware 不应该在 secure EL0 吧，应该在EL3, secure EL0 应该放的是secure的APP

[回复](http://www.wowotech.net/armv8a_arch/238.html#comment-3142)

**[linuxer](http://www.wowotech.net/)**\
2015-11-26 12:14

@tigger：我是借用wowo同学的图片，如果是我画的话我会写 trusted application。\
不过没有关系，领会精神就OK了（^\_^，主要是我不想修改图片了）

[回复](http://www.wowotech.net/armv8a_arch/238.html#comment-3143)

**[wowo](http://www.wowotech.net/)**\
2015-11-26 15:29

@linuxer：哈哈，话说Tiger同学的眼神是相当的好啊。其实这个图片也不只我自己画的，主流的说法（包括ARM的Spec），都是trusted application。我之所以copy了一个非主流的，主要是里面隐藏的一个含义：trusted application的地位和用户application并不对等，虽然从软件层次上在一个level，但在功能上，trusted application更像一个firmware。PS：在嵌入式的语意里，firmware和application的概念类似，只是不能随意改动而已。

[回复](http://www.wowotech.net/armv8a_arch/238.html#comment-3148)

**[tigger](http://www.wowotech.net/)**\
2015-11-26 17:13

@wowo：最近在研究这个，所以比较关注这里。作为工程师就是要不断吸收新的东西，永不停歇啊

[回复](http://www.wowotech.net/armv8a_arch/238.html#comment-3149)

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

  - [ARM Linux上的系统调用代码分析](http://www.wowotech.net/process_management/syscall-arm.html)
  - [显示技术介绍(1)\_概述](http://www.wowotech.net/display/display_tech_overview.html)
  - [我眼中的可穿戴设备](http://www.wowotech.net/tech_discuss/106.html)
  - [进程管理和终端驱动：基本概念](http://www.wowotech.net/process_management/process-tty-basic.html)
  - [SDRAM Internals](http://www.wowotech.net/basic_tech/sdram-internals.html)

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

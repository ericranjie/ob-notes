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

作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2014-8-4 18:26 分类：[中断子系统](http://www.wowotech.net/sort/irq_subsystem)

一、前言

本文主要以ARM体系结构下的中断处理为例，讲述整个中断处理过程中的硬件行为和软件动作。具体整个处理过程分成三个步骤来描述：

1、第二章描述了中断处理的准备过程

2、第三章描述了当发生中的时候，ARM硬件的行为

3、第四章描述了ARM的中断进入过程

4、第五章描述了ARM的中断退出过程

本文涉及的代码来自3.14内核。另外，本文注意描述ARM指令集的内容，有些source code为了简短一些，删除了THUMB相关的代码，除此之外，有些debug相关的内容也会删除。

二、中断处理的准备过程

1、中断模式的stack准备

ARM处理器有多种processor mode，例如user mode（用户空间的AP所处于的模式）、supervisor mode（即SVC mode，大部分的内核态代码都处于这种mode）、IRQ mode（发生中断后，处理器会切入到该mode）等。对于linux kernel，其中断处理处理过程中，ARM 处理器大部分都是处于SVC mode。但是，实际上产生中断的时候，ARM处理器实际上是进入IRQ mode，因此在进入真正的IRQ异常处理之前会有一小段IRQ mode的操作，之后会进入SVC mode进行真正的IRQ异常处理。由于IRQ mode只是一个过度，因此IRQ mode的栈很小，只有12个字节，具体如下：

> struct stack {  
>     u32 irq[3];  
>     u32 abt[3];  
>     u32 und[3];  
> } ____cacheline_aligned;
> 
>   
> static struct stack stacks[NR_CPUS];

除了irq mode，linux kernel在处理abt mode（当发生data abort exception或者prefetch abort exception的时候进入的模式）和und mode（处理器遇到一个未定义的指令的时候进入的异常模式）的时候也是采用了相同的策略。也就是经过一个简短的abt或者und mode之后，stack切换到svc mode的栈上，这个栈就是发生异常那个时间点current thread的内核栈。anyway，在irq mode和svc mode之间总是需要一个stack保存数据，这就是中断模式的stack，系统初始化的时候，cpu_init函数中会进行中断模式stack的设定：

> void notrace cpu_init(void)  
> {
> 
>     unsigned int cpu = smp_processor_id();－－－－－－获取CPU ID  
>     struct stack *stk = &stacks[cpu];－－－－－－－－－获取该CPU对于的irq abt和und的stack指针
> 
> ……
> 
> #ifdef CONFIG_THUMB2_KERNEL  
> #define PLC    "r"－－－－－－Thumb-2下，msr指令不允许使用立即数，只能使用寄存器。  
> #else  
> #define PLC    "I"  
> #endif
> 
>   
>     __asm__ (  
>     "msr    cpsr_c, %1\n\t"－－－－－－让CPU进入IRQ mode  
>     "add    r14, %0, %2\n\t"－－－－－－r14寄存器保存stk->irq  
>     "mov    sp, r14\n\t"－－－－－－－－设定IRQ mode的stack为stk->irq  
>     "msr    cpsr_c, %3\n\t"  
>     "add    r14, %0, %4\n\t"  
>     "mov    sp, r14\n\t"－－－－－－－－设定abt mode的stack为stk->abt  
>     "msr    cpsr_c, %5\n\t"  
>     "add    r14, %0, %6\n\t"  
>     "mov    sp, r14\n\t"－－－－－－－－设定und mode的stack为stk->und  
>     "msr    cpsr_c, %7"－－－－－－－－回到SVC mode  
>         :－－－－－－－－－－－－－－－－－－－－上面是code，下面的output部分是空的  
>         : "r" (stk),－－－－－－－－－－－－－－－－－－－－－－对应上面代码中的%0  
>           PLC (PSR_F_BIT | PSR_I_BIT | IRQ_MODE),－－－－－－对应上面代码中的%1  
>           "I" (offsetof(struct stack, irq[0])),－－－－－－－－－－－－对应上面代码中的%2  
>           PLC (PSR_F_BIT | PSR_I_BIT | ABT_MODE),－－－－－－以此类推，下面不赘述  
>           "I" (offsetof(struct stack, abt[0])),  
>           PLC (PSR_F_BIT | PSR_I_BIT | UND_MODE),  
>           "I" (offsetof(struct stack, und[0])),  
>           PLC (PSR_F_BIT | PSR_I_BIT | SVC_MODE)  
>         : "r14");－－－－－－－－上面是input操作数列表，r14是要clobbered register列表  
> }

嵌入式汇编的语法格式是：asm(code : output operand list : input operand list : clobber list);大家对着上面的code就可以分开各段内容了。在input operand list中，有两种限制符（constraint），"r"或者"I"，"I"表示立即数（Immediate operands），"r"表示用通用寄存器传递参数。clobber list中有一个r14，表示在汇编代码中修改了r14的值，这些信息是编译器需要的内容。

对于SMP，bootstrap CPU会在系统初始化的时候执行cpu_init函数，进行本CPU的irq、abt和und三种模式的内核栈的设定，具体调用序列是：start_kernel--->setup_arch--->setup_processor--->cpu_init。对于系统中其他的CPU，bootstrap CPU会在系统初始化的最后，对每一个online的CPU进行初始化，具体的调用序列是：start_kernel--->rest_init--->kernel_init--->kernel_init_freeable--->kernel_init_freeable--->smp_init--->cpu_up--->_cpu_up--->__cpu_up。__cpu_up函数是和CPU architecture相关的。对于ARM，其调用序列是__cpu_up--->boot_secondary--->smp_ops.smp_boot_secondary(SOC相关代码)--->secondary_startup--->__secondary_switched--->secondary_start_kernel--->cpu_init。

除了初始化，系统电源管理也需要irq、abt和und stack的设定。如果我们设定的电源管理状态在进入sleep的时候，CPU会丢失irq、abt和und stack point寄存器的值，那么在CPU resume的过程中，要调用cpu_init来重新设定这些值。

2、SVC模式的stack准备

我们经常说进程的用户空间和内核空间，对于一个应用程序而言，可以运行在用户空间，也可以通过系统调用进入内核空间。在用户空间，使用的是用户栈，也就是我们软件工程师编写用户空间程序的时候，保存局部变量的stack。陷入内核后，当然不能用用户栈了，这时候就需要使用到内核栈。所谓内核栈其实就是处于SVC mode时候使用的栈。

在linux最开始启动的时候，系统只有一个进程（更准确的说是kernel thread），就是PID等于0的那个进程，叫做swapper进程（或者叫做idle进程）。该进程的内核栈是静态定义的，如下：

> union thread_union init_thread_union __init_task_data =  
>     { INIT_THREAD_INFO(init_task) };
> 
> union thread_union {  
>     struct thread_info thread_info;  
>     unsigned long stack[THREAD_SIZE/sizeof(long)];  
> };

对于ARM平台，THREAD_SIZE是8192个byte，因此占据两个page frame。随着初始化的进行，Linux kernel会创建若干的内核线程，而在进入用户空间后，user space的进程也会创建进程或者线程。Linux kernel在创建进程（包括用户进程和内核线程）的时候都会分配一个（或者两个，和配置相关）page frame，具体代码如下：

> static struct task_struct *dup_task_struct(struct task_struct *orig)  
> {  
>     ......  
>   
>     ti = alloc_thread_info_node(tsk, node);  
>     if (!ti)  
>         goto free_tsk;  
>   
>     ......  
> }

底部是struct thread_info数据结构，顶部（高地址）就是该进程的内核栈。当进程切换的时候，整个硬件和软件的上下文都会进行切换，这里就包括了svc mode的sp寄存器的值被切换到调度算法选定的新的进程的内核栈上来。

3、异常向量表的准备

对于ARM处理器而言，当发生异常的时候，处理器会暂停当前指令的执行，保存现场，转而去执行对应的异常向量处的指令，当处理完该异常的时候，恢复现场，回到原来的那点去继续执行程序。系统所有的异常向量（共计8个）组成了异常向量表。向量表（vector table）的代码如下：

> .section .vectors, "ax", %progbits  
> __vectors_start:  
>     W(b)    vector_rst  
>     W(b)    vector_und  
>     W(ldr)    pc, __vectors_start + 0x1000  
>     W(b)    vector_pabt  
>     W(b)    vector_dabt  
>     W(b)    vector_addrexcptn  
>     W(b)    vector_irq ---------------------------IRQ Vector  
>     W(b)    vector_fiq

对于本文而言，我们重点关注vector_irq这个exception vector。异常向量表可能被安放在两个位置上：

（1）异常向量表位于0x0的地址。这种设置叫做Normal vectors或者Low vectors。

（2）异常向量表位于0xffff0000的地址。这种设置叫做high vectors

具体是low vectors还是high vectors是由ARM的一个叫做的SCTLR寄存器的第13个bit （vector bit）控制的。对于启用MMU的ARM Linux而言，系统使用了high vectors。为什么不用low vector呢？对于linux而言，0～3G的空间是用户空间，如果使用low vector，那么异常向量表在0地址，那么则是用户空间的位置，因此linux选用high vector。当然，使用Low vector也可以，这样Low vector所在的空间则属于kernel space了（也就是说，3G～4G的空间加上Low vector所占的空间属于kernel space），不过这时候要注意一点，因为所有的进程共享kernel space，而用户空间的程序经常会发生空指针访问，这时候，内存保护机制应该可以捕获这种错误（大部分的MMU都可以做到，例如：禁止userspace访问kernel space的地址空间），防止vector table被访问到。对于内核中由于程序错误导致的空指针访问，内存保护机制也需要控制vector table被修改，因此vector table所在的空间被设置成read only的。在使用了MMU之后，具体异常向量表放在那个物理地址已经不重要了，重要的是把它映射到0xffff0000的虚拟地址就OK了，具体代码如下：

> static void __init devicemaps_init(const struct machine_desc *mdesc)  
> {  
>     ……  
>     vectors = early_alloc(PAGE_SIZE * 2); －－－－－分配两个page的物理页帧
> 
>     early_trap_init(vectors); －－－－－－－copy向量表以及相关help function到该区域
> 
>     ……  
>     map.pfn = __phys_to_pfn(virt_to_phys(vectors));  
>     map.virtual = 0xffff0000;  
>     map.length = PAGE_SIZE;  
> #ifdef CONFIG_KUSER_HELPERS  
>     map.type = MT_HIGH_VECTORS;  
> #else  
>     map.type = MT_LOW_VECTORS;  
> #endif  
>     create_mapping(&map); －－－－－－－－－－映射0xffff0000的那个page frame
> 
>     if (!vectors_high()) {－－－如果SCTLR.V的值设定为low vectors，那么还要映射0地址开始的memory  
>         map.virtual = 0;  
>         map.length = PAGE_SIZE * 2;  
>         map.type = MT_LOW_VECTORS;  
>         create_mapping(&map);  
>     }
> 
>   
>     map.pfn += 1;  
>     map.virtual = 0xffff0000 + PAGE_SIZE;  
>     map.length = PAGE_SIZE;  
>     map.type = MT_LOW_VECTORS;  
>     create_mapping(&map); －－－－－－－－－－映射high vecotr开始的第二个page frame
> 
> ……  
> }

为什么要分配两个page frame呢？这里vectors table和kuser helper函数（内核空间提供的函数，但是用户空间使用）占用了一个page frame，另外异常处理的stub函数占用了另外一个page frame。为什么会有stub函数呢？稍后会讲到。

在early_trap_init函数中会初始化异常向量表，具体代码如下：

> void __init early_trap_init(void *vectors_base)  
> {  
>     unsigned long vectors = (unsigned long)vectors_base;  
>     extern char __stubs_start[], __stubs_end[];  
>     extern char __vectors_start[], __vectors_end[];  
>     unsigned i;
> 
>     vectors_page = vectors_base;
> 
>     将整个vector table那个page frame填充成未定义的指令。起始vector table加上kuser helper函数并不能完全的充满这个page，有些缝隙。如果不这么处理，当极端情况下（程序错误或者HW的issue），CPU可能从这些缝隙中取指执行，从而导致不可知的后果。如果将这些缝隙填充未定义指令，那么CPU可以捕获这种异常。  
>     for (i = 0; i < PAGE_SIZE / sizeof(u32); i++)  
>         ((u32 *)vectors_base)[i] = 0xe7fddef1;
> 
>   拷贝vector table，拷贝stub function  
>     memcpy((void *)vectors, __vectors_start, __vectors_end - __vectors_start);  
>     memcpy((void *)vectors + 0x1000, __stubs_start, __stubs_end - __stubs_start);
> 
>     kuser_init(vectors_base); －－－－copy kuser helper function
> 
>     flush_icache_range(vectors, vectors + PAGE_SIZE * 2);  
>     modify_domain(DOMAIN_USER, DOMAIN_CLIENT);  
>   
> }

一旦涉及代码的拷贝，我们就需要关心其编译连接时地址（link-time address)和运行时地址（run-time address）。在kernel完成链接后，__vectors_start有了其link-time address，如果link-time address和run-time address一致，那么这段代码运行时毫无压力。但是，目前对于vector table而言，其被copy到其他的地址上（对于High vector，这是地址就是0xffff00000），也就是说，link-time address和run-time address不一样了，如果仍然想要这些代码可以正确运行，那么需要这些代码是位置无关的代码。对于vector table而言，必须要位置无关。B这个branch instruction本身就是位置无关的，它可以跳转到一个当前位置的offset。不过并非所有的vector都是使用了branch instruction，对于软中断，其vector地址上指令是“W(ldr)    pc, __vectors_start + 0x1000 ”，这条指令被编译器编译成ldr     pc, [pc, #4080]，这种情况下，该指令也是位置无关的，但是有个限制，offset必须在4K的范围内，这也是为何存在stub section的原因了。

4、中断控制器的初始化

具体可以参考[GIC代码分析](http://www.wowotech.net/linux_kenrel/gic-irq-chip-driver.html)。

三、ARM HW对中断事件的处理

当一切准备好之后，一旦打开处理器的全局中断就可以处理来自外设的各种中断事件了。

当外设（SOC内部或者外部都可以）检测到了中断事件，就会通过interrupt requestion line上的电平或者边沿（上升沿或者下降沿或者both）通知到该外设连接到的那个中断控制器，而中断控制器就会在多个处理器中选择一个，并把该中断通过IRQ（或者FIQ，本文不讨论FIQ的情况）分发给该processor。ARM处理器感知到了中断事件后，会进行下面一系列的动作：

1、修改CPSR（Current Program Status Register）寄存器中的M[4:0]。M[4:0]表示了ARM处理器当前处于的模式（ processor modes）。ARM定义的mode包括：

|   |   |   |   |
|---|---|---|---|
|处理器模式|缩写|对应的M[4:0]编码|Privilege level|
|User|usr|10000|PL0|
|FIQ|fiq|10001|PL1|
|IRQ|irq|10010|PL1|
|Supervisor|svc|10011|PL1|
|Monitor|mon|10110|PL1|
|Abort|abt|10111|PL1|
|Hyp|hyp|11010|PL2|
|Undefined|und|11011|PL1|
|System|sys|11111|PL1|

一旦设定了CPSR.M，ARM处理器就会将processor mode切换到IRQ mode。

2、保存发生中断那一点的CPSR值（step 1之前的状态）和PC值

ARM处理器支持9种processor mode，每种mode看到的ARM core register（R0～R15，共计16个）都是不同的。每种mode都是从一个包括所有的Banked ARM core register中选取。全部Banked ARM core register包括：

|   |   |   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|
|Usr|System|Hyp|Supervisor|abort|undefined|Monitor|IRQ|FIQ|
|R0_usr|||||||||
|R1_usr|||||||||
|R2_usr|||||||||
|R3_usr|||||||||
|R4_usr|||||||||
|R5_usr|||||||||
|R6_usr|||||||||
|R7_usr|||||||||
|R8_usr||||||||R8_fiq|
|R9_usr||||||||R9_fiq|
|R10_usr||||||||R10_fiq|
|R11_usr||||||||R11_fiq|
|R12_usr||||||||R12_fiq|
|SP_usr||SP_hyp|SP_svc|SP_abt|SP_und|SP_mon|SP_irq|SP_fiq|
|LR_usr|||LR_svc|LR_abt|LR_und|LR_mon|LR_irq|LR_fiq|
|PC|||||||||
|CPSR|||||||||
|||SPSR_hyp|SPSR_svc|SPSR_abt|SPSR_und|SPSR_mon|SPSR_irq|SPSR_fiq|
|||ELR_hyp|||||||

在IRQ mode下，CPU看到的R0～R12寄存器、PC以及CPSR是和usr mode（userspace）或者svc mode（kernel space）是一样的。不同的是IRQ mode下，有自己的R13(SP，stack pointer）、R14（LR，link register）和SPSR（Saved Program Status Register）。

CPSR是共用的，虽然中断可能发生在usr mode（用户空间），也可能是svc mode（内核空间），不过这些信息都是体现在CPSR寄存器中。硬件会将发生中断那一刻的CPSR保存在SPSR寄存器中（由于不同的mode下有不同的SPSR寄存器，因此更准确的说应该是SPSR-irq，也就是IRQ mode中的SPSR寄存器）。

PC也是共用的，由于后续PC会被修改为irq exception vector，因此有必要保存PC值。当然，与其说保存PC值，不如说是保存返回执行的地址。对于IRQ而言，我们期望返回地址是发生中断那一点执行指令的下一条指令。具体的返回地址保存在lr寄存器中（注意：这个lr寄存器是IRQ mode的lr寄存器，可以表示为lr_irq）：

（1）对于thumb state，lr_irq ＝ PC

（2）对于ARM state，lr_irq ＝ PC － 4

为何要减去4？我的理解是这样的（不一定对）。由于ARM采用流水线结构，当CPU正在执行某一条指令的时候，其实取指的动作早就执行了，这时候PC值＝正在执行的指令地址 ＋ 8，如下所示：

－－－－> 发生中断的指令

               发生中断的指令＋4

－PC－－>发生中断的指令＋8

               发生中断的指令＋12

一旦发生了中断，当前正在执行的指令当然要执行完毕，但是已经完成取指、译码的指令则终止执行。当发生中断的指令执行完毕之后，原来指向（发生中断的指令＋8）的PC会继续增加4，因此发生中断后，ARM core的硬件着手处理该中断的时候，硬件现场如下图所示：

  

－－－－> 发生中断的指令

               发生中断的指令＋4 <-------中断返回的指令是这条指令

              发生中断的指令＋8

－PC－－>发生中断的指令＋12

  

这时候的PC值其实是比发生中断时候的指令超前12。减去4之后，lr_irq中保存了（发生中断的指令＋8）的地址。为什么HW不帮忙直接减去8呢？这样，后续软件不就不用再减去4了。这里我们不能孤立的看待问题，实际上ARM的异常处理的硬件逻辑不仅仅处理IRQ的exception，还要处理各种exception，很遗憾，不同的exception期望的返回地址不统一，因此，硬件只是帮忙减去4，剩下的交给软件去调整。

3、mask IRQ exception。也就是设定CPSR.I = 1

4、设定PC值为IRQ exception vector。基本上，ARM处理器的硬件就只能帮你帮到这里了，一旦设定PC值，ARM处理器就会跳转到IRQ的exception vector地址了，后续的动作都是软件行为了。

四、如何进入ARM中断处理

1、IRQ mode中的处理

IRQ mode的处理都在vector_irq中，vector_stub是一个宏，定义如下：

> .macro    vector_stub, name, mode, correction=0  
>     .align    5
> 
> vector_\name:  
>     .if \correction  
>     sub    lr, lr, #\correction－－－－－－－－－－－－－（1）  
>     .endif
> 
>     @  
>     @ Save r0, lr_ (parent PC) and spsr_  
>     @ (parent CPSR)  
>     @  
>     stmia    sp, {r0, lr}        @ save r0, lr－－－－－－－－（2）  
>     mrs    lr, spsr  
>     str    lr, [sp, #8]        @ save spsr
> 
>     @  
>     @ Prepare for SVC32 mode.  IRQs remain disabled.  
>     @  
>     mrs    r0, cpsr－－－－－－－－－－－－－－－－－－－－－－－（3）  
>     eor    r0, r0, #(\mode ^ SVC_MODE | PSR_ISETSTATE)  
>     msr    spsr_cxsf, r0
> 
>     @  
>     @ the branch table must immediately follow this code  
>     @  
>     and    lr, lr, #0x0f－－－lr保存了发生IRQ时候的CPSR，通过and操作，可以获取CPSR.M[3:0]的值
> 
>                             这时候，如果中断发生在用户空间，lr=0，如果是内核空间，lr=3  
> THUMB( adr    r0, 1f            )－－－－根据当前PC值，获取lable 1的地址  
> THUMB( ldr    lr, [r0, lr, lsl #2]  )－lr根据当前mode，要么是__irq_usr的地址 ，要么是__irq_svc的地址  
>     mov    r0, sp－－－－－－将irq mode的stack point通过r0传递给即将跳转的函数  
> ARM(    ldr    lr, [pc, lr, lsl #2]    )－－－根据mode，给lr赋值，__irq_usr或者__irq_svc  
>     movs    pc, lr            @ branch to handler in SVC mode－－－－－（4）  
> ENDPROC(vector_\name)
> 
>     .align    2  
>     @ handler addresses follow this label  
> 1:  
>     .endm

（1）我们期望在栈上保存发生中断时候的硬件现场（HW context），这里就包括ARM的core register。上一章我们已经了解到，当发生IRQ中断的时候，lr中保存了发生中断的PC＋4，如果减去4的话，得到的就是发生中断那一点的PC值。

（2）当前是IRQ mode，SP_irq在初始化的时候已经设定（12个字节）。在irq mode的stack上，依次保存了发生中断那一点的r0值、PC值以及CPSR值（具体操作是通过spsr进行的，其实硬件已经帮我们保存了CPSR到SPSR中了）。为何要保存r0值？因为随后的代码要使用r0寄存器，因此我们要把r0放到栈上，只有这样才能完完全全恢复硬件现场。

（3）可怜的IRQ mode稍纵即逝，这段代码就是准备将ARM推送到SVC mode。如何准备？其实就是修改SPSR的值，SPSR不是CPSR，不会引起processor mode的切换（毕竟这一步只是准备而已）。

（4）很多异常处理的代码返回的时候都是使用了stack相关的操作，这里没有。“movs    pc, lr ”指令除了字面上意思（把lr的值付给pc），还有一个隐含的操作（movs中‘s’的含义）：把SPSR copy到CPSR，从而实现了模式的切换。

2、当发生中断的时候，代码运行在用户空间

Interrupt dispatcher的代码如下：

> vector_stub    irq, IRQ_MODE, 4 －－－－－减去4，确保返回发生中断之后的那条指令
> 
> .long    __irq_usr            @  0  (USR_26 / USR_32)   <---------------------> base address + 0  
> .long    __irq_invalid            @  1  (FIQ_26 / FIQ_32)  
> .long    __irq_invalid            @  2  (IRQ_26 / IRQ_32)  
> .long    __irq_svc            @  3  (SVC_26 / SVC_32)<---------------------> base address + 12  
> .long    __irq_invalid            @  4  
> .long    __irq_invalid            @  5  
> .long    __irq_invalid            @  6  
> .long    __irq_invalid            @  7  
> .long    __irq_invalid            @  8  
> .long    __irq_invalid            @  9  
> .long    __irq_invalid            @  a  
> .long    __irq_invalid            @  b  
> .long    __irq_invalid            @  c  
> .long    __irq_invalid            @  d  
> .long    __irq_invalid            @  e  
> .long    __irq_invalid            @  f

这其实就是一个lookup table，根据CPSR.M[3:0]的值进行跳转（参考上一节的代码：and    lr, lr, #0x0f）。因此，该lookup table共设定了16个入口，当然只有两项有效，分别对应user mode和svc mode的跳转地址。其他入口的__irq_invalid也是非常关键的，这保证了在其模式下发生了中断，系统可以捕获到这样的错误，为debug提供有用的信息。

>     .align    5  
> __irq_usr:  
>     usr_entry－－－－－－－－－请参考本章第一节（1）保存用户现场的描述  
>     kuser_cmpxchg_check－－－和本文描述的内容无关，这些不就介绍了  
>     irq_handler－－－－－－－－－－核心处理内容，请参考本章第二节的描述  
>     get_thread_info tsk－－－－－－tsk是r9，指向当前的thread info数据结构  
>     mov    why, #0－－－－－－－－why是r8  
>     b    ret_to_user_from_irq－－－－中断返回，下一章会详细描述

why其实就是r8寄存器，用来传递参数的，表示本次放回用户空间相关的系统调用是哪个？中断处理这个场景和系统调用无关，因此设定为0。

（1）保存发生中断时候的现场。所谓保存现场其实就是把发生中断那一刻的硬件上下文（各个寄存器）保存在了SVC mode的stack上。

>     .macro    usr_entry  
>     sub    sp, sp, #S_FRAME_SIZE－－－－－－－－－－－－－－A  
>     stmib    sp, {r1 - r12} －－－－－－－－－－－－－－－－－－－B
> 
>     ldmia    r0, {r3 - r5}－－－－－－－－－－－－－－－－－－－－C  
>     add    r0, sp, #S_PC－－－－－－－－－－－－－－－－－－－D  
>     mov    r6, #-1－－－－orig_r0的值
> 
>     str    r3, [sp] －－－－保存中断那一刻的r0
> 
>   
>     stmia    r0, {r4 - r6}－－－－－－－－－－－－－－－－－－－－E  
>     stmdb    r0, {sp, lr}^－－－－－－－－－－－－－－－－－－－F  
>     .endm

A：代码执行到这里的时候，ARM处理已经切换到了SVC mode。一旦进入SVC mode，ARM处理器看到的寄存器已经发生变化，这里的sp已经变成了sp_svc了。因此，后续的压栈操作都是压入了发生中断那一刻的进程的（或者内核线程）内核栈（svc mode栈）。具体保存多少个寄存器值？S_FRAME_SIZE已经给出了答案，这个值是18个寄存器。r0～r15再加上CPSR也只有17个而已。先保留这个疑问，我们稍后回答。

B：压栈首先压入了r1～r12，这里为何不处理r0？因为r0在irq mode切到svc mode的时候被污染了，不过，原始的r0被保存的irq mode的stack上了。r13（sp）和r14（lr）需要保存吗，当然需要，稍后再保存。执行到这里，内核栈的布局如下图所示：

[![ir1](http://www.wowotech.net/content/uploadfile/201408/261fb83e53959bbb4c5a1010535250ef20140804112621.gif "ir1")](http://www.wowotech.net/content/uploadfile/201408/32572263b95c491559fbd6565ab8ac0120140804112618.gif)

stmib中的ib表示increment before，因此，在压入R1的时候，stack pointer会先增加4，重要是预留r0的位置。stmib    sp, {r1 - r12}指令中的sp没有“！”的修饰符，表示压栈完成后并不会真正更新stack pointer，因此sp保持原来的值。

C：注意，这里r0指向了irq stack，因此，r3是中断时候的r0值，r4是中断现场的PC值，r5是中断现场的CPSR值。

D：把r0赋值为S_PC的值。根据struct pt_regs的定义（这个数据结构反应了内核栈上的保存的寄存器的排列信息），从低地址到高地址依次为：

> ARM_r0  
> ARM_r1  
> ARM_r2  
> ARM_r3  
> ARM_r4  
> ARM_r5  
> ARM_r6  
> ARM_r7  
> ARM_r8  
> ARM_r9  
> ARM_r10  
> ARM_fp  
> ARM_ip  
> ARM_sp   
> ARM_lr  
> ARM_pc<---------add    r0, sp, #S_PC指令使得r0指向了这个位置  
> ARM_cpsr  
> ARM_ORIG_r0

为什么要给r0赋值？因此kernel不想修改sp的值，保持sp指向栈顶。

E：在内核栈上保存剩余的寄存器的值，根据代码，依次是r0，PC，CPSR和orig r0。执行到这里，内核栈的布局如下图所示：

[![ir2](http://www.wowotech.net/content/uploadfile/201408/cd10858b99efb3423f34f0c0ac57690b20140804112629.gif "ir2")](http://www.wowotech.net/content/uploadfile/201408/ea950dbe632d27f5a52c8bd99374722020140804112627.gif)

R0，PC和CPSR来自IRQ mode的stack。实际上这段操作就是从irq stack就中断现场搬移到内核栈上。

F：内核栈上还有两个寄存器没有保持，分别是发生中断时候sp和lr这两个寄存器。这时候，r0指向了保存PC寄存器那个地址（add    r0, sp, #S_PC），stmdb    r0, {sp, lr}^中的“db”是decrement before，因此，将sp和lr压入stack中的剩余的两个位置。需要注意的是，我们保存的是发生中断那一刻（对于本节，这是当时user mode的sp和lr），指令中的“^”符号表示访问user mode的寄存器。

（2）核心处理

irq_handler的处理有两种配置。一种是配置了CONFIG_MULTI_IRQ_HANDLER。这种情况下，linux kernel允许run time设定irq handler。如果我们需要一个linux kernel image支持多个平台，这是就需要配置这个选项。另外一种是传统的linux的做法，irq_handler实际上就是arch_irq_handler_default，具体代码如下：

>     .macro    irq_handler  
> #ifdef CONFIG_MULTI_IRQ_HANDLER  
>     ldr    r1, =handle_arch_irq  
>     mov    r0, sp－－－－－－－－设定传递给machine定义的handle_arch_irq的参数  
>     adr    lr, BSYM(9997f)－－－－设定返回地址  
>     ldr    pc, [r1]  
> #else  
>     arch_irq_handler_default  
> #endif  
> 9997:  
>     .endm

对于情况一，machine相关代码需要设定handle_arch_irq函数指针，这里的汇编指令只需要调用这个machine代码提供的irq handler即可（当然，要准备好参数传递和返回地址设定）。

情况二要稍微复杂一些（而且，看起来kernel中使用的越来越少），代码如下：

>     .macro    arch_irq_handler_default  
>     get_irqnr_preamble r6, lr  
> 1:    get_irqnr_and_base r0, r2, r6, lr  
>     movne    r1, sp  
>     @  
>     @ asm_do_IRQ 需要两个参数，一个是 irq number（保存在r0）  
>     @                                          另一个是 struct pt_regs *（保存在r1中）  
>     adrne    lr, BSYM(1b)－－－－－－－返回地址设定为符号1，也就是说要不断的解析irq状态寄存器
> 
>                                        的内容，得到IRQ number，直到所有的irq number处理完毕  
>     bne    asm_do_IRQ   
>     .endm

这里的代码已经是和machine相关的代码了，我们这里只是简短描述一下。所谓machine相关也就是说和系统中的中断控制器相关了。get_irqnr_preamble是为中断处理做准备，有些平台根本不需要这个步骤，直接定义为空即可。get_irqnr_and_base 有四个参数，分别是：r0保存了本次解析的irq number，r2是irq状态寄存器的值，r6是irq controller的base address，lr是scratch register。

对于ARM平台而言，我们推荐使用第一种方法，因为从逻辑上讲，中断处理就是需要根据当前的硬件中断系统的状态，转换成一个IRQ number，然后调用该IRQ number的处理函数即可。通过get_irqnr_and_base这样的宏定义来获取IRQ是旧的ARM SOC系统使用的方法，它是假设SOC上有一个中断控制器，硬件状态和IRQ number之间的关系非常简单。但是实际上，ARM平台上的硬件中断系统已经是越来越复杂了，需要引入interrupt controller级联，irq domain等等概念，因此，使用第一种方法优点更多。

3、当发生中断的时候，代码运行在内核空间

如果中断发生在内核空间，代码会跳转到__irq_svc处执行：

>     .align    5  
> __irq_svc:  
>     svc_entry－－－－保存发生中断那一刻的现场保存在内核栈上  
>     irq_handler －－－－具体的中断处理，同user mode的处理。
> 
> #ifdef CONFIG_PREEMPT－－－－－－－－和preempt相关的处理  
>     get_thread_info tsk  
>     ldr    r8, [tsk, #TI_PREEMPT]        @ get preempt count  
>     ldr    r0, [tsk, #TI_FLAGS]        @ get flags  
>     teq    r8, #0                @ if preempt count != 0  
>     movne    r0, #0                @ force flags to 0  
>     tst    r0, #_TIF_NEED_RESCHED  
>     blne    svc_preempt  
> #endif
> 
>     svc_exit r5, irq = 1            @ return from exception

一个task的thread info数据结构定义如下（只保留和本场景相关的内容）：

> struct thread_info {  
>     unsigned long        flags;        /* low level flags */  
>     int            preempt_count;    /* 0 => preemptable, <0 => bug */  
>     ……  
> };

flag成员用来标记一些low level的flag，而preempt_count用来判断当前是否可以发生抢占，如果preempt_count不等于0（可能是代码调用preempt_disable显式的禁止了抢占，也可能是处于中断上下文等），说明当前不能进行抢占，直接进入恢复现场的工作。如果preempt_count等于0，说明已经具备了抢占的条件，当然具体是否要抢占当前进程还是要看看thread info中的flag成员是否设定了_TIF_NEED_RESCHED这个标记（可能是当前的进程的时间片用完了，也可能是由于中断唤醒了优先级更高的进程）。

保存现场的代码和user mode下的现场保存是类似的，因此这里不再详细描述，只是在下面的代码中内嵌一些注释。

>     .macro    svc_entry, stack_hole=0  
>     sub    sp, sp, #(S_FRAME_SIZE + \stack_hole - 4)－－－－sp指向struct pt_regs中r1的位置  
>     stmia    sp, {r1 - r12} －－－－－－寄存器入栈。
> 
>     ldmia    r0, {r3 - r5}  
>     add    r7, sp, #S_SP - 4 －－－－－－r7指向struct pt_regs中r12的位置  
>     mov    r6, #-1 －－－－－－－－－－orig r0设为-1  
>     add    r2, sp, #(S_FRAME_SIZE + \stack_hole - 4)－－－－r2是发现中断那一刻stack的现场  
>     str    r3, [sp, #-4]! －－－－保存r0，注意有一个！，sp会加上4，这时候sp就指向栈顶的r0位置了
> 
>     mov    r3, lr －－－－保存svc mode的lr到r3  
>     stmia    r7, {r2 - r6} －－－－－－－－－压栈，在栈上形成形成struct pt_regs  
>     .endm

至此，在内核栈上保存了完整的硬件上下文。实际上不但完整，而且还有些冗余，因为其中有一个orig_r0的成员。所谓original r0就是发生中断那一刻的r0值，按理说，ARM_r0和ARM_ORIG_r0都应该是用户空间的那个r0。 为何要保存两个r0值呢？为何中断将-1保存到了ARM_ORIG_r0位置呢？理解这个问题需要跳脱中断处理这个主题，我们来看ARM的系统调用。对于系统调用，它 和中断处理虽然都是cpu异常处理范畴，但是一个明显的不同是系统调用需要传递参数，返回结果。如果进行这样的参数传递呢？对于ARM，当然是寄存器了， 特别是返回结果，保存在了r0中。对于ARM，r0～r7是各种cpu mode都相同的，用于传递参数还是很方便的。因此，进入系统调用的时候，在内核栈上保存了发生系统调用现场的所有寄存器，一方面保存了hardware context，另外一方面，也就是获取了系统调用的参数。返回的时候，将返回值放到r0就OK了。  
根据上面的描述，r0有两个作用，传递参数，返回结果。当把系统调用的结果放到r0的时候，通过r0传递的参数值就被覆盖了。本来，这也没有什么，但是有些场合是需要需要这两个值的：  
1、ptrace （和debugger相关，这里就不再详细描述了）  
2、system call restart （和signal相关，这里就不再详细描述了）  
正因为如此，硬件上下文的寄存器中r0有两份，ARM_r0是传递的参数，并复制一份到ARM_ORIG_r0，当系统调用返回的时候，ARM_r0是系统调用的返回值。  
OK，我们再回到中断这个主题，其实在中断处理过程中，没有使用ARM_ORIG_r0这个值，但是，为了防止system call restart，可以赋值为非系统调用号的值（例如-1）。

  

五、中断退出过程

无论是在内核态（包括系统调用和中断上下文）还是用户态，发生了中断后都会调用irq_handler进行处理，这里会调用对应的irq number的handler，处理softirq、tasklet、workqueue等（这些内容另开一个文档描述），但无论如何，最终都是要返回发生中断的现场。

1、中断发生在user mode下的退出过程，代码如下：

> ENTRY(ret_to_user_from_irq)  
>     ldr    r1, [tsk, #TI_FLAGS]  
>     tst    r1, #_TIF_WORK_MASK－－－－－－－－－－－－－－－A  
>     bne    work_pending  
> no_work_pending:  
>     asm_trace_hardirqs_on －－－－－－和irq flag trace相关，暂且略过
> 
>     /* perform architecture specific actions before user return */  
>     arch_ret_to_user r1, lr－－－－有些硬件平台需要在中断返回用户空间做一些特别处理  
>     ct_user_enter save = 0 －－－－和trace context相关，暂且略过
> 
>     restore_user_regs fast = 0, offset = 0－－－－－－－－－－－－B  
> ENDPROC(ret_to_user_from_irq)  
> ENDPROC(ret_to_user)

A：thread_info中的flags成员中有一些low level的标识，如果这些标识设定了就需要进行一些特别的处理，这里检测的flag主要包括：

> #define _TIF_WORK_MASK   (_TIF_NEED_RESCHED | _TIF_SIGPENDING | _TIF_NOTIFY_RESUME)

这三个flag分别表示是否需要调度、是否有信号处理、返回用户空间之前是否需要调用callback函数。只要有一个flag被设定了，程序就进入work_pending这个分支（work_pending函数需要传递三个参数，第三个是参数why是标识哪一个系统调用，当然，我们这里传递的是0）。

B：从字面的意思也可以看成，这部分的代码就是将进入中断的时候保存的现场（寄存器值）恢复到实际的ARM的各个寄存器中，从而完全返回到了中断发生的那一点。具体的代码如下：

>     .macro    restore_user_regs, fast = 0, offset = 0  
>     ldr    r1, [sp, #\offset + S_PSR] －－－－r1保存了pt_regs中的spsr，也就是发生中断时的CPSR  
>     ldr    lr, [sp, #\offset + S_PC]!    －－－－lr保存了PC值，同时sp移动到了pt_regs中PC的位置  
>     msr    spsr_cxsf, r1 －－－－－－－－－赋值给spsr，进行返回用户空间的准备  
>     clrex                    @ clear the exclusive monitor  
>   
>     .if    \fast  
>     ldmdb    sp, {r1 - lr}^            @ get calling r1 - lr  
>     .else  
>     ldmdb    sp, {r0 - lr}^ －－－－－－将保存在内核栈上的数据保存到用户态的r0～r14寄存器  
>     .endif  
>     mov    r0, r0   －－－－－－－－－NOP操作，ARMv5T之前的需要这个操作  
>     add    sp, sp, #S_FRAME_SIZE - S_PC－－－－现场已经恢复，移动svc mode的sp到原来的位置  
>     movs    pc, lr               －－－－－－－－返回用户空间  
>     .endm

2、中断发生在svc mode下的退出过程。具体代码如下：

>     .macro    svc_exit, rpsr, irq = 0  
>     .if    \irq != 0  
>     @ IRQs already off  
>     .else  
>     @ IRQs off again before pulling preserved data off the stack  
>     disable_irq_notrace  
>     .endif  
>     msr    spsr_cxsf, \rpsr－－－－－－－将中断现场的cpsr值保存到spsr中，准备返回中断发生的现场  
>   
>     ldmia    sp, {r0 - pc}^ －－－－－这条指令是ldm异常返回指令，这条指令除了字面上的操作，
> 
>                                        还包括了将spsr copy到cpsr中。  
>   
>     .endm

_原创文章，转发请注明出处。蜗窝科技_。[http://www.wowotech.net/irq_handler.html](http://www.wowotech.net/irq_handler.html)

  

附录

change log-2014-10-20，自己又重新阅读了一遍，做了一些修改，如下：

1、“ARM处理器有多种process mode”修改为“ARM处理器有多种processor mode”  
2、增加cpu_init的调用场景说明：  
    （1）bootstrap CPU initialize  
    （2）secondary CPUs initialize  
    （3）CPU resume from sleep  
3、增加对核心中断处理的描述  
4、增加对抢占相关的描述

  

change log-2014-11-20，根据zuoertu网友的提问，做了一些修改，如下：

1、增加对orig_r0的描述

2、增加对why的描述

标签: [irq](http://www.wowotech.net/tag/irq) [handler](http://www.wowotech.net/tag/handler) [中断处理](http://www.wowotech.net/tag/%E4%B8%AD%E6%96%AD%E5%A4%84%E7%90%86)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [傅立叶级数（Fourier Series）和周期现象](http://www.wowotech.net/basic_subject/Fourier_Series.html) | [linux内核中的GPIO系统之（2）：pin control subsystem](http://www.wowotech.net/gpio_subsystem/pin-control-subsystem.html)»

**评论：**

**[ecjtusbs](http://ecjtusbs/)**  
2022-12-14 14:59

2022年12月14日  
读了好几遍，每次感觉受益匪浅！  
前人栽树后人乘凉。感谢前辈的无私分享。

[回复](http://www.wowotech.net/irq_subsystem/irq_handler.html#comment-8722)

**明天**  
2022-05-18 17:09

有两种情况会恢复：1退出中断的时候，因为会恢复cpsr寄存器；2、软中断soft_irq中会打开中断

[回复](http://www.wowotech.net/irq_subsystem/irq_handler.html#comment-8616)

**Uchiha.tan**  
2020-11-16 03:57

“3、mask IRQ exception。也就是设定CPSR.I = 1”  
这里有错误吧？这个位设为1，就表示禁止了中断，那之后的中断岂不是就没法进行了？硬件应该没设定CPSR.I = 1

[回复](http://www.wowotech.net/irq_subsystem/irq_handler.html#comment-8138)

**fengeryi**  
2019-08-09 17:44

能讲这么清楚真的太难得了，非常感谢

[回复](http://www.wowotech.net/irq_subsystem/irq_handler.html#comment-7582)

**andreaji**  
2018-12-27 21:52

```  
.macro    svc_entry, stack_hole=0  
    sub    sp, sp, #(S_FRAME_SIZE + \stack_hole - 4)－－－－sp指向struct pt_regs中r1的位置  
    stmia    sp, {r1 - r12} －－－－－－寄存器入栈。  
  
    ldmia    r0, {r3 - r5}  
    add    r7, sp, #S_SP - 4 －－－－－－r7指向struct pt_regs中r12的位置  
    mov    r6, #-1 －－－－－－－－－－orig r0设为-1  
    add    r2, sp, #(S_FRAME_SIZE + \stack_hole - 4)－－－－r2是发现中断那一刻stack的现场  
    str    r3, [sp, #-4]! －－－－保存r0，注意有一个！，sp会加上4，这时候sp就指向栈顶的r0位置了  
  
    mov    r3, lr －－－－保存svc mode的lr到r3  
    stmia    r7, {r2 - r6} －－－－－－－－－压栈，在栈上形成形成struct pt_regs  
    .endm  
```  
-------------------------------------------------------------------------------  
linuxer 您好，我认为这里的 `add    r7, sp, #S_SP - 4` 应该是让 r7 指向 ARM_sp 。如果是指向 ARM_r12 , 应该是 add r7, offsetof(ARM_r11) 即可。 这里之所以是 add r7, offsetof(ARM_SP)-4 ,我个人理解是因为此前 sp_svc 指向 ARM_r1 , 在此基础上 + offsetof(ARM_SP) 会让 r7 指向到 ARM_lr ,所以需要 -4 以修复偏差。请您看看我的理解对否。

[回复](http://www.wowotech.net/irq_subsystem/irq_handler.html#comment-7111)

**zgq**  
2018-10-18 15:21

中断发生在user mode下的退出过程:  
add    sp, sp, #S_FRAME_SIZE - S_PC－－－－现场已经恢复，移动svc mode的sp到原来的位置  
--看了下代码，这个是thumb2指令集下恢复SP的代码，  
--ARM指令集恢复SP的代码为：  
add    sp, sp, #\offset + S_FRAME_SIZE

[回复](http://www.wowotech.net/irq_subsystem/irq_handler.html#comment-6991)

**太空的树懒**  
2018-08-04 11:27

蜗窝大侠,你好，请问一下：  
保存中断现场那一段里有一句：  
“一旦进入SVC mode，ARM处理器看到的寄存器已经发生变化，这里的sp已经变成了sp_svc了。因此，后续的压栈操作都是压入了发生中断那一刻的进程的（或者内核线程）内核栈（svc mode栈）” 这一句里怎么能看出来 sp_svc 指向的是 发生中断时的那个进程的内核栈呢

[回复](http://www.wowotech.net/irq_subsystem/irq_handler.html#comment-6866)

**明天**  
2022-04-29 14:16

@太空的树懒：讲的很清楚了  （4）很多异常处理的代码返回的时候都是使用了stack相关的操作，这里没有。“movs    pc, lr ”指令除了字面上意思（把lr的值付给pc），还有一个隐含的操作（movs中‘s’的含义）：把SPSR copy到CPSR，从而实现了模式的切换。 模式的切换是靠修改cpsr寄存器实现的

[回复](http://www.wowotech.net/irq_subsystem/irq_handler.html#comment-8609)

**[Physh](http://www.wowotech.net/)**  
2018-07-13 17:41

大侠，求教一个问题：我用perf top分析一个arm-linux进程，结果top symbols如下：  
4.27%  [vectors]             [.] 0x00000fc4                        
3.84%  [kernel]              [k] _raw_spin_unlock_irqrestore      
2.30%  [kernel]              [k] _raw_spin_unlock_irq              
1.94%  libc-2.20.so          [.] 0x0007c35c                        
1.91%  [vectors]             [.] 0x00000fd8                        
1.56%  libGLESv2.so.1.9.6.0  [.] 0x0003a5e0                        
1.34%  libGLESv2.so.1.9.6.0  [.] 0x0003a5cc                        
  
其中，第一行 [vectors] 这行尤其显得奇怪，这行有可能是代表irq handlers吗？我看到_raw_spin_unlock_irqrestore 占比很重，系统中应该是interrupt-heavy，但perf profile是基于PMU，ARM平台PMU应该还不是不可屏蔽中断，不太可能可以落到irq handlers内才对。  
  
疑惑不已，求教。

[回复](http://www.wowotech.net/irq_subsystem/irq_handler.html#comment-6846)

**[Physh](http://www.wowotech.net/)**  
2018-11-08 17:57

@Physh：研究了下，已经理清这个东西，希望有助后者：  
  
4.27%  [vectors]             [.] 0x00000fc4                        
3.84%  [kernel]              [k] _raw_spin_unlock_irqrestore      
2.30%  [kernel]              [k] _raw_spin_unlock_irq              
1.94%  libc-2.20.so          [.] 0x0007c35c                        
1.91%  [vectors]             [.] 0x00000fd8                        
1.56%  libGLESv2.so.1.9.6.0  [.] 0x0003a5e0                        
1.34%  libGLESv2.so.1.9.6.0  [.] 0x0003a5cc        
  
上文中，4.27%  [vectors]             [.] 0x00000fc4 确实表明这个地址位于进程空间，是[verctors]中偏移量为0x00000fc4的func，这个地方是用户kuser_helper，一般是用来实现原子操作等用户空间无法实现需要内核提供帮助实现的函数，在本例中0x00000fc4 是kuser_cmpxchg64。

[回复](http://www.wowotech.net/irq_subsystem/irq_handler.html#comment-7020)

**[passerby](http://www.wowotech.net/)**  
2018-03-26 12:40

在arm64的do_signal()中，如果系统调用需要restart，则要修改PC值。  
但是这里restart_addr = continue_addr - (compat_thumb_mode(regs) ? 2 : 4); 依然与arm32一样  
按道理应该是-4或者-8才对呢

[回复](http://www.wowotech.net/irq_subsystem/irq_handler.html#comment-6622)

**风露**  
2018-01-06 01:59

蜗窝大神，求教一个问题：  
进入IRQ mode后，中断会被自动禁掉，正常情况下，什么时候会再开启中断？

[回复](http://www.wowotech.net/irq_subsystem/irq_handler.html#comment-6457)

1 [2](http://www.wowotech.net/irq_subsystem/irq_handler.html/comment-page-2#comments) [3](http://www.wowotech.net/irq_subsystem/irq_handler.html/comment-page-3#comments) [4](http://www.wowotech.net/irq_subsystem/irq_handler.html/comment-page-4#comments)

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
    
    - Shiina  
        [一个电路（circuit）中，由于是回路，所以用电势差的概念...](http://www.wowotech.net/basic_subject/voltage.html#8926)
    - Shiina  
        [其中比较关键的点是相对位置概念和点电荷的静电势能计算。](http://www.wowotech.net/basic_subject/voltage.html#8925)
    - leelockhey  
        [你这是哪个内核版本](http://www.wowotech.net/pm_subsystem/generic_pm_architecture.html#8924)
    - ja  
        [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
    - 元神高手  
        [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
    - 十七  
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
    
    - [文件系统和裸块设备的page cache问题](http://www.wowotech.net/filesystem/439.html)
    - [小printf大作用（用日志打印的方式调试程序）](http://www.wowotech.net/soft/7.html)
    - [X-026-KERNEL-Linux gpio driver的移植之gpio range](http://www.wowotech.net/x_project/kernel_gpio_driver_porting_2.html)
    - [计算机科学基础知识（一）:The Memory Hierarchy](http://www.wowotech.net/basic_subject/memory-hierarchy.html)
    - [以太网驱动的流程浅析(二)-Ifconfig的详细代码流程](http://www.wowotech.net/linux_kenrel/466.html)
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
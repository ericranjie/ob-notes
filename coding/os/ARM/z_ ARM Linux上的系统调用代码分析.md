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

作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2017-2-20 18:54 分类：[进程管理](http://www.wowotech.net/sort/process_management)

一、前言

当用户空间的程序调用swi指令发起内核服务请求的时候，实际上程序其实是完成了一次“穿越”，该进程从用户态穿越到了内核态。这个过程有点象周末你在家里看片，突然有些内急，随手按下了pause按键，电影里面的世界嘎然而止了。程序世界亦然，一个swi后，用户空间的代码执行暂停了、stack（用户栈）上的数据，正文段、静态数据区、heap去的数据……一切都停下来了，程序的执行突然就转入另外一个世界，使用的栈变成了内核栈、正在执行的正文段程序变成vector_swi开始的binary code、与之匹配数据区也变化了……

一切是怎么发生的呢？CPU只有一套而已，这里硬件做了哪些动作？软件又搞了什么鬼？穿越到另外的世界当然有趣，但是如何找到回来的路？这一切疑问希望能在这样的一篇文档中讲述清楚。

本文的代码来自4.4.6内核，用ARM处理器为例子描述。

二、构建内核栈上的用户现场

代码如下（忽略Cortex-M处理器的代码，忽略THUMB指令集的代码）：

> ENTRY(vector_swi)\
> sub    sp, sp, #S_FRAME_SIZE\
> stmia    sp, {r0 - r12}            @ Calling r0 - r12\
> ARM(    add    r8, sp, #S_PC        )\
> ARM(    stmdb    r8, {sp, lr}^        )    @ Calling sp, lr\
> mrs    r8, spsr            @ called from non-FIQ mode, so ok.\
> str    lr, \[sp, #S_PC\]            @ Save calling PC\
> str    r8, \[sp, #S_PSR\]        @ Save CPSR\
> str    r0, \[sp, #S_OLD_R0\]        @ Save OLD_R0

当执行vector_swi的时候，硬件已经做了不少的事情，包括：

（1）将CPSR寄存器保存到SPSR_svc寄存器中，将返回地址（用户空间执行swi指令的下一条指令）保存在lr_svc中

（2）设定CPSR寄存器的值。具体包括：CPSR.M = '10011'（svc mode），CPSR.I = '1'（disable IRQ），CPSR.IT = '00000000'（TODO），CPSR.J = '0'（），CPSR.T = SCTLR.TE（J和Tbit和Instruction set state有关，和本文无关），CPSR.E = SCTLR.EE（字节序定义，和本文无关）。

（3）PC设定为swi异常向量的地址

随后的行为都是软件行为了，因为代码中涉及压栈动作，所以首先要确定的就是当前在哪里这个问题。sp_svc早在进程切换的时候就已经设定好了，就是该进程的内核栈。

[![未命名](http://www.wowotech.net/content/uploadfile/201702/8991a20bccab1926173bf8d0cb89bdeb20170220105413.gif "未命名")](http://www.wowotech.net/content/uploadfile/201702/575e0854debf480cdbe72e46c1ed798e20170220105412.gif)

当Task A切换到Task B的时候，有一个很重要的步骤就是HW context的切换，由于Task A和Task B都在同一个CPU上运行，因此需要把当前CPU的各种寄存器以及状态信息保存在一块memory data block中（也就是硬件上下文了），并且用Task B的硬件上下文的数值来加载CPU，这里面就包括sp_svc。在内核态，完成进程切换后，最终会返回task B的用户空间执行，但是这时候Task B对应的内核栈（sp_svc0）是确定的了。

当通过系统调用进入内核的时候，内核栈已经是准备好了，不过这时候内核栈上是空的，执行完上述代码之后，在内核栈上形成如下的用户空间现场：

[![systemcall](http://www.wowotech.net/content/uploadfile/201702/fa2dc3f37b146150f55d11a02958133f20170220105416.gif "systemcall")](http://www.wowotech.net/content/uploadfile/201702/05d69dcd09b64cf10ad06973533e6a1720170220105414.gif)

代码我们就不走读了，很简单，大家可自行阅读即可。顺便一提的是：你看到这个保存的现场是不是觉得很熟悉？可以看看[ARM中断处理](http://www.wowotech.net/irq_subsystem/irq_handler.html)这篇文档，中断保存的现场和系统调用是一样的。另外，保存在内核栈上的用户空间现场并不是全部的HW Context，HW context是一段内存中的数据，保存了某个时刻CPU的全部状态，不仅仅是core register，还有很多CPU内部其他的HW block状态，例如FPU的寄存器和状态。这时候，问题来了，在通过系统调用进入内核态的时候，仅仅保存core register够不够？够不够是和系统调用接口的约定相关，实际上，对于linux，我们约定如下：内核态的代码是不允许执行浮点操作指令的（这里用FPU例子，其他类似），如果一定要这样的话，那么需要在内核使用FPU的代码前后增加FPU上下文的保存和恢复的代码，确保在返回用户空间的时候，FPU的上下文是保持不变的。

最后一个有趣的问题是：为何r0被两次压栈？一个是r0，另外一个是old r0。其实在系统调用过程中，r0有两个角色，一个是传递参数，另外一个是返回值。刚进入系统调用现场的时候，old r0和r0其实都是保存了本次系统调用的参数，而在完成系统调用之后，r0保存了返回用户空间的return value。不过你可能觉得用一个r0就OK了，具体为何如此我们后面还会进行描述。

三、几个简单的初始化操作

代码如下：

> zero_fp\
> alignment_trap r10, ip, \_\_cr_alignment\
> enable_irq\
> ct_user_exit\
> get_thread_info tsk

zero_fp用来清除frame pointer，在debugger做栈的回溯的时候，当fp等于0的时候也就意味着到了最外层函数。对于kernel而言，来到这里，函数的调用跟踪就结束了，我们不可能一直回溯到用户空间的函数调用。上一节，我们说过了，硬件会关闭irq，在这里，我们通过enable_irq开启本cpu的中断处理。ct_user_exit和Context tracking subsystem相关的内容，这里就不深入了，关于对齐，可以多聊几句。ARM64的硬件是支持非对齐操作的，当然仅仅限于对normal memory的访问（对memory order没有要求的那些访问，例如exclusive load/store和load-acquire 或者 store-release 指令就不支持了）。由于取指而产生的内存访问或者是访问device type的memory都是必须要对齐的。当指令是非对齐的访问的时候，可以有两个选择（SCTLR_ELx.A控制）：一个是产生fault，另外一个是执行非对齐访问（由硬件完成）。对内存的非对齐的访问在总线上被分解成两个transaction。所有的ARMv8的处理器的硬件都支持非对齐访问，因此，在ARM64应该不需要软件来实现非对齐的访问。

支持ARMv8的处理器当然不需要考虑对齐问题，不过对于ARM processor，有些硬件是不支持非对齐的访问的，这时候，内核配置（CONFIG_ALIGNMENT_TRAP）可以用软件的方法来实现非对齐访问（这是在硬件不支持的情况下的无奈之举），但是对性能的杀伤力极大，不到万不得已不能打开。具体代码很简单，这里就不说明了。

三、如何获取系统调用号？

系统调用有两种规范，一种是老的OABI（系统调用号来自swi指令中），另外一种是ARM ABI，也就是EABI（系统调用号来自r7）。如果想要兼容旧的OABI，那么我们需要定义OABI_COMPAT，这会带来一点系统调用的开销，同时让内核变大一点，对应的好处是使用旧的OABI规格的用户程序也可以运行在内核之上。当然，如果我们确定用户空间只是服从EABI规范，那么可以考虑不定义CONFIG_OABI_COMPAT。

相关的代码如下：

> #if defined(CONFIG_OABI_COMPAT)
>
> USER(    ldr    r10, \[lr, #-4\]        )    @ get SWI instruction\
> ARM_BE8(rev    r10, r10)            @ little endian instruction
>
> #elif defined(CONFIG_AEABI)
>
> #else\
> /\* Legacy ABI only. \*/\
> USER(    ldr    scno, \[lr, #-4\]        )    @ get SWI instruction\
> #endif

如果是符合EABI规范，那么直接从r7中获取系统调用号即可，不需要特别的代码，因此CONFIG_AEABI的情况下，代码是空的。如果是老的规范，那么我们需要从SWI指令那里获取系统调用号，这时候，我们需要lr（实际上就是lr_svc，该寄存器保存了swi指令的下一条指令）来找到swi那一条指令码。

> uaccess_disable tbl
>
> adr    tbl, sys_call_table        @ load syscall table pointer
>
> #if defined(CONFIG_OABI_COMPAT)\
> bics    r10, r10, #0xff000000\
> eorne    scno, r10, #\_\_NR_OABI_SYSCALL_BASE\
> ldrne    tbl, =sys_oabi_call_table\
> #elif !defined(CONFIG_AEABI)\
> bic    scno, scno, #0xff000000        @ mask off SWI op-code\
> eor    scno, scno, #\_\_NR_SYSCALL_BASE    @ check OS number\
> #endif

取出swi指令中的低24bit就可以得出系统调用号，当然，对于EABI标准，我们使用r7传递系统调用号，因此陷入内核的时候永远使用“swi 0”这样的方式，因此，如果swi指令中的低24bit是0，则说明是服从EABI规范。

执行完上面的代码后，r7（scno）中保存了系统调用号，r8（tbl）中是syscall table pointer，通过r7和r8的值，我们已经知道后续的路该如何走了。

四、参数传递

使用swi指令的代码位于glibc中，我们可以大概把代码认为是如下的格式：

> ……
>
> return value = swi( 参数1，参数2，……);
>
> ……

从这个角度看，系统调用和一个普通的c程序调用是类似的，都有参数和返回值的概念。当然，由于模式也发生了切换，因此这里的参数传递不能使用stack压栈的方式（swi产生了stack的切换），只能使用寄存器的方式。

对于ARM处理器，标准过程调用约定使用r0～r3来传递参数，其余的参数压入栈中。经过前面两个小节的描述，我们已经找到系统调用号和系统调用表，下面准备调用内核的系统调用函数，对于内核态的系统调用函数，其格式如下：

> ……
>
> return value = sys_xxx( 参数1，参数2，……);
>
> ……

因此，我们还需要点代码来过渡到sys_xxx，如下：

> local_restart:\
> ldr    r10, \[tsk, #TI_FLAGS\]        @ check for syscall tracing\
> stmdb    sp!, {r4, r5}            @ push fifth and sixth args
>
> tst    r10, #\_TIF_SYSCALL_WORK        @ are we tracing syscalls?\
> bne    \_\_sys_trace
>
> cmp    scno, #NR_syscalls        @ check upper syscall limit\
> badr    lr, ret_fast_syscall        @ return address\
> ldrcc    pc, \[tbl, scno, lsl #2\]        @ call sys\_\* routine

我们这里需要模拟一个c函数调用，因此需要在栈上压入系统调用可能存在的第五和第六个参数（有些系统调用超过4个参数，他们使用r0～r5在swi接口上传递参数）。如果参数OK的话，那么ldrcc    pc, \[tbl, scno, lsl #2\]代码将直接把控制权交给对应的sys_xxx函数。需要注意的是返回地址的设定，我们无法使用bl这样的汇编指令，因此只能是手动设定lr寄存器了（badr    lr, ret_fast_syscall ）。

五、返回用户空间

在返回用户空间之前会处理很多的事情，例如信号处理、进程调度等，这是通过检查struct thread_info中的flag标记来完成的，代码如下：

> disable_irq_notrace            @ disable interrupts\
> ldr    r1, \[tsk, #TI_FLAGS\]        @ re-check for syscall tracing\
> tst    r1, #\_TIF_SYSCALL_WORK | \_TIF_WORK_MASK\
> bne    fast_work_pending
>
> restore_user_regs fast = 1, offset = S_OFF
>
> fast_work_pending:\
> str    r0, \[sp, #S_R0+S_OFF\]!        @ returned r0
>
> ……

这里面最著名的flag就是_TIF_NEED_RESCHED，有了这个flag，说明有调度需求。由此可知在系统调用返回用户空间的时候上有一个调度点。其他的flag和我们这里的场景无关，这里就不描述了，总而言之，如果需要有其他额外的事情要处理，我们需要跳转到fast_work_pending ，否则调用restore_user_regs返回用户空间现场。这里有一个小小的细节：如果需要有额外的事项处理（例如有pending signal），那么r0寄存器实际上会被破坏掉，也就破坏了sys_xxx函数的返回值，这时候，我们把r0保存到了用户现场（pt_regs）中的S_R0的位置，这也是为何pt_regs有S_R0和S_OLD_R0两个和r0相关的域。

恢复用户空间的代码（restore_user_regs ）如下：

> mov    r2, sp\
> ldr    r1, \[r2, #\\offset + S_PSR\]    @ get calling cpsr\
> ldr    lr, \[r2, #\\offset + S_PC\]!    @ get pc\
> msr    spsr_cxsf, r1            @ save in spsr_svc\
> .if    \\fast\
> ldmdb    r2, {r1 - lr}^            @ get calling r1 - lr\
> .else\
> ldmdb    r2, {r0 - lr}^            @ get calling r0 - lr\
> .endif\
> mov    r0, r0                @ ARMv5T and earlier require a nop\
> @ after ldm {}^\
> add    sp, sp, #\\offset + S_FRAME_SIZE\
> movs    pc, lr                @ return & move spsr_svc into cpsr

整个代码比较简单，就是用进入系统调用时候压入内核栈的值来进行用户现场的恢复，其中一个细节是内核栈的操作，在调用movs    pc, lr 返回用户空间现场之前，add    sp, sp, #\\offset + S_FRAME_SIZE指令确保用户栈上是空的。此外，我们需要考虑返回用户空间时候的r0设置问题，毕竟它承载了本次系统调用的返回值，这时候的r0有两种情况：

（1）在没有pending work的情况下（fast等于1），r0保存了sys_xxx函数的返回值

（2）在有pending work的情况下（fast等于0），struct pt_regs（返回用户空间的现场）中的r0保存了sys_xxx函数的返回值

restore_user_regs还有一个参数叫做offset，我们知道，在进入系统调用的时候，我们把参数5和参数6压入栈上，因此产生了到pt_regs 8个字节的偏移，这里需要补偿回来。

_原创文章，转发请注明出处。蜗窝科技_

标签: [ARM](http://www.wowotech.net/tag/ARM) [系统调用](http://www.wowotech.net/tag/%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Linux系统如何标识进程？](http://www.wowotech.net/process_management/pid.html) | [进程切换分析（2）：TLB处理](http://www.wowotech.net/process_management/context-switch-tlb.html)»

**评论：**

**zhangze**\
2021-09-11 17:35

add    sp, sp, #\\offset + S_FRAME_SIZE指令确保用户栈上是空的\
这里应该是确保内核栈为空，应该是笔误了

[回复](http://www.wowotech.net/process_management/syscall-arm.html#comment-8310)

**xavier**\
2019-04-27 11:37

wowo老师,请教个问题,我在移植一个驱动,设备的初始化我放在ioctl中,我这个初始化大约得持续几十ms,都是SPI的异步操作,我发现在内核态执行完设备初始化操作后无法返回调用ioctl的用户空间,这个进程被阻塞了,可是如果我减少写入命令,比如只有一条SPI写入命令,则系统调用后可以返回用户空间,请问这可能是什么原因造成的,是否由解决方案呢?

[回复](http://www.wowotech.net/process_management/syscall-arm.html#comment-7376)

**yahui**\
2018-11-21 14:06

wowo你好，有一个细节不太清楚，请问arm linux上从用户态进入内核态时，怎样获取内核态堆栈的地址？谢谢

[回复](http://www.wowotech.net/process_management/syscall-arm.html#comment-7046)

**[linuxer](http://www.wowotech.net/)**\
2018-11-22 09:07

@yahui：不需要获取，系统调用会导致HW context切换，sp指针自然切换到内核栈。

具体设定sp_el1（内核栈指针）的代码位置是在fork的时候。

[回复](http://www.wowotech.net/process_management/syscall-arm.html#comment-7049)

**dyh**\
2022-02-11 16:13

@linuxer：内核栈空间是8K，这8K的空间在哪申请的？

[回复](http://www.wowotech.net/process_management/syscall-arm.html#comment-8545)

**dyh**\
2022-02-12 11:23

@linuxer：随便取8K的空间，怎样避免出现踩内存问题

[回复](http://www.wowotech.net/process_management/syscall-arm.html#comment-8551)

**sccpecsky**\
2017-11-28 10:37

WOWO老师，我想请教一下，linux的系统调用过程中，是否是可调度的？\
系统调用的过程虽然属于内核态，但是却是进程上下文，那么在这个过程中是否可以被调度\
比如时间片耗尽的情况下，切换去执行其他进程

[回复](http://www.wowotech.net/process_management/syscall-arm.html#comment-6247)

**[bsp](http://www.wowotech.net/)**\
2017-11-28 12:56

@sccpecsky：linux的系统调用过程中，是可调度的。\
比如nanosleep/wait等系统调用会主动在内核态schedule_out；\
Linux的系统调用过程中，也是可抢占的。\
从代码上，并没有看到vector_swi有 preempt_disable 或类似的调用；\
从逻辑上，进程通过系统调用进入内核态也是进程一部分，运行时间也会被计入其vruntime中；如果系统调用长期running占据cpu，这是不合理的。进程内核态 会被中断打断，中断退出时会检查_TIF_NEED_RESCHED，满足条件时 当前进程就会被抢占。

[回复](http://www.wowotech.net/process_management/syscall-arm.html#comment-6248)

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

  - [X-006-UBOOT-pinctrl driver移植(Bubblegum-96平台)](http://www.wowotech.net/x_project/bubblegum_uboot_pinctrl.html)
  - [Concurrency Managed Workqueue之（四）：workqueue如何处理work](http://www.wowotech.net/irq_subsystem/queue_and_handle_work.html)
  - [关于java单线程经常占用cpu100%分析](http://www.wowotech.net/linux_kenrel/483.html)
  - [RCU（1）- 概述](http://www.wowotech.net/kernel_synchronization/461.html)
  - [蓝牙协议分析(5)\_BLE广播通信相关的技术分析](http://www.wowotech.net/bluetooth/ble_broadcast.html)

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

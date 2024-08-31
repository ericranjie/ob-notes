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

作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2014-8-28 20:00 分类：[中断子系统](http://www.wowotech.net/sort/irq_subsystem)

一、前言

当外设触发一次中断后，一个大概的处理过程是：

1、具体CPU architecture相关的模块会进行现场保护，然后调用machine driver对应的中断处理handler

2、machine driver对应的中断处理handler中会根据硬件的信息获取HW interrupt ID，并且通过irq domain模块翻译成IRQ number

3、调用该IRQ number对应的high level irq event handler，在这个high level的handler中，会通过和interupt controller交互，进行中断处理的flow control（处理中断的嵌套、抢占等），当然最终会遍历该中断描述符的IRQ action list，调用外设的specific handler来处理该中断

4、具体CPU architecture相关的模块会进行现场恢复。

上面的1、4这两个步骤在[linux kernel的中断子系统之（六）：ARM中断处理过程](http://www.wowotech.net/linux_kenrel/irq_handler.html)中已经有了较为细致的描述，步骤2在[linux kernel的中断子系统之（二）：irq domain介绍](http://www.wowotech.net/linux_kenrel/irq-domain.html)中介绍，本文主要描述步骤3，也就是linux中断子系统的high level irq event handler。

注：这份文档充满了猜测和空想，很多地方描述可能是有问题的，不过我还是把它发出来，抛砖引玉，希望可以引发大家讨论。

一、如何进入high level irq event handler

1、从具体CPU architecture的中断处理到machine相关的处理模块

说到具体的CPU，我们还是用ARM为例好了。对于ARM，我们在[ARM中断处理](http://www.wowotech.net/linux_kenrel/irq_handler.html)文档中已经有了较为细致的描述。这里我们看看如何从从具体CPU的中断处理到machine相关的处理模块 ，其具体代码如下：

>     .macro    irq_handler  
> #ifdef CONFIG_MULTI_IRQ_HANDLER  
>     ldr    r1, =handle_arch_irq  
>     mov    r0, sp  
>     adr    lr, BSYM(9997f)  
>     ldr    pc, [r1]  
> #else  
>     arch_irq_handler_default  
> #endif  
> 9997:  
>     .endm

其实，直接从CPU的中断处理跳转到通用中断处理模块是不可能的，中断处理不可能越过interrupt controller这个层次。一般而言，通用中断处理模块会提供一些通用的中断代码处理库，然后由interrupt controller这个层次的代码调用这些通用中断处理的完成整个的中断处理过程。“interrupt controller这个层次的代码”是和硬件中断系统设计相关的，例如：系统中有多少个interrupt contrller，每个interrupt controller是如何控制的？它们是如何级联的？我们称这些相关的驱动模块为machine interrupt driver。

在上面的代码中，如果配置了MULTI_IRQ_HANDLER的话，ARM中断处理则直接跳转到一个叫做handle_arch_irq函数，如果系统中只有一个类型的interrupt controller（可能是多个interrupt controller，例如使用两个级联的GIC），那么handle_arch_irq可以在interrupt controller初始化的时候设定。代码如下：

> ……
> 
> if (gic_nr == 0) {  
>         set_handle_irq(gic_handle_irq);  
> }
> 
> ……

gic_nr是GIC的编号，linux kernel初始化过程中，每发现一个GIC，都是会指向GIC driver的初始化函数的，不过对于第一个GIC，gic_nr等于0，对于第二个GIC，gic_nr等于1。当然handle_arch_irq这个函数指针不是per CPU的变量，是全部CPU共享的，因此，初始化一次就OK了。

当使用多种类型的interrupt controller的时候（例如HW 系统使用了S3C2451这样的SOC，这时候，系统有两种interrupt controller，一种是GPIO type，另外一种是SOC上的interrupt controller），则不适合在interrupt controller中进行设定，这时候，可以考虑在machine driver中设定。在这种情况下，handle_arch_irq 这个函数是在setup_arch函数中根据machine driver设定，具体如下：

> handle_arch_irq = mdesc->handle_irq;

关于MULTI_IRQ_HANDLER这个配置项，我们可以再多说几句。当然，其实这个配置项的名字已经出卖它了。multi irq handler就是说系统中有多个irq handler，可以在run time的时候指定。为何要run time的时候，从多个handler中选择一个呢？HW interrupt block难道不是固定的吗？我的理解（猜想）是：一个kernel的image支持多个HW platform，对于不同的HW platform，在运行时检查HW platform的类型，设定不同的irq handler。

2、interrupt controller相关的代码

我们还是以2个级联的GIC为例来描述interrupt controller相关的代码。代码如下：

> static asmlinkage void __exception_irq_entry gic_handle_irq(struct pt_regs *regs)  
> {  
>     u32 irqstat, irqnr;  
>     struct gic_chip_data *gic = &gic_data[0];－－－－－获取root GIC的硬件描述符  
>     void __iomem *cpu_base = gic_data_cpu_base(gic); 获取root GIC mapping到CPU地址空间的信息
> 
>     do {  
>         irqstat = readl_relaxed(cpu_base + GIC_CPU_INTACK);－－－获取HW interrupt ID  
>         irqnr = irqstat & ~0x1c00;
> 
>         if (likely(irqnr > 15 && irqnr < 1021)) {－－－－SPI和PPI的处理  
>             irqnr = irq_find_mapping(gic->domain, irqnr);－－－将HW interrupt ID转成IRQ number  
>             handle_IRQ(irqnr, regs);－－－－处理该IRQ number  
>             continue;  
>         }  
>         if (irqnr < 16) {－－－－－IPI类型的中断处理  
>             writel_relaxed(irqstat, cpu_base + GIC_CPU_EOI);  
> #ifdef CONFIG_SMP  
>             handle_IPI(irqnr, regs);  
> #endif  
>             continue;  
>         }  
>         break;  
>     } while (1);  
> }

更多关于GIC相关的信息，请参考linux kernel的中断子系统之（七）：GIC代码分析。对于ARM处理器，handle_IRQ代码如下：

> void handle_IRQ(unsigned int irq, struct pt_regs *regs)  
> {
> 
> ……  
>         generic_handle_irq(irq);
> 
> ……  
> }

3、调用high level handler

调用high level handler的代码逻辑非常简单，如下：

> int generic_handle_irq(unsigned int irq)  
> {  
>     struct irq_desc *desc = irq_to_desc(irq); －－－通过IRQ number获取该irq的描述符
> 
>     if (!desc)  
>         return -EINVAL;  
>     generic_handle_irq_desc(irq, desc);－－－－调用high level的irq handler来处理该IRQ  
>     return 0;  
> }
> 
> static inline void generic_handle_irq_desc(unsigned int irq, struct irq_desc *desc)  
> {  
>     desc->handle_irq(irq, desc);  
> }

二、理解high level irq event handler需要的知识准备

1、自动探测IRQ

一个硬件驱动可以通过下面的方法进行自动探测它使用的IRQ：

> unsigned long irqs;  
> int irq;
> 
> irqs = probe_irq_on();－－－－－－－－启动IRQ自动探测  
> 驱动那个打算自动探测IRQ的硬件产生中断  
> irq = probe_irq_off(irqs);－－－－－－－结束IRQ自动探测

如果能够自动探测到IRQ，上面程序中的irq(probe_irq_off的返回值)就是自动探测的结果。后续程序可以通过request_threaded_irq申请该IRQ。probe_irq_on函数主要的目的是返回一个32 bit的掩码，通过该掩码可以知道可能使用的IRQ number有哪些，具体代码如下：

> unsigned long probe_irq_on(void)  
> {
> 
> ……  
>     for_each_irq_desc_reverse(i, desc) { －－－－scan 从nr_irqs-1 到0 的中断描述符  
>         raw_spin_lock_irq(&desc->lock);  
>         if (!desc->action && irq_settings_can_probe(desc)) {－－－－－－－－（1）  
>             desc->istate |= IRQS_AUTODETECT | IRQS_WAITING;－－－－－（2）  
>             if (irq_startup(desc, false))  
>                 desc->istate |= IRQS_PENDING;  
>         }  
>         raw_spin_unlock_irq(&desc->lock);  
>     }   
>     msleep(100); －－－－－－－－－－－－－－－－－－－－－－－－－－（3）
> 
>   
>     for_each_irq_desc(i, desc) {  
>         raw_spin_lock_irq(&desc->lock);
> 
>         if (desc->istate & IRQS_AUTODETECT) {－－－－－－－－－－－－（4）  
>             if (!(desc->istate & IRQS_WAITING)) {  
>                 desc->istate &= ~IRQS_AUTODETECT;  
>                 irq_shutdown(desc);  
>             } else  
>                 if (i < 32)－－－－－－－－－－－－－－－－－－－－－－－－（5）  
>                     mask |= 1 << i;  
>         }  
>         raw_spin_unlock_irq(&desc->lock);  
>     }
> 
>     return mask;  
> }

（1）那些能自动探测IRQ的中断描述符需要具体两个条件：

a、该中断描述符还没有通过request_threaded_irq或者其他方式申请该IRQ的specific handler（也就是irqaction数据结构）

b、该中断描述符允许自动探测（不能设定IRQ_NOPROBE）

（2）如果满足上面的条件，那么该中断描述符属于备选描述符。设定其internal state为IRQS_AUTODETECT | IRQS_WAITING。IRQS_AUTODETECT表示本IRQ正处于自动探测中。

（3）在等待过程中，系统仍然允许，各种中断依然会触发。在各种high level irq event handler中，总会有如下的代码：

> desc->istate &= ~(IRQS_REPLAY | IRQS_WAITING);

这里会清除IRQS_WAITING状态。

（4）这时候，我们还没有控制那个想要自动探测IRQ的硬件产生中断，因此处于自动探测中，并且IRQS_WAITING并清除的一定不是我们期待的IRQ（可能是spurious interrupts导致的），这时候，clear IRQS_AUTODETECT，shutdown该IRQ。

（5）最大探测的IRQ是31（mask是一个32 bit的value），mask返回的是可能的irq掩码。

我们再来看看probe_irq_off的代码：

> int probe_irq_off(unsigned long val)  
> {  
>     int i, irq_found = 0, nr_of_irqs = 0;  
>     struct irq_desc *desc;
> 
>     for_each_irq_desc(i, desc) {  
>         raw_spin_lock_irq(&desc->lock);
> 
>         if (desc->istate & IRQS_AUTODETECT) {－－－－只有处于IRQ自动探测中的描述符才会被处理  
>             if (!(desc->istate & IRQS_WAITING)) {－－－－找到一个潜在的中断描述符  
>                 if (!nr_of_irqs)  
>                     irq_found = i;  
>                 nr_of_irqs++;  
>             }  
>             desc->istate &= ~IRQS_AUTODETECT; －－－－IRQS_WAITING没有被清除，说明该描述符  
>             irq_shutdown(desc);                                     不是自动探测的那个，shutdown之  
>         }  
>         raw_spin_unlock_irq(&desc->lock);  
>     }  
>     mutex_unlock(&probing_active);
> 
>     if (nr_of_irqs > 1) －－－－－－如果找到多于1个的IRQ，说明探测失败，返回负的IRQ个数信息  
>         irq_found = -irq_found;
> 
>     return irq_found;  
> }

因为在调用probe_irq_off已经触发了自动探测IRQ的那个硬件中断，因此在该中断的high level handler的执行过程中，该硬件对应的中断描述符的IRQS_WAITING标致应该已经被清除，因此probe_irq_off函数scan中断描述符DB，找到处于auto probe中，而且IRQS_WAITING标致被清除的那个IRQ。如果找到一个，那么探测OK，返回该IRQ number，如果找到多个，说明探测失败，返回负的IRQ个数信息，没有找到的话，返回0。

  
2、resend一个中断

一个ARM SOC总是有很多的GPIO，有些GPIO可以提供中断功能，这些GPIO的中断可以配置成level trigger或者edge trigger。一般而言，大家都更喜欢用level trigger的中断。有的SOC只能是有限个数的GPIO可以配置成电平中断，因此，在项目初期进行pin define的时候，大家都在争抢电平触发的GPIO。

电平触发的中断有什么好处呢？电平触发的中断很简单、直接，只要硬件检测到硬件事件（例如有数据到来），其assert指定的电平信号，CPU ack该中断后，电平信号消失。但是对于边缘触发的中断，它是用一个上升沿或者下降沿告知硬件的状态，这个状态不是一个持续的状态，如果软件处理不好，容易丢失中断。

什么时候会resend一个中断呢？我们考虑一个简单的例子：

（1）CPU A上正在处理x外设的中断

（2）x外设的中断再次到来（CPU A已经ack该IRQ，因此x外设的中断可以再次触发），这时候其他CPU会处理它（mask and ack），并设置该中断描述符是pending状态，并委托CPU A处理该pending状态的中断。需要注意的是CPU已经ack了该中断，因此该中断的硬件状态已经不是pending状态，无法触发中断了，这里的pending状态是指中断描述符的软件状态。

（3）CPU B上由于同步的需求，disable了x外设的IRQ，这时候，CPU A没有处理pending状态的x外设中断就离开了中断处理过程。

（4）当enable x外设的IRQ的时候，需要检测pending状态以便resend该中断，否则，该中断会丢失的

具体代码如下：

> void check_irq_resend(struct irq_desc *desc, unsigned int irq)  
> {  
>   
>     if (irq_settings_is_level(desc)) {－－－－－－－电平中断不存在resend的问题  
>         desc->istate &= ~IRQS_PENDING;  
>         return;  
>     }  
>     if (desc->istate & IRQS_REPLAY)－－－－如果已经设定resend的flag，退出就OK了，这个应该  
>         return;                                                和irq的enable disable能多层嵌套相关  
>     if (desc->istate & IRQS_PENDING) {－－－－－－－如果有pending的flag则进行处理  
>         desc->istate &= ~IRQS_PENDING;  
>         desc->istate |= IRQS_REPLAY; －－－－－－设置retrigger标志
> 
>         if (!desc->irq_data.chip->irq_retrigger ||  
>             !desc->irq_data.chip->irq_retrigger(&desc->irq_data)) {－－－－调用底层irq chip的callback  
> #ifdef CONFIG_HARDIRQS_SW_RESEND  
> 也可以使用软件手段来完成resend一个中断，具体代码省略，有兴趣大家可以自己看看  
> #endif  
>         }  
>     }  
> }

在各种high level irq event handler中，总会有如下的代码：

> desc->istate &= ~(IRQS_REPLAY | IRQS_WAITING);

这里会清除IRQS_REPLAY状态，表示该中断已经被retrigger，一次resend interrupt的过程结束。

3、unhandled interrupt和spurious interrupt

在中断处理的最后，总会有一段代码如下：

> irqreturn_t  
> handle_irq_event_percpu(struct irq_desc *desc, struct irqaction *action)  
> {
> 
> ……
> 
>     if (!noirqdebug)  
>         note_interrupt(irq, desc, retval);  
>     return retval;  
> }

note_interrupt就是进行unhandled interrupt和spurious interrupt处理的。对于这类中断，linux kernel有一套复杂的机制来处理，你可以通过command line参数（noirqdebug）来控制开关该功能。

当发生了一个中断，但是没有被处理（有两种可能，一种是根本没有注册的specific handler，第二种是有handler，但是handler否认是自己对应的设备触发的中断），怎么办？毫无疑问这是一个异常状况，那么kernel是否要立刻采取措施将该IRQ disable呢？也不太合适，毕竟interrupt request信号线是允许共享的，直接disable该IRQ有可能会下手太狠，kernel采取了这样的策略：如果该IRQ触发了100,000次，但是99,900次没有处理，在这种条件下，我们就是disable这个interrupt request line。多么有情有义的策略啊！相关的控制数据在中断描述符中，如下：

> struct irq_desc {  
> ……  
>     unsigned int        irq_count;－－－－－－－－记录发生的中断的次数，每100,000则回滚  
>     unsigned long        last_unhandled;－－－－－上一次没有处理的IRQ的时间点  
>     unsigned int        irqs_unhandled;－－－－－－没有处理的次数  
> ……  
> }

irq_count和irqs_unhandled都是比较直观的，为何要记录unhandled interrupt发生的时间呢？我们来看具体的代码。具体的相关代码位于note_interrupt中，如下：

> void note_interrupt(unsigned int irq, struct irq_desc *desc,  irqreturn_t action_ret)  
> {  
>     if (desc->istate & IRQS_POLL_INPROGRESS ||  irq_settings_is_polled(desc))  
>         return;
> 
>   
>     if (action_ret == IRQ_WAKE_THREAD)－－－－handler返回IRQ_WAKE_THREAD是正常情况  
>         return;
> 
>     if (bad_action_ret(action_ret)) {－－－－－报告错误，这些是由于specific handler的返回错误导致的  
>         report_bad_irq(irq, desc, action_ret);  
>         return;  
>     }
> 
>     if (unlikely(action_ret == IRQ_NONE)) {－－－－－－－是unhandled interrupt  
>         if (time_after(jiffies, desc->last_unhandled + HZ/10))－－－（1）  
>             desc->irqs_unhandled = 1;－－－重新开始计数  
>         else  
>             desc->irqs_unhandled++;－－－判定为unhandled interrupt，计数加一  
>         desc->last_unhandled = jiffies;－－－－－－－保存本次unhandled interrupt对应的jiffies时间  
>     }
> 
> if (unlikely(try_misrouted_irq(irq, desc, action_ret))) {－－－是否启动Misrouted IRQ fixup  
>     int ok = misrouted_irq(irq);  
>     if (action_ret == IRQ_NONE)  
>         desc->irqs_unhandled -= ok;  
> }
> 
>     desc->irq_count++;  
>     if (likely(desc->irq_count < 100000))－－－－－－－－－－－（2）  
>         return;
> 
>     desc->irq_count = 0;  
>     if (unlikely(desc->irqs_unhandled > 99900)) {－－－－－－－－（3）  
>   
>         __report_bad_irq(irq, desc, action_ret);－－－报告错误  
>   
>         desc->istate |= IRQS_SPURIOUS_DISABLED;  
>         desc->depth++;  
>         irq_disable(desc);
> 
>         mod_timer(&poll_spurious_irq_timer,－－－－－－－－－－（4）  
>               jiffies + POLL_SPURIOUS_IRQ_INTERVAL);  
>     }  
>     desc->irqs_unhandled = 0;  
> }

（1）是否是一次有效的unhandled interrupt还要根据时间来判断。一般而言，当硬件处于异常状态的时候往往是非常短的时间触发非常多次的中断，如果距离上次unhandled interrupt的时间超过了10个jiffies（如果HZ＝100，那么时间就是100ms），那么我们要把irqs_unhandled重新计数。如果不这么处理的话，随着时间的累计，最终irqs_unhandled可能会达到99900次的，从而把这个IRQ错误的推上了审判台。

（2）irq_count每次都会加一，记录IRQ被触发的次数。但只要大于100000才启动 step （3）中的检查。一旦启动检查，irq_count会清零，irqs_unhandled也会清零，进入下一个检查周期。

（3）如果满足条件（IRQ触发了100,000次，但是99,900次没有处理），disable该IRQ。

（4）启动timer，轮询整个系统中的handler来处理这个中断（轮询啊，绝对是真爱啊）。这个timer的callback函数定义如下：

> static void poll_spurious_irqs(unsigned long dummy)  
> {  
>     struct irq_desc *desc;  
>     int i;
> 
>     if (atomic_inc_return(&irq_poll_active) != 1)－－－－确保系统中只有一个excuting thread进入临界区  
>         goto out;  
>     irq_poll_cpu = smp_processor_id(); －－－－记录当前正在polling的CPU
> 
>     for_each_irq_desc(i, desc) {－－－－－－遍历所有的中断描述符  
>         unsigned int state;
> 
>         if (!i)－－－－－－－－－－－－－越过0号中断描述符。对于X86，这是timer的中断  
>              continue;
> 
>         /* Racy but it doesn't matter */  
>         state = desc->istate;  
>         barrier();  
>         if (!(state & IRQS_SPURIOUS_DISABLED))－－－－名花有主的那些就不必考虑了  
>             continue;
> 
>         local_irq_disable();  
>         try_one_irq(i, desc, true);－－－－－－－－－OK，尝试一下是不是可以处理  
>         local_irq_enable();  
>     }  
> out:  
>     atomic_dec(&irq_poll_active);  
>     mod_timer(&poll_spurious_irq_timer,－－－－－－－－一旦触发了该timer，就停不下来  
>           jiffies + POLL_SPURIOUS_IRQ_INTERVAL);  
> }

三、和high level irq event handler相关的硬件描述

1、CPU layer和Interrupt controller之间的接口

从逻辑层面上看，CPU和interrupt controller之间的接口包括：

（1）触发中断的signal。一般而言，这个（些）信号是电平触发的。对于ARM CPU，它是nIRQ和nFIQ信号线，对于X86，它是INT和NMI信号线，对于PowerPC，这些信号线包括MC（machine check）、CRIT（critical interrupt）和NON-CRIT（Non critical interrupt）。对于linux kernel的中断子系统，我们只使用其中一个信号线（例如对于ARM而言，我们只使用nIRQ这个信号线）。这样，从CPU层面看，其逻辑动作非常的简单，不区分优先级，触发中断的那个信号线一旦assert，并且CPU没有mask中断，那么软件就会转到一个异常向量执行，完毕后返回现场。

（2）Ack中断的signal。这个signal可能是物理上的一个连接CPU和interrupt controller的铜线，也可能不是。对于X86＋8259这样的结构，Ack中断的signal就是nINTA信号线，对于ARM＋GIC而言，这个信号就是总线上的一次访问（读Interrupt Acknowledge Register寄存器）。CPU ack中断标识cpu开启启动中断服务程序（specific handler）去处理该中断。对于X86而言，ack中断可以让8259将interrupt vector数据送到数据总线上，从而让CPU获取了足够的处理该中断的信息。对于ARM而言，ack中断的同时也就是获取了发生中断的HW interrupt ID，总而言之，ack中断后，CPU获取了足够开启执行中断处理的信息。

（3）结束中断（EOI，end of interrupt）的signal。这个signal用来标识CPU已经完成了对该中断的处理（specific handler或者ISR，interrupt serivce routine执行完毕）。实际的物理形态这里就不描述了，和ack中断signal是类似的。

（4）控制总线和数据总线接口。通过这些接口，CPU可以访问（读写）interrupt controller的寄存器。

2、Interrupt controller和Peripheral device之间的接口

所有的系统中，Interrupt controller和Peripheral device之间的接口都是一个Interrupt Request信号线。外设通过这个信号线上的电平或者边缘向CPU（实际上是通过interrupt controller）申请中断服务。

四、几种典型的high level irq event handler

本章主要介绍几种典型的high level irq event handler，在进行high level irq event handler的设定的时候需要注意，不是外设使用电平触发就选用handle_level_irq，选用什么样的high level irq event handler是和Interrupt controller的行为以及外设电平触发方式决定的。介绍每个典型的handler之前，我会简单的描述该handler要求的硬件行为，如果该外设的中断系统符合这个硬件行为，那么可以选择该handler为该中断的high level irq event handler。

1、边缘触发的handler。

使用handle_edge_irq这个handler的硬件中断系统行为如下：

[![xyz](http://www.wowotech.net/content/uploadfile/201408/1618e77f0e5a8cbc94806355a0ff3c4720140828120041.gif "xyz")](http://www.wowotech.net/content/uploadfile/201408/b844e701747cb80623342178d80fbe0120140828120039.gif)

我们以上升沿为例描述边缘中断的处理过程（下降沿的触发是类似的）。当interrupt controller检测到了上升沿信号，会将该上升沿状态（pending）锁存在寄存器中，并通过中断的signal向CPU触发中断。需要注意：这时候，外设和interrupt controller之间的interrupt request信号线会保持高电平，这也就意味着interrupt controller不可能检测到新的中断信号（本身是高电平，无法形成上升沿）。这个高电平信号会一直保持到软件ack该中断（调用irq chip的irq_ack callback函数）。ack之后，中断控制器才有可能继续探测上升沿，触发下一次中断。

ARM＋GIC组成的系统不符合这个类型。虽然GIC提供了IAR（Interrupt Acknowledge Register）寄存器来让ARM来ack中断，但是，在调用high level handler之前，中断处理程序需要通过读取IAR寄存器获得HW interrpt ID并转换成IRQ number，因此实际上，对于GIC的irq chip，它是无法提供本场景中的irq_ack函数的。很多GPIO type的interrupt controller符合上面的条件，它们会提供pending状态寄存器，读可以获取pending状态，而向pending状态寄存器写1可以ack该中断，让interrupt controller可以继续触发下一次中断。

handle_edge_irq代码如下：

> void handle_edge_irq(unsigned int irq, struct irq_desc *desc)  
> {  
>     raw_spin_lock(&desc->lock); －－－－－－－－－－－－－－－－－（0）
> 
>     desc->istate &= ~(IRQS_REPLAY | IRQS_WAITING);－－－－参考上一章的描述  
>   
>     if (unlikely(irqd_irq_disabled(&desc->irq_data) ||－－－－－－－－－－－（1）  
>              irqd_irq_inprogress(&desc->irq_data) || !desc->action)) {  
>         if (!irq_check_poll(desc)) {  
>             desc->istate |= IRQS_PENDING;  
>             mask_ack_irq(desc);  
>             goto out_unlock;  
>         }  
>     }  
>     kstat_incr_irqs_this_cpu(irq, desc); －－－更新该IRQ统计信息
> 
>   
>     desc->irq_data.chip->irq_ack(&desc->irq_data); －－－－－－－－－（2）
> 
>     do {  
>         if (unlikely(!desc->action)) { －－－－－－－－－－－－－－－－－（3）  
>             mask_irq(desc);  
>             goto out_unlock;  
>         }
> 
>   
>         if (unlikely(desc->istate & IRQS_PENDING)) { －－－－－－－－－（4）  
>             if (!irqd_irq_disabled(&desc->irq_data) &&  
>                 irqd_irq_masked(&desc->irq_data))  
>                 unmask_irq(desc);  
>         }
> 
>         handle_irq_event(desc); －－－－－－－－－－－－－－－－－－－（5）
> 
>     } while ((desc->istate & IRQS_PENDING) &&  
>          !irqd_irq_disabled(&desc->irq_data)); －－－－－－－－－－－－－（6）
> 
> out_unlock:  
>     raw_spin_unlock(&desc->lock); －－－－－－－－－－－－－－－－－（7）  
> }

（0） 这时候，中断仍然是关闭的，因此不会有来自本CPU的并发，使用raw spin lock就防止其他CPU上对该IRQ的中断描述符的访问。针对该spin lock，我们直观的感觉是raw_spin_lock和（7）中的raw_spin_unlock是成对的，实际上并不是，handle_irq_event中的代码是这样的：

> irqreturn_t handle_irq_event(struct irq_desc *desc)  
> {
> 
>     raw_spin_unlock(&desc->lock); －－－－－－－和上面的（0）对应
> 
>     处理具体的action list
> 
>     raw_spin_lock(&desc->lock);－－－－－－－－和上面的（7）对应  
>   
> }

实际上，由于在handle_irq_event中处理action list的耗时还是比较长的，因此处理具体的action list的时候并没有持有中断描述符的spin lock。在如果那样的话，其他CPU在对中断描述符进行操作的时候需要spin的时间会很长的。

（1）判断是否需要执行下面的action list的处理。这里分成几种情况：

a、该中断事件已经被其他的CPU处理了

b、该中断被其他的CPU disable了

c、该中断描述符没有注册specific handler。这个比较简单，如果没有irqaction，根本没有必要调用action list的处理

如果该中断事件已经被其他的CPU处理了，那么我们仅仅是设定pending状态（为了委托正在处理的该中断的那个CPU进行处理），mask_ack_irq该中断并退出就OK了，并不做具体的处理。另外正在处理该中断的CPU会检查pending状态，并进行处理的。同样的，如果该中断被其他的CPU disable了，本就不应该继续执行该中断的specific handler，我们也是设定pending状态，mask and ack中断就退出了。当其他CPU的代码离开临界区，enable 该中断的时候，软件会检测pending状态并resend该中断。

这里的irq_check_poll代码如下：

> static bool irq_check_poll(struct irq_desc *desc)  
> {  
>     if (!(desc->istate & IRQS_POLL_INPROGRESS))  
>         return false;  
>     return irq_wait_for_poll(desc);  
> }

IRQS_POLL_INPROGRESS标识了该IRQ正在被polling（上一章有描述），如果没有被轮询，那么返回false，进行正常的设定pending标记、mask and ack中断。如果正在被轮询，那么需要等待poll结束。

（2）ack该中断。对于中断控制器，一旦被ack，表示该外设的中断被enable，硬件上已经准备好触发下一次中断了。再次触发的中断会被调度到其他的CPU上。现在，我们可以再次回到步骤（1）中，为什么这里用mask and ack而不是单纯的ack呢？如果单纯的ack则意味着后续中断还是会触发，这时候怎么处理？在pending＋in progress的情况下，我们要怎么处理？记录pending的次数，有意义吗？由于中断是完全异步的，也有可能pending的标记可能在另外的CPU上已经修改为replay的标记，这时候怎么办？当事情变得复杂的时候，那一定是本来方向就错了，因此，mask and ack就是最好的策略，我已经记录了pending状态，不再考虑pending嵌套的情况。

（3）在调用specific handler处理具体的中断的时候，由于不持有中断描述符的spin lock，因此其他CPU上有可能会注销其specific handler，因此do while循环之后，desc->action有可能是NULL，如果是这样，那么mask irq，然后退出就OK了

（4）如果中断描述符处于pending状态，那么一定是其他CPU上又触发了该interrupt source的中断，并设定了pending状态，“委托”本CPU进行处理，这时候，需要把之前mask住的中断进行unmask的操作。一旦unmask了该interrupt source，后续的中断可以继续触发，由其他的CPU处理（仍然是设定中断描述符的pending状态，委托当前正在处理该中断请求的那个CPU进行处理）。

（5）处理该中断请求事件

> irqreturn_t handle_irq_event(struct irq_desc *desc)  
> {  
>     struct irqaction *action = desc->action;  
>     irqreturn_t ret;
> 
>     desc->istate &= ~IRQS_PENDING;－－－－CPU已经准备处理该中断了，因此，清除pending状态  
>     irqd_set(&desc->irq_data, IRQD_IRQ_INPROGRESS);－－设定INPROGRESS的flag  
>     raw_spin_unlock(&desc->lock);
> 
>     ret = handle_irq_event_percpu(desc, action); －－－遍历action list，调用specific handler
> 
>     raw_spin_lock(&desc->lock);  
>     irqd_clear(&desc->irq_data, IRQD_IRQ_INPROGRESS);－－－处理完成，清除INPROGRESS标记  
>     return ret;  
> }

（6）只要有pending标记，就说明该中断还在pending状态，需要继续处理。当然，如果有其他的CPU disable了该interrupt source，那么本次中断结束处理。

2、电平触发的handler

使用handle_level_irq这个handler的硬件中断系统行为如下：

[![level](http://www.wowotech.net/content/uploadfile/201408/335dea2bd22abd76c4df50e9953b759920140828120046.gif "level")](http://www.wowotech.net/content/uploadfile/201408/5a88d2c9e746aa35e79f6de6c1f6afd020140828120043.gif)

我们以高电平触发为例。当interrupt controller检测到了高电平信号，并通过中断的signal向CPU触发中断。这时候，对中断控制器进行ack并不能改变interrupt request signal上的电平状态，一直要等到执行具体的中断服务程序（specific handler），对外设进行ack的时候，电平信号才会恢复成低电平。在对外设ack之前，中断状态一直是pending的，如果没有mask中断，那么中断控制器就会assert CPU。

handle_level_irq的代码如下：

> void handle_level_irq(unsigned int irq, struct irq_desc *desc)  
> {  
>     raw_spin_lock(&desc->lock);  
>     mask_ack_irq(desc); －－－－－－－－－－－－－－－－－－－－－（1）
> 
>     if (unlikely(irqd_irq_inprogress(&desc->irq_data)))－－－－－－－－－（2）  
>         if (!irq_check_poll(desc))  
>             goto out_unlock;
> 
>     desc->istate &= ~(IRQS_REPLAY | IRQS_WAITING);－－和retrigger中断以及自动探测IRQ相关  
>     kstat_incr_irqs_this_cpu(irq, desc);
> 
>   
>     if (unlikely(!desc->action || irqd_irq_disabled(&desc->irq_data))) {－－－－－（3）  
>         desc->istate |= IRQS_PENDING;  
>         goto out_unlock;  
>     }
> 
>     handle_irq_event(desc);
> 
>     cond_unmask_irq(desc); －－－－－－－－－－－－－－（4）
> 
> out_unlock:  
>     raw_spin_unlock(&desc->lock);  
> }

（1）考虑CPU<------>interrupt controller<------>device这样的连接方式中，我们认为high level handler主要是和interrupt controller交互，而specific handler（request_irq注册的那个）是和device进行交互。Level类型的中断的特点就是只要外设interrupt request line的电平状态是有效状态，对于interrupt controller，该外设的interrupt总是active的。由于外设检测到了事件（比如数据到来了），因此assert了指定的电平信号，这个电平信号会一直保持，直到软件清除了外设的状态寄存器。但是，high level irq event handler这个层面只能操作Interrupt controller，不能操作具体外设的寄存器（那应该属于具体外设的specific interrupt handler处理内容，该handler会挂入中断描述符中的IRQ action list）。直到在具体的中断服务程序（specific handler中）操作具体外设的寄存器，才能让这个asserted电平信号消息。

正是因为level trigger的这个特点，因此，在high level handler中首先mask并ack该IRQ。这一点和边缘触发的high level handler有显著的不同，在handle_edge_irq中，我们仅仅是ack了中断，并没有mask，因为边缘触发的中断稍纵即逝，一旦mask了该中断，容易造成中断丢失。而对于电平中断，我们不得不mask住该中断，如果不mask住，只要CPU ack中断，中断控制器将持续的assert CPU中断（因为有效电平状态一直保持）。如果我们mask住该中断，中断控制器将不再转发该interrupt source来的中断，因此，所有的CPU都不会感知到该中断，直到软件unmask。这里的ack是针对interrupt controller的ack，本身ack就是为了clear interrupt controller对该IRQ的状态寄存器，不过由于外部的电平仍然是有效信号，其实未必能清除interrupt controller的中断状态，不过这是和中断控制器硬件实现相关的。

（2）对于电平触发的high level handler，我们一开始就mask并ack了中断，因此后续specific handler因该是串行化执行的，为何要判断in progress标记呢？不要忘记spurious interrupt，那里会直接调用handler来处理spurious interrupt。

（3）这里有两个场景

a、没有注册specific handler。如果没有注册handler，那么保持mask并设定pending标记（这个pending标记有什么作用还没有想明白）。

b、该中断被其他的CPU disable了。如果该中断被其他的CPU disable了，本就不应该继续执行该中断的specific handler，我们也是设定pending状态，mask and ack中断就退出了。当其他CPU的代码离开临界区，enable 该中断的时候，软件会检测pending状态并resend该中断。

（4）为何是有条件的unmask该IRQ？正常的话当然是umask就OK了，不过有些threaded interrupt（这个概念在下一份文档中描述）要求是one shot的（首次中断，specific handler中开了一枪，wakeup了irq handler thread，如果允许中断嵌套，那么在specific handler会多次开枪，这也就不是one shot了，有些IRQ的handler thread要求是one shot，也就是不能嵌套specific handler）。

3、支持EOI的handler

TODO

_原创文章，转发请注明出处。蜗窝科技。_[http://www.wowotech.net/linux_kenrel/High_level_irq_event_handler.html](http://www.wowotech.net/linux_kenrel/High_level_irq_event_handler.html)

标签: [中断处理](http://www.wowotech.net/tag/%E4%B8%AD%E6%96%AD%E5%A4%84%E7%90%86)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [linux kernel的中断子系统之（七）：GIC代码分析](http://www.wowotech.net/irq_subsystem/gic_driver.html) | [linux kernel的中断子系统之（三）：IRQ number和中断描述符](http://www.wowotech.net/irq_subsystem/interrupt_descriptor.html)»

**评论：**

**YG**  
2023-03-24 14:51

博主14年写的文章，现在读起来依然受益匪浅，佩服！！  
真想知道博主是怎么一步步到达这种水平的

[回复](http://www.wowotech.net/irq_subsystem/High_level_irq_event_handler.html#comment-8760)

**linux初学者**  
2021-01-26 21:27

是的，我看到这个描述也很困惑，linux里设置eionmod=1，ack只是获取中断号并将中断状态置为active，eoi只是将当前cpu上的running prority改为idle，idr才真正结束中断

[回复](http://www.wowotech.net/irq_subsystem/High_level_irq_event_handler.html#comment-8189)

**zgh**  
2020-11-30 16:32

即使是dts告知系统,如果你无法获知virq,你之所以问这个问题,你是不是没搞清楚virq和hwirq 以及irq_domain的关系?

[回复](http://www.wowotech.net/irq_subsystem/High_level_irq_event_handler.html#comment-8149)

**VincentLi**  
2020-08-06 09:56

请教大神一个问题，对于 每一个 IRQ number 对应的 irq_desc，都必须注册了 high-level irq handler吗？  
但是我看一些驱动代码，是直接 devm_request_irq 这一步的，并没有注册 high-level irq handler ，有点疑惑

[回复](http://www.wowotech.net/irq_subsystem/High_level_irq_event_handler.html#comment-8084)

**yz**  
2022-03-18 13:51

@VincentLi：调用devm_request_irq前先要获取irq，例如platform_get_irq函数；  
这里以GIC为例：  
platform_get_irq  
  -->of_irq_get  
      -->irq_create_of_mapping  
         -->irq_domain_associate  
            -->domain->ops->map(domain, virq, hwirq);  
               -->gic_irq_domain_map  
                  -->irq_domain_set_info(d, irq, hw, &gic_chip,  
                      d->host_data, handle_fasteoi_irq, NULL, NULL);

[回复](http://www.wowotech.net/irq_subsystem/High_level_irq_event_handler.html#comment-8577)

**nowar**  
2019-08-30 11:20

@linuxer，  
这些博客不维护了吗？

[回复](http://www.wowotech.net/irq_subsystem/High_level_irq_event_handler.html#comment-7635)

**[linuxer](http://www.wowotech.net/)**  
2019-08-31 14:00

@nowar：工作太忙，近一段时间估计是无法回答大家的问题了。

[回复](http://www.wowotech.net/irq_subsystem/High_level_irq_event_handler.html#comment-7637)

**nowar**  
2019-08-30 11:18

在阅读前一篇文章的中断流程里面，没有看到哪个环节需要中断自动探测啊。这篇文章怎么突然就出现了中断自动探测的东东？什么情景下需要使用它呢？不是在中断控制器 和 设备driver的初始化流程中已经自动mapping了irq number和hw irq id了吗？这个过程之后它已经知道自己的irq number了啊

[回复](http://www.wowotech.net/irq_subsystem/High_level_irq_event_handler.html#comment-7634)

**abcde1120**  
2018-06-29 10:47

（2）ack该中断。对于中断控制器，一旦被ack，表示该外设的中断被enable，硬件上已经准备好触发下一次中断了。再次触发的中断会被调度到其他的CPU上。  
》》》》》》》》》》  
在V2 specification里说，对于SPI，只有EOI后，才会再次触发中断。ACK后，其它CPU应该是收不到中断的吧  
  
原文：  
3.2  General handling of interrupts  
....  
...  
     For SPIs,the active status of an interupt is common to all CPU interface. This means that if an interrupt is active or active and pending on one CPU then it is not signaled on any CPU interface.

[回复](http://www.wowotech.net/irq_subsystem/High_level_irq_event_handler.html#comment-6827)

**jianwei**  
2018-05-06 18:21

中断描述符状态有 IRQD_IRQ_MAKSED 和 IRQD_IRQ_DISABLED，它们区别是什么？

[回复](http://www.wowotech.net/irq_subsystem/High_level_irq_event_handler.html#comment-6737)

**Jarson**  
2018-03-13 16:20

int generic_handle_irq(unsigned int irq)  
{  
    struct irq_desc *desc = irq_to_desc(irq); －－－通过IRQ number获取该irq的描述符  
    if (!desc)  
        return -EINVAL;  
    generic_handle_irq_desc(irq, desc);－－－－调用high level的irq handler来处理该IRQ  
    return 0;  
}  
在获取irq描述符时，只有一个传入参数irq（IRQ number），考虑有级联中断控制器的情况，如何确保 irq_to_desc返回正确的irq描述符？irq_to_desc是如何知道所传入的IRQ number位于哪一个 irq domain？

[回复](http://www.wowotech.net/irq_subsystem/High_level_irq_event_handler.html#comment-6609)

**Jarson**  
2018-03-13 16:29

@Jarson：补充一个GPIO type中断控制器的中断处理函数，该函数在进一步处理GPIO中断时，也调用了generic_handle_irq函数，但传入的irq参数值为GPIO irq domain下的IRQ number。  
static void gpio_irq_handler(unsigned irq, struct irq_desc *desc)  
{  
        struct irq_chip *chip = irq_desc_get_chip(desc);  
        struct irq_data *idata = irq_desc_get_irq_data(desc);  
        struct at91_gpio_chip *at91_gpio = irq_data_get_irq_chip_data(idata);  
        void __iomem    *pio = at91_gpio->regbase;  
        unsigned long   isr;  
        int             n;  
  
        chained_irq_enter(chip, desc);  
        for (;;) {  
                /* Reading ISR acks pending (edge triggered) GPIO interrupts.  
                 * When there none are pending, we're finished unless we need  
                 * to process multiple banks (like ID_PIOCDE on sam9263).  
                 */  
                isr = __raw_readl(pio + PIO_ISR) & __raw_readl(pio + PIO_IMR);  
                if (!isr) {  
                        if (!at91_gpio->next)  
                                break;  
                        at91_gpio = at91_gpio->next;  
                        pio = at91_gpio->regbase;  
                        continue;  
                }  
  
                n = find_first_bit(&isr, BITS_PER_LONG);  
                while (n < BITS_PER_LONG) {  
                        generic_handle_irq(irq_find_mapping(at91_gpio->domain, n));  
                        n = find_next_bit(&isr, BITS_PER_LONG, n + 1);  
                }  
        }  
        chained_irq_exit(chip, desc);  
        /* now it may re-trigger */  
}

[回复](http://www.wowotech.net/irq_subsystem/High_level_irq_event_handler.html#comment-6610)

**Jarson**  
2018-03-14 11:08

@Jarson：自问自答了，继续跟踪irq描述符分配代码 irq_alloc_desc_from 函数和 irq_domain_associate 函数，可知道 virq 也即IRQ number在内核空间是唯一的，而 hwirq 是与irq domain相关。简单来说，virq 值不会重复，而 hwirq 在不同的irq domain中可能相同。

[回复](http://www.wowotech.net/irq_subsystem/High_level_irq_event_handler.html#comment-6612)

**[jak](http://no%20home%20page%20yet/)**  
2017-11-30 11:14

问题1：同一个irq domain中，多个HW interrupt ID能否映射到同一个IRQ number?  
问题2：多个HW interrupt ID处于多个irq domain中，能否映射到同一个IRQ number?  
问题3：(如果允许多对一映射)那么在high level handler中如果要触发mask时，怎么样根据IRQ number反过去知道要mask哪个HW interrupt

[回复](http://www.wowotech.net/irq_subsystem/High_level_irq_event_handler.html#comment-6268)

1 [2](http://www.wowotech.net/irq_subsystem/High_level_irq_event_handler.html/comment-page-2#comments) [3](http://www.wowotech.net/irq_subsystem/High_level_irq_event_handler.html/comment-page-3#comments)

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
    
    - [Fix-Mapped Addresses](http://www.wowotech.net/memory_management/fixmap.html)
    - [Linux2.6.11版本：classic RCU的实现](http://www.wowotech.net/kernel_synchronization/linux2-6-11-RCU.html)
    - [Linux reset framework](http://www.wowotech.net/pm_subsystem/reset_framework.html)
    - [per-entity load tracking](http://www.wowotech.net/process_management/PELT.html)
    - [我眼中的可穿戴设备](http://www.wowotech.net/tech_discuss/106.html)
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
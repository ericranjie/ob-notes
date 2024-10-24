
作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2014-10-24 11:53 分类：[中断子系统](http://www.wowotech.net/sort/irq_subsystem)

# 一、前言

对于中断处理而言，linux将其分成了两个部分，一个叫做中断handler（top half），是全程关闭中断的，另外一部分是deferable task（bottom half），属于不那么紧急需要处理的事情。在执行bottom half的时候，是开中断的。有多种bottom half的机制，例如：softirq、tasklet、workqueue或是直接创建一个kernel thread来执行bottom half（这在旧的kernel驱动中常见，现在，一个理智的driver厂商是不会这么做的）。本文主要讨论softirq机制。由于tasklet是基于softirq的，因此本文也会提及tasklet，但主要是从需求层面考虑，不会涉及其具体的代码实现。

在普通的驱动中一般是不会用到softirq，但是由于驱动经常使用的tasklet是基于softirq的，因此，了解softirq机制有助于撰写更优雅的driver。softirq不能动态分配，都是静态定义的。内核已经定义了若干种softirq number，例如网络数据的收发、block设备的数据访问（数据量大，通信带宽高），timer的deferable task（时间方面要求高）。本文的第二章讨论了softirq和tasklet这两种机制有何不同，分别适用于什么样的场景。第三章描述了一些context的概念，这是要理解后续内容的基础。第四章是进入softirq的实现，对比hard irq来解析soft irq的注册、触发，调度的过程。

注：本文中的linux kernel的版本是3.14

# 二、为何有softirq和tasklet

## 1、为何有top half和bottom half

中断处理模块是任何OS中最重要的一个模块，对系统的性能会有直接的影响。想像一下：如果在通过U盘进行大量数据拷贝的时候，你按下一个key，需要半秒的时间才显示出来，这个场景是否让你崩溃？因此，对于那些复杂的、需要大量数据处理的硬件中断，我们不能让handler中处理完一切再恢复现场（handler是全程关闭中断的），而是仅仅在handler中处理一部分，具体包括：

（1）有实时性要求的

（2）和硬件相关的。例如ack中断，read HW FIFO to ram等

（3）如果是共享中断，那么获取硬件中断状态以便判断是否是本中断发生

除此之外，其他的内容都是放到bottom half中处理。在把中断处理过程划分成top half和bottom half之后，关中断的top half被瘦身，可以非常快速的执行完毕，大大减少了系统关中断的时间，提高了系统的性能。

我们可以基于下面的系统进一步的进行讨论：

![[Pasted image 20241024193257.png]]

当网卡控制器的FIFO收到的来自以太网的数据的时候（例如半满的时候，可以软件设定），可以将该事件通过irq signal送达Interrupt Controller。Interrupt Controller可以把中断分发给系统中的Processor A or B。

NIC的中断处理过程大概包括：mask and ack interrupt controller-------->ack NIC-------->copy FIFO to ram------>handle Data in the ram----------->unmask interrupt controller

我们先假设Processor A处理了这个网卡中断事件，于是NIC的中断handler在Processor A上欢快的执行，这时候，Processor A的本地中断是disable的。NIC的中断handler在执行的过程中，网络数据仍然源源不断的到来，但是，如果NIC的中断handler不操作NIC的寄存器来ack这个中断的话，NIC是不会触发下一次中断的。还好，我们的NIC interrupt handler总是在最开始就会ack，因此，这不会导致性能问题。ack之后，NIC已经具体再次trigger中断的能力。当Processor A上的handler 在处理接收来自网络的数据的时候，NIC的FIFO很可能又收到新的数据，并trigger了中断，这时候，Interrupt controller还没有umask，因此，即便还有Processor B（也就是说有处理器资源），中断控制器也无法把这个中断送达处理器系统。因此，只能眼睁睁的看着NIC FIFO填满数据，数据溢出，或者向对端发出拥塞信号，无论如何，整体的系统性能是受到严重的影响。

注意：对于新的interrupt controller，可能没有mask和umask操作，但是原理是一样的，只不过NIC的handler执行完毕要发生EOI而已。

要解决上面的问题，最重要的是尽快的执行完中断handler，打开中断，unmask IRQ（或者发送EOI），方法就是把耗时的handle Data in the ram这个步骤踢出handler，让其在bottom half中执行。

## 2、为何有softirq和tasklet

OK，linux kernel已经把中断处理分成了top half和bottom half，看起来已经不错了，那为何还要提供softirq、tasklet和workqueue这些bottom half机制，linux kernel本来就够复杂了，bottom half还来添乱。实际上，在早期的linux kernel还真是只有一个bottom half机制，简称BH，简单好用，但是性能不佳。后来，linux kernel的开发者开发了task queue机制，试图来替代BH，当然，最后task queue也消失在内核代码中了。现在的linux kernel提供了三种bottom half的机制，来应对不同的需求。

workqueue和softirq、tasklet有本质的区别：workqueue运行在process context，而softirq和tasklet运行在interrupt context。因此，出现workqueue是不奇怪的，在有sleep需求的场景中，defering task必须延迟到kernel thread中执行，也就是说必须使用workqueue机制。softirq和tasklet是怎么回事呢？从本质上将，bottom half机制的设计有两方面的需求，一个是性能，一个是易用性。设计一个通用的bottom half机制来满足这两个需求非常的困难，因此，内核提供了softirq和tasklet两种机制。softirq更倾向于性能，而tasklet更倾向于易用性。

我们还是进入实际的例子吧，还是使用上一节的系统图。在引入softirq之后，网络数据的处理如下：

关中断：mask and ack interrupt controller-------->ack NIC-------->copy FIFO to ram------>raise softirq------>unmask interrupt controller

开中断：在softirq上下文中进行handle Data in the ram的动作

同样的，我们先假设Processor A处理了这个网卡中断事件，很快的完成了基本的HW操作后，raise softirq。在返回中断现场前，会检查softirq的触发情况，因此，后续网络数据处理的softirq在processor A上执行。在执行过程中，NIC硬件再次触发中断，Interrupt controller将该中断分发给processor B，执行动作和Processor A是类似的，因此，最后，网络数据处理的softirq在processor B上执行。

为了性能，同一类型的softirq有可能在不同的CPU上并发执行，这给使用者带来了极大的痛苦，因为驱动工程师在撰写softirq的回调函数的时候要考虑重入，考虑并发，要引入同步机制。但是，为了性能，我们必须如此。

当网络数据处理的softirq同时在Processor A和B上运行的时候，网卡中断又来了（可能是10G的网卡吧）。这时候，中断分发给processor A，这时候，processor A上的handler仍然会raise softirq，但是并不会调度该softirq。也就是说，softirq在一个CPU上是串行执行的。这种情况下，系统性能瓶颈是CPU资源，需要增加更多的CPU来解决该问题。

如果是tasklet的情况会如何呢？为何tasklet性能不如softirq呢？如果一个tasklet在processor A上被调度执行，那么它永远也不会同时在processor B上执行，也就是说，tasklet是串行执行的（注：不同的tasklet还是会并发的），不需要考虑重入的问题。我们还是用网卡这个例子吧（注意：这个例子仅仅是用来对比，实际上，网络数据是使用softirq机制的），同样是上面的系统结构图。假设使用tasklet，网络数据的处理如下：

关中断：mask and ack interrupt controller-------->ack NIC-------->copy FIFO to ram------>schedule tasklet------>unmask interrupt controller

开中断：在softirq上下文中（一般使用TASKLET_SOFTIRQ这个softirq）进行handle Data in the ram的动作

同样的，我们先假设Processor A处理了这个网卡中断事件，很快的完成了基本的HW操作后，schedule tasklet（同时也就raise TASKLET_SOFTIRQ softirq）。在返回中断现场前，会检查softirq的触发情况，因此，在TASKLET_SOFTIRQ softirq的handler中，获取tasklet相关信息并在processor A上执行该tasklet的handler。在执行过程中，NIC硬件再次触发中断，Interrupt controller将该中断分发给processor B，执行动作和Processor A是类似的，虽然TASKLET_SOFTIRQ softirq在processor B上可以执行，但是，在检查tasklet的状态的时候，如果发现该tasklet在其他processor上已经正在运行，那么该tasklet不会被处理，一直等到在processor A上的tasklet处理完，在processor B上的这个tasklet才能被执行。这样的串行化操作虽然对驱动工程师是一个福利，但是对性能而言是极大的损伤。

# 三、理解softirq需要的基础知识（各种context）

1、preempt_count

为了更好的理解下面的内容，我们需要先看看一些基础知识：一个task的thread info数据结构定义如下（只保留和本场景相关的内容）：

> struct thread_info { \
> ……\
> int            preempt_count;    /\* 0 => preemptable, \<0 => bug \*/\
> ……\
> };

preempt_count这个成员被用来判断当前进程是否可以被抢占。如果preempt_count不等于0（可能是代码调用preempt_disable显式的禁止了抢占，也可能是处于中断上下文等），说明当前不能进行抢占，如果preempt_count等于0，说明已经具备了抢占的条件（当然具体是否要抢占当前进程还是要看看thread info中的flag成员是否设定了_TIF_NEED_RESCHED这个标记，可能是当前的进程的时间片用完了，也可能是由于中断唤醒了优先级更高的进程）。 具体preempt_count的数据格式可以参考下图：

![[Pasted image 20241024193323.png]]

preemption count用来记录当前被显式的禁止抢占的次数，也就是说，每调用一次preempt_disable，preemption count就会加一，调用preempt_enable，该区域的数值会减去一。preempt_disable和preempt_enable必须成对出现，可以嵌套，最大嵌套的深度是255。

hardirq count描述当前中断handler嵌套的深度。对于ARM平台的linux kernel，其中断部分的代码如下：

> void handle_IRQ(unsigned int irq, struct pt_regs \*regs)\
> {\
> struct pt_regs \*old_regs = set_irq_regs(regs);
>
> irq_enter(); \
> generic_handle_irq(irq);
>
> irq_exit();\
> set_irq_regs(old_regs);\
> }

通用的IRQ handler被irq_enter和irq_exit这两个函数包围。irq_enter说明进入到IRQ context，而irq_exit则说明退出IRQ context。在irq_enter函数中会调用preempt_count_add(HARDIRQ_OFFSET)，为hardirq count的bit field增加1。在irq_exit函数中，会调用preempt_count_sub(HARDIRQ_OFFSET)，为hardirq count的bit field减去1。hardirq count占用了4个bit，说明硬件中断handler最大可以嵌套15层。在旧的内核中，hardirq count占用了12个bit，支持4096个嵌套。当然，在旧的kernel中还区分fast interrupt handler和slow interrupt handler，中断handler最大可以嵌套的次数理论上等于系统IRQ的个数。在实际中，这个数目不可能那么大（内核栈就受不了），因此，即使系统支持了非常大的中断个数，也不可能各个中断依次嵌套，达到理论的上限。基于这样的考虑，后来内核减少了hardirq count占用bit数目，改成了10个bit（在general arch的代码中修改为10，实际上，各个arch可以redefine自己的hardirq count的bit数）。但是，当内核大佬们决定废弃slow interrupt handler的时候，实际上，中断的嵌套已经不会发生了。因此，理论上，hardirq count要么是0，要么是1。不过呢，不能总拿理论说事，实际上，万一有写奇葩或者老古董driver在handler中打开中断，那么这时候中断嵌套还是会发生的，但是，应该不会太多（一个系统中怎么可能有那么多奇葩呢？呵呵），因此，目前hardirq count占用了4个bit，应付15个奇葩driver是妥妥的。

对softirq count进行操作有两个场景：

（1）也是在进入soft irq handler之前给 softirq count加一，退出soft irq handler之后给 softirq count减去一。由于soft irq handler在一个CPU上是不会并发的，总是串行执行，因此，这个场景下只需要一个bit就够了，也就是上图中的bit 8。通过该bit可以知道当前task是否在sofirq context。

（2）由于内核同步的需求，进程上下文需要禁止softirq。这时候，kernel提供了local_bh_enable和local_bh_disable这样的接口函数。这部分的概念是和preempt disable/enable类似的，占用了bit9～15，最大可以支持127次嵌套。

## 2、一个task的各种上下文

看完了preempt_count之后，我们来介绍各种context：

> #define in_irq()        (hardirq_count())\
> #define in_softirq()        (softirq_count())\
> #define in_interrupt()        (irq_count())
>
> #define in_serving_softirq()    (softirq_count() & SOFTIRQ_OFFSET)

这里首先要介绍的是一个叫做IRQ context的术语。这里的IRQ context其实就是hard irq context，也就是说明当前正在执行中断handler（top half），只要preempt_count中的hardirq count大于0（＝1是没有中断嵌套，如果大于1，说明有中断嵌套），那么就是IRQ context。

softirq context并没有那么的直接，一般人会认为当sofirq handler正在执行的时候就是softirq context。这样说当然没有错，sofirq handler正在执行的时候，会增加softirq count，当然是softirq context。不过，在其他context的情况下，例如进程上下文中，有有可能因为同步的要求而调用local_bh_disable，这时候，通过local_bh_disable/enable保护起来的代码也是执行在softirq context中。当然，这时候其实并没有正在执行softirq handler。如果你确实想知道当前是否正在执行softirq handler，in_serving_softirq可以完成这个使命，这是通过操作preempt_count的bit 8来完成的。

所谓中断上下文，就是IRQ context ＋ softirq context＋NMI context。

# 四、softirq机制

softirq和hardirq（就是硬件中断啦）是对应的，因此softirq的机制可以参考hardirq对应理解，当然softirq是纯软件的，不需要硬件参与。

## 1、softirq number

和IRQ number一样，对于软中断，linux kernel也是用一个softirq number唯一标识一个softirq，具体定义如下：

> enum\
> {\
> HI_SOFTIRQ=0,\
> TIMER_SOFTIRQ,\
> NET_TX_SOFTIRQ,\
> NET_RX_SOFTIRQ,\
> BLOCK_SOFTIRQ,\
> BLOCK_IOPOLL_SOFTIRQ,\
> TASKLET_SOFTIRQ,\
> SCHED_SOFTIRQ,\
> HRTIMER_SOFTIRQ,\
> RCU_SOFTIRQ,    /\* Preferable RCU should always be the last softirq \*/
>
> NR_SOFTIRQS\
> };

HI_SOFTIRQ用于高优先级的tasklet，TASKLET_SOFTIRQ用于普通的tasklet。TIMER_SOFTIRQ是for software timer的（所谓software timer就是说该timer是基于系统tick的）。NET_TX_SOFTIRQ和NET_RX_SOFTIRQ是用于网卡数据收发的。BLOCK_SOFTIRQ和BLOCK_IOPOLL_SOFTIRQ是用于block device的。SCHED_SOFTIRQ用于多CPU之间的负载均衡的。HRTIMER_SOFTIRQ用于高精度timer的。RCU_SOFTIRQ是处理RCU的。这些具体使用情景分析会在各自的子系统中分析，本文只是描述softirq的工作原理。

## 2、softirq描述符

我们前面已经说了，softirq是静态定义的，也就是说系统中有一个定义softirq描述符的数组，而softirq number就是这个数组的index。这个概念和早期的静态分配的中断描述符概念是类似的。具体定义如下：

> struct softirq_action\
> {\
> void    (\*action)(struct softirq_action \*);\
> };
>
> static struct softirq_action softirq_vec\[NR_SOFTIRQS\] \_\_cacheline_aligned_in_smp;

系统支持多少个软中断，静态定义的数组就会有多少个entry。\_\_\_\_cacheline_aligned保证了在SMP的情况下，softirq_vec是对齐到cache line的。softirq描述符非常简单，只有一个action成员，表示如果触发了该softirq，那么应该调用action回调函数来处理这个soft irq。对于硬件中断而言，其mask、ack等都是和硬件寄存器相关并封装在irq chip函数中，对于softirq，没有硬件寄存器，只有“软件寄存器”，定义如下：

> typedef struct {\
> unsigned int \_\_softirq_pending;\
> #ifdef CONFIG_SMP\
> unsigned int ipi_irqs\[NR_IPI\];\
> #endif\
> } \_\_\_\_cacheline_aligned irq_cpustat_t;
>
> irq_cpustat_t irq_stat\[NR_CPUS\] \_\_\_\_cacheline_aligned;

ipi_irqs这个成员用于处理器之间的中断，我们留到下一个专题来描述。\_\_softirq_pending就是这个“软件寄存器”。softirq采用谁触发，谁负责处理的。例如：当一个驱动的硬件中断被分发给了指定的CPU，并且在该中断handler中触发了一个softirq，那么该CPU负责调用该softirq number对应的action callback来处理该软中断。因此，这个“软件寄存器”应该是每个CPU拥有一个（专业术语叫做banked register）。为了性能，irq_stat中的每一个entry被定义对齐到cache line。

## 3、如何注册一个softirq

通过调用open_softirq接口函数可以注册softirq的action callback函数，具体如下：

> void open_softirq(int nr, void (\*action)(struct softirq_action \*))\
> {\
> softirq_vec\[nr\].action = action;\
> }

softirq_vec是一个多CPU之间共享的数据，不过，由于所有的注册都是在系统初始化的时候完成的，那时候，系统是串行执行的。此外，softirq是静态定义的，每个entry（或者说每个softirq number）都是固定分配的，因此，不需要保护。

## 4、如何触发softirq？

在linux kernel中，可以调用raise_softirq这个接口函数来触发本地CPU上的softirq，具体如下：

> void raise_softirq(unsigned int nr)\
> {\
> unsigned long flags;
>
> local_irq_save(flags);\
> raise_softirq_irqoff(nr);\
> local_irq_restore(flags);\
> }

虽然大部分的使用场景都是在中断handler中（也就是说关闭本地CPU中断）来执行softirq的触发动作，但是，这不是全部，在其他的上下文中也可以调用raise_softirq。因此，触发softirq的接口函数有两个版本，一个是raise_softirq，有关中断的保护，另外一个是raise_softirq_irqoff，调用者已经关闭了中断，不需要关中断来保护“soft irq status register”。

所谓trigger softirq，就是在\_\_softirq_pending（也就是上面说的soft irq status register）的某个bit置一。从上面的定义可知，\_\_softirq_pending是per cpu的，因此不需要考虑多个CPU的并发，只要disable本地中断，就可以确保对，\_\_softirq_pending操作的原子性。

具体raise_softirq_irqoff的代码如下：

> inline void raise_softirq_irqoff(unsigned int nr)\
> {\
> \_\_raise_softirq_irqoff(nr); －－－－－－－－－－－－－－－－（1）
>
> if (!in_interrupt())\
> wakeup_softirqd();－－－－－－－－－－－－－－－－－－（2）\
> }

（1）\_\_raise_softirq_irqoff函数设定本CPU上的\_\_softirq_pending的某个bit等于1，具体的bit是由soft irq number（nr参数）指定的。

（2）如果在中断上下文，我们只要set \_\_softirq_pending的某个bit就OK了，在中断返回的时候自然会进行软中断的处理。但是，如果在context上下文调用这个函数的时候，我们必须要调用wakeup_softirqd函数用来唤醒本CPU上的softirqd这个内核线程。具体softirqd的内容请参考下一个章节。

## 5、disable/enable softirq

在linux kernel中，可以使用local_irq_disable和local_irq_enable来disable和enable本CPU中断。和硬件中断一样，软中断也可以disable，接口函数是local_bh_disable和local_bh_enable。虽然和想像的local_softirq_enable/disable有些出入，不过bh这个名字更准确反应了该接口函数的意涵，因为local_bh_disable/enable函数就是用来disable/enable bottom half的，这里就包括softirq和tasklet。

先看disable吧，毕竟禁止bottom half比较简单：

> static inline void local_bh_disable(void)\
> {\
> \_\_local_bh_disable_ip(_THIS_IP_, SOFTIRQ_DISABLE_OFFSET);\
> }
>
> static \_\_always_inline void \_\_local_bh_disable_ip(unsigned long ip, unsigned int cnt)\
> {\
> preempt_count_add(cnt);\
> barrier();\
> }

看起来disable bottom half比较简单，就是讲current thread info上的preempt_count成员中的softirq count的bit field9～15加上一就OK了。barrier是优化屏障（Optimization barrier），会在内核同步系列文章中描述。

enable函数比较复杂，如下：

> static inline void local_bh_enable(void)\
> {\
> \_\_local_bh_enable_ip(_THIS_IP_, SOFTIRQ_DISABLE_OFFSET);\
> }
>
> void \_\_local_bh_enable_ip(unsigned long ip, unsigned int cnt)\
> {\
> WARN_ON_ONCE(in_irq() || irqs_disabled());－－－－－－－－－－－（1）
>
> preempt_count_sub(cnt - 1); －－－－－－－－－－－－－－－－－－（2）
>
> if (unlikely(!in_interrupt() && local_softirq_pending())) { －－－－－－－（3）\
> do_softirq();\
> }
>
> preempt_count_dec(); －－－－－－－－－－－－－－－－－－－－－（4）\
> preempt_check_resched();\
> }

（1）disable/enable bottom half是一种内核同步机制。在硬件中断的handler（top half）中，不应该调用disable/enable bottom half函数来保护共享数据，因为bottom half其实是不可能抢占top half的。同样的，soft irq也不会抢占另外一个soft irq的执行，也就是说，一旦一个softirq handler被调度执行（无论在哪一个processor上），那么，本地的softirq handler都无法抢占其运行，要等到当前的softirq handler运行完毕后，才能执行下一个soft irq handler。注意：上面我们说的是本地，是local，softirq handler是可以在多个CPU上同时运行的，但是，linux kernel中没有disable all softirq的接口函数（就好像没有disable all CPU interrupt的接口一样，注意体会local_bh_enable/disable中的local的含义）。

说了这么多，一言以蔽之，local_bh_enable/disable是给进程上下文使用的，用于防止softirq handler抢占local_bh_enable/disable之间的临界区的。

irqs_disabled接口函数可以获知当前本地CPU中断是否是disable的，如果返回1，那么当前是disable 本地CPU的中断的。如果irqs_disabled返回1，有可能是下面这样的代码造成的：

> local_irq_disable();
>
> ……\
> local_bh_disable();
>
> ……
>
> local_bh_enable();\
> ……\
> local_irq_enable();

本质上，关本地中断是一种比关本地bottom half更强劲的锁，关本地中断实际上是禁止了top half和bottom half抢占当前进程上下文的运行。也许你会说：这也没有什么，就是有些浪费，至少代码逻辑没有问题。但事情没有这么简单，在local_bh_enable--->do_softirq--->\_\_do_softirq中，有一条无条件打开当前中断的操作，也就是说，原本想通过local_irq_disable/local_irq_enable保护的临界区被破坏了，其他的中断handler可以插入执行，从而无法保证local_irq_disable/local_irq_enable保护的临界区的原子性，从而破坏了代码逻辑。

in_irq()这个函数如果不等于0的话，说明local_bh_enable被irq_enter和irq_exit包围，也就是说在中断handler中调用了local_bh_enable/disable。这道理是和上面类似的，这里就不再详细描述了。

（2）在local_bh_disable中我们为preempt_count增加了SOFTIRQ_DISABLE_OFFSET，在local_bh_enable函数中应该减掉同样的数值。这一步，我们首先减去了（SOFTIRQ_DISABLE_OFFSET－1），为何不一次性的减去SOFTIRQ_DISABLE_OFFSET呢？考虑下面运行在进程上下文的代码场景：

> ……
>
> local_bh_disable
>
> ……需要被保护的临界区……
>
> local_bh_enable
>
> ……

在临界区内，有进程context 和softirq共享的数据，因此，在进程上下文中使用local_bh_enable/disable进行保护。假设在临界区代码执行的时候，发生了中断，由于代码并没有阻止top half的抢占，因此中断handler会抢占当前正在执行的thread。在中断handler中，我们raise了softirq，在返回中断现场的时候，由于disable了bottom half，因此虽然触发了softirq，但是不会调度执行。因此，代码返回临界区继续执行，直到local_bh_enable。一旦enable了bottom half，那么之前raise的softirq就需要调度执行了，因此，这也是为什么在local_bh_enable会调用do_softirq函数。

调用do_softirq函数来处理pending的softirq的时候，当前的task是不能被抢占的，因为一旦被抢占，下一次该task被调度运行的时候很可能在其他的CPU上去了（还记得吗？softirq的pending 寄存器是per cpu的）。因此，我们不能一次性的全部减掉，那样的话有可能preempt_count等于0，那样就允许抢占了。因此，这里减去了（SOFTIRQ_DISABLE_OFFSET－1），既保证了softirq count的bit field9～15被减去了1，又保持了preempt disable的状态。

（3）如果当前不是interrupt context的话，并且有pending的softirq，那么调用do_softirq函数来处理软中断。

（4）该来的总会来，在step 2中我们少减了1，这里补上，其实也就是preempt count-1。

（5）在softirq handler中很可能wakeup了高优先级的任务，这里最好要检查一下，看看是否需要进行调度，确保高优先级的任务得以调度执行。

## 5、如何处理一个被触发的soft irq

我们说softirq是一种defering task的机制，也就是说top half没有做的事情，需要延迟到bottom half中来执行。那么具体延迟到什么时候呢？这是本节需要讲述的内容，也就是说soft irq是如何调度执行的。

在上一节已经描述一个softirq被调度执行的场景，本节主要关注在中断返回现场时候调度softirq的场景。我们来看中断退出的代码，具体如下：

> void irq_exit(void)\
> {\
> ……\
> if (!in_interrupt() && local_softirq_pending())\
> invoke_softirq();
>
> ……\
> }

代码中“!in_interrupt()”这个条件可以确保下面的场景不会触发sotfirq的调度：

（1）中断handler是嵌套的。也就是说本次irq_exit是退出到上一个中断handler。当然，在新的内核中，这种情况一般不会发生，因为中断handler都是关中断执行的。

（2）本次中断是中断了softirq handler的执行。也就是说本次irq_exit是不是退出到进程上下文，而是退出到上一个softirq context。这一点也保证了在一个CPU上的softirq是串行执行的（注意：多个CPU上还是有可能并发的）

我们继续看invoke_softirq的代码：

> static inline void invoke_softirq(void)\
> {\
> if (!force_irqthreads) {\
> #ifdef CONFIG_HAVE_IRQ_EXIT_ON_IRQ_STACK\
> \_\_do_softirq();\
> #else\
> do_softirq_own_stack();\
> #endif\
> } else {\
> wakeup_softirqd();\
> }\
> }

force_irqthreads是和强制线程化相关的，主要用于interrupt handler的调试（一般而言，在线程环境下比在中断上下文中更容易收集调试数据）。如果系统选择了对所有的interrupt handler进行线程化处理，那么softirq也没有理由在中断上下文中处理（中断handler都在线程中执行了，softirq怎么可能在中断上下文中执行）。本身invoke_softirq这个函数是在中断上下文中被调用的，如果强制线程化，那么系统中所有的软中断都在sofirq的daemon进程中被调度执行。

如果没有强制线程化，softirq的处理也分成两种情况，主要是和softirq执行的时候使用的stack相关。如果arch支持单独的IRQ STACK，这时候，由于要退出中断，因此irq stack已经接近全空了（不考虑中断栈嵌套的情况，因此新内核下，中断不会嵌套），因此直接调用\_\_do_softirq()处理软中断就OK了，否则就调用do_softirq_own_stack函数在softirq自己的stack上执行。当然对ARM而言，softirq的处理就是在当前的内核栈上执行的，因此do_softirq_own_stack的调用就是调用\_\_do_softirq()，代码如下（删除了部分无关代码）：

> asmlinkage void \_\_do_softirq(void)\
> {
>
> ……
>
> pending = local_softirq_pending();－－－－－－－－－－－－－－－获取softirq pending的状态
>
> \_\_local_bh_disable_ip(_RET_IP_, SOFTIRQ_OFFSET);－－－标识下面的代码是正在处理softirq
>
> cpu = smp_processor_id();\
> restart:\
> set_softirq_pending(0); －－－－－－－－－清除pending标志
>
> local_irq_enable(); －－－－－－打开中断，softirq handler是开中断执行的
>
> h = softirq_vec; －－－－－－－获取软中断描述符指针
>
> while ((softirq_bit = ffs(pending))) {－－－－－－－寻找pending中第一个被设定为1的bit\
> unsigned int vec_nr;\
> int prev_count;
>
> h += softirq_bit - 1; －－－－－－指向pending的那个软中断描述符
>
> vec_nr = h - softirq_vec;－－－－获取soft irq number
>
> h->action(h);－－－－－－－－－指向softirq handler
>
> h++;\
> pending >>= softirq_bit;\
> }
>
> local_irq_disable(); －－－－－－－关闭本地中断
>
> pending = local_softirq_pending();－－－－－－－－－－（注1）\
> if (pending) {\
> if (time_before(jiffies, end) && !need_resched() &&\
> --max_restart)\
> goto restart;
>
> wakeup_softirqd();\
> }
>
> \_\_local_bh_enable(SOFTIRQ_OFFSET);－－－－－－－－－－标识softirq处理完毕
>
> }

（注1）再次检查softirq pending，有可能上面的softirq handler在执行过程中，发生了中断，又raise了softirq。如果的确如此，那么我们需要跳转到restart那里重新处理soft irq。当然，也不能总是在这里不断的loop，因此linux kernel设定了下面的条件：

（1）softirq的处理时间没有超过2个ms

（2）上次的softirq中没有设定TIF_NEED_RESCHED，也就是说没有有高优先级任务需要调度

（3）loop的次数小于 10次

因此，只有同时满足上面三个条件，程序才会跳转到restart那里重新处理soft irq。否则wakeup_softirqd就OK了。这样的设计也是一个平衡的方案。一方面照顾了调度延迟：本来，发生一个中断，系统期望在限定的时间内调度某个进程来处理这个中断，如果softirq handler不断触发，其实linux kernel是无法保证调度延迟时间的。另外一方面，也照顾了硬件的thoughput：已经预留了一定的时间来处理softirq。

_原创文章，转发请注明出处。蜗窝科技_

[http://www.wowotech.net/linux_kenrel/soft-irq.html](http://www.wowotech.net/linux_kenrel/soft-irq.html)

标签: [软中断](http://www.wowotech.net/tag/%E8%BD%AF%E4%B8%AD%E6%96%AD) [softirq](http://www.wowotech.net/tag/softirq)

---

« [新技能get: 订阅Linux内核邮件列表](http://www.wowotech.net/linux_application/lkml.html) | [防冲突机制介绍](http://www.wowotech.net/basic_tech/103.html)»

**评论：**

**农夫山泉**\
2022-12-28 19:00

关本地中断实际上是禁止了top half和bottom half抢占当前进程上下文的运行？这个是怎么实现的，看local_irq_disable()的代码并没有关软中断/

[回复](http://www.wowotech.net/irq_subsystem/soft-irq.html#comment-8727)

**shousi**\
2023-03-19 10:03

@农夫山泉：我理解是进exception的时候，硬件关闭了CPU的Interrupt位。直到irq_exit里面才主动打开。可以在qmeu里打个断点看下。

[回复](http://www.wowotech.net/irq_subsystem/soft-irq.html#comment-8756)

**[bsp](http://www.wowotech.net/)**\
2023-10-26 16:13

@农夫山泉：linux的软中断（invoke_softirq） 是在硬中段（top half）退出时才会执行，关了hardirq就等于关了softirq。

[回复](http://www.wowotech.net/irq_subsystem/soft-irq.html#comment-8840)

**owen**\
2020-12-08 16:25

softirqd 会被进程抢占的吧。

[回复](http://www.wowotech.net/irq_subsystem/soft-irq.html#comment-8153)

**chenningjun**\
2023-10-25 17:39

@owen：不会，但softirqd内部 cond_resched 主动出让CPU了。\
按目前理解：调度抢占是针对用户态。内核态只有中断。

[回复](http://www.wowotech.net/irq_subsystem/soft-irq.html#comment-8839)

**[linuxer_fy](http://no%20home%20page%20yet/)**\
2017-09-26 13:51

"调用do_softirq函数来处理pending的softirq的时候，当前的task是不能被抢占的，因为一旦被抢占，下一次该task被调度运行的时候很可能在其他的CPU上去了（还记得吗？softirq的pending 寄存器是per cpu的）。因此，我们不能一次性的全部减掉，那样的话有可能preempt_count等于0，那样就允许抢占了。"\
##############\
我对这一条有不同的看法。\
do_softirq-> 判断 in_interrupt ->无条件禁止本地中断 --> \_\_do_softirq -> \_\_local_bh_disable -> 开本地中断 ->处理softirq handler\
所以当前task不能被抢占是指不能在正在处理softirq handler中，而由于有\_\_local_bh_disable(即bit8为1），所以当前task永远不会被抢占。所以进程上下文调用local_bh_enable时，为了兼顾(3)中将要执行的do_softirq，而只减去 (offset -1) 是不是多余了？\
当前task如果在 (2)中全部减去offset，而在(3)之间被抢占的话，再次返回到当前task时，\
即使是在另一个CPU上运行，也不会有影响。

[回复](http://www.wowotech.net/irq_subsystem/soft-irq.html#comment-6070)

**searchthetruth**\
2018-05-30 17:44

@linuxer_fy：如果我没有记错的话在LDD3中关于tasklet的讲解中曾提到“对于tasklet来说最好保证其始终在同一个cpu上进行处理。”\
tasklet即是一种特殊的softirq，出于这样的角度考虑的话那么此处的（offset-1）就是一个很好方法去保证softirq始终在同一cpu上处理的方法。\
为何要保证tasklet始终在同一cpu上处理的深层原因目前我也不太清楚。

[回复](http://www.wowotech.net/irq_subsystem/soft-irq.html#comment-6775)

**停停走走**\
2020-07-25 09:15

@searchthetruth：softies pending bit是per cpu的，一旦被调度到别的cpu不会再执行原来那个soft irq了吧

[回复](http://www.wowotech.net/irq_subsystem/soft-irq.html#comment-8066)

**[linuxer_fy](http://no%20home%20page%20yet/)**\
2017-09-25 16:24

"NIC的中断处理过程大概包括：mask and ack interrupt controller-------->ack NIC-------->copy FIFO to ram------>handle Data in the ram----------->unmask interrupt controller"\
#########\
由这个想到的，LINUX中针对PIC是8259A的，在do_IRQ中，先mask_and_ack_8259a,然后handle_IRQ_event,最后enable_8259a_irq（即unmask），\
如此一来，岂不是针对某IRQ，不可能会产生中断嵌套，即在handle_IRQ_event中，即使打开了CPU本地的中断，但8259A芯片侧依然mask该IRQ\
因此，LINUX中将同一IRQ上产生的嵌套中断序列化这一做法在这里不起作用了

[回复](http://www.wowotech.net/irq_subsystem/soft-irq.html#comment-6069)

**[linuxer](http://www.wowotech.net/)**\
2017-09-26 18:45

@linuxer_fy：在旧的内核中允许中断嵌套，但是嵌套永远都是发生在不同的irq之间，同一种irq，无论过去还是现在都不会嵌套。

[回复](http://www.wowotech.net/irq_subsystem/soft-irq.html#comment-6073)

**linuxer_fy**\
2017-09-26 20:52

@linuxer：旧内核只是允许进入当前这个中断Handler时，可以通过SA_INTERRRUPT开启中断，这样别的IRQ的中断可以响应。同IRQ都不能嵌套的话，那处理同一个IRQ的handler时，\_\_do_IRQ,当初为什么设计成串行化呢，出于什么目的?\
并且对于global的interrupt,同一个IRQ被Mask掉后，SMP中所有CPU也都无法产生这个IRQ的中断了，也就是说不管是单CPU还是多CPU，好像都没必要将一个IRQ的handler串行化设计阿。

[回复](http://www.wowotech.net/irq_subsystem/soft-irq.html#comment-6075)

**[linuxer](http://www.wowotech.net/)**\
2017-09-27 11:00

@linuxer_fy：A IRQ可以中断A IRQ handler的执行，你觉得意义何在呢？

[回复](http://www.wowotech.net/irq_subsystem/soft-irq.html#comment-6077)

**linuxer_fy**\
2017-10-13 21:08

@linuxer：我只是在这里针对LINUX的中断设计提出疑问。\
另外你上面提到的 “同一种irq，无论过去还是现在都不会嵌套。”\
这种情况是通过第一次进入A的handler是mask当前IRQ line来实现的吧\
我又想了想，但不确定。针对SMP场合，这里的mask IRQ line是只针对当前CPU的 local APIC还是针对SMP中所有global的？\
我想知道linux上实现同一个irq不会嵌套，是指当前正在处理A的CPU本地不会产生同样的中断还是说整个SMP中都不会产生，都被屏蔽掉了？\
总而言这，我想搞清楚，在SMP上，一个中断A产生后，是不是在这期间这个中断不可能再出现了（包括本地CPU和其他 CPU上）

[回复](http://www.wowotech.net/irq_subsystem/soft-irq.html#comment-6108)

**busdriver**\
2017-09-23 11:35

调用do_softirq函数来处理pending的softirq的时候，当前的task是不能被抢占的，因为一旦被抢占，下一次该task被调度运行的时候很可能在其他的CPU上去了（还记得吗？softirq的pending 寄存器是per cpu的）。因此，我们不能一次性的全部减掉，那样的话有可能preempt_count等于0，那样就允许抢占了。因此，这里减去了（SOFTIRQ_DISABLE_OFFSET－1），既保证了softirq count的bit field9～15被减去了1，又保持了preempt disable的状态。

______________________________________________________________________

这段话没看懂。do_softirq处理的是本地cpu pending的softirq，看后面讲述的do_softirq的流程，如果softirq超过2ms或者有更高优先级的任务就绪或者pending的softirq超过10次，都会激活线程softirqd，而这时候又是禁止抢占的，那么softirq将得不到执行而返回当前task吗？假如不禁止抢占，softirqd也将在本地cpu执行，返回也将在本地，当前task有可能被调度到别的cpu吗？这点不明白，谢谢解惑。

[回复](http://www.wowotech.net/irq_subsystem/soft-irq.html#comment-6063)

**[linuxer](http://www.wowotech.net/)**\
2017-09-26 18:39

@busdriver：这里表面上看起来是进程抢占进程，实际上进程抢占softirq。softirq被进程抢占本身就是不合理的，在Linux的内核中，只有softirq抢占进程，没有进程抢占softirq。

如果禁止抢占，那么一定会回到原来的那一点，等打开抢占的时候，调度器会决定是否执行softirqd。

如果softirqd抢占了当前task，那么当前task被挂入了本cpu的run queue，在负载均衡的时候，该task还是有可能调度到其他cpu上去的。

[回复](http://www.wowotech.net/irq_subsystem/soft-irq.html#comment-6072)

**busdriver**\
2017-09-26 20:08

@linuxer：@linuxer 非常感谢！这回懂了。

[回复](http://www.wowotech.net/irq_subsystem/soft-irq.html#comment-6074)

**kuninchengdu**\
2019-07-30 14:56

@busdriver：一般而且，do_softirp都是raiseirqoff后，软中断执行的，为啥一直在强调taskset呢，如果不是taskset，就算do_softirq被硬中断抢占，硬中断执行完后，还是原来的cpu继续执行do_softirq吧？

或者说，你的task就是指进程，进程又怎么执行soft_irq呢

[回复](http://www.wowotech.net/irq_subsystem/soft-irq.html#comment-7558)

**小学生**\
2017-06-25 15:06

楼主，请问为什么softirq在同一个CPU上要串行化处理呢？为什么不可以像中断那样：低优先级的softirq可以被高优先级的softirq抢占并先执行呢？是防止功能错误还是性能考虑呢？

[回复](http://www.wowotech.net/irq_subsystem/soft-irq.html#comment-5731)

**[linuxer](http://www.wowotech.net/)**\
2017-06-26 19:15

@小学生：在讨论你的问题之前，我们先考虑这样的一个问题：A interrupt handler可以抢占B interrupt handler的执行吗？

在2.6.35的内核之前，一个interrupt handler可以是开中断的（slow handler），或者是关中断的（fast handler），只有slow handler才会被其他的打断执行，从而引入nested interrupt handler的概念。

其实在新的内核中，中断已经是不能被抢占的了，所有的interrupt handler都是fast handler，也就是说，即便是一个新的，优先级更高的中断到来，其实cpu并不会响应，要等到当前的中断handler执行完毕才会考虑响应下一个中断，所有的interrupt handler都是关中断执行的。

在新的内核中，实际上是简化了interrupt handler处理，nested interrupt handler有很多坏处：\
1、中断栈的溢出问题不好处理\
2、中断延迟不确定，影响realtime\
3、不断的中断当前CPU处理，cache性能不佳\
4、内核控制路径的同步处理变得复杂\
......

softirq的情况其实有点类似hardirq，你可以考虑一下。任何需要softirq抢占的场景都可以用其他方法实现。

[回复](http://www.wowotech.net/irq_subsystem/soft-irq.html#comment-5737)

**tczb**\
2017-06-23 09:17

文章写得很好,现在看起来也收获很大

@linuxer, 也麻烦请教几个问题(和中断相关)，感谢：

我遇到一个kenerl panic，oops如下，有几个疑惑：

1. Call trace里面分2部分，以Exception stack为分界，上面的中断处理和下面的kthread，在内核中是2个线程分别在处理吗？

1. 中断处理do_softirq，应该是个TIMERIRQ，回调函数是blk_mq_rq_timer，这个软中断是由内核(时钟计时)自动启动线程来调用的吗？用户态不可控？

1. 如果1成立，这2个线程应该存在竞争关系？处理不当，可能会有race condition问题

Call trace:\
\[<ffffffc000350690>\] blk_mq_tag_to_rq+0x34/0xc4\
\[<ffffffc0003552d8>\] bt_for_each+0x78/0x114\
\[<ffffffc0003553e8>\] blk_mq_tag_busy_iter+0x74/0x84\
\[<ffffffc000351254>\] blk_mq_rq_timer+0x68/0x138\
\[<ffffffc0001014e0>\] call_timer_fn.isra.29+0x2c/0x9c\
\[<ffffffc000101788>\] run_timer_softirq+0x238/0x2d8\
\[<ffffffc0000a6ec0>\] \_\_do_softirq+0x110/0x25c\
\[<ffffffc0000a72c8>\] irq_exit+0x90/0xc0\
\[<ffffffc0000f17f8>\] \_\_handle_domain_irq+0x9c/0x10c\
\[<ffffffc00008250c>\] gic_handle_irq+0x50/0xb4\
Exception stack(0xffffffc07b24f840 to 0xffffffc07b24f970)\
f840: 7c20d500 ffffffc0 00000000 00000080 7b24f9b0 ffffffc0 00350b80 ffffffc0\
f860: 80000145 00000000 ffffffff 00000000 7b24fa50 ffffffc0 0000001b 00000000\
f880: 00302200 00000000 00000000 00000000 7c9aa000 ffffffc0 00000038 00000000\
f8a0: 00000000 01000000 00000000 00000000 5d004d80 ffffffc0 00000000 00000000\
f8c0: 60db1478 ffffffc0 00000000 00000000 7d66d068 ffffffc0 60db1000 ffffffc0\
f8e0: 74733b60 ffffffc0 60da5f08 ffffffc0 60d1a208 ffffffc0 38000000 0000185c\
f900: 001fbfdc ffffffc0 5b06a72c 00000072 3f800000 00000000 7c20d500 ffffffc0\
f920: 00000011 00000000 7ffe2e00 ffffffc0 7dbff880 ffffffc0 7c9b1000 ffffffc0\
f940: 7b24fae0 ffffffc0 5d004cc0 ffffffc0 0091b000 ffffffc0 00000009 00000000\
f960: 7d642200 ffffffc0 7b24f9b0 ffffffc0\
\[<ffffffc000085f80>\] el1_irq+0x80/0xf4\
\[<ffffffc000352ac4>\] blk_mq_map_request+0xbc/0x1d4\
\[<ffffffc000353fc8>\] blk_sq_make_request+0x80/0x29c\
\[<ffffffc000344608>\] generic_make_request+0xd8/0x154\
\[<ffffffc000344728>\] submit_bio+0xa4/0x1c0\
\[<ffffffc0001e9108>\] \_submit_bh+0x158/0x240\
\[<ffffffc0001e9214>\] submit_bh+0x24/0x2c\
\[<ffffffc00029cbe4>\] jbd2_journal_commit_transaction+0x9ac/0x1998\
\[<ffffffc0002a246c>\] kjournald2+0xd8/0x2ac\
\[<ffffffc0000c5d4c>\] kthread+0xe8/0x104\
Code: f9403000 f8765813 f9401a74 f9401e77 (f9406a95)\
---\[ end trace 75da3e8cdd880aeb \]---\
Kernel panic - not syncing: Fatal exception in interrupt

[回复](http://www.wowotech.net/irq_subsystem/soft-irq.html#comment-5725)

**[linuxer](http://www.wowotech.net/)**\
2017-06-23 10:02

@tczb：你的场景我猜是这样的：文件系统的日志线程愉快的运行，但是被一次timer的中断无情的插入了，因为blk的一个timer到期了，因此转到那个timer的handler中执行，在这个timer handler执行过程中，出现了严重的问题，导致kernel panic了。

你的场景包括一个内核线程和timer的中断上下文（hw irq和soft irq context），没有两个线程，当然，如果你认为中断上下文也是一种“线程”，那么就两个线程好了，哈哈，不过在linux的世界不是这么玩的。中断上下文是非常轻的，其实不算是线程。

[回复](http://www.wowotech.net/irq_subsystem/soft-irq.html#comment-5728)

**tczb**\
2017-06-23 10:35

@linuxer：谢谢linuxer回复；再麻烦下，还有个疑问，求详细解答下

1. 那个timer中断是怎么发起的呢？由谁 什么时候触发的，和上面的日志线程有关吗

不知道是不是这样：按文章的介绍，应该是硬件发起一个hard irq（这里可能是磁盘驱动），然后raise timer softirq？ timer超时是内核不断去检查么？

2. 明白，但如果中断上下文有访问共享资源，也是要类似线程，加锁吧？

______________________________________________________________________

## 不过在linux的世界不是这么玩的。中断上下文是非常轻的，其实不算是线程。

[回复](http://www.wowotech.net/irq_subsystem/soft-irq.html#comment-5729)

**[cracker](http://www.wowotech.net/)**\
2017-04-13 10:28

Hi  linuxer\
我有几个问题向您请教下：\
1、假如我在中断ISR中调用local_bh_disable  那是否软中断callback因为不满足!in_interrupt的条件就永远不会执行？还是可以后面调用raise_softirq（）通过ksoftirq去触发softirq？\
2、还有啊，执行invoke_sofrirq还有个条件就是检查local_softirq_pending,本地cpu是否有待处理的softirq是在哪里设置的？\
3、softirq执行时根据不同的irq number有不同的处理，是在kernel启动的过程中静态定义的callback，我现在不清楚这些callback大概做了什么事情？和我的dirver module的中断处理有什么关系？？？\
4、在 \_\_local_bh_enable_ip采用SOFTIRQ_DISABLE_OFFSET－1的做法去赤星pending的softirq，是为什么啊？内核不是提供ksoftirq线程去处理了吗？如果此时有更高优先级的进程要占用cpu，该进程会被延时处理吗？

问题太多了，不好意思~

[回复](http://www.wowotech.net/irq_subsystem/soft-irq.html#comment-5460)

**[cracker](http://www.wowotech.net/)**\
2017-04-13 16:35

@cracker：Hi  linuxer\
再追加问个问题，\
5、我tp模块设置的中断触发为边沿触发，当我在触摸tp时，tp会不断给cpu发送中断信号，在某一时刻按下power让系统睡眠，发现设备可以正常休眠，为什么啊？\
我的理解是top half阶段内核禁止抢占，如果不断的有中断产生，cpu此时不是在generic_handle_irq中循环处理中断吗？我看边沿触发是这样处理的

[回复](http://www.wowotech.net/irq_subsystem/soft-irq.html#comment-5463)

**cade**\
2017-03-21 16:21

## 例如：softirq、tasklet、workqueue或是直接创建一个kernel thread来执行bottom half（这在旧的kernel驱动中常见，现在，一个理智的driver厂商是不会这么做的）

@linuxer，很多次提到不创建单独的thread实现bh，除了会占用pid外，还会有哪些弊端？想知道这和workqueue的优劣是什么，workqueue有更高的优先级？\
Thanks～

[回复](http://www.wowotech.net/irq_subsystem/soft-irq.html#comment-5345)

**[linuxer](http://www.wowotech.net/)**\
2017-03-21 19:19

@cade：为什么linux kernel要创立workqueue这种机制？因为太多的驱动有这样的需求：有些中断处理代码需要在进程上下文中执行（例如：这些代码会导致sleep）。没有workqueue这种机制，各个驱动只能是自己创建内核线程，但是毫无疑问，这样会导致资源极大的浪费（进程id是其一，还有很多其他的资源，例如内存消耗也比较大，调度器需要调度更多的thread等），在这样的背景下才有了workqueue这种机制。因此，一般情况下，驱动应该使用workqueue机制而不是创建内核线程。

当然，本质上workqueue也是基于内核线程的，其实使用内核线程也不是不可以，不过需要考虑很多实现面的东西，例如是否在suspend过程中冻结该内核线程？该内核线程是绑定在一个CPU上还是为每一个cpu创建一个？当向该内核线程发送的请求太多的时候，是否串行化处理请求，还是创建更多的内核线程来处理？还有很多需要考量的东西。但是，很快你就会发现，其实这些在workqueue这样的机制中都已经有考量了。所以：为何要使用内核线程呢？内核已经有了一个“轮子”你为何还要重复创建“轮子”呢？

[回复](http://www.wowotech.net/irq_subsystem/soft-irq.html#comment-5349)

**cade**\
2017-03-27 16:35

@linuxer：@linuxer\
感谢！\
其实我针对的应用场景是这样的:VR设备上传感器对latency要求尽量小（kernel-->hal : \<1ms），throughput中断至少1000Hz。现在也没有找到好的测量手段，使用单独的thread总感觉比workqueue轻量，如果仅从自己模块角度而不考虑系统其他模块，单独的thread更容易控制也不会受其他模块阻塞。。。你说的哪些我还没有考虑到:-(

[回复](http://www.wowotech.net/irq_subsystem/soft-irq.html#comment-5383)

**panda**\
2017-04-01 18:01

@cade：在kernel/driver需要的地方增加trace_printk,将android应用层systrace数据和kernel的trace数据输出到同一个文件，然后用chrome打开trace文件，就可以很方便的测量出latency。

[回复](http://www.wowotech.net/irq_subsystem/soft-irq.html#comment-5418)

**lucky**\
2016-10-01 14:45

Hi linuxer,

@linuxer,您好：\
看了您的文章收获非常大，现有几个问题罗列出来希望指点一二，谢谢。

1. 本文中提到的这个流程图：关中断：mask and ack interrupt controller-------->ack NIC-------->copy FIFO to ram------>raise softirq------>unmask interrupt controller\
   =>最后一步unmask interrupt controller 这个是在中断处理程序执行完由硬件操作寄存器来进行操作的吗？\
   irq_exit\
   rcu_irq_exit\
   local_irq_save(flags);\
   local_irq_restore(flags);\
   既然流程第一步已经进行了执行了mask，为什么rcu_irq_exit 还会进行关闭中断的操作啊？硬中断处理中关闭中断的操作是硬件实现的？
1. 关于softirq的执行场景，目前我的理解：\
   1)是在中断上下文：\
   irq_exit\
   if (!in_interrupt() && local_softirq_pending())    //有pending softirq且没有disable_bh\
   invoke_softirq();\
   2)一个使用local_bh_disable和local_bh_enable来保护临界区的task，在local_bh_enable退出临界区时如有pending softirq则在该task的进程上下文执行。\
   3)由wakeup_softirqd 唤醒，在softirqd的进程上下文中执行。\
   综上所述的情况，我们说softirq和tasklet运行在中断上下文是不是有些不恰当？
1. 关于下面这段代码有个疑问想请教一下：\
   do_softirq(void)\
   local_irq_save(flags);              //save本地中断状态，关中断    ----- a\
   pending = local_softirq_pending();\
   \_\_do_softirq();\
   pending = local_softirq_pending();//获取 softirq pending 状态 ----- m\
   set_softirq_pending(0);     //清除pending标志\
   local_irq_enable        //开中断      ----- b\
   h->action(h);                              ----- c\
   local_irq_disable        //关中断      ----- d\
   pending = local_softirq_pending();//重新获取 softirq pending 状态 ----- n\
   if (pending) {\
   if (time_before(jiffies, end) && !need_resched() &&\
   --max_restart)\
   goto restart;\
   wakeup_softirqd();\
   }\
   local_irq_restore(flags);        //restore本地中断状态，开中断 ----- e

在c过程中又发生了中断打断了正在执行的softirq handler，又raise了softirq，那么中断结束又返回到被打断的softirq handler，继续执行m处获取的pending softirq，执行完毕后在n处获取新的\
pending softirq 跳转到restart那里重新处理，我的描述正确吗？

谢谢！

[回复](http://www.wowotech.net/irq_subsystem/soft-irq.html#comment-4639)

**[linuxer](http://www.wowotech.net/)**\
2016-10-07 09:41

@lucky：最后一步unmask interrupt controller 这个是在中断处理程序执行完由硬件操作寄存器来进行操作的吗？\
－－－－－－－－－－－\
这里说的mask或者ack某个IRQ number的操作是通过操作中断控制器（例如GIC）的寄存器来完成的。

既然流程第一步已经进行了执行了mask，为什么rcu_irq_exit 还会进行关闭中断的操作啊？硬中断处理中关闭中断的操作是硬件实现的？\
－－－－－－－－－－－－－－－\
rcu_irq_exit中的中断操作是成对的，分别是local_irq_save和local_irq_restore，都是操作的CPU中断，而mask操作是disable某个外设的IRQ，操作的是中断控制器，是不一样的。

综上所述的情况，我们说softirq和tasklet运行在中断上下文是不是有些不恰当？\
－－－－－－－－－－－－－－\
这是一个见仁见智的问题，不同的人对中断上下文理解不同，我对中断上下文的理解就是当前执行那一点的current thread和当前执行路径是没有任何的关系

关于第三个问题，你的描述是正确的。

[回复](http://www.wowotech.net/irq_subsystem/soft-irq.html#comment-4655)

1 [2](http://www.wowotech.net/irq_subsystem/soft-irq.html/comment-page-2#comments) [3](http://www.wowotech.net/irq_subsystem/soft-irq.html/comment-page-3#comments) [4](http://www.wowotech.net/irq_subsystem/soft-irq.html/comment-page-4#comments)

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

  - [一次触摸屏中断调试引发的深入探究](http://www.wowotech.net/linux_kenrel/437.html)
  - [Linux TTY framework(4)\_TTY driver](http://www.wowotech.net/tty_framework/tty_driver.html)
  - [致驱动工程师的一封信](http://www.wowotech.net/device_model/429.html)
  - [Linux 2.5.43版本的RCU实现（废弃）](http://www.wowotech.net/kernel_synchronization/Linux-2-5-43-RCU.html)
  - [智慧家庭之我见_多媒体篇](http://www.wowotech.net/tech_discuss/zhjt_dmt.html)

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

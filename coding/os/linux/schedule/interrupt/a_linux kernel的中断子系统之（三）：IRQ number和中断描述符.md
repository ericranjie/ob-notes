 作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2014-8-26 17:03 分类：[中断子系统](http://www.wowotech.net/sort/irq_subsystem)
# 一、前言

本文主要围绕IRQ number和中断描述符（interrupt descriptor）这两个概念描述通用中断处理过程。第二章主要描述基本概念，包括什么是IRQ number，什么是中断描述符等。第三章描述中断描述符数据结构的各个成员。第四章描述了初始化中断描述符相关的接口API。第五章描述中断描述符相关的接口API。
# 二、基本概念
## 1、通用中断的代码处理示意图

一个关于通用中断处理的示意图如下：

[![zhongduan](http://www.wowotech.net/content/uploadfile/201408/c1bc605fe5555c059b2e2e3196d2862a20140826100303.gif "zhongduan")](http://www.wowotech.net/content/uploadfile/201408/3e79044a4981e30f35e9b19fe44a441720140826100300.gif)

在linux kernel中，对于每一个外设的IRQ都用struct irq_desc来描述，我们称之中断描述符（struct irq_desc）。linux kernel中会有一个数据结构保存了关于所有IRQ的中断描述符信息，我们称之中断描述符DB（上图中红色框图内）。当发生中断后，首先获取触发中断的HW interupt ID，然后通过irq domain翻译成IRQ nuber，然后通过IRQ number就可以获取对应的中断描述符。调用中断描述符中的highlevel irq-events handler来进行中断处理就OK了。而highlevel irq-events handler主要进行下面两个操作：

（1）调用中断描述符的底层irq chip driver进行mask，ack等callback函数，进行interrupt flow control。
（2）调用该中断描述符上的action list中的specific handler（我们用这个术语来区分具体中断handler和high level的handler）。这个步骤不一定会执行，这是和中断描述符的当前状态相关，实际上，interrupt flow control是软件（设定一些标志位，软件根据标志位进行处理）和硬件（mask或者unmask interrupt controller等）一起控制完成的。

2、中断的打开和关闭

我们再来看看整个通用中断处理过程中的开关中断情况，开关中断有两种：

（1）开关local CPU的中断。对于UP，关闭CPU中断就关闭了一切，永远不会被抢占。对于SMP，实际上，没有关全局中断这一说，只能关闭local CPU（代码运行的那个CPU）

（2）控制interrupt controller，关闭某个IRQ number对应的中断。更准确的术语是mask或者unmask一个 IRQ。

本节主要描述的是第一种，也就是控制CPU的中断。当进入high level handler的时候，CPU的中断是关闭的（硬件在进入IRQ processor mode的时候设定的）。

对于外设的specific handler，旧的内核（2.6.35版本之前）认为有两种：slow handler和fast handle。在request irq的时候，对于fast handler，需要传递IRQF_DISABLED的参数，确保其中断处理过程中是关闭CPU的中断，因为是fast handler，执行很快，即便是关闭CPU中断不会影响系统的性能。但是，并不是每一种外设中断的handler都是那么快（例如磁盘），因此就有slow handler的概念，说明其在中断处理过程中会耗时比较长。对于这种情况，如果在整个specific handler中关闭CPU中断，对系统的performance会有影响。因此，对于slow handler，在从high level handler转入specific handler中间会根据IRQF_DISABLED这个flag来决定是否打开中断，具体代码如下（来自2.6.23内核）：

> irqreturn_t handle_IRQ_event(unsigned int irq, struct irqaction *action)  
> {  
>     ……
> 
>     if (!(action->flags & IRQF_DISABLED))  
>         local_irq_enable_in_hardirq();
> 
>     ……  
> }

如果没有设定IRQF_DISABLED（slow handler），则打开本CPU的中断。然而，随着软硬件技术的发展：

（1）硬件方面，CPU越来越快，原来slow handler也可以很快执行完毕

（2）软件方面，linux kernel提供了更多更好的bottom half的机制

因此，在新的内核中，比如3.14，IRQF_DISABLED被废弃了。我们可以思考一下，为何要有slow handler？每一个handler不都是应该迅速执行完毕，返回中断现场吗？此外，任意中断可以打断slow handler执行，从而导致中断嵌套加深，对内核栈也是考验。因此，新的内核中在interrupt specific handler中是全程关闭CPU中断的。

  
3、IRQ number

从CPU的角度看，无论外部的Interrupt controller的结构是多么复杂，I do not care，我只关心发生了一个指定外设的中断，需要调用相应的外设中断的handler就OK了。更准确的说是通用中断处理模块不关心外部interrupt controller的组织细节（电源管理模块当然要关注具体的设备（interrupt controller也是设备）的拓扑结构）。一言以蔽之，通用中断处理模块可以用一个线性的table来管理一个个的外部中断，这个表的每个元素就是一个irq描述符，在kernel中定义如下：

> struct irq_desc irq_desc[NR_IRQS] __cacheline_aligned_in_smp = {  
>     [0 ... NR_IRQS-1] = {  
>         .handle_irq    = handle_bad_irq,  
>         .depth        = 1,  
>         .lock        = __RAW_SPIN_LOCK_UNLOCKED(irq_desc->lock),  
>     }  
> };

系统中每一个连接外设的中断线（irq request line）用一个中断描述符来描述，每一个外设的interrupt request line分配一个中断号（irq number），系统中有多少个中断线（或者叫做中断源）就有多少个中断描述符（struct irq_desc）。NR_IRQS定义了该硬件平台IRQ的最大数目。

总之，一个静态定义的表格，irq number作为index，每个描述符都是紧密的排在一起，一切看起来很美好，但是现实很残酷的。有些系统可能会定义一个很大的NR_IRQS，但是只是想用其中的若干个，换句话说，这个静态定义的表格不是每个entry都是有效的，有空洞，如果使用静态定义的表格就会导致了内存很大的浪费。为什么会有这种需求？我猜是和各个interrupt controller硬件的interrupt ID映射到irq number的算法有关。在这种情况下，静态表格不适合了，我们改用一个radix tree来保存中断描述符（HW interrupt作为索引）。这时候，每一个中断描述符都是动态分配，然后插入到radix tree中。如果你的系统采用这种策略，那么需要打开CONFIG_SPARSE_IRQ选项。上面的示意图描述的是静态表格的中断描述符DB，如果打开CONFIG_SPARSE_IRQ选项，系统使用Radix tree来保存中断描述符DB，不过概念和静态表格是类似的。

此外，需要注意的是，在旧内核中，IRQ number和硬件的连接有一定的关系，但是，在引入irq domain后，IRQ number已经变成一个单纯的number，和硬件没有任何关系。

三、中断描述符数据结构

1、底层irq chip相关的数据结构

中断描述符中应该会包括底层irq chip相关的数据结构，linux kernel中把这些数据组织在一起，形成struct irq_data，具体代码如下：

> struct irq_data {  
>     u32            mask;－－－－－－－－－－TODO  
>     unsigned int        irq;－－－－－－－－IRQ number  
>     unsigned long        hwirq;－－－－－－－HW interrupt ID  
>     unsigned int        node;－－－－－－－NUMA node index  
>     unsigned int        state_use_accessors;－－－－－－－－底层状态，参考IRQD_xxxx  
>     struct irq_chip        *chip;－－－－－－－－－－该中断描述符对应的irq chip数据结构  
>     struct irq_domain    *domain;－－－－－－－－该中断描述符对应的irq domain数据结构  
>     void            *handler_data;－－－－－－－－和外设specific handler相关的私有数据  
>     void            *chip_data;－－－－－－－－－和中断控制器相关的私有数据  
>     struct msi_desc        *msi_desc;  
>     cpumask_var_t        affinity;－－－－－－－和irq affinity相关  
> };

中断有两种形态，一种就是直接通过signal相连，用电平或者边缘触发。另外一种是基于消息的，被称为MSI (Message Signaled Interrupts)。msi_desc是MSI类型的中断相关，这里不再描述。

node成员用来保存中断描述符的内存位于哪一个memory node上。 对于支持NUMA（Non Uniform Memory Access Architecture）的系统，其内存空间并不是均一的，而是被划分成不同的node，对于不同的memory node，CPU其访问速度是不一样的。如果一个IRQ大部分（或者固定）由某一个CPU处理，那么在动态分配中断描述符的时候，应该考虑将内存分配在该CPU访问速度比较快的memory node上。

2、irq chip数据结构

Interrupt controller描述符（struct irq_chip）包括了若干和具体Interrupt controller相关的callback函数，我们总结如下：

|   |   |
|---|---|
|成员名字|描述|
|name|该中断控制器的名字，用于/proc/interrupts中的显示|
|irq_startup|start up 指定的irq domain上的HW interrupt ID。如果不设定的话，default会被设定为enable函数|
|irq_shutdown|shutdown 指定的irq domain上的HW interrupt ID。如果不设定的话，default会被设定为disable函数|
|irq_enable|enable指定的irq domain上的HW interrupt ID。如果不设定的话，default会被设定为unmask函数|
|irq_disable|disable指定的irq domain上的HW interrupt ID。|
|irq_ack|和具体的硬件相关，有些中断控制器必须在Ack之后（清除pending的状态）才能接受到新的中断。|
|irq_mask|mask指定的irq domain上的HW interrupt ID|
|irq_mask_ack|mask并ack指定的irq domain上的HW interrupt ID。|
|irq_unmask|mask指定的irq domain上的HW interrupt ID|
|irq_eoi|有些interrupt controler（例如GIC）提供了这样的寄存器接口，让CPU可以通知interrupt controller，它已经处理完一个中断|
|irq_set_affinity|在SMP的情况下，可以通过该callback函数设定CPU affinity|
|irq_retrigger|重新触发一次中断，一般用在中断丢失的场景下。如果硬件不支持retrigger，可以使用软件的方法。|
|irq_set_type|设定指定的irq domain上的HW interrupt ID的触发方式，电平触发还是边缘触发|
|irq_set_wake|和电源管理相关，用来enable/disable指定的interrupt source作为唤醒的条件。|
|irq_bus_lock|有些interrupt controller是连接到慢速总线上（例如一个i2c接口的IO expander芯片），在访问这些芯片的时候需要lock住那个慢速bus（只能有一个client在使用I2C bus）|
|irq_bus_sync_unlock|unlock慢速总线|
|irq_suspend  <br>irq_resume  <br>irq_pm_shutdown|电源管理相关的callback函数|
|irq_calc_mask|TODO|
|irq_print_chip|/proc/interrupts中的信息显示|

3、中断描述符

在linux kernel中，使用struct irq_desc来描述一个外设的中断，我们称之中断描述符，具体代码如下：

> struct irq_desc {  
>     struct irq_data        irq_data;  
>     unsigned int __percpu    *kstat_irqs;－－－－－－IRQ的统计信息  
>     irq_flow_handler_t    handle_irq;－－－－－－－－（1）  
>     struct irqaction    *action; －－－－－－－－－－－（2）  
>     unsigned int        status_use_accessors;－－－－－中断描述符的状态，参考IRQ_xxxx  
>     unsigned int        core_internal_state__do_not_mess_with_it;－－－－（3）  
>     unsigned int        depth;－－－－－－－－－－（4）  
>     unsigned int        wake_depth;－－－－－－－－（5）  
>     unsigned int        irq_count;  －－－－－－－－－（6）  
>     unsigned long        last_unhandled;    
>     unsigned int        irqs_unhandled;  
>     raw_spinlock_t        lock;－－－－－－－－－－－（7）  
>     struct cpumask        *percpu_enabled;－－－－－－－（8）  
> #ifdef CONFIG_SMP  
>     const struct cpumask    *affinity_hint;－－－－和irq affinity相关，后续单独文档描述  
>     struct irq_affinity_notify *affinity_notify;  
> #ifdef CONFIG_GENERIC_PENDING_IRQ  
>     cpumask_var_t        pending_mask;  
> #endif  
> #endif  
>     unsigned long        threads_oneshot; －－－－－（9）  
>     atomic_t        threads_active;  
>     wait_queue_head_t       wait_for_threads;  
> #ifdef CONFIG_PROC_FS  
>     struct proc_dir_entry    *dir;－－－－－－－－该IRQ对应的proc接口  
> #endif  
>     int            parent_irq;  
>     struct module        *owner;  
>     const char        *name;  
> } ____cacheline_internodealigned_in_smp

（1）handle_irq就是highlevel irq-events handler，何谓high level？站在高处自然看不到细节。我认为high level是和specific相对，specific handler处理具体的事务，例如处理一个按键中断、处理一个磁盘中断。而high level则是对处理各种中断交互过程的一个抽象，根据下列硬件的不同：

（a）中断控制器

（b）IRQ trigger type

highlevel irq-events handler可以分成：

（a）处理电平触发类型的中断handler（handle_level_irq）

（b）处理边缘触发类型的中断handler（handle_edge_irq）

（c）处理简单类型的中断handler（handle_simple_irq）

（d）处理EOI类型的中断handler（handle_fasteoi_irq）

会另外有一份文档对high level handler进行更详细的描述。

（2）action指向一个struct irqaction的链表。如果一个interrupt request line允许共享，那么该链表中的成员可以是多个，否则，该链表只有一个节点。

（3）这个有着很长名字的符号core_internal_state__do_not_mess_with_it在具体使用的时候被被简化成istate，表示internal state。就像这个名字定义的那样，我们最好不要直接修改它。

> #define istate core_internal_state__do_not_mess_with_it

（4）我们可以通过enable和disable一个指定的IRQ来控制内核的并发，从而保护临界区的数据。对一个IRQ进行enable和disable的操作可以嵌套（当然一定要成对使用），depth是描述嵌套深度的信息。

（5）wake_depth是和电源管理中的wake up source相关。通过irq_set_irq_wake接口可以enable或者disable一个IRQ中断是否可以把系统从suspend状态唤醒。同样的，对一个IRQ进行wakeup source的enable和disable的操作可以嵌套（当然一定要成对使用），wake_depth是描述嵌套深度的信息。

（6）irq_count、last_unhandled和irqs_unhandled用于处理broken IRQ 的处理。所谓broken IRQ就是由于种种原因（例如错误firmware），IRQ handler没有定向到指定的IRQ上，当一个IRQ没有被处理的时候，kernel可以为这个没有被处理的handler启动scan过程，让系统中所有的handler来认领该IRQ。

（7）保护该中断描述符的spin lock。

（8）一个中断描述符可能会有两种情况，一种是该IRQ是global，一旦disable了该irq，那么对于所有的CPU而言都是disable的。还有一种情况，就是该IRQ是per CPU的，也就是说，在某个CPU上disable了该irq只是disable了本CPU的IRQ而已，其他的CPU仍然是enable的。percpu_enabled是一个描述该IRQ在各个CPU上是否enable成员。

（9）threads_oneshot、threads_active和wait_for_threads是和IRQ thread相关，后续文档会专门描述。

四、初始化相关的中断描述符的接口

1、静态定义的中断描述符初始化

> int __init early_irq_init(void)  
> {  
>     int count, i, node = first_online_node;  
>     struct irq_desc *desc;
> 
>     init_irq_default_affinity();
> 
>     desc = irq_desc;  
>     count = ARRAY_SIZE(irq_desc);
> 
>     for (i = 0; i < count; i++) {－－－遍历整个lookup table，对每一个entry进行初始化  
>         desc[i].kstat_irqs = alloc_percpu(unsigned int);－－－分配per cpu的irq统计信息需要的内存  
>         alloc_masks(&desc[i], GFP_KERNEL, node);－－－分配中断描述符中需要的cpu mask内存  
>         raw_spin_lock_init(&desc[i].lock);－－－初始化spin lock  
>         lockdep_set_class(&desc[i].lock, &irq_desc_lock_class);  
>         desc_set_defaults(i, &desc[i], node, NULL);－－－设定default值  
>     }  
>     return arch_early_irq_init();－－－调用arch相关的初始化函数  
> }

2、使用Radix tree的中断描述符初始化

> int __init early_irq_init(void)  
> {……
> 
>     initcnt = arch_probe_nr_irqs();－－－体系结构相关的代码来决定预先分配的中断描述符的个数
> 
>     if (initcnt > nr_irqs)－－－initcnt是需要在初始化的时候预分配的IRQ的个数  
>         nr_irqs = initcnt; －－－nr_irqs是当前系统中IRQ number的最大值
> 
>     for (i = 0; i < initcnt; i++) {－－－－－－－－预先分配initcnt个中断描述符  
>         desc = alloc_desc(i, node, NULL);－－－分配中断描述符  
>         set_bit(i, allocated_irqs);－－－设定已经alloc的flag  
>         irq_insert_desc(i, desc);－－－－－插入radix tree  
>     }  
>   ……  
> }

即便是配置了CONFIG_SPARSE_IRQ选项，在中断描述符初始化的时候，也有机会预先分配一定数量的IRQ。这个数量由arch_probe_nr_irqs决定，对于ARM而言，其arch_probe_nr_irqs定义如下：

> int __init arch_probe_nr_irqs(void)  
> {  
>     nr_irqs = machine_desc->nr_irqs ? machine_desc->nr_irqs : NR_IRQS;  
>     return nr_irqs;  
> }

3、分配和释放中断描述符

对于使用Radix tree来保存中断描述符DB的linux kernel，其中断描述符是动态分配的，可以使用irq_alloc_descs和irq_free_descs来分配和释放中断描述符。alloc_desc函数也会对中断描述符进行初始化，初始化的内容和静态定义的中断描述符初始化过程是一样的。最大可以分配的ID是IRQ_BITMAP_BITS，定义如下：

> #ifdef CONFIG_SPARSE_IRQ  
> # define IRQ_BITMAP_BITS    (NR_IRQS + 8196)－－－对于Radix tree，除了预分配的，还可以动态  
> #else                                                                             分配8196个中断描述符  
> # define IRQ_BITMAP_BITS    NR_IRQS－－－对于静态定义的，IRQ最大值就是NR_IRQS  
> #endif

五、和中断控制器相关的中断描述符的接口

这部分的接口主要有两类，irq_desc_get_xxx和irq_set_xxx，由于get接口API非常简单，这里不再描述，主要描述set类别的接口API。此外，还有一些locked版本的set接口API，定义为__irq_set_xxx，这些API的调用者应该已经持有保护irq desc的spinlock，因此，这些locked版本的接口没有中断描述符的spin lock进行操作。这些接口有自己特定的使用场合，这里也不详细描述了。

1、接口调用时机

kernel提供了若干的接口API可以让内核其他模块可以操作指定IRQ number的描述符结构。中断描述符中有很多的成员是和底层的中断控制器相关，例如：

（1）该中断描述符对应的irq chip

（2）该中断描述符对应的irq trigger type

（3）high level handler

在过去，系统中各个IRQ number是固定分配的，各个IRQ对应的中断控制器、触发类型等也都是清楚的，因此，一般都是在machine driver初始化的时候一次性的进行设定。machine driver的初始化过程会包括中断系统的初始化，在machine driver的中断初始化函数中，会调用本文定义的这些接口对各个IRQ number对应的中断描述符进行irq chip、触发类型的设定。

在引入了device tree、动态分配IRQ number以及irq domain这些概念之后，这些接口的调用时机移到各个中断控制器的初始化以及各个具体硬件驱动初始化过程中，具体如下：

（1）各个中断控制器定义好自己的struct irq_domain_ops callback函数，主要是map和translate函数。

（2）在各个具体的硬件驱动初始化过程中，通过device tree系统可以知道自己的中断信息（连接到哪一个interrupt controler、使用该interrupt controller的那个HW interrupt ID，trigger type为何），调用对应的irq domain的translate进行翻译、解析。之后可以动态申请一个IRQ number并和该硬件外设的HW interrupt ID进行映射，调用irq domain对应的map函数。在map函数中，可以调用本节定义的接口进行中断描述符底层interrupt controller相关信息的设定。

2、irq_set_chip

这个接口函数用来设定中断描述符中desc->irq_data.chip成员，具体代码如下：

> int irq_set_chip(unsigned int irq, struct irq_chip *chip)  
> {  
>     unsigned long flags;  
>     struct irq_desc *desc = irq_get_desc_lock(irq, &flags, 0); －－－－（1）
> 
>     desc->irq_data.chip = chip;  
>     irq_put_desc_unlock(desc, flags);－－－－－－－－－－－－－－－（2）  
>   
>     irq_reserve_irq(irq);－－－－－－－－－－－－－－－－－－－－－－（3）  
>     return 0;  
> }

（1）获取irq number对应的中断描述符。这里用关闭中断＋spin lock来保护中断描述符，flag中就是保存的关闭中断之前的状态flag，后面在（2）中会恢复中断flag。

（3）前面我们说过，irq number有静态表格定义的，也有radix tree类型的。对于静态定义的中断描述符，没有alloc的概念。但是，对于radix tree类型，需要首先irq_alloc_desc或者irq_alloc_descs来分配一个或者一组IRQ number，在这些alloc函数值，就会set那些那些已经分配的IRQ。对于静态表格而言，其IRQ没有分配，因此，这里通过irq_reserve_irq函数标识该IRQ已经分配，虽然对于CONFIG_SPARSE_IRQ（使用radix tree）的配置而言，这个操作重复了（在alloc的时候已经设定了）。

3、irq_set_irq_type

这个函数是用来设定该irq number的trigger type的。

> int irq_set_irq_type(unsigned int irq, unsigned int type)  
> {  
>     unsigned long flags;  
>     struct irq_desc *desc = irq_get_desc_buslock(irq, &flags, IRQ_GET_DESC_CHECK_GLOBAL);  
>     int ret = 0;
> 
>     type &= IRQ_TYPE_SENSE_MASK;  
>     ret = __irq_set_trigger(desc, irq, type);  
>     irq_put_desc_busunlock(desc, flags);  
>     return ret;  
> }

来到这个接口函数，第一个问题就是：为何irq_set_chip接口函数使用irq_get_desc_lock来获取中断描述符，而irq_set_irq_type这个函数却需要irq_get_desc_buslock呢？其实也很简单，irq_set_chip不需要访问底层的irq chip（也就是interrupt controller），但是irq_set_irq_type需要。设定一个IRQ的trigger type最终要调用desc->irq_data.chip->irq_set_type函数对底层的interrupt controller进行设定。这时候，问题来了，对于嵌入SOC内部的interrupt controller，当然没有问题，因为访问这些中断控制器的寄存器memory map到了CPU的地址空间，访问非常的快，因此，关闭中断＋spin lock来保护中断描述符当然没有问题，但是，如果该interrupt controller是一个I2C接口的IO expander芯片（这类芯片是扩展的IO，也可以提供中断功能），这时，让其他CPU进行spin操作太浪费CPU时间了（bus操作太慢了，会spin很久的）。肿么办？当然只能是用其他方法lock住bus了（例如mutex，具体实现是和irq chip中的irq_bus_lock实现相关）。一旦lock住了slow bus，然后就可以关闭中断了（中断状态保存在flag中）。

解决了bus lock的疑问后，还有一个看起来奇奇怪怪的宏：IRQ_GET_DESC_CHECK_GLOBAL。为何在irq_set_chip函数中不设定检查（check的参数是0），而在irq_set_irq_type接口函数中要设定global的check，到底是什么意思呢？既然要检查，那么检查什么呢？和“global”对应的不是local而是“per CPU”，内核中的宏定义是：IRQ_GET_DESC_CHECK_PERCPU。SMP情况下，从系统角度看，中断有两种形态（或者叫mode）：

（1）1-N mode。只有1个processor处理中断

（2）N-N mode。所有的processor都是独立的收到中断，如果有N个processor收到中断，那么就有N个处理器来处理该中断。

听起来有些抽象，我们还是用GIC作为例子来具体描述。在GIC中，SPI使用1-N模型，而PPI和SGI使用N-N模型。对于SPI，由于采用了1-N模型，系统（硬件加上软件）必须保证一个中断被一个CPU处理。对于GIC，一个SPI的中断可以trigger多个CPU的interrupt line（如果Distributor中的Interrupt Processor Targets Registers有多个bit被设定），但是，该interrupt source和CPU的接口寄存器（例如ack register）只有一套，也就是说，这些寄存器接口是全局的，是global的，一旦一个CPU ack（读Interrupt Acknowledge Register，获取interrupt ID）了该中断，那么其他的CPU看到的该interupt source的状态也是已经ack的状态。在这种情况下，如果第二个CPU ack该中断的时候，将获取一个spurious interrupt ID。

对于PPI或者SGI，使用N-N mode，其interrupt source的寄存器是per CPU的，也就是每个CPU都有自己的、针对该interrupt source的寄存器接口（这些寄存器叫做banked register）。一个CPU 清除了该interrupt source的pending状态，其他的CPU感知不到这个变化，它们仍然认为该中断是pending的。

对于irq_set_irq_type这个接口函数，它是for 1-N mode的interrupt source使用的。如果底层设定该interrupt是per CPU的，那么irq_set_irq_type要返回错误。

4、irq_set_chip_data

每个irq chip总有自己私有的数据，我们称之chip data。具体设定chip data的代码如下：

> int irq_set_chip_data(unsigned int irq, void *data)  
> {  
>     unsigned long flags;  
>     struct irq_desc *desc = irq_get_desc_lock(irq, &flags, 0);   
>     desc->irq_data.chip_data = data;  
>     irq_put_desc_unlock(desc, flags);  
>     return 0;  
> }

多么清晰、多么明了，需要文字继续描述吗？

5、设定high level handler

这是中断处理的核心内容，__irq_set_handler就是设定high level handler的接口函数，不过一般不会直接调用，而是通过irq_set_chip_and_handler_name或者irq_set_chip_and_handler来进行设定。具体代码如下：

> void __irq_set_handler(unsigned int irq, irq_flow_handler_t handle, int is_chained, const char *name)  
> {  
>     unsigned long flags;  
>     struct irq_desc *desc = irq_get_desc_buslock(irq, &flags, 0);
> 
> ……  
>     desc->handle_irq = handle;  
>     desc->name = name;
> 
>     if (handle != handle_bad_irq && is_chained) {  
>         irq_settings_set_noprobe(desc);  
>         irq_settings_set_norequest(desc);  
>         irq_settings_set_nothread(desc);  
>         irq_startup(desc, true);  
>     }  
> out:  
>     irq_put_desc_busunlock(desc, flags);  
> }

理解这个函数的关键是在is_chained这个参数。这个参数是用在interrupt级联的情况下。例如中断控制器B级联到中断控制器A的第x个interrupt source上。那么对于A上的x这个interrupt而言，在设定其IRQ handler参数的时候要设定is_chained参数等于1，由于这个interrupt source用于级联，因此不能probe、不能被request（已经被中断控制器B使用了），不能被threaded（具体中断线程化的概念在其他文档中描述）

  

_原创文章，转发请注明出处。蜗窝科技。_[http://www.wowotech.net/linux_kenrel/interrupt_descriptor.html](http://www.wowotech.net/linux_kenrel/interrupt_descriptor.html)

标签: [irq](http://www.wowotech.net/tag/irq) [中断子系统](http://www.wowotech.net/tag/%E4%B8%AD%E6%96%AD%E5%AD%90%E7%B3%BB%E7%BB%9F) [中断描述符](http://www.wowotech.net/tag/%E4%B8%AD%E6%96%AD%E6%8F%8F%E8%BF%B0%E7%AC%A6)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [linux kernel的中断子系统之（四）：High level irq event handler](http://www.wowotech.net/irq_subsystem/High_level_irq_event_handler.html) | [Linux电源管理(6)_Generic PM之Suspend功能](http://www.wowotech.net/pm_subsystem/suspend_and_resume.html)»

**评论：**

**立志学linux**  
2023-05-26 20:18

有没有大佬知道，high level handler 和 通过request_threaded_irq注册的那个的handler（也就是irq_desc->irqaction->handler）是什么关系呀

[回复](http://www.wowotech.net/irq_subsystem/interrupt_descriptor.html#comment-8786)

**5song**  
2023-05-29 15:44

@立志学linux：一般handle_irq()会进行如下操作，可参考(kernel/irq/chip.c: void handle_edge_irq(struct irq_desc *desc))：  
  
调用中断描述符中的底层chip driver进行mask以及中断ACK回调，进行IRQ flow control  
调用handle_irq_event向上发送中断事件，遍历描述符里的action list，并执行action->handler回调（request_threaded_irq可以绑定该handler）, 执行完action->handler后，要么中断已经处理完成，要么就执行__irq_wake_thread唤醒我们绑定的  
irq thread处理函数(也就是我们常说的上半部处理函数)

[回复](http://www.wowotech.net/irq_subsystem/interrupt_descriptor.html#comment-8787)

**刘**  
2022-10-26 20:29

第一个问题就是：为何irq_set_chip接口函数使用irq_get_desc_lock来获取中断描述符，而irq_set_irq_type这个函数却需要irq_get_desc_buslock呢？其实也很简单，irq_set_chip不需要访问底层的irq chip（也就是interrupt controller），但是irq_set_irq_type需要。设定一个IRQ的trigger type最终要调用desc->irq_data.chip->irq_set_type函数对底层的interrupt controller进行设定。这时候，问题来了，对于嵌入SOC内部的interrupt controller，当然没有问题，因为访问这些中断控制器的寄存器memory map到了CPU的地址空间，访问非常的快，因此，关闭中断＋spin lock来保护中断描述符当然没有问题，但是，如果该interrupt controller是一个I2C接口的IO expander芯片（这类芯片是扩展的IO，也可以提供中断功能），这时，让其他CPU进行spin操作太浪费CPU时间了（bus操作太慢了，会spin很久的）。肿么办？当然只能是用其他方法lock住bus了（例如mutex，具体实现是和irq chip中的irq_bus_lock实现相关）。一旦lock住了slow bus，然后就可以关闭中断了（中断状态保存在flag中）。  
  
irq_get_desc_buslock  lock住了bus时也有调用raw_spin_lock_irqsave  
struct irq_desc *  
__irq_get_desc_lock(unsigned int irq, unsigned long *flags, bool bus,  
            unsigned int check)  
{  
    struct irq_desc *desc = irq_to_desc(irq);  
  
    if (desc) {  
        if (check & _IRQ_DESC_CHECK) {  
            if ((check & _IRQ_DESC_PERCPU) &&  
                !irq_settings_is_per_cpu_devid(desc))  
                return NULL;  
  
            if (!(check & _IRQ_DESC_PERCPU) &&  
                irq_settings_is_per_cpu_devid(desc))  
                return NULL;  
        }  
  
        if (bus)  
            chip_bus_lock(desc);  
        raw_spin_lock_irqsave(&desc->lock, *flags);  
    }  
    return desc;  
}

[回复](http://www.wowotech.net/irq_subsystem/interrupt_descriptor.html#comment-8694)

**Toni**  
2022-03-24 23:11

有没有关于PCIE MSI中断的一些介绍？

[回复](http://www.wowotech.net/irq_subsystem/interrupt_descriptor.html#comment-8579)

**Alexander**  
2022-01-16 00:32

你好，请问x86平台的irq_dmain怎么和io apci机制连接起来的？一直搞不懂

[回复](http://www.wowotech.net/irq_subsystem/interrupt_descriptor.html#comment-8492)

**poolloop**  
2021-08-08 17:31

您好，在中断相关的文章中，偶尔会看到“mask”这个动作。  
我的问题是，一般所说的“mask这个动作”是全局的（影响所有的cpu）呢，还是仅仅影响当前local cpu的呢？  
在不同的架构下（arm/x86）是否都有一样的意义，谢谢！

[回复](http://www.wowotech.net/irq_subsystem/interrupt_descriptor.html#comment-8265)

**ailon**  
2018-07-13 17:21

在多路服务器中，每个CPU都有一个uart控制器，在CPU角度来看每个uart的irq都是一样的，那我应该如何区分到底是哪个一个uart控制器来的中断呢，现在的内核版本是不是没有对这方面的考虑

[回复](http://www.wowotech.net/irq_subsystem/interrupt_descriptor.html#comment-6845)

**[Lambert](http://l2h.site/)**  
2018-12-28 17:30

@ailon：说下个人理解：  
  
每个UART的中断输出都是连接到同一个中断控制器吗？  
如果是的话，应该是不同的中断引脚，其硬件中断编号必定不同。  
  
如果不是，中断控制器是多个，那么中断Domain不同。  
  
更甚，多个UART share一个中断Pin，那么就要看中断控制器本身有没有区分方法，知道是哪一路过来的中断

[回复](http://www.wowotech.net/irq_subsystem/interrupt_descriptor.html#comment-7114)

**magichouse**  
2017-10-17 15:52

@linuxer  
本节主要描述的是第一种，也就是控制CPU的中断。当进入high level handler的时候，CPU的中断是关闭的（硬件在进入IRQ processor mode的时候设定的）。  
------------------------------------------------  
hi，请教一下，进入high level handler时，cpu的中断是关闭的，这一点是硬件设定的？还是软件设定的？  
再次打开的时机是什么？

[回复](http://www.wowotech.net/irq_subsystem/interrupt_descriptor.html#comment-6117)

**[wowo](http://www.wowotech.net/)**  
2017-10-18 11:44

@magichouse：这个比较严谨的回答是，要视具体的arch而定，不过对arm来说（这里以arm64为例），是arm的core（硬件）做的，具体可参考下面的资料：  
  
DDI0487A_d_armv8_arm.pdf  
======================================  
...  
  
D1.10 Exception entry  
  
On taking an exception to AArch64 state:  
    ...  
    • All of PSTATE.{D, A, I, F} are set to 1. See Process state, PSTATE on page D1-1434.  
    ...  
  
type ProcState is (  
    ...  
    bits (1) I, // IRQ mask bit  
    ...  
)

[回复](http://www.wowotech.net/irq_subsystem/interrupt_descriptor.html#comment-6120)

**[linuxer_fy](http://no%20home%20page%20yet/)**  
2017-10-13 14:52

我有个疑问。不知道是放到(二)IRQ Domain一文中，还是这篇中。估且放这里吧。  
ARM的不是很熟悉。就X86上来说，多级联interrupt controller中（例A和B），它们是和CPU通过系统数据总线来传递HW interrupt ID的。又由于每个irq domain可能有重复编号的ID，所以才有了domain来抽象和识别。但是  
(1)A和B是独立各自向CPU传递HW ID的话，如果A和B同时发生中断，需要传递HW ID,如何解决冲突。硬件处理？此情况下是否会有中断丢失？  
(2)在CPU只收到HW interrupt ID的情况下，系统还要配合HW情报去翻译成合适的irq number,  
也就是每次都要“遍历”整个irq domain list，且还要能处理不同类型的interrupt controller芯片  
(3)我记得X86上两片8259a级联时(每片8259支持0~7个IRQ Line)，可以直接产生0~15个中断号，即2片8259可以直接组合在数据总线上将0~15中断号(相当于HW interrupt ID)传递给CPU，此时CPU并不需要再去翻译成irq number了，因为HW interrupt ID是唯一的，不重复。  
如果2片 8259设计成domain的方式，那就要系统额外去识别：  
(二)IRQ Domain一文中最后：  
“2、具体如何在中断处理过程中，将HW interrupt ID转成IRQ number  
（2）根据HW 寄存器信息和irq domain信息获取HW interrupt ID  
”  
这里“根据HW寄存器信息”就是看8259A芯片的中断请求寄存器中相应的bit位是否有中断存在？

[回复](http://www.wowotech.net/irq_subsystem/interrupt_descriptor.html#comment-6106)

**于长河**  
2017-06-26 09:53

您好linuxer,我有个疑问：在local_irq_disable中最终是对cpsr寄存器的I位置位，那么这个local是怎样体现出来的呢？是不是在一个多核cpu中其中每一核都有一个独立的cpsr寄存器

[回复](http://www.wowotech.net/irq_subsystem/interrupt_descriptor.html#comment-5732)

**[linuxer](http://www.wowotech.net/)**  
2017-06-26 11:53

@于长河：当然了，cpsr寄存器是per cpu的，每一个cpu core都有自己的cpsr寄存器。

[回复](http://www.wowotech.net/irq_subsystem/interrupt_descriptor.html#comment-5734)

**于长河**  
2017-06-26 14:11

@linuxer：明白了，谢谢

[回复](http://www.wowotech.net/irq_subsystem/interrupt_descriptor.html#comment-5735)

**善昆**  
2017-04-26 17:46

在已经知道hw irq的时候，我想绕过dts方式，有办法获取虚拟irq号吗？

[回复](http://www.wowotech.net/irq_subsystem/interrupt_descriptor.html#comment-5485)

**[linuxer](http://www.wowotech.net/)**  
2017-04-26 19:14

@善昆：当然可以，自己分配一个就OK了，当然，你需要自己创建mapping。当然，不建议用这么底层的接口。

[回复](http://www.wowotech.net/irq_subsystem/interrupt_descriptor.html#comment-5487)

**善昆**  
2017-04-26 21:00

@linuxer：因为特殊需求不能重编译内核，所以没有办法通过修改dts入手，也是因为追查到了irq_create_mapping函数，才阅读了本篇文章。但是这个函数domain参数一直不知道该如何获取，本想自己创建一个irq_desc,但是发现alloc_desc函数还是需要node...  
不知道有没有方法获得这个domain,或许可以通过已经存在与dts中断里设备找到CIG的domain吧

[回复](http://www.wowotech.net/irq_subsystem/interrupt_descriptor.html#comment-5488)

1 [2](http://www.wowotech.net/irq_subsystem/interrupt_descriptor.html/comment-page-2#comments) [3](http://www.wowotech.net/irq_subsystem/interrupt_descriptor.html/comment-page-3#comments)

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
    
    - [ARMv8之Observability](http://www.wowotech.net/armv8a_arch/Observability.html)
    - [ARM64的启动过程之（四）：打开MMU](http://www.wowotech.net/armv8a_arch/turn-on-mmu.html)
    - [Linux 3.18U盘无法正确使用](http://www.wowotech.net/226.html)
    - [spin_lock最简单用法。](http://www.wowotech.net/164.html)
    - [Linux电源管理(4)_Power Management Interface](http://www.wowotech.net/pm_subsystem/pm_interface.html)
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
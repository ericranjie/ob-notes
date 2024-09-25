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

作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2015-7-2 18:10 分类：[中断子系统](http://www.wowotech.net/sort/irq_subsystem)

一、前言

对于中断处理而言，linux将其分成了两个部分，一个叫做中断handler（top half），属于不那么紧急需要处理的事情被推迟执行，我们称之deferable task，或者叫做bottom half，。具体如何推迟执行分成下面几种情况：

1、推迟到top half执行完毕

2、推迟到某个指定的时间片（例如40ms）之后执行

3、推迟到某个内核线程被调度的时候执行

对于第一种情况，内核中的机制包括[softirq机制](http://www.wowotech.net/irq_subsystem/soft-irq.html#2107)和tasklet机制。第二种情况是属于softirq机制的一种应用场景（timer类型的softirq），在本站的时间子系统的系列文档中会描述。第三种情况主要包括threaded irq handler以及通用的workqueue机制，当然也包括自己创建该驱动专属kernel thread（不推荐使用）。本文主要描述tasklet这种机制，第二章描述一些背景知识和和tasklet的思考，第三章结合代码描述tasklet的原理。

注：本文中的linux kernel的版本是4.0

二、为什么需要tasklet？

1、基本的思考

我们的驱动程序或者内核模块真的需要tasklet吗？每个人都有自己的看法。我们先抛开linux kernel中的机制，首先进行一番逻辑思考。

将中断处理分成top half（cpu和外设之间的交互，获取状态，ack状态，收发数据等）和bottom half（后段的数据处理）已经深入人心，对于任何的OS都一样，将不那么紧急的事情推迟到bottom half中执行是OK的，具体如何推迟执行分成两种类型：有具体时间要求的（对应linux kernel中的低精度timer和高精度timer）和没有具体时间要求的。对于没有具体时间要求的又可以分成两种：

（1）越快越好型，这种实际上是有性能要求的，除了中断top half可以抢占其执行，其他的进程上下文（无论该进程的优先级多么的高）是不会影响其执行的，一言以蔽之，在不影响中断延迟的情况下，OS会尽快处理。

（2）随遇而安型。这种属于那种没有性能需求的，其调度执行依赖系统的调度器。

本质上讲，越快越好型的bottom half不应该太多，而且tasklet的callback函数不能执行时间过长，否则会产生进程调度延迟过大的现象，甚至是非常长而且不确定的延迟，对real time的系统会产生很坏的影响。

2、对linux中的bottom half机制的思考

在linux kernel中，“越快越好型”有两种，softirq和tasklet，“随遇而安型”也有两种，workqueue和threaded irq handler。“越快越好型”能否只留下一个softirq呢？对于崇尚简单就是美的程序员当然希望如此。为了回答这个问题，我们先看看tasklet对于softirq而言有哪些好处：

（1）tasklet可以动态分配，也可以静态分配，数量不限。

（2）同一种tasklet在多个cpu上也不会并行执行，这使得程序员在撰写tasklet function的时候比较方便，减少了对并发的考虑（当然损失了性能）。

对于第一种好处，其实也就是为乱用tasklet打开了方便之门，很多撰写驱动的软件工程师不会仔细考量其driver是否有性能需求就直接使用了tasklet机制。对于第二种好处，本身考虑并发就是软件工程师的职责。因此，看起来tasklet并没有引入特别的好处，而且和softirq一样，都不能sleep，限制了handler撰写的方便性，看起来其实并没有存在的必要。在4.0 kernel的代码中，grep一下tasklet的使用，实际上是一个很长的列表，只要对这些使用进行简单的归类就可以删除对tasklet的使用。对于那些有性能需求的，可以考虑并入softirq，其他的可以考虑使用workqueue来取代。Steven Rostedt试图进行这方面的尝试（[http://lwn.net/Articles/239484/](http://lwn.net/Articles/239484/ "http://lwn.net/Articles/239484/")），不过这个patch始终未能进入main line。

三、tasklet的基本原理

1、如何抽象一个tasklet

内核中用下面的数据结构来表示tasklet：

> struct tasklet_struct  
> {  
>     struct tasklet_struct *next;  
>     unsigned long state;  
>     atomic_t count;  
>     void (*func)(unsigned long);  
>     unsigned long data;  
> };

每个cpu都会维护一个链表，将本cpu需要处理的tasklet管理起来，next这个成员指向了该链表中的下一个tasklet。func和data成员描述了该tasklet的callback函数，func是调用函数，data是传递给func的参数。state成员表示该tasklet的状态，TASKLET_STATE_SCHED表示该tasklet以及被调度到某个CPU上执行，TASKLET_STATE_RUN表示该tasklet正在某个cpu上执行。count成员是和enable或者disable该tasklet的状态相关，如果count等于0那么该tasklet是处于enable的，如果大于0，表示该tasklet是disable的。在[softirq文档](http://www.wowotech.net/irq_subsystem/soft-irq.html#2117)中，我们知道local_bh_disable/enable函数就是用来disable/enable bottom half的，这里就包括softirq和tasklet。但是，有的时候内核同步的场景不需disable所有的softirq和tasklet，而仅仅是disable该tasklet，这时候，tasklet_disable和tasklet_enable就派上用场了。

> static inline void tasklet_disable(struct tasklet_struct *t)  
> {  
>     tasklet_disable_nosync(t);－－－－－－－给tasklet的count加一  
>     tasklet_unlock_wait(t);－－－－－如果该tasklet处于running状态，那么需要等到该tasklet执行完毕  
>     smp_mb();  
> }
> 
> static inline void tasklet_enable(struct tasklet_struct *t)  
> {  
>     smp_mb__before_atomic();  
>     atomic_dec(&t->count);－－－－－－－给tasklet的count减一  
> }

tasklet_disable和tasklet_enable支持嵌套，但是需要成对使用。

2、系统如何管理tasklet？

系统中的每个cpu都会维护一个tasklet的链表，定义如下：

> static DEFINE_PER_CPU(struct tasklet_head, tasklet_vec);  
> static DEFINE_PER_CPU(struct tasklet_head, tasklet_hi_vec);

linux kernel中，和tasklet相关的softirq有两项，HI_SOFTIRQ用于高优先级的tasklet，TASKLET_SOFTIRQ用于普通的tasklet。对于softirq而言，优先级就是出现在softirq pending register（__softirq_pending）中的先后顺序，位于bit 0拥有最高的优先级，也就是说，如果有多个不同类型的softirq同时触发，那么执行的先后顺序依赖在softirq pending register的位置，kernel总是从右向左依次判断是否置位，如果置位则执行。HI_SOFTIRQ占据了bit 0，其优先级甚至高过timer，需要慎用（实际上，我grep了内核代码，似乎没有发现对HI_SOFTIRQ的使用）。当然HI_SOFTIRQ和TASKLET_SOFTIRQ的机理是一样的，因此本文只讨论TASKLET_SOFTIRQ，大家可以举一反三。

3、如何定义一个tasklet？

你可以用下面的宏定义来静态定义tasklet：

> #define DECLARE_TASKLET(name, func, data) \  
> struct tasklet_struct name = { NULL, 0, ATOMIC_INIT(0), func, data }
> 
> #define DECLARE_TASKLET_DISABLED(name, func, data) \  
> struct tasklet_struct name = { NULL, 0, ATOMIC_INIT(1), func, data }

这两个宏都可以静态定义一个struct tasklet_struct的变量，只不过初始化后的tasklet一个是处于eable状态，一个处于disable状态的。当然，也可以动态分配tasklet，然后调用tasklet_init来初始化该tasklet。

4、如何调度一个tasklet

为了调度一个tasklet执行，我们可以使用tasklet_schedule这个接口：

> static inline void tasklet_schedule(struct tasklet_struct *t)  
> {  
>     if (!test_and_set_bit(TASKLET_STATE_SCHED, &t->state))  
>         __tasklet_schedule(t);  
> }

程序在多个上下文中可以多次调度同一个tasklet执行（也可能来自多个cpu core），不过实际上该tasklet只会一次挂入首次调度到的那个cpu的tasklet链表，也就是说，即便是多次调用tasklet_schedule，实际上tasklet只会挂入一个指定CPU的tasklet队列中（而且只会挂入一次），也就是说只会调度一次执行。这是通过TASKLET_STATE_SCHED这个flag来完成的，我们可以用下面的图片来描述：

[![tasklet](http://www.wowotech.net/content/uploadfile/201507/2f814e61e193b5a0a694ef3673b4f60620150702100946.gif "tasklet")](http://www.wowotech.net/content/uploadfile/201507/06299a79874281afa05198f9d72a87da20150702100845.gif)

我们假设HW block A的驱动使用的tasklet机制并且在中断handler（top half）中将静态定义的tasklet（这个tasklet是各个cpu共享的，不是per cpu的）调度执行（也就是调用tasklet_schedule函数）。当HW block A检测到硬件的动作（例如接收FIFO中数据达到半满）就会触发IRQ line上的电平或者边缘信号，GIC检测到该信号会将该中断分发给某个CPU执行其top half handler，我们假设这次是cpu0，因此该driver的tasklet被挂入CPU0对应的tasklet链表（tasklet_vec）并将state的状态设定为TASKLET_STATE_SCHED。HW block A的驱动中的tasklet虽已调度，但是没有执行，如果这时候，硬件又一次触发中断并在cpu1上执行，虽然tasklet_schedule函数被再次调用，但是由于TASKLET_STATE_SCHED已经设定，因此不会将HW block A的驱动中的这个tasklet挂入cpu1的tasklet链表中。

下面我们再仔细研究一下底层的__tasklet_schedule函数：

> void __tasklet_schedule(struct tasklet_struct *t)  
> {  
>     unsigned long flags;
> 
>     local_irq_save(flags);－－－－－－－－－－－－－－－－－－－（1）  
>     t->next = NULL;－－－－－－－－－－－－－－－－－－－－－（2）  
>     *__this_cpu_read(tasklet_vec.tail) = t;  
>     __this_cpu_write(tasklet_vec.tail, &(t->next));  
>     raise_softirq_irqoff(TASKLET_SOFTIRQ);－－－－－－－－－－（3）  
>     local_irq_restore(flags);  
> }

（1）下面的链表操作是per-cpu的，因此这里禁止本地中断就可以拦截所有的并发。

（2）这里的三行代码就是将一个tasklet挂入链表的尾部

（3）raise TASKLET_SOFTIRQ类型的softirq。

5、在什么时机会执行tasklet？

上面描述了tasklet的调度，当然调度tasklet不等于执行tasklet，系统会在适合的时间点执行tasklet callback function。由于tasklet是基于softirq的，因此，我们首先总结一下softirq的执行场景：

（1）在中断返回用户空间（进程上下文）的时候，如果有pending的softirq，那么将执行该softirq的处理函数。这里限定了中断返回用户空间也就是意味着限制了下面两个场景的softirq被触发执行：

    （a）中断返回hard interrupt context，也就是中断嵌套的场景

    （b）中断返回software interrupt context，也就是中断抢占软中断上下文的场景

（2）上面的描述缺少了一种场景：中断返回内核态的进程上下文的场景，这里我们需要详细说明。进程上下文中调用local_bh_enable的时候，如果有pending的softirq，那么将执行该softirq的处理函数。由于内核同步的要求，进程上下文中有可能会调用local_bh_enable/disable来保护临界区。在临界区代码执行过程中，中断随时会到来，抢占该进程（内核态）的执行（注意：这里只是disable了bottom half，没有禁止中断）。在这种情况下，中断返回的时候是否会执行softirq handler呢？当然不会，我们disable了bottom half的执行，也就是意味着不能执行softirq handler，但是本质上bottom half应该比进程上下文有更高的优先级，一旦条件允许，要立刻抢占进程上下文的执行，因此，当立刻离开临界区，调用local_bh_enable的时候，会检查softirq pending，如果bottom half处于enable的状态，pending的softirq handler会被执行。

（3）系统太繁忙了，不过的产生中断，raise softirq，由于bottom half的优先级高，从而导致进程无法调度执行。这种情况下，softirq会推迟到softirqd这个内核线程中去执行。

对于TASKLET_SOFTIRQ类型的softirq，其handler是tasklet_action，我们来看看各个tasklet是如何执行的：

> static void tasklet_action(struct softirq_action *a)  
> {  
>     struct tasklet_struct *list;
> 
>     local_irq_disable();－－－－－－－－－－－－－－－－－－－－－－－－－－（1）  
>     list = __this_cpu_read(tasklet_vec.head);  
>     __this_cpu_write(tasklet_vec.head, NULL);  
>     __this_cpu_write(tasklet_vec.tail, this_cpu_ptr(&tasklet_vec.head));  
>     local_irq_enable();
> 
>     while (list) {－－－－－－－－－遍历tasklet链表  
>         struct tasklet_struct *t = list;
> 
>         list = list->next;
> 
>         if (tasklet_trylock(t)) {－－－－－－－－－－－－－－－－－－－－－－－（2）  
>             if (!atomic_read(&t->count)) {－－－－－－－－－－－－－－－－－－（3）  
>                 if (!test_and_clear_bit(TASKLET_STATE_SCHED, &t->state))  
>                     BUG();  
>                 t->func(t->data);  
>                 tasklet_unlock(t);  
>                 continue;－－－－－处理下一个tasklet  
>             }  
>             tasklet_unlock(t);－－－－清除TASKLET_STATE_RUN标记  
>         }
> 
>         local_irq_disable();－－－－－－－－－－－－－－－－－－－－－－－（4）  
>         t->next = NULL;  
>         *__this_cpu_read(tasklet_vec.tail) = t;  
>         __this_cpu_write(tasklet_vec.tail, &(t->next));  
>         __raise_softirq_irqoff(TASKLET_SOFTIRQ); －－－－－－再次触发softirq，等待下一个执行时机  
>         local_irq_enable();  
>     }  
> }

（1）从本cpu的tasklet链表中取出全部的tasklet，保存在list这个临时变量中，同时重新初始化本cpu的tasklet链表，使该链表为空。由于bottom half是开中断执行的，因此在操作tasklet链表的时候需要使用关中断保护

（2）tasklet_trylock主要是用来设定该tasklet的state为TASKLET_STATE_RUN，同时判断该tasklet是否已经处于执行状态，这个状态很重要，它决定了后续的代码逻辑。

> static inline int tasklet_trylock(struct tasklet_struct *t)  
> {  
>     return !test_and_set_bit(TASKLET_STATE_RUN, &(t)->state);  
> }

你也许会奇怪：为何这里从tasklet的链表中摘下一个本cpu要处理的tasklet list，而这个list中的tasklet已经处于running状态了，会有这种情况吗？会的，我们再次回到上面的那个软硬件结构图。同样的，HW block A的驱动使用的tasklet机制并且在中断handler（top half）中将静态定义的tasklet 调度执行。HW block A的硬件中断首先送达cpu0处理，因此该driver的tasklet被挂入CPU0对应的tasklet链表并在适当的时间点上开始执行该tasklet。这时候，cpu0的硬件中断又来了，该driver的tasklet callback function被抢占，虽然tasklet仍然处于running状态。与此同时，HW block A硬件又一次触发中断并在cpu1上执行，这时候，该driver的tasklet处于running状态，并且TASKLET_STATE_SCHED已经被清除，因此，调用tasklet_schedule函数将会使得该driver的tasklet挂入cpu1的tasklet链表中。由于cpu0在处理其他硬件中断，因此，cpu1的tasklet后发先至，进入tasklet_action函数调用，这时候，当从cpu1的tasklet摘取所有需要处理的tasklet链表中，HW block A对应的tasklet实际上已经是在cpu0上处于执行状态了。

我们在设计tasklet的时候就规定，同一种类型的tasklet只能在一个cpu上执行，因此tasklet_trylock就是起这个作用的。

（3）检查该tasklet是否处于enable状态，如果是，说明该tasklet可以真正进入执行状态了。主要的动作就是清除TASKLET_STATE_SCHED状态，执行tasklet callback function。

（4）如果该tasklet已经在别的cpu上执行了，那么我们将其挂入该cpu的tasklet链表的尾部，这样，在下一个tasklet执行时机到来的时候，kernel会再次尝试执行该tasklet，在这个时间点，也许其他cpu上的该tasklet已经执行完毕了。通过这样代码逻辑，保证了特定的tasklet只会在一个cpu上执行，不会在多个cpu上并发。

_原创文章，转发请注明出处。蜗窝科技_

标签: [tasklet](http://www.wowotech.net/tag/tasklet)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [ARMv8-a架构简介](http://www.wowotech.net/armv8a_arch/armv8-a_overview.html) | [Linux cpufreq framework(2)_cpufreq driver](http://www.wowotech.net/pm_subsystem/cpufreq_driver.html)»

**评论：**

**冰霜圣诞节**  
2022-01-15 17:18

关于tasklet串行化的问题，请问如果cpu0在执行回调函数之前响应了另一个中断，这个时候只有run置位，是不是isr也会成功执行tasklet的提交，并raise新的软中断信号，待前一个回调函数执行完毕，第二个同样的回调函数也会再执行一次？

[回复](http://www.wowotech.net/irq_subsystem/tasklet.html#comment-8490)

**小葱aaaa**  
2021-06-22 17:57

楼主好：  
tasklet调度有延迟，该如何修改代码？  
高通通过tasklet来处理ISP的中断，使能高通camera 的metadata功能，camera是60fps，这时候，正常情况，每隔16ms就会打印出isp中断被调度的log.但是当出到一定帧数时，就会出现一段时间内tasklet没有被调度到，导致该帧metadata无法接收到。  
谢谢！

[回复](http://www.wowotech.net/irq_subsystem/tasklet.html#comment-8248)

**[wowo_man](http://www.wowotech.net/)**  
2016-01-15 21:31

该driver的tasklet处于running状态，并且TASKLET_STATE_SCHED已经被清除  
===================================================================  
为什么不run完了再清除TASKLET_STATE_SCHED标志？这样就不用多加一个TASKLET_STATE_RUN判断了。是为了不想丢失 tasklet吗？如果是不想丢失的话，那为什么tasklet_schedule的时候判断已经shed了就直接返回了不设个pending位？

[回复](http://www.wowotech.net/irq_subsystem/tasklet.html#comment-3407)

**我爱芒果叶**  
2016-09-07 00:57

@wowo_man：确实是不想丢失，准确地说是不想在此tasklet正在执行时丢失。  
tasklet_schedule如果遇到TASKLET_STATE_SCHED标志已经被设置，并意味着真正地丢失，因为此tasklet已经被调度了，处在调度状态的此tasklet在执行时也就响应了后面这个“丢失”的调度请求。

[回复](http://www.wowotech.net/irq_subsystem/tasklet.html#comment-4523)

**我爱芒果叶**  
2016-09-07 00:59

@我爱芒果叶：打错字了。  
“并不意味着真正地丢失”

[回复](http://www.wowotech.net/irq_subsystem/tasklet.html#comment-4524)

**抚泪**  
2022-03-22 11:17

@我爱芒果叶：我理解:除了不"丢失"，还保证了一个中断不会打断"自己"。

[回复](http://www.wowotech.net/irq_subsystem/tasklet.html#comment-8578)

**[schedule](http://www.wowotech.net/)**  
2015-07-07 17:33

（b）中断返回software interrupt context，也就是中断抢占软中断上下文的场景  
=============================================  
楼主好，这句我有些疑问  
  
软中断的执行环境不就是 中断上下文吗？ 这时候中断不是被关闭的吗? 为什么中断还能产生呢？

[回复](http://www.wowotech.net/irq_subsystem/tasklet.html#comment-2159)

**[linuxer](http://www.wowotech.net/)**  
2015-07-07 18:06

@schedule：中断上下文是一个统称，包括hard interrupt context和software interrupt context，在hard interrupt context中（也就是top half中）是关闭中断的，在其他情况下，都是开中断的

[回复](http://www.wowotech.net/irq_subsystem/tasklet.html#comment-2162)

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
    
    - [Linux电源管理(5)_Hibernate和Sleep功能介绍](http://www.wowotech.net/pm_subsystem/std_str_func.html)
    - [u-boot启动流程分析(2)_板级(board)部分](http://www.wowotech.net/u-boot/boot_flow_2.html)
    - [X-013-UBOOT-使能autoboot功能](http://www.wowotech.net/x_project/uboot_autoboot.html)
    - [蜗窝流量地域统计](http://www.wowotech.net/168.html)
    - [Linux kernel的中断子系统之（二）：IRQ Domain介绍](http://www.wowotech.net/irq_subsystem/irq-domain.html)
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
作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2015-7-31 12:29 分类：[中断子系统](http://www.wowotech.net/sort/irq_subsystem)

# 一、前言

一种新的机制出现的原因往往是为了解决实际的问题，虽然linux kernel中已经提供了workqueue的机制，那么为何还要引入cmwq呢？也就是说：旧的workqueue机制存在什么样的问题？在新的cmwq又是如何解决这些问题的呢？它接口是如何呈现的呢（驱动工程师最关心这个了）？如何兼容旧的驱动呢？本文希望可以解开这些谜题。

本文的代码来自linux kernel 4.0。

二、为何需要CMWQ？

内核中很多场景需要异步执行环境（在驱动中尤其常见），这时候，我们需要定义一个work（执行哪一个函数）并挂入workqueue。处理该work的线程叫做worker，不断的处理队列中的work，当处理完毕后则休眠，队列中有work的时候就醒来处理，如此周而复始。一切看起来比较完美，问题出在哪里呢？

（1）内核线程数量太多。如果没有足够的内核知识，程序员有可能会错误的使用workqueue机制，从而导致这个机制被玩坏。例如明明可以使用default workqueue，偏偏自己创建属于自己的workqueue，这样一来，对于那些比较大型的系统（CPU个数比较多），很可能内核启动结束后就耗尽了PID space（default最大值是65535），这种情况下，你让user space的程序情何以堪？虽然default最大值是可以修改的，从而扩大PID space来解决这个问题，不过系统太多的task会对整体performance造成负面影响。

（2）尽管消耗了很多资源，但是并发性如何呢？我们先看single threaded的workqueue，这种情况完全没有并发的概念，任何的work都是排队执行，如果正在执行的work很慢，例如4～5秒的时间，那么队列中的其他work除了等待别无选择。multi threaded（更准确的是per-CPU threaded）情况当然会好一些（毕竟多消耗了资源），但是对并发仍然处理的不是很好。对于multi threaded workqueue，虽然创建了thread pool，但是thread pool的数目是固定的：每个oneline的cpu上运行一个，而且是严格的绑定关系。也就是说本来线程池是一个很好的概念，但是传统workqueue上的线程池（或者叫做worker pool）却分割了每个线程，线程之间不能互通有无。例如cpu0上的worker thread由于处理work而进入阻塞状态，那么该worker thread处理的work queue中的其他work都阻塞住，不能转移到其他cpu上的worker thread去，更有甚者，cpu0上随后挂入的work也接受同样的命运（在某个cpu上schedule的work一定会运行在那个cpu上），不能去其他空闲的worker thread上执行。由于不能提供很好的并发性，有些内核模块（fscache）甚至自己创建了thread pool（slow work也曾经短暂的出现在kernel中）。

（3）dead lock问题。我们举一个简单的例子：我们知道，系统有default workqueue，如果没有特别需求，驱动工程师都喜欢用这个workqueue。我们的驱动模块在处理release（userspace close该设备）函数的时候，由于使用了workqueue，那么一般会flush整个workqueue，以便确保本driver的所有事宜都已经处理完毕（在close的时候很有可能有pending的work，因此要flush），大概的代码如下：

> 获取锁A
>
> flush workqueue
>
> 释放锁A

flush work是一个长期过程，因此很有可能被调度出去，这样调用close的进程被阻塞，等到keventd_wq这个内核线程组完成flush操作后就会wakeup该进程。但是这个default workqueue使用很广，其他的模块也可能会schedule work到该workqueue中，并且如果这些模块的work也需要获取锁A，那么就会deadlock（keventd_wq阻塞，再也无法唤醒等待flush的进程）。解决这个问题的方法是创建多个workqueue，但是这样又回到了内核线程数量大多的问题上来。

我们再看一个例子：假设某个驱动模块比较复杂，使用了两个work struct，分别是A和B，如果work A依赖 work B的执行结果，那么，如果这两个work都schedule到一个worker thread的时候就出现问题，由于worker thread不能并发的执行work A和work B，因此该驱动模块会死锁。Multi threaded workqueue能减轻这个问题，但是无法解决该问题，毕竟work A和work B还是有机会调度到一个cpu上执行。造成这些问题的根本原因是众多的work竞争一个执行上下文导致的。

（4）二元化的线程池机制。基本上workqueue也是thread pool的一种，但是创建的线程数目是二元化的设定：要么是1，要么是number of CPU，但是，有些场景中，创建number of CPU太多，而创建一个线程又太少，这时候，勉强使用了single threaded workqueue，但是不得不接受串行处理work，使用multi threaded workqueue吧，占用资源太多。二元化的线程池机制让用户无所适从。

三、CMWQ如何解决问题的呢？

1、设计原则。在进行CMWQ的时候遵循下面两个原则：

（1）和旧的workqueue接口兼容。

（2）明确的划分了workqueue的前端接口和后端实现机制。CMWQ的整体架构如下：

[![cmwq](http://www.wowotech.net/content/uploadfile/201507/fa89b634ddda99e7d302bee62a00f57520150731042913.gif "cmwq")](http://www.wowotech.net/content/uploadfile/201507/9c45a683d212959582073233b703346720150731042811.gif)

对于workqueue的用户而言，前端的操作包括二种，一个是创建workqueue。可以选择创建自己的workqueue，当然也可以不创建而是使用系统缺省的workqueue。另外一个操作就是将指定的work添加到workqueue。在旧的workqueue机制中，workqueue和worker thread是密切联系的概念，对于single workqueue，创建一个系统范围的worker thread，对于multi workqueue，创建per-CPU的worker thread，一切都是固定死的。针对这样的设计，我们可以进一步思考其合理性。workqueue用户的需求就是一个异步执行的环境，把创建workqueue和创建worker thread绑定起来大大限定了资源的使用，其实具体后台是如何处理work，是否否启动了多个thread，如何管理多个线程之间的协调，workqueue的用户并不关心。

基于这样的思考，在CMWQ中，将这种固定的关系被打破，提出了worker pool这样的概念（其实就是一种thread pool的概念），也就是说，系统中存在若干worker pool，不和特定的workqueue关联，而是所有的workqueue共享。用户可以创建workqueue（不创建worker pool）并通过flag来约束挂入该workqueue上work的处理方式。workqueue会根据其flag将work交付给系统中某个worker pool处理。例如如果该workqueue是bounded类型并且设定了high priority，那么挂入该workqueue的work将由per cpu的highpri worker-pool来处理。

让所有的workqueue共享系统中的worker pool，即减少了资源的浪费（没有创建那么多的kernel thread），又保证了灵活的并发性（worker pool会根据情况灵活的创建thread来处理work）。

3、如何解决线程数目过多的问题？

在CMWQ中，用户可以根据自己的需求创建workqueue，但是已经和后端的线程池是否创建worker线程无关了，是否创建新的work线程是由worker线程池来管理。系统中的线程池包括两种：

（1）和特定CPU绑定的线程池。这种线程池有两种，一种叫做normal thread pool，另外一种叫做high priority thread pool，分别用来管理普通的worker thread和高优先级的worker thread，而这两种thread分别用来处理普通的和高优先级的work。这种类型的线程池数目是固定的，和系统中cpu的数目相关，如果系统有n个cpu，如果都是online的，那么会创建2n个线程池。

（2）unbound 线程池，可以运行在任意的cpu上。这种thread pool是动态创建的，是和thread pool的属性相关，包括该thread pool创建worker thread的优先级（nice value），可以运行的cpu链表等。如果系统中已经有了相同属性的thread pool，那么不需要创建新的线程池，否则需要创建。

OK，上面讲了线程池的创建，了解到创建workqueue和创建worker thread这两个事件已经解除关联，用户创建workqueue仅仅是选择一个或者多个线程池而已，对于bound thread pool，每个cpu有两个thread pool，关系是固定的，对于unbound thread pool，有可能根据属性动态创建thread pool。那么worker thread pool如何创建worker thread呢？是否会数目过多呢？

缺省情况下，创建thread pool的时候会创建一个worker thread来处理work，随着work的提交以及work的执行情况，thread pool会动态创建worker thread。具体创建worker thread的策略为何？本质上这是一个需要在并发性和系统资源消耗上进行平衡的问题，CMWQ使用了一个非常简单的策略：当thread pool中处于运行状态的worker thread等于0，并且有需要处理的work的时候，thread pool就会创建新的worker线程。当worker线程处于idle的时候，不会立刻销毁它，而是保持一段时间，如果这时候有创建新的worker的需求的时候，那么直接wakeup idle的worker即可。一段时间过去仍然没有事情处理，那么该worker thread会被销毁。

4、如何解决并发问题？

我们用某个cpu上的bound workqueue来描述该问题。假设有A B C D四个work在该cpu上运行，缺省的情况下，thread pool会创建一个worker来处理这四个work。在旧的workqueue中，A B C D四个work毫无疑问是串行在cpu上执行，假设B work阻塞了，那么C D都是无法执行下去，一直要等到B解除阻塞并执行完毕。

对于CMWQ，当B work阻塞了，thread pool可以感知到这一事件，这时候它会创建一个新的worker thread来处理C D这两个work，从而解决了并发的问题。由于解决了并发问题，实际上也解决了由于竞争一个execution context而引入的各种问题（例如dead lock）。

四、接口API

1、初始化work的接口保持不变，可以静态或者动态创建work。

2、调度work执行也保持和旧的workqueue一致。

3、创建workqueue。和旧的create_workqueue接口不同，CMWQ采用了alloc_workqueue这样的接口符号，相关的接口定义如下：

> #define alloc_workqueue(fmt, flags, max_active, args...)        \\
> \_\_alloc_workqueue_key((fmt), (flags), (max_active),  NULL, NULL, ##args)
>
> #define alloc_ordered_workqueue(fmt, flags, args...)            \\
> alloc_workqueue(fmt, WQ_UNBOUND | \_\_WQ_ORDERED | (flags), 1, ##args)
>
> #define create_freezable_workqueue(name)                \\
> alloc_workqueue("%s", WQ_FREEZABLE | WQ_UNBOUND | WQ_MEM_RECLAIM, 1, (name))
>
> #define create_workqueue(name)                        \\
> alloc_workqueue("%s", WQ_MEM_RECLAIM, 1, (name))
>
> #define create_singlethread_workqueue(name)                \\
> alloc_ordered_workqueue("%s", WQ_MEM_RECLAIM, name)

在描述这些workqueue的接口之前，我们需要准备一些workqueue flag的知识。

标有WQ_UNBOUND这个flag的workqueue说明其work的处理不需要绑定在特定的CPU上执行，workqueue需要关联一个系统中的unbound worker thread pool。如果系统中能找到匹配的线程池（根据workqueue的属性（attribute）），那么就选择一个，如果找不到适合的线程池，workqueue就会创建一个worker thread pool来处理work。

WQ_FREEZABLE是一个和电源管理相关的内容。在系统Hibernation或者suspend的时候，有一个步骤就是冻结用户空间的进程以及部分（标注freezable的）内核线程（包括workqueue的worker thread）。标记WQ_FREEZABLE的workqueue需要参与到进程冻结的过程中，worker thread被冻结的时候，会处理完当前所有的work，一旦冻结完成，那么就不会启动新的work的执行，直到进程被解冻。

和WQ_MEM_RECLAIM这个flag相关的概念是rescuer thread。前面我们描述解决并发问题的时候说到：对于A B C D四个work，当正在处理的B work被阻塞后，worker pool会创建一个新的worker thread来处理其他的work，但是，在memory资源比较紧张的时候，创建worker thread未必能够成功，这时候，如果B work是依赖C或者D work的执行结果的时候，系统进入dead lock。这种状态是由于不能创建新的worker thread导致的，如何解决呢？对于每一个标记WQ_MEM_RECLAIM flag的work queue，系统都会创建一个rescuer thread，当发生这种情况的时候，C或者D work会被rescuer thread接手处理，从而解除了dead lock。

WQ_HIGHPRI说明挂入该workqueue的work是属于高优先级的work，需要高优先级（比较低的nice value）的worker thread来处理。

WQ_CPU_INTENSIVE这个flag说明挂入该workqueue的work是属于特别消耗cpu的那一类。为何要提供这样的flag呢？我们还是用老例子来说明。对于A B C D四个work，B是cpu intersive的，当thread正在处理B work的时候，该worker thread一直执行B work，因为它是cpu intensive的，特别吃cpu，这时候，thread pool是不会创建新的worker的，因为当前还有一个worker是running状态，正在处理B work。这时候C Dwork实际上是得不到执行，影响了并发。

了解了上面的内容，那么基本上alloc_workqueue中flag参数就明白了，下面我们转向max_active这个参数。系统不能允许创建太多的thread来处理挂入某个workqueue的work，最多能创建的线程数目是定义在max_active参数中。

除了alloc_workqueue接口API之外，还可以通过alloc_ordered_workqueue这个接口API来创建一个严格串行执行work的一个workqueue，并且该workqueue是unbound类型的。create\_\*的接口都是为了兼容过去接口而设立的，大家可以自行理解，这里就不多说了。

_原创文章，转发请注明出处。蜗窝科技_

标签: [CMWQ](http://www.wowotech.net/tag/CMWQ)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Concurrency Managed Workqueue之（三）：创建workqueue代码分析](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html) | [linux cpufreq framework(3)\_cpufreq core](http://www.wowotech.net/pm_subsystem/cpufreq_core.html)»

**评论：**

**[bsp](http://www.wowotech.net/)**\
2020-06-10 15:34

大神，碰到一个问题，麻烦有空帮忙分析下。\
在跑camera和ufs测试的时候：\
1，处理camera_work的worker（UNBOuND | HIGHPRI）有时候会运行在cpu0上；\
2，ufs突然频繁操作时，ufs_irq在cpu0（linux的默认conf）上频繁触发，导致中断不断抢占 处理camera_work的worker，持续10ms以上；\
3，由于camera 运行在120fps，需要8ms左右处理一帧数据，而ufs_irq抢占其worker，导致丢帧。

想过一些办法：\
1，迁移irq（利用irqbalance），但是这只会减小概率，如果worker和ufs_irq又都在某个cpu会合了，就又触发了；\
2，不迁移irq，但迁移cpu-loading高的worker到big cpu上，但是此work只要求实时性，所占cpu-loading只有不到10%；\
3，设thread affinity，可是worker-thread是动态create和distroy的；\
4，replace workqueue with irq_thread, 这个工作量太大。

有没有其它办法，让某个worker绑定在非CPU0上？\
或者怎么解决这个丢帧的问题？

[回复](http://www.wowotech.net/irq_subsystem/cmwq-intro.html#comment-8017)

**[bsp](http://www.wowotech.net/)**\
2020-06-10 16:08

@bsp：/sys/devices/virtual/workqueue# echo fe >cpumask\
找到一个方法，可以fix。但总觉的是workaround

[回复](http://www.wowotech.net/irq_subsystem/cmwq-intro.html#comment-8019)

**[bsp](http://www.wowotech.net/)**\
2019-07-31 18:26

个人感觉cmwq还解决了频繁调度的问题\
干同样的活，一个线程或者N个线程 都可以干，N个线程徒增调度的开销。

[回复](http://www.wowotech.net/irq_subsystem/cmwq-intro.html#comment-7561)

**ert**\
2018-09-14 10:20

内核中的create\*\_workqueue中都携带了WQ_MEM_RECLAIM这个flag，这是为什么？看内核代码注释说这个flag和内核回收有关，谁能帮忙具体解释一下？多谢啦！

[回复](http://www.wowotech.net/irq_subsystem/cmwq-intro.html#comment-6952)

**sgy1993**\
2018-02-12 19:55

我感觉这里有点矛盾啊!!望解答....

1. 原文:\
   "例如cpu0上的worker thread由于处理work而进入阻塞状态，那么该worker thread处理的work queue中的其他work都阻塞住"--->这里是说一个 worker thread 如果存在没有处理完的work, 后续的work就没有机会得到处理???

1. 后来原文中又提到一句:\
   "假设某个驱动模块比较复杂，使用了两个work struct，分别是A和B，如果work A依赖 work B的执行结果，毕竟work A和work B还是有机会调度到一个cpu上执行"------>cpu0, work A阻塞了, cpu0还有机会处理B吗? 如果有的话, 1 里面说的为什么会阻塞,  如果没有的话,必须等到处理完A, 怎么会死锁...

[回复](http://www.wowotech.net/irq_subsystem/cmwq-intro.html#comment-6557)

**sgy1993**\
2018-02-12 20:17

@sgy1993：3. 另外请教一个问题, 原文:\
"Multi threaded workqueue能减轻这个问题，但是无法解决该问题，毕竟work A和work B还是有机会调度到一个cpu上执行"-->Multi threaded workqueue为什么能减轻呢?\
假设cpu0 worker thread上执行A的时候,阻塞了... B在cpu1的worker thread执行.这时候获取的 是同一把 锁吗? 如果是的话,不是才会死锁吗?  怎么减轻呢?

[回复](http://www.wowotech.net/irq_subsystem/cmwq-intro.html#comment-6558)

**orangeboyye**\
2022-04-28 19:08

@sgy1993：死锁的意思是A等B，同时B也在等A，等的不一定是锁啊，只要是在等待就行。main先调用func1，再调用func2，func2就是在等func1执行完才能执行啊，这个在逻辑上也是等啊，因为如果func1一直卡在那执行不完，func2就不可能执行。

[回复](http://www.wowotech.net/irq_subsystem/cmwq-intro.html#comment-8604)

**orangeboyye**\
2022-04-28 19:03

@sgy1993：文中的意思是说在work A中要等待work B的执行结果，但是不碰巧，work A 和B被放到了同一个CPU上，而且A在前，那么A就会阻塞等待B，可是B要等A执行完之后才能执行，因此产生了A等B，B等A，相互等待，就是死锁啊。

[回复](http://www.wowotech.net/irq_subsystem/cmwq-intro.html#comment-8603)

**deven**\
2017-09-21 21:32

WQ_MEM_RECLAIM对这个标记的理解有问题~

[回复](http://www.wowotech.net/irq_subsystem/cmwq-intro.html#comment-6056)

**南山南**\
2016-01-06 16:38

4.4.0中的worker_thread好像差异很大。逻辑变化很大。

[回复](http://www.wowotech.net/irq_subsystem/cmwq-intro.html#comment-3356)

**[linuxer](http://www.wowotech.net/)**\
2016-01-07 00:05

@南山南：好的，有机会仔细琢磨一下，看看有什么改动，为何改动。

[回复](http://www.wowotech.net/irq_subsystem/cmwq-intro.html#comment-3359)

**[chen_chuang](http://www.wowotech.net/)**\
2016-12-02 15:38

@linuxer：确实4.4.11版本添加了pool_workqueue 和 好多枚举。希望大神在这一章加个补充~

[回复](http://www.wowotech.net/irq_subsystem/cmwq-intro.html#comment-4953)

**[Jacky](http://none/)**\
2015-09-15 14:13

文中Work thread 和worker thread 是一个东西吗？

[回复](http://www.wowotech.net/irq_subsystem/cmwq-intro.html#comment-2576)

**[linuxer](http://www.wowotech.net/)**\
2015-09-15 14:30

@Jacky：Work thread是笔误，应该是worker thread

[回复](http://www.wowotech.net/irq_subsystem/cmwq-intro.html#comment-2577)

**[linuxer](http://www.wowotech.net/)**\
2015-07-31 18:53

对于第一个问题：\
CMWQ兼容旧的接口，因此也支持create_workqueue，定义如下：\
#define create_workqueue(name)        alloc_workqueue("%s", WQ_MEM_RECLAIM, 1, (name))\
在旧的workqueue机制中，create_workqueue意味着要创建per cpu的worker thread，对应到CMWQ中就是创建一个最大并发是1的bound workqueue。

对于第二个问题：\
后续的CMWQ的文档会描述，敬请期待。

[回复](http://www.wowotech.net/irq_subsystem/cmwq-intro.html#comment-2392)

**passerby**\
2015-07-31 15:20

前排占座，有个2个问题。\
1、现在create_workqueue还在驱动中被使用(kernel 3.10)，这个函数是不是被用alloc_workqueue重新实现过得？

2、按照你上面A  B  C  D 四个work的说法，感觉只有worker pool只会有一个线程会处理work，但是我在work_threader里看到，如果woker_pool->nr_idle = 0 并且 pool-nr_running != 0就会在manage_workers中创建线程，任务会并发处理。能够详细的解释下吗？

[回复](http://www.wowotech.net/irq_subsystem/cmwq-intro.html#comment-2387)

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

  - [Linux下“用户空间修改设备寄存器或者物理内存”的实现](http://www.wowotech.net/soft/186.html)
  - [ARM64的启动过程之（六）：异常向量表的设定](http://www.wowotech.net/armv8a_arch/238.html)
  - [Linux时间子系统系列文章之目录](http://www.wowotech.net/timer_subsystem/time_subsystem_index.html)
  - [玩转BLE(2)\_使用bluepy扫描BLE的广播数据](http://www.wowotech.net/bluetooth/bluepy_scan.html)
  - [我的bash shell学习笔记](http://www.wowotech.net/linux_application/134.html)

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

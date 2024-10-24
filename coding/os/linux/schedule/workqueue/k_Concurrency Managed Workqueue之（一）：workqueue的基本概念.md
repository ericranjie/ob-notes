
作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2015-7-15 18:47 分类：[中断子系统](http://www.wowotech.net/sort/irq_subsystem)

# 一、前言

workqueue是一个驱动工程师常用的工具，在旧的内核中（指2.6.36之前的内核版本）workqueue代码比较简单（大概800行），在2.6.36内核版本中引入了CMWQ（Concurrency Managed Workqueue），workqueue.c的代码膨胀到5000多行，为了深入的理解CMWQ，单单一份文档很难将其描述的清楚，因此CMWQ作为一个主题将会产生一系列的文档，本文是这一系列文档中的第一篇，主要是基于2.6.23内核的代码实现来讲述workqueue的一些基本概念（之所以选择较低版本的内核，主要是因为代码简单，适合理解基本概念）。

# 二、为何需要workqueue

## 1、什么是中断上下文和进程上下文？

在继续描述workqueue之前，我们先梳理一下中断上下文和进程上下文。对于中断上下文，主要包括两种情况：

（1）执行该中断的处理函数（我们一般称之interrupt handler或者叫做top half），也就是hard interrupt context

（2）执行软中断处理函数，执行tasklet函数，执行timer callback函数。（或者统称bottom half），也就是software interrupt context。

top half当然是绝对的interrupt context，但对于上面的第二种情况，稍微有些复杂，其执行的现场包括：

（1）执行完top half，立刻启动bottom half的执行

（2）当负荷比较重的时候（中断产生的比较多），系统在一段时间内都在处理interrupt handler以及相关的softirq，从而导致无法调度到进程执行，这时候，linux kernel采用了将softirq推迟到softirqd这个内核线程中执行

（3）进程在内核态运行的时候，由于内核同步的需求，需要使用local_bh_disable/local_bh_enable来保护临界区。在临界区代码执行的时候，有可能中断触发并raise softirq，但是由于softirq处于disable状态从而在中断返回的时候没有办法invoke softirq的执行，当调用local_bh_enable的时候，会调用已经触发的那个softirq handler。

对于上面的情况1和情况3，毫无疑问，绝对的中断上下文，执行现场的current task和softirq handler没有任何的关系。对于情况2，虽然是在专属的内核线程中执行，但是我也倾向将其归入software interrupt context。

对于linux而言，中断上下文都是惊鸿一瞥，只有进程（线程、或者叫做task）是永恒的。整个kernel都是在各种进程中切来切去，一会儿运行在进程的用户空间，一会儿通过系统调用进入内核空间。当然，系统不是封闭的，还是需要通过外设和User或者其他的系统进行交互，这里就需要中断上下文了，在中断上下文中，完成硬件的交互，最终把数据交付进程或者进程将数据传递给外设。进程上下文有丰富的、属于自己的资源：例如有硬件上下文，有用户栈、有内核栈，有用户空间的正文段、数据段等等。而中断上下文什么也没有，只有一段执行代码及其附属的数据。那么问题来了：中断执行thread中的临时变量应该保存在栈上，那么中断上下文的栈在哪里？中断上下文没有属于自己的栈，肿么办？那么只能借了，当中断发生的时候，遇到哪一个进程就借用哪一个进程的资源（遇到就是缘分呐）。

## 2、如何判定当前的context？

OK，上一节描述中断上下文和进程上下文的含义，那么代码如何知道自己的上下文呢？下面我们结合代码来进一步分析。in_irq()是用来判断是否在hard interrupt context的，我们一起来来看看in_irq()是如何定义的：

> #define in_irq()        (hardirq_count())
>
> #define hardirq_count()    (preempt_count() & HARDIRQ_MASK)

top half的处理是被irq_enter()和irq_exit()所包围，在irq_enter函数中会调用preempt_count_add(HARDIRQ_OFFSET)，为hardirq count的bit field增加1。在irq_exit函数中，会调用preempt_count_sub(HARDIRQ_OFFSET)，为hardirq count的bit field减去1。因此，只要in_irq非零，则说明在中断上下文并且处于top half部分。

解决了hard interrupt context，我们来看software interrupt context。如何判定代码当前正在执行bottom half（softirq、tasklet、timer）呢？in_serving_softirq给出了答案：

> #define in_serving_softirq()    (softirq_count() & SOFTIRQ_OFFSET)

需要注意的是：在2.6.23内核中没有这个定义（上面的代码来自4.0的内核）。内核中还有一个类似的定义：

> #define in_softirq()        (softirq_count())
>
> #define softirq_count()    (preempt_count() & SOFTIRQ_MASK)

in_softirq定义了更大的一个区域，不仅仅包括了in_serving_softirq上下文，还包括了disable bottom half的场景。我们用下面一个图片来描述：

[![sir-context](http://www.wowotech.net/content/uploadfile/201507/77076337836fe9a30a3e8ac3044a605b20150715104416.gif "sir-context")](http://www.wowotech.net/content/uploadfile/201507/84d72f610311e1d3289ed70eb8da5d3420150715104303.gif)

我们知道，在进程上下文中，由于内核同步的要求可能会禁止softirq。这时候，kernel提供了local_bf_enable和local_bf_disable这样的接口函数，这种场景下，在local_bf_enable函数中会执行软中断handler（在临界区中，虽然raise了softirq，但是由于disable了bottom half，因此无法执行，只有等到enable的时候第一时间执行该softirq handler）。in_softirq包括了进程上下文中disable bottom half的临界区部分，而in_serving_softirq精准的命中了software interrupt context。

内核中还有一个in_interrupt的宏定义，从它的名字上看似乎是定义了hard interrupt context和software interrupt context，到底是怎样的呢？我们来看看定义：

> #define in_interrupt()        (irq_count())\
> #define irq_count()    (preempt_count() & (HARDIRQ_MASK | SOFTIRQ_MASK \\
> | NMI_MASK))

注：上面的代码来自4.0的内核。HARDIRQ_MASK定义了hard interrupt contxt，NMI_MASK定义了NMI（对于ARM是FIQ）类型的hard interrupt context，SOFTIRQ_MASK包括software interrupt context加上禁止softirq情况下的进程上下文。因此，in_interrupt()除了包括了中断上下文的场景，还包括了进程上下文禁止softirq的场景。

还有一个in_atomic的宏定义，大家可以自行学习，这里不再描述了。

## 3、为何中断上下文不能sleep？

linux驱动工程师应该都会听说过这句话：中断上下文不能sleep，但是为什么呢？这个问题可以仔细思考一下。所谓sleep就是调度器挂起当前的task，然后在run queue中选择另外一个合适的task运行。规则很简单，不过实际操作就没有那么容易了。有一次，我们调试wifi驱动的时候，有一个issue很有意思：正常工作的时候一切都是OK的，但是当进行压力测试的时候，系统就会down掉。最后发现是在timer的callback函数中辗转多次调用了kmalloc函数，我们都知道，在某些情况下，kmalloc会导致当前进程被block。

从操作系统设计的角度来看，大部分的OS都规定中断上下文不能sleep，有些是例外的，比如solaris，每个中断的handler都是在它自己的task中处理的，因此可以在中断handler中sleep。不过在这样的系统中（很多RTOS也是如此处理的），实际的中断上下文非常的薄，可能就是向该中断handler对应的task发送一个message，所有的处理（ack中断、mask中断、copy FIFO等）都是在该中断的task中处理。这样的系统中，当然可以在中断handler中sleep，不过这有点偷换概念，毕竟这时候的上下文不是interrupt context，更准确的说是中断处理的process context，这样的系统interrupt context非常非常的简单，几乎没有。

当然，linux的设计并非如此（其实在rt linux中已经有了这样的苗头，可以参考中断线程化的文章），中断handler以及bottom half（不包括workqueue）都是在interrupt context中执行。当然一提到context，各种资源还是要存在的，例如说内核栈、例如说memory space等，interrupt context虽然单薄，但是可以借尸还魂。当中断产生的那一个时刻，当前进程有幸成为interrupt context的壳，提供了内核栈，保存了hardware context，此外各种资源（例如mm_struct）也是借用当前进程的。本来呢interrupt context身轻如燕，没有依赖的task，调度器其实是不知道如何调度interrupt context的（它处理的都是task），在interrupt context借了一个外壳后，从理论上将，调度器是完全可以block该interrupt context执行，并将其他的task调入进入running状态。然而，block该interrupt context执行也就block其外壳task的执行，多么的不公平，多么的不确定，中断命中你，你就活该被schedule out，拥有正常思维的linux应该不会这么做的。

因此，在中断上下文中（包括hard interrupt context和software interrupt context）不能睡眠。

## 4、为何需要workqueue

workqueue和其他的bottom half最大的不同是它是运行在进程上下文中的，它可以睡眠，这和其他bottom half机制有本质的不同，大大方便了驱动工程师撰写中断处理代码。当然，驱动模块也可以自己创建一个kernel thread来解决defering work，但是，如果每个driver都创建自己的kernel thread，那么内核线程数量过多，这会影响整体的性能。因此，最好的方法就是把这些需求汇集起来，提供一个统一的机制，也就是传说中的work queue了。

# 三、数据抽象

1、workqueue。定义如下：

> struct workqueue_struct {\
> struct cpu_workqueue_struct \*cpu_wq; －－－－－per-cpu work queue struct\
> struct list_head list; －－－workqueue list\
> const char \*name;\
> int singlethread; －－－－single thread or multi thread\
> int freezeable;  －－－－和电源管理相关的一个flag\
> };

我们知道，workqueue就是一种把某些任务（work）推迟到一个或者一组内核线程中去执行，那个内核线程被称作worker thread（每个processor上有一个work thread）。系统中所有的workqueue会挂入一个全局链表，链表头定义如下：

> static LIST_HEAD(workqueues);

list成员就是用来挂入workqueue链表的。singlethread是workqueue的一个特殊模式，一般而言，当创建一个workqueue的时候会为每一个系统内的processor创建一个内核线程，该线程处理本cpu调度的work。但是有些场景中，创建per-cpu的worker thread有些浪费（或者有一些其他特殊的考量），这时候创建single-threaded workqueue是一个更合适的选择。freezeable成员是一个和电源管理相关的一个flag，当系统suspend的时候，有一个阶段会将所有的用户空间的进程冻结，那么是否也冻结内核线程（包括workqueue）呢？缺省情况下，所有的内核线程都是nofrezable的，当然也可以调用set_freezable让一个内核线程是可以被冻结的。具体是否需要设定该flag是和程序逻辑相关的，具体情况具体分析。OK，上面描述的都是workqueue中各个processor共享的成员，下面我们看看per-cpu的数据结构：

> struct cpu_workqueue_struct {
>
> spinlock_t lock; －－－－用来保护worklist资源的访问
>
> struct list_head worklist;\
> wait_queue_head_t more_work; －－－－－等待队列头\
> struct work_struct \*current_work; －－－－当前正在处理的work
>
> struct workqueue_struct \*wq; －－－－－－指向work queue struct\
> struct task_struct \*thread; －－－－－－－worker thread task
>
> int run_depth;        /\* Detect run_workqueue() recursion depth \*/\
> } \_\_\_\_cacheline_aligned;

worker thread要处理work，这些work被挂入work queue中的链表结构。由于每个processor都需要处理自己的work，因此这个work list是per cpu的。worklist成员就是这个per cpu的链表头，当worker thread被调度到的时候，就从这个队列中一个个的摘下work来处理。

2、work。定义如下：

> struct work_struct {\
> atomic_long_t data;\
> struct list_head entry;\
> work_func_t func;\
> };

所谓work就是异步执行的函数。你可能会觉得，反正是函数，直接调用不就OK了吗？但是，事情没有那么简单，如果该函数的代码中有些需要sleep的场景的时候，那么在中断上下文中直接调用将产生严重的问题。这时候，就需要到进程上下文中异步执行。下面我们仔细看看各个成员：func就是这个异步执行的函数，当work被调度执行的时候其实就是调用func这个callback函数，该函数的定义如下：

> typedef void (\*work_func_t)(struct work_struct \*work);

work对应的callback函数需要传递该work的struct作为callback函数的参数。work是被组织成队列的，entry成员就是挂入队列的那个节点，data包含了该work的状态flag和挂入workqueue的信息。

3、总结

我们把上文中描述的各个数据结构集合在一起，具体请参考下图：

![[Pasted image 20241024183053.png]]

我们自上而下来描述各个数据结构。首先，系统中包括若干的workqueue，最著名的workqueue就是系统缺省的的workqueue了，定义如下：

> static struct workqueue_struct \*keventd_wq \_\_read_mostly;

如果没有特别的性能需求，那么一般驱动使用keventd_wq就OK了，毕竟系统创建太多内核线程也不是什么好事情（消耗太多资源）。当然，如果有需要，驱动模块可以创建自己的workqueue。因此，系统中存在一个workqueues的链表，管理了所有的workqueue实例。一个workqueue对应一组work thread（先不考虑single thread的场景），每个cpu一个，由cpu_workqueue_struct来抽象，这些cpu_workqueue_struct们共享一个workqueue，毕竟这些worker thread是同一种type。

从底层驱动的角度来看，我们只关心如何处理deferable task（由work_struct抽象）。驱动程序定义了work_struct，其func成员就是deferred work，然后挂入work list就OK了（当然要唤醒worker thread了），系统的调度器调度到worker thread的时候，该work自然会被处理了。当然，挂入哪一个workqueue的那一个worker thread呢？如何选择workqueue是driver自己的事情，可以使用系统缺省的workqueue，简单，实用。当然也可以自己创建一个workqueue，并把work挂入其中。选择哪一个worker thread比较简单：work在哪一个cpu上被调度，那么就挂入哪一个worker thread。

# 四、接口以及内部实现

1、初始化一个work。我们可以静态定义一个work，接口如下：

> #define DECLARE_WORK(n, f)                    \\
> struct work_struct n = \_\_WORK_INITIALIZER(n, f)
>
> #define DECLARE_DELAYED_WORK(n, f)                \\
> struct delayed_work n = \_\_DELAYED_WORK_INITIALIZER(n, f)

一般而言，work都是推迟到worker thread被调度的时刻，但是有时候，我们希望在指定的时间过去之后再调度worker thread来处理该work，这种类型的work被称作delayed work，DECLARE_DELAYED_WORK用来初始化delayed work，它的概念和普通work类似，本文不再描述。

动态创建也是OK的，不过初始化的时候需要把work的指针传递给INIT_WORK，定义如下：

> #define INIT_WORK(\_work, \_func)                        \\
> do {                                \\
> (\_work)->data = (atomic_long_t) WORK_DATA_INIT();    \\
> INIT_LIST_HEAD(&(\_work)->entry);            \\
> PREPARE_WORK((\_work), (\_func));                \\
> } while (0)

2、调度一个work执行。调度work执行有两个接口，一个是schedule_work，将work挂入缺省的系统workqueue（keventd_wq），另外一个是queue_work，可以将work挂入指定的workqueue。具体代码如下：

> int fastcall queue_work(struct workqueue_struct \*wq, struct work_struct \*work)\
> {\
> int ret = 0;
>
> if (!test_and_set_bit(WORK_STRUCT_PENDING, work_data_bits(work))) {\
> \_\_queue_work(wq_per_cpu(wq, get_cpu()), work);－－－挂入work list并唤醒worker thread\
> put_cpu();\
> ret = 1;\
> }\
> return ret;\
> }

处于pending状态的work不会重复挂入workqueue。我们假设A驱动模块静态定义了一个work，当中断到来并分发给cpu0的时候，中断handler会在cpu0上执行，我们在handler中会调用schedule_work将该work挂入cpu0的worker thread，也就是keventd 0的work list。在worker thread处理A驱动的work之前，中断很可能再次触发并分发给cpu1执行，这时候，在cpu1上执行的handler在调用schedule_work的时候实际上是没有任何具体的动作的，也就是说该work不会挂入keventd 1的work list，因为该work还pending在keventd 0的work list中。

到底插入workqueue的哪一个worker thread呢？这是由wq_per_cpu定义的：

> static struct cpu_workqueue_struct \*wq_per_cpu(struct workqueue_struct \*wq, int cpu)\
> {\
> if (unlikely(is_single_threaded(wq)))\
> cpu = singlethread_cpu;\
> return per_cpu_ptr(wq->cpu_wq, cpu);\
> }

普通情况下，都是根据当前的cpu id，通过per_cpu_ptr获取cpu_workqueue_struct的数据结构，对于single thread而言，cpu是固定的。

3、创建workqueue，接口如下：

> #define create_workqueue(name) \_\_create_workqueue((name), 0, 0)\
> #define create_freezeable_workqueue(name) \_\_create_workqueue((name), 1, 1)\
> #define create_singlethread_workqueue(name) \_\_create_workqueue((name), 1, 0)

create_workqueue是创建普通workqueue，也就是每个cpu创建一个worker thread的那种。当然，作为“普通”的workqueue，在freezeable属性上也是跟随缺省的行为，即在suspend的时候不冻结该内核线程的worker thread。create_freezeable_workqueue和create_singlethread_workqueue都是创建single thread workqueue，只不过一个是freezeable的，另外一个是non-freezeable的。的代码如下：

> struct workqueue_struct \*\_\_create_workqueue(const char \*name, int singlethread, int freezeable)\
> {\
> struct workqueue_struct \*wq;\
> struct cpu_workqueue_struct \*cwq;\
> int err = 0, cpu;
>
> wq = kzalloc(sizeof(\*wq), GFP_KERNEL);－－－－分配workqueue的数据结构
>
> wq->cpu_wq = alloc_percpu(struct cpu_workqueue_struct);－－－分配worker thread的数据结构
>
> wq->name = name;－－－－－－－－－－初始化workqueue\
> wq->singlethread = singlethread;\
> wq->freezeable = freezeable;\
> INIT_LIST_HEAD(&wq->list);
>
> if (singlethread) {－－－－－－－－－－－－－－－－－－－－－－－（1）\
> cwq = init_cpu_workqueue(wq, singlethread_cpu); －－－初始化cpu_workqueue_struct\
> err = create_workqueue_thread(cwq, singlethread_cpu); －－－创建worker thread\
> start_workqueue_thread(cwq, -1); －－－－wakeup worker thread\
> } else { －－－－－－－－－－－－－－－－－－－－－－－－－－－－－（2）\
> mutex_lock(&workqueue_mutex);\
> list_add(&wq->list, &workqueues);
>
> for_each_possible_cpu(cpu) {\
> cwq = init_cpu_workqueue(wq, cpu);\
> if (err || !cpu_online(cpu)) －－－－没有online的cpu就不需要创建worker thread了\
> continue;\
> err = create_workqueue_thread(cwq, cpu);\
> start_workqueue_thread(cwq, cpu);\
> }\
> mutex_unlock(&workqueue_mutex);\
> } \
> return wq;\
> }

（1）不管是否是single thread workqueue，worker thread（cpu_workqueue_struct）的数据结构总是per cpu分配的（稍显浪费），不过实际上对于single thread workqueue而言，只会使用其中之一，那么问题来了：使用哪一个processor的cpu_workqueue_struct呢？workqueue代码定义了一个singlethread_cpu的变量，如下：

> static int singlethread_cpu \_\_read_mostly;

该变量会在init_workqueues函数中进行初始化。实际上，使用哪一个cpu的cpu_workqueue_struct是无所谓的，选择其一就OK了。由于是single thread workqueue，因此创建的worker thread并不绑定在任何的cpu上，调度器可以自由的调度该内核线程在任何的cpu上运行。

（2）对于普通的workqueue，和single thread的处理有所有不同。一方面，single thread的workqueue没有挂入workqueues的全局链表，另外一方面for_each_possible_cpu确保在每一个cpu上创建了一个worker thread并通过start_workqueue_thread启动其运行，具体代码如下：

> static void start_workqueue_thread(struct cpu_workqueue_struct \*cwq, int cpu)\
> {\
> struct task_struct \*p = cwq->thread;
>
> if (p != NULL) {\
> if (cpu >= 0)\
> kthread_bind(p, cpu);\
> wake_up_process(p);\
> }\
> }

对于single thread，kthread_bind不会执行，对于普通的workqueue，我们必须调用kthread_bind以便让worker thread在特定的cpu上执行。

4、work执行的时机

work执行的时机是和调度器相关的，当系统调度到worker thread这个内核线程后，该thread就会开始工作。每个cpu上执行的worker thread的内核线程的代码逻辑都是一样的，在worker_thread中实现：

> static int worker_thread(void \*\_\_cwq)\
> {\
> struct cpu_workqueue_struct \*cwq = \_\_cwq;\
> DEFINE_WAIT(wait);
>
> if (cwq->wq->freezeable)－－－如果是freezeable的内核线程，那么需要清除task flag中的\
> set_freezable();                    PF_NOFREEZE标记，以便在系统suspend的时候冻结该thread
>
> set_user_nice(current, -5); －－－－提高进程优先级，呵呵，worker thread还是有些特权的哦
>
> for (;;) {\
> prepare_to_wait(&cwq->more_work, &wait, TASK_INTERRUPTIBLE);\
> if (!freezing(current) &&  !kthread_should_stop() &&  list_empty(&cwq->worklist))\
> schedule();－－－－－－－－－－－－－－（1）\
> finish_wait(&cwq->more_work, &wait);
>
> try_to_freeze(); －－－－－－处理来自电源管理模块的冻结请求
>
> if (kthread_should_stop()) －－－－－处理停止该thread的请求\
> break;
>
> run_workqueue(cwq); －－－－－－依次处理work list上的各个work\
> }
>
> return 0;\
> }

（1）导致worker thread进入sleep状态有三个条件：（a）电源管理模块没有请求冻结该worker thread。（b）该thread没有被其他模块请求停掉。（c）work list为空，也就是说没有work要处理

_原创文章，转发请注明出处。蜗窝科技_

标签: [workqueue](http://www.wowotech.net/tag/workqueue)

---

« [Linux CPU core的电源管理(3)\_cpu ops](http://www.wowotech.net/pm_subsystem/cpu_ops.html) | [ARMv8-a架构简介](http://www.wowotech.net/armv8a_arch/armv8-a_overview.html)»

**评论：**

**reborn**\
2018-12-12 15:56

@linuxer\
请问,我们系统出现了一个 空指针反引用的问题涉及到pwq,

\[1353795.671908\] BUG: unable to handle kernel NULL pointer dereference at 0000000000000008\
\[1353795.671972\] IP: process_one_work+0x2e/0x410\
\[1353795.671997\] PGD 0 P4D 0\
\[1353795.672039\] Oops: 0000 \[#1\] SMP PTI\
\[1353795.672070\] Modules linked in: binfmt_misc dccp_diag dccp udp_diag unix_diag af_packet_diag netlink_diag tcp_diag inet_diag nf_conntrack_ipv4 nf_defrag_ipv4 xt_tcpudp xt_conntrack xt_multiport iptable_filter ip_tables x_tables 8021q garp mrp stp llc bonding ipmi_ssif dcdbas intel_rapl skx_edac x86_pkg_temp_thermal intel_powerclamp coretemp kvm_intel kvm irqbypass intel_cstate intel_rapl_perf ipmi_si ipmi_devintf mei_me lpc_ich ipmi_msghandler mei mac_hid shpchp acpi_power_meter ib_iser rdma_cm iw_cm ib_cm ib_core iscsi_tcp libiscsi_tcp libiscsi scsi_transport_iscsi toa(OE) nf_conntrack lp parport autofs4 btrfs zstd_compress raid10 raid456 async_raid6_recov async_memcpy async_pq async_xor async_tx xor raid6_pq libcrc32c raid1 raid0 multipath linear igb dca i40e mgag200 crct10dif_pclmul i2c_algo_bit\
\[1353795.672626\]  crc32_pclmul ttm ghash_clmulni_intel drm_kms_helper pcbc syscopyarea sysfillrect aesni_intel sysimgblt aes_x86_64 fb_sys_fops crypto_simd glue_helper ptp drm megaraid_sas cryptd ahci pps_core libahci \[last unloaded: stap_f2cfe18efb176d4eb6e226c35bf623fd_263331\]\
\[1353795.672827\] CPU: 53 PID: 1314 Comm: kworker/53:2 Tainted: G           OE    4.15.0-21-shopee-generic #22~16.04.1+3\
\[1353795.672900\] Hardware name: Dell Inc. PowerEdge R740xd/0RR8YK, BIOS 1.3.7 02/08/2018\
\[1353795.672999\] RIP: 0010:process_one_work+0x2e/0x410\
\[1353795.673037\] RSP: 0018:ffff9be74f93fe88 EFLAGS: 00010046\
\[1353795.673080\] RAX: 0000000000000000 RBX: ffff8e953d6a2180 RCX: 0000000000000000\
\[1353795.673132\] RDX: 0000000000000000 RSI: ffff9be7824fbb08 RDI: ffff8e9534b5f680\
\[1353795.673184\] RBP: ffff9be74f93fec0 R08: 0000000000000000 R09: 0000000000000000\
\[1353795.673236\] R10: 0000000000000000 R11: 0000000000000010 R12: ffff8e953d6a2180\
\[1353795.673289\] R13: ffff8e953d6a2180 R14: 0000000000000000 R15: ffff8e9534b5f680\
\[1353795.673343\] FS:  0000000000000000(0000) GS:ffff8e953d680000(0000) knlGS:0000000000000000\
\[1353795.676382\] CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033\
\[1353795.680239\] CR2: 00000000000000b0 CR3: 0000000e9260a006 CR4: 00000000007606e0\
\[1353795.682521\] DR0: 0000000000000000 DR1: 0000000000000000 DR2: 0000000000000000\
\[1353795.683576\] DR3: 0000000000000000 DR6: 00000000fffe0ff0 DR7: 0000000000000400\
\[1353795.684812\] PKRU: 55555554\
\[1353795.685832\] Call Trace:\
\[1353795.686860\]  worker_thread+0x253/0x410\
\[1353795.687837\]  kthread+0x121/0x140\
\[1353795.688798\]  ? process_one_work+0x410/0x410\
\[1353795.689769\]  ? kthread_create_worker_on_cpu+0x70/0x70\
\[1353795.690723\]  ret_from_fork+0x35/0x40\
\[1353795.691711\] Code: 00 00 55 48 89 e5 41 57 41 56 41 55 41 54 53 48 83 ec 10 48 8b 06 4c 8b 6f 48 49 89 c6 45 30 f6 a8 04 b8 00 00 00 00 4c 0f 44 f0 \<49> 8b 46 08 44 8b b8 00 01 00 00 41 83 e7 20 41 f6 45 10 04 75\
\[1353795.693324\] RIP: process_one_work+0x2e/0x410 RSP: ffff9be74f93fe88\
\[1353795.694056\] CR2: 0000000000000008

最后发现 get_work_pwq 会返回NULL,

什么情况下它会返回null

[回复](http://www.wowotech.net/irq_subsystem/workqueue.html#comment-7081)

**张飞**\
2018-10-31 12:13

（1）导致worker thread进入sleep状态有三个条件：（a）电源管理模块没有请求冻结该worker thread。（b）该thread没有被其他模块请求停掉。（c）work list为空，也就是说没有work要处理

a,b是不是说反了

[回复](http://www.wowotech.net/irq_subsystem/workqueue.html#comment-7008)

**orangeboyye**\
2022-04-28 18:35

@张飞：没有说反，你仔细看代码，这三个条件不是or的关系，是and的关系，第一个成立了才会去看第二个，第二个成立了才会去看第三个，第一个条件是没有在冻结，如果不成立就是在冻结，整个表达式为false，后面的就不看了，if子语句不执行，下面的语句是try_to_freeze，正好执行冻结啊。第二代条件同理。当三个条件都成立的时候才schedule，也就是去调度其他进程，此时的条件是没有冻结，是合理的，如果是要冻结，此时你去执行调度，不是就不合理了吗。

[回复](http://www.wowotech.net/irq_subsystem/workqueue.html#comment-8602)

**该昵称已屏蔽**\
2018-02-28 16:16

有个问题啊。就是其中一个work block，后续排到同一队列的work都无法执行了

[回复](http://www.wowotech.net/irq_subsystem/workqueue.html#comment-6574)

**小学生**\
2018-02-06 10:21

处于pending状态的work不会重复挂入workqueue。我们假设A驱动模块静态定义了一个work，当中断到来并分发给cpu0的时候，中断handler会在cpu0上执行，我们在handler中会调用schedule_work将该work挂入cpu0的worker thread，也就是keventd 0的work list。在worker thread处理A驱动的work之前，中断很可能再次触发并分发给cpu1执行，这时候，在cpu1上执行的handler在调用schedule_work的时候实际上是没有任何具体的动作的，也就是说该work不会挂入keventd 1的work list，因为该work还pending在keventd 0的work list中。\
————————————————————————————————————————————\
这样不会发生少处理work的事情吗？

[回复](http://www.wowotech.net/irq_subsystem/workqueue.html#comment-6532)

**[linuxer](http://www.wowotech.net/)**\
2018-02-07 09:30

@小学生：不会的，work的目标就是调用对应的work function，连续的work事件只要有一个被处理就OK了。

[回复](http://www.wowotech.net/irq_subsystem/workqueue.html#comment-6537)

**小学生**\
2018-02-07 14:09

@linuxer：那如果假设work function要做的事情就是对某个counter进行加一操作，这样的话，岂不是会少计算counter的值

[回复](http://www.wowotech.net/irq_subsystem/workqueue.html#comment-6541)

**[linuxer](http://www.wowotech.net/)**\
2018-02-08 09:19

@小学生：workqueue机制本身就是提供一个异步进程上下文的执行环境，并不保证调用次数。

[回复](http://www.wowotech.net/irq_subsystem/workqueue.html#comment-6542)

**orangeboyye**\
2022-04-28 18:25

@小学生：如果你这样写代码就会导致少一个work处理，所以不能这么写代码，你应该每次都创建一个新的work来执行，不要复用work。当然我遇到过复用work的，它的逻辑是这么写的，把要处理的事情放到另一个list上，work遍历那个list处理事情，scheddule_work在这里是的作用就是如果work没在运行就唤醒它，如果正在运行就不用管，说明它正在遍历list，这么使用逻辑是正确的。

[回复](http://www.wowotech.net/irq_subsystem/workqueue.html#comment-8601)

**[edison.jiang](http://no/)**\
2017-01-14 16:06

linuxer,你好，我有如下问题想跟您请教一下（麻烦了）：\
1.系统一直卡在cancel_delayed_work_sync（work），走不下去。我现在的workaround是这样的：\
if (!delayed_work_pending (work)) {\
return ;\
}\
cancel_delayed_work_sync(work);\
请问这是为什么？\
2.一个work_struct的func走完后，会和它的workqueue_struct解除绑定关系吗？

[回复](http://www.wowotech.net/irq_subsystem/workqueue.html#comment-5145)

**[wowo](http://www.wowotech.net/)**\
2017-01-16 09:10

@edison.jiang：之所以走不下去，是因为work的func中在等什么事情吧？你这里又在等func走完，所以出现问题。看一下func的实现。

[回复](http://www.wowotech.net/irq_subsystem/workqueue.html#comment-5153)

**[edison.jiang](http://no/)**\
2017-01-16 10:17

@wowo：你好，多谢指点。我看一下我的func的实现，理论上没有问题。加log验证也没问题。不过通过看func， 我发现了是因为我的func里和外面的互斥锁是使用同一个锁，我外面使用了这个锁，等到func执行的时候又去使用这个func，而在外面的流程里有一个spin_lock，所以产生了一直走不下去的情况。

[回复](http://www.wowotech.net/irq_subsystem/workqueue.html#comment-5157)

**hello**\
2017-01-16 10:45

@edison.jiang：2.一个work_struct的func走完后，会和它的workqueue_struct解除绑定关系吗？

当然，如果一个work被调度执行之后，就已经从workqueue上摘下来了。

[回复](http://www.wowotech.net/irq_subsystem/workqueue.html#comment-5158)

**[heziq](http://www.wowotech.net/)**\
2015-08-23 23:17

@linuxer:

tasklet 通过 TASKLET_STATE_SCHED 和 TASKLET_STATE_RUN 来防止并发，threaded irq hander 也通过IRQS_ONESHOT防止并发，workequeue是通过什么手段来防止并发的呢？比如我在两个cpu上同时触发同一个workqueue，我看到一个WORK_STRUCT_PENDING 在work提交的时候使用，是不是workqueue采取与taskelet类似的方法，来防止并发，或者根本就由用户自己考虑并发？我自己先在代码中找了好久都没发现，workqueue的代码变复杂了，我有点晕。

[回复](http://www.wowotech.net/irq_subsystem/workqueue.html#comment-2487)

**[linuxer](http://www.wowotech.net/)**\
2015-08-24 23:39

@heziq：非常好的问题，多谢！但是最近有点忙，而且你的问题不是三言两语就能回答的，稍等两天再回复你。

[回复](http://www.wowotech.net/irq_subsystem/workqueue.html#comment-2488)

**[linuxer](http://www.wowotech.net/)**\
2015-08-25 19:28

@heziq：如果CMWQ的代码让你眼花缭乱，那么不如先看看旧workqueue上的并发。\
1、挂入队列。对于旧的workqueue而言，有两种场景：\
（1）single thread workqueue\
（2）multi thread workqueue或者说per cpu thread workqueue\
无论哪一种场景，当调用queue_work的时候，cpu已经选定。对于single thread workqueue，选择singlethread_cpu定义的那个cpu（一般是first possible cpu），对于per cpu thread workqueue，选择当前的cpu，也就是说，哪个cpu上执行queue_work，就挂入哪一个cpu对应的cpu_workqueue_struct。对于WORK_STRUCT_PENDING标记，你可以再看看queue_work的代码，代码中，test_and_set_bit是原子操作，因此，只能有一个thread进入queue work区域，也就是说，无论有多少个thread（可能来自多个cpu上的多种上下文）调用queue_work，结果只有一个work挂入某个cpu上的work list，变成pending状态。这里还有一个小的同步细节：wq_per_cpu访问了per cpu变量，而get_cpu/put_cpu分别disable preempt和enable preempt，确保了在访问Per-CPU变量的时候，current thread不能调度到其他CPU上去。

需要注意的是，WORK_STRUCT_PENDING可以保证pending状态的work只有一个，但是，如果一旦该work进入执行状态会怎样呢？

2、执行work。在worker thread中，处理work的大概流程是：\
（1）从该worker thread对应的work list中摘下一个work\
（2）clear pending flag\
（3）执行该work的callback function\
如果该work进入执行状态，pending flag已经清除，因此后续queue_work可以继续将work挂入worker list。对于single thread workqueue，由于只有一个worklist，只有一个thread，虽然该thread可以被调度到多个cpu上执行，但是不可能有多个thread同时执行，因此，对于single thread workqueue，其work（不论是否是同一个work）的callback function都是不会重入的，也就是说，各个work都是严格串行化执行。对于per cpu workqueue，各个online的cpu上都有一个worker thread及其要处理的work list。因此，有可能会有多个work在多个cpu上执行。我们举一个实际的例子：假设有一个per cpu workqueue，我们称之WQ，有work A和work B可能来自driver A和driver B，Driver A和Driver B的中断触发并分别送达cpu 0和cpu1，在驱动的handler中会queue相应的work到WQ去，因此work A和work B分别在cpu 0和cpu 1上的worker thread中执行，因此work A和work B的callback function是完全并发的。如果Driver A和Driver B的中断触发并都送达了cpu 0，那么work A和work B的callback function在cpu0上worker thread是串行执行的。上面讨论了多个work实例的并发，那么指定的work，例如work A的并发是怎样的呢？其实也是类似的，如果driver A的中断始终送达cpu 0，那么work A的callback是串行执行的，如果driver A的中断可以送达多个cpu ，那么在多个cpu上，work A的callback也是可以并发执行的。

最后总结一下：挂入到multi thread或者说per cpu workqueue上的指定的work，其callback是不会在一个cpu上并发执行（也就是说在多个cpu上可以并发执行）。

CMWQ的概念是类似的，这里就不再赘述了，后面还会有文档描述。

[回复](http://www.wowotech.net/irq_subsystem/workqueue.html#comment-2492)

**[heziq](http://www.wowotech.net/)**\
2015-08-26 11:25

@linuxer：挂入到multi thread或者说per cpu workqueue上的指定的work，其callback是不会在一个cpu上并发执行（也就是说在多个cpu上可以并发执行）。

如你所说，挂入multi thread上的work的并发引起的同步问题，需要用户自己解决了，不像taskelet那样，同一个tasklet 绝对不会并发。

[回复](http://www.wowotech.net/irq_subsystem/workqueue.html#comment-2493)

**[qq2shui](http://www.wowotech.net/)**\
2015-08-04 08:57

大神们，问一个问题，希望大神们帮忙解决一下。嘻嘻。\
1.创建单线程的work，是绑定到第一个可用的CPU上吗？简单看了一下宏定义，没有彻底弄明白。\
2.为什进程不能睡眠：文中说的只是理论原因，网上说实际原因可能存在两点∶首先在中断上下文，是需要禁用中断的，如果睡眠会导致同级的中断得不到响应，比如时钟中断；其次：由于中断是在他人的进程上下文，如果睡眠，导致进程切出，再次切回来，无法找到回来的路，但是针对这一点实在无法理解。再次：内核中针对中断中睡眠的行为是否存在保护。

[回复](http://www.wowotech.net/irq_subsystem/workqueue.html#comment-2411)

**[qq2shui](http://www.wowotech.net/)**\
2015-08-04 09:34

@qq2shui：更正：“再次：内核中针对中断中睡眠的行为是否存在限制”。

[回复](http://www.wowotech.net/irq_subsystem/workqueue.html#comment-2414)

**[linuxer](http://www.wowotech.net/)**\
2015-08-04 11:02

@qq2shui：第一个问题：single threaded workqueue不会绑定在第一个可用的cpu上，虽然在初始化的时候有：\
singlethread_cpu = first_cpu(cpu_possible_map);\
但是，实际上只是该workqueue的cpu_workqueue_struct选择该cpu，实际调度的时候，single threaded workqueue的worker线程可以被调度到任何一个cpu上运行

第二个问题：我个人不是非常认同您说的这些网络的说法

[回复](http://www.wowotech.net/irq_subsystem/workqueue.html#comment-2415)

**passerby**\
2015-07-27 17:52

@linuxer，在3.10的代码中WQ已经被替换成了CMWQ了吗？

[回复](http://www.wowotech.net/irq_subsystem/workqueue.html#comment-2324)

**[linuxer](http://www.wowotech.net/)**\
2015-07-27 18:01

@passerby：第一个引入cmwq的内核版本是2.6.36

[回复](http://www.wowotech.net/irq_subsystem/workqueue.html#comment-2325)

**[schedule](http://www.wowotech.net/)**\
2015-07-16 07:53

有个弊端，就是其中一个work block，后续排到同一队列的work都无法执行了

[回复](http://www.wowotech.net/irq_subsystem/workqueue.html#comment-2230)

**[linuxer](http://www.wowotech.net/)**\
2015-07-16 09:05

@schedule：对，这也是Concurrency-managed workqueues（CMWQ）要解决的问题之一，具体如何解决且听下回分解，呵呵。\
BTW，4.0的CMWQ的代码要将近5000行代码，而2.6.23的workqueue才800多行代码，复杂度不是增加一点点啊

[回复](http://www.wowotech.net/irq_subsystem/workqueue.html#comment-2233)

**[schedule](http://www.wowotech.net/)**\
2015-07-16 09:27

@linuxer：期待大作，前两年网上讲解CMWQ的资料太少

[回复](http://www.wowotech.net/irq_subsystem/workqueue.html#comment-2234)

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

  - [Linux内核同步机制之（七）：RCU基础](http://www.wowotech.net/kernel_synchronization/rcu_fundamentals.html)
  - [Meltdown论文翻译](http://www.wowotech.net/basic_subject/meltdown.html)
  - [Linux CPU core的电源管理(2)\_cpu topology](http://www.wowotech.net/pm_subsystem/cpu_topology.html)
  - [X-022-OTHERS-git操作记录之合并远端分支的更新](http://www.wowotech.net/x_project/u_boot_merge_denx.html)
  - [Windows系统结合MinGW搭建软件开发环境](http://www.wowotech.net/soft/6.html)

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

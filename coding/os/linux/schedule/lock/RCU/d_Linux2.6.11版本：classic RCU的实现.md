作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2016-1-27 18:31 分类：[内核同步机制](http://www.wowotech.net/sort/kernel_synchronization)
# 一、前言

无论你愿意或者不愿意，linux kernel的版本总是不断的向前推进，做为一个热衷于专研内核的工程师，最大的痛苦莫过于此：当你熟悉了一个版本的内核之后，内核已经推进到一个新的版本，你曾经熟悉的内容可能会变得陌生（这里主要说的是该模块的内部实现，实际上，内核中的每一个子系统都是会尽量保持接口API的不变）。怎么应对这种变化呢？一方面，具体的实现可能千差万别，但是基本的概念是一样的，无论哪一个版本的内核，总是能够理解一个内核子系统的基本概念和运作机理。另外一方面，不同版本之间的实现不同往往是有原因的，新版本中具体实现的不同往往是针对旧版本的问题而改进的，如果你能够理清不同版本之间的差异以及背后的原因，那么你其实也在不断的加深对计算机系统的理解（不断的迭代是一个不错的学习linux内核的方法）。

因此，在进入具体的Linux2.6.11版本内核RCU实现之前，我们首先描述why，也就是说为何修改RCU算法实现？旧内核有哪里不足，新的内核优点是什么？随后需要描述的是如何改进，最后，我们描述的是Linux2.6.11版本内核RCU模块的具体实现。
# 二、 Linux2.5.43版本的可扩展性（scalability）问题
## 1、原理

我们现在研究的并行运算系统大多是基于共享内存的：也就是说系统中有若干个CPU core，共享一个memory space。在这种情况下，各个CPU Core对memory space中的某个固定的变量数据的访问就很值得思考一下了。一个简单的例子：假设程序中有一个全局变量A，每个CPU core都运行一个线程对A进行访问。如果大家都是读，那么没有什么问题，除了第一个CPU core用于cache miss需要从main memory加载之外，其他的CPU core的第一次读操作都可以从通过snoop的操作，从其他cache中获取A的值，并将自己的A变量对应的cacheline设置成shared状态。之后，各个CPU core无论如何的发起多次read操作，都可以从自己的local cache中获取A值，因此，整个系统的性能非常好。

如果有写的操作会怎样呢？事情就不那么美妙了，写操作不能直接施加到shared状态的cache，必须invalidate其他CPU的local cache中A变量对应的cacheline，才能发起写的动作。因此，一次写操作，由于invalidate其他的local cache中的内容，结果会立刻重创其他所有CPU上的read操作的性能。特别是当CPU core数据增加的时候，共享变量的读写性能瓶颈也会逐步的显现出来。

对策是什么呢？主要有两种方法，一种是改成percpu变量。另外一种方法是减少对全局变量的访问。

2、Linux2.5.43版本的全局变量和访问分析

现在，我们先一起review一下Linux2.5.43版本的全局变量（percpu的那些全局变量不考虑）：
```cpp
struct rcu_ctrlblk rcu_ctrlblk =  
{ .mutex = SPIN_LOCK_UNLOCKED, .curbatch = 1,  .maxbatch = 1, .rcu_cpu_mask = 0 };

struct rcu_ctrlblk {  
spinlock_t    mutex;  
long        curbatch;   
long        maxbatch;  
unsigned long    rcu_cpu_mask;  
};
```
对于rcu_ctrlblk数据结构中，rcu_cpu_mask成员修改异常频繁，每个CPU都会在自己经历Quiescent state之后，修改该变量。curbatch和maxbatch仅仅是在一次Grace period才修改，特别是如果cpu core增多，curbatch和maxbatch访问频率不变，但是rcu_cpu_mask成员的写操作呈线性增长。因此，如果RCU算法经常读rcu_cpu_mask，那么其读性能一定是非常的差。

下面，我们再一起过一下Linux2.5.43版本对对rcu_cpu_mask的访问情况。

（1）rcu_pending函数对rcu_cpu_mask的访问情况，代码如下：
```cpp
static inline int rcu_pending(int cpu)  
{  
if ((……忽略其他检查  
test_bit(cpu, &rcu_ctrlblk.rcu_cpu_mask))  
return 1;  
else  
return 0;  
}
```
在各个CPU的tick handler中，需要调用rcu_pending来检测是否进行后续RCU相关的操作（检查Quiescent state，调用callback什么的）。为什么做这个检查呢？逻辑思考是这样的：如果该CPU以及报告了Quiescent state，那么其rcu_cpu_mask必定被修改成0，因此，通过该bit可以知道其是否上报了Quiescent state，如果已经上报，那么后续的RCU相关的操作在该CPU上不需要进行（这时候，就需要等最后一个上报Quiescent state的CPU启动一次新的Grace Period检查过程之后，该CPU才有事情要做）。你知道的，CPU的资源有限，我们得省着点用，能少执行点代码就少执行点代码。

（2）rcu_check_quiescent_state函数对rcu_cpu_mask的访问情况，代码如下：
```cpp
static void rcu_check_quiescent_state(void)  
{  
int cpu = smp_processor_id();

if (!test_bit(cpu, &rcu_ctrlblk.rcu_cpu_mask)) {－－－－－－－－－A   
return;  
}

……

spin_lock(&rcu_ctrlblk.mutex);  
if (!test_bit(cpu, &rcu_ctrlblk.rcu_cpu_mask)) {－－－－－－－－－B   
spin_unlock(&rcu_ctrlblk.mutex);  
return;  
}  
clear_bit(cpu, &rcu_ctrlblk.rcu_cpu_mask);－－－－－－－－－－－C   
RCU_last_qsctr(cpu) = RCU_QSCTR_INVALID;  
if (rcu_ctrlblk.rcu_cpu_mask != 0) {－－－－－－－－－－－－－－－D  
spin_unlock(&rcu_ctrlblk.mutex);  
return;  
}  
rcu_ctrlblk.curbatch++;  
rcu_start_batch(rcu_ctrlblk.maxbatch);－－－－－－－－－－－－－E  
spin_unlock(&rcu_ctrlblk.mutex);  
}
```
在A点的访问，其思路和上一节描述的类似，也是本着解约CPU的MIPS的角度而增加的逻辑判断。B点的访问有必要吗？我没有看出有任何的必要。C和D的访问是必要的，在发现本CPU至少经历一次Quiescent state之后，当然要clear对应的cpu bit。如果rcu_cpu_mask等于0，当然要启动一个新的Grace period的检查过程（处理一批新的callback请求），这对应上面E处的代码。

（3）rcu_start_batch函数对rcu_cpu_mask的访问情况，代码如下：
```cpp
static void rcu_start_batch(long newbatch)  
{  
if (rcu_batch_before(rcu_ctrlblk.maxbatch, newbatch)) {  
rcu_ctrlblk.maxbatch = newbatch;  
}  
if (rcu_batch_before(rcu_ctrlblk.maxbatch, rcu_ctrlblk.curbatch) ||  
(rcu_ctrlblk.rcu_cpu_mask != 0)) {－－－－－－－－需要吗？  
return;  
}  
rcu_ctrlblk.rcu_cpu_mask = cpu_online_map;－－－重置CPU BITMASK，必须的  
}
```
如果rcu_start_batch只是在rcu_check_quiescent_state函数（5）处被调用，那么rcu_start_batch函数中对rcu_cpu_mask的非零检查是没有必要的。不过，在rcu_process_callbacks函数，各个CPU在第一次从nextlist链表摘下请求挂入curlist链表中的时候，也会调用该函数。

从上面的描述可以得出结论：Linux2.5.43版本对rcu_cpu_mask的读操作过于频繁，会导致cache line trashing，影响性能。
# 三、Linux2.5.43版本的其他问题

1、实时性问题

（1）原理。我们知道，对于linux kernel而言，中断上下文（包括softirq context，当然tasklet context是softirq context的一种）优先级总是高过进程优先级，也就是说，完成了中断上下文的执行，才会启动进程调度。RCU callback函数是在tasklet context中执行，本质上属于中断上下文，因此，一旦一个批次的Grace period过去，那么这个批次的callback函数会在中断上下文（bottom half）中被一一执行。

（2）Linux2.5.43版本代码review。具体代码如下：

> static void rcu_process_callbacks(unsigned long unused)  
> {  
>     int cpu = smp_processor_id();  
>     LIST_HEAD(list);
> 
>     if (!list_empty(&RCU_curlist(cpu)) && rcu_batch_after(rcu_ctrlblk.curbatch, RCU_batch(cpu))) {  
>         list_splice(&RCU_curlist(cpu), &list);  
>         INIT_LIST_HEAD(&RCU_curlist(cpu));  
>     }
> 
> ……  
>     if (!list_empty(&list))  
>         rcu_do_batch(&list);  
> }

在Linux2.5.43版本中，一旦一个批次的RCU callback请求完成了Grace Period，各个CPU会将curlist中的RCU callback请求从链表中摘下，并在tasklet上下文中，一次性的完成所有的callback函数的调用（参考rcu_do_batch函数）。在一般情况下，这样设计也没有什么问题，不过，在RCU callback比较多的时候，一次性完成所有的callback函数的执行是一个非常耗时的操作。当在tasklet上下文中调用成千个callback函数的时候，我想，此刻调度器的内心几乎是崩溃的，应该会有成千个草泥马奔腾而过。

2、接口封装问题

（1）原理。做为一种同步机制，RCU的接口应该能够屏蔽底层的细节，应该象其他的同步机制一样，隐含memory barrier的功能。如果其他模块在使用RCU模块提供的接口的时候，还需要调用memory barrier模块的接口API，那么这一点实际上是违背了软件抽象的基本原则。

（2）Linux2.5.43版本代码review。在该版本上，RCU模块的接口API有四个：

> #define **rcu_read_lock**()        preempt_disable()  
> #define **rcu_read_unlock**()    preempt_enable()  
> extern void FASTCALL(**call_rcu**(struct rcu_head *head, void (*func)(void *arg), void *arg));  
> extern void **synchronize_kernel**(void);

前面两个用来标识RCU reader一侧的临界区的，后面两个是用来被updater调用，以便在适当的时候进行reclaimation的。这套接口函数有一个缺点，就是在RCU-protected pointer读写的时候需要一些额外的memory barrier的操作需要调用者自己来完成。

3、内存消耗问题

在Linux2.5.43这个版本中，检测Quiescent state是通过两个变量来完成的，如下：

>     long        qsctr;           
>     long        last_qsctr

qsctr是一个不断累加的值，last_qsctr是记录上次的qsctr的值，通过比对last_qsctr和qsctr的值就可以判断CPU的Quiescent state了。但是，我们可以再想想：判断Quiescent state需要两个变量吗？我们需要用一个counter来记录Quiescent state吗？经历3次Quiescent state和经历5次Quiescent state有区别吗？没有。因此，实际上用一个变量就可以解决的问题却使用两个变量。

另外，在Linux2.5.43这个版本中，RCU callback占用内存太多。将struct rcu_head 组织成双向链表是没有必要的，单链表足矣，此外，arg参数也没有必要，毕竟大多数情况下，struct rcu_head是嵌入在其他数据结构中，因此callback函数接收一个struct rcu_head *类型的指针就够了，在callback函数中，可以通过container_of还原嵌入struct rcu_head的数据结构，那么参数其实是没有必要了。

4、代码可读性问题

在Linux2.5.43这个版本中，对RCU请求批次的管理是通过下面的数据结构控制的：

> struct rcu_ctrlblk {…  
>     long        curbatch;  
>     long        maxbatch;   
> …};

curbatch和maxbatch语义不明晰，给代码阅读带来困难，特别是maxbatch，根本不知道要表达什么。通过代码分析可以知道，maxbatch主要用来确定在上一个Grace Period结束之后（即该批次的Grace period已经过去了）是否立刻启动一个新的Grace period批次。通过maxbatch这个成员可以确定是否启动。如果curbatch==maxbatch，则表示不启动，如果curbatch！=maxbatch则表示需要启动。相信知道真相的你眼泪就会掉下来：一个flag的事，干嘛搞这么复杂？

四、Linux2.6.11上的RCU算法改进

在控诉完Linux2.5.43的RCU实现之后，我们一起看看Linux2.6.11的RCU算法是如何进行改进的。

1、对scalability的改进。可以考虑先把Grace Period相关的数据独立出来，如下：

> struct rcu_state {  
>     spinlock_t    lock;  
>     cpumask_t    cpumask;  
> };

判断Grace Period的基本算法不变，还是依赖cpumask这个共享数据，每个bit表示一个CPU的Quiescent state，全0则表示渡过了Grace Period。write cpumask的策略不变，只要尽量减少对cpumask的读操作就OK了。

当然，程序逻辑上还是要判断各个CPU的Quiescent state，怎么办呢？Linux2.6.11内核的struct rcu_data引入了两个新的成员，如下：

> struct rcu_data {  
>     long        quiescbatch;   
>     int        qs_pending;
> 
> …… };

这两个成员用来判断本CPU的Quiescent state。quiescbatch用来记录本CPU上，正在为哪一个批次号码的RCU callback请求进行Quiescent state的检测。qs_pending用来记录当前CPU是否已经上报了Quiescent state（对应quiescbatch那个批次号），如果等于1，表示处于pending状态，也就是说还没有上报Quiescent state。

在struct rcu_data数据结构中还有一个batch成员，这个成员又表示什么呢？当RCU callback请求从nextlist移动的curlist的时候，会分配一个batch ID，是current batch ID + 1，分配是一回事，能不能立刻启动该批次的Grace period检测是另外一回事。在current batch ID还没有通过Grace period之前，这个新分配的是不会成为current的，只能是pending的批次。因此，struct rcu_data数据结构中的batch成员是具体描述该CPU curlist中的批次号的。

2、检测是否经历至少一次Quiescent state。在Linux2.6.11版本中，只有一个变量来跟踪Quiescent state：

> int        passed_quiesc;

这是个flag，要么是0，要么是1。如果等于1表示至少经历一次Quiescent state了。虽然无法记录CPU经历了多少次的Quiescent state，但是who care？

3、RCU callback数据结构的修改。struct rcu_head定义如下：

> struct rcu_head {  
>     struct rcu_head *next;  
>     void (*func)(struct rcu_head *head);  
> };

这个数据结构减少了50％的memory size，如果系统中大量使用了RCU，那么整体节省的memory不是一个小数目。

4、对实时性的改进。既然不能一次性完成所有的callback函数，那么总有一个限制吧，这就是maxbatch变量定义的：

> static int maxbatch = 10;

缺省定义为10，也就是说如果该批次的callback请求有16个，那么完成10个callback调用之后，必须结束rcu tasklet的执行，等到下一个tick的时候完成剩余callback函数的执行。由于不能一次性执行完所有的callback，因此，需要在struct rcu_data 增加donelist链表，如下：

>     struct rcu_head *donelist;  
>     struct rcu_head **donetail;

如果不能一次处理完所有的请求，那么剩余的callback请求可以调用tasklet_schedule函数，在下次tasklet context中继续完成（参考rcu_do_batch函数）。由于使用了单向链表，因此struct rcu_data中的链表头不再是struct list_head，而是struct rcu_head 类型的指针，donetail指向链表的尾部。

5、对RCU控制块的改进。struct rcu_ctrlblk就是RCU模块的控制块，代码如下：

> struct rcu_ctrlblk {  
>     long    cur;        /* Current batch number.                      */  
>     long    completed;    /* Number of the last completed batch         */  
>     int    next_pending;    /* Is the next batch already waiting?         */  
> } ____cacheline_maxaligned_in_smp;

cur是当前正在处理的那个批次的callback请求（检测Grace Period进行中），这些请求会在各个CPU的curlist链表中（但是不表示各个CPU中的curlist中的RCU callback请求都属于当前批次，有些CPU的curlist中的请求属于next pending的批次）。对于当前正在处理的批次，它们正等待Grace Period过去，一旦过去，当前正在处理的批次会变成completed批次。

completed是上次完成的callback请求的批次。next_pending表示下一批次的callback请求是否ready了。

什么是当前（current batch number）？什么是上个批次（last batch number）？什么是下个批次（next batch number）？其实系统中能同时存在的批次最多就三个，不过全局RCU控制块只监控两个：cur等于current batch number，completed等于last completed batch number。一旦current batch完成了Grace Period，那么它就变成last completed batch number，同时自己加一，表示准备检测下一个batch ID的Grace Period。因此，通过比对各个CPU的rcu_data->quiescbatch和rcu_ctrlblk->cur，可以知道是否需要启动下一个批次的处理。

next batch只会出现在各个CPU的rcu_data->batch中。什么时候会出现这样的场景呢？如果某个特定的CPU处于下面的状态：

（1）没有当前批次的请求挂入本CPU的curlist中。虽然如此，但是Quiescent state仍然需要检测

（2）有新的请求挂入本CPU的nextlist链表中

当本CPU的tick到来的时候，会将nextlist链表中的请求移入curlist，同时分配batch ID。不过，由于本CPU还在检测Quiescent state，因此无法立刻启动新的批次。不过系统始终要记录这个标记，这就是next_pending标记的作用。

6、新功能的增加。在Linux2.6.11版本中，增加了两个新的功能：一个是对CPU HOTPLUG的支持，另外一个是针对reader side的临界区都是在softirq context中的。这种特殊的rcu使用场景需要一点特别的处理，那就是在检测是否经历至少一次Quiescent state的时候，除了context switch之外，执行完成一次softirq handler也是可以判断为是否经历至少一次Quiescent state的条件。

五、RCU数据流和控制流

1、初始状态

我们假设我们现在有一个双CPU的系统，状态定义如下：

|   |   |   |   |
|---|---|---|---|
|rcu_state|rcu_ctrlblk|CPU0的rcu_data|CPU1的rcu_data|
|cpumask（0，1）|Cur（5）  <br>Completed（4）  <br>next_pending（0）|Quiescbatch（5）  <br>passed_quiesc（1）  <br>qs_pending（0）  <br>batch（5）  <br>nextlist（）  <br>curlist（＝＝）  <br>donelist（）|Quiescbatch（5）  <br>passed_quiesc（0）  <br>qs_pending（1）  <br>batch（5）  <br>nextlist（）  <br>curlist（＝＝＝）  <br>donelist（）|

解释如下：

（1）无论CPU0还是CPU1都没有新申请的RCU callback，即nextlist为空，用nextlist（）表示

（2）CPU0已经通过QS（Quiescent State），即passed_quiesc成员已经被置位，用passed_quiesc（1）表示

（3）CPU1还没有通过QS，用passed_quiesc（0）

（4）当前批次是5，用cur（5）表示。上次完成的是批次4，用completed（4）表示。没有pending的批次，用next_pending（0）表示。

（5）curlist链表中有正在检测QS的请求，用curlist（＝＝＝）表示

2、CPU0的tick到来事件处理

我们来看看__rcu_pending函数的处理：

> static inline int __rcu_pending(struct rcu_ctrlblk *rcp, struct rcu_data *rdp)  
> {  
>     if (rdp->curlist && !rcu_batch_before(rcp->completed, rdp->batch))－－－－－（1）  
>         return 1;   
>     if (!rdp->curlist && rdp->nxtlist)－－－－－－－－－－－－－－－－－－－－－（2）  
>         return 1;   
>     if (rdp->donelist)－－－－－－－－－－－－－－－－－－－－－－－－－－－（3）  
>         return 1;   
>     if (rdp->quiescbatch != rcp->cur || rdp->qs_pending)－－－－－－－－－－－－（4）  
>         return 1;   
>     return 0;－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－（5）  
> }

（1）如果rcp->completed大于或者等于rdp->batch，那么说明该CPU上的批次（rdp->batch）已经渡过GP（Grace Period），如果rdp->curlist非空，那说明本CPU需要将curlist链表中的请求转移到donelist中，因此pending状态检测返回true。当然，对于本场景，rcp->completed小于rdp->batch，不成立。

（2）如果有新的请求，本CPU的curlist链表为空，那么pending状态检测返回true，因为需要将这些新的请求移动到curlist中并分配batch number。对应这里的场景，curlist非空而且nxtlist没有请求，因此条件不成立

（3）如果donelist中有需要处理的请求，那么也是需要进行处理的

（4）如果本CPU正在进行QS检测的批次（rdp->quiescbatch）不等于当前正在检测GP的批次（rcp->cur ），说明已经启动了新的批次进行GP检测，那么需要本CPU初始化该CPU QS检测的状态，因此pending状态检测返回true。此外，如果rdp->qs_pending等于1表示本CPU的正在进行QS的检测，那么需要调用rcu_check_quiescent_state函数执行实际的QS状态的检测。和上面一样，对于当前的CPU0的状态，这些不成立。

（5）结论：对于CPU0而言，rcu_pending返回0，不会进行任何实际的动作。

3、CPU1上发生了进程切换

发生进程切换的时候，CPU1会执行schedule函数，在该函数中会调用rcu_qsctr_inc(task_cpu(prev))，从而将CPU1的passed_quiesc设置成1。

只是在进程切换的时候修改passed_quiesc是否足够呢？我们举一个极端的场景：加入当前系统中有一个realtime的进程，并且优先级最高，那么实际上在该进程进入阻塞状态之前，系统不会发生进程切换，如果把passed_quiesc的修改任务只交给进程切换，那么这种场景下不就悲剧了吗？因此在周期性tick中，其handler中也有可能修改passed_quiesc，如下：

> void rcu_check_callbacks(int cpu, int user)  
> {  
>     if (user || －－－－－－－－－－－－－－－－－－－－－－－－－－－－（1）  
>         (idle_cpu(cpu) && !in_softirq() &&  
>                 hardirq_count() <= (1 << HARDIRQ_SHIFT))) {－－－－－－－－（2）  
>         rcu_qsctr_inc(cpu);  
>         rcu_bh_qsctr_inc(cpu);  
>     } else if (!in_softirq())－－－－－－－－－－－－－－－－－－－－－－－（3）  
>         rcu_bh_qsctr_inc(cpu);  
>     tasklet_schedule(&per_cpu(rcu_tasklet, cpu));  
> }

（1）首先，我们知道，rcu_check_callbacks函数是在tick handler中处理，如果timer中断命中了userspace，说明thread已经立刻内核空间，那么必定离开了reader侧的临界区（RCU读侧的临界区不能block，而且我们在rcu_read_lock中disable了preempt，因此RCU读侧的临界区也不会发生抢占）。

（2）OK，我们再思考这样的问题：在进程切换的时候以及在tick handler中命中userspace的场景中修改passed_quiesc是否足够呢？很遗憾，还是不行，考虑这样一个场景，该CPU比较空闲，在idle进程中，而且由于没有其他ready的进程，因此不进行进程切换。此外，在tick handler中，由于always是kernel space，因此也无法标注passed_quiesc的flag。在这样的场景中，由于那个alway idle的CPU上的passed_quiesc的flag无法置位，Grace Period变得非常的长（直到有实际的进程被调度）。

因此，当tick handler命中idle进程的时候，我们还需要进行特别的处理，去掉softirq场景，去掉中断嵌套的场景，那么如果在tick中断中发现CPU实际上是真真正正的执行idle的代码逻辑，那么可以认为本CPU的QS已经通过。

（3）如果tick中断不是打断softirq的执行，那么则说明之前的softirq的handler都执行完毕了，因此bh类型的RCU状态。本文不会对此进行细致描述，读者有兴趣可以自行阅读。

4、CPU1的tick事件到来

CPU1的tick事件到来的时候，会有哪些逻辑动作呢？在rcu_check_quiescent_state函数中，检测到passed_quiesc被置位，这是推动进入下一个状态的原动力。这时候会执行下面的动作：

> rdp->qs_pending = 0;  
> if (likely(rdp->quiescbatch == rcp->cur))  
>     cpu_quiet(rdp->cpu, rcp, rsp);

一方面由于已经探测到本CPU的QS已经通过，因此clear该CPU的qs_pending，说明当前批次在本CPU上的QS状态探测已经完毕。另外，需要调用cpu_quiet来设置本CPU的Quiescent state，代码如下：

> static void cpu_quiet(int cpu, struct rcu_ctrlblk *rcp, struct rcu_state *rsp)  
> {  
>     cpu_clear(cpu, rsp->cpumask);－－－－－－－－－－－－－－－－（1）  
>     if (cpus_empty(rsp->cpumask)) {－－－－－－－－－－－－－－－（2）  
>         rcp->completed = rcp->cur;－－－－－－－－－－－－－－－－（3）  
>         rcu_start_batch(rcp, rsp, 0);－－－－－－－－－－－－－－－－（4）  
>     }  
> }

（1）清除cpumask这个bitmask中代表自己的那个bit

（2）如果cpumask等于0，那么说明整个GP已经过去，一个新的时代，哦，不，一个新的批次就要开始了。

（3）当前批次已经渡过GP，但是这只是一个本CPU才知道的秘密，需要广播出去，让所有CPU都知道，成为尽人皆知的秘密。因此，实际上对于系统中最后一个历经Quiescent state的CPU而言，它的责任重大，它必须肩负起将该批次的Grace period已经过去的消息广播到所有的CPU上去。这个广播操作很简单，就是rcp->completed = rcp->cur，系统中的所有CPU都可以通过查阅rcp->completed了解当前已过Grace Period的批次号是哪个，然后，通过和rdp->batch进行比对，就知道自己的当前批次的请求（注意：这里的当前批次是指该CPU的当前批次，在curlist链表中）是否可以转移到donelist中并进行处理。

（4）我们知道，完成一批就意味着要新启动一批请求的处理，因此调用rcu_start_batch(rcp, rsp, 0)启动下一个批次，但是，由于next_pending等于0，因此实际上没有什么动作

至此，我们可以得到当前的状态如下：

|   |   |   |   |
|---|---|---|---|
|rcu_state|rcu_ctrlblk|CPU0的rcu_data|CPU1的rcu_data|
|cpumask（0，0）|Cur（5）  <br>Completed（5）  <br>next_pending（0）|Quiescbatch（5）  <br>passed_quiesc（1）  <br>qs_pending（0）  <br>batch（5）  <br>nextlist（）  <br>curlist（＝＝）  <br>donelist（）|Quiescbatch（5）  <br>passed_quiesc（1）  <br>qs_pending（0）  <br>batch（5）  <br>nextlist（）  <br>curlist（＝＝＝）  <br>donelist（）|

5、CPU0上有新的RCU请求

虽然在上面的tick中，发现了批次5的请求已经渡过GP，但是，真正的处理还需要等待到下一个tick中去完成。在两个tick之间，如果CPU0有新的RCU请求会怎么样呢？实际的动作很简单，这些新的请求都会挂入nextlist链表。

系统中各个CPU上的updater都会在适当的时机调用call_rcu或者synchronize_kernel接口函数，以便注册一个RCU callback函数。这些请求会被挂入各个CPU的struct rcu_data的nextlist链表中，只要用户调用call_rcu或者synchronize_kernel接口函数，callback请求就会挂入，没有任何的条件约束。

这些nextlist链表中RCU callback请求，没有ID（batch number），不和任何批次相关，不会和任何Grace Period的检测相关，各个CPU都会以HZ的频率来检测是否需要启动一次新的批次，以便可以将挂在nextlist链表中的请求转移到curlist中。对于一个特定的CPU而言，这个检测条件很简单：

（1）有新申请的RCU callback请求（nextlist非空）

（2）该CPU当前批次的请求（在curlist链表中）Grace Period已经过去，并且已经转移到donelist进行处理（curlist链表为空）

OK，至此，我们可以得到当前的状态如下：

|   |   |   |   |
|---|---|---|---|
|rcu_state|rcu_ctrlblk|CPU0的rcu_data|CPU1的rcu_data|
|cpumask（0，0）|Cur（5）  <br>Completed（5）  <br>next_pending（0）|Quiescbatch（5）  <br>passed_quiesc（1）  <br>qs_pending（0）  <br>batch（5）  <br>nextlist（**）  <br>curlist（＝＝）  <br>donelist（）|Quiescbatch（5）  <br>passed_quiesc（1）  <br>qs_pending（0）  <br>batch（5）  <br>nextlist（）  <br>curlist（＝＝＝）  <br>donelist（）|

6、CPU0的tick到来事件处理

这次tick的到来会处理不少的事情，我们一起来看看__rcu_process_callbacks的代码：

> static void __rcu_process_callbacks(struct rcu_ctrlblk *rcp,  
>             struct rcu_state *rsp, struct rcu_data *rdp)  
> {  
>     if (rdp->curlist && !rcu_batch_before(rcp->completed, rdp->batch)) {－－－－（1）  
>         将curlist链表中的请求转移到donelist  
>     }
> 
>     local_irq_disable();  
>     if (rdp->nxtlist && !rdp->curlist) {  
>         将nxtlistlist链表中的请求转移到curlist  
>         local_irq_enable();
> 
>         rdp->batch = rcp->cur + 1;－－－－－－－－－－－－－－－－－－－－－（2）  
>         smp_rmb();
> 
>         if (!rcp->next_pending) {－－－－－－－－－－－－－－－－－－－－－－（3）  
>             spin_lock(&rsp->lock);  
>             rcu_start_batch(rcp, rsp, 1);－－－－－－－－－－－－－－－－－－－（4）  
>             spin_unlock(&rsp->lock);  
>         }  
>     } else {  
>         local_irq_enable();  
>     }  
>     rcu_check_quiescent_state(rcp, rsp, rdp);  
>     if (rdp->donelist)  
>         rcu_do_batch(rdp);－－－－－完成callback函数的执行
> 
> }

（1）一旦该cpu上的curlist链表有请求，并且completed大于或者等于其批次号（rdp->batch），那么说明该CPU上的curlist链表中的批次已经渡过GP，可以移动到donelist中了。

（2）curlist已经清空，可以把nxtlist中的请求放到curlist中，当然，最重要的是分配一个批次号来标识它。当然，这个ID是6的批次虽然在curlist中，但是并不是系统中的当前批次（当前批次仍然是5）。

（3）由于没有pending的批次（next_pending等于0），因此，调用rcu_start_batch来试图启动一个新的批次。

（4）rcu_start_batch函数中的next_pending参数是用来控制是否设定pending标志的，对于这个场景，我们刚刚分配了一个新的batch number，当然传递1的标志了，这样可以将其设定为pending状态。不过，如果能够顺利的启动一个新的批次，那么pending标志还是会被清除（一个新分配的批次如果立刻启动，那么说明其变成当前的批次，而不是pending了）。具体代码如下：

> static void rcu_start_batch(struct rcu_ctrlblk *rcp, struct rcu_state *rsp, int next_pending)  
> {  
>     if (next_pending)  
>         rcp->next_pending = 1; －－－－－先设置为pending，pending的批次号是6
> 
>     if (rcp->next_pending && rcp->completed == rcp->cur) {－－－当前的5号批次刚刚完成  
>         启动6号批次的GP探测  
>         cpus_andnot(rsp->cpumask, cpu_online_map, nohz_cpu_mask);
> 
>         rcp->next_pending = 0;－－－6号批次已经扶正，成为current，因此clear pending flag  
>         smp_wmb();  
>         rcp->cur++;－－－－－让cur成员指向6号批次  
>     }  
> }

如果当前批次刚刚完成（rcp->completed == rcp->cur）并且有pending的批次（批次6刚刚新鲜出炉）要处理，那么还犹豫什么，立刻启动这个pending的批次处理，这时候，pending的6号批次也就变成了current批次了。需要注意的是：当前的批次是6号批次是一个重大事件，需要广播到系统中的所有的CPU上去，怎么广播的呢？通过rcp->cur++这条指令完成。而其他CPU可以通过对比Quiescbatch和rcp->cur就可以知道是否启动了一个新的批次了。

顺便一提的是：虽然CPU1上5号批次尚未完成，CPU0又启动了6号批次GP的处理（整个系统的cpumask（包括CPU1）被重置），不过没有关系，通过对比rcp->completed和该CPU的rdp->batch就可以知道该CPU上的curlist是否完成了GP探测，需要移动到donelist中。

__rcu_process_callbacks中的大部分的代码逻辑已经清晰，除了rcu_check_quiescent_state，代码如下：

> static void rcu_check_quiescent_state(struct rcu_ctrlblk *rcp,  
>             struct rcu_state *rsp, struct rcu_data *rdp)  
> {  
>     if (rdp->quiescbatch != rcp->cur) {－－－－－－－－－－－－－－－－（1）  
>         rdp->qs_pending = 1;  
>         rdp->passed_quiesc = 0;  
>         rdp->quiescbatch = rcp->cur;  
>         return;  
>     }   
>     if (!rdp->qs_pending)－－－－－－－－－－－－－－－－－－－－－－（2）  
>         return;   
>     if (!rdp->passed_quiesc)－－－－－－－－－－－－－－－－－－－－－（3）  
>         return;  
>     rdp->qs_pending = 0; －－－－－－－－－－－－－－－－－－－－－－（4）
> 
> ……  
> }

（1）已经启动了新的6号批次的GP处理，而GP的处理需要各个CPU上的QS的检测，因此，在一个新的批次开始之初，需要reset本CPU上的QS检测的flag。

（2）rcu_check_quiescent_state在各个CPU上都是会反复执行，不过，如果本CPU的QS已经通过，只是在等待其他CPU QS的通过，那么直接return是最好的选择。

（3）如果需要探测本CPU的QS，那么就看看passed_quiesc是否被设置成1

（4）代码执行到这里，说明本CPU的QS已经通过。不过，由于这些代码逻辑和本场景不符，更具体的代码逻辑后续再描述。

至此，我们可以得到当前的状态如下：

|   |   |   |   |
|---|---|---|---|
|rcu_state|rcu_ctrlblk|CPU0的rcu_data|CPU1的rcu_data|
|cpumask（1，1）|Cur（6）  <br>Completed（5）  <br>next_pending（0）|Quiescbatch（6）  <br>passed_quiesc（0）  <br>qs_pending（1）  <br>batch（6）  <br>nextlist（）  <br>curlist（**）  <br>donelist（＝＝）|Quiescbatch（5）  <br>passed_quiesc（1）  <br>qs_pending（0）  <br>batch（5）  <br>nextlist（）  <br>curlist（＝＝＝）  <br>donelist（）|

由上面的状态图可知，当前处理的批次已经变成6，当然5是上次完成的那个批次，在CPU1上还没有处理完毕。此外，CPU0的donelist中的请求也会在本次tick handler中处理。如果请求较少，那么可以一次性处理完毕，这时候donelist是空，否则，会在几次tick handler中（tasklet context）处理完成。

7、CPU1的tick到来事件处理

这里留给大家自己看吧，其过程类似上节，我们这里只是给出状态：

|   |   |   |   |
|---|---|---|---|
|rcu_state|rcu_ctrlblk|CPU0的rcu_data|CPU1的rcu_data|
|cpumask（1，1）|Cur（6）  <br>Completed（5）  <br>next_pending（0）|Quiescbatch（6）  <br>passed_quiesc（0）  <br>qs_pending（1）  <br>batch（6）  <br>nextlist（）  <br>curlist（**）  <br>donelist（）|Quiescbatch（6）  <br>passed_quiesc（0）  <br>qs_pending（1）  <br>batch（5）  <br>nextlist（）  <br>curlist（）  <br>donelist（＝＝＝）|

这时候，CPU1上的batch等于5没有任何实际的意义，因为curlist为空了。

8、CPU1上有新的RCU请求

状态如下：

|   |   |   |   |
|---|---|---|---|
|rcu_state|rcu_ctrlblk|CPU0的rcu_data|CPU1的rcu_data|
|cpumask（1，1）|Cur（6）  <br>Completed（5）  <br>next_pending（0）|Quiescbatch（6）  <br>passed_quiesc（0）  <br>qs_pending（1）  <br>batch（6）  <br>nextlist（）  <br>curlist（**）  <br>donelist（）|Quiescbatch（6）  <br>passed_quiesc（0）  <br>qs_pending（1）  <br>batch（5）  <br>nextlist（****）  <br>curlist（）  <br>donelist（）|

9、CPU0的tick再次到来

只是简单的检测QS状态，不过发现passed_quiesc是0就失望而归了，具体代码大家自己过吧。

10、CPU1的tick再次到来

由于curlist为空并且有新的请求，因此nextlist中的请求又需要移到curlist链表中，同时伴随的动作就是分配一个新的batch number 7，__rcu_process_callbacks相关代码如下：

>         rdp->batch = rcp->cur + 1;－－－－－－－－－－－－－（1）  
>         smp_rmb(); －－－－－－－－－－－－－－－－－－－－（2）
> 
>         if (!rcp->next_pending) {－－－－－－－－－－－－－－－（3）  
>             spin_lock(&rsp->lock);  
>             rcu_start_batch(rcp, rsp, 1);－－－－－－－－－－－－（4）  
>             spin_unlock(&rsp->lock);  
>         }

（1）新的批次号总是current批次号加一，因此7号批次横空出世，请求挂在本cpu的curlist中。

（2）这里的memory barrier需要结合rcu_start_batch的代码来看，我们整理下面的表格：

|   |   |
|---|---|
|CPU A执行分配一个新批次的代码|CPU B执行rcu_start_batch的代码|
|rdp->batch = rcp->cur + 1;－－－－（X）  <br>  <br>smp_rmb();<br><br>if (!rcp->next_pending) {－－－－－（Y）  <br>……启动新的batch  <br>  }|rcp->next_pending = 0;  <br>  <br>smp_wmb();<br><br>  <br>rcp->cur++;|

在CPU B上，真正启动了一个新的批次，并且smp_wmb的memory barrier保证了先store rcp->next_pending然后store rcp->cur。站在CPU A的角度观察，如果在X处对cur load的值是CPU B中加一之后的值，那么smp_rmp可以保证在Y处，CPU A对next_pending的加载操作必定得到0值。只有这样，才能保证进入rcu_start_batch，从而启动一个新的批次。

（3）如果已经有了pending的批次，那么无论哪一个CPU，只要当前的批次没有渡过GP，那么任何后续的新分配的批次号都是7（只有完成GP之后cur才会加一）。如果已经pending，那么其实是没有必要调用rcu_start_batch函数去启动一个新的批次的。

（4）虽然7号批次横空出世，但是是否能启动GP探测呢？当然不能，目前current的6号批次还没有渡过GP，各个CPU还在不断的检测其passed_quiesc呢。因此，这里的rcu_start_batch过程执行如下：

> if (next_pending)  
>     rcp->next_pending = 1; －－－－－－－因为有了7号批次，因此设定pending标志
> 
> if (rcp->next_pending &&  
>         rcp->completed == rcp->cur) {－－－－－completed是5，而current是6，因此是false  
>     ……  
> }

completed是5，而current是6，这就意味着当前批次还没有完成GP探测，不能启动新的批次。至此，我们可以得到当前的状态如下： 

|   |   |   |   |
|---|---|---|---|
|rcu_state|rcu_ctrlblk|CPU0的rcu_data|CPU1的rcu_data|
|cpumask（1，1）|Cur（6）  <br>Completed（5）  <br>next_pending（1）|Quiescbatch（6）  <br>passed_quiesc（0）  <br>qs_pending（1）  <br>batch（6）  <br>nextlist（）  <br>curlist（**）  <br>donelist（）|Quiescbatch（6）  <br>passed_quiesc（0）  <br>qs_pending（1）  <br>batch（7）  <br>nextlist（）  <br>curlist（****）  <br>donelist（）|

这时候，系统中的当前批次是6号批次，正在检测GP，5号批次已经完成，而7号批次是处于pending状态，一旦6号批次完成，7号就可以上位，成为当前批次。

11、6号批次的GP过去之后会怎样？

我们假设CPU0和CPU1的QS都已经通过，也就是说在某个CPU（最后那个通过QS的）cpu_quiet函数中会启动新的批次，这时候的代码如下：

> static void cpu_quiet(int cpu, struct rcu_ctrlblk *rcp, struct rcu_state *rsp)  
> {  
>     cpu_clear(cpu, rsp->cpumask);  
>     if (cpus_empty(rsp->cpumask)) {  
>         rcp->completed = rcp->cur;  
>         rcu_start_batch(rcp, rsp, 0);  
>     }  
> }

这时候，  rcu_start_batch函数传递的next_pending参数是0，也就是说不标记pending状态（又不是向curlist中提交请求，当然不会标记pending了）。但是，实际上RCU控制块中的pending已经置位了，这时候rcu_start_batch中的条件成立，因此会立刻启动7号批次。

12、如果你愿意，你可以设定任何的事件，然后走读代码，推动系统的状态持续运行，本文的分析就到此为止了。

六、参考文献

1、linux2.6.11 source code

2、开源社区提交的patch说明

_原创文章，转发请注明出处。蜗窝科技_

标签: [Read-Copy-Update](http://www.wowotech.net/tag/Read-Copy-Update)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [显示技术介绍(3)_CRT技术](http://www.wowotech.net/display/crt_intro.html) | [Linux 2.5.43版本的RCU实现（废弃）](http://www.wowotech.net/kernel_synchronization/Linux-2-5-43-RCU.html)»

**评论：**

**fear**  
2019-04-04 19:22

static void __rcu_offline_cpu(struct rcu_data *this_rdp,  
    struct rcu_ctrlblk *rcp, struct rcu_state *rsp, struct rcu_data *rdp)  
{  
      
    spin_lock_bh(&rsp->lock);  
    if (rcp->cur != rcp->completed)  
        cpu_quiet(rdp->cpu, rcp, rsp);  
    spin_unlock_bh(&rsp->lock);  
        // 还有就是这里有个疑问：如果offline的cpu的curlist上的callback属于下一个batch  
        // 而当前cpu上的curlist属于当前正在被检测的batch  
        // 这样不会导致没有经过GP的callback被移动到donelist吗？  
        // 因为一旦当前batch结束，offline上的callback就会被一起放入donelist了。  
    rcu_move_batch(this_rdp, rdp->curlist, rdp->curtail);  
    rcu_move_batch(this_rdp, rdp->nxtlist, rdp->nxttail);  
  
}

[回复](http://www.wowotech.net/kernel_synchronization/linux2-6-11-RCU.html#comment-7345)

**fear**  
2019-04-04 19:11

static void __rcu_process_callbacks(struct rcu_ctrlblk *rcp,  
            struct rcu_state *rsp, struct rcu_data *rdp)  
{  
    ....  
                // 如果两个cpu同时进入了下面的if块，后执行rcu_start_batch的cpu会再次设置  
                // next_pending，但是其实这两个cpu的curlist上的callback应该属于同一个batch  
                // 这样不会start一次无用的batch吗  
        if (!rcp->next_pending) {  
            /* and start it/schedule start if it's a new batch */  
            spin_lock(&rsp->lock);  
            rcu_start_batch(rcp, rsp, 1);  
            spin_unlock(&rsp->lock);  
        }

[回复](http://www.wowotech.net/kernel_synchronization/linux2-6-11-RCU.html#comment-7344)

**goku**  
2019-02-25 10:35

我看了这个版本的代码，有个疑问想问一下，为什么rcu_ctrlblk的cur要初始化为-300？这点真想不明白。  
struct rcu_ctrlblk rcu_ctrlblk =  { .cur = -300, .completed = -300 };代码是这样写的。。

[回复](http://www.wowotech.net/kernel_synchronization/linux2-6-11-RCU.html#comment-7203)

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
    
    - [linux cpufreq framework(4)_cpufreq governor](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html)
    - [Process Creation（二）](http://www.wowotech.net/process_management/process-creation-2.html)
    - [Why Memory Barriers中文翻译（下）](http://www.wowotech.net/kernel_synchronization/why-memory-barrier-2.html)
    - [mmap在arm64背后的那个深坑](http://www.wowotech.net/linux_kenrel/516.html)
    - [Linux I2C framework(2)_I2C provider](http://www.wowotech.net/comm/i2c_provider.html)
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
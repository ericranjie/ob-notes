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

作者：[smcdef](http://www.wowotech.net/author/531) 发布于：2018-10-7 17:36 分类：[进程管理](http://www.wowotech.net/sort/process_management)

首先需要思考的问题是：什么是调度器（scheduler）？调度器的作用是什么？调度器是一个操作系统的核心部分。可以比作是CPU时间的管理员。调度器主要负责选择某些就绪的进程来执行。不同的调度器根据不同的方法挑选出最适合运行的进程。目前Linux支持的调度器就有RT scheduler、Deadline scheduler、CFS scheduler及Idle scheduler等。我想用一系列文章呈现Linux 调度器的设计原理。

> 注：文章代码分析基于Linux-4.18.0。

## 什么是调度类

从Linux 2.6.23开始，Linux引入scheduling class的概念，目的是将调度器模块化。这样提高了扩展性，添加一个新的调度器也变得简单起来。一个系统中还可以共存多个调度器。在Linux中，将调度器公共的部分抽象，使用`struct sched_class`结构体描述一个具体的调度类。系统核心调度代码会通过`struct sched_class`结构体的成员调用具体调度类的核心算法。先简单的介绍下`struct sched_class`部分成员作用。

1. struct sched_class {
2. 	const struct sched_class *next;
3. 	void (*enqueue_task) (struct rq *rq, struct task_struct *p, int flags);
4. 	void (*dequeue_task) (struct rq *rq, struct task_struct *p, int flags);
5. 	void (*check_preempt_curr)(struct rq *rq, struct task_struct *p, int flags);
6. 	struct task_struct * (*pick_next_task)(struct rq *rq, struct task_struct *prev, struct rq_flags *rf);
7.     /* ... */
8. }; 

> 1. next：next成员指向下一个调度类（比自己低一个优先级）。在Linux中，每一个调度类都是有明确的优先级关系，高优先级调度类管理的进程会优先获得cpu使用权。
> 2. enqueue_task：向该调度器管理的runqueue中添加一个进程。我们把这个操作称为入队。
> 3. dequeue_task：向该调度器管理的runqueue中删除一个进程。我们把这个操作称为出队。
> 4. check_preempt_curr：当一个进程被唤醒或者创建的时候，需要检查当前进程是否可以抢占当前cpu上正在运行的进程，如果可以抢占需要标记TIF_NEED_RESCHED flag。
> 5. pick_next_task：从runqueue中选择一个最适合运行的task。这也算是调度器比较核心的一个操作。例如，我们依据什么挑选最适合运行的进程呢？这就是每一个调度器需要关注的问题。

## Linux中有哪些调度类

Linux中主要包含`dl_sched_class`、`rt_sched_class`、`fair_sched_class`及`idle_sched_class`等调度类。每一个进程都对应一种调度策略，每一种调度策略又对应一种调度类（每一个调度类可以对应多种调度策略）。例如实时调度器以优先级为导向选择优先级最高的进程运行。每一个进程在创建之后，总是要选择一种调度策略。针对不同的调度策略，选择的调度器也是不一样的。不同的调度策略对应的调度类如下表。

|调度类|描述|调度策略|
|---|---|---|
|dl_sched_class|deadline调度器|SCHED_DEADLINE|
|rt_sched_class|实时调度器|SCHED_FIFO、SCHED_RR|
|fair_sched_class|完全公平调度器|SCHED_NORMAL、SCHED_BATCH|
|idle_sched_class|idle task|SCHED_IDLE|

针对以上调度类，系统中有明确的优先级概念。每一个调度类利用next成员构建单项链表。优先级从高到低示意图如下：

1. sched_class_highest----->stop_sched_class
2.                          .next---------->dl_sched_class
3.                                          .next---------->rt_sched_class
4.                                                          .next--------->fair_sched_class
5.                                                                         .next----------->idle_sched_class
6.                                                                                          .next = NULL 

Linux调度核心在选择下一个合适的task运行的时候，会按照优先级的顺序便利调度类的pick_next_task函数。因此，SCHED_FIFO调度策略的实时进程永远比SCHED_NORMAL调度策略的普通进程优先运行。代码中pick_next_task函数也有体现。pick_next_task函数就是负责选择一个即将运行的进程，以下贴出省略版代码。

1. static inline struct task_struct *pick_next_task(struct rq *rq,
2.                                                  struct task_struct *prev, struct rq_flags *rf)
3. {
4. 	const struct sched_class *class;
5. 	struct task_struct *p;

7. 	for_each_class(class) {          /* 按照优先级顺序便利所有的调度类，通过next指针便利单链表 */
8. 		p = class->pick_next_task(rq, prev, rf);
9. 		if (p)
10. 			return p;
11. 	}
12. } 

针对CFS调度器，管理的进程都属于SCHED_NORMAL或者SCHED_BATCH策略。后面的部分主要针对CFS调度器讲解。

## 普通进程的优先级

CFS是Completely Fair Scheduler简称，即完全公平调度器。CFS的设计理念是在真实硬件上实现理想的、精确的多任务CPU。CFS调度器和以往的调度器不同之处在于没有时间片的概念，而是分配cpu使用时间的比例。例如：2个相同优先级的进程在一个cpu上运行，那么每个进程都将会分配50%的cpu运行时间。这就是要实现的公平。

以上举例是基于同等优先级的情况下。但是现实却并非如此，有些任务优先级就是比较高。那么CFS调度器的优先级是如何实现的呢？首先，我们引入权重的概念，权重代表着进程的优先级。各个进程之间按照权重的比例分配cpu时间。例如：2个进程A和B。A的权重是1024，B的权重是2048。那么A获得cpu的时间比例是1024/(1024+2048) = 33.3%。B进程获得的cpu时间比例是2048/(1024+2048)=66.7%。我们可以看出，权重越大分配的时间比例越大，相当于优先级越高。在引入权重之后，分配给进程的时间计算公式如下：

**分配给进程的时间 = 总的cpu时间 * 进程的权重/就绪队列（runqueue）所有进程权重之和**

CFS调度器针对优先级又提出了nice值的概念，其实和权重是一一对应的关系。nice值就是一个具体的数字，取值范围是[-20, 19]。数值越小代表优先级越大，同时也意味着权重值越大，nice值和权重之间可以互相转换。内核提供了一个表格转换nice值和权重。

1. const int sched_prio_to_weight[40] = {
2.  /* -20 */     88761,     71755,     56483,     46273,     36291,
3.  /* -15 */     29154,     23254,     18705,     14949,     11916,
4.  /* -10 */      9548,      7620,      6100,      4904,      3906,
5.  /*  -5 */      3121,      2501,      1991,      1586,      1277,
6.  /*   0 */      1024,       820,       655,       526,       423,
7.  /*   5 */       335,       272,       215,       172,       137,
8.  /*  10 */       110,        87,        70,        56,        45,
9.  /*  15 */        36,        29,        23,        18,        15,
10. }; 

数组的值可以看作是公式：weight = 1024 / 1.25nice计算得到。公式中的1.25取值依据是：进程每降低一个nice值，将多获得10% cpu的时间。公式中以1024权重为基准值计算得来，1024权重对应nice值为0，其权重被称为NICE_0_LOAD。默认情况下，大部分进程的权重基本都是NICE_0_LOAD。

## 调度延迟

什么是调度延迟？调度延迟就是保证每一个可运行进程都至少运行一次的时间间隔。例如，每个进程都运行10ms，系统中总共有2个进程，那么调度延迟就是20ms。如果有5个进程，那么调度延迟就是50ms。如果现在保证调度延迟不变，固定是6ms，那么系统中如果有2个进程，那么每个进程运行3ms。如果有6个进程，那么每个进程运行1ms。如果有100个进程，那么每个进程分配到的时间就是0.06ms。随着进程的增加，每个进程分配的时间在减少，进程调度过于频繁，上下文切换时间开销就会变大。因此，CFS调度器的调度延迟时间的设定并不是固定的。当系统处于就绪态的进程少于一个定值（默认值8）的时候，调度延迟也是固定一个值不变（默认值6ms）。当系统就绪态进程个数超过这个值时，我们保证每个进程至少运行一定的时间才让出cpu。这个“至少一定的时间”被称为最小粒度时间。在CFS默认设置中，最小粒度时间是0.75ms。用变量sysctl_sched_min_granularity记录。因此，调度周期是一个动态变化的值。调度周期计算函数是__sched_period()。

1. static u64 __sched_period(unsigned long nr_running)
2. {
3. 	if (unlikely(nr_running > sched_nr_latency))
4. 		return nr_running * sysctl_sched_min_granularity;
5. 	else
6. 		return sysctl_sched_latency;
7. } 

> nr_running是系统中就绪进程数量，当超过sched_nr_latency时，我们无法保证调度延迟，因此转为保证调度最小粒度。如果nr_running并没有超过sched_nr_latency，那么调度周期就等于调度延迟sysctl_sched_latency（6ms）。

## 虚拟时间（virtual time）

CFS调度器的目标是保证每一个进程的完全公平调度。CFS调度器就像是一个母亲，她有很多个孩子（进程）。但是，手上只有一个玩具（cpu）需要公平的分配给孩子玩。假设有2个孩子，那么一个玩具怎么才可以公平让2个孩子玩呢？简单点的思路就是第一个孩子玩10分钟，然后第二个孩子玩10分钟，以此循环下去。CFS调度器也是这样记录每一个进程的执行时间，保证每个进程获取CPU执行时间的公平。因此，哪个进程运行的时间最少，应该让哪个进程运行。

例如，调度周期是6ms，系统一共2个相同优先级的进程A和B，那么每个进程都将在6ms周期时间内内各运行3ms。如果进程A和B，他们的权重分别是1024和820（nice值分别是0和1）。进程A获得的运行时间是6x1024/(1024+820)=3.3ms，进程B获得的执行时间是6x820/(1024+820)=2.7ms。进程A的cpu使用比例是3.3/6x100%=55%，进程B的cpu使用比例是2.7/6x100%=45%。计算结果也符合上面说的“进程每降低一个nice值，将多获得10% CPU的时间”。很明显，2个进程的实际执行时间是不相等的，但是CFS想保证每个进程运行时间相等。因此CFS引入了虚拟时间的概念，也就是说上面的2.7ms和3.3ms经过一个公式的转换可以得到一样的值，这个转换后的值称作虚拟时间。这样的话，CFS只需要保证每个进程运行的虚拟时间是相等的即可。虚拟时间vriture_runtime和实际时间（wall time）转换公式如下：

1.                                  NICE_0_LOAD
2. vriture_runtime = wall_time * ----------------
3.                                     weight 

进程A的虚拟时间3.3 * 1024 / 1024 = 3.3ms，我们可以看出nice值为0的进程的虚拟时间和实际时间是相等的。进程B的虚拟时间是2.7 * 1024 / 820 = 3.3ms。我们可以看出尽管A和B进程的权重值不一样，但是计算得到的虚拟时间是一样的。因此CFS主要保证每一个进程获得执行的虚拟时间一致即可。在选择下一个即将运行的进程的时候，只需要找到虚拟时间最小的进程即可。为了避免浮点数运算，因此我们采用先放大再缩小的方法以保证计算精度。内核又对公式做了如下转换。

1.                                  NICE_0_LOAD
2. vriture_runtime = wall_time * ----------------
3.                                     weight

5.                                    NICE_0_LOAD * 2^32
6.                 = (wall_time * -------------------------) >> 32
7.                                         weight
8.                                                                                         2^32
9.                 = (wall_time * NICE_0_LOAD * inv_weight) >> 32        (inv_weight = ------------ )
10.                                                                                         weight 

权重的值已经计算保存到sched_prio_to_weight数组中，根据这个数组我们可以很容易计算inv_weight的值。内核中使用sched_prio_to_wmult数组保存inv_weight的值。计算公式是：sched_prio_to_wmult[i] = 232/sched_prio_to_weight[i]。

1. const u32 sched_prio_to_wmult[40] = {
2.  /* -20 */     48388,     59856,     76040,     92818,    118348,
3.  /* -15 */    147320,    184698,    229616,    287308,    360437,
4.  /* -10 */    449829,    563644,    704093,    875809,   1099582,
5.  /*  -5 */   1376151,   1717300,   2157191,   2708050,   3363326,
6.  /*   0 */   4194304,   5237765,   6557202,   8165337,  10153587,
7.  /*   5 */  12820798,  15790321,  19976592,  24970740,  31350126,
8.  /*  10 */  39045157,  49367440,  61356676,  76695844,  95443717,
9.  /*  15 */ 119304647, 148102320, 186737708, 238609294, 286331153,
10. }; 

系统中使用`struct load_weight`结构体描述进程的权重信息。weight代表进程的权重，inv_weight等于232/weight。

1. struct load_weight {
2. 	unsigned long		weight;
3. 	u32			inv_weight;
4. }; 

将实际时间转换成虚拟时间的实现函数是calc_delta_fair()。calc_delta_fair()调用__calc_delta()函数，__calc_delta()主要功能是实现如下公式的计算。

1. __calc_delta() = (delta_exec * weight * lw->inv_weight) >> 32

3.                                   weight                                 2^32
4.                = delta_exec * ----------------    (lw->inv_weight = --------------- )
5.                                 lw->weight                             lw->weight 

和上面计算虚拟时间计算公式对比发现。如果需要计算进程的虚拟时间，这里的weight只需要传递参数NICE_0_LOAD，lw参数是进程对应的`struct load_weight`结构体。

1. static u64 __calc_delta(u64 delta_exec, unsigned long weight, struct load_weight *lw)
2. {
3. 	u64 fact = scale_load_down(weight);
4. 	int shift = 32;

6. 	__update_inv_weight(lw);

8. 	if (unlikely(fact >> 32)) {
9. 		while (fact >> 32) {
10. 			fact >>= 1;
11. 			shift--;
12. 		}
13. 	}

15. 	fact = (u64)(u32)fact * lw->inv_weight;

17. 	while (fact >> 32) {
18. 		fact >>= 1;
19. 		shift--;
20. 	}

22. 	return mul_u64_u32_shr(delta_exec, fact, shift);
23. } 

按照上面说的理论，calc_delta_fair()函数调用__calc_delta()的时候传递的weight参数是NICE_0_LOAD，lw参数是进程对应的`struct load_weight`结构体。

1. static inline u64 calc_delta_fair(u64 delta, struct sched_entity *se)
2. {
3. 	if (unlikely(se->load.weight != NICE_0_LOAD))             /* 1 */
4. 		delta = __calc_delta(delta, NICE_0_LOAD, &se->load);  /* 2 */

6. 	return delta;
7. } 

> 1. 按照之前的理论，nice值为0（权重是NICE_0_LOAD）的进程的虚拟时间和实际时间是相等的。因此如果进程的权重是NICE_0_LOAD，进程对应的虚拟时间就不用计算。
> 2. 调用__calc_delta()函数。

Linux通过`struct task_struct`结构体描述每一个进程。但是调度类管理和调度的单位是调度实体，并不是task_struct。在支持组调度的时候，一个组也会抽象成一个调度实体，它并不是一个task。所以，我们在`struct task_struct`结构体中可以找到以下不同调度类的调度实体。

1. struct task_struct {
2.          struct sched_entity		se;
3. 	struct sched_rt_entity		rt;
4. 	struct sched_dl_entity		dl;
5.     /* ... */
6. } 

> se、rt、dl分别对应CFS调度器、RT调度器、Deadline调度器的调度实体。

`struct sched_entity`结构体描述调度实体，包括`struct load_weight`用来记录权重信息。除此以外我们一直关心的时间信息，肯定也要一起记录。`struct sched_entity`结构体简化后如下：

1. struct sched_entity {
2. 	struct load_weight		load;
3. 	struct rb_node		run_node;
4. 	unsigned int		on_rq;
5. 	u64			sum_exec_runtime;
6. 	u64			vruntime;
7. }; 

> 1. load：权重信息，在计算虚拟时间的时候会用到inv_weight成员。
> 2. run_node：CFS调度器的每个就绪队列维护了一颗红黑树，上面挂满了就绪等待执行的task，run_node就是挂载点。
> 3. on_rq：调度实体se加入就绪队列后，on_rq置1。从就绪队列删除后，on_rq置0。
> 4. sum_exec_runtime：调度实体已经运行实际时间总合。
> 5. vruntime：调度实体已经运行的虚拟时间总合。

## 就绪队列（runqueue）

系统中每个CPU都会有一个全局的就绪队列（cpu runqueue），使用`struct rq`结构体描述，它是per-cpu类型，即每个cpu上都会有一个`struct rq`结构体。每一个调度类也有属于自己管理的就绪队列。例如，`struct cfs_rq`是CFS调度类的就绪队列，管理就绪态的`struct sched_entity`调度实体，后续通过pick_next_task接口从就绪队列中选择最适合运行的调度实体（虚拟时间最小的调度实体）。`struct rt_rq`是实时调度器就绪队列。`struct dl_rq`是Deadline调度器就绪队列。

1. struct rq {
2.          struct cfs_rq cfs;
3. 	struct rt_rq rt;
4. 	struct dl_rq dl;
5. };

7. struct rb_root_cached {
8. 	struct rb_root rb_root;
9. 	struct rb_node *rb_leftmost;
10. };

12. struct cfs_rq {
13. 	struct load_weight load;
14. 	unsigned int nr_running;
15. 	u64 min_vruntime;
16. 	struct rb_root_cached tasks_timeline;
17. }; 

> 1. load：就绪队列权重，就绪队列管理的所有调度实体权重之和。
> 2. nr_running：就绪队列上调度实体的个数。
> 3. min_vruntime：跟踪就绪队列上所有调度实体的最小虚拟时间。
> 4. tasks_timeline：用于跟踪调度实体按虚拟时间大小排序的红黑树的信息（包含红黑树的根以及红黑树中最左边节点）。

CFS维护了一个按照虚拟时间排序的红黑树，所有可运行的调度实体按照p->se.vruntime排序插入红黑树。如下图所示。

[![RB_tree.png](http://www.wowotech.net/content/uploadfile/201810/afa01538905321.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201810/afa01538905321.png)

CFS选择红黑树最左边的进程运行。随着系统时间的推移，原来左边运行过的进程慢慢的会移动到红黑树的右边，原来右边的进程也会最终跑到最左边。因此红黑树中的每个进程都有机会运行。

现在我们总结一下。Linux中所有的进程使用`task_struct`描述。`task_struct`包含很多进程相关的信息（例如，优先级、进程状态以及调度实体等）。但是，每一个调度类并不是直接管理`task_struct`，而是引入调度实体的概念。CFS调度器使用`sched_entity`跟踪调度信息。CFS调度器使用`cfs_rq`跟踪就绪队列信息以及管理就绪态调度实体，并维护一棵按照虚拟时间排序的红黑树。`tasks_timeline->rb_root`是红黑树的根，`tasks_timeline->rb_leftmost`指向红黑树中最左边的调度实体，即虚拟时间最小的调度实体（为了更快的选择最适合运行的调度实体，因此`rb_leftmost`相当于一个缓存）。每个就绪态的调度实体`sched_entity`包含插入红黑树中使用的节点`rb_node`，同时`vruntime`成员记录已经运行的虚拟时间。我们将这几个数据结构简单梳理，如下图所示。

[](http://www.wowotech.net/content/uploadfile/201810/8bb51538905306.png)[![Structure_hierarchy.png](http://www.wowotech.net/content/uploadfile/201810/8bb51538905306.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201810/8bb51538905306.png)

标签: [CFS](http://www.wowotech.net/tag/CFS)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [CFS调度器（2）-源码解析](http://www.wowotech.net/process_management/448.html) | [per-entity load tracking](http://www.wowotech.net/process_management/PELT.html)»

**评论：**

**[摩斯电码](http://www.wowotech.net/)**  
2022-05-24 22:43

SCHED_IDLE属于fair_sched_class，而不是idle_sched_class

[回复](http://www.wowotech.net/process_management/447.html#comment-8621)

**hahaha**  
2022-03-15 20:05

请教大家一个问题：  
  
在我的理解中，cfs的_sched_period是使每个任务都能运行一次的时间。当进程数小于sched_nr_latency_延迟时，CFS会将时间分割为每个进程预计运行一次的时段  
  
u64 __sched_period(unsigned long nr_running)  
{  
    if (unlikely(nr_running > sched_nr_latency))  
        return nr_running * sysctl_sched_min_granularity;  
    else  
        return sysctl_sched_latency;  
}  
  
1. 如果有两个具有相同权重的cfs进程，每个进程的 最大 运行时间是否为“sysctl_sched_latency/2”(进程可以不运行这么多)，还是说我们是否必须为每个进程执行“sysctl_sched_latency/2”？  
2. 那么上下文切换时间、调度程序成本计算和其他因素产生的时间呢? sched_period到底由哪些东西构成 调度延时/调度周期存在的意义是什么，感觉每时每刻都会有进程从TASK_RUNNING转变成睡眠，nr_running会不断变化，那每次被调度的进程进来算调度时期的值都不一样...

[回复](http://www.wowotech.net/process_management/447.html#comment-8573)

**[linuxer](http://www.wowotech.net/)**  
2022-03-17 09:24

@hahaha：target latency(sched period)本意是保证CPU runqueue上的任务的最小调度延迟，即在target latency的时间内，任务至少被调度一次。目标如此，但是目前的代码不能保证这一点。  
  
诚如你所说，系统的状态不断的变迁，其实sched period和slice本来就是一个不断变化的值，调度器是在tick中不断的检查当前任务的时间片是否耗尽，如果耗尽那么就抢占。tick的精度本来就不高（不考虑sched hrtick），那么我们其实不能精准的控制每次任务执行时间精准的等于其slice的。  
  
此外，如果有异步事件唤醒了其他的任务，那么还要考虑wakeup preempt，这也会影响任务的一次执行时间

[回复](http://www.wowotech.net/process_management/447.html#comment-8575)

**二仙桥老叟**  
2021-12-10 22:07

博主真是大聪明，文章一看就明白原理

[回复](http://www.wowotech.net/process_management/447.html#comment-8404)

**[bsp](http://www.wowotech.net/)**  
2020-07-31 09:27

‘SCHED_FIFO调度策略的实时进程永远比SCHED_NORMAL调度策略的普通进程优先运行’有一个例外场景：  
sysctl sched_rt_runtime_us/sched_rtperiod_us, 可以控制RT任务占用cpu的比例，默认95%，也就是cpu还有%5的时间给非RT任务（normal task）执行；从而是系统更好的运行。

[回复](http://www.wowotech.net/process_management/447.html#comment-8077)

**[linuxer](http://www.wowotech.net/)**  
2020-07-30 21:06

一个小小的bug，SCHED_IDLE其实是属于CFS 调度类的，而不是idle调度类。  
  
CFS调度类实际上有三种调度策略：  
1、 SCHED_NORMAL策略（在用户空间中称为SCHED_OTHER），适用于用于Linux环境中运行的大多数任务。  
2、 SCHED_BATCH策略，用于非交互式任务的批处理，这些任务会在一段时间内不间断地运行（cpu-bound），通常只有在完成所有SCHED_NORMAL任务之后才进行调度。  
3、 SCHED_IDLE策略是为系统中优先级最低的任务设计的，只有在没有其他任务可运行时，这些任务才有机会运行。  
  
目前的CFS调度器，SCHED_NORMAL和SCHED_BATCH基本是一样的（仅仅在yield CPU时候处理不同），SCHED_IDLE是很有意思的策略，不过，标准内核并没有真正实现SCHED_IDLE任务的语义，具体的实现逻辑是：  
1、 设定为优先级最低（139）  
2、 在check_preempt_curr（）回调函数中对SCHED_IDLE任务进行特殊处理，即当前正在运行的SCHED_IDLE任务将立即被新唤醒的SCHED_NORMAL任务抢占。  
  
从这个角度看，所有的CFS任务，无论是SCHED_NORMAL、SCHED_BATCH、还是SCHED_IDLE，都是在标准CFS框架下运作的。例如当一个SCHED_NORMAL和SCHED_IDLE放置到一个cpu runqueue的时候，SCHED_NORMAL并不会一直执行，SCHED_IDLE也会占据一定的CPU时间，只不过很短。

[回复](http://www.wowotech.net/process_management/447.html#comment-8076)

**yyll**  
2020-04-02 12:32

vruntime是一个虚拟的时钟，  
1. 在一个调度周期内T，其关联任务的步长Step和它的权重weight有关系，  
步长公式:  
    step = T * NICE_0_LOAD/weight, NICE_0_LOAD为参考虚拟时钟，与物理时钟一致  
2. 在一个调度周期内，vruntime单调递增，调度周期结束后，所有任务的vruntime都会归一到一个相近的初始值，然后又是一个循环。  
CFS的核心思想，保证在一个CPU上的一个调度周期内，所有的任务调度时间的总和是相等的，这是通过调节各个task的虚拟时钟的步长做到的，如上的step公式。  
1. 优先级高的task，它的步长就小，在一个CPU的调度周期内，调用的次数就多。  
2. 优先级低的task，它的步长就长，在一个CPU的调度周期内，调用的次数就少。

[回复](http://www.wowotech.net/process_management/447.html#comment-7937)

**tandf**  
2020-02-10 16:03

您好，如果红黑树只用于寻找最小的元素，为什么不使用最小堆呢？非常感谢

[回复](http://www.wowotech.net/process_management/447.html#comment-7872)

**StevenSun**  
2021-04-07 23:48

@tandf：事实上考虑的不只是寻找最小的元素. 还可以参考这里 https://www.zhihu.com/question/27840936

[回复](http://www.wowotech.net/process_management/447.html#comment-8218)

**see**  
2019-10-31 11:27

因此CFS主要保证每一个进程获得执行的虚拟时间一致即可。在选择下一个即将运行的进程的时候，只需要找到虚拟时间最小的进程即可-----前面说保证虚拟时间相同，后面说选择虚拟时间最小的，如果相同了，拿来最小虚拟时间，感觉自相矛盾？请大神解释下

[回复](http://www.wowotech.net/process_management/447.html#comment-7730)

**see**  
2019-10-31 11:28

@see：-前面说保证虚拟时间相同，后面说选择虚拟时间最小的，如果相同了，哪来的最小虚拟时间，感觉自相矛盾？

[回复](http://www.wowotech.net/process_management/447.html#comment-7731)

**ccilery**  
2021-09-13 16:43

@see：我的理解是CFS保证一个调度周期/延迟内，就绪队列上的所有任务获得相同的虚拟时间执行，但实际的墙上时间还是不同的（高权重的任务执行时间长）；虽然在一个调度周期内，就绪队列上的任务虚拟时间的增量是相同的，但是因为新任务的创建以及唤醒任务的加入，会导致红黑树上调度实体虚拟时间的差异，所以会有最小虚拟时间的任务。但我觉得这都是理想情况下，现实中一个调度周期内就绪队列是静态的吗，我觉得一个调度周期中就绪队列都是动态的（有任务删除或加入），如何保证每个任务有相同的虚拟时间？

[回复](http://www.wowotech.net/process_management/447.html#comment-8312)

**abcdefg345**  
2021-09-18 10:19

@ccilery：当就绪队列有任务增加、删除时，需要考虑的只是当前任务要不要停止、或者还可以运行多久的问题。因此，每次就绪队列有任务增加、删除时，更新当前任务的虚拟时间，并按新的就绪任务情况来重新计算当前任务的时间片。

[回复](http://www.wowotech.net/process_management/447.html#comment-8316)

**湫**  
2019-10-31 22:00

@see：保证只是说明调度器想要做的事是去保证虚拟运行时间相同。选择最小的虚拟时间运行就是为了保证这一点，让小增大一些使其和大的一样。

[回复](http://www.wowotech.net/process_management/447.html#comment-7732)

**figo**  
2019-11-01 10:54

@湫：请教下什么叫最小的虚拟时间？从文章来看，虚拟时间根据公式推算出来就是相等的，那这个"最小虚拟时间"的概念是从何而来？

[回复](http://www.wowotech.net/process_management/447.html#comment-7734)

**湫**  
2019-11-01 17:06

@figo：准确的说是虚拟运行时间（vruntime），表示的是进程在某种意义上的运行了多长时间，所以它是动态的。虚拟运行时间可以根据那个公式由真实运行时间来转换得到。最小虚拟运行时间是相对于一个就绪队列来说的。就是一个就绪队列上的就绪进程的虚拟运行时间最小的那个。

[回复](http://www.wowotech.net/process_management/447.html#comment-7736)

**figo**  
2019-11-01 10:52

@see：同问

[回复](http://www.wowotech.net/process_management/447.html#comment-7733)

**figo**  
2019-11-01 11:02

@figo：在其他文章里面找到了关于这块内容的解释：  
目前linux内核中所使用的调度器为CFS调度器[16, 26]，CFS定义了一种新的模型，它给CFS的运行队列中的每一个进程安排一个虚拟时钟，vruntime。如果一个进程得以执行，其vruntime将不断增大。没有得到执行的进程vruntime不变。而调度器总是选择vruntime最小的进程来执行。这就是所谓的“完全公平”。为了区别不同进程的优先级，优先级高的进程vruntime增长的较慢，以至于它可能得到更多的运行机会。

[回复](http://www.wowotech.net/process_management/447.html#comment-7735)

**[s](http://s/)**  
2019-11-12 15:37

@see：进程A的虚拟时间3.3 * 1024 / 1024 = 3.3ms，我们可以看出nice值为0的进程的虚拟时间和实际时间是相等的。进程B的虚拟时间是2.7 * 1024 / 820 = 3.3ms。  
如果进程B先执行，执行1ms后发生调度，此时B的vtime=1*1024/820+3.3=4.5ms,  
此时A执行，A可以执行1.2*1024/1024+3.3=4.5。进程A可以执行1.2ms甚至更长一些。  
最终按照比例A/B 3.3/2.7的时间

[回复](http://www.wowotech.net/process_management/447.html#comment-7746)

**小亮**  
2019-12-18 14:23

@see：CFS设计之处的模型是理想的CPU模型，在这种模型中，CPU上的所有任务都是并行执行的，平分CPU的power, 虚拟CPU时间和物理时间是相等的， 但是在现实当中，这种模型是不存在的，CPU上任务是不可能同时并行执行，因此，会造成物理时间不相同，继而导致虚拟时间不相同的情况出现，得到执行权的任务，虚拟时间在不停的增加，没有得到执行权的并没有增加，因此，在每次挑选任务时，挑选虚拟时间最小的任务执行。  
我知道我的这种解释，是否正确？求证

[回复](http://www.wowotech.net/process_management/447.html#comment-7788)

**aly**  
2024-04-11 14:49

@see：cfs保证一个调度周期结束后所有进程的虚拟时间是一样的。  
  
调度周期内，不同进程会累积自己的虚拟时间，因此会有虚拟时间上的差别。

[回复](http://www.wowotech.net/process_management/447.html#comment-8880)

**[markened](http://www.wowotech.net/)**  
2019-04-29 10:23

大神，您好。 关于在CFS调度里面提到的调度延迟保持有界的延迟，这个最小调度时间sysctl_sched_min_granularity（0.75ms）指的是虚拟时间吧？我通过阅读这个系列第二篇的文章猜的，这个是否正确？  谢谢！

[回复](http://www.wowotech.net/process_management/447.html#comment-7381)

**[smcdef](http://www.wowotech.net/)**  
2019-04-29 16:15

@markened：0.75ms是wall time，不指virtual time

[回复](http://www.wowotech.net/process_management/447.html#comment-7385)

**[markened](http://www.wowotech.net/)**  
2019-05-11 18:51

@smcdef：那如果是这样的话，当进程很多的时候，有些优先级高的进程不会仅仅执行）0.75ms就切换成其他进程，那么调度延迟 nr_running * sysctl_sched_min_granularity 这个时间内也就没有办法保证所有的进程都调度一遍了吧。

[回复](http://www.wowotech.net/process_management/447.html#comment-7410)

**[smcdef](http://www.wowotech.net/)**  
2019-05-11 22:21

@markened：这个问题可以看下CFS调度器的最后一篇文章

[回复](http://www.wowotech.net/process_management/447.html#comment-7411)

**[markened](http://www.wowotech.net/)**  
2019-05-13 15:39

@smcdef：谢谢！是不是这样的，的确有可能会出现一个进程单次运行时间大于sysctl_sched_min_granularity。还是要根据vruntime判断在运行了时间大于sysctl_sched_min_granularity时是否要调度下一个进程。 也就是说调度延迟也许或大于 nr_running * sysctl_sched_min_granularity。大神，是否可以这样理解？

[回复](http://www.wowotech.net/process_management/447.html#comment-7414)

**[markened](http://www.wowotech.net/)**  
2019-05-13 18:06

@markened：se = __pick_first_entity(cfs_rq);  
    delta = curr->vruntime - se->vruntime;  
  
    if (delta < 0)  
        return;  
  
    if (delta > ideal_runtime)  
        resched_curr(rq_of(cfs_rq));  
从这段代码应该可以得到上面的结论？谢谢！

[回复](http://www.wowotech.net/process_management/447.html#comment-7415)

**[markened](http://www.wowotech.net/)**  
2019-05-15 17:23

@smcdef：以下推测能否成立，大神，给解解惑呗？

[回复](http://www.wowotech.net/process_management/447.html#comment-7421)

**[smcdef](http://www.wowotech.net/)**  
2019-05-18 14:57

@markened：如果进程nice值小于0，其单次运行时间自然就会大于sysctl_sched_min_granularity。这点来说，挺正常的。后半部分的问题描述我就有点看不懂问题了。你是好奇下面的代码的片段的意思吗？

[回复](http://www.wowotech.net/process_management/447.html#comment-7425)

**[markened](http://www.wowotech.net/)**  
2019-05-19 21:34

@smcdef：那调度周期岂不是有可能大于这个值 nr_running * sysctl_sched_min_granularity？

**[takahashien](http://www.google.com/)**  
2019-08-14 10:13

@smcdef：额，我的理解是， 内核先确定了一个sched_period， 是根据nr_running*sysctl_sched_min_granularity 来的。之后呢在根据自己的就绪队列里的调度实体的权重把上面这一大块时间分给就绪队列里的各个调度实体。对于se而言如果它权重比就绪队列里的其它的某个实体大的话，那它get到的实际时间就比它多些， 即从大块(sched_period)里分到的时间就多些， 最终整个就绪队列里进程都运行一遍的时间不会超过sched_period

**[takahashien](http://www.google.com/)**  
2019-08-14 10:19

@smcdef：对不起我回复错了。。。

**[takahashien](http://www.google.com/)**  
2019-08-14 10:27

@markened：额，我的理解是， 内核先确定了一个sched_period， 是根据nr_running*sysctl_sched_min_granularity 来的。之后呢在根据自己的就绪队列里的调度实体的权重把上面这一大块时间分给就绪队列里的各个调度实体。对于se而言如果它权重比就绪队列里的其它的某个实体大的话，那它get到的实际时间就比它多些， 即从大块(sched_period)里分到的时间就多些， 最终整个就绪队列里进程都运行一遍的时间不会超过sched_period

**sinai**  
2019-04-24 11:10

CPU为什么要有就绪队列呢？调度器选择好进程之后让CPU运行被调度器选择的进程不久可以了么？

[回复](http://www.wowotech.net/process_management/447.html#comment-7374)

**[smcdef](http://www.wowotech.net/)**  
2019-04-29 23:18

@sinai：根据你的描述问题，我反问一个问题吧！你说“调度器选择好进程之后......”,那么调度器怎么选择好进程呢？从哪里选择？为什么要选？  
  
既然你知道要选择进程，也就是你明白了调度器可以有多个选择，每个选择对应不同的进程。那么下个问题，调度器从哪里选择进程呢？这里的“哪里”其实就是就绪队列。就绪队列就是包含了调度器所有的所有选择。所以，我们需要就绪队列记录可以被选择运行的进程。

[回复](http://www.wowotech.net/process_management/447.html#comment-7387)

1 [2](http://www.wowotech.net/process_management/447.html/comment-page-2#comments)

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
    
    - ja  
        [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
    - 元神高手  
        [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
    - 十七  
        [内核空间的映射在系统启动时就已经设定好，并且在所有进程的页表...](http://www.wowotech.net/process_management/context-switch-arch.html#8920)
    - lw  
        [sparse模型和disconti模型没看出来有什么本质区别...](http://www.wowotech.net/memory_management/memory_model.html#8919)
    - 肥饶  
        [一个没设置好就出错](http://www.wowotech.net/linux_kenrel/516.html#8918)
    - orange  
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
    
    - [蓝牙协议分析(8)_BLE安全机制之白名单](http://www.wowotech.net/bluetooth/ble_white_list.html)
    - [防冲突机制介绍](http://www.wowotech.net/basic_tech/103.html)
    - [Linux cpufreq framework(2)_cpufreq driver](http://www.wowotech.net/pm_subsystem/cpufreq_driver.html)
    - [eMMC 原理 3 ：分区管理](http://www.wowotech.net/basic_tech/emmc_partitions.html)
    - [futex基础问答](http://www.wowotech.net/kernel_synchronization/futex.html)
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
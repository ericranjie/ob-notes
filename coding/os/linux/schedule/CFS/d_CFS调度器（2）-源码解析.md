作者：[smcdef](http://www.wowotech.net/author/531) 发布于：2018-10-21 20:55 分类：[进程管理](http://www.wowotech.net/sort/process_management)

经通过上一篇文章《CFS调度器-基本原理》，我们可以了解到CFS调度器基本工作原理。本篇文章主要集中在Linux CFS调度器源码解析。

> 注：文章代码分析基于Linux-4.18.0。

## 进程的创建

进程的创建是通过do_fork()函数完成。新进程的诞生，我们调度核心层会通知调度类，调用特别的接口函数初始化新生儿。我们一路尾随do_fork()函数。do_fork()---->_do_fork()---->copy_process()---->sched_fork()。针对sched_fork()函数，删减部分代码如下：

```c
 
int sched_fork(unsigned long clone_flags, struct task_struct *p){	p->state = TASK_NEW;	p->prio = current->normal_prio;	p->sched_class = &fair_sched_class;         /* 1 */ 	if (p->sched_class->task_fork)		p->sched_class->task_fork(p);           /* 2 */ 	return 0;}
```

> 1. 我们这里以CFS为例，为task选择调度类。fair_sched_class是CFS调度类。
> 2. 调用调度类中的task_fork函数。task_fork方法主要做fork相关的操作。传递的参数p就是创建的`task_struct`。

CFS调度类fair_sched_class方法如下：

```c
 
const struct sched_class fair_sched_class = {	.next				= &idle_sched_class,	.enqueue_task		= enqueue_task_fair,	.dequeue_task		= dequeue_task_fair,	.yield_task			= yield_task_fair,	.yield_to_task		= yield_to_task_fair,	.check_preempt_curr	= check_preempt_wakeup,	.pick_next_task		= pick_next_task_fair,	.put_prev_task		= put_prev_task_fair,#ifdef CONFIG_SMP	.select_task_rq		= select_task_rq_fair,	.migrate_task_rq	= migrate_task_rq_fair,	.rq_online			= rq_online_fair,	.rq_offline			= rq_offline_fair,	.task_dead			= task_dead_fair,	.set_cpus_allowed	= set_cpus_allowed_common,#endif	.set_curr_task      = set_curr_task_fair,	.task_tick			= task_tick_fair,	.task_fork			= task_fork_fair,	.prio_changed		= prio_changed_fair,	.switched_from		= switched_from_fair,	.switched_to		= switched_to_fair,	.get_rr_interval	= get_rr_interval_fair,	.update_curr		= update_curr_fair,#ifdef CONFIG_FAIR_GROUP_SCHED	.task_change_group	= task_change_group_fair,#endif};
```

task_fork_fair实现如下：

```c
 
static void task_fork_fair(struct task_struct *p){	struct cfs_rq *cfs_rq;	struct sched_entity *se = &p->se, *curr;	struct rq *rq = this_rq();	struct rq_flags rf; 	rq_lock(rq, &rf);	update_rq_clock(rq); 	cfs_rq = task_cfs_rq(current);	curr = cfs_rq->curr;                     /* 1 */	if (curr) {		update_curr(cfs_rq);                 /* 2 */		se->vruntime = curr->vruntime;       /* 3 */	}	place_entity(cfs_rq, se, 1);             /* 4 */ 	se->vruntime -= cfs_rq->min_vruntime;    /* 5 */	rq_unlock(rq, &rf);}
```

> 1. cfs_rq是CFS调度器就绪队列，curr指向当前正在cpu上运行的task的调度实体。
> 2. update_curr()函数是比较重要的函数，在很多地方调用，主要是更新当前正在运行的调度实体的运行时间信息。
> 3. 初始化当前创建的新进程的虚拟时间。
> 4. place_entity()函数在进程创建以及唤醒的时候都会调用，创建进程的时候传递参数initial=1。主要目的是更新调度实体得到虚拟时间（se->vruntime成员）。要和cfs_rq->min_vruntime的值保持差别不大，如果非常小的话，岂不是要上天（疯狂占用cpu运行）。
> 5. 这里为什么要减去cfs_rq->min_vruntime呢？因为现在计算进程的vruntime是基于当前cpu上的cfs_rq，并且现在还没有加入当前cfs_rq的就绪队列上。等到当前进程创建完毕开始唤醒的时候，加入的就绪队列就不一定是现在计算基于的cpu。所以，在加入就绪队列的函数中会根据情况加上当前就绪队列cfs_rq->min_vruntime。为什么要“先减后加”处理呢？假设cpu0上的cfs就绪队列的最小虚拟时间min_vruntime的值是1000000，此时创建进程的时候赋予当前进程虚拟时间是1000500。但是，唤醒此进程加入的就绪队列却是cpu1上CFS就绪队列，cpu1上的cfs就绪队列的最小虚拟时间min_vruntime的值如果是9000000。如果不采用“先减后加”的方法，那么该进程在cpu1上运行肯定是“乐坏”了，疯狂的运行。现在的处理计算得到的调度实体的虚拟时间是1000500 - 1000000 + 9000000 = 9000500，因此事情就不是那么的糟糕。

下面就对update_curr()一探究竟。

```c
 
static void update_curr(struct cfs_rq *cfs_rq){	struct sched_entity *curr = cfs_rq->curr;	u64 now = rq_clock_task(rq_of(cfs_rq));	u64 delta_exec; 	if (unlikely(!curr))		return; 	delta_exec = now - curr->exec_start;                    /* 1 */	if (unlikely((s64)delta_exec <= 0))		return; 	curr->exec_start = now;	curr->sum_exec_runtime += delta_exec;	curr->vruntime += calc_delta_fair(delta_exec, curr);    /* 2 */	update_min_vruntime(cfs_rq);                            /* 3 */}
```

> 1. delta_exec计算本次更新虚拟时间距离上次更新虚拟时间的差值。
> 2. 更新当前调度实体虚拟时间，calc_delta_fair()函数根据上面说的虚拟时间的计算公式计算虚拟时间（也就是调用__calc_delta()函数）。
> 3. 更新CFS就绪队列的最小虚拟时间min_vruntime。min_vruntime也是不断更新的，主要就是跟踪就绪队列中所有调度实体的最小虚拟时间。如果min_vruntime一直不更新的话，由于min_vruntime太小，导致后面创建的新进程根据这个值来初始化新进程的虚拟时间，岂不是新创建的进程有可能再一次疯狂了。这一次可能就是cpu0创建，在cpu0上面疯狂。

我们就看看update_min_vruntime()是怎么更新min_vruntime的。

```c
 
static void update_min_vruntime(struct cfs_rq *cfs_rq){	struct sched_entity *curr = cfs_rq->curr;	struct rb_node *leftmost = rb_first_cached(&cfs_rq->tasks_timeline);	u64 vruntime = cfs_rq->min_vruntime; 	if (curr) {		if (curr->on_rq)			vruntime = curr->vruntime;		else			curr = NULL;	} 	if (leftmost) { /* non-empty tree */		struct sched_entity *se;		se = rb_entry(leftmost, struct sched_entity, run_node); 		if (!curr)			vruntime = se->vruntime;		else			vruntime = min_vruntime(vruntime, se->vruntime);	} 	/* ensure we never gain time by being placed backwards. */	cfs_rq->min_vruntime = max_vruntime(cfs_rq->min_vruntime, vruntime);}
```

> 我们既然要更细就绪队列最小虚拟时间min_vruntime，试想一下，拥有最小虚拟时间的地方有哪些了？
> 
> - 就绪队列本身的cfs_rq->min_vruntime成员。
> - 当前正在运行的进程的最下虚拟时间，因为CFS调度器选择最适合运行的进程是选择维护的红黑树中虚拟时间最小的进程。
> - 如果在当前进程运行的过程中，有进程加入就绪队列，那么红黑树最左边的进程的虚拟时间同样也有可能是最小的虚拟时间。
> 
> 因此，update_min_vruntime()函数根据以上种种可能判断，并且保证就绪队列的最小虚拟时间min_vruntime单调递增的特性，更新最小虚拟时间。

我们继续place_entity()函数。

```c
 
static voidplace_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int initial){	u64 vruntime = cfs_rq->min_vruntime; 	/*	 * The 'current' period is already promised to the current tasks,	 * however the extra weight of the new task will slow them down a	 * little, place the new task so that it fits in the slot that	 * stays open at the end.	 */	if (initial && sched_feat(START_DEBIT))		vruntime += sched_vslice(cfs_rq, se);               /* 1 */ 	/* sleeps up to a single latency don't count. */	if (!initial) {		unsigned long thresh = sysctl_sched_latency; 		/*		 * Halve their sleep time's effect, to allow		 * for a gentler effect of sleepers:		 */		if (sched_feat(GENTLE_FAIR_SLEEPERS))			thresh >>= 1; 		vruntime -= thresh;                                 /* 2 */	} 	/* ensure we never gain time by being placed backwards. */	se->vruntime = max_vruntime(se->vruntime, vruntime);    /* 3 */}
```

> 1. 如果是创建进程调用该函数的话，参数initial参数是1。因此这里是处理创建的进程，针对刚创建的进程会进行一定的惩罚，将虚拟时间加上一个值就是惩罚，毕竟虚拟时间越小越容易被调度执行。惩罚的时间由sched_vslice()计算。
> 2. 这里主要是针对唤醒的进程，针对睡眠很久的的进程，我们总是期望它很快得到调度执行，毕竟人家睡了那么久。所以这里减去一定的虚拟时间作为补偿。
> 3. 我们保证调度实体的虚拟时间不能倒退。为何呢？可以想一下，如果一个进程刚睡眠1ms，然后醒来后你却要奖励3ms（虚拟时间减去3ms），然后他竟然赚了2ms。作为调度器，我们不做亏本生意。你睡眠100ms，奖励你3ms，那就是没问题的。

### 新创建的进程惩罚的时间是多少

有上面可知，惩罚的时间计算函数是sched_vslice()函数。

```c
 
static u64 sched_vslice(struct cfs_rq *cfs_rq, struct sched_entity *se){	return calc_delta_fair(sched_slice(cfs_rq, se), se);}
```

calc_delta_fair()函数上面已经分析过，计算实际运行时间delta对应的虚拟时间。这里的delta是sched_slice()函数计算。

```c
 
static u64 sched_slice(struct cfs_rq *cfs_rq, struct sched_entity *se){	u64 slice = __sched_period(cfs_rq->nr_running + !se->on_rq);    /* 1 */ 	for_each_sched_entity(se) {                                     /* 2 */		struct load_weight *load;		struct load_weight lw; 		cfs_rq = cfs_rq_of(se);		load = &cfs_rq->load;                                       /* 3 */ 		if (unlikely(!se->on_rq)) {			lw = cfs_rq->load; 			update_load_add(&lw, se->load.weight);			load = &lw;		}		slice = __calc_delta(slice, se->load.weight, load);         /* 4 */	}	return slice;}
```

> 1. __sched_period()前面已经提到了，根据就绪队列调度实体个数计算调度周期。
> 2. 针对没有使能组调度的情况下，for_each_sched_entity(se)就是for (; se; se = NULL)，循环一次。
> 3. 得到就绪队列的权重，也就是就绪队列上所有调度实体权重之和。
> 4. __calc_delta()函数有两个功能，除了上面说的可以计算进程运行时间转换成虚拟时间以外，还有第二个功能：计算调度实体se的权重占整个就绪队列权重的比例，然后乘以调度周期时间即可得到当前调度实体应该运行的时间（参数weught传递调度实体se权重，参数lw传递就绪队列权重cfs_rq->load）。例如，就绪队列权重是3072，当前调度实体se权重是1024，调度周期是6ms，那么调度实体应该得到的时间是6*1024/3072=2ms。

## 新进程加入就绪队列

经过do_fork()的大部分初始化工作完成之后，我们就可以唤醒新进程准别运行。也就是将新进程加入就绪队列准备调度。唤醒新进程的流程如下图。

1. do_fork()--->_do_fork()--->wake_up_new_task()--->activate_task()--->enqueue_task()--->enqueue_task_fair()
2.                                    |
3.                                    +------------>check_preempt_curr()--->check_preempt_wakeup() 

wake_up_new_task()负责唤醒新创建的进程。简化一下函数如下。

```c
 
void wake_up_new_task(struct task_struct *p){	struct rq_flags rf;	struct rq *rq; 	p->state = TASK_RUNNING;#ifdef CONFIG_SMP	p->recent_used_cpu = task_cpu(p);	__set_task_cpu(p, select_task_rq(p, task_cpu(p), SD_BALANCE_FORK, 0));   /* 1 */#endif	rq = __task_rq_lock(p, &rf);	activate_task(rq, p, ENQUEUE_NOCLOCK);                                   /* 2 */	p->on_rq = TASK_ON_RQ_QUEUED;	check_preempt_curr(rq, p, WF_FORK);                                      /* 3 */}
```

> 1. 通过调用select_task_rq()函数重新选择cpu，通过调用调度类中select_task_rq方法选择调度类中最空闲的cpu。
> 2. 将进程加入就绪队列，通过调用调度类中enqueue_task方法。
> 3. 既然新进程已经准备就绪，那么此时需要检查新进程是否满足抢占当前正在运行进程的条件，如果满足抢占条件需要设置TIF_NEED_RESCHED标志位。

CFS调度类对应的enqueue_task方法函数是enqueue_task_fair()，我们将部分和组调度相关的代码删除，简洁的代码看起来才赏心悦目。

```c
 
static voidenqueue_task_fair(struct rq *rq, struct task_struct *p, int flags){	struct cfs_rq *cfs_rq;	struct sched_entity *se = &p->se; 	for_each_sched_entity(se) {                       /* 1 */		if (se->on_rq)                                /* 2 */			break;		cfs_rq = cfs_rq_of(se);		enqueue_entity(cfs_rq, se, flags);            /* 3 */	} 	if (!se)		add_nr_running(rq, 1); 	hrtick_update(rq);}
```

> 1. 组调度关闭的时候，这里就是循环一次，不用纠结。
> 2. on_rq成员代表调度实体是否已经在就绪队列中。值为1代表在就绪队列中，当然就不需要继续添加就绪队列了。
> 3. enqueue_entity，从名字就可以看得出来是将调度实体加入就绪队列，我们称之为入队（enqueue）。

enqueue_entity()代码如下，删除了一些暂时不需要关注的部分代码。

```c
 
static void enqueue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags){	bool renorm = !(flags & ENQUEUE_WAKEUP) || (flags & ENQUEUE_MIGRATED);	bool curr = cfs_rq->curr == se; 	/*	 * If we're the current task, we must renormalise before calling	 * update_curr().	 */	if (renorm && curr)		se->vruntime += cfs_rq->min_vruntime; 	update_curr(cfs_rq);                        /* 1 */ 	if (renorm && !curr)		se->vruntime += cfs_rq->min_vruntime;   /* 2 */ 	account_entity_enqueue(cfs_rq, se);         /* 3 */ 	if (flags & ENQUEUE_WAKEUP)		place_entity(cfs_rq, se, 0);            /* 4 */ 	if (!curr)		__enqueue_entity(cfs_rq, se);           /* 5 */	se->on_rq = 1;                              /* 6 */}
```

> 1. update_curr()顺便更新当前运行调度实体的虚拟时间信息。
> 2. 还记得之前在task_fork_fair()函数最后减去的min_vruntime吗？现在是时候加回来了。
> 3. 更新就绪队列相关信息，例如就绪队列的权。
> 4. 针对唤醒的进程（flag有ENQUEUE_WAKEUP标识），我们是需要根据情况给予一定的补偿。之前也说了place_entity()函数的两种情况下的作用。当然这里针对新进程第一次加入就绪队列是不需要调用的。
> 5. __enqueue_entity()才是将se加入就绪队列维护的红黑树中，所有的se以vruntime为key。
> 6. 所有的操作完毕也意味着se已经加入就绪队列，置位on_rq成员。

account_entity_enqueue()函数到底更新了就绪队列哪些信息呢？

```c
 
static void account_entity_enqueue(struct cfs_rq *cfs_rq, struct sched_entity *se){	update_load_add(&cfs_rq->load, se->load.weight);  /* 1 */	if (!parent_entity(se))		update_load_add(&rq_of(cfs_rq)->load, se->load.weight);    /* 2 */#ifdef CONFIG_SMP	if (entity_is_task(se)) {		struct rq *rq = rq_of(cfs_rq); 		account_numa_enqueue(rq, task_of(se));		list_add(&se->group_node, &rq->cfs_tasks);                 /* 3 */	}#endif	cfs_rq->nr_running++;                                          /* 4 */}
```

> 1. 更新就绪队列权重，就是将se权重加在就绪队列权重上面。
> 2. cpu就绪队列`struct rq`同样也需要更新权重信息。
> 3. 将调度实体se加入链表。
> 4. nr_running成员是就绪队列中所有调度实体的个数。

### vruntime溢出怎么办

虽然调度实体se的vruntime成员是u64类型，可以保存非常大的数。但是当达到264ns后就溢出了。那么溢出会有问题吗？我们先看看__enqueue_entity()函数加入就绪队列的代码。

```c
 
static void __enqueue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se){	struct rb_node **link = &cfs_rq->tasks_timeline.rb_root.rb_node;	struct rb_node *parent = NULL;	struct sched_entity *entry;	bool leftmost = true; 	/*	 * Find the right place in the rbtree:	 */	while (*link) {		parent = *link;		entry = rb_entry(parent, struct sched_entity, run_node);		/*		 * We dont care about collisions. Nodes with		 * the same key stay together.		 */		if (entity_before(se, entry)) {			link = &parent->rb_left;		} else {			link = &parent->rb_right;			leftmost = false;		}	} 	rb_link_node(&se->run_node, parent, link);	rb_insert_color_cached(&se->run_node, &cfs_rq->tasks_timeline, leftmost);}
```

我们通过便利红黑树查找符合插入节点的位置。利用entity_before()函数比较两个调度实体se的vruntime值大小，以确定搜索方向。

1. static inline int entity_before(struct sched_entity *a, struct sched_entity *b)
2. {
3. 	return (s64)(a->vruntime - b->vruntime) < 0;
4. } 

假设要插入a的vruntime是101，b的vruntime是100，那么entity_before()函数返回0。现在假设a的vruntime溢出了，vruntime是5（我们期望是264 + 5，但是很遗憾溢出结果是5），b的vruntime即将溢出，vruntime的值是264 - 2。那么调度实体a的vruntime无论是5还是264 + 5，entity_before()函数都会返回0。因此计算结果保持了一致性，所以溢出是没有任何问题的。要看懂这里的代码，需要对负数在计算机中表示形式有所了解。

同样样的C语言技巧还应用在就绪队列min_vruntime成员，试想min_vruntime同样式u64类型也是存在溢出的时候。min_vruntime的溢出是否会有问题呢？其实也不会，我们继续看一下update_min_vruntime函数最后一条代码，`cfs_rq->min_vruntime = max_vruntime(cfs_rq->min_vruntime, vruntime);`max_vruntime()函数也利用了类似entity_before()函数的技巧。所以min_vruntime溢出也不会有问题。max_vruntime()依然可以返回正确的结果。

```c
 
static inline u64 max_vruntime(u64 max_vruntime, u64 vruntime){	s64 delta = (s64)(vruntime - max_vruntime);	if (delta > 0)		max_vruntime = vruntime; 	return max_vruntime;}
```

### 抢占当前进程条件

当唤醒一个新进程的时候，此时也是一个检测抢占的机会。因为唤醒的进程有可能具有更高的优先级或者更小的虚拟时间。紧接上节唤醒新进程后调用check_preempt_curr()函数检查是否满足抢占条件。

```c
 
void check_preempt_curr(struct rq *rq, struct task_struct *p, int flags){	const struct sched_class *class; 	if (p->sched_class == rq->curr->sched_class) {		rq->curr->sched_class->check_preempt_curr(rq, p, flags);   /* 1 */	} else {		for_each_class(class) {                                    /* 2 */			if (class == rq->curr->sched_class)				break;			if (class == p->sched_class) {				resched_curr(rq);				break;			}		}	}}
```

> 1. 唤醒的进程和当前的进程同属于一个调度类，直接调用调度类的check_preempt_curr方法检查抢占条件。毕竟调度器自己管理的进程，自己最清楚是否适合抢占当前进程。
> 2. 如果唤醒的进程和当前进程不属于一个调度类，就需要比较调度类的优先级。例如，当期进程是CFS调度类，唤醒的进程是RT调度类，自然实时进程是需要抢占当前进程的，因为优先级更高。

现在考虑唤醒的进程和当前的进程同属于一个CFS调度类的情况。自然调用的就是check_preempt_wakeup()函数。

```c
 
static void check_preempt_wakeup(struct rq *rq, struct task_struct *p, int wake_flags){	struct sched_entity *se = &curr->se, *pse = &p->se;	struct cfs_rq *cfs_rq = task_cfs_rq(curr); 	if (wakeup_preempt_entity(se, pse) == 1)    /* 1 */		goto preempt; 	return;preempt:	resched_curr(rq);                           /* 2 */}
```

> 1. 检查唤醒的进程是否满足抢占当前进程的条件。
> 2. 如果可以抢占当前进程，设置TIF_NEED_RESCHED flag。

wakeup_preempt_entity()函数如下。

```c
 
/* * Should 'se' preempt 'curr'.     */static int wakeup_preempt_entity(struct sched_entity *curr, struct sched_entity *se){	s64 gran, vdiff = curr->vruntime - se->vruntime; 	if (vdiff <= 0)                    /* 1 */		return -1; 	gran = wakeup_gran(se);	if (vdiff > gran)                  /* 2 */		return 1; 	return 0;}
```

wakeup_preempt_entity()函数可以返回3种结果。se1、se2、se3及curr调度实体的虚拟时间如下图所示。如果curr虚拟时间比se小，返回-1；如果curr虚拟时间比se大，并且两者差值小于gran，返回0；否则返回1。默认情况下，wakeup_gran()函数返回的值是1ms根据调度实体se的权重计算的虚拟时间。因此，满足抢占的条件就是，唤醒的进程的虚拟时间首先要比正在运行进程的虚拟时间小，并且差值还要大于一定的值才行（这个值是sysctl_sched_wakeup_granularity，称作唤醒抢占粒度）。这样做的目的是避免抢占过于频繁，导致大量上下文切换影响系统性能。

```txt
 
 se3             se2    curr         se1------|---------------|------|-----------|--------> vruntime          |<------gran------>|                               wakeup_preempt_entity(curr, se1) = -1     wakeup_preempt_entity(curr, se2) =  0     wakeup_preempt_entity(curr, se3) =  1
```

## 周期性调度

周期性调度是指Linux定时周期性地检查当前任务是否耗尽当前进程的时间片，并检查是否应该抢占当前进程。一般会在定时器的中断函数中，通过一层层函数调用最终到scheduler_tick()函数。

```c
 
void scheduler_tick(void){	int cpu = smp_processor_id();	struct rq *rq = cpu_rq(cpu);	struct task_struct *curr = rq->curr;	struct rq_flags rf; 	sched_clock_tick();	rq_lock(rq, &rf);	update_rq_clock(rq);	curr->sched_class->task_tick(rq, curr, 0);        /* 1 */	cpu_load_update_active(rq);	calc_global_load_tick(rq);	rq_unlock(rq, &rf);	perf_event_task_tick();#ifdef CONFIG_SMP	rq->idle_balance = idle_cpu(cpu);	trigger_load_balance(rq);                         /* 2 */#endif}
```

> 1. 调用调度类对应的task_tick方法，针对CFS调度类该函数是task_tick_fair。
> 2. 触发负载均衡，以后有时间可以详谈。

task_tick_fair()函数如下。

```c
 
static void task_tick_fair(struct rq *rq, struct task_struct *curr, int queued){	struct cfs_rq *cfs_rq;	struct sched_entity *se = &curr->se; 	for_each_sched_entity(se) {		cfs_rq = cfs_rq_of(se);		entity_tick(cfs_rq, se, queued);	}}
```

> for循环是针对组调度，组调度未打开的情况下，这里就是一层循环。

entity_tick()是主要干活的。

```c
 
static void entity_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr, int queued){	/*	 * Update run-time statistics of the 'current'.	 */	update_curr(cfs_rq);                      /* 1 */ 	if (cfs_rq->nr_running > 1)		check_preempt_tick(cfs_rq, curr);     /* 2 */}
```

> 1. 调用update_curr()更新当前运行的调度实体的虚拟时间等信息。
> 2. 如果就绪队列就绪态的调度实体个数大于1需要检查是否满足抢占条件，如果可以抢占就设置TIF_NEED_RESCHED flag。

check_preempt_tick()函数如下。

```c
 
static voidcheck_preempt_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr){	unsigned long ideal_runtime, delta_exec;	struct sched_entity *se;	s64 delta; 	ideal_runtime = sched_slice(cfs_rq, curr);    /* 1 */	delta_exec = curr->sum_exec_runtime - curr->prev_sum_exec_runtime;    /* 2 */	if (delta_exec > ideal_runtime) {		resched_curr(rq_of(cfs_rq));              /* 3 */		clear_buddies(cfs_rq, curr);		return;	} 	if (delta_exec < sysctl_sched_min_granularity)    /* 4 */		return; 	se = __pick_first_entity(cfs_rq);             /* 5 */	delta = curr->vruntime - se->vruntime; 	if (delta < 0)                                /* 6 */		return; 	if (delta > ideal_runtime)                    /* 7 */		resched_curr(rq_of(cfs_rq));}
```

> 1. sched_slice()函数上面已经分析过，计算curr进程在本次调度周期中应该分配的时间片。时间片用完就应该被抢占。
> 2. delta_exec是当前进程已经运行的实际时间。
> 3. 如果实际运行时间已经超过分配给进程的时间片，自然就需要抢占当前进程。设置TIF_NEED_RESCHED flag。
> 4. 为了防止频繁过度抢占，我们应该保证每个进程运行时间不应该小于最小粒度时间sysctl_sched_min_granularity。因此如果运行时间小于最小粒度时间，不应该抢占。
> 5. 从红黑树中找到虚拟时间最小的调度实体。
> 6. 如果当前进程的虚拟时间仍然比红黑树中最左边调度实体虚拟时间小，也不应该发生调度。
> 7. 这里把虚拟时间和实际时间比较，看起来很奇怪。感觉就像是bug一样，然后经过查看提交记录，作者的意图是：希望权重小的任务更容易被抢占。

针对以上每一次周期调度（scheduling tick ）流程可以总结如下。

1. 更新当前正在运行进程的虚拟时间。
    
2. 检查当前进程是否满足被抢占的条件。
    
    - if (delta_exec > ideal_runtime)，然后置位TIF_NEED_RESCHED。
3. 检查TIF_NEED_RESCHED flag。
    
    - 如果置位，从就绪队列中挑选最小虚拟时间的进程运行。
    - 将当前被强占的进程重新加入就绪队列红黑树上（enqueue task）。
    - 从就绪队列红黑树上删除即将运行进程的节点（dequeue task）。

## 如何选择下一个合适进程运行

当进程被设置TIF_NEED_RESCHED flag后会在某一时刻触发系统发生调度或者进程调用schedule()函数主动放弃cpu使用权，触发系统调度。我们就以schedule()函数为例分析。

```c
 
asmlinkage __visible void __sched schedule(void){	struct task_struct *tsk = current; 	sched_submit_work(tsk);	do {		preempt_disable();		__schedule(false);		sched_preempt_enable_no_resched();	} while (need_resched());}
```

主要干活的还是__schedule()函数。

```c
 
static void __sched notrace __schedule(bool preempt){	struct task_struct *prev, *next;	struct rq_flags rf;	struct rq *rq;	int cpu; 	cpu = smp_processor_id();	rq = cpu_rq(cpu);	prev = rq->curr; 	if (!preempt && prev->state) {		if (unlikely(signal_pending_state(prev->state, prev))) {			prev->state = TASK_RUNNING;		} else {			deactivate_task(rq, prev, DEQUEUE_SLEEP | DEQUEUE_NOCLOCK);    /* 1 */			prev->on_rq = 0;		}	} 	next = pick_next_task(rq, prev, &rf);    /* 2 */	clear_tsk_need_resched(prev);            /* 3 */ 	if (likely(prev != next)) {		rq->curr = next;		rq = context_switch(rq, prev, next, &rf);    /* 4 */	} 	balance_callback(rq);}
```

> 1. 针对主动放弃cpu进入睡眠的进程，我们需要从对应的就绪队列上删除该进程。
> 2. 选择下个合适的进程开始运行，该函数前面已经分析过。
> 3. 清除TIF_NEED_RESCHED flag。
> 4. 上下文切换，从prev进程切换到next进程。

CFS调度类pick_next_task方法是pick_next_task_fair()函数。

```c
 
static struct task_struct *pick_next_task_fair(struct rq *rq, struct task_struct *prev, struct rq_flags *rf){	struct cfs_rq *cfs_rq = &rq->cfs;	struct sched_entity *se;	struct task_struct *p;	int new_tasks; again:	if (!cfs_rq->nr_running)		goto idle; 	put_prev_task(rq, prev);                        /* 1 */	do {		se = pick_next_entity(cfs_rq, NULL);        /* 2 */		set_next_entity(cfs_rq, se);                /* 3 */		cfs_rq = group_cfs_rq(se);	} while (cfs_rq);                               /* 4 */ 	p = task_of(se);#ifdef CONFIG_SMP	list_move(&p->se.group_node, &rq->cfs_tasks);#endif 	if (hrtick_enabled(rq))		hrtick_start_fair(rq, p); 	return p;idle:	new_tasks = idle_balance(rq, rf); 	if (new_tasks < 0)		return RETRY_TASK; 	if (new_tasks > 0)		goto again; 	return NULL;}
```

> 1. 主要是处理prev进程的后事，当进程让出cpu时就会调用该函数。
> 2. 选择最适合运行的调度实体。
> 3. 选择出来的调度实体se还需要继续加工一下才能投入运行，加工的活就是由set_next_entity()函数负责。
> 4. 针对没有使能组调度的情况下，循环一次就结束了。

put_prev_task()究竟处理了哪些后事呢？CFS调度类put_prev_task方法的函数是put_prev_task_fair()。

```c
 
```

> 1. 针对组调度情况，暂不考虑。
> 2. put_prev_entity()是主要干活的部分。

put_prev_entity()函数如下。

```c
 
```

> 1. 如果prev进程依然在就绪队列上，极有可能是prev进程被强占的情况。在让出cpu之前需要更新进程虚拟时间等信息。如果prev进程不在就绪队列上，这里可以直接跳过更新。因为，prev进程在deactivate_task()中已经调用了update_curr()，所以这里就可以省略了。
> 2. 如果prev进程依然在就绪队列上，我们需要重新将prev进程插入红黑树等待调度。
> 3. update_load_avg()是更新prev进程的负载信息，这些信息在负载均衡的时候会用到。
> 4. 后事已经处理完毕，就绪队列的curr指针也应该指向NULL，代表当前就绪队列上没有正在运行的进程。

prev进程的后事已经处理完毕，接下来继承大统的进程需要借助set_next_entity()函数昭告天下。

```c
 
```

> 1. __dequeue_entity()是将调度实体从红黑树中删除，针对即将运行的进程，我们都会从红黑树中删除当前进程。当进程被强占后，调用put_prev_entity()函数会重新插入红黑树。因此这个地方和put_prev_entity()函数中加入红黑树是个呼应。
> 2. 更新进程的负载信息。负载均衡会使用。
> 3. 更新就绪队列curr成员，昭告天下，“现在我是当前正在运行的进程”。
> 4. update_stats_curr_start()函数就一句话，更新调度实体exec_start成员，为update_curr()函数统计时间做准备。
> 5. check_preempt_tick()函数用到，统计当前进程已经运行的时间，以此判断是否能够被其他进程抢占。

## 进程的睡眠

在__schedule()函数中，如果prev进程主动睡眠。那么会调用deactivate_task()函数。deactivate_task()函数最终会调用调度类dequeue_task方法。CFS调度类对应的函数是dequeue_task_fair()，该函数是enqueue_task_fair()函数反操作。

```c
 
```

> 1. 针对组调度操作，没有使能组调度情况下，循环仅一次。
> 2. 将调度实体se从对应的就绪队列cfs_rq上删除。

dequeue_entity()函数如下。

```c
 
```

> 1. 借机更新当前正在运行进程的虚拟时间信息，如果当前dequeue的进程就是当前正在运行的进程的话，那么此次update_curr()就很有必要了。
> 2. 针对当前正在运行的进程来说，其对应的调度实体已经不在红黑树上了，因此不用在调用__dequeue_entity()函数从红黑树上参数对用的节点。
> 3. 调度实体已经从就绪队列的红黑树上删除，因此更新on_rq成员。
> 4. 更新就绪队列相关信息，例如权重信息。稍后介绍。
> 5. 如果进程不是睡眠（例如从一个CPU迁移到另一个CPU），进程最小虚拟时间需要减去当前就绪队列对应的最小虚拟时间，原因之前也说了。迁移之后会在enqueue的时候加上对应的CFS就绪队列最小拟时间。

account_entity_dequeue()和前面说的account_entity_enqueue()操作相反。account_entity_dequeue()函数如下。

```c
 
```

> 1. 从就绪队列权重总和中减去当前dequeue调度实体的权重。
> 2. 从链表中删除调度实体se。
> 3. 就绪队列中可运行调度实体计数减1。

标签: [CFS](http://www.wowotech.net/tag/CFS)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [CFS调度器（3）-组调度](http://www.wowotech.net/process_management/449.html) | [CFS调度器（1）-基本原理](http://www.wowotech.net/process_management/447.html)»

**评论：**

**abcde**  
2024-05-05 21:47

我想“Linux菜鸟”的本意应该是 “当某个se的slice > sysctl_sched_min_granularity  
&& < tick”时，如果此se用完了自己的slice，但，中断时间却又尚未到达，那么，就没有机制让此se停下来，所以，se实际执行时间，不就超出自己应得的slice了吗？  
  
这个问题的答案，我觉得应该是：  
当切换到se执行时，同时设置了定时器，定时器会保证se的slice用完之后立即触发下一次调度。即：  
http://github.com/torvalds/linux/blob/v4.18/kernel/sched/fair.c#L7011  
  
    if (hrtick_enabled(rq))  
        hrtick_start_fair(rq, p);

[回复](http://www.wowotech.net/process_management/448.html#comment-8897)

**honor**  
2022-01-04 17:49

static void update_curr(struct cfs_rq *cfs_rq)函数中：看描述                
delta_exec计算本次更新虚拟时间距离上次更新虚拟时间的差值。    
  
这个是wall time吧？

[回复](http://www.wowotech.net/process_management/448.html#comment-8471)

**modernoom**  
2021-12-07 16:58

我想问一下为什么主动休眠的进程不需要减去最小虚拟时间？它在被唤醒的时候不是有可能被调度到其他CPU吗

[回复](http://www.wowotech.net/process_management/448.html#comment-8396)

**[linuxer](http://www.wowotech.net/)**  
2021-12-08 06:55

@modernoom：因为主动休眠的进程减去最小虚拟时间没有意义，下次被唤醒的时候，会被挂入某个cpu runqueue，而对于睡眠一段时间的线程而言，为了补偿它，其vruntime的基准就是其挂入的cfs rq的min vruntime减去一个补偿值，不需要过去的vruntime信息。

[回复](http://www.wowotech.net/process_management/448.html#comment-8397)

**modernoom**  
2021-12-09 21:22

@linuxer：上面的源码中，唤醒的进程并不是直接设置成min vruntime减去一个补偿值。而是和该调度实体的虚拟运行时间两个取最大值。那么如果进程休眠后被唤醒调度到一个虚拟时间比较小的CPU上，结果进程的虚拟时间比该CPU的min vruntime减去一个补偿值的值大，那么会取较大者，不是会导致唤醒进程无法及时得到调度吗？

[回复](http://www.wowotech.net/process_management/448.html#comment-8401)

**modernoom**  
2021-12-09 21:26

@linuxer：或者说，这其实也是CFS的一种公平的体现。就算你从休眠中唤醒，但你本身运行了很长时间了，所以就算不会及时调度也没关系

[回复](http://www.wowotech.net/process_management/448.html#comment-8402)

**[linuxer](http://www.wowotech.net/)**  
2021-12-14 07:18

@modernoom：同意你的观点，但是我觉得这里处理的还是有问题的（不同cfs上的min vruntime是不同步的），如果我来写，我可能考虑这么做：  
1、sleep之前，记录其vruntime和cfs minvruntime之间的delta  
2、记录sleep的时间长度，如果长时间的阻塞，那么其vruntime的就是其挂入的cfs rq的min vruntime减去一个补偿值  
3、如果短时间的阻塞，那么其vruntime应该是基于当前cfs minvruntime惩罚（delta-sleep time）。

[回复](http://www.wowotech.net/process_management/448.html#comment-8420)

**jarvis**  
2022-06-16 11:23

@linuxer：我看wakeup task的代码，如果任务原先的cpu和当前选择的select cpu不同的话，会走到migrate_task_rq_fair，其会做se->vruntime -= min_vruntime，即会减去任务原先所在cpu的min_vruntime；这就相当于是在唤醒的时候减去min_vruntime，而不是在deactivate_task的时候减去min_vruntime；  
但这应该也算是按照你们所说的vruntime delta的想法去做的，还是我哪里理解的不对？

[回复](http://www.wowotech.net/process_management/448.html#comment-8630)

**esc**  
2021-10-21 19:21

check_preempt_tick()里面第7点解释delta > ideal_runtime我觉得写的不是很深入，没提为什么这里要有这么一个判断。  
  
根据提交的历史记录，4，5，6，7都在同一次代码提交中(commit f685ceaca)，且这段代码在4的上面有一段注释：  
  
    /*  
     * Ensure that a task that missed wakeup preemption by a  
     * narrow margin doesn't have to wait for a full slice.  
     * This also mitigates buddy induced latencies under load.  
     */  
  
看描述，目的是为了降低从睡眠中被唤醒进程的延迟。睡眠进程被唤醒的时候或许并不满足抢占当前进程的条件，但后续如果让当前进程跑完一整个slice会导致延迟高，因此加了第7点的判断。  
  
至于为什么是delta和ideal_time的比较，我的想法如下：  
  
假设在最普通场景下，nice都为0，没有睡眠和唤醒的情况，有两个进程，分别是A和B，A的vruntime较低，V_a < V_b，调度器先选择了A执行，那么check_preempt_tick()的时候，delta一定小于ideal_time，否则，因为delta_exec > delta，函数在前面就已经返回了。  
  
nice都为0时，如果存在被唤醒的进程，那么被唤醒进程的vruntime会很小，如果这个vruntime比当前执行进程开始执行的时候对应的vruntime都要小，那么满足delta > ideal_time，代表的是相对于被唤醒的进程，当前进程已经运行了ideal_time，所以要发生抢占。至于用slice而不是vslice，就是为了让优先级低的更容易被抢占。

[回复](http://www.wowotech.net/process_management/448.html#comment-8339)

**mmz**  
2021-12-31 11:41

@esc：可以参考下这个链接的讨论：  
http://lkml.iu.edu/hypermail/linux/kernel/1009.1/02198.html

[回复](http://www.wowotech.net/process_management/448.html#comment-8454)

**mmz**  
2021-12-31 16:40

@mmz：还有这个  
http://lkml.iu.edu/hypermail/linux/kernel/1104.0/02256.html

[回复](http://www.wowotech.net/process_management/448.html#comment-8455)

**bang**  
2021-04-23 23:28

提一个小的严谨性问题哈，“新创建的进程惩罚的时间是多少”小节中的“例如，就绪队列权重是3072，当前调度实体se权重是1024，调度周期是6ms，那么调度实体应该得到的时间是6*1024/3072=2ms。”  
->对于新创建的进程se->on_rq=0，则vruntime的理论值应该为6*1024/3072*((2^32-1)/(2^32))=1999999ns

[回复](http://www.wowotech.net/process_management/448.html#comment-8223)

**新手**  
2020-08-13 14:17

请教一下，在update_min_vruntime中    if (curr) {  
        if (curr->on_rq)  
            vruntime = curr->vruntime;  
        else  
            curr = NULL;  
curr代表当前正在运行的调度实体，已经在运行，就肯定从rb tree上摘下来了，那on_rq就应该一定是0，为什么还要判断它是不是1呢

[回复](http://www.wowotech.net/process_management/448.html#comment-8095)

**wangxingxing**  
2020-07-22 17:32

您好，关于周期性调度，如果cfs判断需要立马抢占，只是调用resched_curr去设置抢占标志TIF_NEED_RESCHED，请问真正的进程切换发生在什么时候呢？

[回复](http://www.wowotech.net/process_management/448.html#comment-8061)

**[smcdef](http://www.wowotech.net/)**  
2020-07-23 13:32

@wangxingxing：看看内核__schedule函数上面的注释

[回复](http://www.wowotech.net/process_management/448.html#comment-8063)

**wangxingxing**  
2020-07-23 14:53

@smcdef：系统时钟中断返回的路径上会去检查TIF_NEED_RESCHED，如果有设置这个flag就会立马调用shedule进行抢占吗？如果是能否用源码show一下ARM的具体流程。另外如果是这样的流程，那么如何保证中断结束后调用schedule的时候就能把当前运行的进程切换掉？（是否有可能这个时候的schedule选出的新进程还是当前运行的进程？还是说scheduler_tick判断要切的进程在中断返回调用shedule的时候一定不会被选为下一个新进程？）

[回复](http://www.wowotech.net/process_management/448.html#comment-8064)

**不懂就问**  
2019-11-27 21:21

大神，请教一个问题，  
task_fork_fair函数里 cfs_rq->curr为什么会是NULL呢，当前current进程正在fork，其所在cfs_rq的curr变量应该就是current的entity实体啊？ if判断是不是多余的呢？

[回复](http://www.wowotech.net/process_management/448.html#comment-7766)

**湫**  
2019-10-26 21:56

一个进程连续fork了四个子进程，按照文中的理论，这四个子进程的虚拟运行时间应该是单调递增的。也就是说第一个子进程应该最先执行。可实验结果却是第一个子进程最后执行？有人知道这是为什么吗？（内核版本4.4）

[回复](http://www.wowotech.net/process_management/448.html#comment-7714)

**karas**  
2019-10-05 14:56

楼主，你好，在put_prev_entity()函数中，prev->on_rq不为0时，会重新将prev入队。那么如果prev->on_rq为0的情况下，会将prev放置到哪？我自己的想法是放置到等待队列，等待调度，但是看了pick_next_task_fair的上下文也没有找到类似的操作，应该不至于把prev给丢弃了。请楼主抽空帮忙解惑下，“prev在放弃cpu控制权，调用put_prev_entity后会被放置到什么地方？之后又怎么被重新调度？”谢谢

[回复](http://www.wowotech.net/process_management/448.html#comment-7683)

**zxq**  
2020-07-06 14:39

@karas：我理解这里作者是以主动让出cpu  schedule来讲解的进程切换，一般内核在等待某个事件发生时，会先将当前进程加入到等待队列，然后调用schedule主动让出cpu，等到想要的事件产生时会唤醒进程，这个时候会将进程重新放置到就绪队列等待调度。wake_up->__wake_up_common->default_wake_function->try_to_wake_up->ttwu_queue->ttwu_do_activate->activate_task->enqueue_task。但是对于进程因运行时间耗尽而引起的调度应该会直接将进程插入到就绪队列。

[回复](http://www.wowotech.net/process_management/448.html#comment-8047)

**fei ye**  
2021-11-11 15:20

@zxq：你的理解非常正确。

[回复](http://www.wowotech.net/process_management/448.html#comment-8369)

1 [2](http://www.wowotech.net/process_management/448.html/comment-page-2#comments)

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
    
    - [ARM64的启动过程之（二）：创建启动阶段的页表](http://www.wowotech.net/armv8a_arch/create_page_tables.html)
    - [Linux时间子系统之（二）：软件架构](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html)
    - [Linux PM QoS framework(1)_概述和软件架构](http://www.wowotech.net/pm_subsystem/pm_qos_overview.html)
    - [进程管理和终端驱动：基本概念](http://www.wowotech.net/process_management/process-tty-basic.html)
    - [Linux设备模型(4)_sysfs](http://www.wowotech.net/device_model/dm_sysfs.html)
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
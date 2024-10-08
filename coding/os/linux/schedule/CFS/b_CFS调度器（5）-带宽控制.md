作者：[smcdef](http://www.wowotech.net/author/531) 发布于：2018-12-22 15:07 分类：[进程管理](http://www.wowotech.net/sort/process_management)
# 前言

什么是带宽控制？简而言之就是控制一个用户组在给定周期时间内可以消耗CPU的时间，如果在给定的周期内消耗CPU时间超额，就限制该用户组内任务调度，直到下一个周期。限制某个进程的最大CPU使用率是否真的有必要呢？如果一个系统中仅存在一个进程，限制该进程使用CPU使用率最大50%，当进程使用率达到50%的时候，就限制该进程运行，CPU进入idle状态。看起来好像没有任何意义。但是，有时候，这正是系统管理员可能想要做的事情。如果这些进程属于仅支付了一定CPU时间的客户或者需要提供严格资源的情况，则限制进程（或进程组）可能消耗的CPU时间的最大份额是很有必要的。毕竟付多少钱享受多少服务。本文章仅讨论SCHED_NORMAL进程的CPU带宽控制（CPU bandwidth control）。

> 注：代码分析基于Linux 4.18.0。
# 设计原理

如果使用CPU bandwith control，需要配置CONFIG_FAIR_GROUP_SCHED和CONFIG_CFS_BANDWIDTH选项。该功能是限制一个组的最大使用CPU带宽。通过设置两个变量quota和period，period是指一段周期时间，quota是指在period周期时间内，一个组可以使用的CPU时间限额。当一个组的进程运行时间超过quota后，就会被限制运行，这个动作被称作throttle。直到下一个period周期开始，这个组会被重新调度，这个过程称作unthrottle。

在多核系统中，一个用户组使用`task_group`描述，用户组中包含CPU数量的调度实体，以及调度实体对应的group cfs_rq。如何限制一个用户组中的进程呢？我们可以简单的将用户组管理的调度实体从对应的就绪队列上删除即可，然后标记调度实体对应的group cfs_rq的标志位。quota和period的值存储在`cfs_bandwidth`结构体中，该结构体嵌在`tasak_group`中，`cfs_bandwidth`结构体还包含runtime成员记录剩余限额时间。每当用户组中的进程运行一段时间时，对应的runtime时间也在减少。系统会启动一个高精度定时器，周期时间是period，在定时器时间到达后重置剩余限额时间runtime为quota，开始下一个轮时间跟踪。所有的用户组进程运行的时间累加在一起，保证总的运行时间小于quota。每个用户组会管理CPU个数的就绪队列group cfs_rq。每个group cfs_rq中也有限额时间，该限额时间是从全局用户组quota中申请。例如，周期period值100ms，限额quota值50ms，2个CPU系统。CPU0上group cfs_rq首先从全局限额时间中申请5ms时间（此实runtime值为45），然后运行进程。当5ms时间消耗完时，继续从全局时间限额quota中申请5ms（此实runtime值为40）。CPU1上的情况也同样如此，先以就绪队列cfs_rq的身份从quota中申请一个时间片，然后供进程运行消耗。当全局quota剩余时间不足以满足CPU0或者CPU1申请时，就需要throttle对应的cfs_rq。在定时器时间到达后，unthrottle所有已经throttle的cfs_rq。

总结一下就是，`cfs_bandwidth`就像是一个全局时间池（时间池管理时间，类比内存池管理内存）。每个group cfs_rq如果想让其管理的红黑树上的调度实体调度，必须首先向全局时间池中申请固定的时间片，然后供其进程消耗。当时间片消耗完，继续从全局时间池中申请时间片。终有一刻，时间池中已经没有时间可供申请。此时就是throttle cfs_rq的大好时机。
## 数据结构

每个`task_group`都包含`cfs_bandwidth`结构体，主要记录和管理时间池的时间信息。
```cpp
1. struct cfs_bandwidth {
2. #ifdef CONFIG_CFS_BANDWIDTH
3. 	ktime_t				period;             /* 1 */
4. 	u64					quota;              /* 2 */
5. 	u64					runtime;            /* 3 */

7. 	struct hrtimer		period_timer;       /* 4 */
8. 	struct list_head	throttled_cfs_rq;   /* 5 */
9.     /* ... */
10. #endif
11. };
```

> 1. 设定的定时器周期时间。
> 2. 限额时间。
> 3. 剩余可运行时间，在每次定时器回调函数中更新值为quota。
> 4. 上面一直提到的高精度定时器。
> 5. 所有被throttle的cfs_rq挂入此链表，在定时器的回调函数中便利链表执行unthrottle cfs_rq操作。

CFS就绪队列使用`cfs_rq`结构体描述，和bandwidth相关成员如下：

```c
struct cfs_rq {#ifdef CONFIG_FAIR_GROUP_SCHED	struct rq *rq;	              /* 1 */	struct task_group *tg;        /* 2 */#ifdef CONFIG_CFS_BANDWIDTH	int runtime_enabled;          /* 3 */	u64 runtime_expires;	s64 runtime_remaining;        /* 4 */ 	u64 throttled_clock, throttled_clock_task;     /* 5 */	u64 throttled_clock_task_time;	int throttled, throttle_count;                 /* 6 */	struct list_head throttled_list;               /* 7 */#endif /* CONFIG_CFS_BANDWIDTH */#endif /* CONFIG_FAIR_GROUP_SCHED */};
```

> 1. cfs_rq依附的cpu runqueue，每个CPU有且仅有一个rq运行队列。
> 2. cfs_rq所属的task_group。
> 3. 该就绪队列是否已经开启带宽限制，默认带宽限制是关闭的，如果带宽限制使能，runtime_enabled的值为1。
> 4. cfs_rq从全局时间池申请的时间片剩余时间，当剩余时间小于等于0的时候，就需要重新申请时间片。
> 5. 当cfs_rq被throttle的时候，方便统计被throttle的时间，需要记录throttle开始的时间。
> 6. throttled：如果cfs_rq被throttle后，throttled变量置1，unthrottle的时候，throttled变量置0；throttle_count：由于task_group支持嵌套，当parent task_group的cfs_rq被throttle的时候，其chaild task_group对应的cfs_rq的throttle_count成员计数增加。
> 7. 被throttle的cfs_rq挂入`cfs_bandwidth->throttled_cfs_rq`链表。
## bandwidth贡献

周期性调度中会调用update_curr()函数更新当前正在运行进程的虚拟时间。该进程bandwidth贡献也在此时累计。从进程依附的cfs_rq的可用时间中减去进程运行的时间，如果时间不够，就从全局时间池中申请一定时间片。在update_curr()函数中调用account_cfs_rq_runtime()函数统计cfs_rq剩余可运行时间。
```cpp
1. static __always_inline
2. void account_cfs_rq_runtime(struct cfs_rq *cfs_rq, u64 delta_exec)
3. {
4. 	if (!cfs_bandwidth_used() || !cfs_rq->runtime_enabled)
5. 		return;

7. 	__account_cfs_rq_runtime(cfs_rq, delta_exec);
8. } 
```
  
如果使能CFS bandwidth control功能，cfs_bandwidth_used()返回1，`cfs_rq->runtime_enabled`值为1。__account_cfs_rq_runtime()函数如下：
```cpp
1. static void __account_cfs_rq_runtime(struct cfs_rq *cfs_rq, u64 delta_exec)
2. {
3. 	/* dock delta_exec before expiring quota (as it could span periods) */
4. 	cfs_rq->runtime_remaining -= delta_exec;           /* 1 */
5. 	expire_cfs_rq_runtime(cfs_rq);

7. 	if (likely(cfs_rq->runtime_remaining > 0))         /* 2 */
8. 		return;

10. 	/*
11. 	 * if we're unable to extend our runtime we resched so that the active
12. 	 * hierarchy can be throttled
13. 	 */
14. 	if (!assign_cfs_rq_runtime(cfs_rq) && likely(cfs_rq->curr))         /* 4 */
15. 		resched_curr(rq_of(cfs_rq));                   /* 5 */
16. } 
```

> 1. 进程已经运行delta_exec时间，因此cfs_rq剩余可运行时间减少。
> 2. 如果cfs_rq剩余运行时间还有，那么没必要向全局时间池申请时间片。
> 3. 如果cfs_rq可运行时间不足，assign_cfs_rq_runtime()负责从全局时间池中申请时间片。
> 4. 如果全局时间片时间不够，就需要throttle当前cfs_rq。当然这里是设置TIF_NEED_RESCHED flag。在后面throttle。

assign_cfs_rq_runtime()函数如下：

```c
static int assign_cfs_rq_runtime(struct cfs_rq *cfs_rq){	struct task_group *tg = cfs_rq->tg;	struct cfs_bandwidth *cfs_b = tg_cfs_bandwidth(tg);	u64 amount = 0, min_amount, expires;	int expires_seq; 	/* note: this is a positive sum as runtime_remaining <= 0 */	min_amount = sched_cfs_bandwidth_slice() - cfs_rq->runtime_remaining;    /* 1 */ 	raw_spin_lock(&cfs_b->lock);	if (cfs_b->quota == RUNTIME_INF)                          /* 2 */		amount = min_amount;	else {		start_cfs_bandwidth(cfs_b);                           /* 3 */ 		if (cfs_b->runtime > 0) {			amount = min(cfs_b->runtime, min_amount);			cfs_b->runtime -= amount;                         /* 4 */			cfs_b->idle = 0;		}	}	expires_seq = cfs_b->expires_seq;	expires = cfs_b->runtime_expires;	raw_spin_unlock(&cfs_b->lock); 	cfs_rq->runtime_remaining += amount;                      /* 5 */	/*	 * we may have advanced our local expiration to account for allowed	 * spread between our sched_clock and the one on which runtime was	 * issued.	 */	if (cfs_rq->expires_seq != expires_seq) {		cfs_rq->expires_seq = expires_seq;		cfs_rq->runtime_expires = expires;	} 	return cfs_rq->runtime_remaining > 0;                     /* 6 */}
```

> 1. 从全局时间池申请的时间片默认是5ms。
> 2. 如果该cfs_rq不限制带宽，那么quota的值为RUNTIME_INF，既然不限制带宽，自然时间池的时间是取之不尽用之不竭的，所以申请时间片一定成功。
> 3. 确保定时器是打开的，如果关闭状态，就打开定时器。该定时器会在定时时间到达后，重置全局时间池可用剩余时间。
> 4. 申请时间片成功，全局时间池剩余可用时间更新。
> 5. cfs_rq剩余可用时间增加。
> 6. 如果cfs_rq向全局时间池申请不到时间片，那么该函数返回0，否则返回1，代表申请时间片成功，不需要throttle。
## 如何throttle cfs_rq

假设上述assign_cfs_rq_runtime()函数返回0，意味着申请时间失败。cfs_rq需要被throttle。函数返回后，会设置TIF_NEED_RESCHED flag，意味着调度即将开始。调度器核心层通过pick_next_task()函数挑选出下一个应该运行的进程。CFS调度器的pick_next_task接口函数是pick_next_task_fair()。CFS调度器挑选进程前会先put_prev_task()。在该函数中会调用接口函数put_prev_task_fair()，函数如下：

```c
static void put_prev_task_fair(struct rq *rq, struct task_struct *prev){	struct sched_entity *se = &prev->se;	struct cfs_rq *cfs_rq; 	for_each_sched_entity(se) {		cfs_rq = cfs_rq_of(se);		put_prev_entity(cfs_rq, se);	}}
```

prev指向即将被调度出去的进程，我们会在put_prev_entity()函数中调用check_cfs_rq_runtime()检查`cfs_rq->runtime_remaining`的值是否小于0，如果小于0就需要被throttle。

```c
static bool check_cfs_rq_runtime(struct cfs_rq *cfs_rq){	if (!cfs_bandwidth_used())		return false; 	if (likely(!cfs_rq->runtime_enabled || cfs_rq->runtime_remaining > 0))    /* 1 */		return false; 	if (cfs_rq_throttled(cfs_rq))    /* 2 */		return true; 	throttle_cfs_rq(cfs_rq);         /* 3 */	return true;}
```

> 1. 检查cfs_rq是否满足被throttle的条件，可用运行时间小于0。
> 2. 如果该cfs_rq已经被throttle，这不需要重复操作。
> 3. throttle_cfs_rq()函数是真正throttle的操作，throttle核心函数。

throttle_cfs_rq()函数如下：
```cpp
1. static void throttle_cfs_rq(struct cfs_rq *cfs_rq)
2. {
3. 	struct rq *rq = rq_of(cfs_rq);
4. 	struct cfs_bandwidth *cfs_b = tg_cfs_bandwidth(cfs_rq->tg);
5. 	struct sched_entity *se;
6. 	long task_delta, dequeue = 1;
7. 	bool empty;

9. 	se = cfs_rq->tg->se[cpu_of(rq_of(cfs_rq))];              /* 1 */

11. 	/* freeze hierarchy runnable averages while throttled */
12. 	rcu_read_lock();
13. 	walk_tg_tree_from(cfs_rq->tg, tg_throttle_down, tg_nop, (void *)rq);   /* 2 */
14. 	rcu_read_unlock();

16. 	task_delta = cfs_rq->h_nr_running;
17. 	for_each_sched_entity(se) {
18. 		struct cfs_rq *qcfs_rq = cfs_rq_of(se);
19. 		/* throttled entity or throttle-on-deactivate */
20. 		if (!se->on_rq)
21. 			break;

23. 		if (dequeue)
24. 			dequeue_entity(qcfs_rq, se, DEQUEUE_SLEEP);   /* 3 */
25. 		qcfs_rq->h_nr_running -= task_delta;

27. 		if (qcfs_rq->load.weight)                         /* 4 */
28. 			dequeue = 0;
29. 	}

31. 	if (!se)
32. 		sub_nr_running(rq, task_delta);

34. 	cfs_rq->throttled = 1;                                /* 5 */
35. 	cfs_rq->throttled_clock = rq_clock(rq);
36. 	raw_spin_lock(&cfs_b->lock);
37. 	empty = list_empty(&cfs_b->throttled_cfs_rq);
38. 	list_add_rcu(&cfs_rq->throttled_list, &cfs_b->throttled_cfs_rq);   /* 6 */
39. 	if (empty)
40. 		start_cfs_bandwidth(cfs_b);
41. 	raw_spin_unlock(&cfs_b->lock);
42. } 
```
  
> 1. throttle对应的cfs_rq可以将对应的group se从其就绪队列的红黑树上删除，这样在pick_next_task的时候，顺着根cfs_rq的红黑树往下便利，就不会找到已经throttle的se，也就是没有机会运行。
> 2. `task_group`可以父子关系嵌套。walk_tg_tree_from()函数功能是顺着cfs_rq->tg往下便利每一个child task_group，并且对每个task_group调用tg_throttle_down()函数。tg_throttle_down()负责增加`cfs_rq->throttle_count`计数。
> 3. 从依附的cfs_rq的红黑树上删除。
> 4. 如果qcfs_rq运行的进程只有即将被dequeue的se一个的话，那么parent se也需要dequeue。如果`qcfs_rq->load.weight`不为0，说明qcfs_rq就绪队列上运行的进程不止se一个，那么parent se理所应当不能被dequeue。
> 5. 设置throttle标志位。
> 6. 记录throttle时刻。
> 7. 被throttle的cfs_rq加入cfs_b链表中，方便后续unthrottle操作可以找到这些已经被throttle的cfs_rq。

tg_throttle_down()函数如下，主要是cfs_rq->throttle_count计数递增：

```c
static int tg_throttle_down(struct task_group *tg, void *data){	struct rq *rq = data;	struct cfs_rq *cfs_rq = tg->cfs_rq[cpu_of(rq)]; 	/* group is entering throttled state, stop time */	if (!cfs_rq->throttle_count)		cfs_rq->throttled_clock_task = rq_clock_task(rq);	cfs_rq->throttle_count++; 	return 0;}
```

throttle cfs_rq时，数据结构示意图如下：

[![throttle-cfs_rq.png](http://www.wowotech.net/content/uploadfile/201812/43d91545462538.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201812/43d91545462538.png)
顺着被throttle cfs_rq依附的task_group的children链表，找到所有的task_group，并增加对应CPU的cfs_rq->throttle_count成员。
## 如何unthrottle cfs_rq

unthrottle cfs_rq操作会在周期定时器定时时间到达之际进行。负责unthrottle cfs_rq操作的函数是unthrottle_cfs_rq()，该函数和throttle_cfs_rq()的操作相反。函数如下：
```cpp
1. void unthrottle_cfs_rq(struct cfs_rq *cfs_rq)
2. {
3. 	struct rq *rq = rq_of(cfs_rq);
4. 	struct cfs_bandwidth *cfs_b = tg_cfs_bandwidth(cfs_rq->tg);
5. 	struct sched_entity *se;
6. 	int enqueue = 1;
7. 	long task_delta;

9. 	se = cfs_rq->tg->se[cpu_of(rq)];             /* 1 */
10. 	cfs_rq->throttled = 0;                       /* 2 */
11. 	update_rq_clock(rq);
12. 	raw_spin_lock(&cfs_b->lock);
13. 	cfs_b->throttled_time += rq_clock(rq) - cfs_rq->throttled_clock;  /* 3 */
14. 	list_del_rcu(&cfs_rq->throttled_list);       /* 4 */
15. 	raw_spin_unlock(&cfs_b->lock);

17. 	/* update hierarchical throttle state */
18. 	walk_tg_tree_from(cfs_rq->tg, tg_nop, tg_unthrottle_up, (void *)rq);    /* 5 */

20. 	if (!cfs_rq->load.weight)                    /* 6 */
21. 		return;

23. 	task_delta = cfs_rq->h_nr_running;
24. 	for_each_sched_entity(se) {
25. 		if (se->on_rq)
26. 			enqueue = 0;

28. 		cfs_rq = cfs_rq_of(se);
29. 		if (enqueue)
30. 			enqueue_entity(cfs_rq, se, ENQUEUE_WAKEUP);    /* 7 */
31. 		cfs_rq->h_nr_running += task_delta;

33. 		if (cfs_rq_throttled(cfs_rq))
34. 			break;
35. 	}

37. 	if (!se)
38. 		add_nr_running(rq, task_delta);

40. 	/* Determine whether we need to wake up potentially idle CPU: */
41. 	if (rq->curr == rq->idle && rq->cfs.nr_running)
42. 		resched_curr(rq);
43. } 
```
  
> 1. unthrottle操作的做cfs_rq对应的调度实体，调度实体在parent cfs_rq上才有机会运行。
> 2. throttled标志位清零。
> 3. throttled_time记录cfs_rq被throttle的总时间，throttled_clock在throttle_cfs_rq()函数中记录开始throttle时刻。
> 4. 从链表上删除自己。
> 5. tg_unthrottle_up()函数是tg_throttle_down()函数的反操作，递减`cfs_rq->throttle_count`计数。
> 6. 如果unthrottle的cfs_rq上没有进程，那么无需进行enqueue操作。`cfs_rq->load.weight`为0就代表就绪队列上没有可运行的进程。
> 7. 将调度实体入队，这里的for循环操作和throttle_cfs_rq()函数的dequeue操作对应。

tg_unthrottle_up()函数如下：
```cpp
1. static int tg_unthrottle_up(struct task_group *tg, void *data)
2. {
3. 	struct rq *rq = data;
4. 	struct cfs_rq *cfs_rq = tg->cfs_rq[cpu_of(rq)];

6. 	cfs_rq->throttle_count--;
7. 	if (!cfs_rq->throttle_count) {
8. 		/* adjust cfs_rq_clock_task() */
9. 		cfs_rq->throttled_clock_task_time += rq_clock_task(rq) -
10. 					     cfs_rq->throttled_clock_task;
11. 	}

13. 	return 0;
14. } 
```

除了递减`cfs_rq->throttle_count`计数外，还计算了throttled_clock_task_time时间。和throttled_time不同的是，throttled_clock_task_time时间还包括由于parent cfs_rq被throttle的时间。虽然自己是unthrottle状态，但是parent cfs_rq是throttle状态，自己也是没办法运行的。所以throttled_clock_task_time统计的是`cfs_rq->throttle_count`从非零变成0经历的时间总和。
## 周期更新quota

带宽的限制是以`task_group`为单位，每一个`task_group`内嵌`cfs_bandwidth`结构体。周期性的更新quota利用的是高精度定时器，周期是period。`struct hrtimer period_timer`嵌在`cfs_bandwidth`结构体就是为了这个目的。定时器的初始化函数是init_cfs_bandwidth()。
```cpp
1. void init_cfs_bandwidth(struct cfs_bandwidth *cfs_b)
2. {
3. 	raw_spin_lock_init(&cfs_b->lock);
4. 	cfs_b->runtime = 0;
5. 	cfs_b->quota = RUNTIME_INF;
6. 	cfs_b->period = ns_to_ktime(default_cfs_period());

8. 	INIT_LIST_HEAD(&cfs_b->throttled_cfs_rq);
9. 	hrtimer_init(&cfs_b->period_timer, CLOCK_MONOTONIC, HRTIMER_MODE_ABS_PINNED);
10. 	cfs_b->period_timer.function = sched_cfs_period_timer;
11. 	hrtimer_init(&cfs_b->slack_timer, CLOCK_MONOTONIC, HRTIMER_MODE_REL);
12. 	cfs_b->slack_timer.function = sched_cfs_slack_timer;
13. } 
```

初始化两个hrtimer，分别是period_timer和slack_timer。period_timer的回调函数是sched_cfs_period_timer()。回调函数中刷新quota，并调用distribute_cfs_runtime()函数unthrottle cfs_rq。distribute_cfs_runtime()函数如下：
```cpp
1. static u64 distribute_cfs_runtime(struct cfs_bandwidth *cfs_b,
2. 		u64 remaining, u64 expires)
3. {
4. 	struct cfs_rq *cfs_rq;
5. 	u64 runtime;
6. 	u64 starting_runtime = remaining;

8. 	rcu_read_lock();
9. 	list_for_each_entry_rcu(cfs_rq, &cfs_b->throttled_cfs_rq,   /* 1 */
10. 				throttled_list) {
11. 		struct rq *rq = rq_of(cfs_rq);
12. 		struct rq_flags rf;

14. 		rq_lock(rq, &rf);
15. 		if (!cfs_rq_throttled(cfs_rq))
16. 			goto next;

18. 		runtime = -cfs_rq->runtime_remaining + 1;
19. 		if (runtime > remaining)
20. 			runtime = remaining;
21. 		remaining -= runtime;                                /* 2 */

23. 		cfs_rq->runtime_remaining += runtime;                /* 3 */
24. 		cfs_rq->runtime_expires = expires;

26. 		/* we check whether we're throttled above */
27. 		if (cfs_rq->runtime_remaining > 0)
28. 			unthrottle_cfs_rq(cfs_rq);                       /* 3 */

30. next:
31. 		rq_unlock(rq, &rf);

33. 		if (!remaining)
34. 			break;
35. 	}
36. 	rcu_read_unlock();

38. 	return starting_runtime - remaining;
39. } 
```

> 1. 循环便利所有已经throttle cfs_rq，函数参数remaining是全局时间池剩余可运行时间。
> 2. remaining是全局时间池剩余时间，这里借给cfs_rq的时间是runtime。
> 3. 如果从全局时间池借到的时间保证cfs_rq->runtime_remaining的值应该大于0，执行unthrottle操作。

另外一个slack_timer的作用是什么呢？我们先思考另外一个问题，如果cfs_rq从全局时间池申请5ms时间片，该cfs_rq上只有一个进程，该进程运行0.5ms后就睡眠了，按照CFS的代码逻辑，整个cfs_rq对应的group se都会被dequeue。那么剩余的4.5ms是否应该归返给全局时间池呢？如果不归返，可能这个进程失眠很久，而其他CPU的cfs_rq很有可能申请不到5ms时间片（全局时间池时间剩余4ms）导致throttle，实际上可用时间是8.5ms。因此，我们针对这种情况会归返部分时间，可以用在其他CPU上消耗。这步处理的函数调用流程是dequeue_entity()->return_cfs_rq_runtime()->__return_cfs_rq_runtime()。
```cpp
1. static void __return_cfs_rq_runtime(struct cfs_rq *cfs_rq)
2. {
3. 	struct cfs_bandwidth *cfs_b = tg_cfs_bandwidth(cfs_rq->tg);
4. 	s64 slack_runtime = cfs_rq->runtime_remaining - min_cfs_rq_runtime;  /* 1 */

6. 	if (slack_runtime <= 0)
7. 		return;

9. 	raw_spin_lock(&cfs_b->lock);
10. 	if (cfs_b->quota != RUNTIME_INF &&
11. 	    cfs_rq->runtime_expires == cfs_b->runtime_expires) {
12. 		cfs_b->runtime += slack_runtime;                                 /* 2 */

14. 		/* we are under rq->lock, defer unthrottling using a timer */
15. 		if (cfs_b->runtime > sched_cfs_bandwidth_slice() &&
16. 		    !list_empty(&cfs_b->throttled_cfs_rq))
17. 			start_cfs_slack_bandwidth(cfs_b);                            /* 3 */
18. 	}
19. 	raw_spin_unlock(&cfs_b->lock);

21. 	/* even if it's not valid for return we don't want to try again */
22. 	cfs_rq->runtime_remaining -= slack_runtime;                          /* 4 */
23. } 
```

> 1. min_cfs_rq_runtime的值是1ms，我们选择至少保留min_cfs_rq_runtime时间给自己，剩下的时间归返全局时间池。全部归返的做法也是不明智的，有可能该cfs_rq上很快就会有进程运行。如果全部归返，进程运行的时候需要立刻去全局时间池申请，效率低。
>     
> 2. 归返全局时间池slack_runtime时间。
>     
> 3. 开启slack_timer定时器条件有2个（从注释可以得知，使用定时器的原因是当前持有rq->lock锁）。
>     
>     - 全局时间池的时间大于5ms，这样才有可能供其他cfs_rq申请时间片（最小申请时间片大小是5ms）。
>     - 已经存在throttle的cfs_rq，现在开启slack_timer，在回调函数中尝试分配时间片，并unthrottle cfs_rq。
> 4. cfs_rq剩余可用时间减少。
>     

slack_timer定时器的回调函数是sched_cfs_slack_timer()。sched_cfs_slack_timer()调用do_sched_cfs_slack_timer()处理主要逻辑。
```cpp
1. static void do_sched_cfs_slack_timer(struct cfs_bandwidth *cfs_b)
2. {
3. 	u64 runtime = 0, slice = sched_cfs_bandwidth_slice();
4. 	u64 expires;

6. 	/* confirm we're still not at a refresh boundary */
7. 	raw_spin_lock(&cfs_b->lock);
8. 	if (runtime_refresh_within(cfs_b, min_bandwidth_expiration)) {     /* 1 */
9. 		raw_spin_unlock(&cfs_b->lock);
10. 		return;
11. 	}

13. 	if (cfs_b->quota != RUNTIME_INF && cfs_b->runtime > slice)         /* 2 */
14. 		runtime = cfs_b->runtime;

16. 	expires = cfs_b->runtime_expires;
17. 	raw_spin_unlock(&cfs_b->lock);

19. 	if (!runtime)
20. 		return;

22. 	runtime = distribute_cfs_runtime(cfs_b, runtime, expires);        /* 3 */

24. 	raw_spin_lock(&cfs_b->lock);
25. 	if (expires == cfs_b->runtime_expires)
26. 		cfs_b->runtime -= min(runtime, cfs_b->runtime);
27. 	raw_spin_unlock(&cfs_b->lock);
28. } 
```
  

> 1. 检查period_timer定是时间是否即将到来，如果period_timer时间到了会刷新全局时间池。因此借助于period_timer即可unthrottle cfs_rq。如果，period_timer定是时间还有一段时间，那么此时此刻需要借助当前函数unthrottle cfs_rq。
> 2. 全局时间池剩余可运行时间必须大于slice（默认5ms），因为cfs_rq申请时间片的单位是5ms。
> 3. distribute_cfs_runtime()函数已经分析过，根据传递的参数runtime计算可以unthrottle多少个cfs_rq，就unthrottle几个cfs_rq，尽力而为。

## 用户空间如何使用

CFS bandwidth control提供的接口是以cgroupfs的形式呈现。提供以下三个文件。

- cpu.cfs_quota_us: 一个周期内限额时间，就是文中一直说的quota
- cpu.cfs_period_us: 一个周期时间，就是文中一直说的period
- cpu.stat: 带宽限制状态信息

默认情况下cpu.cfs_quota_us=-1，cpu.cfs_period_us=100ms。quota的值为-1，代表不限制带宽。我们如果想限制带宽，可以往这两个文件写入合法值。quota和period合法值范围是1ms~1000ms。除此之外还需要考虑层级关系。写入cpu.cfs_quota_us负值将不限制带宽。

关于上文一直提到cfs_rq向全局时间池申请时间片固定大小默认是5ms，当然该值也是可以更改的。文件路径如下：

1. /proc/sys/kernel/sched_cfs_bandwidth_slice_us 

cpu.stat文件会输出以下3点信息。

- nr_periods: 目前已经经历时间周期数
- nr_throttled: 用户组发生带宽限制次数
- throttled_time: 用户组中调度实体总的限制时间和

### 用户组层级限制

cpu.cfs_quota_us和cpu.cfs_period_us接口可以将一个task_group带宽控制在：max(c_i) <= C(这里C代表parent task_group带宽，c_i代表它的children taskgroup)。所有的children task_group中最大带宽不能超过parent task_group带宽。但是，允许所有的children task_group带宽总额大于parent task_group带宽。即：\Sum (c_i) >= C。所以，task_group被throttle有两种可能原因：

1. task_group在一个周期时间内消耗完自己的quota
2. parent task_group在一个周期时间内消耗完自己的quota

第2种情况下，虽然child task_group仍然剩余quota没有消耗，但是child task_group也必须等到parent task_group下个周期时间到来。

### 使用举例

1. 设置task_group带宽100%
    
    period和quota都设置250ms，相当于提供task_group 1个CPU的带宽资源，总的CPU使用率是100%。
    
    1. echo 250000 > cpu.cfs_quota_us  /* quota = 250ms */
    2. echo 250000 > cpu.cfs_period_us /* period = 250ms */ 
    
2. 设置task_group带宽200%（多核系统）
    
    500ms period和1000ms quota设置相当于提供task_group 2个CPU的带宽资源，总的CPU使用率是200%。
    
    ```shell
     
    echo 1000000 > cpu.cfs_quota_us /* quota = 1000ms */echo 500000 > cpu.cfs_period_us /* period = 500ms */
    ```
    
    更大的period时间，可以增加task_group吞吐量。
    
3. 设置task_group带宽20%
    
    以下配置可以使用20% CPU带宽资源。
    
    1. `echo 10000 > cpu.cfs_quota_us  /* quota = 10ms */`
    2. `echo 50000 > cpu.cfs_period_us /* period = 50ms */` 
    
    在使用更小的period的情况下，周期越短相应延迟越小。
    

标签: [CFS](http://www.wowotech.net/tag/CFS) [bandwidth](http://www.wowotech.net/tag/bandwidth)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [CFS调度器（6）-总结](http://www.wowotech.net/process_management/452.html) | [CFS调度器（4）-PELT(per entity load tracking)](http://www.wowotech.net/process_management/450.html)»

**评论：**

**Tim**  
2021-09-12 23:09

按照CFS的代码逻辑，整个cfs_rq对应的group se都会被dequeue。那么剩余的4.5ms是否应该归返给全局时间池呢？如果不归返，可能这个进程失眠很久，而其他CPU的cfs_rq很有可能申请不到5ms时间片（全局时间池时间剩余4ms）导致throttle  
  
assign_cfs_rq_runtime 函数中下列代码应该是选择 全局时间池剩余时间和 5ms的较小值，所谓我理解，不会因为申请不到5ms时间片而导致 throttle ？  
amount = min(cfs_b->runtime, min_amount);

[回复](http://www.wowotech.net/process_management/451.html#comment-8311)

**wangxingxing**  
2020-08-12 20:09

HI您好：  
在unthrottle_cfs_rq中，CFS调度器会调用for_each_sched_entity循环当前cfs_rq向上依附的所有cfs_rq，如果他们没有on_rq的话，会调用enqueue_entity把他们都入队，但是我看到这里有一个检查：  
        if (cfs_rq_throttled(cfs_rq))  
            break;  
也就是说如果向上有一个依附的cfs_rq处于throttle状态的话直接break。这样的话即便当前cfs_rq成功enqueue，因为他依附的cfs_rq没有enqueue，所以也不能得到CPU执行的机会。所以想请教一下cfs如何保证unthrottle的cfs_rq可以立马被调度器调度到呢？

[回复](http://www.wowotech.net/process_management/451.html#comment-8093)

**wangxingxing**  
2020-08-12 15:42

Hi 您好：  
在period_timer的回调中，我们调用distribute_cfs_runtime为受限的cfs_rq重新设置runtime_remaining：  
        runtime = -cfs_rq->runtime_remaining + 1;  
        if (runtime > remaining)  
            runtime = remaining;  
        remaining -= runtime;  
  
        cfs_rq->runtime_remaining += runtime;  
这样的处理之后cfs_rq->runtime_remaining很多时候会等于1ns，看起来并没有为cfs_rq分配足够的运行时间。所以我的疑问是为什么不在100ms的period_timer到期的时候为cfs_rq分配足够的（比如5ms）可运行时间呢？只分配1ns的背后设计原理是什么呢？

[回复](http://www.wowotech.net/process_management/451.html#comment-8092)

**小顽石**  
2020-05-07 16:37

感觉：assign_cfs_rq_runtime()负责从“全局时间池中”申请时间片。感觉全局时间池说的容易让人误解，看源码我理解应该是从这个cfs_rq所在的taks_group中申请时间片；仅从“全局时间池”这个次来看，容易理解成从root group直接申请；

[回复](http://www.wowotech.net/process_management/451.html#comment-7980)

**xyz**  
2019-07-31 17:34

期待load balance和numa balance

[回复](http://www.wowotech.net/process_management/451.html#comment-7560)

**cw**  
2019-05-14 11:18

如果qcfs_rq运行的进程只有即将被dequeue的se一个的话，那么parent se也需要dequeue。如果qcfs_rq->load.weight不为0，说明qcfs_rq就绪队列上运行的进程不止se一个，那么parent se理所应当不能被dequeue  
  
“parent se理所应当不能被dequeue”这句话有没有问题？

[回复](http://www.wowotech.net/process_management/451.html#comment-7416)

**[smcdef](http://www.wowotech.net/)**  
2019-05-14 23:10

@cw：parent se 指的是即将被dequeue的se挂载的cfs rq对应的group se。感觉没什么问题，不知你指什么问题？

[回复](http://www.wowotech.net/process_management/451.html#comment-7420)

**conlin_lc**  
2019-03-20 09:23

学习一下

[回复](http://www.wowotech.net/process_management/451.html#comment-7328)

**null**  
2019-03-20 00:15

设置task_group带宽200%,这里是不是写错了，应该是1s的quota和500ms的period？

[回复](http://www.wowotech.net/process_management/451.html#comment-7326)

**[smcdef](http://www.wowotech.net/)**  
2019-04-21 15:08

@null：是的，多谢指正。

[回复](http://www.wowotech.net/process_management/451.html#comment-7365)

**gnx**  
2018-12-25 12:38

期待load balance和numa balance

[回复](http://www.wowotech.net/process_management/451.html#comment-7106)

**[smcdef](http://www.wowotech.net/)**  
2018-12-27 08:50

@gnx：后续可以考虑，现在暂时还没打算

[回复](http://www.wowotech.net/process_management/451.html#comment-7109)

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
    
    - [Linux电源管理(1)_整体架构](http://www.wowotech.net/pm_subsystem/pm_architecture.html)
    - [Debian下的WiFi实验（一）：通过无线网卡连接AP](http://www.wowotech.net/linux_application/wifi-test-1.html)
    - [Device Tree（四）：文件结构解析](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html)
    - [MMC/SD/SDIO介绍](http://www.wowotech.net/basic_tech/mmc_sd_sdio_intro.html)
    - [Linux PM domain framework(1)_概述和使用流程](http://www.wowotech.net/pm_subsystem/pm_domain_overview.html)
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
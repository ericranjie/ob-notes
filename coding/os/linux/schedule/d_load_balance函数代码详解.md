
作者：[OPPO内核团队](http://www.wowotech.net/author/538) 发布于：2022-2-16 7:29 分类：[进程管理](http://www.wowotech.net/sort/process_management)

# 前言

我们描述CFS任务负载均衡的系列文章一共三篇，第一篇是框架部分，第二篇描述了task placement和active upmigration两个典型的负载均衡场景，第三篇是负载均衡的情景分析，包括tick balance、nohz idle balance和new idle balance。在负载均衡情景分析文档最后，我们给出了结论：tick balancing、nohz idle balancing、new idle balancing都是万法归宗，汇聚到load_balance函数来完成具体的负载均衡工作。本文就是第三篇负载均衡情景分析的附加篇，重点给大家展示load_balance函数的精妙。

本文出现的内核代码来自Linux5.10.61，为了减少篇幅，我们对引用的代码进行了删减（例如去掉了NUMA的代码，毕竟手机平台上我们暂时不关注这个特性），如果有兴趣，读者可以配合完整的源代码代码阅读本文。

# 一、概述

本文主要分成三个部分，第一个部分就是本章，简单的描述了本文的结构和阅读前提条件。第二章是对load_balance函数设计的数据结构进行描述。这一章不需要阅读，只是在有需要的时候可以查阅几个主要数据结构的各个成员的具体功能。随后的若干个章节是以load_balance函数为主线，对各个逻辑过程进行逐行分析。

需要强调的是本文不是独立成文的，很多负载均衡的基础知识（例如sched domain、sched group，什么是负载、运行负载、利用率utility，什么是均衡......）在CFS任务负载均衡系列文章的第一篇已经描述，如果没有阅读过，强烈建议提前阅读。如果已经具体负载均衡的基础概念，那么希望本文能够给你带来研读代码的快乐。

# 二、load_balance函数使用的数据结构

## 1、struct lb_env

在负载均衡的时候，通过 lb_env数据结构来表示本次负载均衡的上下文：

|                                      |                                                                                                                                                                                                        |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 成员                                   | 描述                                                                                                                                                                                                     |
| struct sched_domain \*sd              | 要进行负载均衡的sched domain                                                                                                                                                                                   |
| struct rq \*dst_rq<br><br>int dst_cpu | 本次均衡的目标CPU。均衡操作试图从该sched domain的busiest cpu的runqueue拉取任务到 dest CPU rq，从而完成本次sched domain的均衡动作。第一轮均衡的dst cpu和dst rq一般设置为发起均衡的cpu及其runqueue，后续如果需要，可以重新设定为local group中的其他cpu，具体可以参考后文的描述。                |
| struct cpumask \*dst_grpmask          | Dst_cpu所在sched group的cpu mask，即本次均衡dest cpu所在的范围。                                                                                                                                                      |
| struct rq \*src_rq<br><br>int src_cpu | 该sched domain中最繁忙的那个cpu及其runqueue，均衡的目标就是从该cpu的runqueue拉取任务出来                                                                                                                                          |
| int new_dst_cpu                      | 一般而言，均衡的dst cpu是发起均衡的cpu，但是，如果因为affinity的原因，src上有任务无法迁移到dst cpu，从而不能完成均衡操作的时候，我们会选择一个新的（仍然在local group内）CPU作为dst cpu，发起第二轮均衡。                                                                          |
| enum cpu_idle_type idle              | 在进行均衡的时候，dst_cpu的idle状态，这个状态会影响均衡的走向                                                                                                                                                                   |
| long imbalance                       | 对这个成员的解释需要结合migration_type：<br><br>migrate_load---表示要迁移的负载量<br><br>migrate_util----表示要迁移的utility<br><br>migrate_task---表示要迁移的任务个数<br><br>migrate_misfit---设定为1                                         |
| struct cpumask \*cpus                 | Load_balance的过程中会有多轮的均衡操作，不同轮次的均衡会涉及不同的cpus，这个成员指明了本次均衡有哪些CPUs参与。                                                                                                                                      |
| unsigned int flags                   | 标记负载均衡的标志。LBF_NOHZ_STATS和LBF_NOHZ_AGAIN主要用于负载均衡过程中更新nohz状态使用。当选中的busiest cpu上的所有任务都因为affinity无法进行迁移，这时会设置LBF_ALL_PINNED，负载均衡会寻找次忙CPU进行下一轮的均衡。LBF_NEED_BREAK主要用来减少均衡过程中关中断时间的。其他的flag的含义可以参考下面对代码的具体解释。 |
| unsigned int loop                    | 如果确定需要通过迁移任务来保持负载均衡，那么load_balance函数会通过循环遍历src rq上的cfs task链表来确定迁移的任务数量。Loop会跟踪循环的次数，其值不能大于Loop_max。                                                                                                   |
| unsigned int loop_break              | 如果一次迁移任务数量比较多，那么每迁移sched_nr_migrate_break个任务就休息一下，让关中断的临界区小一点。                                                                                                                                         |
| unsigned int loop_max                | 扫描dest cpu运行队列的最大次数。                                                                                                                                                                                   |
| enum migration_type migration_type   | 为了达到sched domain负载均衡的目标，本次迁移的类型为何？有四种迁移类型：<br><br>migrate_load---迁移一定量的负载<br><br>migrate_util----迁移一定量的utility<br><br>migrate_task---迁移一定数量的任务<br><br>migrate_misfit---迁移misfit task                   |
| struct list_head tasks               | 需要进行迁移的任务链表                                                                                                                                                                                            |

## 2、struct sd_lb_stats

在负载均衡的时候，通过sd_lb_stats数据结构来表示sched domain的负载统计信息：

|   |   |
|---|---|
|成员|描述|
|struct sched_group \*busiest|该sched domain中，最繁忙的那个sched group（非local group）|
|struct sched_group \*local|在该sched domain上进行均衡的时候，标记该sd中哪一个group是local group，即dest cpu所在的group|
|unsigned long total_load|该sched domain中所有sched group的负载之和。如果没有特别说明，本文说的负载都是指cfs任务的负载。|
|unsigned long total_capacity|该sched domain中所有sched group的CPU算力之和（可以用于cfs task的算力）|
|unsigned long avg_load|该sched domain中sched groups的平均负载|
|struct sg_lb_stats busiest_stat|本sched domain中最忙的那个sched group的负载统计信息|
|struct sg_lb_stats local_stat|Dest cpu所在的本地sched group的负载统计|

3、struct sg_lb_stats

在负载均衡的时候，通过sg_lb_stats数据结构来表示sched group的负载统计信息：

|   |   |
|---|---|
|成员|描述|
|avg_load|该sched group上每个CPU的平均负载。仅在sched group处于group_overloaded状态下才计算该值，方便计算迁移负载量。|
|group_load|该sched group上所有CPU的负载之和|
|group_capacity|该sched group的所有cpu算力之和。这里的cpu算力是指可以用于cfs任务的算力。|
|group_util|该sched group上所有CPU利用率之和|
|group_runnable|该sched group上所有CPU的运行负载之和。|
|sum_nr_running|该sched group上所有任务的数量，包括rt、dl任务|
|sum_h_nr_running|该sched group上所有cfs任务的数量|
|idle_cpus|该group中idle cpu的数量|
|group_weight|该group中的cpu数量|
|group_type|该group在负载均衡时候所处的状态，下面代码分析过程中会详细解析各种状态。|
|group_misfit_task_load|该组内至少有一个cpu上有misfit task，这里记录了该组所有CPU中，misfit task load最大的值。|

## 4、struct sched_group_capacity

数据结构sched_group_capacity用来描述sched group的算力信息：

|                            |                                                     |
| -------------------------- | --------------------------------------------------- |
| 成员                         | 描述                                                  |
| atomic_t ref               | 有可能多个sched group会共享sched_group_capacity，因此需要一个引用计数。 |
| unsigned long capacity     | 该group中可以用于cfs任务的总算力（各个CPU算力之和）                     |
| unsigned long min_capacity | 该sched group中最小的可用于cfs任务的capacity（对单个CPU而言）         |
| unsigned long max_capacity | 该sched group中最大的可用于cfs任务的capacity（对单个CPU而言）         |
| unsigned long next_update  | 下一次更新算力的时间点                                         |
| int imbalance              | 该group中是否有由于affinity原因产生的不均衡问题                      |
| unsigned long cpumask\[\]    | Balance mask                                        |

# 三、load_balance函数整体逻辑

从本章开始我们进行代码分析，这一章是load_balance函数的整体逻辑，后面的章节都是对本章中的一些细节内容进行补充。load_balance函数实在是太长了，我们分段解读。第一段的逻辑如下：

|   |
|---|
|static int load_balance(int this_cpu, struct rq \*this_rq,<br><br>struct sched_domain \*sd, enum cpu_idle_type idle,<br><br>int \*continue_balancing)---------------------A<br><br>{<br><br>    struct lb_env env = {---------------------------B<br><br>    .sd = sd,<br><br>    .dst_cpu = this_cpu,<br><br>    .dst_rq = this_rq,<br><br>    .dst_grpmask    = sched_group_span(sd->groups),<br><br>    .idle = idle,<br><br>    .loop_break = sched_nr_migrate_break,<br><br>    .cpus = cpus,<br><br>    .fbq_type = all,<br><br>    .tasks = LIST_HEAD_INIT(env.tasks),<br><br>};|

A、对load_balance函数的参数以及返回值解释如下：

|   |   |
|---|---|
|参数及返回值|描述|
|this_cpu|本次要进行负载均衡的CPU。需要注意的是：对于new idle balance和tick balance而言，this_cpu等于current cpu，在nohz idle balance场景中，this_cpu未必等于current cpu。|
|this_rq|本次负载均衡CPU对应的runqueue|
|sd|本次均衡的范围，即本次均衡要保证该sched domain上各个group处于负载平衡状态|
|idle|this_cpu在发起均衡的时候所处的状态，通过这个状态可以识别new idle load balance和tick balance。|
|continue_balancing|负载均衡是从发起CPU的base domain开始，不断向上，直到顶层的sched domain。continue_balancing是用来控制是否继续进行上层sched domain的均衡|
|返回值|本次负载均衡迁移的任务总数|

B、初始化本次负载均衡的上下文信息。具体可以参考对struct lb_env的解释。

初始化完第一轮均衡的上下文，下面就看看具体的均衡操作为何。第二段的逻辑如下：

|                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| cpumask_and(cpus, sched_domain_span(sd), cpu_active_mask);-------A<br><br>redo:<br><br>if (!should_we_balance(&env)) {<br><br>    \*continue_balancing = 0;<br><br>    goto out_balanced;-------------------B<br><br>}<br><br>group = find_busiest_group(&env);<br><br>if (!group) {<br><br>    goto out_balanced;------------------C<br><br>}<br><br>busiest = find_busiest_queue(&env, group);<br><br>if (!busiest) {<br><br>    goto out_balanced;-----------------D<br><br>} |

A、确定本轮负载均衡涉及的cpu，因为是第一轮均衡，所以所有的sched domain中的cpu都参与均衡（cpu_active_mask用来剔除无法参与均衡的CPU）。后续如果发现一些异常状况（例如由于affinity原因无法完成任务迁移），那么会清除选定的busiest cpu，跳转到redo进行全新一轮的均衡。

B、判断env->dst_cpu这个CPU是否适合做指定sched domain的均衡。如果被认定不适合发起balance，那么后续更高层level的均衡也不必进行了（设置continue_balancing等于0）。在base domain，每个group都只有一个CPU，因此所有的cpu都可以发起均衡。在non-base domain，每个group有多个CPU，如果每一个cpu都可以进行均衡，那么均衡就太密集了，白白消耗CPU资源，所以限制只有第一个idle的cpu可以发起均衡，如果没有idle的CPU，那么group中的第一个CPU可以发起均衡。当然，对于new idle balance没有这样的限制，所以的cpu都可以发起均衡。

C、在该sched domain中寻找最繁忙的sched group。具体逻辑后文会详细描述。如果没有找到busiest group，那么退出本level的均衡

D、在最繁忙的sched group寻找最繁忙的CPU。具体逻辑后文会详细描述。如果没有找到busiest cpu，那么退出本level的均衡

至此已经找到了source CPU，dest cpu就是发起均衡的this cpu，那么就可以开始第一轮的任务迁移了，具体的代码逻辑如下：

|                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| if (busiest->nr_running > 1) {-------------------A<br><br>    env.flags |= LBF_ALL_PINNED;<br><br>    env.loop_max = min(sysctl_sched_nr_migrate, busiest->nr_running);-----B<br><br>more_balance:-----------------------C<br><br>    rq_lock_irqsave(busiest, &rf);<br><br>    update_rq_clock(busiest);<br><br>    cur_ld_moved = detach_tasks(&env);----------D<br><br>    rq_unlock(busiest, &rf);<br><br>    if (cur_ld_moved) {<br><br>        attach_tasks(&env);--------------------E<br><br>        ld_moved += cur_ld_moved;<br><br>    }<br><br>    local_irq_restore(rf.flags);<br><br>    if (env.flags & LBF_NEED_BREAK) {<br><br>        env.flags &= ~LBF_NEED_BREAK;<br><br>        goto more_balance;-------------------F<br><br>    }<br><br>.......<br><br>} |

A、如果要从busiest cpu迁移任务到this cpu，那么至少要有可以拉取的任务。在拉取任务之前，我们先设定all pinned标志。当然后续如果发现不是all pinned的状况就会清除这个标志。

B、为了达到sched domain的负载均衡，我们需要进行任务的迁移，因此我们这里需要遍历busiest rq上的任务，看看哪些任务最适合被迁移到this cpu rq。loop_max就是扫描src rq上runnable任务的次数。一般而言，任务迁移上限就是busiest runqueue上的任务个数，确保了每一个任务都被扫描到，但是一次均衡操作不适合迁移太多的任务（关中断区间太长），因此，即便busiest runqueue上的任务个数非常多，一次任务迁移不能大于sysctl_sched_nr_migrate个（目前设定是32个）。

C、和redo不同，跳转到more_balance的新一轮迁移不需要寻找busiest cpu，只是继续扫描busiest rq上的任务列表，寻找适合迁移的任务。之所以这么做主要是为了降低关中断的时长。

D、detach_tasks函数用来从busiest cpu的rq中摘取适合的任务。具体逻辑后面会详细描述。由于关中断时长的问题，detach_tasks函数也不会一次性把所有任务迁移到dest cpu上。

E、将detach_tasks函数摘下的任务挂入到src rq上去。由于detach_tasks、attach_tasks会进行多轮，ld_moved记录了总共迁移的任务数量，cur_ld_moved是本轮迁移的任务数

F、在任务迁移过程中，src cpu的中断是关闭的，为了降低这个关中断时间，迁移大量任务的时候需要break一下。

至此已经对dest rq上的任务列表完成了loop_max次扫描，要看情况是否要发起下一轮次的均衡。具体代码如下：

|   |
|---|
|if ((env.flags & LBF_DST_PINNED) && env.imbalance > 0) {<br><br>    \_\_cpumask_clear_cpu(env.dst_cpu, env.cpus);<br><br>    env.dst_rq  = cpu_rq(env.new_dst_cpu);<br><br>    env.dst_cpu  = env.new_dst_cpu;<br><br>    env.flags &= ~LBF_DST_PINNED;<br><br>    env.loop  = 0;<br><br>    env.loop_break  = sched_nr_migrate_break;<br><br>    goto more_balance;-----------------------A<br><br>}<br><br>if (sd_parent) {----------------------B<br><br>    int \*group_imbalance = &sd_parent->groups->sgc->imbalance;<br><br>    if ((env.flags & LBF_SOME_PINNED) && env.imbalance > 0)<br><br>        \*group_imbalance = 1;<br><br>}<br><br>if (unlikely(env.flags & LBF_ALL_PINNED)) {<br><br>    \_\_cpumask_clear_cpu(cpu_of(busiest), cpus);<br><br>    if (!cpumask_subset(cpus, env.dst_grpmask)) {<br><br>        env.loop = 0;<br><br>        env.loop_break = sched_nr_migrate_break;<br><br>        goto redo;-------------C<br><br>    }<br><br>    goto out_all_pinned;<br><br>}|

A、如果sched domain仍然未达均衡均衡状态，并且在之前的均衡过程中，有因为affinity的原因导致任务无法迁移到dest cpu，这时候要继续在src rq上搜索任务，迁移到备选的dest cpu，因此，这里再次发起均衡操作。这里的均衡上下文的dest cpu设定为备选的cpu，loop也被清零，重新开始扫描。

B、本层次的sched domain因为affinity而无法达到均衡状态，我们需要把这个状态标记到上层sched domain的group中去，在上层sched domain进行均衡的时候，该group会被判定为group_imbalanced，从而有更大的机会选定为busiest group，从而解决该sched domain的均衡问题。

C、如果选中的busiest cpu上的任务全部都是通过affinity锁定在了该cpu上，那么清除该cpu（为了确保下轮均衡不考虑该cpu），再次发起均衡。这种情况下，需要重新搜索source cpu，因此跳转到redo。

至此，source rq上的cfs任务链表已经被遍历（也可能遍历多次），基本上对runnable 任务的扫描已经到位了，如果不行就只能考虑running task了，具体代码逻辑如下：

|                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| if (!ld_moved) {<br><br>    if (idle != CPU_NEWLY_IDLE)<br><br>        sd->nr_balance_failed++;-----------------------A<br><br>    if (need_active_balance(&env)) {-------------------B<br><br>        unsigned long flags;<br><br>        raw_spin_lock_irqsave(&busiest->lock, flags);<br><br>        if (!cpumask_test_cpu(this_cpu, busiest->curr->cpus_ptr)) {<br><br>            raw_spin_unlock_irqrestore(&busiest->lock,    flags);<br><br>            env.flags |= LBF_ALL_PINNED;<br><br>            goto out_one_pinned;------------------C<br><br>        }<br><br>        if (!busiest->active_balance) {--------------D<br><br>            busiest->active_balance = 1;<br><br>            busiest->push_cpu = this_cpu;<br><br>            active_balance = 1;<br><br>        }<br><br>        raw_spin_unlock_irqrestore(&busiest->lock, flags);<br><br>        if (active_balance) {--------------------E<br><br>            stop_one_cpu_nowait(cpu_of(busiest),<br><br>                        active_load_balance_cpu_stop, busiest,<br><br>                        &busiest->active_balance_work);<br><br>        }<br><br>        sd->nr_balance_failed = sd->cache_nice_tries+1;<br><br>    }<br><br>} else<br><br>sd->nr_balance_failed = 0;-----------F |

A、经过上面的一系列操作，没有完成任何任务的迁移，那么就需要累计sched domain的均衡失败次数。这个失败次数会导致后续进行更激进的均衡，例如迁移cache hot的任务、启动active balance。此外，这里过滤掉了new idle balance的失败，仅统计周期性均衡失败的次数，这是因为系统中new idle balance次数太多，累计其失败次数会导致nr_balance_failed过大，容易触发后续激进的均衡。

B、判断是否要启动active balance。所谓active balance就是把当前正在运行的任务迁移到dest cpu上。也就是说经过前面一番折腾，runnable的任务都无法迁移到dest cpu，从而达到均衡，那么就考虑当前正在运行的任务。

C、在启动active balance之前，先看看busiest cpu上当前正在运行的任务是否可以运行在dest cpu上。如果不可以的话，那么不再试图执行均衡操作，跳转到out_one_pinned

D、Busiest cpu runqueue上设置active balance的标记

E、发起主动迁移

F、完成了至少一个任务迁移，重置均衡失败计数

Load_balance最后一段的程序逻辑主要是进行一些清理工作和设定balance_interval的工作，逻辑比较简单，不再详述，我们会在随后的章节中对load_balance函数中的一些过程做进一步的描述。

四、寻找sched domain中最繁忙的group

判断当前sched domain是否均衡并返回最忙group的功能是在find_busiest_group函数中完成的，我们分段来描述该函数的逻辑，我们先看第一段代码：

|   |
|---|
|update_sd_lb_stats(env, &sds);--------------A<br><br>if (sched_energy_enabled()) {<br><br>    struct root_domain \*rd = env->dst_rq->rd;<br><br>    if (rcu_dereference(rd->pd) && !READ_ONCE(rd->overutilized))<br><br>        goto out_balanced;-----------------B<br><br>}|

A、负载信息都是不断的在变化，在寻找最繁忙group的时候，我们首先要更新sched domain负载均衡信息，以便可以根据最新的负载情况来搜寻。update_sd_lb_stats会更新该sched domain上各个sched group的负载和算力，得到local group以及非local group最忙的那个group的均衡信息，以便后续给出最适合的均衡决策。具体的逻辑后面的章节会详述

B、在系统没有进入overutilized状态之前，EAS起作用。如果EAS起作用，那么负载可能是不均衡的（考虑功耗），因此，这时候不进行负载均衡，依赖task placement的结果。

update_sd_lb_stats函数找到了busiest group，结合local group的状态就可以判断系统的不均衡状态了。当然有一些比较容易判断是否进行均衡的场景，具体代码如下：

|   |
|---|
|if (!sds.busiest)---------------------------A<br><br>    goto out_balanced;<br><br>if (busiest->group_type == group_misfit_task)<br><br>    goto force_balance;--------------B<br><br>if (busiest->group_type == group_imbalanced)<br><br>    goto force_balance;--------------C<br><br>if (local->group_type > busiest->group_type)<br><br>    goto out_balanced;---------------D|

A、如果没有找到最忙的那个group，说明当前sched domain中，其他的非local的最繁忙的group（后文称之busiest group）没有可以拉取到local group的任务，不需要均衡处理。

B、Busiest group中有misfit task，那么必须要进行均衡，把misfit task拉取到local group中

C、Busiest group是一个由于cpu affinity导致的不均衡，这个不均衡在底层sched domain无法处理，需要在本层domain进行均衡。

D、如果local group比busiest group还要忙，那么不需要进行均衡（目前的均衡只能从其他group拉任务到local group）

其他的复杂场景需要进一步比拼local group和busiest group的情况，group_overloaded状态下判断是否均衡的代码如下：

|   |
|---|
|if (local->group_type == group_overloaded) {---------A<br><br>    if (local->avg_load >= busiest->avg_load)<br><br>        goto out_balanced;----------------------------B<br><br>    if (local->avg_load >= sds.avg_load)<br><br>        goto out_balanced;-----------------------------C<br><br>    if (100 * busiest->avg_load \<= env->sd->imbalance_pct * local->avg_load)<br><br>        goto out_balanced;----------------------------D<br><br>}|

A、如果local group和busiest group都比较繁忙（group_overloaded），那么需要通过avg_load的比拼来做均衡决策

B、如果local group的平均负载比busiest group还要高，那么不需要进行均衡

C、如果local group的平均负载高于sched domain的平均负载，那么不需要进行均衡

D、虽然busiest group的平均负载高于local group，但是高的不多，那也不需要进行均衡，毕竟均衡需要额外的开销。具体的门限是有sched domain的imbalance_pct确定的。

非group_overloaded不看平均负载，主要看idle cpu的情况，具体代码如下：

|   |
|---|
|if (busiest->group_type != group_overloaded) {-----------A<br><br>    if (env->idle == CPU_NOT_IDLE)<br><br>        goto out_balanced;-------------------------B<br><br>    if (busiest->group_weight > 1 &&<br><br>        local->idle_cpus \<= (busiest->idle_cpus + 1))<br><br>        goto out_balanced;-------------------------C<br><br>    if (busiest->sum_h_nr_running == 1)<br><br>        goto out_balanced;-------------------------D<br><br>}<br><br>force_balance:<br><br>calculate_imbalance(env, &sds);---------------------E<br><br>return env->imbalance ? sds.busiest : NULL;|

A、这里处理busiest group没有overload的场景，这时候说明该sched domain中其他的group的算力都是cover当前的任务负载，是否要进行均衡，主要看idle cpu的情况。

B、反正busiest group当前算力能处理其runqueue上的任务，那么在本CPU处于active的情况下没有必要进行均衡，因为这时候关注的是idle cpu，即让更多的idle cpu参与运算，因此，如果本CPU不是idle cpu，那么判断sched domain处于均衡状态。

C、如果busiest group中有更多的idle CPU，那么也没有必要进行均衡

D、如果busiest group中只有一个cfs任务，那么也没有必要进行均衡

E、所有其他情况都是需要进行均衡。calculate_imbalance用来计算sched domain中不均衡的状态是怎样的。

具体如何进行均衡决策可以参考下面的表格（第一行是local group状态，第一列是busiest group的状态）：

|   |   |   |   |   |   |
|---|---|---|---|---|---|
||Has spare|Fully busy|misfit|imbalanced|overloaded|
|Has spare|Nr idle|balanced|N/A|balanced|balanced|
|Fully busy|Nr idle|Nr idle|N/A|balanced|balanced|
|misfit|force|N/A|N/A|force|force|
|imbalanced|force|force|N/A|force|force|
|overloaded|force|force|N/A|force|Avg load|

Balanced：该sched domain处于均衡状态，不需要均衡。

Force：该sched domain处于不均衡状态，通过calculate_imbalance计算不均衡指数，并有可能通过任务迁移让系统进入均衡状态。

Avg load：通过sched group的平均负载来判断是否需要均衡。尽量不均衡，除非非常的不均衡（通过sched domain的imbalance_pct参数来设定）

Nr idle：dest cpu处于idle状态，并且local group的idle cpu个数大于busiest group的idle cpu个数，只有在这种情况下才进行均衡。

五、更新sched domain的负载统计

Sched domain的负载统计更新主要在update_sd_lb_stats函数中，其逻辑大致如下：

|   |
|---|
|do {<br><br>    local_group = cpumask_test_cpu(env->dst_cpu, sched_group_span(sg));<br><br>    if (local_group) {---------------------A<br><br>        sds->local = sg;<br><br>        sgs = local;<br><br>        if (env->idle != CPU_NEWLY_IDLE |<br><br>            time_after_eq(jiffies, sg->sgc->next_update))<br><br>             update_group_capacity(env->sd, env->dst_cpu);<br><br>    }<br><br>    update_sg_lb_stats(env, sg, sgs, &sg_status);------------B<br><br>    if (local_group)-----------------C<br><br>        goto next_group;<br><br>    if (update_sd_pick_busiest(env, sds, sg, sgs)) {-----------D<br><br>        sds->busiest = sg;<br><br>        sds->busiest_stat = \*sgs;<br><br>    }<br><br>next_group:<br><br>    sds->total_load += sgs->group_load;--------------E<br><br>    sds->total_capacity += sgs->group_capacity;<br><br>    sg = sg->next;<br><br>} while (sg != env->sd->groups);<br><br>if (!env->sd->parent) {----------------------F<br><br>    struct root_domain \*rd = env->dst_rq->rd;<br><br>    WRITE_ONCE(rd->overload, sg_status & SG_OVERLOAD);<br><br>    WRITE_ONCE(rd->overutilized, sg_status & SG_OVERUTILIZED);<br><br>} else if (sg_status & SG_OVERUTILIZED) {<br><br>    struct root_domain \*rd = env->dst_rq->rd;<br><br>    WRITE_ONCE(rd->overutilized, SG_OVERUTILIZED);<br><br>}|

这一段主要是遍历该sched domain的所有group，对其负载统计进行更新。更新完负载之后，我们选定两个sched group：其一是local group，另外一个是最繁忙的non local group。具体逻辑过程解释如下：

A、更新sched group的算力。在base domain（在手机平台上就是MC domain）上，我们会更新发起均衡所在CPU的算力。注意：这里说的CPU算力指的是该CPU可以用于cfs任务的算力，即需要去掉由于thermal pressure而损失的算力，去掉RT/DL/IRQ消耗的算力。具体请参考update_cpu_capacity函数。在其他non-base domain（在手机平台上就是DIE domain）上，我们需要对本地sched group（包括发起均衡的CPU所在的group）进行算力更新。这个比较简单，就是把child domain（即MC domain）的所有sched group的算力加起来就OK了。更新后的算力保存在sched group中的sgc成员中。

另外，更新算力没有必要更新的太频繁，这里做了两个限制：其一是只有local group才进行算力更新，其二是通过时间间隔来减少new idle频繁的更新算力。

B、更新该sched group的负载统计，下面的章节会详细描述。

C、在sched domain的各个group遍历中，我们需要两个group信息，一个是local group，另外一个就是non local group中的最忙的那个group。显然，如果是local group，不需要下面的比拼最忙的过程。

D、找到non local group中的最忙的那个group。由于涉及各种group type，我们在下一章详述如何判断一个group更忙。

E、更新sched domain上各个sched group总的负载和算力

F、更新root domain的overload和overutil状态。对于顶层的sched domain，我们需要把各个sched group的overload和overutil状态体现到root domain中。

六、更新sched group的负载

更新sched group负载是在update_sg_lb_stats函数中完成的，我们分段来描述该函数的逻辑，我们先看第一段代码：

|   |
|---|
|for_each_cpu_and(i, sched_group_span(group), env->cpus) {<br><br>    struct rq \*rq = cpu_rq(i);<br><br>    sgs->group_load += cpu_load(rq);<br><br>    sgs->group_util += cpu_util(i);<br><br>    sgs->group_runnable += cpu_runnable(rq);<br><br>    sgs->sum_h_nr_running += rq->cfs.h_nr_running;<br><br>    nr_running = rq->nr_running;<br><br>    sgs->sum_nr_running += nr_running;-------------------A<br><br>    if (nr_running > 1)<br><br>        \*sg_status |= SG_OVERLOAD;------------------B<br><br>    if (cpu_overutilized(i))<br><br>        \*sg_status |= SG_OVERUTILIZED;-------------C<br><br>     if (!nr_running && idle_cpu(i)) {-----------------D<br><br>        sgs->idle_cpus++;<br><br>        continue;<br><br>    }<br><br>    if (local_group)------------E<br><br>        continue;<br><br>    if (env->sd->flags & SD_ASYM_CPUCAPACITY &&<br><br>        sgs->group_misfit_task_load \< rq->misfit_task_load) {<br><br>        sgs->group_misfit_task_load = rq->misfit_task_load;<br><br>        \*sg_status |= SG_OVERLOAD;<br><br>    }<br><br>}|

A、sched group负载有三种，load、runnable load和util，把所有cpu上load、runnable load和util累计起来就是sched group的负载。除了PELT跟踪的load avg信息，我们还统计了sched group中的cfs任务和总任务数量。

B、只要该sched group上有一个CPU上有1个以上的任务，那么就标记该sched group为overload状态。

C、只要该sched group上有一个CPU处于overutilized（该cpu利用率已经达到cpu算力的80%），那么就标记该sched group为overutilized状态。

D、统计该sched group中的idle cpu的个数

E、当sched domain包括了算力不同的CPU（例如DIE domain），那么即便cpu上只有一个任务，但是如果该任务是misfit task那么也标记sched group为overload状态，并记录sched group中最大的misfit task load。需要注意的是：idle cpu不需要检测misfit task，此外，对于local group，也没有必要检测misfit task，毕竟同一个group，算力相同，不可能拉取misfit task到本cpu上。

第二段代码如下：

|   |
|---|
|sgs->group_capacity = group->sgc->capacity;--------------A<br><br>sgs->group_weight = group->group_weight;<br><br>sgs->group_type = group_classify(env->sd->imbalance_pct, group, sgs);------B<br><br>if (sgs->group_type == group_overloaded)-----C<br><br>sgs->avg_load = (sgs->group_load * SCHED_CAPACITY_SCALE) /<br><br>sgs->group_capacity;|

A、更新sched group的总算力和cpu个数。再次强调一下，这里的capacity是指cpu可以用于cfs任务的算力

B、判定sched group当前的负载状态

C、计算sched group平均负载（仅在group overloaded状态才计算）。在overload的情况下，通过sched group平均负载可以识别更繁忙的group。

sched group负载状态如下（按照负载从重到轻排列），括号中的数字是该group type的值，数值越大，载荷越重：

|                            |                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| -------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| group_type                 | 描述                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| group_overloaded（5）        | 这个状态说明该group已经过载，无法为其他任务提供算力。Overloaded的Sched group需要同时满足下面的条件：<br><br>1、group上总的任务数（不仅仅是cfs任务）大于group中cpu的个数，即至少有一个任务处于runnable状态。<br><br>2、Group上总的util或者runnalbe load大于group上cpu总算力。当然，判断过载需要考虑margin，不能等到util/runnable大于capacity才处理。具体margin和sched domain的imbalance_pct参数相关）<br><br>注意：正常运行起来之后，任务的runnable load总是大于util的。Runnable load是记录running+runnable，而util仅计算running time。因此这里util采用了对capacity的margin，而对runnable load采用了对runnable load的margin。 |
| group_imbalanced（4）        | 由于task affinity的原因导致该group处于不均衡状态                                                                                                                                                                                                                                                                                                                                                                                                                        |
| group_misfit_task（2）       | 该group中有misfit task需要迁移到算力更强的CPU上去                                                                                                                                                                                                                                                                                                                                                                                                                       |
| group_fully_busy（1）        | 该group没有空闲算力                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| group_has_spare<br><br>（0） | 该group还有空闲算力，可以承载其他group的任务。当sched group满足下面条件之一就处于这种状态：<br><br>1、group中总任务量小于group的cpu个数，即至少有一个cpu处于idle状态。<br><br>2、group总算力可以承载当前的group中的任务量<br><br>注意：这里的“任务量”需要考虑group util和group runnable load                                                                                                                                                                                                                                                     |

判断sched group繁忙程度的函数是group_classify，可以对照代码理解各sched group繁忙状态的含义。

在对比sched group繁忙程度的时候，我们主要是对比group_type的值，大的值更忙，小的值比较闲，在相等的时候的判断规则如下：

|   |   |
|---|---|
|group_type|group_type等值时候的判断条件|
|group_overloaded|那个平均负载高则更忙|
|group_imbalanced|第一个找到的group最忙|
|group_misfit_task|对比misfit task load，大的更忙|
|group_fully_busy|第一个找到的group最忙|
|group_has_spare|Idle cpu个数少的更忙，如果idle cpu数目相等，看group上的任务数|

七、如何计算sched domain的不均衡程度

一旦通过local group和busiest group的信息确定sched domain处于不均衡状态，我们就可以调用calculate_imbalance函数来计算通过什么方式（migrate task还是migrate load/util）来恢复sched domain的负载均衡状态，也就是设定均衡上下文的migration_type和imbalance 成员，下面我们分段来描述该函数的逻辑，我们先看第一段代码：

|   |
|---|
|if (busiest->group_type == group_misfit_task) {<br><br>    env->migration_type = migrate_misfit;<br><br>    env->imbalance = 1;<br><br>    return;-----------------------------------A<br><br>}<br><br>if (busiest->group_type == group_imbalanced) {<br><br>    env->migration_type = migrate_task;<br><br>    env->imbalance = 1;<br><br>    return;----------------------------------B<br><br>}|

A、如果busiest group上有misfit task，那么优先对其进行misfit任务迁移，并且一次迁移一个misfit task。

B、如果busiest group是因为cpu affinity而导致的不均衡，那么通过通过迁移任务来达到平衡，并且一次迁移一个任务。

上面的代码主要处理busiest group中的一些特殊情况，后面的代码主要分两段段来根据local group的状态来进行不均衡的计算。我们首先看local group有空闲算力的情况，我们分成两段分析，第一段代码如下：

|   |
|---|
|if (local->group_type == group_has_spare) {--------------A<br><br>    if ((busiest->group_type > group_fully_busy) &&<br><br>        !(env->sd->flags & SD_SHARE_PKG_RESOURCES)) {------B<br><br>        env->migration_type = migrate_util;<br><br>        env->imbalance = max(local->group_capacity, local->group_util) -<br><br>        local->group_util;-------------------C<br><br>        if (env->idle != CPU_NOT_IDLE && env->imbalance == 0) {<br><br>            env->migration_type = migrate_task;<br><br>            env->imbalance = 1;---------------D<br><br>        }<br><br>        return;<br><br>    }<br><br>......<br><br>}|

A、如果local group有一些空闲算力，那么我们还是争取把它利用起来，只要迁移的负载量既不overload local group，也不会让busiest group变得无事可做。

B、如果sched domain标记了SD_SHARE_PKG_RESOURCES（MC domain），那么其在task placement的时候会尽量选择idle cpu。这里load balance路径需要和placement对齐：不使用空闲capacity而是使用nr_running来进行均衡。如果没有设置SD_SHARE_PKG_RESOURCES那么考虑使用migrate_util方式来达到均衡。

C、如果local group有一些空闲算力，busiest group又处于繁忙状态（大于full busy），同时满足未设定SD_SHARE_PKG_RESOURCES（对于手机场景就是DIE domain，MC domain需要使用nr_running而不是util来进行均衡）。这种状态下，我们采用util来指导均衡，具体迁的utility设定为local group当前空闲的算力。

D、有些场景下，local group的util大于其group capacity，根据步骤C计算的imbalance等于0（意味着不需要均衡）。然而，在这种场景下，如果local cpu处于idle状态，那么需要从busiest group迁移过来一个runnable task，从而确保了性能。

Local gorup有空闲算力的第二段代码如下：

|   |
|---|
|if (local->group_type == group_has_spare) {<br><br>......<br><br>    if (busiest->group_weight == 1 ) {---------A<br><br>        unsigned int nr_diff = busiest->sum_nr_running;<br><br>        env->migration_type = migrate_task;<br><br>        lsub_positive(&nr_diff, local->sum_nr_running);<br><br>        env->imbalance = nr_diff >> 1;<br><br>    } else {-----------B<br><br>        env->migration_type = migrate_task;<br><br>        env->imbalance = max_t(long, 0, (local->idle_cpus -<br><br>           busiest->idle_cpus) >> 1);<br><br>    }<br><br>    return;<br><br>}|

代码逻辑走到这里，说明busiest group也有空闲算力（local group也一样），这时候主要考虑的是任务的迁移，让sched domain中的idle cpu尽量的均衡。还有一种可能就是busiest group的状态是繁忙（大于fully busy），但是是在MC domain中进行均衡，这时候均衡的逻辑也是一样的看idle cpu。

A、对于base domain（group只有一个CPU）情况，我们还是希望任务散布在各个sched group（cpu）上。因此，这时候需要从busiest group中迁移任务，保证迁移之后，local group和busiest group中的任务数量相等。

B、如果group中有多个CPU，那么我们的目标就是让local group和busiest group中的idle cpu的数量相等

上面处理了local group有空闲算力的情况，下面的代码处理local group处于非group_has_spare状态的情况，代码如下：

|   |
|---|
|if (local->group_type \< group_overloaded) {<br><br>    local->avg_load = (local->group_load * SCHED_CAPACITY_SCALE) /<br><br>        local->group_capacity;<br><br>    sds->avg_load = (sds->total_load * SCHED_CAPACITY_SCALE) /<br><br>        sds->total_capacity;<br><br>    if (local->avg_load >= busiest->avg_load) {<br><br>        env->imbalance = 0;<br><br>        return;<br><br>    }<br><br>}|

如果local group没有空闲算力，但是也没有overloaded，可以从busiest group迁移一些负载过来，但是这也许会导致local group进入overloaded状态。因此这里使用了avg_load来进一步确认是否进行负载迁移。具体的判断方法是local group的平均负载是否大于sched domain的平均负载。如果local group和busiest group都overloaded并且走入calculate imbalance，那么早就确认了busiest group的平均负载大于local group的平均负载。当local group或者busiest group都进入（或者即将进入）overloaded状态，这时候采用迁移负载的方式进行均衡，具体代码如下：

|   |
|---|
|env->migration_type = migrate_load;<br><br>env->imbalance = min(<br><br>(busiest->avg_load - sds->avg_load) * busiest->group_capacity,<br><br>(sds->avg_load - local->avg_load) * local->group_capacity<br><br>) / SCHED_CAPACITY_SCALE;|

具体迁移的负载量是综合考虑local group、busiest group和sched domain的平均负载情况，确保迁移负载之后，local group、busiest group向sched domain的平均负载靠拢。

八、如何寻找busiest group中最忙的CPU

find_busiest_queue函数用来寻找busiest group中最繁忙的cpu。代码逻辑比较简单，和buiest group在上面判断的migrate type相关，不同的type使用不同的方法来寻找busiest cpu：

|   |   |
|---|---|
|Migrate type|寻找最忙CPU的方法|
|migrate_load|最忙CPU是（cpu load/cpu capacity）最大的那个CPU|
|migrate_util|最忙CPU是utility最大的那个CPU|
|migrate_task|最忙CPU是任务最多的那个CPU|
|migrate_misfit|最忙CPU是misfit task load最重的那个CPU|

一旦找到最忙的CPU，那么任务迁移的目标和源头都确定了，后续就可以通过detach tasks和attach tasks进行任务迁移了。

九、detach_tasks和attach_tasks

至此，我们已经确定了从src cpu runqueue（即最繁忙的group中最繁忙的cpu）搬移若干load/util/task到dest cpu runqueue。不过无论是load还是util，最后还是要转成任务。detach_tasks就是确定具体从src rq迁移哪些任务，并把这些任务挂入lb_env->tasks链表中。detach_tasks函数第一段的代码逻辑如下：

|   |
|---|
|while (!list_empty(tasks)) {----------A<br><br>    if (env->idle != CPU_NOT_IDLE && env->src_rq->nr_running \<= 1)<br><br>        break;-----------------B<br><br>    p = list_last_entry(tasks, struct task_struct, se.group_node);------C<br><br>    env->loop++;<br><br>    if (env->loop > env->loop_max)<br><br>        break;--------------------------D<br><br>    if (env->loop > env->loop_break) {<br><br>        env->loop_break += sched_nr_migrate_break;<br><br>        env->flags |= LBF_NEED_BREAK;<br><br>        break;------------------------E<br><br>    }<br><br>    if (!can_migrate_task(p, env))-----------F<br><br>        goto next;<br><br>......<br><br>next:<br><br>    list_move(&p->se.group_node, tasks);<br><br>}|

A、src rq的cfs_tasks链表就是该队列上的全部cfs任务，detach_tasks函数的主要逻辑就是遍历这个cfs_tasks链表，找到最适合迁移到目标cpu rq的任务，并挂入lb_env->tasks链表

B、在idle balance的时候，没有必要把src上的唯一的task拉取到本cpu上，否则的话任务可能会在两个CPU上来回拉扯。

C、从cfs_tasks链表队尾摘下一个任务。这个链表的头部是最近访问的任务。从尾部摘任务可以保证任务是cache cold的。

D、当把dest rq上的任务都遍历过之后，或者当达到循环上限（sysctl_sched_nr_migrate）的时候退出循环。

E、当dest rq上的任务数比较多的时候，并且需要迁移大量的任务才能完成均衡，为了减少关中断的区间，迁移需要分段进行（每sched_nr_migrate_break暂停一下），把大的临界区分成几个小的临界区，确保系统的延迟性能。

F、如果该任务不适合迁移，那么将其移到cfs_tasks链表头部。

上面对从cfs_tasks链表摘下的任务进行基本的判断，具体迁移该任务是否能达到均衡是由detach_tasks函数第二段代码逻辑完成的，具体如下：

|   |
|---|
|switch (env->migration_type) {<br><br>case migrate_load:<br><br>    load = max_t(unsigned long, task_h_load(p), 1);-------A<br><br>    if (sched_feat(LB_MIN) &&<br><br>        load \< 16 && !env->sd->nr_balance_failed)<br><br>        goto next;-------------------B<br><br>    if (shr_bound(load, env->sd->nr_balance_failed) > env->imbalance)<br><br>        goto next;<br><br>    env->imbalance -= load;-----------------------------C<br><br>    break;<br><br>case migrate_util:<br><br>    util = task_util_est(p);<br><br>    if (util > env->imbalance)<br><br>        goto next;<br><br>    env->imbalance -= util;------------------D<br><br>    break;<br><br>case migrate_task:<br><br>    env->imbalance--;------------E<br><br>    break;<br><br>case migrate_misfit:<br><br>    if (task_fits_capacity(p, capacity_of(env->src_cpu)))<br><br>        goto next;<br><br>    env->imbalance = 0;----------F<br><br>    break;<br><br>}|

A、计算该任务的负载。这里设定任务的最小负载是1。

B、LB_MIN特性限制迁移小任务，如果LB_MIN等于true，那么task load小于16的任务将不参与负载均衡。目前LB_MIN系统缺省设置为false。

C、不要迁移过多的load，确保迁移的load不大于env->imbalance。随着迁移错误次增加，这个限制可以适当放宽一些。

D、对于migrate_util类型的迁移，我们通过任务的util和env->imbalance来判断是否迁移了足够的utility。需要注意的是这里使用了任务的estimation utilization。

E、migrate_task类型的迁移不关注load或者utility，只关心迁移的任务数

F、找到misfit task即完成迁移

detach_tasks函数最后一段的代码逻辑如下：

|   |
|---|
|detach_task(p, env);---------------------------------A<br><br>list_add(&p->se.group_node, &env->tasks);<br><br>detached++;<br><br>#ifdef CONFIG_PREEMPTION<br><br>if (env->idle == CPU_NEWLY_IDLE)-----B<br><br>    break;<br><br>#endif<br><br>if (env->imbalance \<= 0)----------C<br><br>    break;<br><br>continue;|

A、程序执行至此，说明任务P需要被迁移（不能迁移的都跳转到next符号了），因此需要从src rq上摘下，挂入env->tasks链表

B、New idle balance是调度延迟的主要来源，所有对于这种balance，我们一次只迁移一个任务

C、如果完成迁移，那么就退出遍历src rq的cfs task链表。

attach_tasks主要的逻辑就是遍历均衡上下文的tasks链表，摘下一个个的任务，挂入目标cpu的队列。

十、如何判断一个任务是否可以迁移至目标CPU

can_migrate_task函数用来判断一个任务是否可以迁移至目标CPU，具体代码逻辑如下：

|   |
|---|
|if (throttled_lb_pair(task_group(p), env->src_cpu, env->dst_cpu))<br><br>    return 0;----------------------------A<br><br>if ((p->flags & PF_KTHREAD) && kthread_is_per_cpu(p))<br><br>    return 0;----------------------------B<br><br>if (!cpumask_test_cpu(env->dst_cpu, p->cpus_ptr)) {<br><br>    int cpu;<br><br>    env->flags |= LBF_SOME_PINNED;------C<br><br>    if (env->idle == CPU_NEWLY_IDLE | (env->flags & LBF_DST_PINNED))<br><br>        return 0;-----------------D<br><br>    for_each_cpu_and(cpu, env->dst_grpmask, env->cpus) {<br><br>        if (cpumask_test_cpu(cpu, p->cpus_ptr)) {<br><br>            env->flags |= LBF_DST_PINNED;<br><br>            env->new_dst_cpu = cpu;<br><br>            break;------------------E<br><br>        }<br><br>    }<br><br>    return 0;<br><br>}|

A、如果任务p所在的task group在src或者dest cpu上被限流了，那么不能迁移该任务，否者限流的逻辑会有问题

B、Percpu的内核线程不能迁移

C、任务由于affinity的原因不能在dest cpu上运行，因此这里设置上LBF_SOME_PINNED标志，表示至少有一个任务由于affinity无法迁移

D、下面的逻辑（E段）会设备备选目标CPU，如果是已经设定好了备选CPU那么直接返回，如果是new idle balance那么也不需要备选CPU，它的主要目标就是迁移一个任务到本idle的cpu。

E、设定备选CPU，以便后续第二轮的均衡可以把任务迁移到备选CPU上

can_migrate_task函数第二段代码逻辑如下：

|   |
|---|
|env->flags &= ~LBF_ALL_PINNED;--------A<br><br>if (task_running(env->src_rq, p))<br><br>    return 0;---------------------------------B<br><br>tsk_cache_hot = task_hot(p, env);-------C<br><br>if (tsk_cache_hot \<= 0 |<br><br>    env->sd->nr_balance_failed > env->sd->cache_nice_tries) {<br><br>    return 1;-------------------------------D<br><br>}|

A、至少有一个任务是可以运行在dest cpu上（从affinity角度），因此清除all pinned标记

B、正处于运行状态的任务不参与迁移，迁移running task是后续active migration的逻辑。

C、判断该任务是否是cache-hot的，这主要从近期在src cpu上的执行时间点来判断，如果上次任务在src cpu上开始执行的时间比较久远（sysctl_sched_migration_cost是门限，目前设定0.5ms），那么其在cache中的内容大概率是被刷掉了，可以认为是cache-cold的。此外如果任务p是src cpu上的next buddy或者last buddy，那么任务是cache hot的。

D、一般而言，我们只迁移cache cold的任务。但是如果进行了太多轮的尝试仍然未能让负载达到均衡，那么cache hot的任务也一样迁移。

参考文献：

1、内核源代码

2、linux-5.10.61\\Documentation\\scheduler\*

本文首发在“内核工匠”微信公众号，欢迎扫描以下二维码关注公众号获取最新Linux技术分享：

![](http://www.wowotech.net/content/uploadfile/202111/b9c21636066902.png)

标签: [load_balance](http://www.wowotech.net/tag/load_balance)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [PELT算法浅析](http://www.wowotech.net/process_management/pelt.html) | [CPU 多核指令 —— WFE 原理](http://www.wowotech.net/armv8a_arch/499.html)»

**评论：**

**Ren Zhijie**\
2022-11-17 15:27

E、将detach_tasks函数摘下的任务挂入到src rq上去。由于detach_tasks、attach_tasks会进行多轮，ld_moved记录了总共迁移的任务数量，cur_ld_moved是本轮迁移的任务数

将detach_tasks函数摘下的任务挂入到*dst_rq*上去。

[回复](http://www.wowotech.net/process_management/load_balance_function.html#comment-8700)

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

  - ja\
    [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
  - 元神高手\
    [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
  - 十七\
    [内核空间的映射在系统启动时就已经设定好，并且在所有进程的页表...](http://www.wowotech.net/process_management/context-switch-arch.html#8920)
  - lw\
    [sparse模型和disconti模型没看出来有什么本质区别...](http://www.wowotech.net/memory_management/memory_model.html#8919)
  - 肥饶\
    [一个没设置好就出错](http://www.wowotech.net/linux_kenrel/516.html#8918)
  - orange\
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

  - [内存初始化代码分析（二）：内存布局](http://www.wowotech.net/memory_management/memory-layout.html)
  - [Linux PM QoS framework(3)\_per-device PM QoS](http://www.wowotech.net/pm_subsystem/per_device_pm_qos.html)
  - [process credentials](http://www.wowotech.net/process_management/19.html)
  - [X-023-KERNEL-Linux pinctrl driver的移植](http://www.wowotech.net/x_project/kernel_pinctrl_driver_porting.html)
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

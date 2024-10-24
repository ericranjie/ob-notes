
作者：[OPPO内核团队](http://www.wowotech.net/author/538) 发布于：2021-12-31 7:00 分类：[进程管理](http://www.wowotech.net/sort/process_management)

# 前言

我们描述CFS任务负载均衡的系列文章一共三篇，第一篇是框架部分，第二篇描述了task placement的逻辑过程，第三篇是负载均衡的情景分析，包括tick balance、nohz idle balance和new idle balance。之前已经有一篇关于task placement的文档发表在本站，为了更精细的讲解代码逻辑，我们这次增加了代码分析部分。本文作为第二篇任务放置的附篇，深入讲解task placement的代码流程。

本文出现的内核代码来自Linux5.10.61，为了减少篇幅，我们对引用的代码进行了删减（例如去掉了NUMA的代码，毕竟手机平台上我们暂时不关注这个特性），如果有兴趣，读者可以配合完整的源代码代码阅读本文。

# 一、能量模型框架及其相关数据结构

## 1、概述

在嵌入式平台上，为了控制功耗，很多硬件模块被设计成可以运行在多种频率下工作（配合对应的电压，形成不同的performance level），这种硬件的驱动模块可以感知到在不同performance level下的能耗。系统中的某些模块可能会希望能感知到硬件能耗从而做出不同的判决。能量模型框架（Energy Model (EM) framework）是一个通用的接口模块，该接口连接了支持不同perf level的驱动模块和系统中的其他想要感知能量消耗的模块。

一个典型的例子就是CPU调度器和CPU驱动模块，调度器希望能够感知底层CPU能量的消耗，从而做出更优的选核策略。对于CPU设备，各个cluster有各自独立的调频机制，cluster内的CPU统一工作在一个频率下。因此每个cluster就会形成一个性能域（performance domain）。调度器通过EM framework接口可以获取CPU在各个performance level的能量消耗。在了解了能量模型的基本概念之后，我们一起来看看和调度器相关的EM数据结构。

## 2、struct root_domain

最初引入root domain的概念是因为rt调度的问题。对于cfs task，我们只要保证每一个cpu runqueue上的公平就OK了（load balancer也会尽量cpu之间的公平），不需要严格保证全局公平。但是对于rt task就不行了，我们必须保证从全局范围看，优先级最高的任务最优先得到调度。然而这样会引入扩展性问题：随着cpu core数据增大，维持全局的rt调度策略有些困难，这样的情况下，我们把CPU分成一个个的区域，每一个区域对应一个root domain。对于rt调度，我们不需要维持全局，只要保证一个root domain中，最高优先级的rt任务最优先得到调度即可。当然，后来更多的成员增加到这个数据结构中（例如本文要描述的performance domain），丰富了它的含义。在手机平台上，我们只有一个root domain，有点类似整个系统的味道了。本文只描述和任务放置相关的成员，具体如下：

|   |   |
|---|---|
|成员|描述|
|int overload|该root domain是否处于overload状态|
|int overutilized|该root domain是否处于overutilized状态|
|unsigned long max_cpu_capacity;|该root domain内CPU的最大算力|
|struct perf_domain \_\_rcu \*pd|该root domain的perf domain链表。通过cpu的runqueue我们可以获取该cpu对应的root domain，从而可以获取到整个root domain的性能域链表。|

这里需要澄清一下overload和overutilized这两个概念，对于一个CPU而言，其处于overload状态则说明其runqueue上有大于等于2个任务，或者虽然只有一个任务，但是是misfit task。对于一个CPU而言，其处于overutilized状态说明：该cpu的utility超过其capacity（缺省预留20%的算力，另外，这里的capacity是用于cfs任务的算力）。在单个cpu overload/overutilized基础上，我们定义root domain（即整个系统）的overload和overutilized。对于root domain，overload表示至少有一个cpu处于overload状态。overutilized表示至少有一个cpu处于overutilized状态。overutilized状态非常重要，它决定了调度器是否启用EAS，只有在系统没有overutilized的情况下，EAS才会生效。Overload和newidle balance的频次控制相关，当系统在overload的情况下，newidle balance才会启动进行均衡。

在CPU拓扑初始化的时候，通过build_perf_domains函数创建各个perf domain，形成root domain的perf domain链表。

## 3、struct perf_domain

Struct perf_comain表示一个CPU性能域，perf_comain和cpufreq policy是一一对应的，对于一个4+3+1结构的平台，每一个性能域都是由perf_domain抽象，因此系统共计3个perf domain，形成链表，链表头在root domain。Perf_domain的各个成员如下：

|   |   |
|---|---|
|成员|描述|
|struct em_perf_domain \*em_pd|EM performance domain|
|struct perf_domain \*next|系统中的perf domain会形成链表，这里指向下一个perf domain|
|struct rcu_head rcu|保护perf domain list的rcu|

## 4、struct em_perf_domain

在EM framework中，我们使用em_perf_domain来抽象一个performance domain：

|   |   |
|---|---|
|成员|描述|
|struct em_perf_state \*table|perf states表，记录了各个performance level的频率、能耗信息。|
|int nr_perf_states|该perf domain支持多少个perf states，即上面perf states表格的条目数|
|unsigned long cpus\[\]|该性能域包括哪些cpu？|

## 5、struct em_perf_state

每个性能域都有若干个perf level，每一个perf level对应能耗是不同的，我们用struct em_perf_state来表示一个perf level下的能耗信息：

|   |   |
|---|---|
|成员|描述|
|unsigned long frequency|该perf level对应的运行频率，单位是KHz|
|unsigned long power|该perf level对应的功率|
|unsigned long cost|为了方便能量运算的一个中间参数，等于power * max_frequency / frequency，下一章会详细解释|

# 二、能量计算概述

我们都知道一个基本的能量计算公式，如下：

|   |
|---|
|能量 = 功率 x 时间|

对于CPU而言，我们要计算其能量需要进一步细化公式（省略了CPU处于idle状态的能耗）：

|   |
|---|
|CPU的能量消耗 = CPU运行频率对应的功率 x  CPU在该频点运行时间|

在内核中，EM中记录了CPU各个频点的功率，而运行时间是通过cpu utility来呈现的。有一个不太方便的地方就是CPU utility是归一化到1024的一个值，失去了在某个频点的运行时间长度的信息，不过没有关系，我们可以转换：

CPU在该频点运行时间 =  cpu utility / cpu current capacity

CPU在某个perf state的算力如下：

![[Pasted image 20241024202029.png]]

不考虑idle state的功耗，Cpu在某个perf state的能量估计如下：

![[Pasted image 20241024202038.png]]

把公式（1）带入公式（2）可以得到：

![[Pasted image 20241024202046.png]]

上面公式的第一项是一个常量，保存在em_perf_state的cost成员中。由于每个perf domain中的微架构都是一样的，因此scale_cpu是一样的，那么cost也是一样的，通过提取公因式我们可以得到整个perf domain的能耗公式：

![[Pasted image 20241024202057.png]]

# 三、Task placement概述

## 1、唤醒场景的Task placement

一个线程进入阻塞状态后，异步事件或者其他线程会调用try_to_wake_up函数唤醒该线程，这时候会引发一次唤醒场景的task placement，在手机平台上任务放置大概的选核思路如下：

（1）如果使能了EAS，那么优先采用EAS选核。当然，只有在轻载（系统没有overutilized）才会启用EAS，重载下（只要有一个cpu处于over utilized状态）还是使用传统内核算法

（2）Wake affine场景下的Non-EAS选核。所谓Wake affine就是选择靠近waker所在的cpu（具体靠近的含义是waker所在CPU的LLC domain范围），当然也有可能靠近prev cpu？在this cpu和prev cpu中选定一个目标CPU之后，走快速路径，在目标cpu的LLC domain选择一个空闲的CPU。

（3）Non-wake affine场景下的Non-EAS选核。走快速路径，在prev cpu的LLC domain选择一个空闲的CPU。

一般而言Non-wake affine场景下的Non-EAS选核是要走慢速路径的，但是在手机平台，所有的各个level的sched domain都没有设定SD_BALANCE_WAKE的标记，因此我们无法确定wakeup均衡的范围，因此走快速路径。

## 2、Fork和exec场景的Task placement

对fork后新任务的唤醒，内核不是用try_to_wake_up来处理，而是用了wake_up_new_task。这个函数会调用fork类型的选核函数完成任务放置。当新任务调用exec类函数开启自己新的旅程之后，内核会调用sched_exec启动一次exec类型的placement。这两种类型的placement选核思路类似：

（1）对于fork场景，找到最高level且支持SD_BALANCE_FORK的sched domain（即找到fork均衡的范围），走慢速路径来进行选核

（2）对于exec场景，找到最高level且支持SD_BALANCE_EXEC的sched domain（即找到exec均衡的范围），走慢速路径来进行选核

# 四、select_task_rq_fair函数代码分析

无论哪种类型的task placement，最终都是调用select_task_rq_fair函数完成逻辑，只是传递参数不同。该函数的逻辑分成三个段落：第一段是EAS的bypass路径，第二段是选择sched domain的逻辑（快速路径的的sched domain不需要选择，固定为为LLC domain），第三段是CPU的选择。我们先看第一段逻辑，它是唤醒类型的EAS处理，如下：

|   |
|---|
|if (sd_flag & SD_BALANCE_WAKE) {-----------A<br><br>    record_wakee(p);----------B<br><br>    if (sched_energy_enabled()) {-----------C<br><br>        new_cpu = find_energy_efficient_cpu(p, prev_cpu);<br><br>        if (new_cpu >= 0)<br><br>            return new_cpu;---------D<br><br>        new_cpu = prev_cpu;<br><br>    }<br><br>    want_affine = !wake_wide(p) && cpumask_test_cpu(cpu, p->cpus_ptr);----E<br><br>}|

A、对唤醒路径（通过try_to_wake_up进入该函数）进行特别的处理，主要包括两部分内容，一是对wake affine的处理（B和E），另外一个是EAS的选核。由此可见，EAS只用于wakeup，fork和exec均衡固定走传统选核算法。

B、更新current task（waker）的wakee信息，关于wake affine后面会详细描述。

C、find_energy_efficient_cpu是EAS的主选核路径，使用EAS选核需要满足两个条件：唤醒路径且enable了energy aware scheduler特性。具体EAS的选核逻辑后面会详细描述

D、EAS选中了适合的CPU就直接返回。如果EAS选核不成功，那么恢复缺省cpu为prev cpu，走传统选核路径。

E、判断本次唤醒是否想要wake affine（即wakee task靠近waker所在的cpu）。

如果EAS选中了适合的CPU，那么就直接返回了，不需要进行sched domain/cpu的选择了。如果不需要EAS选核，或者EAS没有选择到合适的CPU，那么就需要走传统的快速或者慢速路径的选核了，具体走那条路主要是第二段代码逻辑中是否找到了适合的sched domain。第二段寻找sched domain的代码如下：

|   |
|---|
|for_each_domain(cpu, tmp) {---------A<br><br>    if (want_affine && (tmp->flags & SD_WAKE_AFFINE) &&<br><br>      cpumask_test_cpu(prev_cpu, sched_domain_span(tmp))) {----------B<br><br>        if (cpu != prev_cpu)<br><br>            new_cpu = wake_affine(tmp, p, cpu, prev_cpu, sync);<br><br>        sd = NULL; /\* Prefer wake_affine over balance flags \*/<br><br>        break;<br><br>    }<br><br>    if (tmp->flags & sd_flag)-------C<br><br>        sd = tmp;<br><br>    else if (!want_affine)<br><br>        break;<br><br>}|

A、从当前cpu（waker所在的cpu）所在的base sched domain，自下而上，寻找合适的sched domain，从而确定任务放置的范围。

B、对于wake affine场景，我们需要自下而上遍历sd找到一个符合要求的sched domain，它必须要满足：（1）该sched domain支持wake affine，（2）该sched domain包括了prev cpu和当前cpu（即waker所在的cpu）。找到这样的sched domain之后，我们需要调用wake_affine快速的在当前cpu和prev cpu中选中其一（如果当前cpu就是prev cpu，那么不需要通过wake_affine选择了，直接prev cpu即可），作为后续选核快速路径的输入（sd==NULL确保走快速路径）。

C、对于non wake affine场景，我们一样也是需要自下而上遍历sd找到最顶层且支持该sd flag的sched domain。目前手机平台上，DIE domain和MC domain都没有设定SD_BALANCE_WAKE，因此在唤醒场景，我们无法找到sched domain，从而走入快速路径。对于SD_BALANCE_FORK和SD_BALANCE_EXEC场景，DIE domain和MC domain都设置了对应的flag，因此遍历搜索到了DIE domain，后续会走入慢速路径。

至此，我们应该选中了一个sched domain，给后续的placement定义了一个范围。当然，也有可能sd等于NULL，表示是快速路径，直接从LLC doamin选核。第三段代码逻辑如下：

|   |
|---|
|if (unlikely(sd)) {/\* Slow path */--------A<br><br>    new_cpu = find_idlest_cpu(sd, p, cpu, prev_cpu, sd_flag);<br><br>} else if (sd_flag & SD_BALANCE_WAKE) {/* Fast path \*/ ---------B<br><br>    new_cpu = select_idle_sibling(p, prev_cpu, new_cpu);<br><br>    if (want_affine)<br><br>        current->recent_used_cpu = cpu;<br><br>}|

A、慢速选核路径，需要在选定的sched domain中寻找最空闲的group，然后在最空闲的group中选择最空闲的cpu。然后继续在下层sched domain重复上面的过程，最终选择一个空闲CPU并把任务p放置在这个最空闲的CPU上。

B、快速选核路径，主要是在LLC domain中选择距离new_cpu比较近的空闲CPU。对于wake affine，这里的new_cpu就是wake_affine选定的target cpu，对于其他场景new_cpu就是prev cpu。

# 五、EAS选核

find_energy_efficient_cpu函数用来为被唤醒的任务找到一个能效比最高的目标CPU。主要搜索的步骤如下：

1、在每一个cluster（performance domain）中找到空闲算力最多的cpu作为备选

2、从备选cpu中通过能量模型找到消耗能量最小的cpu。

这个算法思路也比较简单，在一个cluster中，选择一个空闲算力最大cpu放置任务会让schedutil governor请求较低的CPU频率，从而消耗较低的能量。在实际中，从能量的角度看，把小任务挤压到一个CPU core上可以有助于让其他CPU core进入更深的idle状态，但是这样会让cluster进入idle的机会大大降低。因此，实际上从能量模型上，我们并不能说明这样的小任务挤压的策略是好的还是坏的。因此，find_energy_efficient_cpu函数还是选择把任务挤压在一个cluster内，然后在选中的cluster之中，尽量把任务散布到各个cpu上。这样的策略对延迟而言肯定是好事情。这种策略也和“只有异构系统中的EAS才能带来的能量节省”是一致的。这也是为什么我们只在SD_ASYM_CPUCAPACITY的系统中打开EAS的原因。

在fork场景，被fork出来的任务到底放置在哪个CPU并不采用EAS算法，因为它不知道自己的util数据，也不太能够预测出能量的消耗。因此，对fork出来的任务仍然使用慢速路径（find_idlest_cpu函数）找到最空闲的group/cpu并将其放置在该cpu上执行，在有些用户场景下，这种方法可能是恰好命中能效比最高的CPU，但未必总是如此。另一种可能的方法是先将新fork出来的任务放置在特定类型的cpu（例如小核），或者尝试从父任务推断它们的util_avg，但是这种placement策略也可能损害其他用户场景。在没有找到其他更好的方法之前，我们仍然采用传统的慢速路径的方式来选核。Exec场景和fork场景类似，不再赘述。

Overutilized主要用来切换EAS调度器还是传统调度器。如果系统处于overutilized的状态，那么不再执行EAS的task placement的逻辑，如下：

|   |
|---|
|struct root_domain \*rd = cpu_rq(smp_processor_id())->rd;<br><br>rcu_read_lock();<br><br>pd = rcu_dereference(rd->pd);<br><br>if (!pd | READ_ONCE(rd->overutilized))<br><br>    goto fail;|

EAS的选核逻辑是先确定一个sched domain，然后从sched domain中选择一个能效比高的CPU，具体选择sched domain的方式如下：

|   |
|---|
|sd = rcu_dereference(\*this_cpu_ptr(&sd_asym_cpucapacity));<br><br>while (sd && !cpumask_test_cpu(prev_cpu, sched_domain_span(sd)))<br><br>    sd = sd->parent;|

从最低level的，并且包括不同算力CPU的domain开始向上搜索，直到该domain覆盖了this cpu和prev cpu为止。对于手机平台一般找到的就是Die domain。之所以要求包括异构CPU的domain是因为同构的cpu不需要EAS，不会有功耗的节省。

因为后面需要任务p的utility参与计算（估计调频点和能耗），而该任务在被唤醒之前应该已经阻塞了一段时间，它的utility还停留在阻塞前那一点，因此这里需要进行负载更新：

|   |
|---|
|sync_entity_load_avg(&p->se);<br><br>if (!task_util_est(p))<br><br>    goto unlock;|

sync_entity_load_avg函数用来更新该任务的sched avg，由于任务p从睡眠中被唤醒，因此同步也就是衰减其负载。具体是衰减到其挂入的cfs rq的最近的负载更新时间点（而不是当前时间点）。注意：这里的cfs rq是任务p阻塞之前挂入的队列。如果任务的utility等于0那么通过能量模型对其进行运算是没有意义的，所以直接退出。

EAS最核心的代码如下：

|   |
|---|
|for (; pd; pd = pd->next) {<br><br>    base_energy_pd = compute_energy(p, -1, pd);-----A<br><br>    base_energy += base_energy_pd;<br><br>    for_each_cpu_and(cpu, perf_domain_span(pd), sched_domain_span(sd)) {<br><br>        util = cpu_util_next(cpu, p, cpu);-------------B<br><br>        cpu_cap = capacity_of(cpu);<br><br>        spare_cap = cpu_cap;<br><br>        lsub_positive(&spare_cap, util);<br><br>        util = uclamp_rq_util_with(cpu_rq(cpu), util, p);<br><br>        if (!fits_capacity(util, cpu_cap))---------------C<br><br>            continue;<br><br>        if (cpu == prev_cpu) {----------------------D<br><br>            prev_delta = compute_energy(p, prev_cpu, pd);<br><br>            prev_delta -= base_energy_pd;<br><br>            best_delta = min(best_delta, prev_delta);<br><br>        }<br><br>        if (spare_cap > max_spare_cap) {--------E<br><br>            max_spare_cap = spare_cap;<br><br>            max_spare_cap_cpu = cpu;<br><br>        }<br><br>    }<br><br>    if (max_spare_cap_cpu >= 0 && max_spare_cap_cpu != prev_cpu) {------F<br><br>        cur_delta = compute_energy(p, max_spare_cap_cpu, pd);<br><br>        cur_delta -= base_energy_pd;<br><br>        if (cur_delta \< best_delta) {<br><br>            best_delta = cur_delta;<br><br>            best_energy_cpu = max_spare_cap_cpu;<br><br>        }<br><br>    }<br><br>}|

A、计算本performance domain的基础能量消耗，即未放置任务p的能耗（base_energy_pd）。同时这里会累计各个pd的能耗（base_energy），也就是整个cpu系统消耗的能量。

B、遍历该pd的各个cpu，计算各个cpu的剩余算力。这里的剩余算力是这样计算的：获取该cpu用于cfs task的算力（即原始算力去掉rt irq等消耗的算力），减去该cpu的util（即cfs task的总的utility）。这里的cpu util是把任务p放置在该cpu的预估utility（cpu_util_next）。

C、本身EAS就是在系统没有overutilized的情况下才启用的，如果任务p放置该CPU导致该cpu进入overutilized，那么就跳过该CPU。当然，这里的cpu util是经过uclamp修正之后的util。上面步骤B中是估计cpu的空闲算力，需要真实的util，所以没有经过uclamp修正。

D、Prev cpu永远是备选对象，因此这里计算任务p放置在prev cpu上带来的能耗增量，并且参与到能耗最优的对比中。

E、寻找该perf domain中，空闲算力最大的那个CPU，这个cpu也是备选对象之一。

F、对比各个perf domain中选择的最佳CPU（剩余空闲算力最大）的能耗，选择一个最小的CPU，赋值给best_energy_cpu。

在EAS选核最后，选出的best_energy_cpu还要和prev cpu进行PK，毕竟prev cpu有可能带来潜在的性能收益（提高cache命中率），只有当best_energy_cpu相对prev cpu能耗收益大于CPU总耗能的1/16，才选择best_energy_cpu，否则还是运行在prev cpu上。

# 六、能量计算

compute_energy函数原型如下：

|   |
|---|
|static long compute_energy(struct task_struct \*p, int dst_cpu, struct perf_domain \*pd)|

该函数主要用来计算当任务p放置到dst_cpu的时候，pd所在的perf domain会消耗多少能量。任务p放置到dst cpu的时候，这会影响pd所在的perf domain中各个CPU的utility，进而会驱动cpufreq进行频率调整。计算能量的时候，我们必须预测可能的调频，并通过这些utility可以估计该perf domain的能耗（任务放置在dst cpu之后）。详细的代码逻辑如下：

|   |
|---|
|for_each_cpu_and(cpu, pd_mask, cpu_online_mask) {<br><br>    unsigned long cpu_util, util_cfs = cpu_util_next(cpu, p, dst_cpu);---------A<br><br>    struct task_struct \*tsk = cpu == dst_cpu ? p : NULL;<br><br>    sum_util += schedutil_cpu_util(cpu, util_cfs, cpu_cap,<br><br>       ENERGY_UTIL, NULL);--------------B<br><br>    cpu_util = schedutil_cpu_util(cpu, util_cfs, cpu_cap,<br><br>      FREQUENCY_UTIL, tsk);-----------C<br><br>    max_util = max(max_util, cpu_util);---------------D<br><br>}<br><br>return em_cpu_energy(pd->em_pd, max_util, sum_util);-----E|

整个能量计算分成两个部分，首先遍历perf domain上的cpu，累计整个perf domain中的cpu utility总和。此外，还要记录最大cpu utility（推测运行频率）。完成sum_util和max_util之后调用em_cpu_energy即可。

A、遍历perf domain上的cpu，通过cpu_util_next计算任务p放置在dst cpu之后，各个cpu上的cfs任务的utility

B、在步骤A中仅仅得到了cfs任务的utility，通过schedutil_cpu_util得到该cpu的utility（累计rt、dl、irq等的util）并累计起来。这里的util是用来计算能量的，因此返回的util要等于真实CPU busy time，不需要uclamp的处理。遍历perf domain上的所有CPU就得到其上的util总和，用于能量计算。

C、计算该CPU的调频util。这里是调频需要的util，需要进行utility clamp处理。

D、找到该perf domain中最大的cpu util。

E、调用em_cpu_energy函数进行该perf domain的耗能估计。Max_util用来估计该perf domain潜在运行的频点，sum_util表示该perf domain的总运行时间。具体的运算非常简单，在第三章已经给出，不再赘述。

七、Wake affine

task_struct中有三个成员用来记录被唤醒线程（wakee）的信息，如下：

|   |   |
|---|---|
|成员|描述|
|unsigned int wakee_flips|该线程唤醒不同wakee的次数。wakee_flips数值较大，则说明该该线程唤醒多个不同的任务。并且数值越大，说明唤醒的频率越快|
|unsigned long wakee_flip_decay_ts|wakee_flips会随时间衰减，这里定义了上次进行衰减的时间点|
|struct task_struct \*last_wakee|该线程上次唤醒的线程|

每次waker唤醒wakee的时候都会调用record_wakee来更新上面的成员，具体代码如下：

|   |
|---|
|if (time_after(jiffies, current->wakee_flip_decay_ts + HZ)) {-------A<br><br>    current->wakee_flips >>= 1;<br><br>    current->wakee_flip_decay_ts = jiffies;<br><br>}<br><br>if (current->last_wakee != p) {------B<br><br>    current->last_wakee = p;<br><br>    current->wakee_flips++;<br><br>}|

A、每隔一秒对wakee_flips进行衰减。如果一个线程能够经常的唤醒不同的其他线程，那么该线程的wakee_flips会保持在一个较高的值。相反，如果仅仅是偶尔唤醒一次其他线程，和某个固定的线程有唤醒关系，那么这里的wakee_flips应该会趋向0

B、如果上次唤醒的不是p，那么要切换wakee，并累加wakee翻转次数

Waker唤醒wakee的场景中，有两种placement思路：一种是聚合的思路，即让waker和wakee尽量的close，从而提高cache hit。另外一种思考是分散，即让load尽量平均分配在多个cpu上。不同的唤醒模型使用不同的放置策略。我们来看看下面两种简单的唤醒模型：

（1）在1：N模型中，一个server会不断的唤醒多个不同的client

（2）1：1模型，线程A和线程B不断的唤醒对方

在1：N模型中，如果N是一个较大的数值，那么让waker和wakee尽量的close会导致负荷的极度不平均，这会让waker所在的sched domain会承担太多的task，从而引起性能下降。在1：1模型中，让waker和wakee尽量的close不存在这样的问题，同时还能提高性能。但是，实际的程序中，唤醒关系可能没有那么简单，一个wakee可能是另外一个关系中的waker，交互可能是M：N的形式。考虑这样一个场景：waker把wakee拉近，而wakee自身的wakee flips比较大，那么更多的线程也会拉近waker所在的sched domain，从而进一步加剧CPU资源的竞争。因此waker和wakee的wakee flips的数值都不能太大，太大的时候应该禁止wake affine。内核中通过wake_wide来判断是否使能wake affine：

|   |
|---|
|unsigned int master = current->wakee_flips;---------A<br><br>unsigned int slave = p->wakee_flips;<br><br>int factor = \_\_this_cpu_read(sd_llc_size);-------------B<br><br>if (master \< slave)<br><br>    swap(master, slave);-------------C<br><br>if (slave \< factor | master \< slave * factor)--------D<br><br>    return 0;------waker和wakee需要靠近<br><br>return 1;|

A、这里的场景是current task唤醒任务p的场景，master是current唤醒不同线程的次数，slave是被唤醒的任务p唤醒不同线程的次数。

B、Wake affine场景下任务放置要走快速路径，即在LLC上选择空闲的CPU。sd_llc_size是LLC domain上CPU的个数。Wake affine本质上是把wakee线程们拉到waker所在的LLC domain，如果超过了LLC domain的cpu个数，那么必然有任务需要等待，这也就失去了wake affine提升性能的初衷。对于手机平台，llc domain是MC domain。

C、一般而言，执行更多唤醒动作（并且唤醒不同task）的任务是master，因此这里根据翻转次数来交换master和slave，确保master的翻转次数大于slave。

D、Slave和master的wakee_flips如果比较小，那么启动wake affine，否则disable wake affine，走正常选核逻辑。这里的or逻辑是存疑的，因为master和slave其一的wakee_flips比较小就会wake affine，这会使得任务太容易在LLC domain堆积了。在1：N模型中（例如手机转屏的时候，一个线程会唤醒非常非常多的线程来处理configChange消息），master的wakee_flips巨大无比，slave的wakee_flips非常小，如果仍然wake affine是不合理的。

如果判断需要进行wake affine，那么我们需要在waking cpu和该任务的prev cpu中选择一个CPU，后续在该CPU的LLC domain上进行wake affine。选择waking cpu还是prev cpu的逻辑是在wake_affine中实现：

|   |
|---|
|int target = nr_cpumask_bits;<br><br>if (sched_feat(WA_IDLE))----------------A<br><br>    target = wake_affine_idle(this_cpu, prev_cpu, sync);<br><br>if (sched_feat(WA_WEIGHT) && target == nr_cpumask_bits)--------------B<br><br>    target = wake_affine_weight(sd, p, this_cpu, prev_cpu, sync);|

A、这里根据this_cpu（即waking CPU）和prev cpu的idle状态来选择，哪个cpu处于idle状态，那么就选择那个cpu。如果都idle，那么优先prev cpu。this cpu有一个特殊情况：在sync wakeup的场景下，this cpu上如果有唯一的running task（也就是waker了），那么优先this cpu，毕竟waker很快就阻塞了。另外，当this cpu处于idle状态的时候（中断唤醒），我们也不能盲目选择它，毕竟在中断大量下发的情况下，有些平台会把硬件中断信号只送到this cpu所在的节点，这会导致 this cpu所在的节点非常繁忙。

B、当根据idle状态无法完成选核的时候，我们会比拼this cpu和prev cpu上的负载。当然，在计算this cpu和prev cpu负载的时候要考虑任务p的迁移。当this cpu的负载小于prev cpu的负载的时候选择this cpu。否则，选择prev cpu。

八、快速选核路径

快速选核路径的入口是select_idle_sibling函数，在这个函数的开始是一些快速选择cpu的逻辑，即不需要扫描整个LLC domain的CPU，找idle的cpu，而是直接选择某个具体的cpu，具体如下：

|   |
|---|
|if (static_branch_unlikely(&sched_asym_cpucapacity)) {----A<br><br>    sync_entity_load_avg(&p->se);<br><br>    task_util = uclamp_task_util(p);<br><br>}<br><br>if ((available_idle_cpu(target) | sched_idle_cpu(target)) &&<br><br>    asym_fits_capacity(task_util, target))-------B<br><br>    return target;|

A、在异构系统中，我们后面的代码逻辑需要使用该任务的utility，看看是否能够适合CPU的算力，因此，这里需要进行一个负载更新动作，主要是衰减阻塞这段时间的负载。

B、如果之前选定的target cpu（waker所在的cpu或者任务prev cpu）是idle cpu或者虽然没有idle，但是runqueue中都是SCHED_IDLE的任务，而且该CPU的算力能够承载该被唤醒的任务，这时候，直接返回target cpu即可。Wake affine的场景下，target cpu是通过wake_affine函数寻找的target cpu，其他场景下，target CPU其实等于prev CPU。

第二段代码如下：

|   |
|---|
|if (prev != target && cpus_share_cache(prev, target) &&<br><br>    (available_idle_cpu(prev) | sched_idle_cpu(prev)) &&<br><br>    asym_fits_capacity(task_util, prev))-----------A<br><br>       return prev;<br><br>if (is_per_cpu_kthread(current) &&<br><br>    prev == smp_processor_id() &&<br><br>    this_rq()->nr_running \<= 1) {-----------------B<br><br>      return prev;<br><br>}|

A、如果prev cpu是idle状态（包括runqueue上仅SCHED_IDLE的任务），并且prev cpu算力能承接该任务的utili，同时prev和target cpu在一个LLC domain，那么优选prev cpu

B、当waker是一个per cpu的kthread线程的时候，在wakee的prev cpu也是this cpu的时候，允许把wakee拉到waker所在的cpu，这是为了解决XFS性能问题而引入的提交。这时候kthread很快就会阻塞（类似sync wakeup），wakee也不会在runqueue上等太久。

第三段代码如下：

|   |
|---|
|recent_used_cpu = p->recent_used_cpu;<br><br>if (recent_used_cpu != prev &&<br><br>    recent_used_cpu != target &&<br><br>    cpus_share_cache(recent_used_cpu, target) &&<br><br>    (available_idle_cpu(recent_used_cpu) | sched_idle_cpu(recent_used_cpu)) &&<br><br>    cpumask_test_cpu(p->recent_used_cpu, p->cpus_ptr) &&<br><br>    asym_fits_capacity(task_util, recent_used_cpu)) {<br><br>        p->recent_used_cpu = prev;<br><br>        return recent_used_cpu;<br><br>}|

我们假设有这样的一个场景，任务A和任务B互相唤醒，如果没有这段代码，选核如下：

（1）任务A（在CPU0上）唤醒任务B并将其放置在CPU1

（2）任务B（在CPU1上）唤醒任务A并将其放置在CPU2

（3）任务A（在CPU2上）唤醒任务B并将其放置在CPU3

（4）......

这样的选核让任务A和任务B相互拉扯，均匀的使用LLC domain中的各个CPU。这会导致任务的util平均散布在各个CPU上，从而导致CPU util比较小，处于比较低的频率，影响性能。有了上面的这段代码逻辑，任务A和任务B的唤醒过程如下：

（1）任务A（在CPU0上）唤醒任务B并将其放置在CPU1

（2）任务B（在CPU1上）唤醒任务A并将其放置在CPU0

（3）......

至此，快速选择CPU的场景结束了，后续代码是在LLC domain上选择合适CPU的过程。对于异构平台（例如手机），我们更喜欢idle CPU，这时候我们扩大搜寻范围，从LLC扩大到sd_asym_cpucapacity（DIE domain），代码如下：

|   |
|---|
|if (static_branch_unlikely(&sched_asym_cpucapacity)) {<br><br>    sd = rcu_dereference(per_cpu(sd_asym_cpucapacity, target));<br><br>    if (sd) {<br><br>        i = select_idle_capacity(p, sd, target);<br><br>        return ((unsigned)i \< nr_cpumask_bits) ? i : target;<br><br>   }<br><br>}|

通过select_idle_capacity我们在DIE domain寻找第一个能够承载该任务的idle cpu，如果虽然找到idle cpu，但是cpu capacity不足，这种情况下，我们返回算力最大的CPU即可。如果找不到idle cpu，那么选择之前选择的target cpu。

对于同构平台，我们需要通过扫描llc domain寻找最空闲CPU，代码逻辑如下（删去了SMT代码）：

|   |
|---|
|sd = rcu_dereference(per_cpu(sd_llc, target));<br><br>i = select_idle_cpu(p, sd, target);<br><br>if ((unsigned)i \< nr_cpumask_bits)<br><br>    return i;|

select_idle_cpu逻辑比较简单，就是从LLC domain中找到第一个idle的cpu。为了防止不必要的scan开销，在扫描之前需要对比该cpu rq的平均idle时间和扫描开销，如果扫描开销相对于平均idle时间太大，那么就不再进行扫描。

九、慢速选核路径

1、概览

和快速路径不同，慢速路径在一个指定的sched domain开始，层层递进，不断搜索最空闲的group，然后在这个group中搜索最不忙的CPU，直到base domain。这样的结果就是在各个level的sched domain上都达到了均衡的效果。慢速路径的入口是find_idlest_cpu，其函数原型是：

|   |
|---|
|int find_idlest_cpu(struct sched_domain \*sd, struct task_struct \*p,<br><br>  int cpu, int prev_cpu, int sd_flag)|

该函数的参数以及返回值解释如下：

|   |   |
|---|---|
|参数及返回值|描述|
|Sd|扫描范围，即从这个sched domain覆盖的cpu中选择任务放置的目标cpu|
|P|需要选择目标CPU的那个任务|
|Cpu|当前cpu，即唤醒任务p的CPU|
|Prev_cpu|任务p上次运行的CPU|
|sd_flag|标示任务p的唤醒场景|
|返回值|找到的最适合任务放置的CPU|

我们分两段来解析find_idlest_cpu的逻辑。第一段代码流程如下：

|   |
|---|
|if (!cpumask_intersects(sched_domain_span(sd), p->cpus_ptr))<br><br>    return prev_cpu;---------------A<br><br>if (!(sd_flag & SD_BALANCE_FORK))<br><br>    sync_entity_load_avg(&p->se);-----------B|

A、如果任务p由于affinity的原因无法在sd指定的sched domain中运行，那么直接选择prev_cpu

B、由于后续需要进行cpu util的比拼，不同的选核会导致任务p util从一个cpu移动到另外的cpu，因此需要调用sync_entity_load_avg来更新任务p的负载。当然，新fork出来的任务没有必要更新，使用初始负载即可

第二段代码流程如下：

|   |
|---|
|while (sd) {---------------A<br><br>    group = find_idlest_group(sd, p, cpu);------------B<br><br>    if (!group) {<br><br>        sd = sd->child; continue;<br><br>    }<br><br>    new_cpu = find_idlest_group_cpu(group, p, cpu);--------C<br><br>    if (new_cpu == cpu) {<br><br>        sd = sd->child; continue;<br><br>    }<br><br>    cpu = new_cpu;<br><br>    weight = sd->span_weight;<br><br>    sd = NULL;<br><br>    for_each_domain(cpu, tmp) {---------------D<br><br>        if (weight \<= tmp->span_weight)<br><br>            break;<br><br>        if (tmp->flags & sd_flag)<br><br>            sd = tmp;<br><br>    }<br><br>}|

A、Task placement也是均衡中的一个环节，而均衡是一个层层递进的过程：从指定的sched domain为起点，遍历其child sched domain，直到base domain。

B、首先在参数sd中指定的domain中调用find_idlest_group函数找到最空闲的sched group，如果local group最闲，那么直接进入下一级的child domain继续搜寻。

C、在找到的最空闲的sched group中调用find_idlest_group_cpu函数寻找最空闲的CPU，如果不需要切换CPU，那么进入下一级的child domain继续搜寻。

D、如果需要切换CPU，那么寻找该CPU对应的下一级child domain，继续进行搜寻，直到base domain。

2、寻找最空闲组

find_idlest_group的代码逻辑主要分成两个部分：第一部分是更新各个group上的负载并记录non-local中最空闲的group和local group（总是备选）。第二部分是idlest group和local group的比拼。我们先看第一段代码：

|   |
|---|
|do {<br><br>    int local_group;<br><br>    if (!cpumask_intersects(sched_group_span(group), p->cpus_ptr))<br><br>        continue;------------------A<br><br>    local_group = cpumask_test_cpu(this_cpu, ---------------B<br><br>       sched_group_span(group));<br><br>    if (local_group) {<br><br>        sgs = &local_sgs; local = group;<br><br>    } else sgs = &tmp_sgs;<br><br>    update_sg_wakeup_stats(sd, group, sgs, p);--------C<br><br>    if (!local_group && update_pick_idlest(idlest, &idlest_sgs, group, sgs)) {<br><br>        idlest = group;<br><br>        idlest_sgs = \*sgs;<br><br>    }--------------------------D<br><br>} while (group = group->next, group != sd->groups);|

A、任务至少要能在该group中的一个cpu上运行，否则就不考虑该group

B、判断是否是local group。我们区别对待local group和non-local group，对于non-local group要找到最空闲的group。

C、更新该group上的负载信息。这里和load_balance中更新group负载的概念是一样的，有兴趣的可以看看内核工匠之前发布的load_balance详解的文章。不再赘述。

D、寻找最空闲的idlest group。

至此，我们已经拿到了两个group的统计信息：local group和non-group中idlest group。那么下一级进入那个group对应的domain就需要在这两个group中比拼，有些选择是显而易见的，例如当idlest group不存在或者虽然idlest group存在，但是local group更空闲，这时候选择local group，可以使得该level的sched domain更加均衡。而当local group不存在，或者idlest group比local group更空闲，这时候选择idlest group。比较困难的场景是当idlest group和local group空闲程度差不多（group_type相对），这又分成集中情况：

（1）group_overloaded或者group_fully_busy的情况下，对比group的平均负载，选择平均负载轻的

（2）group_misfit_task情况下，选择算力大的group

（3）group_has_spare情况下，看哪个group中的idle cpu数量多

3、寻找最空闲CPU

find_idlest_group_cpu的基本逻辑过程如下：

（1）如果group内只有一个cpu（base domain），那么直接返回该cpu

（2）如果group内有多个cpu，那么需要遍历进行比拼

（3）如果一个cpu上只有SCHED_IDLE类型的任务，那么直接返回该cpu。

（4）如果有处于idle状态的cpu，那么选择退出延时最小的那个idle cpu（即睡眠最浅的）。如果有多个cpu处于同等深度的idle状态，那么选择最近那个进入idle的cpu（cache的状态会好一些）

（5）如果所有的cpu都忙碌中，那么选择cpu load最小的那个。

参考文献：

1、内核源代码

2、linux-5.10.61/Documentation/scheduler/\*

3、linux-5.10.61/Documentation/power/ energy-model.rst

本文首发在“内核工匠”微信公众号，欢迎扫描以下二维码关注公众号获取最新Linux技术分享：

![](http://www.wowotech.net/content/uploadfile/202111/b9c21636066902.png)

标签: [任务放置](http://www.wowotech.net/tag/%E4%BB%BB%E5%8A%A1%E6%94%BE%E7%BD%AE) [task](http://www.wowotech.net/tag/task) [placement](http://www.wowotech.net/tag/placement)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [CPU 多核指令 —— WFE 原理](http://www.wowotech.net/armv8a_arch/499.html) | [CFS任务的负载均衡（load balance）](http://www.wowotech.net/process_management/load_balance_detail.html)»

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

  - [ARM CPU性能实验](http://www.wowotech.net/arm-performance-test.html)
  - [Linux PM domain framework(1)\_概述和使用流程](http://www.wowotech.net/pm_subsystem/pm_domain_overview.html)
  - [schedutil governor情景分析](http://www.wowotech.net/process_management/schedutil_governor.html)
  - [Linux kernel内核配置解析(1)\_概述(基于ARM64架构)](http://www.wowotech.net/linux_kenrel/kernel_config_overview.html)
  - [以太网驱动的流程浅析(三)-ifconfig的-19错误最底层分析](http://www.wowotech.net/linux_kenrel/465.html)

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

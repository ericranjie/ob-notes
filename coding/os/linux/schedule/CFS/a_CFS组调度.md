Original Ham OPPO内核工匠
_2023年01月06日 17:00_ _广东_

注：本文缩写说明

![Image](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjP7W0s65zd4aJAdJYNYnlGDnUHpbzoVicq4WFVVhAXYXEpRLkyXq17wTmPd9icqwR14icicricH6ooIXTA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

# **一、CFS组调度简介**

**1.1. 存在的原因**

总结来说是希望不同分组的任务在高负载下能分配可控比例的CPU资源。为什么会有这个需求呢，比如多用户计算机系统每个用户的所有任务划分到一个分组中，A用户90个相同任务，而B用户只有10个相同任务，在CPU完全跑满的情况下，那么A用户将占90%的CPU时间，而B用户只占到了10%的CPU时间，这对B用户显然是不公平的。再或者同一个用户，既想-j64快速编译，又不想被编译任务影响使用体验，也可将编译任务设置到对应分组中，限制其CPU资源。

**1.2. 手机设备上的分组状态**

![Image](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjP7W0s65zd4aJAdJYNYnlGDoFNmoSeqiaWS8ZtWSdWiaeeelC5PlJfZDpocW2sV4V14v9UaWEEiaqb6g/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

/dev/cpuctl 目录使用struct task_group root_task_group 表示。其下的每一层级的子目录都抽象为一个task_group结构。有几个需要注意的点：

1. 根组下的cpu.shares 文件默认值是1024，而且不支持设置。根组的负载也不会更新。

1. 根组也是一个task group，根组下的任务也是被分过组的，不会再属于其它组。

1. 内核线程默认在根组下，其直接从根cfs_rq上分配时间片，有非常大的优势，若是其一直跑，那在trace上看就几乎就是”一直跑”，比如常见的kswapd内核线程。

1. 默认cpu.shares配置下，若全是nice=0的任务，且只考虑单核的情况下，根组下的每个任务在完全满载情况下能得到的时间片等于其它分组下所有任务能分配到的时间片的总和。

注意：task和task group都是通过权重来分配时间片的，但是task的权重来自其优先级，而task group的权重则来自与其cgroup目录下cpu.shares文件设置的值。使能组调度后，看任务分得的时间片，就不能单看其prio对应的权重了，还要看其task group分得的权重和本group中其它任务的运行情况。

**二、任务的task group分组**

CFS组调度功能主要是通过对任务进行分组体现出来的，一个分组由一个struct task_group来表示。

**2.1. 如何设置分组**

task group分组配置接口由cpu cgroup子系统通过cgroup目录层次结构导出到用户空间。

![Image](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjP7W0s65zd4aJAdJYNYnlGD9DHRTxjuMBxOVvOCKfyKIQicib5xmp2J8AAIaeFdTFsBG3HsMqv5fWZQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

如何从task group中移除一个任务呢，没有办法直接移除的，在cgroup语义下，一个任务某一时刻必须属于一个task group，只有通过将其echo到其它分组中才能将其从当前分组中移除。

**2.2. Android中如何设置分组**

Process.java中向其它模块提供 setProcessGroup(int pid, int group) 将pid进程设置进group参数指定的分组中，供其它模块进行调用设置。比如OomAdjuster.java 中将任务切前/后台分别调用传参group=THREAD_GROUP_TOP_APP/THREAD_GROUP_BACKGROUND。

libprocessgroup 中提供了一个名为 task_profiles.json 的配置文件，它里面 AggregateProfiles 聚合属性字段配置了上层设置下来后的对应的行为。比如 THREAD_GROUP_TOP_APP 对应的聚合属性为 SCHED_SP_TOP_APP，其中的MaxPerformance属性对应的行为就是加入到cpu top-app分组。

![Image](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjP7W0s65zd4aJAdJYNYnlGDHWI6nxWD5M9D0EulszDFX5EPicdxcnwkt64b48j5MRdWzWc0GzRg8Kg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

”MaxPerformance”属性的配置可读性非常强，可以看出是加入到cpu子系统的top-app分组中。

**2.3. Android中设置为TOP-APP分组，对cgroup设置了什么**

因为有多个cgroup子系统，除了我们正在讲的CFS组调度依附的cpu cgroup子系统外，还有cpuset cgroup子系统(限制任务可运行的CPU和可使用的内存节点)，blkio cgroup子系统(限制进程的块设备io)，freezer cgroup子系统(提供进程冻结功能)等。上层配置分组下来可能不只切一个cgroup，具体切了哪些子系统体现在集合属性 AggregateProfiles 的数组成员上，比如上例中另外两个属性对应的行为分别是加入blkio子系统的根组和将任务的timer_slack_ns(一个平衡hrtimer定时唤醒及时性与功耗的参数)设置为50000ns。

![Image](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjP7W0s65zd4aJAdJYNYnlGDSuoQnOxgCXRyeACBU4rm47HRX4D7Wm8UZ77sV0WRQe2b1zHGrMvw0g/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

**2.4. 服务启动后就放在指定分组**

在启动服务的时候使用 task_profiles进行配置，举例。

![Image](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjP7W0s65zd4aJAdJYNYnlGDHs14eMEzmaaUGaibQVThC738p8xrC1Bc0iak7Y21FdKiafBxZibLVEFSjg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

**三、内核实现概述**

上面我们讨论了对task group分组的配置，本节开始将进入到内核中，了解其实现。

**3.1. 相关功能依赖关系**

内核相关CONFIG\_\*依赖如下：

图1:

![Image](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjP7W0s65zd4aJAdJYNYnlGDHO2LqVIkl2UHq5bDJ4LYDvkEcPNZoyTpsiagadtFPJq9lM8F1Pmd9vg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

CGROUP提供cgroup目录层次结构的功能；CGROUP_SCHED提供cpu cgroup目录层次结构（如 /dev/cpuctl/top-app），并为每个cpu cgroup目录提供task group的概念；FAIR_GROUP_SCHED基于cpu cgroup提供的task group提供CFS任务组调度功能。图1中灰色的虚线框图是Android手机内核中默认不使能的。

**3.2. task group数据结构框图**

如下图2所示，展示了一个task group在内核中维护的主要数据结构的框图，对照着图下面概念更容易理解：

1. 由于不能确定一个分组内的任务跑在哪个CPU上，因此一个task group在每个CPU上都维护了一个group cfs_rq，由于task group也要参与调度(要先选中task group才能选中其group cfs_rq上的任务) ，因此在每个CPU上也都维护了一个group se。

1. task se的my_q成员为NULL，而group se的my_q成员指向其对应的group cfs_rq，其分组在本CPU上的就绪任务就挂在这个group cfs_rq上。

1. task group可以嵌套，由 parent/siblings/children 成员构成一个倒立树状层次结构，根task group的parent指向NULL。

1. 所有CPU的根cfs_rq属于root task group。

图2:

![Image](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjP7W0s65zd4aJAdJYNYnlGDZZBmfIgJ3XEiaXAAh6Z2DKY3cz5RSkuqHCgUbhP8af5j8Yoia9EdgGmg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

注：画图比较麻烦，此图是借鉴蜗窝的。

**四、相关结构体**

**4.1. struct task_group**

一个 struct task_group 就表示一个cpu cgroup分组。在使能FAIR_GROUP_SCHED的情况下，一个 struct task_group 就表示CFS组调度中的一个任务组。

![Image](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjP7W0s65zd4aJAdJYNYnlGDmJkcNzOy8mZTOHd82rrgLrWQMtE8V6uGiarOuaVPWr5EAqbs3EcQqhA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

css: 该task group对应的cgroup状态信息，通过它依附到cgroup目录层次结构中。

se:  group se，是个数组指针，数组大小为CPU的个数。因为一个task group中有多个任务，且可以跑在所有CPU上，因此需要每个CPU上都要有本task group的一个 se。

cfs_rq:  group se的cfs_rq, 是本task group的在各个CPU上的cfs_rq，也是个数组指针，数组大小为CPU的个数。当本task group的任务就绪后就挂在这个cfs_rq上。它在各个CPU上对应的指针和task group在各个CPU上的se->my_q具有相同指向，见 init_tg_cfs_entry().

shares:  task group的权重，默认为scale_up(1024)。和task se的权重类似，值越大task group能获取的CPU时间片就越多。但是和task 不同的是，task group 在每个CPU上都有一个group se，因此需要按照一定规则分配给各CPU上的group se, 下面会讲解分配规则。

load_avg: 此task group的负载，而且只是一个load_avg 变量(不像se和cfs_rq是一个结构)，下面会对其进行讲解。注意它不是per-cpu的，此task group的任务在各个CPU上进行更新时都会更新它，因此需要注意对性能的影响。

parent/siblings/children: 构成task group的层次结构。

内核中有个全局变量 struct task_group root_task_group, 表示根组。其cfs_rq\[\]就是各个cpu的cfs_rq, 其se\[\]都为NULL。其权重不允许被设置，见 sched_group_set_shares()。负载也不会被更新，见 update_tg_load_avg()。

系统中的所有task group 结构都会被添加到task_groups链表上，在CFS带宽控制时使用。

初学者可能会混淆组调度和调度组这两个概念，以及 struct task_group 与struct sched_group 的区别。组调度对应struct task_group，用来描述一组任务，主要用于util uclamp(对一组任务的算力需求进行钳制)和CPU资源使用限制，也是本文中要讲解的内容。调度组对应 struct sched_group，是CPU拓扑结构sched_domain中的概念，用来描述一个CPU(MC层级)/Cluster(DIE层级)的属性，主要用于选核和负载均衡。

**4.2. struct sched_entity**

一个sched_entity 既可以表示一个task se, 又可以表示一个group se。下面主要介绍使能组调度后新增的一些成员。
!\[\[Pasted image 20240926174104.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

load: 表示se的权重，对于gse, 新建时初始化为NICE_0_LOAD, 见init_tg_cfs_entry()。

depth:  表示task group的嵌套深度，根组下的se的深度为0，每嵌套深一层就加1。比如/dev/cpuctl目录下有个tg1目录，tg1目录下又有一个tg2目录，tg1对应的group se的深度为0，tg1下的task se的深度是1，tg2下的task se的深度是2。更新位置见 init_tg_cfs_entry()/attach_entity_cfs_rq()。

parent: 指向父se节点, 父子se节点都是对应同一cpu的。根组下任务的指向为NULL。

cfs_rq: 该se挂载到的cfs_rq。对于根组下的任务指向rq的cfs_rq，非根组的任务指向其parent->my_rq，见init_tg_cfs_entry()。

my_q: 该se的cfs_rq, 只有group se才有cfs_rq，task se的为NULL，entity_is_task()宏通过这个成员来判断是task se还是group se。

runnable_weight: 缓存 gse->my_q->h_nr_running 的值，在计算gse的runnable负载时使用。

avg: se的负载，对于tse会初始化为其权重(创建时假设其负载很高)，而gse则会初始化为0，见init_entity_runnable_average()。task se的和group se的有一定区别，下面第五章会进行讲解。

**4.3. struct cfs_rq**

struct cfs_rq 既可以表示per-cpu的CFS就绪队列，又可以用来表示gse的my_q队列。下面列出对组调度比较关键的一些成员进行讲解。
!\[\[Pasted image 20240926174111.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

load: 表示cfs_rq的权重，无论是根cfs_rq还是grq，这里的权重都等于其队列上挂的所有任务的权重之和。

nr_running: 当前层级下 task se 和 group se的个数和。

h_nr_running: 当前层级以及所有子层级下task se的个数和，不包括group se。

avg: cfs_rq的负载。下面将对比task se、group se、cfs_rq讲解负载。

removed: 当一个任务退出或者唤醒后迁移到到其他cpu上的时候，原来CPU的cfs rq上需要移除该任务带来的负载。这个移除动作会先把移除的负载记录在这个removed成员中，在下次调用update_cfs_rq_load_avg()更新cfs_rq负载时再移除。nr表示要移除的se的个数，\*\_avg则表示要移除的各类负载之和。

tg_load_avg_contrib: 是对 grq->avg.load_avg 的一个缓存，表示当前grq的load负载对tg的贡献值。用于在更新 tg->load_avg 的同时降低对 tg->load_avg 的访问次数。在计算gse从tg分得的权重配额时的近似算法中也有用到，见 calc_group_shares()/update_tg_load_avg()。

propagate: 标记是否有负载需要向上层传播。下面7.3节会进行讲解。

prop_runnable_sum: 在负载沿着task group层级结构向上层传播的时候，表示要上传的tse/gse的load_sum值。

h_load: 层次负载hierarchy load，表示本层cfs_rq的load_avg对CPU的load_avg的贡献值，主要在负载均衡路径中使用。下面会对其进行讲解。

last_h_load_update: 表示上一次更新h_load的时间点(单位jiffies)。

h_load_next: 指向子gse，为了获取任务的hierarchy load（task_h_load函数），需要从顶层cfs向下，依次更新各个level的cfs rq的h_load。因此，这里的h_load_next就是为了形成一个从顶层cfs rq到底层cfs rq的cfs rq--se--cfs rq--se的关系链。

rq: 使能组调度后才加的这个成员，若没有使能组调度的话，cfs_rq就是rq的一个成员，使用 container_of进行路由，使能组调度后，增加了一个rq成员进行cfs_rq到rq的路由。

on_list/leaf_cfs_rq_list: 尝试将叶cfs_rq串联起来，在CFS负载均衡和带宽控制相关逻辑中使用。

tg: 该cfs_rq隶属的task group。

**五、task group权重**

task group的权重使用struct task_group 的 shares 成员来表示，默认值是scale_load(1024)。可以通过cgroup目录下的cpu.shares文件进行读写，echo weight > cpu.shares 就是将task group权重配置为weight，保存到shares 成员变量中的值是scale_load(weight)。root_task_group不支持设置权重。

不同task group的权重大小表示系统CPU跑满后，哪个task group组可以跑多一些，哪个task group组要跑的少一些。

**5.1. gse的权重**

task group在每个CPU上都有一个group se，那么就需要将task group的权重 tg->shares 按照一定的规则分配到各个gse上。规则就是公式(1)：

- tg->weight * grq->load.weight

- ge->load.weight =   -----------------------------------------              (1)

- \\Sum grq->load.weight

其中 tg->weight 就是tg->shares, grq->load.weight 表示tg在各个CPU上的grq的权重。也就是每个gse根据其cfs_rq的权重比例来分配tg的权重。cfs_rq的权重等于其上挂载的任务的权重之和。假设tg的权重是1024，系统中只有2个CPU，因此有两个gse, 若其grq上任务状态如下如下图3，则gse\[0\]分得的权重为 1024 * (1024+2048+3072)/(1024+2048+3072+1024+1024) = 768；gse\[1\]分得的权重为 1024 * (1024+1024)/(1024+2048+3072+1024+1024) = 256。

图3:
!\[\[Pasted image 20240926174126.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

gse的权重更新函数为 update_cfs_group()，下面看其具体实现:

!\[\[Pasted image 20240926174131.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

tg的权重向gse\[X\]的分配动作是在 calc_group_shares() 中完成的。

公式(1)中使用到 \\Sum grq->load.weight，也就是说一个gse权重的更新需要访问各个CPU上的grq，锁竞争代价比较高，因此进行了一系列的近似计算。

首先进行替换：

- grq->load.weight --> grq->avg.load_avg                         (2)

然后得到：

- tg->weight * grq->avg.load_avg

- ge->load.weight =    ----------------------------------------             (3)

- tg->load_avg

-

- Where: tg->load_avg ~= \\Sum grq->avg.load_avg

由于cfs_rq->avg.load_avg = cfs_rq->avg.load_sum/divider。而 cfs_rq->avg.load_sum 等于 cfs_rq->load.weight 乘以非idle状态下的几何级数。这个近似是在tg的每个CPU上的grq的非idle状态的时间级数是相同的前提下才严格相等的。也就是说tg的任务在各个CPU上的运行状态越一致，越接近这个近似值。

task group 空闲的情况下，启动一个任务。grq->avg.load_avg 需要时间来建立，在建立时间这种特殊情况下公式1简化为：

- tg->weight * grq->load.weight

- ge->load.weight =    ---------------------------------------   =   tg->weight   (4)

- grp->load.weight

相当于一个单核系统下的状态了。为了让公式(3)在这种特殊情况下更贴近与公式(4)，又做了一次近似，得到：

- tg->weight * grq->load.weight

- ge->load.weight =     --------------------------------------------------------------------         (5)

- tg->load_avg - grq->avg.load_avg + grq->load.weight

但是因为grq上没有任务时，grq->load.weight 可以下降到 0，导致除以零，需要使用 grq->avg.load_avg作为它的下限，然后给出：

- tg->weight * grq->load.weight

- ge->load.weight =    ------------------------------------------   (6)

- tg_load_avg'

-

- 其中:

- tg_load_avg' = tg->load_avg - grq->avg.load_avg + max(grq->load.weight, grq->avg.load_avg)

max(grq->load.weight, grq->avg.load_avg) 一般都是取grq->load.weight，因为只有grq上一直有任务running+runnable才会趋近于grq->load.weight。

calc_group_shares() 函数是通过公式(6)近似计算各个gse分得的权重：

!\[\[Pasted image 20240926174142.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

由于tg中的每个任务都对gse的权重有贡献，因此grq上任务个数变更时都要更新gse的权重，近似过程中使用到了se的负载，在entity_tick()中也进行了一次更新。调用路径：
!\[\[Pasted image 20240926174148.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**5.2. gse上每个tse分到的权重**

任务组中的任务也是按其权重比例分配gse的权重。如上图2中gse\[0\]的grq上挂的3个任务，tse1分得的权重就是768*1024/(1024+2048+3072)=128, tse2分得的权重就是768*2048/(1024+2048+3072)=256, tse3分得的权重就是768\*3072/(1024+2048+3072)=384。

在tg中的任务分配tg分得的时间片的时候，会使用到这个按比例分得的权重。分组嵌套的越深，能按比例分得的权重就越小，由此可见，在高负载时task group中的任务是不利于分配时间片的。

**六、task group时间片**

**6.1. 时间片分配**

若使能CFS组调度会从上到下逐层通过权重比例来分配上层分得的时间片，分配函数是sched_slice()。但是从上到下不便于遍历，因此改为从下到上进行遍历，毕竟 A*B*C 和 C*B*A 是相等的。

!\[\[Pasted image 20240926174158.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

sched_slice的主要路径如下：
!\[\[Pasted image 20240926174206.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在tick中断中，若发现se此次运行时间已经超过了其分得的时间片，就触发抢占，以便让其让出CPU。

如下图4，假设tg嵌套2层，且在当前CPU上各层gse从tg那里分得的权重都是1024，且假设直接通过任务个数来计算周期，5个tse，period 就是 3 * 5 = 15ms那么：

tse1 获得 1024/(1024+1024) * 15 = 7.5ms;

tse2 获得 \[1024/(1024+1024+1024)\] * {\[1024/(1024+1024)\] * 15 }= 2.5ms

tse4 获得 \[1024/(1024+1024)\] * {\[1024/(1024+1024+1024)\] * \[1024/(1024+1024)\] * 15} = 1.25ms

图4:
!\[\[Pasted image 20240926174214.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

注：tg1和tg2的权重通过 cpu.shares 文件进行配置，然后各个cpu上的gse从 cpu.shares 配置的权重中按其上的grq的权重比例分配权重。gse的权重不再和nice值挂钩。

**6.2. 运行时间传导**

pick_next_task_fair() 会优先pick虚拟时间最小的se。gse的虚拟时间是怎么更新的呢。虚拟时间是在 update_curr()中进行更新，然后通过 for_each_sched_entity 向上逐层遍历更新gse的虚拟时间。若tse运行5ms，则其父级各gse都运行5ms，然后各层级根据自己的权重更新虚拟时间。

!\[\[Pasted image 20240926174223.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

主要调用路径：
!\[\[Pasted image 20240926174227.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在选择下一个任务出来运行时逐层级选择虚拟时间最小的se，若选到gse就从其grq上继续选，直到选到tse。

**七、task group的PELT负载**

**7.1. 计算负载使用的timeline**

计算负载使用的timeline和计算虚拟时间使用的timeline不同。计算虚拟时间时使用的timeline是 rq->clock_task, 这个是运行多长时间就是多长时间。而计算负载使用的timeline是rq->clock_pelt，它是根据CPU的算力和当前频点scale后的，在CPU进idle是会同步到rq->clock_task上。因此PELT计算出来的负载可以直接使用，而不用像WALT计算出来的负载那样还需要scale。更新rq->clock_pelt这个timeline的函数是 update_rq_clock_pelt()

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

最终计算的 delta= delta * (capacity_cpu / capacity_max(1024)) * (cur_cpu_freq / max_cpu_freq) 也就是将当前cpu在当前频点上运行得到的delta时间值，缩放到最大性能CPU的最大频点上对应的delta时间值。然后累加到 clock_pelt 上。比如在小核上1GHz下跑了5ms，可能只等效于在超大核上运行1ms，因此在不同Cluster的CPU核上跑相同的时间，负载增加量是不一样的。

**7.2. 负载定义与计算**

load_avg 定义为：load_avg = runnable% * scale_load_down(load)。

runnable_avg 定义为：runnable_avg = runnable% * SCHED_CAPACITY_SCALE。

util_avg 定义为：util_avg = running% * SCHED_CAPACITY_SCALE。

这些负载值保存在struct sched_avg结构中，此结构内嵌到se和cfs_rq结构中。此外，struct sched_avg中还引入了load_sum、runnable_sum、util_sum成员来辅助计算。不同实体(tse/gse/grq/cfs_rq)的负载只是其runnable% 多么想运行，和 running% 运行了多少的表现形式不同。这两个因数只对tse取值是\[0,1\]的，对其它实体则超出了这个范围。

**7.2. 1. tse负载**

下面看一下tse负载计算公式，为了加深印象，举一个跑死循环的例子。计算函数见 update_load_avg --> \_\_update_load_avg_se().

load_avg: 等于 weight * load_sum / divider, 其中 weight = sched_prio_to_weight\[prio-100\]。由于 load_sum 是任务 running+runnable 状态的几何级数，divider 近似为几何级数最大值，因此一个死循环任务的 load_avg 接近于其权重。

runnable_avg: 等于 runnable_sum / divider。由于 runnable_sum 是任务 running+runnable 状态的几何级数然后scale up后的值，divider 近似为几何级数最大值，因此一个死循环任务的 runnable_avg 接近于 SCHED_CAPACITY_SCALE。

util_avg: 等于 util_sum / divider。由于 util_sum 是任务 running 状态的几何级数然后scale up后的值，divider 近似为几何级数最大值，因此一个死循环任务的 util_avg 接近于 SCHED_CAPACITY_SCALE。

load_sum: 是对任务是单纯的 running+runnable 状态的几何级数累加值。对于一个死循环，此值趋近于 LOAD_AVG_MAX 。

runnable_sum: 是对任务 running+runnable 状态的几何级数累加值然后scale up后的值。对于一个死循环，此值趋近于 LOAD_AVG_MAX * SCHED_CAPACITY_SCALE 。

util_sum: 是对任务 running 状态的几何级数累加值然后scale up后的值。对于一个独占某个核的死循环，此值趋近于 LOAD_AVG_MAX * SCHED_CAPACITY_SCALE，若不能独占，会比此值小。

**7.2.2. cfs_rq的负载**

下面看一下cfs_rq负载计算公式，为了加深印象，举一个跑死循环的例子。计算函数见 update_load_avg --> update_cfs_rq_load_avg --> \_\_update_load_avg_cfs_rq()。

load_avg: 直接等于 load_sum / divider。cfs_rq 跑满(跑一个死循环或多个死循环)，趋近于cfs_rq的权重，cfs_rq的权重也就是其上挂的所有调度实体的权重之和，即Sum(sched_prio_to_weight\[prio-100\]) 。

runnable_avg: 等于 runnable_sum / divider。cfs_rq 跑满(跑一个死循环或多个死循环)，趋近于cfs_rq上任务个数乘以 SCHED_CAPACITY_SCALE。

util_avg: 等于 util_sum / divider。cfs_rq 跑满(跑一个死循环或多个死循环)，趋近于 SCHED_CAPACITY_SCALE。

load_sum: cfs_rq 的 weight，也就是本层级下所有se的权重之和乘以非idle状态下的几何级数。注意是本层级，下面讲解层次负载h_load时有用到。

runnable_sum: cfs_rq上所有层级的runnable+running 状态任务个数和乘以非idle状态下的几何级数，然后再乘以 SCHED_CAPACITY_SCALE 后的值。见 \_\_update_load_avg_cfs_rq().

util_sum: cfs_rq 上所有任务 running 状态下的几何级数之和再乘以 SCHED_CAPACITY_SCALE 后的值。

load_avg、runnable_avg、util_avg分别从权重(优先级)、任务个数、CPU时间片占用三个维度来描述CPU的负载。

**7.2.3. gse 负载**

对比着tse来讲解gse:

(1) gse会和tse走一样的负载更新流程(逐层向上更新，就会更新到gse)。

(2) gse的runnable负载与tse是不同的。tse的 runnable_sum是任务 running+runnable 状态的几何级数累加值然后scale up后的值。而gse是其当前层级下所有层级的tse的个数之和乘以时间几何级数然后scale up后的值，见 \_\_update_load_avg_se() 函数 runnable 参数的差异。

(3) gse 和tse的 load_avg 虽然都等于 se->weight * load_sum/divider, 见 \_\_\_update_load_avg() 的参数差异。但是weight 来源不同，因此也算的上是一个差异点，tse->weight来源于其优先级，而gse来源于其从tg中分得的配额。

(4) gse会比tse多出了一个负载传导更新过程，放到下面讲解(若不使能CFS组调度，只有一层，没有tg的层次结构，因此不需要传导，只需要更新到cfs_rq上即可)。

**7.2. 4. grq 负载**

grq的负载和cfs_rq的负载在更新上没有什么不同。grq会比cfs_rq多了一个负载传导更新过程，放到下面讲解。

**7.2.5. tg的负载**

tg只有一个load负载，就是tg->load_avg，取值为\\Sum tg->cfs_rq\[\]->avg.load_avg，也即tg所有CPU上的grq的 load_avg 之和。tg负载更新是在update_tg_load_avg()中实现的，主要用于给gse\[\]分配权重。
!\[\[Pasted image 20240926174240.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

调用路径：
!\[\[Pasted image 20240926174245.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**7.3. 负载传导**

负载传导是使能CFS组调度后才有的概念。当tg层次结构上插入或删除一个tse的时候，整个层次结构的负载都变化了，因此需要逐层向上层进行传导。

**7.3.1. 负载传导触发条件**

是否需要进行负载传导是通过struct cfs_rq 的 propagate 成员进行标记。grq上增加/删除的tse时会触发负载传导过程。tse的负载load_sum值会记录在 struct cfs_rq 的 prop_runnable_sum 成员上，然后逐层向上传导。其它负载(runnable\_*、util\_*)则会通过tse-->grq-->gse-->grq...逐层向上层传导。

在 add_tg_cfs_propagate() 中标记需要进行负载传导：

!\[\[Pasted image 20240926174456.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

此函数调用路径：
!\[\[Pasted image 20240926174502.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

由上可见，当从非CSF调度类变为CFS调度类、移到当前tg中来、新建的任务开始挂到cfs_rq上、迁移到当前CPU都会触发负载传导过程，此时会向整个层次结构中传导添加这个任务带来的负载。当任务从当前CPU迁移走、变为非CFS调度类、从tg迁移走，此时会向整个层次结构中传导移除这个任务降低的负载。

注意，任务休眠时并没有将其负载移除，只是休眠期间其负载不增加了，随时间衰减。

**7.3.2. 负载传导过程**

负载传导过程体现在逐层更新负载的过程中。如下，负载更新函数update_load_avg() 在主要路径下，每层都会进行调用：
!\[\[Pasted image 20240926174507.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

load负载传导函数和标记需要进行传导的函数是同一个，为 add_tg_cfs_propagate(), 其调用路径如下：
!\[\[Pasted image 20240926174513.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

7.3.2.1. update_tg_cfs_util() 更新gse和grq的util\_\* 负载，并负责将负载传递给上层。
!\[\[Pasted image 20240926174518.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可见gse的util负载在传导时直接取的是其grq上的util负载。然后通过更新上层 grq 的 util_avg 向上层传导。

7.3.2.2. update_tg_cfs_runnable() 更新gse和grq的runnable\_\*负载，并负责将负载传递给上层。
!\[\[Pasted image 20240926174524.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可见gse的runnable负载在传导时也是直接取的是其grq上的runnable负载。然后通过更新上层 grq 的 runnable_avg 向上层传导。

7.3.2.3. update_tg_cfs_load() 更新gse和grq的load\_\*负载，并负责将负载传递给上层。

load负载比较特殊，负载传导时并不是直接取自grq的load负载，而是在向grq添加/删除任务时就记录了tse 的load_sum值，然后在 add_tg_cfs_propagate() 中逐层向上传导，传导位置调用路径：
!\[\[Pasted image 20240926174529.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

对load负载的标记和传导都是这个函数:
!\[\[Pasted image 20240926174535.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

load负载更新函数：
!\[\[Pasted image 20240926174542.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

删除任务就是将grq上的se的平均load_sum赋值给gse。添加任务是将gse的load_sum直接加上delta值。

load_avg和普通tse计算方式一样，为load_sum\*se_weight(gse)/divider。

对比可见，runnable负载和util负载的传导方向是由grq-->gse，分别通过runnable_avg/util_avg进行传导，gse直接取grq的值。而load负载的传导方向是由gse-->grq进行传导，且是通过load_sum进行传导的。

load负载传导赋值方式上为什么和runnable负载和util负载有差异，可能和其统计算法有关。对于runnable_avg，gse计算的是当前层级下所有层级上tse的个数和乘以runnable状态时间级数的比值，底层增加一个tse对上层相当于tse个数增加了一个；对于util_avg，gse计算的是其下所有tse的running状态几何级数和与时间级数的比值，底层增加一个tse对上层就相当于增加了tse的running状态的几何级数；而 load_avg 和se的权重有关，gse和tse的权重来源不同，前者来自从tg->shares中分得的配额，而后者来源于优先级，不能直接相加减。而load_sum对于se来说是一个单纯的runnable状态的时间级数，不涉及权重，因此tse和gse都可以使用它。

对于load_avg的传导举个例子，如下图5，假如ts2一直休眠，ts1和ts3是两个死循环，那么gse1的grq1的load_avg将趋近于4096，而根cfs_rq的负载将趋近于2048，若此时要将ts3迁移走，若像计算runnable和util负载那样直接想减，得到的delta值是-4096，那么根cfs_rq的load_avg将会是个负值(2048-4096\<0)，这显然是不合理的。若通过load_sum进行传导，它只是个时间级数，相减后根cfs_rq上只相当于损失了50%的负载。

图5:
!\[\[Pasted image 20240926174551.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

注: 这只是在tg的层次结构中添加/删除任务时的负载的传导更新路径，随着时间的流逝，即使没有添加/移除任务，gse/grq的负载也会更新，因为普通的负载更新函数 \_\_update_load_avg_se()/update_cfs_rq_load_avg() 并没有区分是tse还是gse，是cfs_rq还是grq。

**7.4. 层次负载**

在负载均衡的时候，需要迁移CPU上的负载以便达到均衡，为了达成这个目的，需要在CPU之间进行任务迁移。然而各个task se的load avg并不能真实反映它对root cfs rq（即对该CPU）的负载贡献，因为task se/cfs rq总是在某个具体level上计算其load avg。比如grq的load_avg并不会等于其上挂的所有tse的load_avg的和，因为runnable的时间级数肯定是Sum(tse) > grq的(有runnable等待运行的状态存在)。

为了计算task对CPU的负载(h_load)，在各个cfs rq上引入了hierarchy load的概念，对于顶层cfs rq而言，其hierarchy load等于该cfs rq的load avg，随着层级的递进，cfs rq的hierarchy load定义如下：

下一层的cfs rq的h_load = 上一层cfs rq的h_load  x  gse负载在上一层cfs负载中的占比

在计算最底层tse的h_load的时候，我们就使用如下公式：

tse的h_load = grq的h_load  x  tse的load avg / grq的load avg

获取和更新task的h_load的函数如下：
!\[\[Pasted image 20240926174559.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

更新grq的h_load的函数如下：

!\[\[Pasted image 20240926174628.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

调用路径：

!\[\[Pasted image 20240926174634.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可以看到，主要是唤醒wake_affine_weight机制和负载均衡逻辑中使用。比如迁移类型为load的负载均衡中，要迁移多少load_avg可以使负载达到均衡，使用的就是task_h_load()，见 detach_tasks()。

**八、总结**

本文介绍了CFS组调度功能引入的原因，配置方法，和一些实现细节。此功能可以在高负载下"软限制"(相比与CFS带宽控制)各分组任务对CPU资源的使用占比，以达到各组之间公平使用CPU资源的目的。在老版原生Android代码中对后台分组限制的较狠(甚是将 background/cpu.shares 设置到52)，将CPU资源重点向前台分组进行倾斜，但这个配置可能会在某些场景下出现前台任务被后台任务卡住的情况，对于普适性配置，最新的一些Android版本中将各个分组的 cpu.shares 都设置为1024以追求CPU资源在各组之间的公平。

**九、参考**

1.内核源码(https://www.kernel.org/)和Android源码(https://source.android.com/docs/setup/download/downloading)

2.内核文档Documentation/admin-guide/cgroup-v1

3.CFS调度器（3）-组调度: http://www.wowotech.net/process_management/449.html

4.PELT算法浅析: http://www.wowotech.net/process_management/pelt.html

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

长按关注内核工匠微信

Linux内核黑科技| 技术文章 | 精选教程

Reads 2778

​

Send Message

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjNfxSer7sH7b1yJBvlpWyx1AdCugCLScXKu60Ezh9oSCrZw36x9GTL1qvIzptqlefgS1vkQwBE7OA/300?wx_fmt=png&wxfrom=18)

OPPO内核工匠

14183

Send Message

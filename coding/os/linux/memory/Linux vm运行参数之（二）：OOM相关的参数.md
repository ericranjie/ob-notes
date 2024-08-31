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

作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2015-9-28 12:10 分类：[内存管理](http://www.wowotech.net/sort/memory_management)

一、前言

本文是描述Linux virtual memory运行参数的第二篇，主要是讲OOM相关的参数的。为了理解OOM参数，第二章简单的描述什么是OOM。如果这个名词对你毫无压力，你可以直接进入第三章，这一章是描述具体的参数的，除了描述具体的参数，我们引用了一些具体的内核代码，本文的代码来自4.0内核，如果有兴趣，可以结合代码阅读，为了缩减篇幅，文章中的代码都是删减版本的。按照惯例，最后一章是参考文献，本文的参考文献都是来自linux内核的Documentation目录，该目录下有大量的文档可以参考，每一篇都值得细细品味。

二、什么是OOM

OOM就是out of memory的缩写，虽然linux kernel有很多的内存管理技巧（从cache中回收、swap out等）来满足各种应用空间的vm内存需求，但是，当你的系统配置不合理，让一匹小马拉大车的时候，linux kernel会运行非常缓慢并且在某个时间点分配page frame的时候遇到内存耗尽、无法分配的状况。应对这种状况首先应该是系统管理员，他需要首先给系统增加内存，不过对于kernel而言，当面对OOM的时候，咱们也不能慌乱，要根据OOM参数来进行相应的处理。

三、OOM参数

1、panic_on_oom

当kernel遇到OOM的时候，可以有两种选择：

（1）产生kernel panic（就是死给你看）。

（2）积极面对人生，选择一个或者几个最“适合”的进程，启动OOM killer，干掉那些选中的进程，释放内存，让系统勇敢的活下去。

panic_on_oom这个参数就是控制遇到OOM的时候，系统如何反应的。当该参数等于0的时候，表示选择积极面对人生，启动OOM killer。当该参数等于2的时候，表示无论是哪一种情况，都强制进入kernel panic。panic_on_oom等于其他值的时候，表示要区分具体的情况，对于某些情况可以panic，有些情况启动OOM killer。kernel的代码中，enum oom_constraint 就是一个进一步描述OOM状态的参数。系统遇到OOM总是有各种各样的情况的，kernel中定义如下：

> enum oom_constraint {  
>     CONSTRAINT_NONE,  
>     CONSTRAINT_CPUSET,  
>     CONSTRAINT_MEMORY_POLICY,  
>     CONSTRAINT_MEMCG,  
> };

对于UMA而言， oom_constraint永远都是CONSTRAINT_NONE，表示系统并没有什么约束就出现了OOM，不要想太多了，就是内存不足了。在NUMA的情况下，有可能附加了其他的约束导致了系统遇到OOM状态，实际上，系统中还有充足的内存。这些约束包括：

（1）CONSTRAINT_CPUSET。cpusets是kernel中的一种机制，通过该机制可以把一组cpu和memory node资源分配给特定的一组进程。这时候，如果出现OOM，仅仅说明该进程能分配memory的那个node出现状况了，整个系统有很多的memory node，其他的node可能有充足的memory资源。

（2）CONSTRAINT_MEMORY_POLICY。memory policy是NUMA系统中如何控制分配各个memory node资源的策略模块。用户空间程序（NUMA-aware的程序）可以通过memory policy的API，针对整个系统、针对一个特定的进程，针对一个特定进程的特定的VMA来制定策略。产生了OOM也有可能是因为附加了memory policy的约束导致的，在这种情况下，如果导致整个系统panic似乎有点不太合适吧。

（3）CONSTRAINT_MEMCG。MEMCG就是memory control group，Cgroup这东西太复杂，这里不适合多说，Cgroup中的memory子系统就是控制系统memory资源分配的控制器，通俗的将就是把一组进程的内存使用限定在一个范围内。当这一组的内存使用超过上限就会OOM，在这种情况下的OOM就是CONSTRAINT_MEMCG类型的OOM。

OK，了解基础知识后，我们来看看内核代码。内核中sysctl_panic_on_oom变量是和/proc/sys/vm/panic_on_oom对应的，主要的判断逻辑如下：

> void check_panic_on_oom(enum oom_constraint constraint, gfp_t gfp_mask,  
>             int order, const nodemask_t *nodemask)  
> {  
>     if (likely(!sysctl_panic_on_oom))－－－－0表示启动OOM killer，因此直接return了  
>         return;  
>     if (sysctl_panic_on_oom != 2) {－－－－2是强制panic，不是2的话，还可以商量  
>         if (constraint != CONSTRAINT_NONE)－－－在有cpuset、memory policy、memcg的约束情况下  
>             return;                                                  的OOM，可以考虑不panic，而是启动OOM killer  
>     }  
>     dump_header(NULL, gfp_mask, order, NULL, nodemask);  
>     panic("Out of memory: %s panic_on_oom is enabled\n",  
>         sysctl_panic_on_oom == 2 ? "compulsory" : "system-wide");－－－死给你看啦  
> }

2、oom_kill_allocating_task

当系统选择了启动OOM killer，试图杀死某些进程的时候，又会遇到这样的问题：干掉哪个，哪一个才是“合适”的哪那个进程？系统可以有下面的选择：

（1）谁触发了OOM就干掉谁

（2）谁最“坏”就干掉谁

oom_kill_allocating_task这个参数就是控制这个选择路径的，当该参数等于0的时候选择（2），否则选择（1）。具体的代码可以在参考__out_of_memory函数，具体如下：

> static void __out_of_memory(struct zonelist *zonelist, gfp_t gfp_mask,  
>         int order, nodemask_t *nodemask, bool force_kill)   {
> 
> ……  
>     check_panic_on_oom(constraint, gfp_mask, order, mpol_mask);
> 
>     if (sysctl_oom_kill_allocating_task && current->mm &&  
>         !oom_unkillable_task(current, NULL, nodemask) &&  
>         current->signal->oom_score_adj != OOM_SCORE_ADJ_MIN) {  
>         get_task_struct(current);  
>         oom_kill_process(current, gfp_mask, order, 0, totalpages, NULL,  
>                  nodemask, "Out of memory (oom_kill_allocating_task)");  
>         goto out;  
>     }
> 
> ……  
> }

当然也不能说杀就杀，还是要考虑是否用户空间进程（不能杀内核线程）、是否unkillable task（例如init进程就不能杀），用户空间是否通过设定参数（oom_score_adj）阻止kill该task。如果万事俱备，那么就调用oom_kill_process干掉当前进程。

3、oom_dump_tasks

当系统的内存出现OOM状况，无论是panic还是启动OOM killer，做为系统管理员，你都是想保留下线索，找到OOM的root cause，例如dump系统中所有的用户空间进程关于内存方面的一些信息，包括：进程标识信息、该进程使用的total virtual memory信息、该进程实际使用物理内存（我们又称之为RSS，Resident Set Size，不仅仅是自己程序使用的物理内存，也包含共享库占用的内存），该进程的页表信息等等。拿到这些信息后，有助于了解现象（出现OOM）之后的真相。

当设定为0的时候，上一段描述的各种进程们的内存信息都不会打印出来。在大型的系统中，有几千个进程，逐一打印每一个task的内存信息有可能会导致性能问题（要知道当时已经是OOM了）。当设定为非0值的时候，在下面三种情况会调用dump_tasks来打印系统中所有task的内存状况：

（1）由于OOM导致kernel panic

（2）没有找到适合的“bad”process

（3）找适合的并将其干掉的时候

4、oom_adj、oom_score_adj和oom_score

准确的说这几个参数都是和具体进程相关的，因此它们位于/proc/xxx/目录下（xxx是进程ID）。假设我们选择在出现OOM状况的时候杀死进程，那么一个很自然的问题就浮现出来：到底干掉哪一个呢？内核的算法倒是非常简单，那就是打分（oom_score，注意，该参数是read only的），找到分数最高的就OK了。那么怎么来算分数呢？可以参考内核中的oom_badness函数：

> unsigned long oom_badness(struct task_struct *p, struct mem_cgroup *memcg,  
>               const nodemask_t *nodemask, unsigned long totalpages)  
> {……
> 
>     adj = (long)p->signal->oom_score_adj;  
>     if (adj == OOM_SCORE_ADJ_MIN) {－－－－－－－－－－－－－－－－－－－－－－（1）  
>         task_unlock(p);  
>         return 0;－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－（2）  
>     }
> 
>     points = get_mm_rss(p->mm) + get_mm_counter(p->mm, MM_SWAPENTS) +  
>         atomic_long_read(&p->mm->nr_ptes) + mm_nr_pmds(p->mm);－－－－－－－－－（3）  
>     task_unlock(p);
> 
>   
>     if (has_capability_noaudit(p, CAP_SYS_ADMIN))－－－－－－－－－－－－－－－－－（4）  
>         points -= (points * 3) / 100;
> 
>     adj *= totalpages / 1000;－－－－－－－－－－－－－－－－－－－－－－－－－－－－（5）  
>     points += adj;   
>   
>     return points > 0 ? points : 1;  
> }

（1）对某一个task进行打分（oom_score）主要有两部分组成，一部分是系统打分，主要是根据该task的内存使用情况。另外一部分是用户打分，也就是oom_score_adj了，该task的实际得分需要综合考虑两方面的打分。如果用户将该task的 oom_score_adj设定成OOM_SCORE_ADJ_MIN（-1000）的话，那么实际上就是禁止了OOM killer杀死该进程。

（2）这里返回了0也就是告知OOM killer，该进程是“good process”，不要干掉它。后面我们可以看到，实际计算分数的时候最低分是1分。

（3）前面说过了，系统打分就是看物理内存消耗量，主要是三部分，RSS部分，swap file或者swap device上占用的内存情况以及页表占用的内存情况。

（4）root进程有3%的内存使用特权，因此这里要减去那些内存使用量。

（5）用户可以调整oom_score，具体如何操作呢？oom_score_adj的取值范围是-1000～1000，0表示用户不调整oom_score，负值表示要在实际打分值上减去一个折扣，正值表示要惩罚该task，也就是增加该进程的oom_score。在实际操作中，需要根据本次内存分配时候可分配内存来计算（如果没有内存分配约束，那么就是系统中的所有可用内存，如果系统支持cpuset，那么这里的可分配内存就是该cpuset的实际额度值）。oom_badness函数有一个传入参数totalpages，该参数就是当时的可分配的内存上限值。实际的分数值（points）要根据oom_score_adj进行调整，例如如果oom_score_adj设定-500，那么表示实际分数要打五折（基数是totalpages），也就是说该任务实际使用的内存要减去可分配的内存上限值的一半。

了解了oom_score_adj和oom_score之后，应该是尘埃落定了，oom_adj是一个旧的接口参数，其功能类似oom_score_adj，为了兼容，目前仍然保留这个参数，当操作这个参数的时候，kernel实际上是会换算成oom_score_adj，有兴趣的同学可以自行了解，这里不再细述了。

四、参考文献

1、Documentation/vm/numa_memory_policy.txt

2、Documentation/sysctl/vm.txt

3、Documentation/cgroup/cpusets.txt

4、Documentation/cgroup/memory.txt

5、Documentation/filesystems/proc.txt

_原创文章，转发请注明出处。蜗窝科技_

标签: [oom](http://www.wowotech.net/tag/oom)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [ARM64的启动过程之（一）：内核第一个脚印](http://www.wowotech.net/armv8a_arch/arm64_initialize_1.html) | [Linux vm运行参数之（一）：overcommit相关的参数](http://www.wowotech.net/memory_management/overcommit.html)»

**评论：**

**[andy01011501](http://www.wowotech.net/)**  
2016-03-09 11:14

各位，我想保持一个普通文件在内存中的cache不被置换，也就是永久保留在内存中，有没有什么好办法啊？

[回复](http://www.wowotech.net/memory_management/oom.html#comment-3620)

**[andy01011501](http://www.wowotech.net/)**  
2016-02-25 20:05

有没有办法关闭linux的内存的cache的功能？就是让meminfo中的cached这个值为0，每次文件都从flash读取？

[回复](http://www.wowotech.net/memory_management/oom.html#comment-3531)

**[郭健](http://www.wowotech.net/)**  
2016-02-26 12:31

@andy01011501：利用file cache来加速文件的读写是一个挺好的机制，为何要在整个linux kernel级别上关掉这个功能呢？内核配置项我是不记得有这个option的。  
  
如果你是想在访问文件的时候越过file cache，那么可以考虑在open文件的时候使用O_DIRECT这个flag。

[回复](http://www.wowotech.net/memory_management/oom.html#comment-3540)

**[andy01011501](http://www.wowotech.net/)**  
2016-02-26 14:37

@郭健：这个仅仅是为了做一个测试，让读写文件的时候直接操作flash，而且是所有的文件都要这么操作，哈

[回复](http://www.wowotech.net/memory_management/oom.html#comment-3544)

**[郭健](http://www.wowotech.net/)**  
2016-02-26 22:11

@andy01011501：我觉得操作系统层面应该不会设定这样一个flag，操作系统可能会提供接口来设定page cache的额度范围，应该不会ON/OFF page cache吧。文件系统层面的话还是有可能的（可能调试文件系统的时候会打开这个开关）。  
  
如果是测试，可以自己修改内核代码，反正是玩玩嘛。

[回复](http://www.wowotech.net/memory_management/oom.html#comment-3555)

**[andy01011501](http://www.wowotech.net/)**  
2016-02-27 09:43

@郭健：嗯，正在看代码呢，谢啦

[回复](http://www.wowotech.net/memory_management/oom.html#comment-3557)

**[tigger](http://www.wowotech.net/)**  
2015-10-28 15:36

hi linuxer  
请教一个问题，什么情况下，会一直杀死一个进程，但是杀不死。比如一直有个signal 9 要杀死这个进程，然后do_exit 然后过一会又 重新来一遍。但是这个进程就是杀不死。  
会不会进程处于freeze状态 会出现这样的情景？

[回复](http://www.wowotech.net/memory_management/oom.html#comment-2878)

**[linuxer](http://www.wowotech.net/)**  
2015-10-29 08:54

@tigger：这个杀不死的进程是否被其父进程监控？父进程发现其子进程挂掉，然后会重新加载该进程，由于该进程自身的问题（代码有issue，或者消耗太多内存等），被内核干掉，父进程又reload该程序，然后就反反复复......  
  
freeze和电源管理有关，你的场景和电源管理相关吗？

[回复](http://www.wowotech.net/memory_management/oom.html#comment-2886)

**[tigger](http://www.wowotech.net/)**  
2015-10-29 10:05

@linuxer：跟电源管理有关系，怀疑suspend 过程与kill 进程冲突，导致进程无法被杀死

[回复](http://www.wowotech.net/memory_management/oom.html#comment-2889)

**pga**  
2015-10-08 09:45

hi linuxer:  
  请教你一个困扰我很久的问题，具体描述如下：  
在系统调用的汇编部分有：  
  
ENTRY(vector_swi)  
.....  
    @ are we tracing syscalls?  
    bne    __sys_trace  
  
    cmp    scno, #NR_syscalls        @ check upper syscall limit  
    adr    lr, BSYM(ret_fast_syscall)    @ return address  
    ldrcc    pc, [tbl, scno, lsl #2]        @ call sys_* routine  
......  
  
ENDPROC(vector_swi)  
  
从这个汇编片段及注释可知，pc地址为sys_xx所指向的地址值，其返回lr为ret_fast_syscall，按这个我理解为：最先执行的应该是sys_xxx的函数，但实际上我看调用栈都是ret_fast_syscall为最先执行。如：  
  
[   51.426327] [<c073fb0c>] (__schedule+0x37c/0x7c4) from [<c073f0f8>]  (schedule_hrtimeout_range_clock+0x130/0x14c)  
[   51.426343] [<c073f0f8>] (schedule_hrtimeout_range_clock+0x130/0x14c) from [<c0161c84>] (SyS_epoll_wait+0x304/0x458)  
[   51.426357] [<c0161c84>] (SyS_epoll_wait+0x304/0x458) from [<c0161f0c>] (SyS_epoll_pwait+0x134/0x140)  
[   51.426372] [<c0161f0c>] (SyS_epoll_pwait+0x134/0x140) from [<c000f300>] (ret_fast_syscall+0x0/0x30)  
  
为何会这样呢？ret_fast_syscall不是lr的值吗？为何不是最先执行的是pc值而是lr值呢？期待大师的回复，谢谢！

[回复](http://www.wowotech.net/memory_management/oom.html#comment-2725)

**[linuxer](http://www.wowotech.net/)**  
2015-10-08 16:05

@pga：大师是不敢当了，就是普通爱好者而已。  
  
具体的调用顺序当然如你所说：首先执行sys_xxx函数，执行完毕之后，再调用ret_fast_syscall返回现场。  
  
我看到你贴出来的栈的回溯也很正常：  
__schedule  
schedule_hrtimeout_range_clock  
SyS_epoll_wait  
SyS_epoll_pwait  
ret_fast_syscall  
  
正常而言，kernel stack上的内容就是这样的啊，只不过SyS_epoll_pwait和ret_fast_syscall之间不是函数调用关系，也就是说不是先执行ret_fast_syscall再执行SyS_epoll_pwait，之所以kernel stack上的内容如此，主要是因为下面的代码：  
  
adr    lr, BSYM(ret_fast_syscall)    @ return address  
  
这句代码模拟了一个从ret_fast_syscall到SyS_epoll_pwait的调用现场（因此栈的回溯看起来好象先调用了ret_fast_syscal），实际上这件事情从来没有发生过，这么做仅仅是为了完成SyS_epoll_pwait函数之后返回执行ret_fast_syscall。

[回复](http://www.wowotech.net/memory_management/oom.html#comment-2727)

**[pgalxx](http://www.wowotech.net/)**  
2015-10-08 21:33

@linuxer：敬您一点拨，完全理解了，之前掉进惯性思维以为栈回溯就肯定是需要执行的。谢谢哈

[回复](http://www.wowotech.net/memory_management/oom.html#comment-2731)

**[passerby](http://www.wowotech.net/)**  
2015-10-07 16:31

android的oom增加了另外一套并行的Lowmemkiller机制，会主动的监控内存。当内存低于设定的门阀值时就会主动的杀掉占有内存较高的进程。

[回复](http://www.wowotech.net/memory_management/oom.html#comment-2719)

**[linuxer](http://www.wowotech.net/)**  
2015-10-08 09:06

@passerby：为何android会创建Lowmemkiller这个机制？是内核OOM模块不满足需求吗？有时间的话可以给大家讲讲

[回复](http://www.wowotech.net/memory_management/oom.html#comment-2722)

**[passerby](http://www.wowotech.net/)**  
2015-10-08 09:15

@linuxer：我猜测可能android认为发生do_page_fault时再进行oom已经太晚了，而是在swapd线程中增加了对内存监控的功能，当发现当前内存不够时主动的去杀掉从而释放内存。  
static short lowmem_adj[6] = {  
    0,  
    1,  
    6,  
    12,  
};  
static int lowmem_adj_size = 4;  
static int lowmem_minfree[6] = {  
    3 * 512,    /* 6MB */  
    2 * 1024,    /* 8MB */  
    4 * 1024,    /* 16MB */  
    16 * 1024,    /* 64MB */  
};  
  
从上面可以看出，当内存小于64M时，android会遍历进程oom_core > 12的所有进程，然后杀掉占有最多内存的进程。内存的紧张程度不同，android选择杀掉进程的oom_adj就不同。

[回复](http://www.wowotech.net/memory_management/oom.html#comment-2723)

**[passerby](http://www.wowotech.net/)**  
2015-10-08 09:18

@linuxer：我猜测可能android认为发生do_page_fault时再进行oom已经太晚了，而是在swapd线程中增加了对内存监控的功能，当发现当前内存不够时主动的去杀掉从而释放内存。  
static short lowmem_adj[6] = {  
    0,  
    1,  
    6,  
    12,  
};  
static int lowmem_adj_size = 4;  
static int lowmem_minfree[6] = {  
    3 * 512,    /* 6MB */  
    2 * 1024,    /* 8MB */  
    4 * 1024,    /* 16MB */  
    16 * 1024,    /* 64MB */  
};  
  
从上面可以看出，当内存小于64M时，android会遍历进程oom_core > 12的所有进程，然后杀掉占有最多内存的进程。内存的紧张程度不同，android选择杀掉进程的oom_adj就不同。

[回复](http://www.wowotech.net/memory_management/oom.html#comment-2724)

**cy**  
2015-10-09 18:21

@passerby：android的这个机制，比如上面的lowmem_minfree，是否可以用户通过sysfs动态调整?

[回复](http://www.wowotech.net/memory_management/oom.html#comment-2740)

**[passerby](http://www.wowotech.net/)**  
2015-10-09 19:07

@cy：lowmem_minfree是用module_param_array_named定义的，应该是可以通过sys设置它的值得。

[回复](http://www.wowotech.net/memory_management/oom.html#comment-2741)

**[安安博客](http://xuzian.com/)**  
2015-10-07 11:33

看不懂...

[回复](http://www.wowotech.net/memory_management/oom.html#comment-2718)

**江南王公子**  
2020-11-24 17:47

@安安博客：太菜了，好好学习啊

[回复](http://www.wowotech.net/memory_management/oom.html#comment-8145)

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
    
    - [DRAM 原理 5 ：DRAM Devices Organization](http://www.wowotech.net/basic_tech/343.html)
    - [load_balance函数代码详解](http://www.wowotech.net/process_management/load_balance_function.html)
    - [Linux common clock framework(1)_概述](http://www.wowotech.net/pm_subsystem/clk_overview.html)
    - [Dynamic DMA mapping Guide](http://www.wowotech.net/memory_management/DMA-Mapping-api.html)
    - [“蜗窝”使用的软件开发环境介绍](http://www.wowotech.net/soft/5.html)
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
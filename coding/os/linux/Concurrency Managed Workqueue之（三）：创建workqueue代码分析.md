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

作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2015-8-6 18:22 分类：[中断子系统](http://www.wowotech.net/sort/irq_subsystem)

一、前言

本文主要以__alloc_workqueue_key函数为主线，描述CMWQ中的创建一个workqueue实例的代码过程。

二、WQ_POWER_EFFICIENT的处理

__alloc_workqueue_key函数的一开始有如下的代码：

> if ((flags & WQ_POWER_EFFICIENT) && wq_power_efficient)  
>         flags |= WQ_UNBOUND;

在kernel中，有两种线程池，一种是线程池是per cpu的，也就是说，系统中有多少个cpu，就会创建多少个线程池，cpu x上的线程池创建的worker线程也只会运行在cpu x上。另外一种是unbound thread pool，该线程池创建的worker线程可以调度到任意的cpu上去。由于cache locality的原因，per cpu的线程池的性能会好一些，但是对power saving有一些影响。设计往往如此，workqueue需要在performance和power saving之间平衡，想要更好的性能，那么最好让一个cpu上的worker thread来处理work，这样的话，cache命中率会比较高，性能会更好。但是，从电源管理的角度来看，最好的策略是让idle状态的cpu尽可能的保持idle，而不是反复idle，working，idle again。

我们来一个例子辅助理解上面的内容。在t1时刻，work被调度到CPU A上执行，t2时刻work执行完毕，CPU A进入idle，t3时刻有一个新的work需要处理，这时候调度work到那个CPU会好些呢？是处于working状态的CPU B还是处于idle状态的CPU A呢？如果调度到CPU A上运行，那么，由于之前处理过work，其cache内容新鲜热辣，处理起work当然是得心应手，速度很快，但是，这需要将CPU A从idle状态中唤醒。选择CPU B呢就不存在将CPU 从idle状态唤醒，从而获取power saving方面的好处。

了解了上面的基础内容之后，我们再来检视per cpu thread pool和unbound thread pool。当workqueue收到一个要处理的work，如果该workqueue是unbound类型的话，那么该work由unbound thread pool处理并把调度该work去哪一个CPU执行这样的策略交给系统的调度器模块来完成，对于scheduler而言，它会考虑CPU core的idle状态，从而尽可能的让CPU保持在idle状态，从而节省了功耗。因此，如果一个workqueue有WQ_UNBOUND这样的flag，则说明该workqueue上挂入的work处理是考虑到power saving的。如果workqueue没有WQ_UNBOUND flag，则说明该workqueue是per cpu的，这时候，调度哪一个CPU core运行worker thread来处理work已经不是scheduler可以控制的了，这样，也就间接影响了功耗。

有两个参数可以控制workqueue在performance和power saving之间的平衡：

1、各个workqueue需要通过WQ_POWER_EFFICIENT来标记自己在功耗方面的属性

2、系统级别的内核参数workqueue.power_efficient。

使用workqueue的用户知道自己在电源管理方面的特点，如果该workqueue在unbound的时候会极大的降低功耗，那么就需要加上WQ_POWER_EFFICIENT的标记。这时候，如果没有标记WQ_UNBOUND，那么缺省workqueue会创建per cpu thread pool来处理work。不过，也可以通过workqueue.power_efficient这个内核参数来修改workqueue的行为：

> #ifdef CONFIG_WQ_POWER_EFFICIENT_DEFAULT  
> static bool wq_power_efficient = true;  
> #else  
> static bool wq_power_efficient;  
> #endif
> 
> module_param_named(power_efficient, wq_power_efficient, bool, 0444);

如果wq_power_efficient设定为true，那么WQ_POWER_EFFICIENT的标记的workqueue就会强制按照unbound workqueue来处理，即使没有标记WQ_UNBOUND。

三、分配workqueue的内存

> if (flags & WQ_UNBOUND)  
>     tbl_size = nr_node_ids * sizeof(wq->numa_pwq_tbl[0]); －－－only for unbound workqueue
> 
> wq = kzalloc(sizeof(*wq) + tbl_size, GFP_KERNEL);
> 
> if (flags & WQ_UNBOUND) {  
>         wq->unbound_attrs = alloc_workqueue_attrs(GFP_KERNEL); －－only for unbound workqueue  
>     }

代码很简单，与其要解释代码，不如来解释一些基本概念。

1、workqueue和pool workqueue的关系

我们先给出一个简化版本的workqueue_struct定义，如下：

> struct workqueue_struct {  
>     struct list_head    pwqs;   
>     struct list_head    list;
> 
>   
>     struct pool_workqueue __percpu *cpu_pwqs;  －－－－－指向per cpu的pool workqueue  
>     struct pool_workqueue __rcu *numa_pwq_tbl[]; －－－－指向per node的pool workqueue  
> };

这里涉及2个数据结构：workqueue_struct和pool_workqueue，为何如此处理呢？我们知道，在CMWQ中，workqueue和thread pool没有严格的一一对应关系了，因此，系统中的workqueue们共享一组thread pool，因此，workqueue中的成员包括两个类别：global类型和per thread pool类型的，我们把那些per thread pool类型的数据集合起来就形成了pool_workqueue的定义。

挂入workqueue的work终究需要worker pool中的某个worker thread来处理，也就是说，workqueue要和系统中那些共享的worker thread pool进行连接，这是通过pool_workqueue（该数据结构会包含一个指向worker pool的指针）的数据结构来管理的。和这个workqueue相关的pool_workqueue被挂入一个链表，链表头就是workqueue_struct中的pwqs成员。

和旧的workqueue机制一样，系统维护了一个所有workqueue的list，list head定义如下：

> static LIST_HEAD(workqueues);

workqueue_struct中的list成员就是挂入这个链表的节点。

workqueue有两种：unbound workqueue和per cpu workqueue。对于per cpu类型，cpu_pwqs指向了一组per cpu的pool_workqueue数据结构，用来维护workqueue和per cpu thread pool之间的关系。每个cpu都有两个thread pool，normal和高优先级的线程池，到底cpu_pwqs指向哪一个pool_workqueue（worker thread）是和workqueue的flag相关，如果标有WQ_HIGHPRI，那么cpu_pwqs指向高优先级的线程池。unbound workqueue对应的pool_workqueue和workqueue属性相关，我们在下一节描述。

2、workqueue attribute

挂入workqueue的work终究是需要worker线程来处理，针对worker线程有下面几个考量点（我们称之attribute）：

（1）该worker线程的优先级

（2）该worker线程运行在哪一个CPU上

（3）如果worker线程可以运行在多个CPU上，且这些CPU属于不同的NUMA node，那么是否在所有的NUMA node中都可以获取良好的性能。

对于per-CPU的workqueue，2和3不存在问题，哪个cpu上queue的work就在哪个cpu上执行，由于只能在一个确定的cpu上执行，因此起NUMA的node也是确定的（一个CPU不可能属于两个NUMA node）。置于优先级，per-CPU的workqueue使用WQ_HIGHPRI来标记。综上所述，per-CPU的workqueue不需要单独定义一个workqueue attribute，这也是为何在workqueue_struct中只有unbound_attrs这个成员来记录unbound workqueue的属性。

unbound workqueue由于不绑定在具体的cpu上，可以运行在系统中的任何一个cpu，直觉上似乎系统中有一个unbound thread pool就OK了，不过让一个thread pool创建多种属性的worker线程是一个好的设计吗？本质上，thread pool应该创建属性一样的worker thread。因此，我们通过workqueue属性来对unbound workqueue进行分类，workqueue属性定义如下：

> struct workqueue_attrs {  
>     int            nice;        /* nice level */  
>     cpumask_var_t        cpumask;    /* allowed CPUs */  
>     bool            no_numa;    /* disable NUMA affinity */  
> };

nice是一个和thread优先级相关的属性，nice越低则优先级越高。cpumask是该workqueue挂入的work允许在哪些cpu上运行。no_numa是一个和NUMA affinity相关的设定。

3、unbound workqueue和NUMA之间的联系

UMA系统中，所有的processor看到的内存都是一样的，访问速度也是一样，无所谓local or remote，因此，内核线程如果要分配内存，那么也是无所谓，统一安排即可。在NUMA系统中，不同的一个或者一组cpu看到的memory是不一样的，我们假设node 0中有CPU A和B，node 1中有CPU C和D，如果运行在CPU A上内核线程现在要迁移到CPU C上的时候，悲剧发生了：该线程在A CPU创建并运行的时候，分配的内存是node 0中的memory，这些memory是local的访问速度很快，当迁移到CPU C上的时候，原来local memory变成remote，性能大大降低。因此，unbound workqueue需要引入NUMA的考量点。

NUMA是内存管理的范畴，本文不会深入描述，我们暂且放开NUMA，先思考这样的一个问题：一个确定属性的unbound workqueue需要几个线程池？看起来一个就够了，毕竟workqueue的属性已经确定了，一个线程池创建相同属性的worker thread就行了。但是我们来看一个例子：假设workqueue的work是可以在node 0中的CPU A和B，以及node 1中CPU C和D上处理，如果只有一个thread pool，那么就会存在worker thread在不同node之间的迁移问题。为了解决这个问题，实际上unbound workqueue实际上是创建了per node的pool_workqueue（thread pool）

当然，是否使用per node的pool workqueue用户是可以通过下面的参数进行设定的：

（1）workqueue attribute中的no_numa成员

（2）通过workqueue.disable_numa这个参数，disable所有workqueue的numa affinity的支持。

> static bool wq_disable_numa;  
> module_param_named(disable_numa, wq_disable_numa, bool, 0444);

四、初始化workqueue的成员

> va_start(args, lock_name);  
> vsnprintf(wq->name, sizeof(wq->name), fmt, args);－－－－－set workqueue name  
> va_end(args);
> 
> max_active = max_active ?: WQ_DFL_ACTIVE;  
> max_active = wq_clamp_max_active(max_active, flags, wq->name);  
> wq->flags = flags;  
> wq->saved_max_active = max_active;  
> mutex_init(&wq->mutex);  
> atomic_set(&wq->nr_pwqs_to_flush, 0);  
> INIT_LIST_HEAD(&wq->pwqs);  
> INIT_LIST_HEAD(&wq->flusher_queue);  
> INIT_LIST_HEAD(&wq->flusher_overflow);  
> INIT_LIST_HEAD(&wq->maydays);
> 
> lockdep_init_map(&wq->lockdep_map, lock_name, key, 0);  
> INIT_LIST_HEAD(&wq->list);

除了max active，没有什么要说的，代码都简单而且直观。如果用户没有设定max active（或者说max active等于0），那么系统会给出一个缺省的设定。系统定义了两个最大值WQ_MAX_ACTIVE（512）和WQ_UNBOUND_MAX_ACTIVE（和cpu数目有关，最大值是cpu数目乘以4，当然也不能大于WQ_MAX_ACTIVE），分别限定per cpu workqueue和unbound workqueue的最大可以创建的worker thread的数目。wq_clamp_max_active可以将max active限制在一个确定的范围内。

五、分配pool workqueue的内存并建立workqueue和pool workqueue的关系

这部分的代码主要涉及alloc_and_link_pwqs函数，如下：

> static int alloc_and_link_pwqs(struct workqueue_struct *wq)  
> {  
>     bool highpri = wq->flags & WQ_HIGHPRI;－－－－normal or high priority？  
>     int cpu, ret;
> 
>     if (!(wq->flags & WQ_UNBOUND)) {－－－－－per cpu workqueue的处理  
>         wq->cpu_pwqs = alloc_percpu(struct pool_workqueue);
> 
>         for_each_possible_cpu(cpu) {－－－－－逐个cpu进行设定  
>             struct pool_workqueue *pwq =    per_cpu_ptr(wq->cpu_pwqs, cpu);  
>             struct worker_pool *cpu_pools = per_cpu(cpu_worker_pools, cpu);
> 
>             init_pwq(pwq, wq, &cpu_pools[highpri]);   
>             link_pwq(pwq);－－－－上面两行代码用来建立workqueue、pool wq和thread pool之间的关系  
>         }  
>         return 0;  
>     } else if (wq->flags & __WQ_ORDERED) {－－－－－ordered unbound workqueue的处理  
>         ret = apply_workqueue_attrs(wq, ordered_wq_attrs[highpri]);  
>         return ret;  
>     } else {－－－－－unbound workqueue的处理  
>         return apply_workqueue_attrs(wq, unbound_std_wq_attrs[highpri]);  
>     }  
> }

通过alloc_percpu可以为每一个cpu分配一个pool_workqueue的memory。每个pool_workqueue都有一个对应的worker thread pool，对于per-CPU workqueue，它是静态定义的，如下：

> static DEFINE_PER_CPU_SHARED_ALIGNED(struct worker_pool [NR_STD_WORKER_POOLS],  
>                      cpu_worker_pools);

init_pwq函数初始化pool_workqueue，最重要的是设定其对应的workqueue和worker pool。link_pwq主要是将pool_workqueue挂入它所属的workqueue的链表中。对于unbound workqueue，apply_workqueue_attrs完成分配pool workqueue并建立workqueue和pool workqueue的关系。

六、应用新的attribute到workqueue中

unbound workqueue有两种，一种是normal type，另外一种是ordered type，这种workqueue上的work是严格按照顺序执行的，不存在并发问题。ordered unbound workqueue的行为类似过去的single thread workqueue。但是，无论那种类型的unbound workqueue都使用apply_workqueue_attrs来建立workqueue、pool wq和thread pool之间的关系。

1、健康检查。

> if (WARN_ON(!(wq->flags & WQ_UNBOUND)))  
>     return -EINVAL;
> 
> if (WARN_ON((wq->flags & __WQ_ORDERED) && !list_empty(&wq->pwqs)))  
>     return -EINVAL;

只有unbound类型的workqueue才有attribute，才可以apply attributes。对于ordered类型的unbound workqueue，属于它的pool workqueue（worker thread pool）只能有一个，否则无法限制work是按照顺序执行。

2、分配内存并初始化

> pwq_tbl = kzalloc(nr_node_ids * sizeof(pwq_tbl[0]), GFP_KERNEL);  
> new_attrs = alloc_workqueue_attrs(GFP_KERNEL);  
> tmp_attrs = alloc_workqueue_attrs(GFP_KERNEL);  
> copy_workqueue_attrs(new_attrs, attrs);  
> cpumask_and(new_attrs->cpumask, new_attrs->cpumask, cpu_possible_mask);  
> copy_workqueue_attrs(tmp_attrs, new_attrs);

pwq_tbl数组用来保存unbound workqueue各个node的pool workqueue的指针，new_attrs和tmp_attrs都是一些计算workqueue attribute的中间变量，开始的时候设定为用户传入的workqueue的attribute。

3、如何为unbound workqueue的pool workqueue寻找对应的线程池？

具体的代码在get_unbound_pool函数中。本节不描述具体的代码，只说明基本原理，大家可以自行阅读代码。

per cpu的workqueue的pool workqueue对应的线程池也是per cpu的，每个cpu有两个线程池（normal和high priority），因此将pool workqueue和thread pool对应起来是非常简单的事情。对于unbound workqueue，对应关系没有那么直接，如果属性相同，多个unbound workqueue的pool workqueue可能对应一个thread pool。

系统使用哈希表来保存所有的unbound worker thread pool，定义如下：

> static DEFINE_HASHTABLE(unbound_pool_hash, UNBOUND_POOL_HASH_ORDER);

在创建unbound workqueue的时候，pool workqueue对应的worker thread pool需要在这个哈希表中搜索，如果有相同属性的worker thread pool的话，那么就不需要创建新的线程池，代码如下：

> hash_for_each_possible(unbound_pool_hash, pool, hash_node, hash) {  
>     if (wqattrs_equal(pool->attrs, attrs)) { －－－－检查属性是否相同  
>         pool->refcnt++;  
>         return pool; －－－－－－－在哈希表找到适合的unbound线程池  
>     }  
> }

如果没有相同属性的thread pool，那么需要创建一个并挂入哈希表。

4、给各个node分配pool workqueue并初始化

在进入代码之前，先了解一些基础知识。缺省情况下，挂入unbound workqueue的works最好是考虑NUMA Affinity，这样可以获取更好的性能。当然，实际上用户可以通过workqueue.disable_numa这个内核参数来关闭这个特性，这时候，系统需要一个default pool workqueue（workqueue_struct的dfl_pwq成员），所有的per node的pool workqueue指针都是执行default pool workqueue。

workqueue.disable_numa是enable的情况下是否不需要default pool workqueue了呢？也不是，我们举一个简单的例子，一个系统的构成是这样的：node 0中有CPU A和B，node 1中有CPU C和D，node 2中有CPU E和F，假设workqueue的attribute规定work只能在CPU A 和C上运行，那么在node 0和node 1中创建自己的pool workqueue是ok的，毕竟node 0中有CPU A，node 1中有CPU C，该node创建的worker thread可以在A或者C上运行。但是对于node 2节点，没有任何的CPU允许处理该workqueue的work，在这种情况下，没有必要为node 2建立自己的pool workqueue，而是使用default pool workqueue。

OK，我们来看代码：

> dfl_pwq = alloc_unbound_pwq(wq, new_attrs); －－－－－分配default pool workqueue
> 
> for_each_node(node) { －－－－遍历node  
>     if (wq_calc_node_cpumask(attrs, node, -1, tmp_attrs->cpumask)) { －－－是否使用default pool wq  
>         pwq_tbl[node] = alloc_unbound_pwq(wq, tmp_attrs); －－－该node使用自己的pool wq  
>     } else {  
>         dfl_pwq->refcnt++;  
>         pwq_tbl[node] = dfl_pwq; －－－－该node使用default pool wq  
>     }  
> }

值得一提的是wq_calc_node_cpumask这个函数，这个函数会根据该node的cpu情况以及workqueue attribute中的cpumask成员来更新tmp_attrs->cpumask，因此，在pwq_tbl[node] = alloc_unbound_pwq(wq, tmp_attrs); 这行代码中，为该node分配pool workqueue对应的线程池的时候，去掉了本node中不存在的cpu。例如node 0中有CPU A和B，workqueue的attribute规定work只能在CPU A 和C上运行，那么创建node 0上的pool workqueue以及对应的worker thread pool的时候，需要删除CPU C，也就是说，node 0上的线程池的属性中的cpumask仅仅支持CPU A了。

5、安装

所有的node的pool workqueue及其worker thread pool已经ready，需要安装到workqueue中了：

> for_each_node(node)  
>         pwq_tbl[node] = numa_pwq_tbl_install(wq, node, pwq_tbl[node]);   
>     link_pwq(dfl_pwq);  
>     swap(wq->dfl_pwq, dfl_pwq);

代码非常简单，这里就不细述了。

_原创文章，转发请注明出处。蜗窝科技_

标签: [alloc_workqueue](http://www.wowotech.net/tag/alloc_workqueue)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Concurrency Managed Workqueue之（四）：workqueue如何处理work](http://www.wowotech.net/irq_subsystem/queue_and_handle_work.html) | [Concurrency Managed Workqueue之（二）：CMWQ概述](http://www.wowotech.net/irq_subsystem/cmwq-intro.html)»

**评论：**

**ele**  
2019-05-22 14:17

@linuxer 请教下这种情况怎么处理：对于per cpu的workqueue，如果queue_work的cpu非常繁忙（大量中断或者有线程长时间占用），workqueque有可能很久才被执行吧？使用unbound workqueue是否是个好方法，由调度器调度到空闲cpu上。

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-7435)

**坚强的小孩**  
2024-01-05 23:02

@ele：以下是一些自己的看法，麻烦看看有什么不对的地方，希望大家指出来，谢谢。  
1、workqueue的调度其实还是依赖与进程调度吧，虽然是软中断触发的，但是调度还是进程的调度，所以不存在有大量的线程长时间占用，而调度不到workqueue的情况，只是workqueue会等待的时间比正常的时间长一点而已；  
2、大量的中断到来，导致系统频繁的处理中断程序，从而导致进程一直获取不到cpu，从而阻塞系统的正常运行（一般硬中断占用的时间不长，一般是soft irq占用资源长），不过内核，内部有检测，但发生大量的中断到来，大量触发软中断，系统会把soft irq进行线程化softirqd,这是是有进程的调度器接管系统资源，这个时候就是正常的进程调度，workqueue也还是能正常的进行调度  
3，虽然上面分析workqueue肯定会得到调用，但是关于是per workqueue和unbound workqueue哪一个性能好一点，这里确实不太清楚。

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-8856)

**SearchTheTruth**  
2018-11-14 16:58

对于文中开头WQ_POWER_EFFICIENT的讲解，贴一段原文帮助大家理解  
Workqueues marked with WQ_POWER_EFFICIENT are per-cpu by default  
but become unbound if workqueue.power_efficient kernel param is  
specified.  Per-cpu workqueues which are identified to  
contribute significantly to power-consumption are identified and  
marked with this flag and enabling the power_efficient mode  
leads to noticeable power saving at the cost of small  
performance disadvantage.

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-7037)

**xuwukong**  
2018-10-26 14:40

每次执行alloc_workqueue 创建一个bounded workqueue，都会要申请per cpu 的变量pool_workqueue,那如果系统一共有10个bounded workqueu，那是不是每个cpu 下面有10个对应的pool_workqueue变量副本？

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-7002)

**steven**  
2017-10-18 09:48

Hi wowo,  
请教几个问题：  
1.如果自己创建workqueue，用什么接口函数？  
2.使用系统默认的workqueue指的是不需要用create_workqueue函数去重新创建？只需要初始化一个work_struct,并用queue_work调度下就可以了？

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-6119)

**[wowo](http://www.wowotech.net/)**  
2017-10-18 13:22

@steven：1. 建议你看一下include/linux/workqueue.h，里面有各种alloc workqueue，然后queue_work就行了。  
2. 还是建议你看一下include/linux/workqueue.h，系统定义了一堆堆的默认workqueue，你可以根据需求使用。

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-6122)

**hhtrace**  
2017-05-22 15:43

最近在学习NUMA，有个问题不明白：  
比如一个驱动，使用的是per cpu workqueu，但是使用的内存却在node0上，那么workqueue最后会跑到node1上运行吗？  
  
谢谢

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-5581)

**hhtrace**  
2017-05-22 15:48

@hhtrace：是不是任意指定一个cpu运行？

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-5582)

**[linuxer](http://www.wowotech.net/)**  
2017-05-22 19:18

@hhtrace：如果一个驱动使用了per cpu workqueue，那么所有node上的所有的cpu都会有对应的pool_workqueue数据结构，进而有自己的线程池（worker_pool）。当queue work的时候，除非你指定cpu，否则会挂入local cpu对应的那个work list中。  
  
回到你的问题中来，使用per cpu workqueue的驱动，使用node0的内存，当queue work发生在node 1上的CPU的时候，那么该work会被node 1上的某个cpu上的worker线程处理。  
  
不过一般而言，我们是在该驱动的中断handler中queue work，那么问题来了，如果驱动使用的内存在node 0上，那么其irq为何送达node 1的cpu呢？这样驱动的设计其实就不合理了，应该将该驱动的irq affinity设定为node 0的cpu，这样，work就总在node 0上处理，从而不会影响性能。

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-5586)

**[chen_chuang](http://www.wowotech.net/)**  
2016-12-07 20:28

值得一提的是wq_calc_node_cpumask这个函数，这个函数会根据该node的cpu情况以及workqueue attribute中的cpumask成员来更新tmp_attrs->cpumask，因此，在pwq_tbl[node] = alloc_unbound_pwq(wq, tmp_attrs); 这行代码中，为该node分配pool workqueue对应的线程池的时候，去掉了本node中不存在的cpu。例如node 0中有CPU A和B，workqueue的attribute规定work只能在CPU A 和C上运行，那么创建node 0上的pool workqueue以及对应的worker thread pool的时候，需要删除CPU C，也就是说，node 0上的线程池的属性中的cpumask仅仅支持CPU A了。  
-------------------------------------------------------------------------------------------------  
大神，关于wq_calc_node_cpumask()函数的疑问。先贴下代码的实现：  
static bool wq_calc_node_cpumask(const struct workqueue_attrs *attrs, int node,  
                 int cpu_going_down, cpumask_t *cpumask)  
{  
    if (!wq_numa_enabled || attrs->no_numa)  
        goto use_dfl;  
  
    /* does @node have any online CPUs @attrs wants? */  
    cpumask_and(cpumask, cpumask_of_node(node), attrs->cpumask);  
    if (cpu_going_down >= 0)  
        cpumask_clear_cpu(cpu_going_down, cpumask);  
  
    if (cpumask_empty(cpumask))  
        goto use_dfl;  
  
    /* yeap, return possible CPUs in @node that @attrs wants */  
    cpumask_and(cpumask, attrs->cpumask, wq_numa_possible_cpumask[node]);  
    return !cpumask_equal(cpumask, attrs->cpumask);  
  
use_dfl:  
    cpumask_copy(cpumask, attrs->cpumask);  
    return false;  
}  
我的理解是：假设node1有cpu0和cpu1,node2有cpu2和cpu3，那么wq_numa_possible_cpumask[1] = 0011, wq_numa_possible_cpumask[2] = 1100;  
而如果unbound workqueue的work能在cpu0和cpu2上运行那这里的attr->cpumask即为0101，两者按位与后得出cpumask = 0001;这样的话cpumask_equal返回的值就为false了，但这样就直接走了dfl_pwq的分支了，感觉与您说的就不符了，不知道我哪理解错了

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-4989)

**[linuxer](http://www.wowotech.net/)**  
2016-12-08 10:14

@chen_chuang：在进入代码分析之前，我们先解释一下传入的参数：  
attrs：本次要创建的workqueue的属性  
node：当前节点  
cpu_going_down：与本场景无关  
cpumask：输出参数  
  
static bool wq_calc_node_cpumask(const struct workqueue_attrs *attrs, int node,  
                 int cpu_going_down, cpumask_t *cpumask)  
{  
如果系统没有打开unbound NUMA affinity选项（这个选项其实就是是否创建per node的pwq）或者该属性disable了NUMA affinity选项，那么就直接使用default好了。  
    if (!wq_numa_enabled || attrs->no_numa)  
        goto use_dfl;  
  
将该node的cpu mask（我们称之A）和属性中的cpu mask（我们称之B）相与，以此判断该node中是否和创建workqueue属性中的cpu mask有交集，结果保存在cpumask变量中  
    cpumask_and(cpumask, cpumask_of_node(node), attrs->cpumask);  
  
本场景传入-1，因此本段代码不执行。  
    if (cpu_going_down >= 0)  
        cpumask_clear_cpu(cpu_going_down, cpumask);  
  
如果A和B完全没有交集，那么使用default  
    if (cpumask_empty(cpumask))  
        goto use_dfl;  
  
下面是在有交集的情况下的处理  
    cpumask_and(cpumask, attrs->cpumask, wq_numa_possible_cpumask[node]);  
    return !cpumask_equal(cpumask, attrs->cpumask);－－如果A==B，我们要返回false，否则返回true。  
  
use_dfl:  
    cpumask_copy(cpumask, attrs->cpumask);  
    return false;  
}  
－－－－－－－－－－－－－－－－－－－－－－－－  
现在回到你说的例子，在你给出的例子中：  
1、A和B不相等  
2、A和B有交集  
这时候，wq_calc_node_cpumask返回true，创建per node的pwq，只有在A和B完全相等的情况下，wq_calc_node_cpumask返回false，使用default，因为这时候没有必要创建per node的pwq。

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-4991)

**[chen_chuang](http://www.wowotech.net/)**  
2016-12-08 14:24

@linuxer：谢谢大神讲解，不过我还是有几点点不太明白：  
1. 您所说的B有两处，一处是cpumask_of_node(node)，另一处是wq_numa_possible_cpumask[node]，这两者是等价的吗，因为我以arm为例，我查看到的cpumask_of_node(node)是cpu_online_mask。不知道这两者是什么关系。  
  
2. 在有交集的情况下，如果A==B，说明work只会在该node下运行，并且该node的任意一CPU均可处理，这样理解对吗。如果对的话，我觉得也应该创建per node，因为别的node不执行该work;如果不对，还望大神指点一二

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-4996)

**[linuxer](http://www.wowotech.net/)**  
2016-12-08 19:23

@chen_chuang：好吧，上面我的描述想偷一下懒，因此模糊了cpu电源管理的内容。为了方便描述，我们给出标记如下：  
1、cpumask_of_node(node)是B1，表示当前该节点online cpu的掩码。  
2、wq_numa_possible_cpumask[node]是B2，表示当前该节点possible cpu的掩码。  
当然，想要理解oneline cpu和possible cpu可以参考蜗窝网站的文章这里不再描述了，只要知道和cpu hotplug相关就OK了。  
  
OK，回到代码逻辑，在B1处，如果unbound wq的cpu mask和当前节点的online cpu没有交集，那么有必要创建per node的pwq吗？当然没有必要，例如：  
possible的cpu如下：  
node 0有cpu0、1、2、3  
node 1有cpu4、5、6、7  
online的cpu如下：  
node 0有cpu2、3  
node 1有cpu6、7  
unbound wq希望运行的cpu包括：  
node 0有cpu0、1  
node 1有cpu6、7  
对于这个unbound wq，在node 0上，虽然该wq期望在node 0的cpu0和cpu1上运行，但是这两个cpu没有oneline，即便为该node创建一个pwq其实也没有什么实际的意义，对于该node而言，即便创建了per node的线程池，也无法调度work在该node的cpu上运行，因为没有online的cpu。对于node 1，当然可以为其创建per node的pwq了，因为6、7cpu是oneline的。  
  
在B2处，代码逻辑是这样的，虽然有该node有oneline cpu，并且也是unbound wq打算运行其上的，但是是不是一定要创建per node的pwq呢？也不一定，例如：  
possible的cpu如下：  
node 0有cpu0、1、2、3  
node 1有cpu4、5、6、7  
online的cpu如下：  
node 0有cpu2、3  
node 1有cpu6、7  
unbound wq希望运行的cpu包括：  
node 1有cpu4、5、6、7  
  
在这种情况下，该unbound wq实际上是期望在node 1的任何一个cpu上运行而不在其他cpu上运行，这时候，创建per node pwq是没有意义的，使用default就OK了。  
  
当然，workqueue的代码也需要处理cpu hotplug的event，当cpu up或者down的时候，需要作出相应的调整。  
  
上面是我的理解。

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-5001)

**[chen_chuang](http://www.wowotech.net/)**  
2016-12-16 15:23

@linuxer：好的，谢谢大神，明白了。

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-5030)

**每天一小步**  
2020-08-21 20:48

@linuxer：-------------------  
在这种情况下，该unbound wq实际上是期望在node 1的任何一个cpu上运行而不在其他cpu上运行，这时候，创建per node pwq是没有意义的，使用default就OK了。  
------------------  
这里有个疑问，如果只想在node1上任何一个cpu运行而不在其它cpu上运行，为何创建per node pwq没有意义呢？ 使用deault的话不是就可能在其它cpu运行了吗？

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-8102)

**西人**  
2016-09-21 21:42

我的两个问题，请wowo大神指点，谢谢：  
（1）对于文中一段话，“workqueue中的成员包括两个类别：global类型和per thread pool类型的，我们把那些per thread pool  
  
类型的数据集合起来就形成了pool_workqueue的定义。”感觉是不是有歧义啊，这样感觉只有per thread pool类型的才能是  
  
pool_workqueue的  
  
（2）根据workqueue_attrs，一个workqueue_struct会对应一个worker_pool，pool_workqueue是中间桥梁，而一个pool_workqueue  
  
对应一个worker_pool，那么应该是一个workqueue_struct会对应一个pool_workqueue，为什么还会有workqueue_struct->pwqs这个  
  
链表来连接好几个pool_workqueue啊

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-4585)

**[wowo](http://www.wowotech.net/)**  
2016-09-22 08:46

@西人：这里是linuxer的主场哈:-)~~~~

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-4586)

**西人**  
2016-09-22 21:26

@wowo：linuxer大神，请指点一下吧，谢谢，上面的格式真乱，我重写两个问题：  
（1）对于文中一段话，“workqueue中的成员包括两个类别：global类型和per thread pool类型的，我们把那些per thread pool 类型的数据集合起来就形成了pool_workqueue的定义。”感觉是不是有歧义啊，这样感觉只有per thread pool类型的才能是 pool_workqueue的  
  
（2）根据workqueue_attrs，一个workqueue_struct会对应一个worker_pool，pool_workqueue是中间桥梁，而一个pool_workqueue 对应一个worker_pool，那么应该是一个workqueue_struct会对应一个pool_workqueue，为什么还会有workqueue_struct->pwqs这个链表来连接好几个pool_workqueue啊

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-4599)

**[linuxer](http://www.wowotech.net/)**  
2016-09-23 10:23

@西人：其实你问的问题是一个，只要能够理清workqueue、pool workqueue和worker thread pool以及worker thread之间的关系就OK了。  
  
在过去（非CMWQ），内核抽象了workqueue、cpu workqueue和worker thread三个数据结构来完成对workqueue逻辑的控制。那时候，这三个数据结构的关系非常简单，cpu workqueue和worker thread是一一对应，而workqueue和cpu workqueue关系也非常简单，系统有多少个cpu，那么workqueue就拥有多少个cpu workqueue。为何有cpu workqueue？无它，唯性能尔。因此，对于一个workqueue的实例，其数据有两种，一种是全局的，例如workqueue的名字，这个和哪个cpu没有关系。还有有一些数据是per cpu的，例如：worker thread，work list。上面说的是通常意义的workqueue，实际上还有一种single thread workqueue，但是内核在内存分配的时候，不管是否是single thread workqueue，其对应的cpu workqueue数据结构总是per cpu分配的（没错，简直是浪费），当然，实际上对于single thread workqueue而言，我们只会使用其中之一。  
  
来到CWMQ后，我们引入worker thread pool的概念，因此，workqueue只能看到worker thread pool这个层次，具体底层如何创建线程来处理work，它就不关心了。所以，在设计workqueue数据结构的时候，我们需要包括：  
（1）全局的数据成员，和哪一个worker thread pool没有关系。  
（2）和worker thread pool相关的数据  
如果workqueue和worker thread pool是一一对应的，那么struct workqueue_struct可以包括（1）和（2）中的成员，但是事实并非如此，一个workqueue可以对应多个worker thread pool，因此，（2）的那些数据成员被合成一个struct pool_workqueue的数据结构。  
  
对于bounded类型的workqueue，其对应的pool_workqueue是per cpu的（参考cpu_pwqs成员），对于unbounded类型的workqueue，其对应pool_workqueue是per node的（参考pwqs和numa_pwq_tbl）。  
  
BTW，我写文章的时候可能有些遣词不是那么规范，见谅，呵呵～～～

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-4600)

**西人**  
2016-09-23 13:28

@linuxer：谢谢您的回答，另外我不理解的一点是：  
就像您说的，“对于bounded类型的workqueue，其对应的pool_workqueue是per cpu的”，也就是此时的workqueue->cpu_pwqs是指向一个对应的pool_workqueue，那么此时的workqueue->pwqs这个链表中，是不是只有一个pool_workqueue，就是上面那个对应的pool_workqueue啊

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-4601)

**[linuxer](http://www.wowotech.net/)**  
2016-09-24 23:08

@西人：对于bounded类型的workqueue，其对应的pool_workqueue是per cpu的，如果系统中有N个cpu，那么就有N个pool_workqueue，一方面保存在workqueue->cpu_pwqs，此外，workqueue->pwqs链表中也会挂入N个pool_workqueue的节点。

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-4602)

**西人**  
2016-09-25 18:55

@linuxer：那我觉得既然已经在workqueue->cpu_pwqs有pool_workqueue的节点了，为什么还要有workqueue->pwqs这个链表来多此一举呢，感觉这不是重复了吗？对这点还不是很明白，难道有其他用途吗

**[linuxer](http://www.wowotech.net/)**  
2016-09-26 09:52

@linuxer：@西人，对于bounded类型的workqueue，其实不需要workqueue->pwqs这个链表也OK的，当需要遍历一个workqueue上的所有pool_workqueue节点的时候，通过workqueue->cpu_pwqs就OK了，但是，这仅限于bounded类型的workqueue，对于unbound类型的workqueue，其线程池不是per cpu的，workqueue->cpu_pwqs也就没有节点，只能是依靠workqueue->pwqs这个链表了。虽然这个设计看起来有些冗余，不过当flush或者destroy一个workqueue的时候，不需要区分workqueue的类型，直接使用for_each_pwq就可以遍历一个workqueue上的所有pool_workqueue了。

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-4605)

**西人**  
2016-09-26 10:38

@linuxer：哦，我明白了，谢谢linuxer不耐其烦的答惑，^_^

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-4606)

**zcjfish**  
2016-06-22 16:51

现在的ARM core都有hotplug功能，就是说如果loading很低的话，除了CPU0之外，CPU1， CPU2，CPU3...都会plugout, 那么非CPU0的per cpu workqueue thread会同步的被destory吗？   第二个问题是假设有4个CPU core,但是在创建per cpu workqueue thread时， 只有CPU0 &CPU1 在执行， 那么此时会创建CPU2&CPU3的per cpu workqueue thread吗？

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-4133)

**zcjfish**  
2016-06-22 16:52

@zcjfish：修正下第二个问题：  第二个问题是假设有4个CPU core,但是在创建per cpu workqueue thread时， 只有CPU0 &CPU1  online, CPU2&CPU3处于offline状态（plugout）， 那么此时会创建CPU2&CPU3的per cpu workqueue thread吗？

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-4134)

**[linuxer](http://www.wowotech.net/)**  
2016-06-23 09:34

@zcjfish：第二个问题是假设有4个CPU core,但是在创建per cpu workqueue thread时， 只有CPU0 &CPU1  online, CPU2&CPU3处于offline状态（plugout）， 那么此时会创建CPU2&CPU3的per cpu workqueue thread吗？  
--------------------------------------  
不会

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-4138)

**[madang](http://www.wowotech.net/)**  
2016-03-29 19:16

分别限定per cpu workqueue和unbound workqueue的最大可以创建的worker thread的数目  
//-------------------  
我怎么觉得好像是分别限定这个workqueue可以处理的work的数量 ？

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-3748)

**[hello_world](http://www.wowotech.net/)**  
2016-03-30 22:15

@madang：正常情况下，如果work不会引起阻塞，那么是不会创建新的worker thread的，也就是说，如果当前正在处理的work导致当前的worker thread进入sleep状态，那么新的worker thread会被创建，处理队列中的下一个work。在这种情况下，其实有两个worker thread是active的，当然，也可以说有两个work是active的（指正在处理），只不过其中一个是处于阻塞状态。  
  
因此，我们两个的说法是类似的，当然，如果更严密一些的话，你的表述可以修正为：  
....限定这个workqueue的active（或者说in flight，或者说“处理中”状态） work的数量

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-3758)

**c**  
2016-03-06 14:23

文中有一段话：  
“如果该workqueue是unbound类型的话，那么该work由unbound thread pool处理并把调度该work去哪一个CPU执行这样的策略交给系统的调度器模块来完成，对于scheduler而言，它会考虑CPU core的idle状态，从而尽可能的让CPU保持在idle状态，从而节省了功耗”  
cpu load balance尽可能的把进程分摊到负荷比较低的cpu去执行，所以unbound work会很频繁的唤醒处于idle的cpu？ 这个跟节能是不是矛盾了。

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-3594)

**[郭健](http://www.wowotech.net/)**  
2016-03-06 22:12

@c：负载平衡的算法本来就是要面对这样的矛盾：如果启动负载平衡的机制过于频繁，虽然负载时均衡了，但是cache 命中率会降低（从而影响性能和功耗）。

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-3596)

**c**  
2016-03-07 23:40

@郭健：我的疑点是： unbound work的代码注释说的是这种work是很save power的（即scheduler会尽量不唤醒idle的cpu，把这个work所在的线程调度到正在工作cpu的就绪队列上）。 然后实际的情况是，unbound work是被load balance管辖的，而load balance的策略是往idle cpu上push/pull任务。 这个跟上面的不唤醒idle cpu的解释是矛盾的。

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-3605)

**[郭健](http://www.wowotech.net/)**  
2016-03-08 08:41

@c：而load balance的策略是往idle cpu上push/pull任务  
----------------------------------------------  
我没有看过负载均衡的算法，不过我总觉得向idle cpu上压入任务而实现负载均衡有点过于粗暴，你确定linux kernel是这么做的吗？

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-3606)

**[passerby](http://www.wowotech.net/)**  
2016-05-11 21:09

@c：在linux的SMP调度算法里确实是这样，但是随着功耗在嵌入式设备中越来越受到重视，特别是HMP + EAS的出现，就让workqueue的power efficient的作用得到体现。以高通为例子，在worker thread被唤醒后，会调用select_task_rq_fair进行CPU选择。高通设计了自己的HMP调度，在这里面就会尽可能使用非idle CPU或者睡眠最浅的CPU，这样就实现了workqueue power efficient的初衷。在idle balance的时候，高通也会考虑到Idle cpu的情况，所以感觉必须是EAS这类的能量感知调度加入后才能让workqueue的作用体现出来。

[回复](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html#comment-3921)

1 [2](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html/comment-page-2#comments)

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
    
    - [蓝牙协议分析(1)_基本概念](http://www.wowotech.net/bluetooth/bt_overview.html)
    - [进程切换分析（3）：同步处理](http://www.wowotech.net/process_management/scheudle-sync.html)
    - [Linux PWM framework(1)_简介和API描述](http://www.wowotech.net/comm/pwm_overview.html)
    - [Linux kernel内核配置解析(1)_概述(基于ARM64架构)](http://www.wowotech.net/linux_kenrel/kernel_config_overview.html)
    - [ARM64 Kernel Image Mapping的变化](http://www.wowotech.net/memory_management/436.html)
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
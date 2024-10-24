
作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2015-8-17 19:41 分类：[中断子系统](http://www.wowotech.net/sort/irq_subsystem)

# 一、前言

本文主要讲述下面两部分的内容：

1、将work挂入workqueue的处理过程
2、如何处理挂入workqueue的work

# 二、用户将一个work挂入workqueue

## 1、queue_work_on函数

使用workqueue机制的模块可以调用queue_work_on（有其他变种的接口，这里略过，其实思路是一致的）将一个定义好的work挂入workqueue，具体代码如下：

```cpp
bool queue_work_on(int cpu, struct workqueue_struct wq, struct work_struct *work)  
{  
……

if (!test_and_set_bit(WORK_STRUCT_PENDING_BIT, work_data_bits(work))) {  
queue_work(cpu, wq, work);－－－挂入work list并通知worker thread pool来处理  
ret = true;  
}

……  
}
```

work_struct的data member中的WORK_STRUCT_PENDING_BIT这个bit标识了该work是处于pending状态还是正在处理中，pending状态的work只会挂入一次。大部分的逻辑都是在\_\_queue_work函数中，下面的小节都是描述该函数的执行过程。

## 2、\_\_WQ_DRAINING的解释

\_\_queue_work函数一开始会校验\_\_WQ_DRAINING这个flag，如下：

```cpp
if (unlikely(wq->flags & WQ_DRAINING) && WARN_ON_ONCE(!is_chained_work(wq)))  
return;
```

\_\_WQ_DRAINING这个flag表示该workqueue正在进行draining的操作，这多半是发送在销毁workqueue的时候，既然要销毁，那么挂入该workqueue的所有的work都要处理完毕，才允许它消亡。当想要将一个workqueue中所有的work都清空的时候，如果还有work挂入怎么办？一般而言，这时候当然是不允许新的work挂入了，毕竟现在的目标是清空workqueue中的work。但是有一种特例（通过is_chained_work判定），也就是正在清空的work（隶属于该workqueue）又触发了一个queue work的操作（也就是所谓chained work），这时候该work允许挂入。

## 3、选择pool workqueue

```cpp
if (req_cpu == WORK_CPU_UNBOUND)  
cpu = raw_smp_processor_id();

if (!(wq->flags & WQ_UNBOUND))  
pwq = per_cpu_ptr(wq->cpu_pwqs, cpu);  
else  
pwq = unbound_pwq_by_node(wq, cpu_to_node(cpu));
```

WORK_CPU_UNBOUND表示并不指定cpu，这时候，选择当前代码运行的那个cpu了。一旦确定了cpu了，对于非unbound的workqueue，当然使用per cpu的pool workqueue。如果是unbound的workqueue，那么要根据numa node id来选择。cpu_to_node可以从cpu id获取node id。需要注意的是：这里选择的pool wq只是备选的，可能用也可能不用，它有可能会被替换掉，具体参考下一节描述。

## 4、选择worker thread pool

与其说挂入workqueue，不如说挂入worker thread pool，因为毕竟是线程池来处理具体的work。pool_workqueue有一个相关联的worker thread pool（struct pool_workqueue的pool成员），因此看起来选择了pool wq也就选定了worker pool了，但是，不是当前选定的那个pool wq对应的worker pool就适合该work，因为有时候该work可能正在其他的worker thread上执行中，在这种情况下，为了确保work的callback function不会重入，该work最好还是挂在那个worker thread pool上，具体代码如下：

```cpp
last_pool = get_work_pool(work);  
if (last_pool && last_pool != pwq->pool) {  
struct worker worker;

spin_lock(&last_pool->lock);

worker = find_worker_executing_work(last_pool, work);

if (worker && worker->current_pwq->wq == wq) {  
pwq = worker->current_pwq;  
} else {  
/ meh... not running there, queue here /  
spin_unlock(&last_pool->lock);  
spin_lock(&pwq->pool->lock);  
}  
} else {  
spin_lock(&pwq->pool->lock);  
}
```

last_pool记录了上一次该work是被哪一个worker pool处理的，如果last_pool就是pool wq对应的worker pool，那么皆大欢喜，否则只能使用last pool了。使用last pool的例子比较复杂一些，因为这时候需要根据last worker pool找到对应的pool workqueue。find_worker_executing_work函数可以找到具体哪一个worker线程正在处理该work，如果没有找到，那么还是使用第3节中选定的pool wq吧，否则，选择该worker线程当前的那个pool workqueue（其实也就是选定了线程池）。

## 5、选择work挂入的队列

队列有两个，一个是被推迟执行的队列（pwq->delayed_works），一个是线程池要处理的队列（pwq->pool->worklist），如果挂入线程池要处理的队列，也就意味着该work进入active状态，线程池会立刻启动处理流程，如果挂入推迟执行的队列，那么该work还是pending状态：

> pwq->nr_in_flight\[pwq->work_color\]++;\
> work_flags = work_color_to_flags(pwq->work_color);
>
> if (likely(pwq->nr_active \< pwq->max_active)) {\
> pwq->nr_active++;\
> worklist = &pwq->pool->worklist;\
> } else {\
> work_flags |= WORK_STRUCT_DELAYED;\
> worklist = &pwq->delayed_works;\
> }
>
> insert_work(pwq, work, worklist, work_flags);

具体的挂入队列的动作是在insert_work函数中完成的。

## 6、唤醒idle的worker来处理该work

在insert_work函数中有下面的代码：

> if (\_\_need_more_worker(pool))\
> wake_up_worker(pool);

当线程池中正在运行状态的worker线程数目等于0的时候，说明需要wakeup线程池中处于idle状态的的worker线程来处理work。

# 三、线程池如何创建worker线程？

## 1、per cpu worker pool什么时候创建worker线程？

对于per-CPU workqueue，每个cpu有两个线程池，一个是normal，一个是high priority的。在初始化函数init_workqueues中有对这两个线程池的初始化：

> for_each_online_cpu(cpu) {\
> struct worker_pool \*pool;
>
> for_each_cpu_worker_pool(pool, cpu) {\
> pool->flags &= ~POOL_DISASSOCIATED;\
> BUG_ON(!create_worker(pool));\
> }\
> }

因此，在系统初始化的时候，per cpu workqueue共享的那些线程池（2 x cpu nr）就会通过create_worker创建一个initial worker。

一旦initial worker启动，该线程会执行worker_thread函数来处理work，在处理过程中，如果有需要， worker会创建新的线程。

## 2、unbound thread pool什么时候创建worker线程？

我们先看看unbound thread pool的建立，和per-CPU不同的是unbound thread pool是全局共享的，因此，每当创建不同属性的unbound workqueue的时候，都需要创建pool_workqueue及其对应的worker pool，这时候就会调用get_unbound_pool函数在当前系统中现存的线程池中找是否有匹配的worker pool，如果没有就需要创建新的线程池。在创建新的线程池之后，会立刻调用create_worker创建一个initial worker。和per cpu worker pool一样，一旦initial worker启动，随着work不断的挂入以及worker处理work的具体情况，线程池会动态创建worker。

## 3、如何创建worker。代码如下：

> static struct worker \*create_worker(struct worker_pool \*pool)\
> {\
> struct worker \*worker = NULL;\
> int id = -1;\
> char id_buf\[16\];
>
> id = ida_simple_get(&pool->worker_ida, 0, 0, GFP_KERNEL);－－－－分配ID
>
> worker = alloc_worker(pool->node);－－－－－分配worker struct的内存
>
> worker->pool = pool;\
> worker->id = id;
>
> if (pool->cpu >= 0)－－－－－－－－－worker的名字\
> snprintf(id_buf, sizeof(id_buf), "%d:%d%s", pool->cpu, id,  pool->attrs->nice \< 0  ? "H" : "");\
> else\
> snprintf(id_buf, sizeof(id_buf), "u%d:%d", pool->id, id);
>
> worker->task = kthread_create_on_node(worker_thread, worker, pool->node,   "kworker/%s", id_buf);
>
> set_user_nice(worker->task, pool->attrs->nice); －－－创建task并设定nice value\
> worker->task->flags |= PF_NO_SETAFFINITY; \
> worker_attach_to_pool(worker, pool); －－－－－建立worker和线程池的关系
>
> spin_lock_irq(&pool->lock);\
> worker->pool->nr_workers++;\
> worker_enter_idle(worker);\
> wake_up_process(worker->task);－－－－－－让worker运行起来\
> spin_unlock_irq(&pool->lock);
>
> return worker;\
> }

代码不复杂，通过线程池（struct worker_pool）绑定的cpu信息（struct worker_pool的cpu成员）可以知道该pool是per-CPU还是unbound，对于per-CPU线程池，pool->cpu是大于等于0的。对于对于per-CPU线程池，其worker线程的名字是kworker/_cpu_：_worker id_，如果是high priority的，后面还跟着一个H字符。对于unbound线程池，其worker线程的名字是kworker/u _pool id_：_worker id。_

# 四、work的处理

本章主要描述worker_thread函数的执行流程，部分代码有删节，保留主干部分。

## 1、PF_WQ_WORKER标记

worker线程函数一开始就会通过PF_WQ_WORKER来标注自己：

> worker->task->flags |= PF_WQ_WORKER;

有了这样一个flag，调度器在调度当前进程sleep的时候可以检查这个准备sleep的进程是否是一个worker线程，如果是的话，那么调度器不能鲁莽的调度到其他的进程，这时候，还需要找到该worker对应的线程池，唤醒一个idle的worker线程。通过workqueue模块和调度器模块的交互，当work A被阻塞后（处理该work的worker线程进入sleep），调度器会唤醒其他的worker线程来处理其他的work B，work C……

## 2、管理线程池中的线程

> recheck:\
> if (!need_more_worker(pool))\
> goto sleep;
>
> if (unlikely(!may_start_working(pool)) && manage_workers(worker))\
> goto recheck;

如何判断是否需要创建更多的worker线程呢？原则如下：

（1）有事情做：挂在worker pool中的work list不能是空的，如果是空的，那么当然sleep就好了

（2）比较忙：worker pool的nr_running成员表示线程池中当前正在干活（running状态）的worker线程有多少个，当nr_running等于0表示所有的worker线程在处理work的时候阻塞了，这时候，必须要启动新的worker线程来处理worker pool上处于active状态的work链表上的work们。

## 3、worker线程开始处理work

> worker_clr_flags(worker, WORKER_PREP | WORKER_REBOUND);
>
> do {\
> struct work_struct \*work =   list_first_entry(&pool->worklist,  struct work_struct, entry);
>
> if (likely(!(\*work_data_bits(work) & WORK_STRUCT_LINKED))) {\
> process_one_work(worker, work);\
> if (unlikely(!list_empty(&worker->scheduled)))\
> process_scheduled_works(worker);\
> } else {\
> move_linked_works(work, &worker->scheduled, NULL);\
> process_scheduled_works(worker);\
> }\
> } while (keep_working(pool));
>
> worker_set_flags(worker, WORKER_PREP);

按理说worker线程处理work应该比较简单，从线程池的worklist中取一个work，然后调用process_one_work处理之就OK了，不过现实稍微复杂一些，work和work之间并不是独立的，也就是说，work A和work B可能是linked work，这些linked work应该被一个worker来处理。WORK_STRUCT_LINKED标记了work是属于linked work，如果是linked work，worker并不直接处理，而是将其挂入scheduled work list，然后调用process_scheduled_works来处理。毫无疑问，process_scheduled_works也是调用process_one_work来处理一个一个scheduled work list上的work。

scheduled work list并非仅仅应用在linked work，在worker处理work的时候，有一个原则要保证：同一个work不能被同一个cpu上的多个worker同时执行。这时候，如果worker发现自己要处理的work正在被另外一个worker线程处理，那么本worker线程将不处理该work，只需要挂入正在执行该work的worker线程的scheduled work list即可。

_原创文章，转发请注明出处。蜗窝科技_

标签: [workqueue](http://www.wowotech.net/tag/workqueue) [Concurrency](http://www.wowotech.net/tag/Concurrency) [Managed](http://www.wowotech.net/tag/Managed)

---

« [linux cpufreq framework(4)\_cpufreq governor](http://www.wowotech.net/pm_subsystem/cpufreq_governor.html) | [Concurrency Managed Workqueue之（三）：创建workqueue代码分析](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html)»

**评论：**

**www**\
2019-10-06 13:52

对于不同的bound的workqueue，最终大家都要添加到同样的worker pool的list上，而在一个worker pool上只会有一个active的线程处理事件，是否意味着驱动根本没有必要自己去创建workqueue，因为通过queue_work将work添加到自己创建的workqueue和添加到default的workqueue都同样要在同一个worker pool的list上排队，和直接queue_work到default的workqueue上效果一样，即不会并行处理。\
是否是我什么地方的理解有问题？

谢谢

[回复](http://www.wowotech.net/irq_subsystem/queue_and_handle_work.html#comment-7684)

**[bsp](http://www.wowotech.net/)**\
2021-01-04 11:33

@www：我觉得前半部分理解没有问题，从日志也可以看到：\
echo t >/proc/sysrq-trigger; dmesg\
\[  367.575510\] Showing busy workqueues and worker pools:\
\[  367.575513\] workqueue events: flags=0x0\
\[  367.575520\] pwq 4: cpus=0 node=0 flags=0x0 nice=0 active=10/256\
\[  367.575530\] in-flight: 9297 :mxt_work\
\[  367.576205\] pending: mxt_work, watchdog_work, \*\*\*\
这些work都是挂在到同一个pool中的。\
但并行处理是会的，有时候worker_pool太忙了，就会创建新的work-thread去并行处理多个work，这在Concurrency Managed Workqueue之（三/二）有介绍。

[回复](http://www.wowotech.net/irq_subsystem/queue_and_handle_work.html#comment-8171)

**cade**\
2017-07-20 20:57

1. “work item” 是什么？“work item" 指work_struct 这个结构体，

ret = queue_work(lsm_workqueue, &sdata->input_work);\
if (!ret)\
LOG_INFO("work was already on the queue.\\n");\
\
如果上次提交的work没有执行，而再次调用queue_work，则这个应该是同一个”work item“\
还是不同的”work item“?\
即同一个work_struct 是否可以被提交多次，处于pending状态的work就没法再提交了？\
我的需求是每10ms执行一次work，即100Hz，读数据并上报给用户空间，\
通过加log发现好像是如果上次提交的work没有执行完，则无法提交；造成的结果就是在上层看来丢数据了，\
读数据的频率不是100Hz。\
但我觉得不应该这样设计的，不是可以一直提交才更合理吗，类似链表管理一个work_struct的各个提交项目\
请帮忙解答下，谢谢~

2. 关于 @max_active:\
   如下解释\
   @max_active determines the maximum number of execution contexts per\
   CPU which can be assigned to the work items of a wq.  For example,\
   with @max_active of 16, at most 16 work items of the wq can be\
   executing at the same time per CPU.

我的场景是driver中创建一个workqueue_struct，有两个work_struct A和B，A和B的执行逻辑一样，对应同一个call back\
所以问题是：\
lsm_workqueue = alloc_workqueue("%s", WQ_HIGHPRI | WQ_UNBOUND | WQ_MEM_RECLAIM, 2, "lsm_wq");\
这个参数max_active应该传入2 更合适吗？

[回复](http://www.wowotech.net/irq_subsystem/queue_and_handle_work.html#comment-5829)

**[bsp](http://www.wowotech.net/)**\
2021-01-05 09:45

@cade：1，相同的work是可以提交多次的，确实有pool->worklist来管理待处理（pending）的work，从log也可以看到：\
echo t >/proc/sysrq-trigger; dmesg\
\[  367.575510\] Showing busy workqueues and worker pools:\
\[  367.575513\] workqueue events: flags=0x0\
\[  367.575520\] pwq 4: cpus=0 node=0 flags=0x0 nice=0 active=10/256\
\[  367.575530\] in-flight: 9297 :mxt_work\
\[  367.576205\] pending: mxt_work, vmstat_shepherd, tsens_poll, tsens_poll, sdhci_msm_pm_qos_cpu_unvote_work, sdhci_msm_pm_qos_cpu_unvote_work, sdhci_msm_pm_qos_irq_unvote_work\
这里有两个tsens_poll，两个sdhci_msm_pm_qos_cpu_unvote_work。\
至于丢包，那可能是pending太久上层认为timeout丢掉了。\
2，关于max_active，Documentation//core-api/workqueue.rst有很好的解释和例子。对于你这例子，传入默认的0最合适，系统会设为256（512/2）。\
The following example execution scenarios try to illustrate how cmwq\
behave under different configurations.

Work items w0, w1, w2 are queued to a bound wq q0 on the same CPU.\
w0 burns CPU for 5ms then sleeps for 10ms then burns CPU for 5ms\
again before finishing.  w1 and w2 burn CPU for 5ms then sleep for\
10ms.

Ignoring all other tasks, works and processing overhead, and assuming\
simple FIFO scheduling, the following is one highly simplified version\
of possible sequences of events with the original wq. ::

TIME IN MSECS  EVENT\
0              w0 starts and burns CPU\
5              w0 sleeps\
15             w0 wakes up and burns CPU\
20             w0 finishes\
20             w1 starts and burns CPU\
25             w1 sleeps\
35             w1 wakes up and finishes\
35             w2 starts and burns CPU\
40             w2 sleeps\
50             w2 wakes up and finishes

And with cmwq with `@max_active` >= 3, ::

TIME IN MSECS  EVENT\
0              w0 starts and burns CPU\
5              w0 sleeps\
5              w1 starts and burns CPU\
10             w1 sleeps\
10             w2 starts and burns CPU\
15             w2 sleeps\
15             w0 wakes up and burns CPU\
20             w0 finishes\
20             w1 wakes up and finishes\
25             w2 wakes up and finishes

If `@max_active` == 2, ::\
......

[回复](http://www.wowotech.net/irq_subsystem/queue_and_handle_work.html#comment-8177)

**lamaboy**\
2017-06-12 17:51

你好！ 在看到如下代码：

static struct worker \*create_worker(struct worker_pool \*pool)\
{\
...\
worker->task = kthread_create_on_node(worker_thread, worker, pool->node,\
"kworker/%s", id_buf);\
...\
}

每一个线程创建的用的是同一个函数，这样，会不会引入重入的问题，是不是，只要我保证这个回调函数是线程安全的，不管在kernel、user 空间懂不会有问题，，希望博主解惑！！

[回复](http://www.wowotech.net/irq_subsystem/queue_and_handle_work.html#comment-5658)

**lamaboy**\
2017-06-12 17:57

@lamaboy：还有，如果没有问题？？ 我在申请中断的时候，如gpio_key ，我要申请10 个中断，用的是统一个函数，在什么情况下会有什么问题呢？？

[回复](http://www.wowotech.net/irq_subsystem/queue_and_handle_work.html#comment-5659)

**[linuxer](http://www.wowotech.net/)**\
2017-06-12 19:06

@lamaboy：多个irq number对应一个interrupt handler也是没有什么问题，当然需要仔细考虑清楚内核不同问题（也是保证重入安全的）。

[回复](http://www.wowotech.net/irq_subsystem/queue_and_handle_work.html#comment-5662)

**[linuxer](http://www.wowotech.net/)**\
2017-06-12 18:58

@lamaboy：同意你的说法。重入问题在kernel space和userspace都是一样的，解决的方法就是：\
1、全部访问thread local资源\
2、在thread之间共享的资源使用锁机制保护

你可以看看worker_thread函数实现，显然，它使用了方法二来保证了多个thread重入是安全的。

[回复](http://www.wowotech.net/irq_subsystem/queue_and_handle_work.html#comment-5661)

**lamaboy**\
2017-06-12 17:29

追代码的时候发现，当前工作队列线程休眠，在找不到空闲的线程的情况下 ：如to_wakeup == null，是如何处理的呢？？ 具体代码如下，\
if (prev->flags & PF_WQ_WORKER) {\
struct task_struct \*to_wakeup;

to_wakeup = wq_worker_sleeping(prev); // 如果当前一休眠，\
if (to_wakeup)\
try_to_wake_up_local(to_wakeup, cookie);\
}\
.//////////////

struct task_struct \*wq_worker_sleeping(struct task_struct \*task)\
{\
struct worker \*worker = kthread_data(task), \*to_wakeup = NULL;\
struct worker_pool \*pool;

/\*\
\* Rescuers, which may not have all the fields set up like normal\
\* workers, also reach here, let's not access anything before\
\* checking NOT_RUNNING.\
\*/\
if (worker->flags & WORKER_NOT_RUNNING)\
return NULL;

pool = worker->pool;

/\* this can only happen on the local cpu \*/\
if (WARN_ON_ONCE(pool->cpu != raw_smp_processor_id()))\
return NULL;

/\*\
\* The counterpart of the following dec_and_test, implied mb,\
\* worklist not empty test sequence is in insert_work().\
\* Please read comment there.\
\*\
\* NOT_RUNNING is clear.  This means that we're bound to and\
\* running on the local cpu w/ rq lock held and preemption\
\* disabled, which in turn means that none else could be\
\* manipulating idle_list, so dereferencing idle_list without pool\
\* lock is safe.\
\*/\
if (atomic_dec_and_test(&pool->nr_running) &&\
!list_empty(&pool->worklist))\
to_wakeup = first_idle_worker(pool);  // 找不到空闲的！！\
return to_wakeup ? to_wakeup->task : NULL;\
}

[回复](http://www.wowotech.net/irq_subsystem/queue_and_handle_work.html#comment-5656)

**[linuxer](http://www.wowotech.net/)**\
2017-06-12 18:04

@lamaboy：如果to_wakeup等于null，说明当前线程池中已经有足够的worker线程了，不需要唤醒多余的worker thread了。一个典型的例子是挂在worker pool中的work list是空的，如果是空的，那么worker thread阻塞就阻塞吧，反正也没有什么活要干。

[回复](http://www.wowotech.net/irq_subsystem/queue_and_handle_work.html#comment-5660)

**[nimisolo](http://www.wowotech.net/)**\
2017-03-28 16:08

请问，一个pool的多个work能并发处理吗？\
例如：一个unbound work pool，目前有100个work需要处理，有5个worker，这5个worker能够同时运行并处理一部分work吗？\
我看worker_thread中处理一个work前会首先获得pool->lock，处理好后再释放它，这是不是说明一个pool多个worker无法并发运行？

[回复](http://www.wowotech.net/irq_subsystem/queue_and_handle_work.html#comment-5389)

**[nimisolo](http://www.wowotech.net/)**\
2017-03-28 16:52

@nimisolo：解决了...\
process_one_work中在处理work之前会释放pool->lock，所以可以的

[回复](http://www.wowotech.net/irq_subsystem/queue_and_handle_work.html#comment-5390)

**michael**\
2017-03-22 15:24

Hi, linuxer

有几个问题要咨询一下:\
在旧机制下的workqueue，在workqueue下，一个workqueue可能包含了多个work。比如，当前正在执行work A时，被调度了，让出了CPU，那么workqueue中的其它work，可以在其它CPU上执行吗，还是说也被阻塞了？

[回复](http://www.wowotech.net/irq_subsystem/queue_and_handle_work.html#comment-5356)

**[linuxer](http://www.wowotech.net/)**\
2017-03-22 16:28

@michael：这分两种情况：\
（1）对于single threaded的workqueue，任何的work都是排队执行。如果workqueue中挂入work A B C，那么如果A阻塞，那么B和C也是会等待，直到A work完成。\
（2）对于multi threaded（更准确的是per-CPU threaded）情况当然会好一些，因为该workqueue会为每一个online cpu创建一个线程来处理work。对于一个cpu上的线程，其处理逻辑等于single threaded的workqueue，即一个work阻塞了，挂入该cpu的work都会阻塞，但是其他cpu的线程处理是不受影响的。

[回复](http://www.wowotech.net/irq_subsystem/queue_and_handle_work.html#comment-5357)

**lucky**\
2016-10-04 12:12

Hi linuxer

@linuxer\
你之前的文章关于work同步的解释如下：\
=>挂入到multi thread或者说per cpu workqueue上的指定的work，其callback是不会在一个cpu上并发执行（也就是说在多个cpu上可以并发执行）

此篇文中有这样一段话：“不是当前选定的那个pool wq对应的worker pool就适合该work，因为有时候该work可能正在其他的worker thread上执行中，在这种情况下，为了确保work的callback function不会重入，该work最好还是挂在那个worker thread pool上”\
=>这一段的意思是：

1. 对于per cpu来讲保证同一个work不能在同一个cpu对应的worker pool里的多个worker thread里面同时执行，但可以在不同的cpu 的worker pool里的worker thread执行。
1. 对于UNBOUND的情况，可以将属于同一个node id的若干个个cpu视作一个逻辑cpu，保证同一个逻辑cpu里面不会出现多个worker thread执行一个相同的work，不同的node id（即不同的逻辑cpu）里面的worker thread是可以同时执行一个相同的work的。

我这样理解是正确的吗？如有偏差请指正，谢谢！

[回复](http://www.wowotech.net/irq_subsystem/queue_and_handle_work.html#comment-4643)

**[linuxer](http://www.wowotech.net/)**\
2016-10-06 19:04

@lucky：的确，我之前在回复某个网友的提问的时候说到：挂入到multi thread或者说per cpu workqueue上的指定的work，其callback是不会在一个cpu上并发执行（也就是说在多个cpu上可以并发执行）。不过这句话是有条件限制的，当时，我的描述是针对旧的workqueue上的并发（参考的是linux2.6.23的workqueue代码）。

对于新的内核，例如4.4.6的内核，不论是bound（per cpu）或者unbounded workqueue，其work的处理都遵守下面的处理原则：在给定的时间点，同一个work只能被系统中的一个worker线程处理，也就是说，任何的work都是non-reentrant的。

[回复](http://www.wowotech.net/irq_subsystem/queue_and_handle_work.html#comment-4653)

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

  - [CFS调度器（1）-基本原理](http://www.wowotech.net/process_management/447.html)
  - [DRAM 原理 5 ：DRAM Devices Organization](http://www.wowotech.net/basic_tech/343.html)
  - [Concurrency Managed Workqueue之（一）：workqueue的基本概念](http://www.wowotech.net/irq_subsystem/workqueue.html)
  - [DMB DSB ISB以及SMP CPU 乱序](http://www.wowotech.net/77.html)
  - [X-016-KERNEL-串口驱动开发之驱动框架](http://www.wowotech.net/x_project/serial_driver_porting_1.html)

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

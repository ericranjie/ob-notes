 作者：[smcdef](http://www.wowotech.net/author/531) 发布于：2018-11-10 20:43 分类：[进程管理](http://www.wowotech.net/sort/process_management)

## 前言

现在的计算机基本都支持多用户登陆。如果一台计算机被两个用户A和B使用。假设用户A运行9个进程，用户B只运行1个进程。按照之前文章对CFS调度器的讲解，我们认为用户A获得90% CPU时间，用户B只获得10% CPU时间。随着用户A不停的增加运行进程，用户B可使用的CPU时间越来越少。这显然是不公平的。因此，我们引入组调度（Group Scheduling ）的概念。我们以用户组作为调度的单位，这样用户A和用户B各获得50% CPU时间。用户A中的每个进程分别获得5.5%（50%/9）CPU时间。而用户B的进程获取50% CPU时间。这也符合我们的预期。本篇文章讲解CFS组调度实现原理。

> 注：代码分析基于Linux 4.18.0。使能组调度需要配置CONFIG_CGROUPS和CONFIG_FAIR_GROUP_SCHED。图片变形了该怎么办？浏览器单独打开吧！我也没办法![](http://www.wowotech.net/admin/editor/plugins/emoticons/images/2.gif)

## 再谈调度实体

通过之前的文章，我们已经介绍了CFS调度器主要管理的是调度实体。每一个进程通过`task_struct`描述，`task_struct`包含调度实体`sched_entity`参与调度。暂且针对这种调度实体，我们称作**task se**。现在引入组调度的概念，我们使用`task_group`描述一个组。在这个组中管理组内的所有进程。因为CFS就绪队列管理的单位是调度实体，因此，`task_group`也脱离不了`sched_entity`，所以在`task_group`结构体也包含调度实体`sched_entity`，我们称这种调度实体为**group se**。`task_group`定义在`kernel/sched/sched.h`文件。

```c
 1. `struct task_group {`
2. `struct cgroup_subsys_state css;`
3. `#ifdef CONFIG_FAIR_GROUP_SCHED`
4. `/* schedulable entities of this group on each CPU */`
5. `struct sched_entity **se; /* 1 */`
6. `/* runqueue "owned" by this group on each CPU */`
7. `struct cfs_rq **cfs_rq; /* 2 */`
8. `unsigned long shares; /* 3 */`
9. `#ifdef CONFIG_SMP`
10. `atomic_long_t load_avg ____cacheline_aligned; /* 4 */`
11. `#endif`
12. `#endif`
13. `struct cfs_bandwidth cfs_bandwidth;`
14. `/* ... */`
15. `};`
```

> 1. 指针数组，数组大小等于CPU数量。现在假设只有一个CPU的系统。我们将一个用户组也用一个调度实体代替，插入对应的红黑树。例如，上面用户组A和用户组B就是两个调度实体se，挂在顶层的就绪队列cfs_rq中。用户组A管理9个可运行的进程，这9个调度实体se作为用户组A调度实体的child。通过se->parent成员建立关系。用户组A也维护一个就绪队列cfs_rq，暂且称之为group cfs_rq，管理的9个进程的调度实体挂在group cfs_rq上。当我们选择进程运行的时候，首先从根就绪队列cfs_rq上选择用户组A，再从用户组A的group cfs_rq上选择其中一个进程运行。现在考虑多核CPU的情况，用户组中的进程可以在多个CPU上运行。因此，我们需要CPU个数的调度实体se，分别挂在每个CPU的根cfs_rq上。
> 2. 上面提到的group cfs_rq，同样是指针数组，大小是CPU的数量。因为每一个CPU上都可以运行进程，因此需要维护CPU个数的group cfs_rq。
> 3. 调度实体有权重的概念，以权重的比例分配CPU时间。用户组同样有权重的概念，share就是`task_group`的权重。
> 4. 整个用户组的负载贡献总和。

如果我们CPU数量等于2，并且只有一个用户组，那么系统中组调度示意图如下。

[![group_scheduling1.png](http://www.wowotech.net/content/uploadfile/201811/a2ef1541854244.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201811/a2ef1541854244.png)

 系统中一共运行8个进程。CPU0上运行3个进程，CPU1上运行5个进程。其中包含一个用户组A，用户组A中包含5个进程。CPU0上group se获得的CPU时间为group se对应的group cfs_rq管理的所有进程获得CPU时间之和。系统启动后默认有一个`root_task_group`，管理系统中最顶层CFS就绪队列cfs_rq。在2个CPU的系统上，`task_group`结构体se和cfs_rq成员数组长度是2，每个group se都对应一个group cfs_rq。

## 数据结构之间的关系

假设系统包含4个CPU，组调度的打开的情况下，各种结构体之间的关系如下图。[![group_schedule_data structure.png](http://www.wowotech.net/content/uploadfile/201811/1f8a1541854243.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201811/1f8a1541854243.png)

  

 在每个CPU上都有一个全局的就绪队列`struct rq`，在4个CPU的系统上会有4个全局就绪队列，如图中紫色结构体。系统默认只有一个根`task_group`叫做root_task_group。`rq->cfs_rq`指向系统根CFS就绪队列。根CFS就绪队列维护一棵红黑树，红黑树上一共10个就绪态调度实体，其中9个是task se，1个group se（图上蓝色se）。group se的`my_q`成员指向自己的就绪队列。该就绪队列的红黑树上共9个task se。其中`parent`成员指向group se。每个group se对应一个group cfs_rq。4个CPU会对应4个group se和group cfs_rq，分别存储在`task_group`结构体se和cfs_rq成员。`se->depth`成员记录se嵌套深度。最顶层CFS就绪队列下的se的深度为0，group se往下一层层递增。`cfs_rq->nr_runing`成员记录CFS就绪队列所有调度实体个数，不包含子就绪队列。`cfs_rq->h_nr_running`成员记录就绪队列层级上所有调度实体的个数，包含group se对应group cfs_rq上的调度实体。例如，图中上半部，nr_runing和h_nr_running的值分别等于10和19，多出的9是group cfs_rq的h_nr_running。group cfs_rq由于没有group se，因此nr_runing和h_nr_running的值都等于9。

## 组进程调度

用户组内的进程该如何调度呢？通过上面的分析，我们可以通过根CFS就绪队列一层层往下便利选择合适进程。例如，先从根就绪队列选择适合运行的group se，然后找到对应的group cfs_rq，再从group cfs_rq上选择task se。在CFS调度类中，选择进程的函数是pick_next_task_fair()。

```c
 
static struct task_struct *pick_next_task_fair(struct rq *rq, struct task_struct *prev, struct rq_flags *rf){	struct cfs_rq *cfs_rq = &rq->cfs;               /* 1 */	struct sched_entity *se;	struct task_struct *p; 	put_prev_task(rq, prev);	do {		se = pick_next_entity(cfs_rq, NULL);        /* 2 */		set_next_entity(cfs_rq, se);		cfs_rq = group_cfs_rq(se);                  /* 3 */	} while (cfs_rq);                               /* 4 */ 	p = task_of(se); 	return p;}
```

> 1. 从根CFS就绪队列开始便利。
> 2. 从就绪队列cfs_rq的红黑树中选择虚拟时间最小的se。
> 3. group_cfs_rq()返回`se->my_q`成员。如果是task se，那么group_cfs_rq()返回NULL。如果是group se，那么group_cfs_rq()返回group se对应的group cfs_rq。
> 4. 如果是group se，我们需要从group cfs_rq上的红黑树选择下一个虚拟时间最小的se，以此循环直到最底层的task se。

## 组进程抢占

周期性调度会调用task_tick_fair()函数。

```c
 
static void task_tick_fair(struct rq *rq, struct task_struct *curr, int queued){	struct cfs_rq *cfs_rq;	struct sched_entity *se = &curr->se; 	for_each_sched_entity(se) {		cfs_rq = cfs_rq_of(se);		entity_tick(cfs_rq, se, queued);	}}
```

for_each_sched_entity()是一个宏定义`for (; se; se = se->parent)`，顺着se的parent链表往上走。entity_tick()函数继续调用check_preempt_tick()函数，这部分在之前的文章已经说过了。check_preempt_tick()函数会根据满足抢占当前进程的条件下设置**TIF_NEED_RESCHED**标志位。满足抢占条件也很简单，只要顺着`se->parent`这条链表便利下去，如果有一个se运行时间超过分配限额时间就需要重新调度。

## 用户组的权重

每一个进程都会有一个权重，CFS调度器依据权重的大小分配CPU时间。同样`task_group`也不例外，前面已经提到使用share成员记录。按照前面的举例，系统有2个CPU，`task_group`中势必包含两个group se和与之对应的group cfs_rq。这2个group se的权重按照比例分配`task_group`权重。如下图所示。[![weight_time_slice.png](http://www.wowotech.net/content/uploadfile/201811/a0011541854246.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201811/a0011541854246.png)

  

 CPU0上group se下有2个task se，权重和是3072。CPU1上group se下有3个task se，权重和是4096。`task_group`权重是1024。因此，CPU0上group se权重是439（1024*3072/(3072+4096)）,CPU1上group se权重是585（1024-439）。当然这里的计算group se权重的方法是最简单的方式，代码中实际计算公式是考虑每个group cfs_rq的负载贡献比例，而不是简单的考虑权重比例。

## 用户组时间限额分配

  

分配给每个进程时间计算函数是sched_slice()，之前的分析都是基于不考虑组调度的情况下。现在考虑组调度的情况下进程应该分配的时间如何调整呢？先举个简单不考虑组调度的例子，在一个单核系统上2个进程，权重都是1024。在不考虑组调度的情况下，调度实体se分配的时间限额计算公式如下：

1.                            se->load.weight
2. time = sched_period * -------------------------
3.                          cfs_rq->load.weight 

我们还需要计算se的权重占整个CFS就绪队列权重的比例乘以调度周期时间即可。2个进程根据之前文章的分析，调度周期是6ms，那么每个进程分配的时间是6ms*1024/(1024+1024)=3ms。

现在考虑组调度的情况。系统依然是单核，存在一个`task_group`，所有的进程权重是1024。`task_group`权重也是1024（即share值）。如下图所示。

[![time_slice_weight.png](http://www.wowotech.net/content/uploadfile/201811/abcf1542200351.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201811/abcf1542200351.png) 

 group cfs_rq下的进程分配时间计算公式如下（gse := group se; gcfs_rq := group cfs_rq）：

1.                            se->load.weight            gse->load.weight 
2. time = sched_period * ------------------------- * ------------------------
3.                          gcfs_rq->load.weight        cfs_rq->load.weight

 

根据公式，计算group cfs_rq下进程的配时间如下：

```txt
 
                   1024              1024 time = 6ms * --------------- * ---------------- = 1.5ms               1024 + 1024        1024 + 1024
```

依据上面的2个计算公式，我们可以计算上面例子中每个进程分配的时间如下图所示。

[![time_slice.png](http://www.wowotech.net/content/uploadfile/201811/db481542200351.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201811/db481542200351.png) 

以上简单介绍了`task_group`嵌套一层的情况，如果`task_group`下面继续包含`task_group`，那么上面的计算公式就要再往上计算一层比例。实现该计算公式的函数是sched_slice()。

```c
 
static u64 sched_slice(struct cfs_rq *cfs_rq, struct sched_entity *se){	u64 slice = __sched_period(cfs_rq->nr_running + !se->on_rq);    /* 1 */ 	for_each_sched_entity(se) {                                     /* 2 */		struct load_weight *load;		struct load_weight lw; 		cfs_rq = cfs_rq_of(se);		load = &cfs_rq->load;                                       /* 3 */ 		if (unlikely(!se->on_rq)) {			lw = cfs_rq->load; 			update_load_add(&lw, se->load.weight);			load = &lw;		}		slice = __calc_delta(slice, se->load.weight, load);         /* 4 */	}	return slice;}
```

> 1. 根据当前就绪进程个数计算调度周期，默认情况下，进程不超过8个情况下，调度周期默认6ms。
> 2. for循环根据se->parent链表往上计算比例。
> 3. 获得se依附的cfs_rq的负载信息。
> 4. 计算slice = slice * se->load.weight / cfs_rq->load.weight的值。

## Group Se权重计算

上面举例说到group se的权重计算是根据权重比例计算。但是，实际的代码并不是。当我们dequeue task、enqueue task以及task tick的时候会通过update_cfs_group()函数更新group se的权重信息。

```c
 
static void update_cfs_group(struct sched_entity *se){	struct cfs_rq *gcfs_rq = group_cfs_rq(se);              /* 1 */	long shares, runnable; 	if (!gcfs_rq)		return; 	shares   = calc_group_shares(gcfs_rq);                 /* 2 */	runnable = calc_group_runnable(gcfs_rq, shares); 	reweight_entity(cfs_rq_of(se), se, shares, runnable);  /* 3 */}
```

> 1. 获得group se对应的group cfs_rq。
> 2. 计算新的权重值。
> 3. 更新group se的权重值为shares。

calc_group_shares()根据当前group cfs_rq负载情况计算新的权重。

```c
 
static long calc_group_shares(struct cfs_rq *cfs_rq){	long tg_weight, tg_shares, load, shares;	struct task_group *tg = cfs_rq->tg; 	tg_shares = READ_ONCE(tg->shares);	load = max(scale_load_down(cfs_rq->load.weight), cfs_rq->avg.load_avg);	tg_weight = atomic_long_read(&tg->load_avg);	/* Ensure tg_weight >= load */	tg_weight -= cfs_rq->tg_load_avg_contrib;	tg_weight += load;	shares = (tg_shares * load);	if (tg_weight)		shares /= tg_weight; 	return clamp_t(long, shares, MIN_SHARES, tg_shares);}
```

根据calc_group_shares()函数，我们可以得到权重计算公式如下（grq := group cfs_rq）：

```txt
 
                                tg->shares * loadge->load.weight = -------------------------------------------------                   tg->load_avg - grq->tg_load_avg_contrib + load load = max(grq->load.weight, grq->avg.load_avg)
```

`tg->load_avg`指所有的group cfs_rq负载贡献和。`grq->tg_load_avg_contrib`是指该group cfs_rq已经向`tg->load_avg`贡献的负载。因为tg是一个全局共享变量，多个CPU可能同时访问，为了避免严重的资源抢占。group cfs_rq负载贡献更新的值并不会立刻加到`tg->load_avg`上，而是等到负载贡献大于tg_load_avg_contrib一定差值后，再加到`tg->load_avg`上。例如，2个CPU的系统。CPU0上group cfs_rq初始值tg_load_avg_contrib为0，当group cfs_rq每次定时器更新负载的时候并不会访问tg变量，而是等到group cfs_rq的负载grp->avg.load_avg大于tg_load_avg_contrib很多的时候，这个差值达到一个数值（假设是2000），才会更新`tg->load_avg`为2000。然后，tg_load_avg_contrib的值赋值2000。又经过很多个周期后，grp->avg.load_avg和tg_load_avg_contrib的差值又等于2000，那么再一次更新`tg->load_avg`的值为4000。这样就避免了频繁访问tg变量。

但是上面的计算公式的依据是什么呢？如何得到的？首先我觉得我们能介绍的计算方法是上一节《用户组的权重》说的方法，计算group cfs_rq的权重占的比例。公式如下。

```c
                    tg->shares * grq->load.weightge->load.weight = -------------------------------               (1)		      \Sum grq->load.weight
```

由于计算**\Sum grq->load.weight**这个总和开销太大（原因可能是CPU数量比较大的系统，访问其他CPU group cfs_rq造成数据访问竞争激烈）。因此我们使用平均负载来近似处理，平均负载值变化缓慢，因此近似后的值更容易计算且更稳定。近似处理条件如下，将权重和平均负载近似处理。

```c
grq->load.weight -> grq->avg.load_avg                           (2)
```

经过近似处理后的公式(1)变换如下：

```c
                  tg->shares * grq->avg.load_avgge->load.weight = ------------------------------                (3)			tg->load_avg Where: tg->load_avg ~= \Sum grq->avg.load_avg
```

公式(3)问题在于，因为平均负载值变化很慢 (它的设计正是如此) ，这会导致在边界条件的时候的瞬变。 具体而言，当空闲group开始运行一个进程的时候。 我们的CPU的grq->avg.load_avg需要花费时间来慢慢变化，产生不良的延迟。在这种特殊情况下（单核CPU也是这种情况），公式(1)计算如下：

```c
                   tg->shares * grq->load.weightge->load.weight = ------------------------------- = tg->shares (4)			grq->load.weight
```

我们的目标就是将近似公式(3)在UP情景时修改成公式(4)的情况。

```c
ge->load.weight =              tg->shares * grq->load.weight   ---------------------------------------------------         (5)   tg->load_avg - grq->avg.load_avg + grq->load.weight
```

但是因为grq->load.weight可以降到0，导致除数是0。因此我们需要使用grq->avg.load_avg作为其下限，然后给出：

```c
                  tg->shares * grq->load.weightge->load.weight = -----------------------------		           (6)            		tg_load_avg'  Where: tg_load_avg' = tg->load_avg - grq->avg.load_avg +                       max(grq->load.weight, grq->avg.load_avg)
```

在UP系统上，公式(6)和公式(4)相似。在正常情况下，公式(6)和公式(3)相似。

说实话，真的是一大堆的公式，而且是各种近似处理和怼参数。一下看到公式的结果总是一头雾水，因为这可能涉及多次不同的优化修改，有些可能是经验总结，有些可能是实际环境测试。当你看不懂公式的时候，不妨会退到这个功能刚刚添加时候的样子，最初的版本总是让人容易接受。然后，顺着每一笔提交记录查看优化代码的原因，一步一个脚印，或许“面向大海春暖花开”。

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [CFS调度器（4）-PELT(per entity load tracking)](http://www.wowotech.net/process_management/450.html) | [CFS调度器（2）-源码解析](http://www.wowotech.net/process_management/448.html)»

**评论：**

**abcde**  
2024-05-03 03:24

图中上半部，nr_runing和h_nr_running的值分别等于10和19，多出的9是group cfs_rq的h_nr_running。group cfs_rq由于没有group se，因此nr_runing和h_nr_running的值都等于9。  
==========================================  
“nr_runing和h_nr_running的值分别等于10和19”错了，应该是“分别等于10和18”。一个关键的问题是h_nr_running只包含task se（不包含group se），因此，h_nr_running应该是18（而不是文中的19）。  
  
可以通过以下方式来证明：  
  
1. 通过代码证明，http://elixir.bootlin.com/linux/v5.19.17/source/kernel/sched/fair.c#L5659：  
代码每次加入的se 必定是task se（不会无缘无故的只加入group se的），group se是在for_each中“被动”加入的相应的cfs_rq的，即：将task se加入之后，for_each一路向上，在这个过程中，如果task se的某个祖先group se尚未加入相应的cfs_rq，则会在for_each中会将其加入相应的cfs_rq。for_each为每一个层级的cfs_rq->h_nr_running都加了1。所以，cfs_rq->h_nr_running永远只包含task se的数量。  
  
2. 通过下面代码做实验，可以证明:  
   a. cfs_rq->h_nr_running永远只包含task se的数量；cfs_rq->nr_running  
   b. group se也会被放入 cfs_rq，且，cfs_rq->nr_running 是包含 group se的数量的  
  
sudo cgcreate -g cpu:/tg1/tg2/tg3/tg4  
sudo cgexec -g cpu:/tg1/tg2  taskset 2  nice -n 0  /home/marvin/yourvm/capture/h2 2  1000  
sudo cgexec -g cpu:/tg1/tg2/tg3  taskset 2  nice -n 0  /home/marvin/yourvm/capture/h3 3  1000  
（其中，h2, h3 是  cpu-bound的代码，比如计算sha或无限for循环等）  
cat /sys/kernel/debug/sched/debug > /home/marvin/sched_debug.txt  
查看sched_debug.txt中的 cfs_rq[1]:/tg1...... 可以看到 其中的 nr_running和h_nr_running满足猜想。

[回复](http://www.wowotech.net/process_management/449.html#comment-8896)

**slava_chen**  
2024-01-11 15:47

我认为struct task_group里边的成员**se，他的长度和该主机中的CPU数量没有关系，成员**cfs_rq的长度和CPU数量数目相等。  
**se的长度只取决于当前task_group有多少个任务实体，每一个cfs_rq队列都同时可以接受多个任务实体se，但是一个cfs_rq队列只能对应于一个CPU上的rq

[回复](http://www.wowotech.net/process_management/449.html#comment-8859)

**slava_chen**  
2024-01-11 16:42

@slava_chen：抱歉，我又看了sched_create_group()->alloc_fair_sched_group()->init_tg_cfs_entry()的代码，发现task_group的cfs_rq和se长度的确是CPU数目

[回复](http://www.wowotech.net/process_management/449.html#comment-8860)

**550W**  
2023-02-24 20:05

好问题。  
这个是因为:  
grq->avg.load_avg =  
(cfs_rq->load.weight * delta_time) / LOAD_AVG_MAX  
当这个CPU一直运行,idle时间很少的时间，可以近似认为delta_time = LOAD_AVG_MAX，也就是可以用load_avg来模拟load.weight

[回复](http://www.wowotech.net/process_management/449.html#comment-8746)

**wangxingxing**  
2020-07-27 19:00

您好，有一个简单问题想请教一下：  
新创建的sched_entity group(tg->se[cpu] = se)是如何被插入其隶属于的rbtree的？

[回复](http://www.wowotech.net/process_management/449.html#comment-8068)

**wangxingxing**  
2020-07-27 19:18

@wangxingxing：请问是在enqueue_task_fair添加进程组进程的时候循环添加的对吧？

[回复](http://www.wowotech.net/process_management/449.html#comment-8069)

**lulu10922**  
2020-06-28 11:18

“但是因为grq->load.weight可以降到0，导致除数是0”  
这里应该是被除数为0哈，虽然除数也为0，但引起问题的显然是被除数为0，源码的注释里也是“a divide by zero”。因为UP scenario，tg->load_**g = grq->**g.load_**g，所以grq->load.weight如果也为0，被除数就为0了，所以grq->load.weight改为 max(...)，从这里改的是分母而不是分子也可以判断，引起问题的是被除数为0

[回复](http://www.wowotech.net/process_management/449.html#comment-8037)

**lulu10922**  
2020-06-23 09:58

大佬，能否告知下作图工具，谢谢

[回复](http://www.wowotech.net/process_management/449.html#comment-8033)

**linux123**  
2020-06-15 19:44

请教下大神，  
task_group中的struct sched_entity **se，在每个cpu上面的se list是一定不同的吗？  
还是可能不同，有没有什么标准，就是不考虑cpu affine的前提下，task group中的某个se在某个cpu上调度是随机的吗

[回复](http://www.wowotech.net/process_management/449.html#comment-8024)

**1+1=2**  
2020-04-15 18:07

请教一下，sched_slice有点奇怪，__sched_period的参数是cfs_rq的nr_running  
如果有两个位于同一层的cfs_rq A和B, A下辖10个se，B只有一个se，那么A __sched_period的时间就比B多，不太公平  
是不是应该用顶层的h_nr_running?

[回复](http://www.wowotech.net/process_management/449.html#comment-7951)

**[smcdef](http://www.wowotech.net/)**  
2020-04-16 18:24

@1+1=2：A会不会比B多，取决于A和B的权重，即share值。而不是取决于A和B下面有多少task。

[回复](http://www.wowotech.net/process_management/449.html#comment-7954)

**lollipop**  
2019-07-17 20:58

请教作者，实际场景下task_group会有哪些，可以给几个示例吗，谢谢

[回复](http://www.wowotech.net/process_management/449.html#comment-7536)

**[smcdef](http://www.wowotech.net/)**  
2020-04-16 18:25

@lollipop：docker就用的很多

[回复](http://www.wowotech.net/process_management/449.html#comment-7955)

**wenjianhn**  
2021-10-18 10:49

@lollipop：桌面发行版应该默认都会启用autogroup:  
  
$ grep termi /proc/sched_debug    
Sgnome-terminal-  3207       475.470142     17885   120         0.000000      5103.923704         0.000000 0 0 /autogroup-168  
$ grep -m1 chrome /proc/sched_debug  
S         chrome 24624     60331.977128       805   120         0.000000       380.673857         0.000000 0 0 /autogroup-129蓝  
  
这样配置,能降低在终端编译内核时,对浏览器影响.

[回复](http://www.wowotech.net/process_management/449.html#comment-8337)

**hzm**  
2019-04-23 10:58

有个问题想不明白，请教下楼主:  
假如A group有两个子se B和C， 如果C被移出rq，那么A也会被移出rq，这样B岂不是也被动的被移出去了？

[回复](http://www.wowotech.net/process_management/449.html#comment-7371)

**[smcdef](http://www.wowotech.net/)**  
2019-04-23 22:42

@hzm：你应该是QQ群提问的同学吧！问题一样的！

[回复](http://www.wowotech.net/process_management/449.html#comment-7372)

**hzm**  
2019-04-24 08:41

@smcdef：是的，多谢大神解答...

[回复](http://www.wowotech.net/process_management/449.html#comment-7373)

**miaolong**  
2019-04-25 10:57

@smcdef：QQ群能加不? 求群号

[回复](http://www.wowotech.net/process_management/449.html#comment-7375)

1 [2](http://www.wowotech.net/process_management/449.html/comment-page-2#comments)

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
    
    - [Linux时间子系统之（十二）：periodic tick](http://www.wowotech.net/timer_subsystem/periodic-tick.html)
    - [ARM64架构下地址翻译相关的宏定义](http://www.wowotech.net/memory_management/arm64-memory-addressing.html)
    - [eMMC 原理 3 ：分区管理](http://www.wowotech.net/basic_tech/emmc_partitions.html)
    - [Deadline调度器之（二）：细节和使用方法](http://www.wowotech.net/process_management/dl-scheduler-2.html)
    - [内存一致性模型](http://www.wowotech.net/memory_management/456.html)
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
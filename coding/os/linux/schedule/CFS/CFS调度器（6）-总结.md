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

作者：[smcdef](http://www.wowotech.net/author/531) 发布于：2019-1-1 12:37 分类：[进程管理](http://www.wowotech.net/sort/process_management)

经过前面一系列的文章描述，我们已经对CFS调度器有了一定的认识。那么本篇文章就作为一篇总结和思考。我们就回忆一下CFS调度器的那些事。我们就以问题的形式重新回顾一遍CFS调度器设计的原理。现在开始，我们问题来了。

#### CFS调度器为什么要引入虚拟时间片vruntime？

> 我觉得如果所有的进程不存在优先级区分的话，我们完全可以不引入vruntime的概念。所有的进程需要运行的实际时间都是一样的，大家都保持绝对的公平。如果由我们设计调度器，当然完全可以记录每个进程运行的实际时间。每次调度选择下一个进程的时候，我们完全可以挑选出已经运行时间最短的进程。当然，进程是存在轻重关系的。用户认为重要的进程，就是应该运行更长的时间。此时不同的进程由于优先级的原因导致运行时间不等。如果我们依然想采用像没有优先级的时候的方法去选择下一个运行的进程的话，自然有点困难。因为现在不同的进程运行的时间就是应该不一样，我们还怎么评判哪个进程运行时间最少呢。所以我们引入虚拟时间的概念。现在我们希望不同的进程根据优先级分配的物理时间通过一个公式计算得到一个相同的值，我们称这个值为虚拟时间。我们记录每个进程运行的虚拟时间，当需要选择下一个运行进程的时候，找出虚拟时间最小的进程即可。

#### 新建进程的vruntime的初值是不是0？

> 我们先考虑另一个问题，通过fork()创建的新进程的vruntime如果是0会怎么样？就绪队列上所有的进程的vruntime都已经是很大的一个值。如果新建进程的vruntime的值是0的话，根据CFS调度器pick_next_task_fair()逻辑，我们会倾向选择新建进程，一直让其更多的运行，追赶上就绪队列中其他进程的vruntime。既然不能是0，初值应该是什么比较合理呢？当然是和就绪队列上所有进程的vruntime的值差不多。具体怎么操作，下个问题揭晓。

#### 就绪队列记录的min_vruntime的作用是什么？

> 首先我们需要明白的是min_vruntime记录的究竟是什么。min_vruntime记录的是就绪队列管理的所有进程的最小虚拟时间。理论上来说，所有的进程的虚拟时间都大于min_vruntime。记录这个时间有什么用呢？我认为主要有3点作用。
> 
> - 当我们fork()一个新的进程的时候，虚拟时间肯定不能赋值0。否则的话，新建的进程会追赶其他进程，直到和就绪队列上其他进程的虚拟时间相当。这当然是不公平的，也是不可合理的设计。所以针对新fork()的进程，我们根据min_vruntime的值适当调整后赋值给新进程的虚拟时间。这样就不会出现新建的进程疯狂的运行。
> - 当进程sleep一段时间被wakeup的时候，此时也仅是物是人非。同样面临着类似新建进程的境况。同样我们需要根据min_vruntime的值适当调整后赋值给该进程。
> - 针对migration进程，我们面临一个新的问题。每个CPU上就绪队列的min_vruntime的值各不相同。有可能相差很多。如果进程从CPU0迁移到CPU1的话，进程的vruntime是否应该改变呢？当时是需要的，否则就会面临迁移后受到惩罚或者奖励。我们的方法是进程进程的vruntime减去CPU0的min_vruntime，然后加上CPU1的min_vruntime。

#### 唤醒的进程的vruntime该如何处理？

> 经过上一个问题，我们应该有点答案了。如果睡眠时间很长，自然是根据min_vruntime的值处理。问题是我们该如何处理？我们会根据min_vruntime的值减去一个数值作为唤醒进程的vruntime。为何减去一个值呢？我认为该进程已经sleep很长时间，本身就没有太占用CPU时间。给点补偿也是正常的。大多数的交互式应用，基本都是属于这种情况。这样处理，又提高了交互式应用的相应速度。如果sleep时间很短呢？当然是不需要干涉该进程的vruntime。

#### 就绪队列上所有的进程的vruntime都一定大于min_vruntime吗？

> 答案当然不是的。我们虽然引入min_vruntime的意义是最终就绪队列上所有进程的最小虚拟时间，但是并不能代表所有的进程vruntime都大于min_vruntime。这个问题在部分的情况下是成立的。例如，上面提到给唤醒进程vruntime一定的补偿，就会出现唤醒的进程的vruntime的值小于min_vruntime。

#### 唤醒的进程会抢占当前正在运行的进程吗？

> 分成两种情况，这个取决于唤醒抢占特性是否打开。即sched_feat的WAKEUP_PREEMPTION。如果没有打开唤醒抢占特性，那么就没有后话了。现在考虑该特性打开的情况。由于唤醒的进程会根据min_vruntime的值进行一定的奖励，因此存在很大的可能vruntime小于当前正在运行进程的vruntime。当时是否意味着只要唤醒进程的vruntime比当前运行进程的vruntime小就抢占呢？并不是。我们既要满足小的条件，又要在此基础上附加条件。两者差值必须大于唤醒粒度时间。该时间存在变量`sysctl_sched_wakeup_granularity`中，默认值1ms。

#### min_vruntime初始值为何如此奇怪？

> 就绪队列`struct cfs_rq`初始化是通过init_cfs_rq()函数进行。该函数如下：
> 
> ```c
>  
> void init_cfs_rq(struct cfs_rq *cfs_rq){	cfs_rq->min_vruntime = (u64)(-(1LL << 20));}
> ```
> 
> 初始值是U64_MAX - (1LL << 20)，U64_MAX代表64 bits无符号整型最大值。这里，我也有同样的疑问，min_vruntime为何初值不是0，搞个这么大的数是什么意思，和0相比有什么好处吗。当然，我也没有找到答案。下面都是我的猜测，和大家分享。min_vruntime单位是ns。也就是说系统运行大概(1<<20)ns，大约1ms的时间min_vruntime就会溢出。因此，原因可能就是为了更早的发现由于min_vruntime数值溢出导致的问题。如果初值是0的话，我们如果要提前发现min_vruntime溢出导致的问题大概需要545年时间（以NICE为0的进程计算，如果以NICE值为20计算的话，只需要8年左右时间）。

#### 最小粒度时间sysctl_sched_min_granularity是否一定会满足？

> CFS调度周期的时间设定取决于进程的数量，根据__sched_period()函数可知，当进程的数量大于`sched_nr_latency`时，调度周期的时间等于进程数量乘以`sysctl_sched_min_granularity`。
> 
> ```c
>  
> static u64 __sched_period(unsigned long nr_running){	if (unlikely(nr_running > sched_nr_latency))		return nr_running * sysctl_sched_min_granularity;	else		return sysctl_sched_latency;}
> ```
> 
> 但是这样做是否就意味着进程至少运行`sysctl_sched_min_granularity`时间才会被抢占呢？如果所有进程的优先级都一样的话，结果的确是这样的。但是当存在优先级不同的进程的时候，而且系统进程数量大于`sched_nr_latency`，那么NICE值高的进程并不能保证最少运行`sysctl_sched_min_granularity`时间被强占。这是一种特殊的情况，这种进程在一个调度周期内分配的总时间都不足`sysctl_sched_min_granularity`。
> 
> 如果有进程在一个周期内分配的份额大于`sysctl_sched_min_granularity`会是什么情况呢？在这种情况下，CFS调度器倒是可能可以保证最小粒度时间。我们看下check_preempt_tick()。
> 
> ```c
> 
> static void check_preempt_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr){	unsigned long ideal_runtime, delta_exec;	struct sched_entity *se;	s64 delta; 	ideal_runtime = sched_slice(cfs_rq, curr);	delta_exec = curr->sum_exec_runtime - curr->prev_sum_exec_runtime;	if (delta_exec > ideal_runtime) {				/* 1 */		resched_curr(rq_of(cfs_rq));		clear_buddies(cfs_rq, curr);		return;	} 	if (delta_exec < sysctl_sched_min_granularity)	/* 2 */		return; 	se = __pick_first_entity(cfs_rq);	delta = curr->vruntime - se->vruntime; 	if (delta < 0)		return; 	if (delta > ideal_runtime)		resched_curr(rq_of(cfs_rq));}
> ```
> 
> 1. 如果进程运行时间超过本周期分配时间，just reschedule it。
> 2. 第1处不满足，如果这里运行时间不足最小粒度时间，直接退出。

标签: [CFS](http://www.wowotech.net/tag/CFS)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [编译乱序(Compiler Reordering)](http://www.wowotech.net/kernel_synchronization/453.html) | [CFS调度器（5）-带宽控制](http://www.wowotech.net/process_management/451.html)»

**评论：**

**marvin263**  
2022-12-10 11:08

时间片太小，调度就会更为频繁。从v5.13起，加了个开关，确保最小执行时间为sysctl_sched_min_granularity的。  
  
http(s)://github.com/torvalds/linux/commit/0c2de3f054a59f15e01804b75a04355c48de628c  
  
The current sched_slice() seems to have issues; there's two possible  
things that could be improved:  
  
- the 'nr_running' used for __sched_period() is daft when cgroups are  
   considered. Using the RQ wide h_nr_running seems like a much more  
   consistent number.  
  
- (esp) cgroups can slice it real fine, which makes for easy  
   over-scheduling, ensure min_gran is what the name says.  
  
  
if (sched_feat(BASE_SLICE))  
        slice = max(slice, (u64)sysctl_sched_min_granularity);

[回复](http://www.wowotech.net/process_management/452.html#comment-8717)

**wasd**  
2023-11-01 10:53

@marvin263：那请问，如果是这样的话，那按照nr_running * sysctl_sched_min_granularity算出来的调度周期不是不准确了吗？因为nice值小的进程会运行超过sysctl_sched_min_granularity

[回复](http://www.wowotech.net/process_management/452.html#comment-8841)

**orangeboyye**  
2022-08-08 02:38

原文：“如果以NICE值为20计算的话，只需要8年左右时间”，应该是-20，少写个负号

[回复](http://www.wowotech.net/process_management/452.html#comment-8664)

**ryancat**  
2019-10-18 09:25

请问，每个tick到来时，当前进程的vruntime都会增加，那么在tick中断里面是何时触发调度的呢？  
1、每个tick都会检查当前进程在本调度周期内已经执行的时间是否达到了预期分配的总时间，达到就开始调度？  
2、每个tick都检查当前进程的vruntime是否是运行队列里面最小的，不是最小且已执行时间超过了sysctl_sched_min_granularity就开始调度？  
  
谢谢！

[回复](http://www.wowotech.net/process_management/452.html#comment-7703)

**收信的加西亚**  
2020-06-23 17:39

@ryancat：1.首先tick中断是per_cpu的时钟中断，在tick中只是检查当前进程是否需要重新调度，如果满足重新调度条件设置TIF_NEED_RECHED标志，如运行的时间超过了理想的时间，或优先级更高的进程，等待最近的一个调度点到达才会触发调度（如：系统调用或者中断返回用户空间前夕，或者是重新开启内核抢占的时候（支持内核抢占）等）  
2.达到预期分配的总时间是设置TIF_NEED_RECHED标志，不会立马调度，会等待最近的调度点到来  
3.还需要比较当前调度实体的虚拟运行时间和cfs运行队列最小的调度实体的虚拟运行时间差值，只有差值大于当前调度实体的理想运行时间才设置重新调度标志。

[回复](http://www.wowotech.net/process_management/452.html#comment-8034)

**ccc**  
2019-06-11 22:34

想到一个例子 那分配工作当一个例子  把一项任务按照全组平均水平估算时间 例如6小时  全组有六个人 每个人按照这个换算关系领到1个小时任务 但是由于个人能力不同  实际工作时间也不同 和这个算法思想感觉很相似 保证公平性

[回复](http://www.wowotech.net/process_management/452.html#comment-7467)

**ccc**  
2019-06-11 22:30

联想到一个例子 那分配工作当一个例子  把一项任务按照全组平均水平估算时间 例如6小时  全组有六个人 每个人按照这个换算关系领到1个小时任务 但是由于个人能力不同  实际工作时间也不同 和这个算法思想感觉很相似 保证公平性

[回复](http://www.wowotech.net/process_management/452.html#comment-7466)

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
    
    - [Linux内核同步机制之（八）：mutex](http://www.wowotech.net/kernel_synchronization/504.html)
    - [Linux I2C framework(3)_I2C consumer](http://www.wowotech.net/comm/i2c_consumer.html)
    - [中断上下文中调度会怎样？](http://www.wowotech.net/process_management/schedule-in-interrupt.html)
    - [Linux设备模型(1)_基本概念](http://www.wowotech.net/device_model/13.html)
    - [Linux时间子系统之（十二）：periodic tick](http://www.wowotech.net/timer_subsystem/periodic-tick.html)
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
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

作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2017-3-14 18:46 分类：[进程管理](http://www.wowotech.net/sort/process_management)

一、前言

本文主要描述的是进程优先级这个概念。从用户空间来看，进程优先级就是nice value和scheduling priority，对应到内核，有静态优先级、realtime优先级、归一化优先级和动态优先级等概念，我们希望能在第二章将这些相关的概念描述清楚。为了加深理解，在第三章我们给出了几个典型数据流过程的分析。

二、overview

1、蓝图

[![priority](http://www.wowotech.net/content/uploadfile/201703/771adb345f01f5f2f7f816f57fae9e1e20170314104605.gif "priority")](http://www.wowotech.net/content/uploadfile/201703/578560e4870564b7532a4dc1f087150520170314104603.gif)

2、用户空间的视角

在用户空间，进程优先级有两种含义：nice value和scheduling priority。对于普通进程而言，进程优先级就是nice value，从-20（优先级最高）～19（优先级最低），通过修改nice value可以改变普通进程获取cpu资源的比例。随着实时需求的提出，进程又被赋予了另外一种属性scheduling priority，而这些进程被称为实时进程。实时进程的优先级的范围可以通过sched_get_priority_min和sched_get_priority_max，对于linux而言，实时进程的scheduling priority的范围是1（优先级最低）～99（优先级最高）。当然，普通进程也有scheduling priority，被设定为0。

3、内核中的实现

内核中，task struct中有若干和进程优先级有个的成员，如下：

> struct task_struct {  
> ......  
>     int prio, static_prio, normal_prio;  
>     unsigned int rt_priority;  
> ......  
>     unsigned int policy;  
> ......  
> }

policy成员记录了该线程的调度策略，而其他的成员表示了各种类型的优先级，下面的小节我们会一一描述。

4、静态优先级

task struct中的static_prio成员。我们称之静态优先级，其特点如下：

（1）值越小，进程优先级越高

（2）0 – 99用于real-time processes（没有实际的意义），100 – 139用于普通进程

（3）缺省值是 120

（4）用户空间可以通过nice()或者setpriority对该值进行修改。通过getpriority可以获取该值。

（5）新创建的进程会继承父进程的static priority。

静态优先级是所有相关优先级的计算的起点，要么继承自父进程，要么用户空间自行设定。一旦修改了静态优先级，那么normal priority和动态优先级都需要重新计算。

5、实时优先级

task struct中的rt_priority成员表示该线程的实时优先级，也就是从用户空间的视角来看的scheduling priority。0是普通进程，1～99是实时进程，99的优先级最高。

6、归一化优先级

task struct中的normal_prio成员。我们称之归一化优先级（normalized priority），它是根据静态优先级、scheduling priority和调度策略来计算得到，代码如下：

> static inline int normal_prio(struct task_struct *p)  
> {  
>     int prio;
> 
>     if (task_has_dl_policy(p))  
>         prio = MAX_DL_PRIO-1;  
>     else if (task_has_rt_policy(p))  
>         prio = MAX_RT_PRIO-1 - p->rt_priority;  
>     else  
>         prio = __normal_prio(p);  
>     return prio;  
> }

这里我们先聊聊归一化（Normalization）这个看起来稍微有点晦涩的术语。如果你做过音视频定点算法的优化，应该对这个词不陌生。不同的定点数据有不同的表示，有Q31的，有Q15，这些数据的小数点的位置不同，无法进行比较、加减等操作，因此需要归一化，全部转换成某个特定的数据格式（其实就是确定小数点的位置）。在数学上，1米和1mm在进行操作的时候也需要归一化，全部转换成同一个量纲就OK了。对于这里的优先级，调度器需要综合考虑各种因素，例如调度策略，nice value、scheduling priority等，把这些factor全部考虑进来，归一化成一个数轴上的number，以此来表示其优先级，这就是normalized priority。对于一个线程，其normalized priority的number越小，其优先级越大。

调度策略是deadline的进程比RT进程和normal进程的优先级还要高，因此它的归一化优先级是负数：-1。如果采用实时调度策略，那么该线程的normalized priority和rt_priority相关。task struct中的rt_priority成员是用户空间视角的实时优先级（scheduling priority），MAX_RT_PRIO-1是99，MAX_RT_PRIO-1 - p->rt_priority则翻转了实时进程的scheduling priority，最高优先级是0，最低是98。顺便说一句，normalized priority是99的情况是没有意义的。对于普通进程，normalized priority就是其静态优先级。

7、动态优先级

task struct中的prio成员表示了该线程的动态优先级，也就是调度器在进行调度时候使用的那个优先级。动态优先级在运行时可以被修改，例如在处理优先级翻转问题的时候，系统可能会临时调升一个普通进程的优先级。一般设定动态优先级的代码是这样的：p->prio = effective_prio(p)，具体计算动态优先级的代码如下：

> static int effective_prio(struct task_struct *p)  
> {  
>     p->normal_prio = normal_prio(p);  
>     if (!rt_prio(p->prio))  
>         return p->normal_prio;  
>     return p->prio;  
> }

rt_prio是一个根据当前优先级来确定是否是实时进程的函数，包括两种情况，一种情况是该进程是实时进程，调度策略是SCHED_FIFO或者SCHED_RR。另外一种情况是人为的将该进程提升到RT priority的区域（例如在使用优先级继承的方法解决系统中优先级翻转问题的时候）。在这两种情况下，我们都不改变其动态优先级，即effective_prio返回当前动态优先级p->prio。其他情况，进程的动态优先级跟随归一化的优先级。

三、典型数据流程分析

1、用户空间设定nice value

用户空间设定nice value的操作，在内核中主要是set_user_nice函数实现的，无论是sys_nice或者sys_setpriority，在参数检查和权限检查之后都会调用set_user_nice函数，完成具体的设定。代码如下：

> void set_user_nice(struct task_struct *p, long nice)  
> {  
>     int old_prio, delta, queued;  
>     unsigned long flags;  
>     struct rq *rq;   
>     rq = task_rq_lock(p, &flags);  
>     if (task_has_dl_policy(p) || task_has_rt_policy(p)) {－－－－－－－－－－－（1）  
>         p->static_prio = NICE_TO_PRIO(nice);  
>         goto out_unlock;  
>     }  
>     queued = task_on_rq_queued(p);－－－－－－－－－－－－－－－－－－－（2）  
>     if (queued)  
>         dequeue_task(rq, p, DEQUEUE_SAVE);
> 
>     p->static_prio = NICE_TO_PRIO(nice);－－－－－－－－－－－－－－－－（3）  
>     set_load_weight(p);  
>     old_prio = p->prio;  
>     p->prio = effective_prio(p);  
>     delta = p->prio - old_prio;
> 
>     if (queued) {  
>         enqueue_task(rq, p, ENQUEUE_RESTORE);－－－－－－－－－－－－（2）  
>         if (delta < 0 || (delta > 0 && task_running(rq, p)))－－－－－－－－－－－－（4）  
>             resched_curr(rq);  
>     }  
> out_unlock:  
>     task_rq_unlock(rq, p, &flags);  
> }

（1）如果是实时进程或者deadline类型的进程，那么nice value其实是没有什么实际意义的，不过我们还是设定其静态优先级，当然，这样的设定其实不会起到什么作用的，也不会实际改变调度器行为，因此直接返回，没有dequeue和enqueue的动作。

（2）在step中已经处理了调度策略是RT类和DEADLINE类的进程，因此，执行到这里，只可能是普通进程了，使用CFS算法。如果该task在run queue上（queued 等于true），那么由于我们修改了nice value，调度器需要重新审视当前runqueue中的task。因此，我们需要将该task从rq中摘下，在重新计算优先级之后，再次插入该runqueue对应的runable task的红黑树中。

（3）最核心的代码就是p->static_prio = NICE_TO_PRIO(nice);这一句了，其他的都是side effect。比如说load weight。当cpu一刻不停的运算的时候，其load是100％，没有机会调度到idle进程休息一下。当系统中没有实时进程或者deadline进程的时候，所有的runnable的进程一起来瓜分cpu资源，以此不同的进程分享一个特定比例的cpu资源，我们称之load weight。不同的nice value对应不同的cpu load weight，因此，当更改nice value的时候，也必须通过set_load_weight来更新该进程的cpu load weight。除了load weight，该线程的动态优先级也需要更新，这是通过p->prio = effective_prio(p);来完成的。

（4）delta 记录了新旧线程的动态优先级的差值，当调试了该线程的优先级（delta < 0），那么有可能产生一个调度点，因此，调用resched_curr，给当前正在运行的task做一个标记，以便在返回用户空间的时候进行调度。此外，如果修改当前running状态的task的动态优先级，那么调降（delta > 0）意味着该进程有可能需要让出cpu，因此也需要resched_curr标记当前running状态的task需要reschedule。

2、进程缺省的调度策略和调度参数

我们先思考这样的一个问题：在用户空间设定调度策略和调度参数之前，一个线程的default scheduling policy是什么呢？这需要追溯到fork的时候（具体代码在sched_fork函数中），这个和task struct中sched_reset_on_fork设定相关。如果没有设定这个flag，那么说明在fork的时候，子进程跟随父进程的调度策略，如果设定了这个flag，则说明子进程的调度策略和调度参数不能继承自父进程，而是需要设定为default。代码片段如下：

> int sched_fork(unsigned long clone_flags, struct task_struct *p)  
> {
> 
> ……  
>     p->prio = current->normal_prio; －－－－－－－－－－－－－－－－－－－（1）  
>     if (unlikely(p->sched_reset_on_fork)) {  
>         if (task_has_dl_policy(p) || task_has_rt_policy(p)) {－－－－－－－－－－（2）  
>             p->policy = SCHED_NORMAL;  
>             p->static_prio = NICE_TO_PRIO(0);  
>             p->rt_priority = 0;  
>         } else if (PRIO_TO_NICE(p->static_prio) < 0)  
>             p->static_prio = NICE_TO_PRIO(0);
> 
>         p->prio = p->normal_prio = __normal_prio(p); －－－－－－－－－－－－（3）  
>         set_load_weight(p);   
>         p->sched_reset_on_fork = 0;  
>     }
> 
> ……
> 
> }

（1）sched_fork只是fork过程中的一个片段，在fork一开始，dup_task_struct已经复制了一个和父进程完全一个的进程描述符（task struct），因此，如果没有步骤2中的重置，那么子进程是跟随父进程的调度策略和调度参数（各种优先级），当然，有时候为了解决PI问题而临时调升父进程的动态优先级，在fork的时候不宜传递到子进程中，因此这里重置了动态优先级。

（2）缺省的调度策略是SCHED_NORMAL，静态优先级等于120（也就是说nice value等于0），rt priority等于0（普通进程）。不管父进程如何，即便是deadline的进程，其fork的子进程也需要恢复到缺省参数。

（3）既然调度策略和静态优先级已经修改了，那么也需要更新动态优先级和归一化优先级。此外，load weight也需要更新。一旦子进程中恢复到了缺省的调度策略和优先级，那么sched_reset_on_fork这个flag已经完成了历史使命，可以clear掉了。

OK，至此，我们了解了在fork过程中对调度策略和调度参数的处理，这里还是要追加一个问题：为何不一切继承父进程的调度策略和参数呢？为何要在fork的时候reset to default呢？在linux中，对于每一个进程，我们都会进行资源限制。例如对于那些实时进程，如果它持续消耗cpu资源而没有发起一次可以引起阻塞的系统调用，那么我们猜测这个realtime进程跑飞了，从而锁住了系统。对于这种情况，我们要进行干预，因此引入了RLIMIT_RTTIME这个per-process的资源限制项。但是，如果用户空间的realtime进程通过fork其实也可以绕开RLIMIT_RTTIME这个限制，从而肆意的攫取cpu资源。然而，机智的内核开发人员早已经看穿了这一切，为了防止实时进程“泄露”到其子进程中，sched_reset_on_fork这个flag被提出来。

3、用户空间设定调度策略和调度参数

通过sched_setparam接口函数可以修改rt priority的调度参数，而通过sched_setscheduler功能会更强一些，不但可以设定rt priority，还可以设定调度策略。而sched_setattr是一个集大成之接口，可以设定一个线程的调度策略以及该调度策略下的调度参数。当然，对于内核，这些接口都通过__sched_setscheduler这个内核函数来完成对指定线程调度策略和调度参数的修改。

__sched_setscheduler分成两个部分，首先进行安全性检查和参数检查，其次进行具体的设定。

我们先看看安全性检查。如果用户空间可以自由的修改调度策略和调度优先级，那么世界就乱套了，每个进程可能都想把自己的调度策略和优先级提升上去，从而获取足够的CPU 资源。因此用户空间设定调度策略和调度参数要遵守一定的规则：如果没有CAP_SYS_NICE的能力，那么基本上该线程能被允许的操作只是降级而已。例如从SCHED_FIFO修改成SCHED_NORMAL，异或不修改scheduling policy，而是降低静态优先级（nice value）或者实时优先级（scheduling priority）。这里例外的是SCHED_DEADLINE的设定，按理说如果进程本身的调度策略就是SCHED_DEADLINE，那么应该允许“优先级”降低的操作（这里用优先级不是那么合适，其实就是减小run time，或者加大period，这样可以放松对cpu资源的获取），但是目前的4.4.6内核不允许（也许以后版本的内核会允许）。此外，如果没有CAP_SYS_NICE的能力，那么设定调度策略和调度参数的操作只能是限于属于同一个登录用户的线程。如果拥有CAP_SYS_NICE的能力，那么就没有那么多限制了，可以从普通进程提升成实时进程（修改policy），也可以提升静态优先级或者实时优先级。

具体的修改比较简单，是通过__setscheduler_params函数完成，其实也就是是根据sched_attr中的参数设定到task struct相关成员中，大家可以自行阅读代码进行理解。

参考文档：

1、linux下的各种man page

2、linux 4.4.6内核源代码

_原创文章，转发请注明出处。蜗窝科技_

标签: [进程优先级](http://www.wowotech.net/tag/%E8%BF%9B%E7%A8%8B%E4%BC%98%E5%85%88%E7%BA%A7)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [中断上下文中调度会怎样？](http://www.wowotech.net/process_management/schedule-in-interrupt.html) | [玩转BLE(3)_使用微信蓝牙精简协议伪造记步数据](http://www.wowotech.net/bluetooth/weixin_ble_1.html)»

**评论：**

**ccilery**  
2021-09-18 20:06

请问如何在用户空间观察到 scheduling priority？在top中似乎没有体现到。

[回复](http://www.wowotech.net/process_management/process-priority.html#comment-8318)

**ldzq**  
2020-01-19 17:19

hi,linuxer:  
static_prio,新创建的进程会继承父进程的static priority。  
这个观点，我在代码里没有得到证实，在进程fork过程中没有看到对这个值的设置，而从网上看到的结果是在启动时会设置，然后可以通过nice和sched_setscheduler，所以不是继承父进程，请 check一下，感谢 。

[回复](http://www.wowotech.net/process_management/process-priority.html#comment-7851)

**ccilery**  
2021-09-18 20:50

@ldzq：dup_task_struct已经复制了一个和父进程完全一个的进程描述符（task struct），没有对static_prio进行设置，那不就和父进程的一致吗

[回复](http://www.wowotech.net/process_management/process-priority.html#comment-8319)

**MarcusKin**  
2019-09-22 15:27

感谢作者，图画的非常棒，将不同的优先级概念层次化地清晰呈现出来，解决了之前看代码过程中云里雾里的感觉，后续只需要看这张图就行啦。

[回复](http://www.wowotech.net/process_management/process-priority.html#comment-7662)

**tanker**  
2018-09-28 11:29

蓝图上提到“RT类型也有nice value，但没有实际意义”。RR类型的，运行时间片是根据nice值计算的吧。  
“用户空间接口”这篇文章提到过“实际上调整nice value就是调整调度器分配给该进程的CPU时间”。

[回复](http://www.wowotech.net/process_management/process-priority.html#comment-6964)

**[heikang](http://www.wowotech.net/)**  
2018-12-20 20:35

@tanker：可能是作者觉得实时进程优先级不同会有直接的压制效果，比如70和88优先级的RR进程相比较，RR属性完全没用

[回复](http://www.wowotech.net/process_management/process-priority.html#comment-7094)

**liang_1911**  
2017-11-22 16:54

请教下, 之前只了解到进程调度有 SCHED_RR 、SCHED_FIFO、SCHED_OTHER 这三种调度策略，怎么还有 deadline 的调度策略？ 我百度了一下，linux 的 I/O 调度方法里倒有一个 deadline 调度策略。

[回复](http://www.wowotech.net/process_management/process-priority.html#comment-6231)

**[heikang](http://www.wowotech.net/)**  
2018-12-20 20:33

@liang_1911：核间通信会用到

[回复](http://www.wowotech.net/process_management/process-priority.html#comment-7093)

**caid**  
2017-10-02 10:38

对于动态优先级与静态优先级有一点疑惑，仔细阅读代码后(linux-2.6.10)，有一些浅见，  
  
1.    static_prio这里的nice值的说明，是否应该去掉或者放到第一行nice值部分说明，虽然普通进程的static_prio是由nice值及bonus计算而得，但是放在这里始终感觉会导致理解错误，或者将注意力从优先级上分散  
2.    LKD和ULK都强调实时进程是基于静态优先级的，内核不为实时进程计算动态优先级。因此我以为rt_priority就是实时进程的静态优先级(需反转)，包括普通进程，拥有rt_priority=0的rt_priority；但是task_struct结构中又有static_prio的字段，这个是专为普通进程准备的，其取值范围是[100,139]。与实时进程的静态优先级取值范围[1, 99]正好构成一个完整的序列[1, 139]。所以更加辅证了rt_priority就是实时进程的静态优先级的判断。  
3.    对于prio，被认为是普通进程的动态优先级。事实上，这个也是实时进程的动态优先级，只不过这个动态优先级并没有去计算，直接由rt_priority反转，即实时进程的静态优先级就是它的动态优先级。对于普通进程，prio直接就等于static_prio； 所以最后的结论是，实时进程与普通进程都有静态优先级[1~139]，通过转换变为动态优先级prio。动态优先级是内核执行调度的依据。  
4.    rt_priority=0时，反转得到的是99这个优先级，这个是无意义的，并且也不会得到99这个值，因为在归一化之前，就已经过滤了:策略是SCHED_NORMAL且rt_priority非0，则直接跳出。故而不会将0值去反转。

[回复](http://www.wowotech.net/process_management/process-priority.html#comment-6088)

**[linuxer](http://www.wowotech.net/)**  
2017-10-09 11:34

@caid：多谢你的补充。  
  
所谓bonus也就是对交互式进程和cpu密集型进程进行奖惩，但是引入CFS之后，这些都不需要了。static_prio和nice值是等同的，只不过一个是内核空间视角，一个是用户空间视角。

[回复](http://www.wowotech.net/process_management/process-priority.html#comment-6092)

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
    
    - [文件缓存回写简述](http://www.wowotech.net/memory_management/327.html)
    - [为什么会有文件系统(二)](http://www.wowotech.net/filesystem/396.html)
    - [eMMC 原理 1 ：Flash Memory 简介](http://www.wowotech.net/basic_tech/flash_memory_intro.html)
    - [X-015-KERNEL-ARM generic timer driver的移植](http://www.wowotech.net/x_project/generic_timer_porting.html)
    - [《奔跑吧，Linux内核》已经上架预售了](http://www.wowotech.net/tech_discuss/running_kernel.html)
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
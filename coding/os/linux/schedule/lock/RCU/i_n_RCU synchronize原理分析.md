
作者：[itrocker](http://www.wowotech.net/author/295) 发布于：2015-10-27 19:10 分类：[内核同步机制](http://www.wowotech.net/sort/kernel_synchronization)

RCU（Read-Copy Update）是Linux内核比较成熟的新型读写锁，具有较高的读写并发性能，常常用在需要互斥的性能关键路径。在kernel中，rcu有tiny rcu和tree rcu两种实现，tiny rcu更加简洁，通常用在小型嵌入式系统中，tree rcu则被广泛使用在了server, desktop以及android系统中。本文将以tree rcu为分析对象。

# **1** **如何度过宽限期**

RCU的核心理念是读者访问的同时，写者可以更新访问对象的副本，但写者需要等待所有读者完成访问之后，才能删除老对象。这个过程实现的关键和难点就在于如何判断所有的读者已经完成访问。通常把写者开始更新，到所有读者完成访问这段时间叫做宽限期（Grace Period）。内核中实现宽限期等待的函数是synchronize_rcu。

## **1.1** **读者锁的标记**

在普通的TREE RCU实现中，rcu_read_lock和rcu_read_unlock的实现非常简单，分别是关闭抢占和打开抢占：

```cpp
1. static inline void __rcu_read_lock(void)
2. {
3. 	preempt_disable();
4. }

6. static inline void __rcu_read_unlock(void)
7. {
8. 	preempt_enable();
9. }
```

这时是否度过宽限期的判断就比较简单：每个CPU都经过一次抢占。因为发生抢占，就说明不在rcu_read_lock和rcu_read_unlock之间，必然已经完成访问或者还未开始访问。

## **1.2** **每个CPU度过quiescnet state**

接下来我们看每个CPU上报完成抢占的过程。kernel把这个完成抢占的状态称为quiescent state。每个CPU在时钟中断的处理函数中，都会判断当前CPU是否度过quiescent state。

```cpp
1. void update_process_times(int user_tick)
2. {
3. ......
4. 	rcu_check_callbacks(cpu, user_tick);
5. ......
6. }

8. void rcu_check_callbacks(int cpu, int user)
9. {
10. ......
11. 	if (user || rcu_is_cpu_rrupt_from_idle()) {
12. 		/*在用户态上下文，或者idle上下文，说明已经发生过抢占*/
13. 		rcu_sched_qs(cpu);
14. 		rcu_bh_qs(cpu);
15. 	} else if (!in_softirq()) {
16. 		/*仅仅针对使用rcu_read_lock_bh类型的rcu，不在softirq，
17. 		 *说明已经不在read_lock关键区域*/
18. 		rcu_bh_qs(cpu);
19. 	}
20. 	rcu_preempt_check_callbacks(cpu);
21. 	if (rcu_pending(cpu))
22. 		invoke_rcu_core();
23. ......
24. }
```

这里补充一个细节说明，Tree RCU有多个类型的RCU State，用于不同的RCU场景，包括rcu_sched_state、rcu_bh_state和rcu_preempt_state。不同的场景使用不同的RCU API，度过宽限期的方式就有所区别。例如上面代码中的rcu_sched_qs和rcu_bh_qs，就是为了标记不同的state度过quiescent state。普通的RCU例如内核线程、系统调用等场景，使用rcu_read_lock或者rcu_read_lock_sched，他们的实现是一样的；软中断上下文则可以使用rcu_read_lock_bh，使得宽限期更快度过。

细分这些场景是为了提高RCU的效率。rcu_preempt_state将在下文进行说明。

## **1.3** **汇报宽限期度过**

每个CPU度过quiescent state之后，需要向上汇报直至所有CPU完成quiescent state，从而标识宽限期的完成，这个汇报过程在软中断RCU_SOFTIRQ中完成。软中断的唤醒则是在上述的时钟中断中进行。

```cpp
update_process_times

    -> rcu_check_callbacks

        -> invoke_rcu_core

RCU_SOFTIRQ软中断处理的汇报流程如下：

rcu_process_callbacks

    -> __rcu_process_callbacks

        -> rcu_check_quiescent_state

            -> rcu_report_qs_rdp

                -> rcu_report_qs_rnp
```

其中rcu_report_qs_rnp是从叶子节点向根节点的遍历过程，同一个节点的子节点都通过quiescent state后，该节点也设置为通过。

![[Pasted image 20241024184506.png]]

这个树状的汇报过程，也就是“Tree RCU”这个名字得来的缘由。

树结构每层的节点数量和叶子节点数量由一系列的宏定义来决定：

1. #define MAX_RCU_LVLS 4

1. #define RCU_FANOUT_1	      (CONFIG_RCU_FANOUT_LEAF)

1. #define RCU_FANOUT_2	      (RCU_FANOUT_1 * CONFIG_RCU_FANOUT)

1. #define RCU_FANOUT_3	      (RCU_FANOUT_2 * CONFIG_RCU_FANOUT)

1. #define RCU_FANOUT_4	      (RCU_FANOUT_3 * CONFIG_RCU_FANOUT)

1. #if NR_CPUS \<= RCU_FANOUT_1

1. # define RCU_NUM_LVLS	      1

1. # define NUM_RCU_LVL_0	      1

1. # define NUM_RCU_LVL_1	      (NR_CPUS)

1. # define NUM_RCU_LVL_2	      0

1. # define NUM_RCU_LVL_3	      0

1. # define NUM_RCU_LVL_4	      0

1. #elif NR_CPUS \<= RCU_FANOUT_2

1. # define RCU_NUM_LVLS	      2

1. # define NUM_RCU_LVL_0	      1

1. # define NUM_RCU_LVL_1	      DIV_ROUND_UP(NR_CPUS, RCU_FANOUT_1)

1. # define NUM_RCU_LVL_2	      (NR_CPUS)

1. # define NUM_RCU_LVL_3	      0

1. # define NUM_RCU_LVL_4	      0

1. #elif NR_CPUS \<= RCU_FANOUT_3

1. # define RCU_NUM_LVLS	      3

1. # define NUM_RCU_LVL_0	      1

1. # define NUM_RCU_LVL_1	      DIV_ROUND_UP(NR_CPUS, RCU_FANOUT_2)

1. # define NUM_RCU_LVL_2	      DIV_ROUND_UP(NR_CPUS, RCU_FANOUT_1)

1. # define NUM_RCU_LVL_3	      (NR_CPUS)

1. # define NUM_RCU_LVL_4	      0

1. #elif NR_CPUS \<= RCU_FANOUT_4

1. # define RCU_NUM_LVLS	      4

1. # define NUM_RCU_LVL_0	      1

1. # define NUM_RCU_LVL_1	      DIV_ROUND_UP(NR_CPUS, RCU_FANOUT_3)

1. # define NUM_RCU_LVL_2	      DIV_ROUND_UP(NR_CPUS, RCU_FANOUT_2)

1. # define NUM_RCU_LVL_3	      DIV_ROUND_UP(NR_CPUS, RCU_FANOUT_1)

1. # define NUM_RCU_LVL_4	      (NR_CPUS)

## **1.3** **宽限期的发起与完成**

所有宽限期的发起和完成都是由同一个内核线程rcu_gp_kthread来完成。通过判断rsp->gp_flags & RCU_GP_FLAG_INIT来决定是否发起一个gp；通过判断! (rnp->qsmask) && !rcu_preempt_blocked_readers_cgp(rnp))来决定是否结束一个gp。

发起一个GP时，rsp->gpnum++；结束一个GP时，rsp->completed = rsp->gpnum。

## **1.4 rcu callbacks处理**

rcu的callback通常是在sychronize_rcu中添加的wakeme_after_rcu，也就是唤醒synchronize_rcu的进程，它正在等待GP的结束。

callbacks的处理同样在软中断RCU_SOFTIRQ中完成

rcu_process_callbacks

-> \_\_rcu_process_callbacks

-> invoke_rcu_callbacks

-> rcu_do_batch

-> \_\_rcu_reclaim

这里RCU的callbacks链表采用了一种分段链表的方式，整个callback链表，根据具体GP结束的时间，分成若干段：nxtlist -- \*nxttail\[RCU_DONE_TAIL\] -- \*nxttail\[RCU_WAIT_TAIL\] -- \*nxttail\[RCU_NEXT_READY_TAIL\] -- \*nxttail\[RCU_NEXT_TAIL\]。

rcu_do_batch只处理nxtlist -- *nxttail\[RCU_DONE_TAIL\]之间的callbacks。每个GP结束都会重新调整callback所处的段位，每个新的callback将会添加在末尾，也就是*nxttail\[RCU_NEXT_TAIL\]。

# **2** **可抢占的RCU**

如果config文件定义了CONFIG_TREE_PREEMPT_RCU=y，那么sychronize_rcu将默认使用rcu_preempt_state。这类rcu的特点就在于read_lock期间是允许其它进程抢占的，因此它判断宽限期度过的方法就不太一样。

从rcu_read_lock和rcu_read_unlock的定义就可以知道，TREE_PREEMPT_RCU并不是以简单的经过抢占为CPU渡过GP的标准，而是有个rcu_read_lock_nesting计数

1. void \_\_rcu_read_lock(void)

1. {

1. current->rcu_read_lock_nesting++;

1. barrier();  /\* critical section after entry code. \*/

1. }

1. void \_\_rcu_read_unlock(void)

1. {

1. struct task_struct \*t = current;

1. ```
   if (t->rcu_read_lock_nesting != 1) {
   ```

1. ```
   	--t->rcu_read_lock_nesting;
   ```

1. ```
   } else {
   ```

1. ```
   	barrier();  /* critical section before exit code. */
   ```

1. ```
   	t->rcu_read_lock_nesting = INT_MIN;
   ```

1. ```
   	barrier();  /* assign before ->rcu_read_unlock_special load */
   ```

1. ```
   	if (unlikely(ACCESS_ONCE(t->rcu_read_unlock_special)))
   ```

1. ```
   		rcu_read_unlock_special(t);
   ```

1. ```
   	barrier();  /* ->rcu_read_unlock_special load before assign */
   ```

1. ```
   	t->rcu_read_lock_nesting = 0;
   ```

1. ```
   }
   ```

1. }

当抢占发生时，\_\_schedule函数会调用rcu_note_context_switch来通知RCU更新状态，如果当前CPU处于rcu_read_lock状态，当前进程将会放入rnp->blkd_tasks阻塞队列，并呈现在rnp->gp_tasks链表中。

从上文1.3节宽限期的结束处理过程我们可以知道，rcu_gp_kthread会判断! (rnp->qsmask) && !rcu_preempt_blocked_readers_cgp(rnp))两个条件来决定GP是否完成，其中!rnp->qsmask代表每个CPU都经过一次quiescent state，quiescent state的定义与传统RCU一致；!rcu_preempt_blocked_readers_cgp(rnp)这个条件就代表了rcu是否还有阻塞的进程。

标签: [RCU](http://www.wowotech.net/tag/RCU)

---

« [Linux 3.18U盘无法正确使用](http://www.wowotech.net/226.html) | [作業系統之前的程式 for rpi2 (1) - mmu (0) : 位址轉換](http://www.wowotech.net/linux_kenrel/address-translation.html)»

**评论：**

**Linuxkernel**\
2018-06-28 14:57

作者您好，想问一下RCU中softirq_snap 与 per-cpu中softriq\[RCU_SOFTIRQ\]在GP是否完成时的关系是什么样的？

[回复](http://www.wowotech.net/kernel_synchronization/223.html#comment-6826)

**sfn0331**\
2017-12-13 11:48

作者您好，感谢您对这些知识的整理。\
现在有个问题需要请教您:\
我可以让rcu忽略某个cpu的检测吗？比如我那个cpu是isolation的，专门运行一个进程。

[回复](http://www.wowotech.net/kernel_synchronization/223.html#comment-6350)

**[wowo](http://www.wowotech.net/)**\
2017-12-13 14:10

@sfn0331：当前的kernel实现是没有考虑这种需求的。\
不过呢，你要是有代码，想这样做，应该也是有办法的，可以看看rcu_init中的实现，例如打一下rcu_process_callbacks的主意，仿照下面的判断，加入你那个cpu的条件：\
if (cpu_is_offline(smp_processor_id()))\
return;

[回复](http://www.wowotech.net/kernel_synchronization/223.html#comment-6357)

**sfn0331**\
2017-12-13 16:28

@wowo：让rcu忽略那个isolation的cpu会不会导致那个核上使用rcu_lock的代码出现异常呢？感觉忽略某个isolation的cpu也不是解决问题的方法。您有什么好的建议吗？

[回复](http://www.wowotech.net/kernel_synchronization/223.html#comment-6361)

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

  - [Linux graphic subsystem(2)\_DRI介绍](http://www.wowotech.net/graphic_subsystem/dri_overview.html)
  - [Deadline调度器之（二）：细节和使用方法](http://www.wowotech.net/process_management/dl-scheduler-2.html)
  - [教育漫谈(1)\_《乌合之众》摘录](http://www.wowotech.net/tech_discuss/jymt1.html)
  - [linux内核中的GPIO系统之（1）：软件框架](http://www.wowotech.net/gpio_subsystem/io-port-control.html)
  - [Linux graphic subsytem(1)\_概述](http://www.wowotech.net/graphic_subsystem/graphic_subsystem_overview.html)

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

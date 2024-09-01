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

作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2014-9-18 23:42 分类：[电源管理子系统](http://www.wowotech.net/sort/pm_subsystem)

#### 1. 前言

Autosleep也是从Android wakelocks补丁集中演化而来的（[Linux电源管理(9)_wakelocks](http://www.wowotech.net/linux_kenrel/wakelocks.html)），用于取代Android wakelocks中的自动休眠功能。它基于wakeup source实现，从代码逻辑上讲，autosleep是一个简单的功能，但背后却埋藏着一个值得深思的话题：

计算机的休眠（通常是STR、Standby、Hibernate等suspend操作），应当在什么时候、由谁触发？

蜗蜗在“[Linux电源管理(2)_Generic PM之基本概念和软件架构](http://www.wowotech.net/linux_kenrel/generic_pm_architecture.html)”中有提过，在传统的操作场景下，如PC、笔记本电脑，这个问题很好回答：由用户、在其不想或不再使用时。

但在移动互联时代，用户随时随地都可能使用设备，上面的回答就不再成立，怎么办？这时，Android提出了“Opportunistic suspend（这个词汇太传神了，很难用简洁的中文去翻译，就不翻译了）”的理论，通俗的讲，就是“逮到机会就睡”。而autosleep功能，无论是基于Android wakelocks的autosleep，还是基于wakeup source的autosleep，都是为了实现“Opportunistic suspend”。

相比较“对多样的系统组件单独控制”的电源管理方案（如Linux kernel的Dynamic PM），“Opportunistic suspend”是非常简单的，只要检测到系统没有事情在做（逮到机会），就suspend整个系统。这对系统的开发人员（特别是driver开发者）来说，很容易实现，几乎不需要特别处理。

但困难的是，“系统没有事情在做”的判断依据是什么？能判断准确吗？会不会浪费过多的资源在"susend->resume-supsend…”的无聊动作上？如果只有一个设备在做事情，其它设备岂不是也得陪着耗电？等等…

所以，实现“Opportunistic suspend”机制的autosleep功能，是充满争议的。说实话，也是不优雅的。但它可以解燃眉之急，因而虽然受非议，却在Android设备中广泛使用。

其实Android中很多机制都是这样的（如wakelocks，如binder，等等），可以这样比方：Android是设计中的现实主义，Linux kernel是设计中的理想主义，当理想和现实冲突时，怎么调和？不只是Linux kernel，其它的诸如设计、工作和生活，都会遇到类似的冲突，怎么对待？没有答案，但有一个原则：不要偏执，不要试图追求非黑即白的真理！

我们应该庆幸有Android这样的开源软件，让我们可以对比，可以思考。偏题有点远，言归正传吧，去看看autosleep的实现。

#### 2. 功能总结和实现原理

经过前言的瞎聊，Autosleep的功能很已经很直白了，“系统没有事情在做”的时候，就将系统切换到低功耗状态。

根据使用场景，低功耗状态可以是Freeze、[Standby、Suspend to RAM（STR）](http://www.wowotech.net/linux_kenrel/suspend_and_resume.html)和Suspend to disk（STD）中的任意一种。而怎么判断系统没有事情在做呢？依赖[wakeup events framework](http://www.wowotech.net/linux_kenrel/wakeup_events_framework.html)。只要系统没有正在处理和新增的wakeup events，就尝试suspend，如果suspend的过程中有events产生，再resume就是了。

由于suspend/resume的操作如此频繁，解决同步问题就越发重要，这也要依赖[wakeup events framework](http://www.wowotech.net/linux_kenrel/wakeup_events_framework.html)及其[wakeup count](http://www.wowotech.net/linux_kenrel/wakeup_count.html)功能。

#### 3. 在电源管理中的位置

autosleep的实现位于kernel/power/autosleep.c中，基于wakeup count和suspend & hibernate功能，并通过PM core的main模块向用户空间提供sysfs文件（/sys/power/autosleep）

[![Autosleep architecture](http://www.wowotech.net/content/uploadfile/201409/0130d03a1c50db886306fc8ef117845c20140918154225.gif "Autosleep architecture")](http://www.wowotech.net/content/uploadfile/201409/8c2a922ba16e543e769f0607a3fc86fb20140918154224.gif)

注1：我们在“[Linux电源管理(8)_Wakeup count功能](http://www.wowotech.net/linux_kenrel/wakeup_count.html)”中，讨论过wakeup count功能，本文的autosleep，就是使用wakeup count的一个实例。

#### 4. 代码分析

**4.1 /sys/power/autosleep**

/sys/power/autosleep是在kernel/power/main.c中实现的，如下：

   1: #ifdef CONFIG_PM_AUTOSLEEP

   2: static ssize_t autosleep_show(struct kobject *kobj,

   3:                               struct kobj_attribute *attr,

   4:                               char *buf)

   5: {

   6:         suspend_state_t state = pm_autosleep_state();

   7:  

   8:         if (state == PM_SUSPEND_ON)

   9:                 return sprintf(buf, "off\n");

  10:  

  11: #ifdef CONFIG_SUSPEND

  12:         if (state < PM_SUSPEND_MAX)

  13:                 return sprintf(buf, "%s\n", valid_state(state) ?

  14:                                                 pm_states[state] : "error");

  15: #endif

  16: #ifdef CONFIG_HIBERNATION

  17:         return sprintf(buf, "disk\n");

  18: #else

  19:         return sprintf(buf, "error");

  20: #endif

  21: }

  22:  

  23: static ssize_t autosleep_store(struct kobject *kobj,

  24:                                struct kobj_attribute *attr,

  25:                                const char *buf, size_t n)

  26: {

  27:         suspend_state_t state = decode_state(buf, n);

  28:         int error;

  29:  

  30:         if (state == PM_SUSPEND_ON

  31:             && strcmp(buf, "off") && strcmp(buf, "off\n"))

  32:                 return -EINVAL;

  33:  

  34:         error = pm_autosleep_set_state(state);

  35:         return error ? error : n;

  36: }

  37:  

  38: power_attr(autosleep);

  39: #endif /* CONFIG_PM_AUTOSLEEP */

> a）autosleep不是一个必须的功能，可以通过CONFIG_PM_AUTOSLEEP打开或关闭该功能。
> 
> b）autosleep文件和state文件类似：
> 
>      读取，返回“freeze”，“standby”，“mem”，“disk”， “off”，“error”等6个字符串中的一个，表示当前autosleep的状态，分别是auto freeze、auto standby、auto STR、auto STD、autosleep功能关闭和当前系统不支持该autosleep的错误指示；
> 
>      写入freeze”，“standby”，“mem”，“disk”， “off”等5个字符串中的一个，代表将autosleep切换到指定状态。
> 
> c）autosleep的读取，由pm_autosleep_state实现；autosleep的写入，由pm_autosleep_set_state实现。这两个接口为autosleep模块提供的核心接口，位于kernel/power/autosleep.c中。

**4.2 pm_autosleep_init**

开始之前，先介绍一下autosleep的初始化函数，该函数在kernel PM初始化时（.\kernel\power\main.c:pm_init）被调用，负责初始化autosleep所需的2个全局参数：

1）一个名称为“autosleep”的wakeup source（autosleep_ws），在autosleep执行关键操作时，阻止系统休眠（我们可以从中理解wakeup source的应用场景和使用方法）。

2）一个名称为“autosleep”的有序workqueue，用于触发实际的休眠动作（休眠应由经常或者线程触发）。这里我们要提出2个问题：什么是有序workqueue？为什么要使用有序workqueue？后面分析代码时会有答案。

如下：

   1: int __init pm_autosleep_init(void)

   2: {

   3:         autosleep_ws = wakeup_source_register("autosleep");

   4:         if (!autosleep_ws)

   5:                 return -ENOMEM;

   6:  

   7:         autosleep_wq = alloc_ordered_workqueue("autosleep", 0);

   8:         if (autosleep_wq)

   9:                 return 0;

  10:  

  11:         wakeup_source_unregister(autosleep_ws);

  12:         return -ENOMEM;

  13: }

**4.3 pm_autosleep_set_state**

pm_autosleep_set_state负责设置autosleep的状态，autosleep状态和“[Linux电源管理(5)_Hibernate和Sleep功能介绍](http://www.wowotech.net/linux_kenrel/std_str_func.html)”所描述的电源管理状态一致，共有freeze、standby、STR、STD和off五种（具体依赖于系统实际支持的电源管理状态）。具体如下：

   1: int pm_autosleep_set_state(suspend_state_t state)

   2: {

   3:  

   4: #ifndef CONFIG_HIBERNATION

   5:         if (state >= PM_SUSPEND_MAX)

   6:                 return -EINVAL;

   7: #endif

   8:  

   9:         __pm_stay_awake(autosleep_ws);

  10:  

  11:         mutex_lock(&autosleep_lock);

  12:  

  13:         autosleep_state = state;

  14:  

  15:         __pm_relax(autosleep_ws);

  16:  

  17:         if (state > PM_SUSPEND_ON) {

  18:                 pm_wakep_autosleep_enabled(true);

  19:                 queue_up_suspend_work();

  20:         } else {

  21:                 pm_wakep_autosleep_enabled(false);

  22:         }

  23:  

  24:         mutex_unlock(&autosleep_lock);

  25:         return 0;

  26: }

> a）判断state是否合法。
> 
> b）调用[__pm_stay_awake](http://www.wowotech.net/linux_kenrel/wakeup_events_framework.html)，确保系统不会休眠。
> 
> c）将state保存在一个全局变量中（autosleep_state）。
> 
> d）调用[__pm_relax](http://www.wowotech.net/linux_kenrel/wakeup_events_framework.html)，允许系统休眠。
> 
> e）根据state的状态off还是其它，调用[wakeup events framework](http://www.wowotech.net/linux_kenrel/wakeup_events_framework.html)提供的接口pm_wakep_autosleep_enabled，使能或者禁止autosleep功能。
> 
> f）如果是使能状态，调用内部接口queue_up_suspend_work，将suspend work挂到autosleep workqueue中。
> 
> 注2：由这里的实例可以看出，此时wakeup source不再是wakeup events的载体，而更像一个lock（呵呵，Android wakelocks的影子）。
> 
> 注3：  
> 该接口并没有对autosleep state的当前值做判断，也就意味着用户程序可以不停的调用该接口，设置autosleep state，如写“mem”，写“freeze”，写“disk”等等。那么suspend work将会多次queue到wrokqueue上。  
> 而在多核CPU上，普通的workqueue是可以在多个CPU上并行执行多个work的。这恰恰是autosleep所不能接受的，因此autosleep workqueue就必须是orderd workqueue。所谓ordered workqueue，就是统一时刻最多执行一个work的worqueue（具体可参考include\linux\workqueue.h中的注释）。  
> 那我们再问，为什么不判断一下状态内？首先，orderd workqueue可以节省资源。其次，这样已经够了，何必多费心思呢？简洁就是美。  

pm_wakep_autosleep_enabled主要用于更新wakeup source中和autosleep有关的信息，代码和执行逻辑如下：

   1: #ifdef CONFIG_PM_AUTOSLEEP

   2: /**

   3:  * pm_wakep_autosleep_enabled - Modify autosleep_enabled for all wakeup sources.

   4:  * @enabled: Whether to set or to clear the autosleep_enabled flags.

   5:  */

   6: void pm_wakep_autosleep_enabled(bool set)

   7: {

   8:         struct wakeup_source *ws;

   9:         ktime_t now = ktime_get();

  10:  

  11:         rcu_read_lock();

  12:         list_for_each_entry_rcu(ws, &wakeup_sources, entry) {

  13:                 spin_lock_irq(&ws->lock);

  14:                 if (ws->autosleep_enabled != set) {

  15:                         ws->autosleep_enabled = set;

  16:                         if (ws->active) {

  17:                                 if (set)

  18:                                         ws->start_prevent_time = now;

  19:                                 else

  20:                                         update_prevent_sleep_time(ws, now);

  21:                         }

  22:                 }

  23:                 spin_unlock_irq(&ws->lock);

  24:         }

  25:         rcu_read_unlock();

  26: }

  27: #endif /* CONFIG_PM_AUTOSLEEP */

> a）更新系统所有wakeup souce的autosleep_enabled标志（太浪费了！！）。
> 
> b）如果wakeup source处于active状态（意味着它会阻止autosleep），且当前autosleep为enable，将start_prevent_time设置为当前实现（开始阻止）。
> 
> c）如果wakeup source处于active状态，且autosleep为disable（说明这个wakeup source一直坚持到autosleep被禁止），调用update_prevent_sleep_time接口，更新wakeup source的prevent_sleep_time。

queue_up_suspend_work比较简单，就是把suspend_work挂到workqueue，等待被执行。而suspend_work的处理函数为try_to_suspend，如下：

   1: static DECLARE_WORK(suspend_work, try_to_suspend);

   2:  

   3: void queue_up_suspend_work(void)

   4: {

   5:         if (autosleep_state > PM_SUSPEND_ON)

   6:                 queue_work(autosleep_wq, &suspend_work);

   7: }

**4.4 try_to_suspend**

try_to_suspend是suspend的实际触发者，代码如下：

   1: static void try_to_suspend(struct work_struct *work)

   2: {

   3:         unsigned int initial_count, final_count;

   4:  

   5:         if (!pm_get_wakeup_count(&initial_count, true))

   6:                 goto out;

   7:  

   8:         mutex_lock(&autosleep_lock);

   9:  

  10:         if (!pm_save_wakeup_count(initial_count) ||

  11:                 system_state != SYSTEM_RUNNING) {

  12:                 mutex_unlock(&autosleep_lock);

  13:                 goto out;

  14:         }

  15:  

  16:         if (autosleep_state == PM_SUSPEND_ON) {

  17:                 mutex_unlock(&autosleep_lock);

  18:                 return;

  19:         }

  20:         if (autosleep_state >= PM_SUSPEND_MAX)

  21:                 hibernate();

  22:         else

  23:                 pm_suspend(autosleep_state);

  24:  

  25:         mutex_unlock(&autosleep_lock);

  26:  

  27:         if (!pm_get_wakeup_count(&final_count, false))

  28:                 goto out;

  29:  

  30:         /*

  31:          * If the wakeup occured for an unknown reason, wait to prevent the

  32:          * system from trying to suspend and waking up in a tight loop.

  33:          */

  34:         if (final_count == initial_count)

  35:                 schedule_timeout_uninterruptible(HZ / 2);

  36:  

  37:  out:

  38:         queue_up_suspend_work();

  39: }

> 该接口是wakeup count的一个例子，根据我们在“[Linux电源管理(8)_Wakeup count功能](http://www.wowotech.net/linux_kenrel/wakeup_count.html)”的分析，就是read wakeup count，write wakeup count，suspend，具体为：
> 
> a）调用pm_get_wakeup_count（block为true），获取wakeup count，保存在initial_count中。如果有wakeup events正在处理，阻塞等待。
> 
> b）将读取的count，写入。如果成功，且当前系统状态为running，根据autosleep状态，调用hibernate或者pm_suspend，suspend系统。
> 
> d）如果写count失败，说明读写的过程有events产生，退出，进行下一次尝试。
> 
> e）如果suspend的过程中，或者是suspend之后，产生了events，醒来，再读一次wakeup count（此时不再阻塞），保存在final_count中。
> 
> f）如果final_count和initial_count相同，发生怪事了，没有产生events，竟然醒了。可能有异常，不能再立即启动autosleep（恐怕陷入sleep->wakeup->sleep->wakeup的快速loop中），等待0.5s，再尝试autosleep。

**4.5 pm_autosleep_state**

该接口比较简单，获取autosleep_state的值返回即可。

_原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/linux_kenrel/autosleep.html)。_

标签: [Linux](http://www.wowotech.net/tag/Linux) [PM](http://www.wowotech.net/tag/PM) [电源管理](http://www.wowotech.net/tag/%E7%94%B5%E6%BA%90%E7%AE%A1%E7%90%86) [autosleep](http://www.wowotech.net/tag/autosleep)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Linux kernel中断子系统之（五）：驱动申请中断API](http://www.wowotech.net/irq_subsystem/request_threaded_irq.html) | [Linux电源管理(9)_wakelocks](http://www.wowotech.net/pm_subsystem/wakelocks.html)»

**评论：**

**chenan**  
2023-11-02 17:16

autosleep 早已经废弃了，蜗窝也好久没更新了

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-8842)

**[schspa](http://www.wowotech.net/)**  
2023-11-06 14:43

@chenan：autosleep只是android不使用吧, 内核里边的支持还是在的吧。

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-8843)

**chenan**  
2023-11-08 20:27

@schspa：我理解的 autosleep 本身就是给 Android 打的补丁，autosleep 在内核里本身也是一个配置项。  
对于 android 的休眠机制，完成由上层控制了，只要没唤醒锁，抓住机会就睡

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-8845)

**whaaj**  
2019-03-07 11:27

我理解：Android中libsuspend为上层应用提供了申请Opportunistic Suspend的途径。例如关闭屏幕时，DisplayPowerState通过PowerManagerService实现的回调，调用libsuspend操作/sys/power/state。此时，如果没有active wakeup source，就可以suspend。但是此时如果系统开了autosleep，则libsuspend不会最终触发suspend，而是依赖autosleep去调用pm_suspend()走休眠流程。  
如我理解有误，请帮忙指正。谢谢！

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-7241)

**[wowo](http://www.wowotech.net/)**  
2019-03-07 19:18

@whaaj：Android的机制好久没看了，不太确定哦

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-7244)

**myh123**  
2020-04-24 18:17

@whaaj：你说的Android的流程我也看到了，但最后一句话是你的猜测还是代码有体现呢？我并没有看到libsuspend操作/sys/power/state的整个流程会受到autosleep的影响啊？是不是Android默认不使用内核提供的这个功能呢？

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-7969)

**myh123**  
2020-04-27 10:47

@myh123：我最新的理解是，libsuspend和autosleep是并行不悖的~

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-7972)

**chenan**  
2023-10-23 14:41

@myh123：android 9以后已经没有 libsuspend，取而代之的 SystemSuspend service

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-8838)

**xuwukong**  
2018-09-14 10:29

@wowo,进入autosleep 的时机还是Android 上层决定的吧，并不是内核有个什么东西在监测没有wakeup event就调用orderqueue进入autosleep,比如在设置->显示->休眠里面选择15s无操作就会自动灭屏待机。

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-6953)

**[wowo](http://www.wowotech.net/)**  
2018-09-17 11:50

@xuwukong：如果用的是原生的kenrel机制，上层只能决定何时开启autosleep，不能决定何时sleep。auto这个事情是kernel负责。

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-6957)

**myh123**  
2020-04-27 10:43

@wowo：我觉得上层能决定何时开启autosleep，也能通过wakelock（本质是通过wakeup source更改wakeup count）决定何时sleep吧？  
上层有两个suspendblocker，WakeLockSuspendBlocker和DisplaySuspendBlocker，对应到底层，都是写wake_lock和wake_unlock文件，从而决定是否允许sleep。允许了之后，再通过autosleep进入sleep，或者上层也会在屏幕状态改变时向state文件写入mem尝试进入suspend。

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-7971)

**xingxing.wang**  
2017-06-20 10:58

Hi wowo:  
     请问autosleep和suspend有什么区别？/sys/power/state和/sys/power/autosleep二者有什么联系，分别实现什么功能？谢谢。

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-5707)

**[wowo](http://www.wowotech.net/)**  
2017-06-20 11:43

@xingxing.wang：你的问题这篇文章已经讲了很多了（自我感觉还是比较清楚的），因此你可以先读读文章:-)

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-5710)

**xingxing.wang**  
2017-06-20 15:35

@wowo：Hi Wowo：  
   看了一下源码发现suspend和autosleep到pm_suspend(autosleep_state)之后是一样的流程，在pm_suspend()之前，能否分别列举一个suspend和autosleep的应用场景，谢谢。

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-5712)

**废柴**  
2017-12-05 15:27

@wowo：wowo你好，请问autosleep被设为mem之后，uevent还会进行传递么？  
谢谢

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-6285)

**废柴**  
2017-12-05 15:28

@wowo：@wowo  
你好，请问autosleep被设为mem之后，uevent还会进行传递么？  
谢谢

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-6286)

**废柴**  
2017-11-29 19:39

@xingxing.wang：wowo你好，请问autosleep被设为mem之后，uevent还会进行传递么？  
谢谢

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-6260)

**[wowo](http://www.wowotech.net/)**  
2017-12-05 19:45

@废柴：抱歉，您之前的留言被漏掉了，这里的uevent指的是什么呢？谁的uevent呢？

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-6287)

**废柴**  
2017-12-05 19:49

@wowo：以关机充电这个为例，battery status都是通过uevent来更新的，那么当关机充电程序进入了suspend之后，这个uevent应该停止传递了吧？

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-6288)

**[wowo](http://www.wowotech.net/)**  
2017-12-06 21:39

@废柴：是的。除非有事件可以把系统唤醒，否则不会发uevent的。

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-6292)

**cym**  
2017-02-27 09:06

hi wowo，所谓的wakeup count与autosleep两种方式有什么不一样呢？？

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-5254)

**[wowo](http://www.wowotech.net/)**  
2017-02-27 15:56

@cym：你把这篇文章读完，就得到答案啦:-)

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-5261)

**maze**  
2016-12-13 18:06

休眠应由经常或者线程触发  
休眠应由进程或者线程触发

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-5016)

**gaos**  
2016-04-06 20:33

start_prevent_time被用来做什么啊？  autosleep 除了在状态修改的时候trytosuspend    如果没有用户主动调用suspend相关的东西    autosleep是怎么做到sleep的啊？   我搜了一下使用start_prevent_time的地方  并没有别的地方来对这个值进行判断来判定是否要进入睡眠啊。

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-3803)

**[wowo](http://www.wowotech.net/)**  
2016-04-07 08:45

@gaos：某一个wakeup source处于active状态的时候，它会阻止autosleep。这个变量记录了该wakeup source开始阻止autosleep的时间点，主要用于统计wakeup source阻止autosleep的时间（prevent_sleep_time）。在电源管理debug 的时候比较有用（一直无法sleep，查找原因的时候）。

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-3805)

**gaos**  
2016-04-07 09:31

@wowo：也就是最终决定要sleep的是应用层的程序，而不是kernel内部的一个自动执行sleep?    
  
以autosleep字面上的理解，我以为它的最终目标是，在没有wakesource 处于active的一段时间后，比如说两分钟内都没有wakeevent产生， 会有某个kernel的线程来尝试suspend。    
  
  
/sys/power/autosleep 和/sys/power/state 最终的功能上不就基本上差不多了么？    
  
文章中写到：autosleep是Opportunistic suspend的一种实现，   那么通过/sys/power/state也能实现Opportunistic suspend了？    
  
  
在使用state来进行suspend的时候  需要跟wakeup_count配合使用来做同步，  
通过/sys/power/autosleep 来suspend的时候  需要跟wakeup_count陪配合做同步么？

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-3807)

**[wowo](http://www.wowotech.net/)**  
2016-04-07 11:05

@gaos：state提供方法（怎样suspend），autosleep提供策略（何时suspend），因此我不认同你2、3个问题。  
第一个问题，autosleep的“auto”，是指没有active的wakeup source的时候，sleep，我不明白你的意思。  
最后一个问题，请你再阅读一下本文的最后一段。

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-3809)

**gaos**  
2016-04-07 11:06

@wowo：又看了看  明白了  autosleep打开的时候会一直queuework   有wakesource处于active就会阻塞到 event处理完，  然后执行try_to_suspend

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-3810)

**misakamm20001**  
2015-11-16 17:08

如果开了earlysuspend功能 autosleep通常是放在earlysuspend后主动触发吧 感觉和以前检查wake_lock那套也没差多少啊

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-3062)

**[wowo](http://www.wowotech.net/)**  
2015-11-16 19:31

@misakamm20001：是的，不过early suspend可以在suspend之前处理一些事情，也就是说，还未进入suspend的流程时，就suspend一些设备。这是它存在的意义。

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-3063)

**misakamm20001**  
2015-11-17 15:31

@wowo：话说为什么不叫autosuspend呢 sleep和suspend的区别是啥

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-3072)

**[wowo](http://www.wowotech.net/)**  
2015-11-17 15:48

@misakamm20001：我的理解，suspend是通用表述，表示暂停运行。  
Sleep是suspend的一种，以睡眠的方式，暂停执行。  
不过有时候大家会混着用。

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-3074)

**qinghu**  
2015-09-28 14:58

你好，请教您个问题，android4.4 +kernel3.0版本的基于 autosleep机制进行suspend操作，是可以的。  
但是wakeup的后屏幕没有点亮，就又suspend下去了。请问，这是怎么回事呢？系统回来后，没有 wakeup source吗？需要自己添加block系统继续睡眠下去吗？如果是的话，在什么地方来添加呢？期待您的答复，谢谢！

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-2706)

**[wowo](http://www.wowotech.net/)**  
2015-09-28 19:33

@qinghu：可以从这个角度看这种现象：是谁产生这次wakeup event的呢？谁（一般指的是应用程序）又关心这个event呢？  
如果没有人关系这个event，那系统肯定睡下去。  
如果有人关心，如power management service关心power key的事件，觉得有必要点亮屏幕并保持系统清醒一段时间，它就会去拿一个timeout的锁，在一定时间内，阻止系统休眠。

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-2709)

**lynn_LY**  
2015-09-18 11:26

感谢wowo分享这么精彩的文章，有个问题我想问下，pm_wakep_autosleep_enabled，autosleep为enable是什么意思，就是允许autosleep吗？还有这个函数还更新了一些time，阻止总时间这个变量有什么作用呢？

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-2607)

**[wowo](http://www.wowotech.net/)**  
2015-09-18 12:18

@lynn_LY：是的，enable后系统会在没有wakeup event的时候自动Sleep。那些时间是一些统计信息，可以不关注。

[回复](http://www.wowotech.net/pm_subsystem/autosleep.html#comment-2609)

1 [2](http://www.wowotech.net/pm_subsystem/autosleep.html/comment-page-2#comments)

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
    
    - [PELT算法浅析](http://www.wowotech.net/process_management/pelt.html)
    - [futex基础问答](http://www.wowotech.net/kernel_synchronization/futex.html)
    - [Linux PM QoS framework(2)_PM QoS class](http://www.wowotech.net/pm_subsystem/pm_qos_class.html)
    - [Linux serial framework(1)_概述](http://www.wowotech.net/comm/serial_overview.html)
    - [计算机科学基础知识（二）:Relocatable Object File](http://www.wowotech.net/basic_subject/compile-link-load.html)
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
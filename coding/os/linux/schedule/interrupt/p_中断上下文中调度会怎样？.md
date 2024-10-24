
作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2017-3-20 19:08 分类：[进程管理](http://www.wowotech.net/sort/process_management)

# 一、前言

每一个Linux驱动工程师都知道这样一个准则：在中断上下文中不能睡眠。但是为什么interrupt context中不能调用导致睡眠的kernel API呢？如果驱动这么做会导致什么样的后果呢？这就是本文探讨的主题。为了理解这个主题，我们设计了一些非常简单的驱动程序和用户空间的程序，实际做实验观察实验效果，最后给出了结果和分析。

BTW，本文的实验在X86 64bit ＋ 标准4.4内核上完成。

# 二、测试程序

## 1、cst驱动模块

我们首先准备一个能够在中断上下文中睡眠的驱动程序，在这里我称之Context schedule test module（后文简称cst模块）。这个驱动程序类似潜伏在内核中的“捣蛋鬼”，每隔1秒随机命中一个进程，然后引发调度。首先准备一个Makefile，代码如下：

> KERNELSRC ?= /home/xxxx/work/linux-4.4.6
>
> default:\
> $(MAKE) -C $(KERNELSRC) M=$$PWD
>
> clean:\
> $(MAKE) -C $(KERNELSRC) M=$$PWD clean

按理说代码中的xxxx应该是我的名字，如果你愿意在你的环境中测试，可以修改成你的名字，当然，最重要的是需要某个版本的内核代码。在[内核升级](http://www.wowotech.net/linux_application/debian8-upgrade-kernel.html)文档中，我已经编译了/home/xxxx/work/linux-4.4.6目录下的内核，并把我的计算机升级到4.4.6的内核上，如果你愿意可以按照那份文档进行升级，否则可能会有一些版本问题需要处理。除了Makefile之外，还需要一个Kbuild文件：

> obj-m := cst.o

当然，最重要的是cst模块的源代码：

> #include\
> #include\
> #include\
> #include
>
> #define DRIVER_DESC "context schedule test driver"
>
> static struct timer_list cst_timer;
>
> static void cst_timer_handler (unsigned long data)\
> {\
> struct task_struct \*p = current;
>
> pr_info("=====in timer handler=====\\n");\
> pr_info("cst shoot %16s \[%x\] task:\\n", p->comm, preempt_count());\
> mod_timer(&cst_timer, jiffies + HZ);\
> schedule();\
> }
>
> static int \_\_init cst_init(void)\
> {\
> init_timer(&cst_timer);\
> cst_timer.function = cst_timer_handler;\
> cst_timer.expires = jiffies + HZ;\
> add_timer(&cst_timer);
>
> pr_info(DRIVER_DESC " : init on cpu:%d\\n", smp_processor_id());\
> return 0;\
> }\
> module_init(cst_init);
>
> static void \_\_exit cst_exit(void)\
> {\
> del_timer_sync(&cst_timer);\
> pr_info(DRIVER_DESC " : exit\\n");\
> }\
> module_exit(cst_exit);
>
> MODULE_DESCRIPTION(DRIVER_DESC);\
> MODULE_AUTHOR("linuxer ");\
> MODULE_LICENSE("GPL");

代码非常的简单，无需多说，直接make就可以编译得到cst.ko的内核模块了。

## 2、用户空间测试程序

为了更方便的测试，我们需要准备一个“受害者”，代码如下：

> #include\
> #include
>
> int main(int argc, char \*\*argv)\
> {\
> int i = 0;
>
> while (1) {\
> sqrt (rand ());
>
> if ((i % 0xffffff) == 0)\
> printf ("=\\n");
>
> if ((i % 0xfffffff) == 0)\
> printf ("haha......still alive\\n");
>
> i++;\
> }
>
> return 0;\
> }

这段代码也很简单：不断的产生一个随机数，并运算其平方根，在使得的时候，向用户输出一些字符，表明自己的状态。当程序执行起来的时候，大部分时间在用户态（运算），偶尔进入内核态（printf）。这个进程并不知道在内核态有一个cst的模块，每隔一秒就发射一只休眠之箭，可能命中用户态，也可能命中内核态，看运气吧，但是无论怎样，该进程被射中之后都会进入睡眠。

# 三、执行测试并观察结果

## 1、先把用户空间的测试程序跑起来

要想测试导弹（呵呵～～我们的cst模块就是一个捣蛋） 的性能，必须要有靶机或者靶舰。当然也可以不用“靶机”程序，只不过捣蛋鬼cst总是命中swapper进程，有点无趣，因此这里需要把我们用户空间的那个测试程序跑起来，让CPU先活跃起来。

需要注意的是，在多核cpu上，我们需要多跑几个“靶机”进程才能让系统不会always进入idle状态。例如我的T450是4核cpu，因此我需要运行4个靶机程序才能让系统中的4个cpu core都燥起来。可以通过下面的命令确认：

> ps –eo comm,psr | grep cst

BTW，靶机程序是cst_test。通过上面的命令，可以看到系统中运行了四个cst_test进程，分别在4个cpu上。

## 2、插入内核模块

靶机已经就绪，是时候发射捣蛋了，命令如下：

> sudo insmod ./cst.ko

一旦插入了cst内核模块，捣蛋鬼就开始运作了，每隔1秒钟发射一次，总有一个倒霉蛋被命中，被调度。当然，在我们的测试中，一般总是cst_test这个进程被命中。

## 3、观察结果

一切准备就绪，是时候搬个小板凳坐下来看好戏了。当然，我们需要一个观察的工具，输入如下命令：

> sudo tail –f /var/log/messages

在上面的cst模块中，输出并没有直接到控制台，因此我们需要通过内核日志来看看cst的运行情况。

# 四、结果和分析

## 1、结果

很奇怪，一切都是正常的，系统没有死，cst模块也运行正常，cst_test进程也始终保持alive状态，不断的运行在无聊的平方根、打印着无聊的字符串。唯一异常的是日志，每隔1秒钟就会dump stack一次。

## 2、分析

当cst模块命中cst_test进程，无论是userspace还是kernel space，都会在内核栈上保存中断发生那一点的上下文，唯一不同是如果发生在userspace，那么发生中断那一刻，内核栈是空的，而如果在kernel space，内核栈上已经有了cst_test通过系统调用进入内核的现场以及该系统调用各个函数的stack frame，当中断发生的时候，在当前内核栈的栈顶上继续压入了中断现场，之后就是中断处理的各个函数的stack frame，最后是cst_timer_handler的stack frame，由于调用了schedule函数，cst_test进程的现场被继续压入内核栈，调度器决定下一个调度的进程。

cst_test进程虽然被调度了，但是仍然在runqueue中，调度器一定会在适当的时机调度cst_test进程重新进入执行态，这时候恢复其执行就OK了，cpu执行cst_timer_handler函数schedule之后的代码，继续未完的中断上下文的执行，然后从内核栈恢复中断现场，一切又按照原路返回了。

当然，这里的测试看起来一切OK，但这并不是说可以自由的在中断上下文中调用导致睡眠的内核API，因为我们这里给出了一个简单的例子，实际上也有可能导致系统死锁。例如在内核态持有锁的时候被中断，然后发生调度。有兴趣的同学可以自己修改上面的代码实验这种情况。

## 3、why

最后还是回到这个具体的技术问题：为什么interrupt context中不能调用导致睡眠的kernel API？

我的看法是这样的：调度器是每一个OS的必备组件，在编码阶段之前，我们往往要制定下我们的设计概念。对于Linux 调度器，它的目标就是调度一个线程，一个线程就是调度实体（暂不考虑group sched）。中断上下文是不是调度实体呢？当然不是，它没有专属的task struct，内核无从调度。这是调度器设计者的决定，这样的决定让调度器设计起来简洁而美丽。

基于上面的设计概念，中断上下文（hard irq和softirq context）并不参与调度（暂不考虑中断线程化），它们是异步事件的处理机制，目标就是尽快完成处理，返回现场。因此，所有中断上下文的优先级都是高于进程上下文的，也就是说，对于用户进程（无论内核态还是用户态）或者内核线程，除非disable了CPU的本地中断，否则一旦中断发生，它们是没有任何能力阻挡中断上下文抢占当前进程上下文的执行的。

因此，Linux kernel的设计者制定了规则：

1、中断上下文不是调度实体

2、中断上下文的优先级高于进程上下文

而在中断上下文中调度毫无疑问会打破规则，因此不能在硬中断、软中断环境中调用阻塞函数。不过，在linux调度器的具体实现的时候，检测到了在中断上下文中调度schedule函数也并没有强制linux进入panic，可能是linux的开发者认为一个好的内核调度器无论如何也尽自己最大的努力让系统运行下去吧。但是，在厂商自己提供的内核中，往往修改调度器行为，在中断上下文中检测到调度就直接panic了，对于内核开发者而言，这样更好，可以尽早的发现问题。

_原创文章，转发请注明出处。蜗窝科技_

标签: [中断](http://www.wowotech.net/tag/%E4%B8%AD%E6%96%AD) [调度](http://www.wowotech.net/tag/%E8%B0%83%E5%BA%A6)

---

« [蓝牙协议分析(10)\_BLE安全机制之LE Encryption](http://www.wowotech.net/bluetooth/le_encryption.html) | [Linux调度器：进程优先级](http://www.wowotech.net/process_management/process-priority.html)»

**评论：**

**mengensun**\
2020-04-09 13:56

有一个问题:\
当前cpu上的中断计数，是被记录在当前task的私有数据结构里面的，如果调度出去，那这个当前cpu上的中断计数会不会就丢了呢？

[回复](http://www.wowotech.net/process_management/schedule-in-interrupt.html#comment-7941)

**fanfan**\
2019-04-27 18:16

实际这里的问题应该是两种，一种是实现上能不能支持中断中的调度/休眠；一种是设计上应该不应该在中断handler中调度/休眠；\
1、对于第二种大家都比较意见统一，驱动设计上不应该使用调度/休眠接口；\
2、对于第一种，实际从你的实验上可以得出结论，是能够支持在中断handler中使用调度/休眠接口的；因为中断处理函数的上下文依附于当前被中断的线程，当中断线程被休眠后，实际是可以被调度回来的。\
对于你给出的链接https://blog.csdn.net/maray/article/details/5770889，比较有趣的一点是，对于THREAD_SIZE的说明，如果THREAD_SIZE设置为8K，则休眠时中断上下文也是依附于某个进程；而如何设置为4k,则是使用专用的栈空间，也就是说如果在中断中被休眠了，是真的没有上下文了，中断就丢了？是这个意思吗

[回复](http://www.wowotech.net/process_management/schedule-in-interrupt.html#comment-7377)

**[man8266](http://www.wowotech.net/)**\
2018-12-15 18:43

个人觉得测试结果还与所运行的版本或平台有关，我猜想本文测试的：X86 64bit ＋ 标准4.4内核，中断上下文是保存在当前被中断进程的内核栈中了。在有些其他版本或平台中，中断上下文是保存在独立于进程内核栈的上下文中，对于这类配置，后果应该更严重，因为这个专门用于中断的栈无法重新被调度了（也可能前面schedule()识别出当前处于中断上下文中，根本不做任何事情，或许打印出call stack）。

即使在X86 64bit ＋ 标准4.4内核 配置中，schedule()也有可能识别出当前处于中断上下文中，根本不做任何事情。

[回复](http://www.wowotech.net/process_management/schedule-in-interrupt.html#comment-7084)

**aiLinux**\
2018-12-10 14:21

你好，针对于linuxer举的两个例子，不只是在该场景下出现，对于例子1(连续调用非递归的互斥锁)导致死锁，以及例子2(多线程不按固定顺序访问多个互斥锁)导致死锁，在用户空间的多线程编程场景中也十分的普遍。所以，个人觉得造成系统deadlock不是“ISR中不能调用引起系统调度API”的原因吧？个人觉得还是没有把根本的原因说透呢？不知道理解的对不对

[回复](http://www.wowotech.net/process_management/schedule-in-interrupt.html#comment-7075)

**aiLinux**\
2018-12-10 14:45

@aiLinux：这里有一篇比较早的博客，我觉得说的比较具体，大家可以参看一下：https://blog.csdn.net/maray/article/details/5770889

[回复](http://www.wowotech.net/process_management/schedule-in-interrupt.html#comment-7076)

**lamaboy**\
2018-12-07 16:09

有个问题，想请教下：\
就是苹果的内核设计的调度器中有一种设计是在进程 切换的时候，是不保存现场和恢复现场的，来加快调度的切换，不过限于有限的水平，针对这个实现逻辑我是想不明白。 我描述的清除不，你就看着字面意思理解下，有如下疑问：

1、进程运行一般（也可以理解为一个函数运行一部分）如果调度回来的时候他是如何开始运行，还是全新开始运行？\
2、 是不是有一些特别的task，才可以不保存现场？

[回复](http://www.wowotech.net/process_management/schedule-in-interrupt.html#comment-7073)

**冬瓜哥**\
2018-02-28 17:52

不知大侠能否深入论述一下，为何在ret_from_intr结尾调用schedule，就没啥问题，而isr内部调用，就不合适，哪些地方形成了最根本的限制？多谢

[回复](http://www.wowotech.net/process_management/schedule-in-interrupt.html#comment-6575)

**[linuxer](http://www.wowotech.net/)**\
2018-03-01 15:37

@冬瓜哥：ret_from_intr结尾调用schedule是OS的行为，即中断处理已经完成，os认为这里是一个潜在的抢占时机，如果满足一定条件，那么调用schedule抢占当前task。

如果在isr内部，那么中断处理还在进行中，调用scheduler是主动调度，即强迫当前进程（其实和这个中断没有什么关系）让出CPU。

而且中断中调用schedule这个动作违背了Linux的设计准则：中断上下文总是应该先于进程上下文执行。

[回复](http://www.wowotech.net/process_management/schedule-in-interrupt.html#comment-6578)

**[bsp](http://www.wowotech.net/)**\
2017-11-30 15:28

请教个问题，scheduler 上下文可以调用wake_up_process 吗？会不会发生死循环？\
问题背景：有一个新的cpufreq govenor叫 schedutil（kernel-4.9），它在schedler中要更新cpufreq：sugov_update_commit()，我的arch不支持fast_swtich，这个函数就发个IPI给自己，然后IPI handler中去唤醒一个线程去处理 真正的调频。\
我的疑问是，为什么不在 sugov_update_commit()去唤醒那个线程呢？\
kernel/sched/cpufreq_schedutil.c ： irq_work_queue(&sg_policy->irq_work);

[回复](http://www.wowotech.net/process_management/schedule-in-interrupt.html#comment-6270)

**[bsp](http://www.wowotech.net/)**\
2017-12-01 10:21

@bsp：scheduler 上下文应该不可以调用wake_up_process，可能会发生死循环？\
代码中（kernel-4.9: arch/arm64/kernel/entry.S）有一处是中断完成后的调度时机：\
#ifdef CONFIG_PREEMPT\
el1_preempt:\
mov     x24, lr\
1:      bl      preempt_schedule_irq            // irq en/disable is done inside\
ldr     x0, \[tsk, #TSK_TI_FLAGS\]        // get new tasks TI_FLAGS\
tbnz    x0, #TIF_NEED_RESCHED, 1b       // needs rescheduling?\
ret     x24\
#endif\
如果我在preempt_schedule_irq中调用wake_up_process会将current->ti_flags置位TIF_NEED_RESCHED，这里会发生死循环。

[回复](http://www.wowotech.net/process_management/schedule-in-interrupt.html#comment-6271)

**内核初学者**\
2017-09-20 23:43

喜欢这里讨论问题的气氛，真正进步的地方

[回复](http://www.wowotech.net/process_management/schedule-in-interrupt.html#comment-6049)

**Alulu**\
2017-06-19 09:31

既然中断上下文不会被调度，那是不是说明也不会被抢占啊？如果是这样为什么有的代码里要在中断处理函数里加自旋锁呢？

[回复](http://www.wowotech.net/process_management/schedule-in-interrupt.html#comment-5693)

**[linuxer](http://www.wowotech.net/)**\
2017-06-19 11:35

@Alulu：第一个问题：在旧的内核中，有fast handler和slow handler之分，fast handler不会被抢占，但是slow handler由于中断处理时间长，从是开中断执行，这时候，其他类型的中断可以抢占该中断的执行。不过在新的内核中，slow handler的概念被移除，所有的handler都是fast handler，都是关中断执行，因此中断handler是不可能被抢占的。\
BTW，上面说的是hardirq context，至于bottom half，其实也属于中断上下文，它是always能被hardirq context抢占的。

第二个问题：不会被抢占并不意味着不会有并发。在SMP环境下，一个CPU上的中断上下文可以和其他CPU上的中断上下文或者进程上下文并发执行，因此需要spin lock。

[回复](http://www.wowotech.net/process_management/schedule-in-interrupt.html#comment-5695)

**navadoo**\
2017-06-16 12:04

偶尔进入内核态（printf）

=====printf进入内核态怎么理解？

[回复](http://www.wowotech.net/process_management/schedule-in-interrupt.html#comment-5683)

**hello-world**\
2017-06-16 19:18

@navadoo：printf最终需要通过系统调用进入内核

[回复](http://www.wowotech.net/process_management/schedule-in-interrupt.html#comment-5685)

1 [2](http://www.wowotech.net/process_management/schedule-in-interrupt.html/comment-page-2#comments)

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

  - [Linux vm运行参数之（二）：OOM相关的参数](http://www.wowotech.net/memory_management/oom.html)
  - [Linux电源管理(15)\_PM OPP Interface](http://www.wowotech.net/pm_subsystem/pm_opp.html)
  - [Why Memory Barriers？中文翻译（上）](http://www.wowotech.net/kernel_synchronization/Why-Memory-Barriers.html)
  - [关于java单线程经常占用cpu100%分析](http://www.wowotech.net/linux_kenrel/483.html)
  - [Process Creation（二）](http://www.wowotech.net/process_management/process-creation-2.html)

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

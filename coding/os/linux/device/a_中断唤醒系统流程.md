作者：[smcdef](http://www.wowotech.net/author/531) 发布于：2018-1-1 17:03 分类：[中断子系统](http://www.wowotech.net/sort/irq_subsystem)

# 1. 前言

曾几何时，不知道你是否想过外部中断是如何产生的呢？又是如何唤醒系统的呢？在项目中，一般具有中断唤醒的设备会有一个interrupt pin硬件连接到SoC的gpio pin。一般来说，当设备需要唤醒系统的时候，会通过改变interrupt pin电平状态，而SoC会检测到这个变化，将SoC从睡眠中唤醒，该设备通过相关的子系统通知上层应用做出相应的处理。这就是中断唤醒的过程。说起来很简洁，可以说是涵盖了软硬件两大块。是不是？

为了使能设备的唤醒能力，设备驱动中会在系统suspend的时候通过enable_irq_wake(irq)接口使能设备SoC引脚的中断唤醒能力。然后呢？然后当然是万事大吉了，静静的等待设备中断的到来，最后唤醒系统。假设我们做一款手机，手机有一个压感传感器，重压点亮屏幕，轻压在灭屏的时候无响应，在亮屏的时候作为home键功能，压力值通过i2c总线读取（描述挺像iPhone8的home键！）。假如有一天，你突然发现重压按键，屏幕不亮。于是你开始探究所以然，聪明的你一定会先去用示波器测量irq pin的波形，此时你发现了重压按键，的确产生了一个电平信号的变化，此时可就怪不得硬件了。而你又发现插入USB使用ADB工具抓取log的情况下（Android的adb工具需要通过USB协议通信，一般不会允许系统休眠），重压可以亮屏。此时，我觉得就很有可能是唤醒系统了，但是系统醒来后又睡下去了，而你注册的中断服务函数中的代码没有执行完成就睡了。什么情况下会出现呢？试想一下，你通过request_irq接口注册的handle函数中queue work了一个延迟工作队列（主要干活的，类似下半部吧），由于时间太长，还没来得及调度呢，系统又睡下了，虽然你不愿意，但是事情就是可能这样发生的。那这一切竟然是为什么呢？作为驱动工程师最关注的恐怕就是如何避开这些问题呢？

1. 设备唤醒cpu之后是立即跳转中断向量表指定的位置吗？如果不是，那么是什么时候才会跳转呢？
1. 已经跳转到中断服务函数开始执行代码，后续就会调用你注册的中断handle 代码吗？如果不是，那中断服务函数做什么准备呢？而你注册的中断handle又会在什么时候才开始执行呢？
1. 假如register_thread_irq方式注册的threaded irq中调用msleep(1000)，睡眠1秒，请问系统此时会继续睡下去而没调度回来吗？因此导致msleep后续的操作没有执行。
1. 如果在注册的中断handle中把主要的操作都放在delayed work中，然后queue delayed work，work延时1秒执行，请问系统此时会继续睡下去而没调度delayed work 吗？因此导致delayed work 中的操作没有执行呢？
1. 如果4)成立的话，我们该如何编程避免这个问题呢？\
   好了，本片文章就为你解答所有的疑问。\
   注：文章代码分析基于linux-4.15.0-rc3。

2) 中断唤醒流程

现在还是假设你有一个上述的设备，现在你开始编写driver代码了。假设部分代码如下：

```cpp
1. static irqreturn_t smcdef_event_handler(int irq, void *private)
2. {
3.     /* do something you want, like report input events through input subsystem */

5.     return IRQ_HANDLED;
6. }

8. static int smcdef_suspend(struct device *dev)
9. {
10.     enable_irq_wake(irq);
11. }

13. static int smcdef_resume(struct device *dev)
14. {
15.     disable_irq_wake(irq);
16. }

18. static int smcdef_probe(struct i2c_client *client,
19.         const struct i2c_device_id *id)
20. {
21.     /* ... */
22.     request_thread_irq(irq,
23.             smcdef_event_handler,
24.             NULL,
25.             IRQF_TRIGGER_FALLING,
26.             "smcdef",
27.             pdata);

29.     return 0;
30. }

32. static int smcdef_remove(struct i2c_client *client)
33. {
34.     return 0;
35. }

37. static const struct of_device_id smcdef_dt_ids[] = {
38.     {.compatible = "wowo,smcdef" },
39.     { }
40. };
41. MODULE_DEVICE_TABLE(of, smcdef_dt_ids);

43. static SIMPLE_DEV_PM_OPS(smcdef_pm_ops, smcdef_suspend, smcdef_resume);

45. static struct i2c_driver smcdef_driver = {
46.     .driver = {
47.         .name             = "smcdef",
48.         .of_match_table = of_match_ptr(smcdef_dt_ids),
49.         .pm                = &smcdef_pm_ops,
50.     },
51.     .probe  = smcdef_probe,
52.     .remove = smcdef_remove,
53. };
54. module_i2c_driver(smcdef_driver);

56. MODULE_AUTHOR("smcdef");
57. MODULE_DESCRIPTION("IRQ test");
58. MODULE_LICENSE("GPL");
```

在probe函数中通过request_thread_irq接口注册驱动的中断服务函数smcdef_event_handler，注意这里smcdef_event_handler的执行环境是中断上下文，thread_fn的方式下面也会介绍。

2.1. enable_irq_wake

当系统睡眠（echo "mem" > /sys/power/state）的时候，回想一下suspend的流程就会知道，最终会调用smcdef_suspend使能中断唤醒功能。enable_irq_wake主要工作是在irq_set_irq_wake中完成，代码如下：

```cpp
1. int irq_set_irq_wake(unsigned int irq, unsigned int on) 
2. {
3.     unsigned long flags;
4.     struct irq_desc *desc = irq_get_desc_buslock(irq, &flags, IRQ_GET_DESC_CHECK_GLOBAL);
5.     int ret = 0;

7.     /* wakeup-capable irqs can be shared between drivers that
8.      * don't need to have the same sleep mode behaviors.
9.      */
10.     if (on) {
11.         if (desc->wake_depth++ == 0) {
12.             ret = set_irq_wake_real(irq, on);
13.             if (ret)
14.                 desc->wake_depth = 0;
15.             else
16.                 irqd_set(&desc->irq_data, IRQD_WAKEUP_STATE);
17.         }
18.     } else {
19.         if (desc->wake_depth == 0) {
20.             WARN(1, "Unbalanced IRQ %d wake disable\n", irq);
21.         } else if (--desc->wake_depth == 0) {
22.             ret = set_irq_wake_real(irq, on);
23.             if (ret)
24.                 desc->wake_depth = 1;
25.             else
26.                 irqd_clear(&desc->irq_data, IRQD_WAKEUP_STATE);
27.         }
28.     }
29.     irq_put_desc_busunlock(desc, flags);
30.     return ret;
31. }
```

> 1. 首先在set_irq_wake_real函数中通过irq_chip的irq_set_wake回调函数设置SoC相关wakeup寄存器使能中断唤醒功能，如果不使能的话，即使设备在那疯狂的产生中断signal，SoC可不会理睬你哦！
> 1. 设置irq的state为IRQD_WAKEUP_STATE，这步很重要，suspend流程会用到的。

2.2. Suspend to RAM流程

先画个图示意一下系统Suspend to RAM流程。我们可以看到图片画的很漂亮。从enter_state开始到suspend_ops->enter()结束。对于suspend_ops->enter()调用，我的理解是CPU停在这里了，待到醒来的时候，就从这里开始继续前行的脚步。

[![STR流程.png](http://www.wowotech.net/content/uploadfile/201801/7dc41514798124.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201801/7dc41514798124.png)

> 1. enable_irq_wake()可以有两种途径，一是在driver的suspend函数中由驱动开发者主动调用；二是在driver的probe函数中调用dev_pm_set_wake_irq()和device_init_wakeup()。因为suspend的过程中会通过dev_pm_arm_wake_irq()打开所有wakeup source的irq wake功能。我更推荐途径1，因为系统已经帮我们做了，何必重复造轮子呢！
> 1. 对于已经enable 并且使能wakeup的irq，置位IRQD_WAKEUP_ARMED，然后等待IRQ handler和threaded handler执行完成。后续详细分析这一块。
> 1. 针对仅仅enable的irq，设置IRQS_SUSPENDED标志位，并disable irq。
> 1. 图中第④步关闭noboot cpu，紧接着第⑤步diasble boot cpu的irq，即cpu不再响应中断。
> 1. 在cpu sleep之前进行最后一步操作就是syscore suspend。既然是最后suspend，那一定是其他device都依赖的系统核心驱动。后面说说什么的设备会注册syscore suspend。

2.3. resume流程

假设我们使用的是gic-v3代码，边沿触发中断设备。现在设备需要唤醒系统了，产生一个边沿电平触发中断。此时会唤醒boot cpu（因为noboot cpu在suspend的时候已经被disable）。你以为此时就开始跳转中断服务函数了吗？no！还记得上一节说的吗？suspend之后已经diasble boot cpu的irq，因此中断不会立即执行。什么时候会执行呢？当然是等到local_irq_enable()之后。resume流程如下图。

[![resume流程.png](http://www.wowotech.net/content/uploadfile/201801/a16a1514798123.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201801/a16a1514798123.png)

> 1. 首先执行syscore resume，马上为你讲解syscore的用意。
> 1. arch_suspend_enable_irqs()结束后就会进入中断服务函数，因为中断打开了，interrupt controller的pending寄存器没有清除，因此触发中断。你以为此时会调用到你注册的中断handle吗？错了！此时中断服务函数还没执行你注册的handle就返回了。马上为你揭晓为什么。先等等。

先说到这里，先看看什么是syscore。

2.4. system core operations有什么用？

先想一想为什么要等到syscore_resume之后才arch_suspend_enable_irqs()呢？试想一下，系统刚被唤醒，最重要的事情是不是先打开相关的时钟以及最基本driver（例如：gpio、irq_chip等）呢？因此syscore_resume主要是clock以及gpio的驱动resume，因为这是其他设备依赖的最基本设备。回想一下上一节中Susoend to RAM流程中，syscore_suspend也同样是最后suspend的，毕竟人家是大部分设备的基础，当然最后才能suspend。可以通过register_syscore_ops()接口注册syscore operation。

2.5. gic interrupt controller中断执行流程

接下来arch_suspend_enable_irqs()之后就是中断流程了，其函数执行流程如下。

[![中断执行流程.png](http://www.wowotech.net/content/uploadfile/201801/e8b41514798125.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201801/e8b41514798125.png)

图片中是一个中断从汇编开始到结束的流程。假设我们的设备是边沿触发中断，那么一定会执行到handle_edge_irq()，如果你不想追踪代码，或者对中断流程不熟悉，我教你个方法，在注册的中断handle中加上一句WARN_ON(1);语句，请查看log信息即可。handle_edge_irq()代码如下：

```cpp
1. void handle_edge_irq(struct irq_desc *desc)
2. {
3.     raw_spin_lock(&desc->lock);

5.     desc->istate &= ~(IRQS_REPLAY | IRQS_WAITING);

7.     if (!irq_may_run(desc)) {
8.         desc->istate |= IRQS_PENDING;
9.         mask_ack_irq(desc);
10.         goto out_unlock;
11.     }

13.     /*
14.      * If its disabled or no action available then mask it and get
15.      * out of here.
16.      */
17.     if (irqd_irq_disabled(&desc->irq_data) || !desc->action) {
18.         desc->istate |= IRQS_PENDING;
19.         mask_ack_irq(desc);
20.         goto out_unlock;
21.     }

23.     kstat_incr_irqs_this_cpu(desc);

25.     /* Start handling the irq */
26.     desc->irq_data.chip->irq_ack(&desc->irq_data);

28.     do {
29.         if (unlikely(!desc->action)) {
30.             mask_irq(desc);
31.             goto out_unlock;
32.         }

34.         /*
35.          * When another irq arrived while we were handling
36.          * one, we could have masked the irq.
37.          * Renable it, if it was not disabled in meantime.
38.          */
39.         if (unlikely(desc->istate & IRQS_PENDING)) {
40.             if (!irqd_irq_disabled(&desc->irq_data) &&
41.                 irqd_irq_masked(&desc->irq_data))
42.                 unmask_irq(desc);
43.         }

45.         handle_irq_event(desc);

47.     } while ((desc->istate & IRQS_PENDING) &&
48.          !irqd_irq_disabled(&desc->irq_data));

50. out_unlock:
51.     raw_spin_unlock(&desc->lock);
52. }
```

> 1) irq_may_run()判断irq是否有IRQD_WAKEUP_ARMED标志位，当然这里是有的。随后调用irq_pm_check_wakeup()清除IRQD_WAKEUP_ARMED flag顺便置位IRQS_SUSPENDED和IRQS_PENDING flag，又irq_disable关闭了中断。\
> 2) irq_may_run()返回false，因此这里直接返回了，所以你注册的中断handle并没有执行。你绝望，也没办法。当然这里也可以知道，唤醒系统的这次中断注册的handle的执行环境不是硬件中断上下文。

2.6. dpm_resume_noirq()

我们来继续分析2.3节resume的后续流程，把图继续搬过来。

[![resume流程.png](http://www.wowotech.net/content/uploadfile/201801/a16a1514798123.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201801/a16a1514798123.png)

> 1) 继续enable所有的noboot cpu之后，开始dpm_resume_noirq()。这里为什么起名noirq呢？中断已经可以响应了，我猜测是这样的：虽然可以响应中断，但是也是仅限于suspend之前的enable_irq_wake的irq，因为其他irq已经被disable。并且具有唤醒功能的irq也仅仅是进入中断后设置一些flag就立即退出了，没有执行irq handle，因此相当于noirq。\
> 2) dpm_noirq_resume_devices()会调用"noirq" resume callbacks，这个就是struct dev_pm_ops结构体的resume_noirq成员。那么什么的设备驱动需要填充resume_noirq成员呢？我们考虑一个事情，到现在为止唤醒系统的irq的handle还没有执行，如果注册的中断handle是通过spi、i2c等方式通信，那么在即将执行之前，我们是不是应该首先resume spi、i2c等设备呢！所以说，很多设备依赖的设备，尽量填充resume_noirq成员，这样才比较合理。毕竟唤醒的设备是要使用的嘛！而gpio驱动就适合syscore resume，因为这里i2c设备肯定依赖gpio设备。大家可以看看自己平台的i2c、spi等设备驱动是不是都实现resume_noirq成员。当然了，前提是这个设备需要resume操作，如果不需要resume就可以使用，那么完全没有必要resume_noirq。所以，写driver也是要考虑很多问题的，driver应该实现哪些dev_pm_ops的回调函数？\
> 3) resume_device_irqs中会帮我们把已经enable_irq_wake的设备进行disable_irq_wake，但是前提是driver中通过2.2节中途径1的方式。\
> 4) resume_irqs继续调用，最终会enable所有在susoend中关闭的irq。\
> 5) check_irq_resend才是真正触发你注册的中断handle执行的真凶。

check_irq_resend代码如下:

```cpp
1. void check_irq_resend(struct irq_desc *desc)
2. {
3.     /*
4.      * We do not resend level type interrupts. Level type
5.      * interrupts are resent by hardware when they are still
6.      * active. Clear the pending bit so suspend/resume does not
7.      * get confused.
8.      */
9.     if (irq_settings_is_level(desc)) {
10.         desc->istate &= ~IRQS_PENDING;
11.         return;
12.     }
13.     if (desc->istate & IRQS_REPLAY)
14.         return;
15.     if (desc->istate & IRQS_PENDING) {
16.         desc->istate &= ~IRQS_PENDING;
17.         desc->istate |= IRQS_REPLAY;

19.         if (!desc->irq_data.chip->irq_retrigger ||
20.             !desc->irq_data.chip->irq_retrigger(&desc->irq_data)) {

22.             unsigned int irq = irq_desc_get_irq(desc);

24.             /* Set it pending and activate the softirq: */
25.             set_bit(irq, irqs_resend);
26.             tasklet_schedule(&resend_tasklet);
27.         }
28.     }
29. }
```

由于在之前分析已经设置了IRQS_PENDING flag，因此这里会tasklet_schedule(&resend_tasklet)并且置位irqs_resend变量中相应的bit位，代表软中断触发。然后就开始tasklet_schedule最终会唤醒ksoftirqd线程，在ksoftirqd线程中会调用你注册的中断handle。具体调用过程可以参考wowo的softirq和tasklet文章。这里我们也可以得出中断handle执行的上下文环境是软中断上下文的结论。当然我们还是有必要分析一下tasklet最后一步resend_irqs()函数的作用，代码如下：

```cpp
1. /* Bitmap to handle software resend of interrupts: */
2. static DECLARE_BITMAP(irqs_resend, IRQ_BITMAP_BITS);

4. /*
5.  * Run software resends of IRQ's
6.  */
7. static void resend_irqs(unsigned long arg)
8. {
9.     struct irq_desc *desc;
10.     int irq;

12.     while (!bitmap_empty(irqs_resend, nr_irqs)) {
13.         irq = find_first_bit(irqs_resend, nr_irqs);
14.         clear_bit(irq, irqs_resend);
15.         desc = irq_to_desc(irq);
16.         local_irq_disable();
17.         desc->handle_irq(desc);
18.         local_irq_enable();
19.     }
20. }
21. /* Tasklet to handle resend: */
22. static DECLARE_TASKLET(resend_tasklet, resend_irqs, 0);
```

> 1)  irqs_resend是一个unsigned int类型的数组，每一个bit都代表一个irq是否resend。\
> 2)  resend_irqs是注册的resend_tasklet的callback函数，当tasklet_schedule(&resend_tasklet)之后就会被调度执行。\
> 3)  在resend_irqs函数中，通过判断irqs_resend变量中的每一个bit位是否为1（即是否需要resend，也就是调用irq注册的中断handle）。\
> 好了，现在可以解答清楚的解答第一个问题了：设备唤醒cpu之后是立即跳转中断向量表指定的位置吗？如果不是，那么是什么时候才会跳转呢？设备唤醒cpu之后并不是立即跳转中断向量执行中断，而是等到syscore_resume以及打开cpu中断之后才开始。第二个问题也有答案了，已经跳转到中断服务函数开始执行代码，后续就会调用你注册的中断handle 吗？如果不是，那中断服务函数做什么准备呢？而你注册的中断handle又会在什么时候才开始执行呢？第一次跳转执行中断仅仅是设置相关的flag并且disable_irq，在执行完成设备的resume noirq回调函数之后通过check_irq_resend中调度tasklet，最终执行注册的中断handle，至于为什么要这么做，前面分析也给了答案。

2.7. IRQ handler会睡眠吗？

你想过request_thread_irq函数注册的hardirq handler或者是threaded handler会执行一半时，系统会再一次的休眠下去吗？再看看2.2节的图，实际上对于所有已经打开的irq在suspend_device_irqs()会调用synchronize_irq()等待正在处理的hardirq handler或者threaded handler。synchronize_irq()代码如下：

```cpp
1. void synchronize_irq(unsigned int irq)
2. {
3.     struct irq_desc *desc = irq_to_desc(irq);

5.     if (desc) {
6.         __synchronize_hardirq(desc);
7.         /*
8.          * We made sure that no hardirq handler is
9.          * running. Now verify that no threaded handlers are
10.          * active.
11.          */
12.         wait_event(desc->wait_for_threads,
13.                !atomic_read(&desc->threads_active));
14.     }
15. }
```

> 1) \_\_synchronize_hardirq()是等待hardirq handler执行完毕。\
> 2) 只要threads_active计数不为0就等待threaded handler执行完毕。

\_\_synchronize_hardirq()代码如下：

```cpp
1. static void __synchronize_hardirq(struct irq_desc *desc)
2. {
3.     bool inprogress;

5.     do {
6.         unsigned long flags;

8.         /*
9.          * Wait until we're out of the critical section.  This might
10.          * give the wrong answer due to the lack of memory barriers.
11.          */
12.         while (irqd_irq_inprogress(&desc->irq_data))
13.             cpu_relax();

15.         /* Ok, that indicated we're done: double-check carefully. */
16.         raw_spin_lock_irqsave(&desc->lock, flags);
17.         inprogress = irqd_irq_inprogress(&desc->irq_data);
18.         raw_spin_unlock_irqrestore(&desc->lock, flags);

20.         /* Oops, that failed? */
21.     } while (inprogress);
22. }
```

irqd_irq_inprogress()是判断irq是否设置了IRQD_IRQ_INPROGRESS 标志位。标识hardirq thread正在执行，IRQD_IRQ_INPROGRESS在handle_irq_event()执行开始设置，等到handle_irq_event_percpu()执行完毕之后，同样在handle_irq_event()之后清除。因此hardirq handler执行结束之前系统不会睡眠。那么threaded handler情况也是这样吗？在\_\_handle_irq_event_percpu()函数中通过\_\_irq_wake_thread()函数唤醒irq_thread线程。\_\_irq_wake_thread()函数如下：

```cpp
1. void __irq_wake_thread(struct irq_desc *desc, struct irqaction *action)
2. {
3.     /*
4.      * In case the thread crashed and was killed we just pretend that
5.      * we handled the interrupt. The hardirq handler has disabled the
6.      * device interrupt, so no irq storm is lurking.
7.      */
8.     if (action->thread->flags & PF_EXITING)
9.         return;

11.     /*
12.      * Wake up the handler thread for this action. If the
13.      * RUNTHREAD bit is already set, nothing to do.
14.      */
15.     if (test_and_set_bit(IRQTF_RUNTHREAD, &action->thread_flags))
16.         return;

19.     desc->threads_oneshot |= action->thread_mask;

21.     atomic_inc(&desc->threads_active);

23.     wake_up_process(action->thread);
24. }
```

> 1) 如果irq的中断线程已经设置IRQTF_RUNTHREAD标志位，代表irq线程已经正在运行，因此无需重新唤醒，直接返回即可。\
> 2) 使用atomic_inc()增加threads_active计数，在synchronize_irq()函数中会判断threads_active计数是否为0来决定是否需要等待irq_thread执行完毕。

说了这些，不知道你是否知道什么是irq_thread呢？我们通过request_thread_irq()函数指定thread_fn，这个thread_fn就是irq_thread线程最终调用的函数。而每个irq都会创建一个irq线程，创建的过程在setup_irq_thread()函数进行，setup_irq_thread()函数代码如下：

```cpp
1. static int setup_irq_thread(struct irqaction *new, unsigned int irq, bool secondary)
2. {
3.     struct task_struct *t;
4.     struct sched_param param = {
5.         .sched_priority = MAX_USER_RT_PRIO/2,
6.     };

8.     if (!secondary) {
9.         t = kthread_create(irq_thread, new, "irq/%d-%s", irq, new->name);
10.     } else {
11.         t = kthread_create(irq_thread, new, "irq/%d-s-%s", irq, new->name);
12.         param.sched_priority -= 1;
13.     }

15.     if (IS_ERR(t))
16.         return PTR_ERR(t);

18.     sched_setscheduler_nocheck(t, SCHED_FIFO, &param);

20.     get_task_struct(t);
21.     new->thread = t;

23.     set_bit(IRQTF_AFFINITY, &new->thread_flags);
24.     return 0;
25. }
```

通过kthread_create()创建irq/irq-new->name的线程，该线程的入口函数值irq_thread。在irq_thread()中每执行完成一个thread_fn就会threads_active计数减1。\
现在可以考虑第三个问题了，假如register_thread_irq方式注册的threaded irq中调用msleep(1000)，睡眠1秒，请问系统此时会继续睡下去而没调度回来吗？因此导致msleep后续的操作没有执行。答案就是不会，因为suspend时候会等待threaded handler执行完毕，所以系统不会睡眠，放心好了。

2.8. 工作队列会睡眠吗？

现在来思考一个按键消抖问题。如果你还不知道什么是按键消抖的话，我……。按键消抖在内核中通常是这样处理，通过变压触发中断，在中断handler中通过queue delayed work一段时间，计时结束执行按键上报处理。从内核的gpio_keys抠出部分代码如下：

```cpp
1. static void gpio_keys_gpio_work_func(struct work_struct *work)
2. {
3.     struct gpio_button_data *bdata =
4.         container_of(work, struct gpio_button_data, work.work);

6.     gpio_keys_gpio_report_event(bdata);
7. }

9. static irqreturn_t gpio_keys_gpio_isr(int irq, void *dev_id)
10. {
11.     struct gpio_button_data *bdata = dev_id;

13.     mod_delayed_work(system_wq,
14.              &bdata->work,
15.              msecs_to_jiffies(bdata->software_debounce));

17.     return IRQ_HANDLED;
18. }
```

当按键按下，中断handler gpio_keys_gpio_isr执行，设定delayed work的定时器，等到定时器计时结束执行gpio_keys_gpio_work_func()，在gpio_keys_gpio_work_func()上报键值。你有考虑过一个问题吗？假如系统已经睡眠，此时第一次按下按键，是否有可能出现gpio_keys_gpio_work_func()函数没有执行，系统又继续睡眠，在第二次按键的时候执行第一次按键应该调用的gpio_keys_gpio_work_func()的情况吗？其实是有可能出现。只要bdata->software_debounce大于一定的时间就有可能出现。如果这个时间巧合，还有可能出现有时候正确上报，有时候没有上报。其实原因就是，内核的suspend只保证了IRQ handler的执行完成，并没有保证工作队列的执行完毕。

这里说的问题是work_queue没有机会调度，系统就休眠了。如果使用的不是delayed work，就是普通的work，只是在work中使用类似msleep的操作，系统是否也会继续睡眠呢？修改代码如下：

```cpp
1. static void gpio_keys_gpio_work_func(struct work_struct *work)
2. {
3.     struct gpio_button_data *bdata =
4.         container_of(work, struct gpio_button_data, work.work);

6. 	msleep(1000);
7.     gpio_keys_gpio_report_event(bdata);
8. }

10. static irqreturn_t gpio_keys_gpio_isr(int irq, void *dev_id)
11. {
12.     struct gpio_button_data *bdata = dev_id;

14.     schedule_work(&bdata->work);

16.     return IRQ_HANDLED;
17. }
```

这里的gpio_keys_gpio_work_func()中添加一句msleep(1000)会怎么样呢？由于此时使用的不是delayed work，因此一般不会出现没有调度work就睡眠的情况，与上面的情况还是有点区别的。但是这里其实也是有可能睡眠的，一旦msleep(1000)语句执行完毕，系统满足sleep条件，此时系统还是有可能睡眠导致后面的操作没有执行。在下次唤醒系统的时候才可能执行。所以这种情况下也是危险的。

结论就是：内核的suspend只保证了IRQ handler（hardirq handler or threaded handler）的执行完成，并没有保证工作队列的执行完毕。因此我们使用工作队列的话，必须要考虑这种情况的发生，并解决。

2.9. 如何解决工作队列睡眠问题？

系统suspend的过程中，主要是通过pm_wakeup_pending()判断suspend是否需要abort。如果你对我说的这一块不清楚，可以看看wowo其他几篇关于电源管理的文章。pm_wakeup_pending()主要是判断combined_event_count变量在suspend的过程中是否改变，如果改变suspend就应该abort。既然知道了原理，那么就好办了。在中断handler开始处增加combined_event_count计数，工作队列函数结尾位置减小combined_event_count计数即可。当然是不用你自己写代码，系统提供了接口函数pm_stay_awake()和pm_relax()。2.8节修改后的代码如下：

```cpp
1. static void gpio_keys_gpio_work_func(struct work_struct *work)
2. {
3.     struct gpio_button_data *bdata =
4.         container_of(work, struct gpio_button_data, work.work);

6.     gpio_keys_gpio_report_event(bdata);
7.     if (bdata->button->wakeup)
8.         pm_relax(bdata->input->dev.parent);
9. }

11. static irqreturn_t gpio_keys_gpio_isr(int irq, void *dev_id)
12. {
13.     struct gpio_button_data *bdata = dev_id;

15.     if (bdata->button->wakeup) {
16.         const struct gpio_keys_button *button = bdata->button;

18.         pm_stay_awake(bdata->input->dev.parent);
19.     }

21.     mod_delayed_work(system_wq,
22.              &bdata->work,
23.              msecs_to_jiffies(bdata->software_debounce));

25.     return IRQ_HANDLED;
26. }
```

好了，现在你放心好了，即使你是在gpio_keys_gpio_work_func()中msleep(2000)，系统也会等到pm_relax()执行之后系统才可能suspend。

3. 驱动工程师建议

看了这么多代码总是想说点东西。不管是建议还是什么。我由衷地希望驱动工程师可以写出完美没有bug并且简洁的代码。因此，这里有点小建议给驱动工程师（某些特性可能需要比较新的内核版本）。\
1) 如果设备具有唤醒系统的功能，请在probe函数中调用device_init_wakeup()和dev_pm_set_wake_irq()（注意调用顺序，先device_init_wakeup()再dev_pm_set_wake_irq()）。毕竟这样系统suspend的时候会自动帮助我们enable_irq_wake()和disable_irq_wake()，何乐而不为呢！简单就是美。如果你是i2c设备，那么可以更完美。连probe函数里面也可以不用调用了。只需要在设备的dts中添加wakeup-source属性即可。i2c core会自动帮我们完成这些操作。\
2) 如果你习惯在driver的suspend()中关闭中断，在resum()中打开中断，我觉你没必要这么做，何必要这些冗余代码呢！\
3) 既然dts现在这么流行了，你又何必不用呢！设备dts中的interrupts属性都会指明中断触发type，那你就用嘛！怎么获取这个flag呢？irqd_get_trigger_type()可以通过dts获取irq的触发type。所以request_threaded_irq()的第四个参数irqflags可以使用irqd_get_trigger_type()获得。如果你的内核版本更新的话，还可以更简单，irqflags传入0即可，在request_threaded_irq()中会自动帮我们调用irqd_get_trigger_type()获取。当然了，我也看聪明的IC厂家提供的driver，在dts中自定义一个属性表明irqflags，在driver中获取。我只能猜测driver的编写者不知道irqd_get_trigger_type()接口吧！\
4) 如果中断下半部使用工作队列，请成对使用pm_stay_awake()和pm_relax()。否则，谁也无法保证系统不会再一次的睡眠。

_原创文章，转发请注明出处。蜗窝科技，www.wowotech.net。_

______________________________________________________________________

标签: [电源管理](http://www.wowotech.net/tag/%E7%94%B5%E6%BA%90%E7%AE%A1%E7%90%86) [中断处理](http://www.wowotech.net/tag/%E4%B8%AD%E6%96%AD%E5%A4%84%E7%90%86) [中断子系统](http://www.wowotech.net/tag/%E4%B8%AD%E6%96%AD%E5%AD%90%E7%B3%BB%E7%BB%9F) [中断唤醒](http://www.wowotech.net/tag/%E4%B8%AD%E6%96%AD%E5%94%A4%E9%86%92)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [O(n)、O(1)和CFS调度器](http://www.wowotech.net/process_management/scheduler-history.html) | [USB-C(USB Type-C)规范的简单介绍和分析](http://www.wowotech.net/usb/usb_type_c_overview.html)»

**评论：**

**一只卤蛋**\
2023-04-11 17:39

写的真好哇，逻辑好清楚

[回复](http://www.wowotech.net/irq_subsystem/irq_handle_procedure.html#comment-8769)

**Jason**\
2021-12-21 14:00

请问前辈，一个中断配置可以唤醒系统，在系统 suspend 以后，唤醒需要触发30-60秒以后，系统才起来，这个问题如何debug？

[回复](http://www.wowotech.net/irq_subsystem/irq_handle_procedure.html#comment-8441)

**Wilson**\
2020-10-20 17:43

@smcdef, 请教大神文章开头说的具有唤醒功能的外设一般都有interrupt pin连接到soc的gpio pin，想问下这个连接跟中断request line是同一条吗？另外这个唤醒功能的line是直接由外设连接soc吗？是否通过interrupt controller作为媒介呢？求解答！

[回复](http://www.wowotech.net/irq_subsystem/irq_handle_procedure.html#comment-8128)

**hhb**\
2020-04-15 16:24

写的真好，谢谢

[回复](http://www.wowotech.net/irq_subsystem/irq_handle_procedure.html#comment-7949)

**qxrk**\
2019-12-10 15:39

请问下作者，我在我自己的SOC平台下将suspends_ops->enter()实现为进入WFI状态，等待一个中断唤醒，但是这个中断在进入WFI之前就已经产生了，产生的时间范围大概在suspends_ops->enter()至WFI状态之间，依然能够唤醒WFI，这是为什么，不应该是在WFI之后的中断才能唤醒吗？谢谢

[回复](http://www.wowotech.net/irq_subsystem/irq_handle_procedure.html#comment-7780)

**[smcdef](http://www.wowotech.net/)**\
2020-01-04 18:13

@qxrk：你所理解“在WFI之后的中断才能唤醒”应该是指唤醒CPU，也就是硬件角度的唤醒。但是对于suspend的流程来说，任何的中断都会阻止系统进入休眠。所以只要在suspend的过程中有中断产生就会唤醒系统。这里面的唤醒纯粹是软件生命的理解。

[回复](http://www.wowotech.net/irq_subsystem/irq_handle_procedure.html#comment-7815)

**wangyunqian**\
2019-10-21 22:29

不一定，看soc实现了。我们公司的cpu，resume是从bootrom开始执行的。

[回复](http://www.wowotech.net/irq_subsystem/irq_handle_procedure.html#comment-7710)

**morgan**\
2019-01-30 11:35

唤醒安卓上层应用，这个需要在framwork监听来自kernel的消息才能。中断触发之后，通过驱动函数内input子系统唤醒framework系统也行或者通过netlink去唤醒安卓应用。\
去看电源键的唤醒流程

[回复](http://www.wowotech.net/irq_subsystem/irq_handle_procedure.html#comment-7167)

**maiba**\
2018-06-07 16:09

中断handler执行完毕了，work queue 也正常运行了，cpu这些都正常了，app部分还是无法唤醒，怎么唤醒Android的上层应用呢？

[回复](http://www.wowotech.net/irq_subsystem/irq_handle_procedure.html#comment-6787)

**melo**\
2018-02-06 17:32

hi,对于文章结尾提到的\
"irqflags传入0即可，在request_threaded_irq()中会自动帮我们调用irqd_get_trigger_type()获取"\
这个用法很感兴趣，在网上没有看到过相关信息。。

我自己动手试了一下，一开始我直接写了个0,中断申请失败了，然后我去看了下irqd_get_trigger_type()这个函数，返回值和IRQF_TRIGGER_MASK相与了，因为我的驱动中断的顶半部传的NULL,这里我之前研究过是\_\_setup_irq里有一个判断是，如果顶半部是默认的primary_handler 同时没有IRQF_ONESHOT这个标识就会报错，申请失败。

然后我就传了一个IRQF_ONESHOT进去，正常申请中断，调试也正常\
我追代码没有成功理解整个过程。从结果上看起来是，传入的irqflags和IRQF_TRIGGER_MASK相与的值为0就会尝试调用irqd_get_trigger_type()?  可以提点一下这个过程吗，感激不尽

[回复](http://www.wowotech.net/irq_subsystem/irq_handle_procedure.html#comment-6534)

**[smcdef](http://www.wowotech.net/)**\
2018-02-08 09:39

@melo：首先你要确定，你的内核版本是否支持传递0，在request_threaded_irq()中是否有判断0时调用irqd_get_trigger_type()，因为不同的版本kernel这块差别挺大的。假如你的kernel不支持，那就自己调用request_threaded_irq()传递的flags通过irqd_get_trigger_type()获取，既然你的handler是NULL，那就要保证irqd_get_trigger_type()获取的flags包含IRQF_ONESHOT，你可以修改dts中interrupts属性的flags加上IRQF_ONESHOT，例如0x2008这个0x2000就是IRQF_ONESHOT。

[回复](http://www.wowotech.net/irq_subsystem/irq_handle_procedure.html#comment-6543)

**好运气**\
2018-02-05 16:01

请教大神：我在devicetree里面的gpio-keys节点中增加了wakeup-source属性，gpio-keys里面自动处理了suspend-resume的逻辑。现在发现按键无法唤醒系统，我想看是resume的时候哪个环节失败了。此时串口没有输出，请问有什么好的方法可以debug这种case吗？

[回复](http://www.wowotech.net/irq_subsystem/irq_handle_procedure.html#comment-6530)

**[smcdef](http://www.wowotech.net/)**\
2018-02-08 09:51

@好运气：1.首先会确定gpio-keys的驱动是否支持wakeup-source property，解析的时候是不是自动帮我们调用了device_init_wakeup()和dev_pm_set_wake_irq()。\
2.你可以先在不睡眠的情况下验证一下key是否正常工作。例如中断是否可以正常触发，可以看下proc/interrupts节点信息。\
3.如果不睡眠可以工作的话，我有个问题，什么是“唤醒系统”，你说的这个现象是什么？亮屏？准确的来说，key仅仅是唤醒了cpu，上报了事件，这个事件上层用来干什么是由上层决定的，例如亮屏。\
你所说的串口无法输出是指resume中有串口输出没有打印？所以你想排查resume环节吗？还是还说串口不能用？你的平台没有相关的log保存机制吗？可以把log导出来。如果没有的话，不知道有没有led灯，或许也可以帮助你debug一下进度。

[回复](http://www.wowotech.net/irq_subsystem/irq_handle_procedure.html#comment-6544)

1 [2](http://www.wowotech.net/irq_subsystem/irq_handle_procedure.html/comment-page-2#comments)

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

  - [Linux电源管理(14)\_从设备驱动的角度看电源管理](http://www.wowotech.net/pm_subsystem/device_driver_pm.html)
  - [X-017-KERNEL-串口驱动开发之uart driver框架](http://www.wowotech.net/x_project/serial_driver_porting_2.html)
  - [deadline调度器之（一）：原理](http://www.wowotech.net/process_management/deadline-scheduler-1.html)
  - [Deadline调度器之（二）：细节和使用方法](http://www.wowotech.net/process_management/dl-scheduler-2.html)
  - [Linux系统如何标识进程？](http://www.wowotech.net/process_management/pid.html)

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

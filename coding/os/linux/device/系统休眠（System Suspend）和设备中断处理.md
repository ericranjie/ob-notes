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

作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2017-4-21 12:02 分类：[电源管理子系统](http://www.wowotech.net/sort/pm_subsystem)

一、设备IRQ的suspend和resume

本小节主要解决这样一个问题：在系统休眠过程中，如何suspend设备中断（IRQ）？在从休眠中唤醒的过程中，如何resume设备IRQ？

一般而言，在系统suspend过程的后期，各个设备的IRQ (interrupt request line)会被disable掉。具体的时间点是在各个设备的late suspend阶段之后。代码如下（删除了部分无关代码）：

> static int suspend_enter(suspend_state_t state, bool \*wakeup)
>
> {……
>
> error = dpm_suspend_late(PMSG_SUSPEND);－－－－－late suspend阶段
>
> error = platform_suspend_prepare_late(state);
>
> 下面的代码中会disable各个设备的irq
>
> error = dpm_suspend_noirq(PMSG_SUSPEND);－－－－进入noirq的阶段
>
> error = platform_suspend_prepare_noirq(state);
>
> ……
>
> }

在dpm_suspend_noirq函数中，会针对系统中的每一个device，依次调用device_suspend_noirq来执行该设备noirq情况下的suspend callback函数，当然，在此之前会调用suspend_device_irqs函数来disable所有设备的irq。

之所以这么做，其思路是这样的：在各个设备驱动完成了late suspend之后，按理说这些已经被suspend的设备不应该再触发中断了。如果还有一些设备没有被正确的suspend，那么我们最好的策略是mask该设备的irq，从而阻止中断的递交。此外，在过去的代码中（指interrupt handler），我们对设备共享IRQ的情况处理的不是很好，存在这样的问题：在共享IRQ的设备们完成suspend之后，如果有中断触发，这时候设备驱动的interrupt handler并没有准备好。在有些场景下，interrupt handler会访问已经suspend设备的IO地址空间，从而导致不可预知的issue。这些issue很难debug，因此，我们引入了suspend_device_irqs()以及设备noirq阶段的callback函数。

系统resume过程中，在各个设备的early resume过程之前，各个设备的IRQ会被重新打开，具体代码如下（删除了部分无关代码）：

> static int suspend_enter(suspend_state_t state, bool \*wakeup)
>
> {……
>
> platform_resume_noirq(state);－－－－首先执行noirq阶段的resume
>
> dpm_resume_noirq(PMSG_RESUME);－－－－－－在这里会恢复irq，然后进入early resume阶段
>
> platform_resume_early(state);
>
> dpm_resume_early(PMSG_RESUME);
>
> ……}

在dpm_resume_noirq函数中，会调用各个设备驱动的noirq callback，在此之后，调用resume_device_irqs函数，完成各个设备irq的enable。

二、关于IRQF_NO_SUSPEND Flag

当然，有些中断需要在整个系统的suspend-resume过程中（包括在noirq阶段，包括将nonboot CPU推送到offline状态以及系统resume后，将其重新设置为online的阶段）保持能够触发的状态。一个简单的例子就是timer中断，此外IPI以及一些特殊目的设备中断也需要如此。

在中断申请的时候，IRQF_NO_SUSPEND flag可以用来告知IRQ subsystem，这个中断就是上一段文字中描述的那种中断：需要在系统的suspend-resume过程中保持enable状态。有了这个flag，suspend_device_irqs并不会disable该IRQ，从而让该中断在随后的suspend和resume过程中，保持中断开启。当然，这并不能保证该中断可以将系统唤醒。如果想要达到唤醒的目的，请调用enable_irq_wake。

需要注意的是：IRQF_NO_SUSPEND flag影响使用该IRQ的所有外设（一个IRQ可以被多个外设共享，不过ARM中不会这么用）。如果一个IRQ被多个外设共享，并且各个外设都注册了对应的interrupt handler，如果其一在申请中断的时候使用了IRQF_NO_SUSPEND flag，那么在系统suspend的时候（指suspend_device_irqs之后，按理说各个IRQ已经被disable了），所有该IRQ上的各个设备的interrupt handler都可以被正常的被触发执行，即便是有些设备在调用request_irq(或者其他中断注册函数)的时候没有设定IRQF_NO_SUSPEND flag。正因为如此，我们应该尽可能的避免同时使用IRQF_NO_SUSPEND 和IRQF_SHARED这两个flag。

三、系统中断唤醒接口：enable_irq_wake() 和 disable_irq_wake()

有些中断可以将系统从睡眠状态中唤醒，我们称之“可以唤醒系统的中断”，当然，“可以唤醒系统的中断”需要配置才能启动唤醒系统这样的功能。这样的中断一般在工作状态的时候就是作为普通I/O interrupt出现，只要在准备使能唤醒系统功能的时候，才会发起一些特别的配置和设定。

这样的配置和设定有可能是和硬件系统（例如SOC）上的信号处理逻辑相关的，我们可以考虑下面的HW block图：

[![irq-suspend](http://www.wowotech.net/content/uploadfile/201704/d77ad10fce03b9d181a2be9351c6113420170421040223.gif "irq-suspend")](http://www.wowotech.net/content/uploadfile/201704/6a651c54519aab85c314f0af79b75e3820170421040222.gif)

外设的中断信号被送到“通用的中断信号处理模块”和“特定中断信号接收模块”。正常工作的时候，我们会turn on“通用的中断信号处理模块”的处理逻辑，而turn off“特定中断信号接收模块” 的处理逻辑。但是，在系统进入睡眠状态的时候，有可能“通用的中断信号处理模块”已经off了，这时候，我们需要启动“特定中断信号接收模块”来接收中断信号，从而让系统suspend-resume模块（它往往是suspend状态时候唯一能够工作的HW block了）可以正常的被该中断信号唤醒。一旦唤醒，我们最好是turn off“特定中断信号接收模块”，让外设的中断处理回到正常的工作模式，同时，也避免了系统suspend-resume模块收到不必要的干扰。

IRQ子系统提供了两个接口函数来完成这个功能：enable_irq_wake()函数用来打开该外设中断线通往系统电源管理模块（也就是上面的suspend-resume模块）之路，另外一个接口是disable_irq_wake()，用来关闭该外设中断线通往系统电源管理模块路径上的各种HW block。

调用了enable_irq_wake会影响系统suspend过程中的suspend_device_irqs处理，代码如下：

> static bool suspend_device_irq(struct irq_desc \*desc)
>
> {
>
> ……
>
> if (irqd_is_wakeup_set(&desc->irq_data)) {
>
> irqd_set(&desc->irq_data, IRQD_WAKEUP_ARMED);
>
> return true;
>
> }
>
> 省略Disable 中断的代码
>
> }

也就是说，一旦调用enable_irq_wake设定了该设备的中断作为系统suspend的唤醒源，那么在该外设的中断不会被disable，只是被标记一个IRQD_WAKEUP_ARMED的标记。对于那些不是wakeup source的中断，在suspend_device_irq 函数中会标记IRQS_SUSPENDED并disable该设备的irq。在系统唤醒过程中（resume_device_irqs），被diable的中断会重新enable。

当然，如果在suspend的过程中发生了某些事件（例如wakeup source产生了有效信号），从而导致本次suspend abort，那么这个abort事件也会通知到PM core模块。事件并不需要被立刻通知到PM core模块，一般而言，suspend thread会在某些点上去检查pending的wakeup event。

在系统suspend的过程中，每一个来自wakeup source的中断都会终止suspend过程或者将系统唤醒（如果系统已经进入suspend状态）。但是，在执行了suspend_device_irqs之后，普通的中断被屏蔽了，这时候，即便HW触发了中断信号也无法执行其interrupt handler。作为wakeup source的IRQ会怎样呢？虽然它的中断没有被mask掉，但是其interrupt handler也不会执行（这时候的HW Signal只是用来唤醒系统）。唯一有机会执行的interrupt handler是那些标记IRQF_NO_SUSPEND flag的IRQ，因为它们的中断始终是enable的。当然，这些中断不应该调用enable_irq_wake进行唤醒源的设定。

四、Interrupts and Suspend-to-Idle

Suspend-to-idle (也被称为"freeze" 状态)是一个相对比较新的系统电源管理状态，相关代码如下：

> static int suspend_enter(suspend_state_t state, bool \*wakeup)
>
> {
>
> ……
>
> 各个设备的late suspend阶段
>
> 各个设备的noirq suspend阶段
>
> if (state == PM_SUSPEND_FREEZE) {
>
> freeze_enter();
>
> goto Platform_wake;
>
> }
>
> ……
>
> }

Freeze和suspend的前面的操作基本是一样的：首先冻结系统中的进程，然后是suspend系统中的形形色色的device，不一样的地方在noirq suspend完成之后，freeze不会disable那些non-BSP的处理器和syscore suspend阶段，而是调用freeze_enter函数，把所有的处理器推送到idle状态。这时候，任何的enable的中断都可以将系统唤醒。而这也就意味着那些标记IRQF_NO_SUSPEND（其IRQ没有在suspend_device_irqs过程中被mask掉）是有能力将处理器从idle状态中唤醒（不过，需要注意的是：这种信号并不会触发一个系统唤醒信号），而普通中断由于其IRQ被disable了，因此无法唤醒idle状态中的处理器。

那些能够唤醒系统的wakeup interrupt呢？由于其中断没有被mask掉，因此也可以将系统从suspend-to-idle状态中唤醒。整个过程和将系统从suspend状态中唤醒一样，唯一不同的是：将系统从freeze状态唤醒走的中断处理路径，而将系统从suspend状态唤醒走的唤醒处理路径，需要电源管理HW BLOCK中特别的中断处理逻辑的参与。

五、IRQF_NO_SUSPEND 标志和enable_irq_wake函数不能同时使用

针对一个设备，在申请中断的时候使用IRQF_NO_SUSPEND flag，又同时调用enable_irq_wake设定唤醒源是不合理的，主要原因如下：

1、如果IRQ没有共享，使用IRQF_NO_SUSPEND flag说明你想要在整个系统的suspend-resume过程中（包括suspend_device_irqs之后的阶段）保持中断打开以便正常的调用其interrupt handler。而调用enable_irq_wake函数则说明你想要将该设备的irq信号设定为中断源，因此并不期望调用其interrupt handler。而这两个需求明显是互斥的。

2、IRQF_NO_SUSPEND 标志和enable_irq_wake函数都不是针对一个interrupt handler的，而是针对该IRQ上的所有注册的handler的。在一个IRQ上共享唤醒源以及no suspend中断源是比较荒谬的。

不过，在非常特殊的场合下，一个IRQ可以被设定为wakeup source，同时也设定IRQF_NO_SUSPEND 标志。为了代码逻辑正确，该设备的驱动代码需要满足一些特别的需求。

参考文献

1、内核文档power/suspend-and-interrupts.txt

_原创文章，转发请注明出处。蜗窝科技_

标签: [suspend](http://www.wowotech.net/tag/suspend) [irq](http://www.wowotech.net/tag/irq)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Linux DMA Engine framework(2)\_功能介绍及解接口分析](http://www.wowotech.net/linux_kenrel/dma_engine_api.html) | [Linux DMA Engine framework(1)\_概述](http://www.wowotech.net/linux_kenrel/dma_engine_overview.html)»

**评论：**

**hly**\
2020-01-14 10:34

@wowo\
不过，在非常特殊的场合下，一个IRQ可以被设定为wakeup source-----怎么设置为wakeup source.

[回复](http://www.wowotech.net/pm_subsystem/suspend-irq.html#comment-7837)

**hly**\
2020-01-14 10:31

@wowo\
我现在遇到一个问题，在休眠的过程中产生了一个中断，然后中断的下半部分去操作i2c，此时i2c已经休眠了，所以一直在报错。这个中断我已经设置为enable_irq_wake了，为什么在休眠流程过程中产生中断不会去唤醒系统呢，而且直接走CPU kill 流程。只中断只能完全进去suspend 后才能wakeup 系统。\
怎么样让中断持有wakeup source

[回复](http://www.wowotech.net/pm_subsystem/suspend-irq.html#comment-7836)

**Definitely**\
2020-03-02 09:14

@hly：i2c一般会通过2.2k上拉接到一个单独的regulator(和touch IC的regulator不同)，你在DTS和driver里需要单独配置这个regulator，确保系统休眠时，touch使用的i2c还有上拉，如果你的driver里只配置了touch IC的regulator，就会导致这个问题

[回复](http://www.wowotech.net/pm_subsystem/suspend-irq.html#comment-7903)

**crash**\
2018-11-16 14:19

一直觉得很费解，假如suspend的过程一切顺利，进入到suspend_ops->enter，休眠一段时间后，外部中断唤醒产生wakeup event,这时候程序是从哪里开始跑起来的呢，继续events_check_enabled = false;这样往下跑吗\
{\
\*wakeup = pm_wakeup_pending();\
if (!(suspend_test(TEST_CORE) || \*wakeup)) {\
error = suspend_ops->enter(state);\
events_check_enabled = false;\
}\
syscore_resume();\
}

[回复](http://www.wowotech.net/pm_subsystem/suspend-irq.html#comment-7039)

**[bsp](http://www.wowotech.net/)**\
2020-06-04 09:47

@crash：分享下通过psci处理system_suspend, suspend_ops->enter的流程：

1， psci初始化：suspend_set_ops(&psci_suspend_ops);\
static const struct platform_suspend_ops psci_suspend_ops = {\
.enter          = psci_system_suspend_enter,\
};

2， suspend_ops->enter，即    \
static int psci_system_suspend_enter(suspend_state_t state)\
{\
return cpu_suspend(0, psci_system_suspend);\
}\
\
arch/arm64/kernel/suspend.c\
int cpu_suspend(unsigned long arg, int (\*fn)(unsigned long))\
{\
struct sleep_stack_data state;\
\
这里类似fork，有返回两次的效果：\
if (\_\_cpu_suspend_enter(&state)) {\
//第一次返回1， 往下走\
ret = invoke_psci_fn(PSCI_FN_NATIVE(1_0, SYSTEM_SUSPEND),\
\_\_pa_symbol(cpu_resume), 0, 0);;\
} else {\
//第二次由于cpu_resume返回值为0，往这走：\
\_\_cpu_suspend_exit();\
}\
\
2.1，这里通过psci组装smc指令，跳转到TZ执行tzbsp_psci_cpu_suspend_wrapper；\
TZ会和RPM通信配置lpm mode,比如power down l3/ power down cpu-core；\
PSCI(Power State Coordination Interface), 是arm定义的一套处理cpu on/off/suspend等的协议，\
比如Linux-kernel发送相应的cpu operation到和Trust-Zone（EL3 by smc）或Hypervirsor(EL2 by hvc)，再由TZ或Hyp做对应的操作：\
比如： Linux-kernel 要offline、suspend一个cpu，Hypervisor只是把这个cpu给其它的OS使用。\
\
2.2 cpu_resume是物理地址；\
唤醒cpu重新power-up时，cpu 会call cpu_resume；\
因为此时mmu还未打开，所以2.2必须传入cpu_resume的物理地址；\
cpu_resume中会打开mmu；\
\
2.3 \_\_cpu_suspend_enter两次返回的效果：\
\_\_cpu_suspend_enter做一些准备工作后返回 1，then call smc into TZ and suspend system/cpu;\
重新唤醒cpu时，cpu call cpu_resume() and return 0 to the next instruction of \_\_cpu_suspend_enter;\
因为是0，所以会call \_\_cpu_suspend_exit。\
\
3， arch/arm64/kernel/sleep.S\
ENTRY(\_\_cpu_suspend_enter)\
//第一步保存当前lr到state中\
stp     x29, lr, \[x0, #SLEEP_STACK_DATA_CALLEE_REGS\]\
...\
mov     x0, #1        //这里返回1\
ret\
ENDPROC(\_\_cpu_suspend_enter)

ENTRY(cpu_resume)\
bl      el2_setup               // if in EL2 drop to EL1 cleanly\
bl      \_\_cpu_setup\
/\* enable the MMU early - so we can access sleep_save_stash by va \*/\
bl      \_\_enable_mmu\
ldr     x8, =\_cpu_resume\
br      x8\
ENDPROC(cpu_resume)\
\
ENTRY(\_cpu_resume)\
...\
ldp     x29, lr, \[x29\]    //从之前的state中恢复lr\
mov     x0, #0            //返回值为0\
ret                        //return到\_\_cpu_suspend_enter下一行\
ENDPROC(\_cpu_resume)

4,  for cpu suspend, intead of system suspend:\
static int psci_suspend_finisher(u32 state, unsigned long entry_point)\
{\
int err;\
u32 fn;

fn = psci_function_id\[PSCI_FN_CPU_SUSPEND\];\
err = invoke_psci_fn(fn, state, \_\_pa_symbol(cpu_resume), 0);\
return psci_to_linux_errno(err);\
}

cpu_suspend(state_id, psci_suspend_finisher);

[回复](http://www.wowotech.net/pm_subsystem/suspend-irq.html#comment-8012)

**[bsp](http://www.wowotech.net/)**\
2022-01-07 10:14

@crash：是的，这样往下跑。但中间会经历很多操作，比如通过psci处理suspend_ops->enter时，要经过和TZ的通信；唤醒第一步是在TZ里处理的，然后再到EL1的linux-kernel中的cpu_resume。。

[回复](http://www.wowotech.net/pm_subsystem/suspend-irq.html#comment-8476)

**linux菜鸟**\
2018-04-29 16:08

@wowo：Hi, 作为wakeup source的IRQ会怎样呢？虽然它的中断没有被mask掉，但是其interrupt handler也不会执行（这时候的HW Signal只是用来唤醒系统）。\
这句话的意思是,即使系统唤醒后，该IRQ对应的中断处理函数也不会执行是吗？并不是延迟执行，而是根本不会执行，是这样理解的吗

[回复](http://www.wowotech.net/pm_subsystem/suspend-irq.html#comment-6718)

**[wowo](http://www.wowotech.net/)**\
2018-04-29 22:07

@linux菜鸟：文章中你引用的话后面还有这样的表述：\
唯一有机会执行的interrupt handler是那些标记IRQF_NO_SUSPEND flag的IRQ，因为它们的中断始终是enable的。\
所以如果不标记IRQF_NO_SUSPEND flag的IRQ，是不会执行的。

[回复](http://www.wowotech.net/pm_subsystem/suspend-irq.html#comment-6720)

**orange**\
2019-04-18 16:32

@wowo：@wowo\
根据上面的描述，意思时wakeup source的IRQ，如果其注册中断Handler时带了IRQF_NO_SUSPEND标志，这个IRQ的Handler是会执行的，是这个意思吗？

[回复](http://www.wowotech.net/pm_subsystem/suspend-irq.html#comment-7361)

**wall.e**\
2019-11-19 16:07

@orange：我也好奇，感觉IRQF_NO_SUSPEND 标志和enable_irq_wake没有看明白，大神能不能再解释一下

[回复](http://www.wowotech.net/pm_subsystem/suspend-irq.html#comment-7758)

**Rafe**\
2017-09-11 09:41

@wowo\
@linuxer\
请教一个问题：\
当有中断唤醒系统的时候，我们要把该event上报到App层(通过uevent)，那么这个时候如何保证App层能收到该事件？\
resume的过程，应该是kernel先resume，然后App才resume，那么上报event的时候，App岂不是存在没有ready的情况？

[回复](http://www.wowotech.net/pm_subsystem/suspend-irq.html#comment-6015)

**Rafe**\
2017-09-11 10:06

@Rafe：@wowo\
@linuxer\
我想了一下，这是不是跟uevent的上报机制有关？这样问吧，当我们用uevent上报数据的时候，如果用户空间还没有resume，那么我们上报的数据这个时候去哪里了？是阻塞在那里或者说存储在了什么地方，等待用户空间来读取吗？

[回复](http://www.wowotech.net/pm_subsystem/suspend-irq.html#comment-6017)

**[wowo](http://www.wowotech.net/)**\
2017-09-11 11:10

@Rafe：其实这是一个应用层面的事情，和kernel的机制已经无关了：\
上报event可以唤醒系统，也可以阻止系统休眠；\
如果这个event有timeout，则系统会在timeout之后重新休眠，除非其它模块重新报一个event（例如拿一把锁）；\
如果这个event没有timeout，则会一直阻止系统休眠，直到有人把这个event清除（例如driver把数据放在某一个位置，并上报event，app读取数据的时候，顺便清除event。

如果理解了原理，我们有N多种方法保证你所提到的这种同步。当然，如果什么都不考虑、不去做，系统出问题是很正常的。

[回复](http://www.wowotech.net/pm_subsystem/suspend-irq.html#comment-6018)

**Rafe**\
2017-09-11 12:07

@wowo：Hi wowo,\
非常感谢回复。\
是这样的，目前用户空间和kernel空间都没有timeout机制，suspend的进入完全由用户空间根据收到kernel的消息来控制。例如kernel发出一个1，用户空间判断是1才会让系统进入suspend.也就是说发送的event并没有在kernel阻止休眠或是重新进入休眠的状态,它只是发送消息，让用户空间去处理。

而我现在困惑的点是，kernel被唤醒了（例如是被USB唤醒的吧），这个时候，用户空间要根据USB的状态来做一些处理，那么USB driver就通过uevent发送了状态改变的消息，但是这个时候用户空间还没有resume完全，接收USB driver发送消息的进程还没resume,那么这个时候USB driver发送的消息是否就丢失了？用户空间也就接收不到这个消息了？

另外我们的uevent发送机制，会将数据存放起来，直到有人去获取？

[回复](http://www.wowotech.net/pm_subsystem/suspend-irq.html#comment-6019)

**[wowo](http://www.wowotech.net/)**\
2017-09-12 13:20

@Rafe：uevent发送的是异步消息，会不会丢，取决于实现方法，可参考下面的文章：\
http://www.wowotech.net/linux_kenrel/uevent.html\
如果是kmod的方式，会调用用户空间程序，接下来用户空间程序要保证后面的事情；\
如果是netlink，取决于msg buffer的大小。

[回复](http://www.wowotech.net/pm_subsystem/suspend-irq.html#comment-6028)

**michael**\
2017-08-25 17:10

@linuxer\
1）IRQF_NO_SUSPEND保证irq处于enable状态，相应的handler可以被执行，不明白的是，既然系统已经suspend，如何处理其handler，难道是系统被唤醒了？\
2）如果IRQF_NO_SUSPEND可以唤醒系统，函数enable_irq_wake也可以唤醒系统，岂不是重复了？

[回复](http://www.wowotech.net/pm_subsystem/suspend-irq.html#comment-5952)

**[linuxer](http://www.wowotech.net/)**\
2017-08-25 19:06

@michael：第一个问题：suspend是一个漫长的过程，在这个过程中，如果有中断发生，那么handler都会执行，从而让整个suspend过程回滚。一旦进入了suspend状态，那么只有几个设定的唤醒源可以把系统唤醒。\
第二个问题：IRQF_NO_SUSPEND并不能唤醒系统，只是保证在suspend的过程中，中断是开启的。函数enable_irq_wake才是设定唤醒源的接口。

[回复](http://www.wowotech.net/pm_subsystem/suspend-irq.html#comment-5953)

**leo**\
2018-10-14 16:16

@linuxer：你好，我也有相同的疑问，既然IRQF_NO_SUSPEND标志并不能唤醒系统，只是保证suspend的过程中该中断enable，并且可以执行handler。但是我的理解是系统都无法唤醒，那么内核的调度应该也没有起来，此时进程都冻结了，谁来执行这个handler呢？\
还有假设这个handler被执行了，系统没起来，也做不了什么事情啊。\
对这一块很模糊，还请linuxer指点下

[回复](http://www.wowotech.net/pm_subsystem/suspend-irq.html#comment-6981)

**yinjian**\
2017-08-21 14:19

Hi,wowo and linuxer\
有个问题请教下，在disable irq之前已经执行了suspend device，那如果在suspend device之后来了中断，此时中断处理程序还可以正常执行吗？

[回复](http://www.wowotech.net/pm_subsystem/suspend-irq.html#comment-5934)

**[wowo](http://www.wowotech.net/)**\
2017-08-21 14:41

@yinjian：device被suspend之后，应该保证不能再产生中断了，如果不能保证，当然，相应的中断处理程序还是会执行。

[回复](http://www.wowotech.net/pm_subsystem/suspend-irq.html#comment-5936)

**melo**\
2017-08-18 17:02

@wowo\
@linuxer\
有个问题请教下博主,最近在接触高通的板子,对于gpio而言,所有的gpio都能产生中断,但是只有MPM interrupt才能wakeup

现在的情况是,spec里msm8909这颗U支持的最大MPM 数目是64个\
但是硬件给到我们能用的是42个\
这42个所谓的能用的 我的理解是还需要设备树配置加enable_irq_wake申请

也就是硬件支持 加 设备树配置 加 驱动申请么(这里其实我有疑问,因为我半路出家所有对硬件有点茫然,不知道我这里理解的硬件支持和设备树配置是不是同一件事)

[回复](http://www.wowotech.net/pm_subsystem/suspend-irq.html#comment-5923)

**melo**\
2017-08-18 17:03

@melo：不对   HW给我们能用的在先  配置设备树在后 应该没关系

[回复](http://www.wowotech.net/pm_subsystem/suspend-irq.html#comment-5924)

**melo**\
2017-08-18 17:39

@melo：..我蠢了  其实我一开始就是没找到对应的寄存器,后来发现msm的spec里面已经把TLMM_MPM_WAKEUP_INT 都给出来了

[回复](http://www.wowotech.net/pm_subsystem/suspend-irq.html#comment-5925)

**[wowo](http://www.wowotech.net/)**\
2017-08-21 08:50

@melo：哈哈，找出来了就行，多折腾一下，总能找到答案的，加油！～

[回复](http://www.wowotech.net/pm_subsystem/suspend-irq.html#comment-5930)

**hw_boboy**\
2017-08-08 11:51

Hi,\
我们的设备在启动的时候，有时候中间会停个30s左右再继续启动。（已经过了uboot阶段）\
这个情况不是每次都会这样，能帮忙猜测一下是什么原因？或者有什么好的办法可以debug?\
谢谢！

[回复](http://www.wowotech.net/pm_subsystem/suspend-irq.html#comment-5889)

**[wowo](http://www.wowotech.net/)**\
2017-08-08 14:04

@hw_boboy：把打印都打开，加上时间戳，看看那部分代码耗时较久。\
如果还找不到，就手动的添加打印，知道找到为止。

[回复](http://www.wowotech.net/pm_subsystem/suspend-irq.html#comment-5893)

**hw_boboy**\
2017-08-10 10:30

@wowo：Hi,\
通过log，发现30秒delay是在/kernel/init/main.c文件中的static int \_\_ref kernel_init(void \*unused)函数中，为方便查阅代码部分代码如下。最值得怀疑的\
有两个地方：

1. 938行的free_initmem()->free_reserved_area()->free_reserved_page(),这里面如果有错误是否会导致sleep.
1. 946行的ret = run_init_process(ramdisk_execute_command)，当满足(!strcmp(basename(argv\[0\]), "watchdogd"))时，会watchdogd_main(argc, argv);\
   因为现在很难复现，所以帮忙猜一下。谢谢！

static int \_\_ref kernel_init(void *unused)\
932{\
933    int ret;\
934\
935    kernel_init_freeable();\
936    /* need to finish all async \_\_init code before freeing the memory \*/\
937    async_synchronize_full();\
938    free_initmem();\
939    mark_rodata_ro();\
940    system_state = SYSTEM_RUNNING;\
941    numa_default_policy();\
942\
943    flush_delayed_fput();\
944\
945    if (ramdisk_execute_command) {\
946        ret = run_init_process(ramdisk_execute_command);\
947        if (!ret)\
948            return 0;\
949        pr_err("Failed to execute %s (error %d)\\n",\
950               ramdisk_execute_command, ret);\
951    }

[回复](http://www.wowotech.net/pm_subsystem/suspend-irq.html#comment-5903)

**Ekko**\
2017-07-17 10:44

enable_irq_wake接口需要平台支持吗？

[回复](http://www.wowotech.net/pm_subsystem/suspend-irq.html#comment-5816)

1 [2](http://www.wowotech.net/pm_subsystem/suspend-irq.html/comment-page-2#comments)

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

  - [Linux common clock framework(2)\_clock provider](http://www.wowotech.net/pm_subsystem/clock_provider.html)
  - [Debian下的WiFi实验（二）：无线网卡自动连接AP](http://www.wowotech.net/linux_application/wifi-test-2.html)
  - [eMMC 原理 1 ：Flash Memory 简介](http://www.wowotech.net/basic_tech/flash_memory_intro.html)
  - [slub分配器](http://www.wowotech.net/memory_management/247.html)
  - [显示技术介绍(1)\_概述](http://www.wowotech.net/display/display_tech_overview.html)

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

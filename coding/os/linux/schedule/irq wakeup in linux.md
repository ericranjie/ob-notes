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

作者：[schspa](http://www.wowotech.net/author/542) 发布于：2021-11-15 18:41 分类：[电源管理子系统](http://www.wowotech.net/sort/pm_subsystem)

## Table of Contents

- [术语](http://www.wowotech.net/admin/#org5c40aed)
- [硬件架构](http://www.wowotech.net/admin/#orgd7b01ed)
    - [gic对唤醒功能的支持](http://www.wowotech.net/admin/#org7c77031)
    - [GIC power down ？](http://www.wowotech.net/admin/#orgc654f8d)
    - [CPU Power Down](http://www.wowotech.net/admin/#orgdf72b0e)
    - [CPU Power Up](http://www.wowotech.net/admin/#orgd3f14d0)
        - [RESET的执行地址](http://www.wowotech.net/admin/#org76e8360)
- [Suspend In Linux](http://www.wowotech.net/admin/#org786b4f8)
- [Linux irq wakeup](http://www.wowotech.net/admin/#orgc55abf5)
    - [irq flags](http://www.wowotech.net/admin/#orgc3391b3)
        - [IRQCHIP_MASK_ON_SUSPEND](http://www.wowotech.net/admin/#orgbabe822)
        - [IRQCHIP_SKIP_SET_WAKE](http://www.wowotech.net/admin/#org7c947f0)
    - [arm-gic](http://www.wowotech.net/admin/#org6d4deb1)
    - [linux内核接口](http://www.wowotech.net/admin/#orgc146640)
    - [implement stacked irqchip](http://www.wowotech.net/admin/#org2113d60)
        - [used function & macors](http://www.wowotech.net/admin/#orge144b9c)
        - [irq domain](http://www.wowotech.net/admin/#org34598ff)
        - [irq initialization](http://www.wowotech.net/admin/#orge030d08)
        - [irq domain ops](http://www.wowotech.net/admin/#orgb944abf)
- [ATF软件](http://www.wowotech.net/admin/#org036e12d)
    - [参数：](http://www.wowotech.net/admin/#org1958417)
    - [平台相关实现](http://www.wowotech.net/admin/#org9d23227)
- [refs](http://www.wowotech.net/admin/#org0417c6b)

## 术语

|   |   |
|---|---|
|简写|全称|
|TRM|Technical Reference Manual|
|GIC|Generic Interrupt Controller|
|smccc|SMC CALLING CONVENTION|

## 硬件架构

在本篇文章中基于AARCH64平台，GIC作为中断控制器来进行讨论  
下面是GIC-600的系统框架图：

![2021-03-05_12-47-10_screenshot.png](https://schspa.tk/2020/07/10/assets/2021-03-05_12-47-10_screenshot.png)  
看图可知，Wake Request模块就是为了唤醒功能提供的信号输出。

### gic对唤醒功能的支持

在[gic600的TRM手册](https://developer.arm.com/documentation/101206/0000/signal-descriptions/power-control-signals)中，可以看到gic600有wake_request的输出信号，Power controller可以根据这个信号来将对应的cpu唤醒。  
唤醒信号：

|   |   |   |   |
|---|---|---|---|
|Signal|Type|Source or destination|Description|
|wake_request[]|Output|Power controller|Wake request signal to power controller indicating that an interrupt is targeting      <br>this core and it must be woken. When asserted, the wake_request is sticky unless the    <br>Distributor is put into the gated state.|

### GIC power down ？

GIC可以被关闭电源，也可以不关闭电源，这个在SOC设计阶段就可以选择。  
对于Arm gic来言，一般具有两种情况

1. GIC处于always on的power domain之中，这样gic在休眠时由于没有断电，仍旧可以正常工作，任何的中断都可以正常唤醒soc [1](http://www.wowotech.net/admin/#fn.1), 如图中的Wakeup Request的信号  
    
2. GIC在可以断电的power domain中，这样在休眠时一般都会对gic进行断电，这种状况下，系统需要soc特定的实现来进行唤醒。  
    具体实现各家SOC都不相同，在此不做讨论，只需要知道这个唤醒功能和GIC没有关系，完全由SOC厂商在设计时实现。  
    

### CPU Power Down

下面是ARM Crotext A55 CPU的下电流程，CPU有寄存器CPUPWRCTLR来控制电源控制相关功能，当CPUPWRCTLR.CORE_PWRDN_EN置1时，就是代表CPU在下次执行WFI指令时要进入掉电状态。[2](http://www.wowotech.net/admin/#fn.2)

> The Cortex-A55 core uses the following power down sequence.
> 
> To power down a core, perform the following programming sequence:
> 
> 1. Save all architectural state.  
>     
> 2. Configure the GIC distributor to disable or reroute interrupts away from this core.  
>     
> 3. Set the CPUPWRCTLR.CORE_PWRDN_EN bit to 1 to indicate to the power controller that a powerdown is requested.  
>     
> 4. Execute an Instruction Synchronization Barrier (ISB) instruction.  
>     
> 5. Execute a WFI instruction.  
>     
> 
> After executing WFI and then receiving a powerdown request from the power controller, the hardware performs the following:
> 
> • Disabling and flushing of caches (L1 and L2).
> 
> • Removal of the core from coherency.

从上面的掉电流程可知，A55在下电时，还需要SOC内部power controller的支持，不同的SOC厂商实现的方式千奇百怪，并且设计泄密在此不做深入。

### CPU Power Up

从上边Power Down的流程可知，系统在进入Power Down之后需要进行reset才可以将CPU重新启动，具体的过程也是SOC厂商自己定义的行为，没有太大分析的必要。  
Reset之后，CPU需要从RVBAR寄存器所显示的地址来重新启动。RVBAR寄存器是通过CPU的外部信号线进行输入的。

#### RESET的执行地址

![2021-03-05_13-40-42_screenshot.png](https://schspa.tk/2020/07/10/assets/2021-03-05_13-40-42_screenshot.png)

由于没有在A55的TRM中找到关于此信号的定义，拿A53的来充个数。

![2021-03-05_13-43-32_screenshot.png](https://schspa.tk/2020/07/10/assets/2021-03-05_13-43-32_screenshot.png)

从手册可知，复位执行地址是可以通过配置来修改的，并且4byte对齐(因为bit0,1不可配置)

## Suspend In Linux

下图是Linux系统中suspend以及resume的流程图，图中以PSCI为例，描写了系统在休眠时的调用流程。

![suspend-to-ram.png](https://schspa.tk/2020/07/10/assets/suspend-to-ram.png)  
上面的流程着重描写了系统通过PSCI休眠时的调用流程，包括Linux系统从休眠状态唤醒之后的resume流程。  
Linux唤醒时，运行的Linux内核的第一段指令是cpu_resume，之后再经过arm64 psci相关的一系列调用返回到psci_system_suspend_enter继续执行。

## Linux irq wakeup

在Linux系统中，如果想要唤醒已经进入休眠状态中的cpu，就需要通过~enable_irq_wake~来设置开启唤醒功能[3](http://www.wowotech.net/admin/#fn.3)。  
实际上，芯片支持的唤醒源个数是有限的，并不是每种中断都可以正常的唤醒系统。

### irq flags

#### IRQCHIP_MASK_ON_SUSPEND

Mask non wake irqs in the suspend path

#### IRQCHIP_SKIP_SET_WAKE

Skip chip.irq_set_wake(), for this irq chip  
由以下patch引入，提交说明已经很明确

60f96b41f71d2a13d1c0a457b8b77958f77142d1
Author:     Santosh Shilimkar <santosh.shilimkar@ti.com>
AuthorDate: Fri Sep 9 13:59:35 2011 +0530
Commit:     Thomas Gleixner <tglx@linutronix.de>
CommitDate: Mon Sep 12 09:52:49 2011 +0200

Parent:     ed585a651681e genirq: Make irq_shutdown() symmetric vs. irq_startup again
Contained:  (no branch) master
Follows:    v3.1-rc5 (93)
Precedes:   v3.2-rc1 (11433)

genirq: Add IRQCHIP_SKIP_SET_WAKE flag

Some irq chips need the irq_set_wake() functionality, but do not
require a irq_set_wake() callback. Instead of forcing an empty
callback to be implemented add a flag which notes this fact. Check for
the flag in set_irq_wake_real() and return success when set.

Signed-off-by: Santosh Shilimkar <santosh.shilimkar@ti.com>
Cc: Thomas Gleixner <tglx@linutronix.de>

### arm-gic

在arm gic的实现中，既没有提供IRQCHIP_SKIP_SET_WAKE的flag，也没有实现set_irq_wake的实现。这是因为Linux内核认为gic不处理休眠唤醒的问题，这些应该由平台来基于stacked irqchip来实现。[4](http://www.wowotech.net/admin/#fn.4)  
Marc Zyngier在内核的邮件有如下回复

> I don't have any strong feeling against this series (anything that  
> removes hacks from the GIC code has my full and unconditional support),  
> but I'd just like to make sure I understand the issue.
> 
> There is (AFAIU) 3 cases when suspending:
> 
> 1. The GIC is in an always-on domain: SKIP_SET_WAKE is set, because  
>     
> 
> there is nothing to do (we can always wake up). Problem solved.
> 
> 1. The GIC gets powered off, but we have additional HW that will take  
>     
> 
> care of the wake-up: this is implemented by a stacked irqchip that will  
> do the right thing: irq_set_wake only looks at the top level irqchip, so  
> the GIC flag isn't observed, and this should work (maybe by luck…).
> 
> 1. The GIC gets powered off and nothing will wake us up. I'd say that in  
>     
> 
> this case, having programmed a wake-up interrupt is a bit silly, and  
> doing S2R is equivalent to committing suicide. Do we have any mechanism  
> that would avoid getting in that situation?
> 
> Thanks,
> 
> M.

在Marc Zyngier的回复中，已经提到3种case

1. GIC处于always on的电源域，这种情况下，通过设置SKIP_SET_WAKE就可以简单的解决我们的问题，因为不需要做任何事，系统总可以通过gic来唤醒。  
    
2. GIC会被掉电，这种情况下，需要额外的硬件来将系统唤醒（意味着各个SOC厂商都有各不相同的实现），这种情况下，gic的驱动也无法去处理，这就需要通过stacked irqchip来实现，平台的irqchip通过作为顶级的irqchip，gic作为下级irqchip就可以由SOC厂商去具体定制唤醒功能的设置。  
    
3. GIC会掉电，没有任何办法唤醒系统。这种情况完全是错误的，休眠等于自杀，需要避免进入这个状况。  
    

所以当soc据有唤醒能力时，需要通过平台的gic驱动来实现，arm gic驱动作为平台gic的父设备。

### linux内核接口

- /sys/power/pm_wakeup_irq  
    可以查询到最近一次休眠的唤醒源  
    

### implement stacked irqchip

#### used function & macors

- IRQCHIP_DECLARE  
    
    用来声明一个irqchip驱动，包含compatible属性，irqchip的名称，以及初始化函数  
    device tree  
    dts中配置了sysirq，其parent是gic，
    
    gic: interrupt-controller@58000000 {
        status = "okay";
        compatible = "arm,gic-v3";
        #interrupt-cells = <3>;
        #address-cells = <2>;
        #size-cells = <2>;
        ranges;
        interrupt-controller;
        reg =	<0x0 0x58000000 0x0 0x10000>,		// GICD
            <0x0 0x58040000 0x0 0x100000>;		// GICR0~7
        interrupts = <1 9 4>;
    };
    
    sysirq: sysirq@system {
        compatible = "xxx,XX-sysirq", "syscon";
        #interrupt-cells = <3>;
        interrupt-parent = <&gic>;
        interrupt-controller;
        ranges;
        services = <&tee_regmap>;
        irq-map = <48 2 0>, // GPIO
        <29 4 1>, // AON RTC
        <31 6 0>, // aon timer1
        <30 8 0>, // aon time0
        <9 10 0>, // eth0
        <33 18 0>, // canfd0
        <36 14 0>; // canfd1
        reg = <0x0 0x43060000 0x0 0x634>;
    };
    
- get map configuration from device tree node.  
    
    irq-map = <irq-number enable-bit polar-bit>;
    #address-cells = <2>;
    #size-cells = <2>;
    reg = <0x0 0x43840000 0x0 0x634>;
    

#### irq domain

关于irq domain可以参考以下文档  
[https://lwn.net/Articles/487684/](https://lwn.net/Articles/487684/)

#### irq initialization

(gdb) bt
#0  XX_irq_of_init (node=0xffffffc03efed568, parent=0x0 <bl1_entrypoint>) at ./include/linux/irqdomain.h:298
#1  0xffffff8008856224 in of_irq_init (matches=<optimized out>) at drivers/of/irq.c:546
#2  0xffffff8008849240 in irqchip_init () at drivers/irqchip/irqchip.c:29
#3  0xffffff800883241c in init_IRQ () at arch/arm64/kernel/irq.c:91
#4  0xffffff80088309f8 in start_kernel () at init/main.c:611
#5  0x0000000000000000 in ?? ()
Backtrace stopped: previous frame identical to this frame (corrupt stack?)

查找partent的irq, 并添加新的irq_domain

domain_parent = irq_find_host(parent);
if (!domain_parent) {
    pr_err("XX_irq: interrupt-parent not found\n");
    return -EINVAL;
}

domain = irq_domain_add_hierarchy(domain_parent, 0, intpol_num, node,
                  &sysirq_domain_ops, chip_data);
if (!domain) {
    ret = -ENOMEM;
    goto out_unmap;
}

上面是主要内容，分为两步

1. 获取parent的irq domain  
    
2. 添加新的irq_domain来实现需要的操作  
    

#### irq domain ops

static int XX_irq_domain_translate(struct irq_domain *d,
                                struct irq_fwspec *fwspec,
                                unsigned long *hwirq,
                                unsigned int *type)
{
    if (is_of_node(fwspec->fwnode)) {
        if (fwspec->param_count != 3)
            return -EINVAL;

        /* No PPI should point to this domain */
        if (fwspec->param[0] != GIC_SPI)
            return -EINVAL;

        *hwirq = fwspec->param[1];
        *type = fwspec->param[2] & IRQ_TYPE_SENSE_MASK;
        return 0;
    }

    return -EINVAL;
}

static const struct irq_domain_ops sysirq_domain_ops = {
    .translate	= XX_irq_domain_translate,
    .alloc		= XX_irq_domain_alloc,
    .free		= irq_domain_free_irqs_common,
};

其中最重要的就是 translate 接口，这个接口实现了irq号的转换功能。

## ATF软件

arm atf中实现了psci协议，当系统休眠时，可以通过psci来进入休眠。psci方法：PSCI_CPU_SUSPEND_AARCH64，PSCI_CPU_SUSPEND_AARCH32  
suspend的方法原型为：

int psci_cpu_suspend(unsigned int power_state,
        uintptr_t entrypoint,
        u_register_t context_id);

### 参数：

power_state

power_state参数比较复杂，power_state定义了系统要处于的低功耗模式，根据系统的实现，有两种格式可供选择。  

entrypoint

resume之后的入口地址，需要提供PA，或者IPA

> The entry_point_address parameter is used by the caller to specify where code execution needs to resume at wakeup time. The parameter must be a Physical Address (PA), or, for a guest OS in a virtualized platform, an Intermediate Physical Address (IPA). In this case, the hypervisor must trap the call. Further details can be found in section

context_id

回传参数，当系统resume之后，psci将这个值放入X0/W0/R0，作为第一个参数传递过去。  

### 平台相关实现

在平台的实现中，ATF有几个函数是需要SOC平台进行实现的。  
在atf中，psci lib已经实现了平台无关部份的代码，部份代码需要soc平台来根据自己的需要来实现。psci lib中与休眠唤醒关系不大，上述参数中entrypoint以及context_id都时psci代码来处理，SOC平台无需关心此部份代码。  
为了有一个对atf中的休眠唤醒流程有一个直观的影响，使用下面的代码调用时序图来说明

![atf-suspend-resume.png](https://schspa.tk/2020/07/10/assets/atf-suspend-resume.png)

## refs

[https://patchwork.kernel.org/patch/9343061/](https://patchwork.kernel.org/patch/9343061/)  
[https://patchwork.kernel.org/patch/6799191/](https://patchwork.kernel.org/patch/6799191/)

## Footnotes:

[1](http://www.wowotech.net/admin/#fnr.1)

虽然gic可以工作来输出唤醒信号，但是输出的唤醒信号仍需gic之外的控制器来处理

[2](http://www.wowotech.net/admin/#fnr.2)

CPU power down还需要SOC内部power controller的支持，具体SOC也有较大区别，在此不做关系

[3](http://www.wowotech.net/admin/#fnr.3)

视SOC的实现不同，enable_irq_wake也不是必须调用。

[4](http://www.wowotech.net/admin/#fnr.4)

[https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg1598347.html](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg1598347.html)

标签: [suspend](http://www.wowotech.net/tag/suspend) [sleep](http://www.wowotech.net/tag/sleep) [电源管理](http://www.wowotech.net/tag/%E7%94%B5%E6%BA%90%E7%AE%A1%E7%90%86) [中断子系统](http://www.wowotech.net/tag/%E4%B8%AD%E6%96%AD%E5%AD%90%E7%B3%BB%E7%BB%9F) [aarch64](http://www.wowotech.net/tag/aarch64)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Atomic operation in aarch64](http://www.wowotech.net/armv8a_arch/492.html) | [CFS任务的负载均衡（任务放置）](http://www.wowotech.net/process_management/task_placement.html)»

**评论：**

**[bsp](http://www.wowotech.net/)**  
2021-11-26 09:57

"在Marc Zyngier的回复中，已经提到3种case  
  
2.GIC会被掉电，这种情况下，需要额外的硬件来将系统唤醒（意味着各个SOC厂商都有各不相同的实现），这种情况下，gic的驱动也无法去处理，这就需要通过stacked irqchip来实现，平台的irqchip通过作为顶级的irqchip，gic作为下级irqchip就可以由SOC厂商去具体定制唤醒功能的设置。  
  
所以当soc据有唤醒能力时，需要通过平台的gic驱动来实现，arm gic驱动作为平台gic的父设备。"  
  
这两段描述有冲突吧！vendor的irqchip 和arm的gic，到底谁是在最顶级？谁做父设备？

[回复](http://www.wowotech.net/pm_subsystem/491.html#comment-8382)

**[schspa](http://www.wowotech.net/)**  
2021-11-26 12:40

@bsp：谢谢指正，这里写反了。  
arm gic驱动作为平台gic的父设备。  
  
从下方示例中可以看到：gic是sysirq的parent  
  
sysirq: sysirq@system {  
    compatible = "xxx,XX-sysirq", "syscon";  
    #interrupt-cells = <3>;  
    interrupt-parent = <&gic>;  
    interrupt-controller;  
    ranges;  
    services = <&tee_regmap>;  
    irq-map = <48 2 0>, // GPIO  
    <29 4 1>, // AON RTC  
    <31 6 0>, // aon timer1  
    <30 8 0>, // aon time0  
    <9 10 0>, // eth0  
    <33 18 0>, // canfd0  
    <36 14 0>; // canfd1  
    reg = <0x0 0x43060000 0x0 0x634>;  
};

[回复](http://www.wowotech.net/pm_subsystem/491.html#comment-8383)

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
    
    - [X-008-UBOOT-支持命令行(Bubblegum-96平台)](http://www.wowotech.net/x_project/bubblegum_uboot_cmdline.html)
    - [fixmap addresses原理](http://www.wowotech.net/memory_management/440.html)
    - [Linux系统如何标识进程？](http://www.wowotech.net/process_management/pid.html)
    - [Linux电源管理(10)_autosleep](http://www.wowotech.net/pm_subsystem/autosleep.html)
    - [Linux PM QoS framework(3)_per-device PM QoS](http://www.wowotech.net/pm_subsystem/per_device_pm_qos.html)
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
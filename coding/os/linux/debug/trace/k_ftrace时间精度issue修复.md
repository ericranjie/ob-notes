作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2018-4-16 19:07 分类：[Linux内核分析](http://www.wowotech.net/sort/linux_kenrel)

一、前言

在调试ATMEL SAMA5D3 上的ftrace功能的时候，发现了一个时间精度的问题，本文主要记录这个issue修复的过程，方便后续查阅。linux内核版本是4.4.19，当然，最新的内核中仍然存在这个issue。

二、操作步骤

在通过irqsoff tracer跟踪最大中断延迟的时候发现了这个问题，具体操作步骤如下：

（1）挂载debugfs

|   |
|---|
|# mount -t debugfs nodev /sys/kernel/debug|

（2）查看目前系统中已经内嵌的tracer

|   |
|---|
|# cat /sys/kernel/debug/tracing/available_tracers|

可能的输出是：wakeup wakeup_rt preemptirqsoff preemptoff irqsoff function nop，这个和系统配置有关。这里我们将使用irqsoff这个tracer。

（3）查看当前的tracer

|   |
|---|
|# cat /sys/kernel/debug/tracing/current_tracer|

如果没有执行trace任务，那么一般是nop

（4）设定irqsoff是当前的tracer

|   |
|---|
|# echo irqsoff > /sys/kernel/debug/tracing/current_tracer|

当然可以通过step （3）中的命令来确定tracer设定情况。

（5）清除上次的max latency

|   |
|---|
|# echo 0 > /sys/kernel/debug/tracing/tracing_max_latency|

（6）随后我们需要打开跟踪，将跟踪信息输入到ring buffer中：

|   |
|---|
|# echo 1 > /sys/kernel/debug/tracing/tracing_on|

这时候需要给系统一些负载，以便让最大中断延迟场景出现。

（7）关闭输出

|   |
|---|
|# echo 0 > /sys/kernel/debug/tracing/tracing_on|

（8）保存跟踪结果

|   |
|---|
|# cat /sys/kernel/debug/tracing/trace > /tmp/trace.txt|

查阅/tmp/trace.txt文件并分析

（9）设定nop是当前的tracer

|   |
|---|
|# echo nop > /sys/kernel/debug/tracing/current_tracer|

三、问题的发现

/tmp/trace.txt文件如下：

```cpp
# tracer: irqsoff  
#  
# irqsoff latency trace v1.1.5 on 4.4.19  
# --------------------------------------------------------------------  
# latency: 10000 us, 441/441, CPU#0 | (M:server VP:0, KP:0, SP:0 HP:0)  
#    -----------------  
#    | task: swapper-0 (uid:0 nice:0 policy:0 rt_prio:0)  
#    -----------------  
#  => started at: irq_svc  
#  => ended at:   __do_softirq  
#  
#  
#                  ------=> CPU#  
#                 / _-----=> irqs-off  
#                | / _----=> need-resched  
#                || / _---=> hardirq/softirq  
#                ||| / _--=> preempt-depth  
#                |||| /     delay  
#  cmd     pid   ||||| time  |   caller  
#     \   /      |||||  \    |   /  
<...>-593     0d...    0us : do_work_pending  
<...>-593     0d...    0us : do_work_pending  
<...>-593     0d...    0us : trace_hardirqs_on <-do_work_pending

……

-0       0d.h.    0us : tick_handle_periodic <-ch2_irq  
-0       0d.h.    0us : tick_periodic.constprop.5 <-tick_handle_periodic  
**-0       0d.h.    0us#: do_timer <-tick_periodic.constprop.5  
-0       0d.h. 10000us : calc_global_load <-do_timer**  
-0       0d.h. 10000us : update_wall_time <-tick_periodic.constprop.5  
-0       0d.h. 10000us : tc_get_cycles32 <-update_wall_time
……
```

通过这个文件，我们可以看成跟踪到的最大中断延迟（最长的关闭中断的时间）是10000us，高达10ms的关中断时间，这有点离谱啊。不过还好后面有更详细的信息：header中的started at和ended at给出了关中断的函数区间，后面给出了详细的栈的回溯信息。这里有一个很奇怪的地方（上面黑色粗体的两行），时间戳在这两行中有一个跳变，从0之间跳变到了10000us，很显然，ftrace给出的时间戳精度不足。

四、寻找问题原因

ftrace中的时间戳信息是通过sched_clock实现的，从名字也能看出来，这是调度器模块相关的函数，在调度器代码中（kernel/sched/clock.c）有一个弱符号定义：

```cpp
unsigned long long weak sched_clock(void)  
{  
return (unsigned long long)(jiffies - INITIAL_JIFFIES)  
* (NSEC_PER_SEC / HZ);  
}
```

显然这是一个基于jiffies的clock，其精度和tick相关。在我们的系统中，HZ=100，每10ms产生一次tick，符合上面的观察：ftrace的时间戳精度是10ms。

当然，如果你嫌弃上面这个sched_clock时间精度差，arch相关的代码可以自定义一个sched_clock来代替上面这个调度器中的弱符号，显然，在ATMEL SAMA5D3的timer驱动中没有实现这个功能，从而使用了调度器模块中的那个弱符号的schec clock，从而导致了ftrace的时间精度问题。

五、问题的解决

既然如此，看来我们需要修改一下ATMEL SAMA5D3的timer驱动代码。这里我们还需要了解一件事情：在内核时间子系统的代码中（kernel/time/sched_clock.c）有通用sched clock的支持。本身sched clock需要是64bit（不容易溢出），ns为单位、单调上升，更重要的是要快。当然未必所有平台都支持64 bit的一个HW counter，因此时间子系统提供了这样一个通用模块，当你的平台的counter不足64 bit的时候，你可以借用它来扩展成64 bit的sched clock。

ATMEL SAMA5D3的timer驱动代码位于drivers/clocksource/tcb_clksrc.c中，首先在头文中中增加一行：

```cpp
include <linux/sched_clock.h>
```

之所以要包含这个头文件是因为下面我们要调用时间子系统的通用sched clock模块的sched_clock_register接口。

时间子系统的通用sched clock模块仍然需要底层HW counter（不足64bit）的支持，在ATMEL SAMA5D3平台中，我们有3个32bit的Time Counter channel，我们选择其一作为sched clock，并定义如下函数获取这个32 bit counter的cycle值：

```cpp
static cycle_t tc_read_cycles(void)  
{  
return raw_readl(tcaddr + ATMEL_TC_REG(0, CV));  
}
```

最后一步就是注册了，在注册完clocksource之后增加注册sched clock的代码：

```cpp
/* register sched clock */  
sched_clock_register(tc_read_cycles, 32, divided_rate);
```

OK，一切大功告成。

六、问题验证

重复第二节中的步骤，观察输出的跟踪文件，发现所有的时间戳都不再是10ms这样粗糙粒度的了，至此问题解决。

_原创文章，转发请注明出处。蜗窝科技_

标签: [ftrace](http://www.wowotech.net/tag/ftrace) [sched_clock](http://www.wowotech.net/tag/sched_clock)

______________________________________________________________________

« [tty驱动分析](http://www.wowotech.net/tty_framework/435.html) | [致驱动工程师的一封信](http://www.wowotech.net/device_model/429.html)»

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

  - [ACCESS_ONCE宏定义的解释](http://www.wowotech.net/process_management/access-once.html)
  - [X-019-KERNEL-串口驱动开发之数据收发](http://www.wowotech.net/x_project/serial_driver_porting_4.html)
  - [Linux PM QoS framework(1)\_概述和软件架构](http://www.wowotech.net/pm_subsystem/pm_qos_overview.html)
  - [X-023-KERNEL-Linux pinctrl driver的移植](http://www.wowotech.net/x_project/kernel_pinctrl_driver_porting.html)
  - [进化论、人工智能和外星人](http://www.wowotech.net/tech_discuss/toe_ai_et.html)

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

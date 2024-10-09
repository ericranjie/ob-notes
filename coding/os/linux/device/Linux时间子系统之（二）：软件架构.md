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

作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2015-3-7 18:37 分类：[时间子系统](http://www.wowotech.net/sort/timer_subsystem)

一、前言

本文的主要内容是描述内核时间子系统的软件框架。首先介绍了从旧的时间子系统迁移到新的时间子系统的源由，介绍新的时间子系统的优势。第三章汇整了时间子系统的相关文件以及内核配置。最后描述各种内核配置下的时间子系统的数据流和控制流。

二、背景介绍

1、传统内核时间子系统的软件架构

让我们先回到远古的2.4内核时代，其内核的时间子系统的软件框架如下：

[![old time](http://www.wowotech.net/content/uploadfile/201503/747216fbc17ec06dc03f51d2be4ff47e20150307103444.gif "old time")](http://www.wowotech.net/content/uploadfile/201503/e7d0070cea441d7af85c4f11787b05b620150307103110.gif)

首先，在每个architecture相关的代码中要有实现clock event和clock source模块。这里听起来名字是高大上，实际上，这里仅仅是借用了新时间子系统的名词，对于旧的内核，clock event就是通过timer硬件的中断处理函数完成的，在此基础上可以构建tick模块，tick模块维护了系统的tick，例如系统存在10ms的tick，每次tick到来的时候，timekeeping模块就增加系统时间，如果timekeeping完全是tick驱动，那么它精度只能是10ms，为了更高精度，clock source模块就是一个提供tick之间的offset时间信息的接口函数。

最初的linux kernel中只支持低精度的timer，也就是基于tick的timer。如果内核采用10ms的tick，那么低精度的timer的最高的精度也只有10ms，想要取得us甚至ns级别的精度是不可能的。各个内核的驱动模块都可以使用低精度timer实现自己的定时功能。各个进程的时间统计也是基于tick的，内核的调度器根据这些信息进行调度。System Load和Kernel Profiling模块也是基于tick的，用于计算系统负荷和进行内核性能剖析。

从用户空间空间来看，有两种需求，一种是获取或者设定当前的系统时间的接口函数：例如time，stime，gettimeofday等。另外一种是timer相关的接口函数：例如setitimer、alarm等，当timer到期后会向该进程发送signal。

2、为何会引入新的时间子系统软件架构？

但是随着技术发展，出现了下面两种新的需求：

（1）嵌入式设备需要较好的电源管理策略。传统的linux会有一个周期性的时钟，即便是系统无事可做的时候也要醒来，这样导致系统不断的从低功耗（idle）状态进入高功耗的状态。这样的设计不符合电源管理的需求。

（2）多媒体的应用程序需要非常精确的timer，例如为了避免视频的跳帧、音频回放中的跳动，这些需要系统提供足够精度的timer

和低精度timer不同，高精度timer使用了人类的最直观的时间单位ns（低精度timer使用的tick是和内核配置相关，不够直接）。本质上linux kernel提供了高精度timer之后，其实不必提供低精度timer了，不过由于低精度timer存在了很长的历史，并且在渗入到内核各个部分，如果去掉低精度timer很容易引起linux kernel稳定性和健壮性的问题，因此目前的linux kernel保持了低精度timer和高精度timer并存。

在新的需求的推动下，内核开发者对linux的时间子系统的软件框架进行修改，让代码层次更清晰，同时又是灵活可配置的，一个示意性的block图如下所示：

[![timehl](http://www.wowotech.net/content/uploadfile/201503/9285257f74afdad0d1f5e604c1c60d7f20150307103645.gif "timehl")](http://www.wowotech.net/content/uploadfile/201503/2b777e4e50bc7dfec1c153d44904859620150307103544.gif)

在引入multi-core之后，过去HW timer的功能被分成两个部分，一个是free running的system counter，是全局的，不属于任何一个CPU。另外一部分就是产生定时事件的HW block，我们称之timer，timer硬件被嵌入到各个cpu core中，因此，我们更准确的称之为CPU local Timer，这些timer都是基于一个Global counter运作的。在驱动层，我们提供一个clock source chip driver的模块来驱动硬件，这是模块是和硬件体系结构有关的。如果系统内存在多个HW timer和counter block，那么系统中可能会存在多个clock source chip driver。

面对形形色色的timer和counter硬件，linux kernel抽象出了通用clock event layer和通用clock source模块，这两个模块和硬件无关。底层的clock source chip driver会调用通用clock event和clock source模块的接口函数，注册clock source和clock event设备。clock source设备对应硬件的system free running counter，提供了一个基础的timeline。当然了，实际的timeline象一条直线，无限延伸。对于kernel而言，其timeline是构建在system free running counter之上的，因此clocksource 对应的timeline存在溢出问题。如果选用64bit的HW counter，并且输入频率不那么高，那么溢出时间可能会长达50年甚至更多，那么从应用的角度来看，能维持50年左右的timeline也可以接受。如果说clock source是一个time line，那么clock event是在timeline上指定的点产生clock event的设备，之所以能产生异步事件，当然是基于中断子系统了，clock source chip driver会申请中断并调用通用clock event模块的callback函数来通知这样的异步事件。

tick device layer基于clock event设备进行工作的：一般而言，每个CPU形成自己的一个小系统，有自己的调度、有自己的进程统计等，这个小系统都是拥有自己的tick设备，而且是唯一的。对于clock event设备而言就不是这样了，硬件有多少个timer硬件就注册多少个clock event device，各个cpu的tick device会选择自己适合的那个clock event设备。tick device可以工作在periodic mode或者one shot mode，当然，这是和系统配置有关。因此，在tick device layer，有多少个cpu，就会有多少个tick device，我们称之local tick device。当然，有些事情（例如整个系统的负荷计算）不适合在local tick驱动下进行，因此，所有的local tick device中会有一个被选择做global tick device，该device负责维护整个系统的jiffies，更新wall clock，计算全局负荷什么的。

高精度的timer需要高精度的clock event，工作在one shot  mode的tick device工提供高精度的clock event。因此，基于one shot mode下的tick device，系统实现了高精度timer，系统的各个模块可以使用高精度timer的接口来完成定时服务。虽然有了高精度timer的出现, 内核并没有抛弃老的低精度timer机制(内核开发人员试图整合高精度timer和低精度的timer，不过失败了，所以目前内核中，两种timer是同时存在的)。当系统处于高精度timer的时候（tick device处于one shot mode），系统会setup一个特别的高精度timer（可以称之sched timer），该高精度timer会周期性的触发，从而模拟的传统的periodic tick，从而推动了传统低精度timer的运转。因此，一些传统的内核模块仍然可以调用经典的低精度timer模块的接口。

三、时间子系统的文件整理

1、文件汇整

linux kernel 时间子系统的源文件位于linux/kernel/time/目录下，我们整理如下：

|   |   |
|---|---|
|文件名|描述|
|time.c  <br>timeconv.c|time.c文件是一个向用户空间提供时间接口的模块。具体包括：time, stime, gettimeofday, settimeofday,adjtime。除此之外，该文件还提供一些时间格式转换的接口函数（其他内核模块使用），例如jiffes和微秒之间的转换，日历时间（Gregorian date）和xtime时间的转换。xtime的时间格式就是到linux epoch的秒以及纳秒值。  <br>timeconv.c中包含了从calendar time到broken-down time之间的转换函数接口。|
|timer.c|传统的低精度timer模块，基本tick的。|
|time_list.c  <br>timer_status.c|向用户空间提供的调试接口。在用户空间，可以通过/proc/timer_list接口可以获得内核中的时间子系统的相关信息。例如：系统中的当前正在使用的clock source设备、clock event设备和tick device的信息。通过/proc/timer_stats可以获取timer的统计信息。|
|hrtimer.c|高精度timer模块|
|itimer.c|interval timer模块|
|posix-timers.c  <br>posix-cpu-timers.c  <br>posix-clock.c|POSIX timer模块和POSIX clock模块|
|alarmtimer.c|alarmtimer模块|
|clocksource.c  <br>jiffies.c|clocksource.c是通用clocksource driver。其实也可以把system tick也看成一个特定的clocksource，其代码在jiffies.c文件中|
|timekeeping.c  <br>timekeeping_debug.c|timekeeping模块|
|ntp.c|NTP模块|
|clockevent.c|clockevent模块|
|tick-common.c  <br>tick-oneshot.c  <br>tick-sched.c|这三个文件属于tick device layer。  <br>tick-common.c文件是periodic tick模块，用于管理周期性tick事件。  <br>tick-oneshot.c文件是for高精度timer的，用于管理高精度tick时间。  <br>tick-sched.c是用于dynamic tick的。|
|tick-broadcast.c  <br>tick-broadcast-hrtimer.c|broadcast tick模块。|
|sched_clock.c|通用sched clock模块。这个模块主要是提供一个sched_clock的接口函数，调用该函数可以获取当前时间点到系统启动之间的纳秒值。  <br>底层的HW counter其实是千差万别的，有些平台可以提供64-bit的HW counter，因此，在那样的平台中，我们可以不使用这个通用sched clock模块（不配置CONFIG_GENERIC_SCHED_CLOCK这个内核选项），而在自己的clock source chip driver中直接提供sched_clock接口。  <br>使用通用sched clock模块的好处是：该模块扩展了64-bit的counter，即使底层的HW counter比特数目不足（有些平台HW counter只有32个bit）。|

2、通用clock source和clock event的内核配置

（1）CONFIG_GENERIC_CLOCKEVENTS和CONFIG_GENERIC_CLOCKEVENTS_BUILD：使用新的时间子系统的构架，如果不配置，那么将使用第二节描述的旧的时间子系统架构。

（2）曾经有一个CONFIG\_ GENERIC_TIME的配置项对应clocksource的配置，不过在某个版本中删除了，也就是说目前的内核都是使用通用clocksource模块的，无法再退回到过去使用arch相关的clocksource的时代。为了兼容旧风格的timekeeping接口，kernel仍然提供了CONFIG_ARCH_USES_GETTIMEOFFSET这个配置项。由此可见，在软件框架在演化的过程中，如果这是一个被其他模块使用的基础组件，我们不可能是完全推到重来，必须考虑对旧的软件的兼容性，虽然是一个沉重的负担，但是必须这么做。

3、tick device的配置

如果选择了新的时间子系统的软件架构（配置了CONFIG_GENERIC_CLOCKEVENTS），那么内核会打开Timers subsystem的配置选项，主要是和tick以及高精度timer配置相关。和tick相关的配置有三种，包括：

（1）无论何时，都启用用周期性的tick，即便是在系统idle的时候。这时候要配置CONFIG_HZ_PERIODIC选项。

（2）在系统idle的时候，停掉周期性tick。对应的配置项是CONFIG_NO_HZ_IDLE。配置tickless idle system也会同时enable NO_HZ_COMMON的选项。

（3）Full dynticks system。即便在非idle的状态下，也就是说cpu上还运行在task，也可能会停掉tick。这个选项和实时应用相关。对应的配置项是CONFIG_NO_HZ_FULL。配置Full dynticks system也会同时enable NO_HZ_COMMON的选项。本文不描述该系统，有兴趣的同学可以自行阅读。

上面的三个选项只能是配置其一。上面描述的是新的内核配置方法，对于旧的内核，CONFIG_NO_HZ用来配置dynamic tick或者叫做tickless idle system（非idle时有周期性tick，idle状态，timer中断不再周期性触发，只会按照需要触发），为了兼容旧的系统，新的内核仍然支持了这个选项。

4、timer模块的配置

和高精度timer相关的配置比较简单，只有一个CONFIG_HIGH_RES_TIMERS的配置项。如果配置了高精度timer，或者配置了NO_HZ_COMMON的选项，那么一定需要配置CONFIG_TICK_ONESHOT，表示系统支持支持one-shot类型的tick device。

5、 如何进行时间子系统的内核配置

根据上一节的描述，linux内核可以有下面的两种时间子系统的构架：

（1）新的通用时间子系统软件框架（配置了CONFIG_GENERIC_CLOCKEVENTS）

（2）传统时间子系统软件框架（不配置CONFIG_GENERIC_CLOCKEVENTS，配置CONFIG_ARCH_USES_GETTIMEOFFSET）

对于我们工程人员，除非你是维护一个旧的系统，否则当然使用新的通用时间子系统软件框架了，这时候可能的配置包括：

（1）使用低精度timer和周期tick。传统的linuxer应该会喜欢这个配置，保持和传统的unix的一致性。

（2）使用低精度timer和Dynamic tick

（3）使用高精度timer和周期tick

（4）使用高精度timer和Dynamic tick。新潮的linux应该会喜欢这个配置，一个字，cool……

注：本文主要描述普通的dynamic tick系统（tickless idle system），后续会有专门的文章描述full dynamic tick系统。

四、时间子系统的数据流和控制流

1、使用低精度timer + 周期tick

我们首先看周期性tick的实现。起始点一定是底层的clock source chip driver，该driver会调用注册clock event的接口函数（clockevents_config_and_register或者clockevents_register_device），一旦增加了一个clock event device，需要通知上层的tick device layer，毕竟有可能新注册的这个device更好、更适合某个tick device呢（通过调用tick_check_new_device函数实现）。要是这个clock event device被某个tick device收留了（要么该tick device之前没有匹配的clock event device，要么新的clock event device更适合该tick device），那么就启动对该tick device的配置（参考tick_setup_device）。根据当前系统的配置情况（周期性tick），会调用tick_setup_periodic函数，这时候，该tick device对应的clock event device的clock event handler被设置为tick_handle_periodic。底层硬件会周期性的产生中断，从而会周期性的调用tick_handle_periodic从而驱动整个系统的运转。需要注意的是：即便是配置了CONFIG_NO_HZ和CONFIG_TICK_ONESHOT，系统中没有提供one shot的clock event device，这种情况下，整个系统仍然是运行在周期tick的模式下。

下面来到低精度timer模块了，其实即便没有使能高精度timer，内核也会把高精度timer模块的代码编译进kernel的image中，这一点可以从Makefile文件中看出：

> obj-y += time.o timer.o **hrtimer.o** itimer.o posix-timers.o posix-cpu-timers.o

高精度timer总是会被编入最后的kernel中。在这种构架下，各个内核模块也可以调用linux kernel中的高精度timer模块的接口函数来实现高精度timer，但是，这时候高精度timer模块是运行在低精度的模式，也就是说这些hrtimer虽然是按照高精度timer的红黑树进行组织，但是系统只是在每一周期性tick到来的时候调用hrtimer_run_queues函数，来检查是否有expire的hrtimer。毫无疑问，这里的高精度timer也就是没有意义了。

由于存在周期性tick，低精度timer的运作毫无压力，和过去一样。

2、低精度timer + Dynamic Tick

系统开始的时候并不是直接进入Dynamic tick mode的，而是经历一个切换过程。开始的时候，系统运行在周期tick的模式下，各个cpu对应的tick device的（clock event device的）event handler是tick_handle_periodic。在timer的软中断上下文中，会调用tick_check_oneshot_change进行是否切换到one shot模式的检查，如果系统中有支持one-shot的clock event device，并且没有配置高精度timer的话，那么就会发生tick mode的切换（调用tick_nohz_switch_to_nohz），这时候，tick device会切换到one shot模式，而event handler被设置为tick_nohz_handler。由于这时候的clock event device工作在one shot模式，因此当系统正常运行的时候，在event handler中每次都要reprogram clock event，以便正常产生tick。当cpu运行idle进程的时候，clock event device不再reprogram产生下次的tick信号，这样，整个系统的周期性的tick就停下来。

高精度timer和低精度timer的工作原理同上。

3、高精度timer + Dynamic Tick

同样的，系统开始的时候并不是直接进入Dynamic tick mode的，而是经历一个切换过程。系统开始的时候是运行在周期tick的模式下，event handler是tick_handle_periodic。在周期tick的软中断上下文中（参考run_timer_softirq），如果满足条件，会调用hrtimer_switch_to_hres将hrtimer从低精度模式切换到高精度模式上。这时候，系统会有下面的动作：

（1）Tick device的clock event设备切换到oneshot mode（参考tick_init_highres函数）

（2）Tick device的clock event设备的event handler会更新为hrtimer_interrupt（参考tick_init_highres函数）

（3）设定sched timer（也就是模拟周期tick那个高精度timer，参考tick_setup_sched_timer函数）

这样，当下一次tick到来的时候，系统会调用hrtimer_interrupt来处理这个tick（该tick是通过sched timer产生的）。

在Dynamic tick的模式下，各个cpu的tick device工作在one shot模式，该tick device对应的clock event设备也工作在one shot的模式，这时候，硬件Timer的中断不会周期性的产生，但是linux kernel中很多的模块是依赖于周期性的tick的，因此，在这种情况下，系统使用hrtime模拟了一个周期性的tick。在切换到dynamic tick模式的时候会初始化这个高精度timer，该高精度timer的回调函数是tick_sched_timer。这个函数执行的函数类似周期性tick中event handler执行的内容。不过在最后会reprogram该高精度timer，以便可以周期性的产生clock event。当系统进入idle的时候，就会stop这个高精度timer，这样，当没有用户事件的时候，CPU可以持续在idle状态，从而减少功耗。

4、高精度timer + 周期性Tick

这种配置不多见，多半是由于硬件无法支持one shot的clock event device，这种情况下，整个系统仍然是运行在周期tick的模式下。

_原创文章，转发请注明出处。蜗窝科技_

[http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html)

标签: [软件框架](http://www.wowotech.net/tag/%E8%BD%AF%E4%BB%B6%E6%A1%86%E6%9E%B6) [时间子系统](http://www.wowotech.net/tag/%E6%97%B6%E9%97%B4%E5%AD%90%E7%B3%BB%E7%BB%9F)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [计算机科学基础知识（五）: 动态链接](http://www.wowotech.net/basic_subject/dynamic-link.html) | [计算机科学基础知识（四）: 动态库和位置无关代码](http://www.wowotech.net/basic_subject/pic.html)»

**评论：**

**奥特曼**\
2022-10-18 16:11

你好 我有一个问题\
在用户层调用usleep(1000)时，发现实际用时有10000us(100HZ的配置)，但是我是有自己的HRTIMER的，不知道为啥没起效，经过排查，我自己注册的clk_event被系统的dummy_timer替代作为了pre_cpu的clk_evnent （我是ARMV7的），而我注册的clk_event却作为了Broadcast。所以定时应该是用了dummy_timer的，而系统又是100HZ的，所以精度只有10ms了。\
我想知道我应该怎么做才能让我的clk_event作为定时器？\
谢谢大哥

[回复](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html#comment-8688)

**奥特曼**\
2022-10-18 16:16

@奥特曼：补充: kernel版本是4.14.73\
启动时有\
Clockevents: could not switch to one-shot\
Clockevents: could not switch to one-shot\
Clockevents: could not switch to one-shot\
Clockevents: could not switch to one-shot\
等打印，经排查是tick_switch_to_oneshot()中的tick_device_is_functional()返回了真，说明是挑选了当前的pre_cpu的dummy_timer。

[回复](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html#comment-8689)

**kenneth**\
2018-02-27 15:57

2.4 内核应该是没有timekeeping吧，timekeeping是在高版本跟hrtimer 一起出来的

[回复](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html#comment-6568)

**gnx**\
2017-11-22 23:08

我觉得下面一段描述值得商榷\
“4、高精度timer + 周期性Tick\
这种配置不多见，多半是由于硬件无法支持one shot的clock event device，这种情况下，整个系统仍然是运行在周期tick的模式下。”\
既然支持高精度timer，那么clock event device或者tick device肯定是支持one shot的，这种组合模式一般是由于在启动参数中配置了nohz=on，或者没有配置NO_HZ_COMMON选项，这种组合确实不多见

[回复](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html#comment-6232)

**gnx**\
2017-11-22 23:10

@gnx：夜深了，脑袋有点糊了，应该是nohz=off。。。

[回复](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html#comment-6233)

**江南书生**\
2016-10-17 15:44

你好，我想问一下，现在我开启了高精度的时钟，在用户空间定时ms级别还是比较准的，但是在us级别就出现实时性很差的情况，有时快有时慢。我想知道是不是实时性不好啊？

[回复](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html#comment-4723)

**[linuxer](http://www.wowotech.net/)**\
2016-10-18 18:28

@江南书生：在用户空间其实永远也不能达到很高的精度，因为时间精度依赖于调度器。

[回复](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html#comment-4737)

**[江南书生](http://no/)**\
2016-10-19 17:35

@linuxer：我现在直接用硬件timer玩起来的，感觉你写的东西很不错，有没有群，加一下大家交流一下啊？

[回复](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html#comment-4746)

**[linuxer](http://www.wowotech.net/)**\
2016-10-19 18:51

@江南书生：网站上有写的，呵呵～～\
联系我们

QQ: 2841962892

QQ交流群:457024058

E-mail: service#wowotech.net\
发送邮件时请将#替换为@

[回复](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html#comment-4747)

**驴肉火烧**\
2016-10-11 18:53

wowo你好，我现在遇到一个问题，向您请教一下，arm64架构单板如果只使用arch_sys_counter作为时钟源，发现定时器不准，每睡眠10ms就会有10ms左右的误差，时钟源是50MHZ的，是否因为我内核版本低呢，3.10的内核，arm64下没有sched_clock

[回复](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html#comment-4694)

**[wowo](http://www.wowotech.net/)**\
2016-10-11 21:35

@驴肉火烧：50MHZ 10ms差10ms？怎么可能？

[回复](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html#comment-4695)

**驴肉火烧**\
2016-10-12 08:56

@wowo：我也是很纳闷的，如果使用了jiffes到时可以解释的通，因为CONFIG_HZ = 100，但是从启动流程上很明确的将这个jiffies的默认时钟源切换为arch_sys_counter时钟源，我也是百思不得其解，参考了arm64的JUNO单板，如果将其arm,armv7-timer-mem时钟源禁用，也会有这种情况，难道arm64的时钟源必须依赖一个arm,armv7-timer-mem的时钟源么？

[回复](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html#comment-4696)

**[wowo](http://www.wowotech.net/)**\
2016-10-12 14:11

@驴肉火烧：从你的描述里面,我觉得,你的系统是不是没有使能"arch_sys_counter" clock source?\
因为只有dts中配置了"arm,armv7-timer"或者"arm,armv8-timer",才会走注册流程(arch_timer_init).

/\* drivers/clocksource/arm_arch_timer.c \*/\
CLOCKSOURCE_OF_DECLARE(armv7_arch_timer, "arm,armv7-timer", arch_timer_init);  \
CLOCKSOURCE_OF_DECLARE(armv8_arch_timer, "arm,armv8-timer", arch_timer_init);

[回复](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html#comment-4697)

**驴肉火烧**\
2016-10-12 16:51

@wowo：这里dts有描述，也完成了注册，但是我读/proc/timer_list\
Tick Device: mode:     0\
Per CPU device: 0\
Clock Event Device: arch_sys_timer\
max_delta_ns:   42949672900\
min_delta_ns:   1000\
mult:           214748365\
shift:          32\
mode:           3\
next_event:     88105540000000 nsecs\
set_next_event: arch_timer_set_next_event_phys\
set_mode:       arch_timer_set_mode_phys\
event_handler:  tick_handle_periodic\
retries:        0

event_handler这里看着是有问题的，拜读了你们的文章，看样子是时钟没能完成切换？

[回复](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html#comment-4698)

**驴肉火烧**\
2016-10-12 18:06

@驴肉火烧：知道原因了，仅描述了compatible = "arm,armv8-timer";还是不够的，系统中是存在compatible = "arm,armv7-timer-mem";时钟源的，dts中补充上就正常了，现在纳闷的是，我在飞腾arm64的服务器上测试，其dts中没有compatible = "arm,armv7-timer-mem";的描述，时钟也是准的，想不通了

[回复](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html#comment-4699)

**liana07151018**\
2016-07-22 21:56

那么请问wowo大神，hrtimer模式下的时候，更高的时间精度是基于怎样的时钟源选择？因为内核中，默认clock event的时钟频率是32.768KHz，所以想问问，要使用高精度定时器，内核是如何一层层实现ns级精度的

[回复](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html#comment-4305)

**[wowo](http://www.wowotech.net/)**\
2016-07-23 15:17

@liana07151018：如果要用hrtimer，clock event就不能用32K的了，要用一个更高精度的。

[回复](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html#comment-4306)

**sched_fork**\
2015-11-11 16:24

好文章，拜读。以下两个小细节：

> 能维持50年左右的timeline也可以接受。\
> clocksource 回饶不是问题，就算回饶了，算出来的 delta 还是正确的。只要两次读取 clocksource 的时间内没有回饶就不会有bug。\
> 有些事情（例如整个系统的负荷计算）不适合在local tick驱动下进行，因此，所有的local tick device中会有一个被选择做global tick device，该device负责维护整个系统的jiffies，更新wall clock，计算全局负荷什么的。\
> x86 架构里面的 local tick 是 apci timer，broadcast timer 是 hpet timer。\
> arm 架构里面的 local tick 是 arch timer，而 broadcast timer 一般是 soc 中额外的 timer。因为如果整个 cluster 掉电的时候， arch timer 也 clock gating 了。不能产生一个唤醒信号把 cluster 唤醒。

[回复](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html#comment-3028)

**[linuxer](http://www.wowotech.net/)**\
2015-11-12 10:19

@sched_fork：多谢您的意见。关于回滚问题，我又仔细看了看代码，系统的timeline实际上是timekeeping模块负责的，而该模块是在tick到来的时候，根据上次和当前的clocksource的delta来更新timeline，因此，clocksource的回滚不是问题，只要delta值是对的就OK了。

关于tick问题，x86的架构我不是很熟悉，不过根据您的描述，看起来apci timer应该是per cpu的，而hpet timer应该所有cpu core共享的，并且在CPU core idle的时候，仍然可以work。

[回复](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html#comment-3036)

**[electrlife](http://www.wowotech.net/)**\
2016-02-26 14:39

@linuxer：这里有点不解关于clocksource的回饶问题，正如上面所说，clocksource 回饶不是问题，就算回饶了，算出来的 delta 还是正确的。只要两次读取 clocksource 的时间内没有回饶就不会有bug。我的疑惑是如何保证两次读取 clocksource 的时间内没有回饶。在clocksource中确实有max_idle_ns这个变量，不知是否和这个回饶有关，如果样，Linux应该会在适当的时间点去更新clocksource的cycle_last变量，以保证用户在使用clocksource->read来计算cycle_delta出现回铙问题。关于这个部分代码在哪里可以找到，谢谢！

[回复](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html#comment-3545)

**[madang](http://www.wowotech.net/)**\
2015-11-10 17:30

周期tick的软中断上下文中（参考run_timer_softirq），如果满足条件，会调用hrtimer_switch_to_hres将hrtimer从低精度模式切换到高精度模式上。\
//---------------------------------\
这里好像有点问题，看代码应该是hrtimer_run_queues 被调用到的，这里应该是在硬中断上下文。

[回复](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html#comment-3022)

**[linuxer](http://www.wowotech.net/)**\
2015-11-11 12:14

@madang：void run_local_timers(void)\
{\
hrtimer_run_queues();\
raise_softirq(TIMER_SOFTIRQ);\
}\
在timer的硬件上下文中会调用hrtimer_run_queues，只是用于在周期性tick下驱动高精度timer（这时候，系统配置是低精度timer + 周期tick ），但是，我的文章中描述的是从低精度模式切换到高精度模式的过程，实际上，还是在timer的软中断上下文中（参考run_timer_softirq）中实现切换的，在hrtimer_run_queues函数中，我没有看到模式切换的内容啊

[回复](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html#comment-3024)

**[madang](http://www.wowotech.net/)**\
2015-11-11 15:51

@linuxer：void hrtimer_run_queues(void)\
{\
......\
if (hrtimer_hres_active())\
return;\
.......\
}\
这个代码的意思应该是：如果处于高精度模式则退出 ？

[回复](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html#comment-3027)

**[linuxer](http://www.wowotech.net/)**\
2015-11-11 17:37

@madang：在这个场景下，驱动hrtimer_run_queues函数的是周期性tick。这时候，系统会在每一次周期性tick到来的时候调用hrtimer_run_queues函数，该函数来检查是否有expire的hrtimer。毫无疑问，这里的高精度timer也就是没有意义了。

如果处于高精度模式的话，不会使用这个函数。我上面的回复有些武断了，需要修改。

[回复](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html#comment-3030)

**[firo](http://firoyang.org/)**\
2015-09-22 15:18

一些传统的内核moik...

模块?

[回复](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html#comment-2657)

**[linuxer](http://www.wowotech.net/)**\
2015-09-22 23:32

@firo：多谢指正，已经修改了。

[回复](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html#comment-2667)

**[printk](http://www.wowotech.net/)**\
2015-04-21 17:22

在kernel中，time 就是 frequence吗 这两个概念感觉是一样的，因为我要调整一个LDB的时钟频率，就要从源头看起 ，看这货是怎么从PLL5上分频出来的。。而PLL5 又来源于时钟晶振。 所以我是在调时间吗。。。\
原谅我的思维混乱，毕竟工作了一天。。

[回复](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html#comment-1540)

**[wowo](http://www.wowotech.net/)**\
2015-04-21 21:33

@printk：只有“愚蠢”的人类才需要“时间”这个概念啦，对kernel而言，它只知道tick啊，Tick，Tick，Tick...\
Tick就是频率啊，无论是周期固定的频率还是周期不定的频率。\
频率是什么呢？是变化。嗯，只有变化的才是有生命的...

[回复](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html#comment-1542)

**[printk](http://www.wowotech.net/)**\
2015-04-22 11:16

@wowo：wowo大虾简直就是kernel中的哲学家阿。。。这个地方太有意思啦，以后常来分享~

[回复](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html#comment-1544)

1 [2](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html/comment-page-2#comments)

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

  - [Linux PM QoS framework(2)\_PM QoS class](http://www.wowotech.net/pm_subsystem/pm_qos_class.html)
  - [显示技术介绍(3)\_CRT技术](http://www.wowotech.net/display/crt_intro.html)
  - [linux kernel内存回收机制](http://www.wowotech.net/memory_management/233.html)
  - [CMA模块学习笔记](http://www.wowotech.net/memory_management/cma.html)
  - [Linux电源管理(8)\_Wakeup count功能](http://www.wowotech.net/pm_subsystem/wakeup_count.html)

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

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

作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2017-5-17 18:55 分类：[时间子系统](http://www.wowotech.net/sort/timer_subsystem)

一、前言

时钟或者钟表（clock）是一种计时工具，每个人都至少有一块，可能在你的手机里，也可能佩戴在你的手腕上。如果Linux也是一个普通人的话，那么她的手腕上应该有十几块手表，包括：CLOCK_REALTIME、CLOCK_MONOTONIC、CLOCK_PROCESS_CPUTIME_ID、CLOCK_THREAD_CPUTIME_ID、CLOCK_MONOTONIC_RAW、CLOCK_REALTIME_COARSE、CLOCK_MONOTONIC_COARSE、CLOCK_BOOTTIME、CLOCK_REALTIME_ALARM、CLOCK_BOOTTIME_ALARM、CLOCK_TAI。本文主要就是介绍Linux内核中的形形色色的“钟表”。

二、理解Linux中各种clock分类的基础

既然本文讲Linux中的计时工具，那么我们首先面对的就是“什么是时间？”，这个问题实在是太难回答了，因此我们这里就不正面回答了，我们只是从几个侧面来窥探时间的特性，而时间的本质就留给物理学家和哲学家思考吧。

1、如何度量时间

时间往往是和变化相关，因此人们往往喜欢使用有固定周期变化规律的运动行为来定义时间，于是人们把地球围自转一周的时间分成24份，每一份定义为一个小时，而一个小时被平均分成3600份，每一份就是1秒。然而，地球的运动周期不是那么稳定，怎么办？多测量几个，平均一下嘛。

虽然通过天体的运动定义了秒这样的基本的时间度量单位，但是，要想精确的表示时间，我们依赖一种有稳定的周期变化的现象。上一节我们说过了：地球围绕太阳运转不是一个稳定的周期现象，因此每次观察到的周期不是固定的（当然都大约是24小时的样子），用它来定义秒多少显得不是那么精准。科学家们发现铯133原子在能量跃迁时候辐射的电磁波的振荡频率非常的稳定（不要问我这是什么原理，我也不知道），因此被用来定义时间的基本单位：秒（或者称之为原子秒）。

2、Epoch

定义了时间单位，等于时间轴上有了刻度，虽然这条代表时间的直线我们不知道从何开始，最终去向何方，我们终归是可以把一个时间点映射到这条直线上了。甚至如果定义了原点，那么我们可以用一个数字（到原点的距离）来表示时间。

如果说定义时间的度量单位是技术活，那么定义时间轴的原点则完全是一个习惯问题。拿出你的手表，上面可以读出2017年5月10，23时17分28秒07毫秒……作为一个地球人，你选择了耶稣诞辰日做原点，讲真，这弱爆了。作为linuxer，你应该拥有这样的一块手表，从这个手表上只能看到一个从当前时间点到linux epoch的秒数和毫秒数。Linux epoch定义为1970-01-01 00:00:00 +0000 (UTC)，后面的这个UTC非常非常重要，我们后面会描述。

除了wall time，linux系统中也需要了解系统自启动以来过去了多少的时间，这时候，我们可以把钟表的epoch调整成系统的启动时间点，这时候获取系统启动时间就很容易了，直接看这块钟表的读数即可。

3、时间调整

记得小的时候，每隔一段时间，老爸的手表总会慢上一分钟左右的时间，也是他总是在7点钟，新闻联播之前等待那校时的最后一响。一听到“刚才最后一响是北京时间7点整”中那最后“滴”的一声，老爸也把自己的手表调整成为7点整。对于linux系统，这个操作类似clock_set接口函数。

类似老爸机械表的时间调整，linux的时间也需要调整，机械表的发条和齿轮结构没有那么精准，计算机的晶振亦然。前面讲了，UTC的计时是基于原子钟的，但是来到Linux内核这个场景，我们难道要为我们的计算机安装一个原子钟来计时吗？当然可以，如果你足够有钱的话。我们一般人的计算机还是基于系统中的本地振荡器来计时的，虽然精度不理想，但是短时间内你也不会有太多的感觉。当然，人们往往是向往更精确的计时（有些场合也需要），因此就有了时间同步的概念（例如NTP（Network Time Protocol））。

所谓时间同步其实就是用一个精准的时间来调整本地的时间，具体的调整方式有两种，一种就是直接设定当前时间值，另外一种是采用了润物细无声的形式，对本地振荡器的输出进行矫正。第一种方法会导致时间轴上的时间会向前或者向后的跳跃，无法保证时间的连续性和单调性。第二种方法是对时间轴缓慢的调整（而不是直接设定），从而保证了连续性和单调性。

4、闰秒（leap second）

通过原子秒延展出来的时间轴就是TAI（International Atomic Time）clock。这块“表”不管日出、日落，机械的按照ce原子定义的那个秒在推进时间。冷冰冰的TAI clock虽然精准，但是对人类而言是不友好的，毕竟人还是生活在这颗蓝色星球上。而那些基于地球自转，公转周期的时间（例如GMT）虽然符合人类习惯，但是又不够精确。在这样的背景下，UTC（Coordinated Universal Time）被提出来了，它是TAI clock的基因（使用原子秒），但是又会适当的调整（leap second），满足人类生产和生活的需要。

OK，至此，我们了解了TAI和UTC两块表的情况，这两块表的发条是一样的，按照同样的时间滴答（tick，精准的根据原子频率定义的那个秒）来推动钟表的秒针的转动，唯一不同的是，UTC clock有一个调节器，在适当的时间，可以把秒针向前或者向后调整一秒。

TAI clock和UTC clock在1972年进行了对准（相差10秒），此后就各自独立运行了。在大部分的时间里，UTC clock跟随TAI clock，除了在适当的时间点，realtime clock会进行leap second的补偿。从1972年到2017年，已经有了27次leap second，因此TAI clock的读数已经比realtime clock（UTC时间）快了37秒。换句话说，TAI和UTC两块表其实可以抽象成一个时间轴，只不过它们之间有一个固定的偏移。在1972年，它们之间的offset是10秒，经过多年的运转，到了2017年，offset累计到37秒，让我静静等待下一个leap second到了的时刻吧。

5、计时范围

有一类特殊的clock称作秒表，启动后开始计时，中间可以暂停，可以恢复。我们可以通过这样的秒表来记录一个人睡眠的时间，当进入睡眠状态的时候，按下start按键开始计时，一旦醒来则按下stop，暂停计时。linux中也有这样的计时工具，用来计算一个进程或者线程的执行时间。

6、时间精度

时间是连续的吗？你眼中的世界是连续的吗？看到窗外清风吹拂的树叶的时候，你感觉每一个树叶的形态都被你捕捉到了。然而，未必，你看急速前进的汽车的轮胎的时候，感觉车轮是倒转的。为什么？其实这仅仅是因为我们的眼睛大约是每秒15～20帧的速度在采样这个世界，你看到的世界是离散的。算了，扯远了，我们姑且认为时间的连续的，但是Linux中的时间记录却不是连续的，我们可以用下面的图片表示：

[![time-sample](http://www.wowotech.net/content/uploadfile/201705/94ff6ac23fe58e532aa5c373cef2430520170517105555.gif "time-sample")](http://www.wowotech.net/content/uploadfile/201705/4e10109636cbf40837f52aa2565816ce20170517105545.gif)

系统在每个tick到来的时候都会更新系统时间（到linux epoch的秒以及纳秒值记录），当然，也有其他场景进行系统时间的更新，这里就不赘述了。因此，对于linux的时间而言，它是一些离散值，是一些时间采样点的值而已。当用户请求时间服务的时候，例如获取当前时间（上图中的红线），那么最近的那个Tick对应的时间采样点值再加上一个当前时间点到上一个tick的delta值就精准的定位了当前时间。不过，有些场合下，时间精度没有那么重要，直接获取上一个tick的时间值也基本是OK的，不需要校准那个delta也能满足需求。而且粗粒度的clock会带来performance的优势。

7、睡觉的时候时间会停止运作吗？

在现实世界提出这个问题会稍显可笑，鲁迅同学有一句名言：时间永是流逝，街市依旧太平。但是对于Linux系统中的clock，这个就有现实的意义了。比如说clock的一个重要的派生功能是创建timer（也就是说timer总是基于一个特定的clock运作）。在一个5秒的timer超期之前，系统先进入了suspend或者关机状态，这时候，5秒时间到达的时候，一般的timer都不会触发，因为底层的clock可能是基于一个free running counter的，在suspend或者关机状态的时候，这个HW counter都不再运作了，你如何期盼它能唤醒系统，来执行timer expired handler？但是用户还是有这方面的实际需求的，最简单的就是关机闹铃。怎么办？这就需要一个特别的clock，能够在suspend或者关机的时候，仍然可以运作，推动timer到期触发。

三、Linux下的各种clock总结

在linux系统中定义了如下的clock id：

> #define CLOCK_REALTIME            0  
> #define CLOCK_MONOTONIC            1  
> #define CLOCK_PROCESS_CPUTIME_ID    2  
> #define CLOCK_THREAD_CPUTIME_ID        3  
> #define CLOCK_MONOTONIC_RAW        4  
> #define CLOCK_REALTIME_COARSE        5  
> #define CLOCK_MONOTONIC_COARSE        6  
> #define CLOCK_BOOTTIME            7  
> #define CLOCK_REALTIME_ALARM        8  
> #define CLOCK_BOOTTIME_ALARM        9  
> #define CLOCK_SGI_CYCLE            10    /* Hardware specific */  
> #define CLOCK_TAI            11

CLOCK_PROCESS_CPUTIME_ID和CLOCK_THREAD_CPUTIME_ID这两个clock是专门用来计算进程或者线程的执行时间的（用于性能剖析），一旦进程（线程）被切换出去，那么该进程（线程）的clock就会停下来。因此，这两种的clock都是per-process或者per-thread的，而其他的clock都是系统级别的。

根据上面一章的各种分类因素，我们可以将其他clock总结整理如下：

|   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|
||leap second？|clock set？|clock tunning？|original point|resolution|active in suspend？|
|realtime|yes|yes|yes|Linux epoch|ns|no|
|monotonic|yes|no|yes|Linux epoch|ns|no|
|monotonic raw|yes|no|no|Linux epoch|ns|no|
|realtime coarse|yes|yes|yes|Linux epoch|tick|no|
|monotonic coarse|yes|no|yes|Linux epoch|tick|no|
|boot time|yes|no|yes|machine start|ns|no|
|realtime alarm|yes|yes|yes|Linux epoch|ns|yes|
|boottime alarm|yes|no|yes|machine start|ns|yes|
|tai|no|no|no|Linux epoch|ns|no|

 _原创文章，转发请注明出处。蜗窝科技_

标签: [clock](http://www.wowotech.net/tag/clock)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Linux DMA Engine framework(3)_dma controller驱动](http://www.wowotech.net/linux_kenrel/dma_controller_driver.html) | [Linux DMA Engine framework(2)_功能介绍及解接口分析](http://www.wowotech.net/linux_kenrel/dma_engine_api.html)»

**评论：**

**Joy**  
2021-09-11 17:07

蜗窝科技这个论坛的内容质量，我觉得应该算是论坛界的天花板了。

[回复](http://www.wowotech.net/timer_subsystem/clock-id-in-linux.html#comment-8309)

**dkfdhrsn**  
2018-12-29 11:16

@linuxer,有个问题请教下，最近发现了个add_timer无法执行的问题，感觉和dynamic tick有关：  
--------------  
1.high res mode下，clockevent的handler是hrtimer_interrupt,里面会执行expires到达的hrtimer_callback  
2.higg_res里面会构建一个特殊的hrtimer:sched_timer来模拟periodic tick，expires会reprogram来模拟  
3.当进入tickless mode时，比如是通过idle进入，里面会停掉periodic tick,当然这里会遍历time wheel来设置最近一次的trigger event时间点，不过这里会设置tick_sched->is_stopped=1，表示stop periodic tick  
4.假设现在已经停掉了periodic tick，最近一次的event也到来了，那么会执行hrtimer_interrupt,一些关键的flow如下：  
hrtimer_interrupt->__hrtimer_run_queues->__run_hrtimer->这里遍历执行hrtimer的callback，以sched_timer为例，那么执行的是tick_sched_timer  
这里面有个判断如下：  
if(ts->tick_stopped) return HRTIMER_NORESTART, 也就是前面提到的因为进入了tickless，这个不需要再次trigger了  
  
但是在hrtimer执行的过程中：  
__run_hrtimer  
    __remove_hrtimer  
    rlt = hrtimer->function()  
    if (rlt != HRTIMER_NORESTART)  
        __enqueue_hrtimer  
这就是说一旦进入了tickless，sched_timer就被从hrtimer_list里面remove掉了，当然再退出idle的时候会再次加进来  
  
但是问题就在于此，假如idle持续一段时间，add_timer这种形式添加的low resolution岂不是无法执行了？  
虽然oneshot还在更新time wheel的trigger时间点，但是add_timer这种是属于SOFTIRQ_TIMER类型的软中断，执行时机是在irq_exit的时候判断对应type的pending状态位，  
而对于high res mode，这个raise_softirq(SOFTIRQ_TIMER)的地方是在如下地方：  
tick_sched_timer->tick_sched_handle->update_process_times->run_local_timers->raise_softirq(SOFTIRQ_TIMER)，  
而在tickless模式，这个sched_timer已经被拿掉了，也就是说没有地方进行raise设置pending了，所以就无法执行了？  
  
不知道上面的逻辑有问题没，因为加log看到的现象差不多就是如此，如果理解没有问题的话，那么这是个bug吗？还是说在high res mode下这种是预期的，现在这边产品是利用add_timer来进行喂狗操作[先不管是否合理]，有时候会出现咬狗复位，感觉就是这个问题导致的  
  
PS:现在论坛注册被关闭了，不知道放在这里大神还能看见不，求解答啊 0.0

[回复](http://www.wowotech.net/timer_subsystem/clock-id-in-linux.html#comment-7116)

**dkfdhrsn**  
2018-12-29 15:33

@dkfdhrsn：又跟了下代码，在irq_exit里面会执行这个流程：  
tick_irq_exit->tick_nohz_irq_exit  
如果还是idle，里面会检测是否有新的timer，然后将sched_timer加进去~  
估计还是要加log看到底跑哪去了，从之前debug的表象来看，timer是add进去，但就是没执行到，不知道哪里流程出问题了

[回复](http://www.wowotech.net/timer_subsystem/clock-id-in-linux.html#comment-7117)

**mouse**  
2017-07-03 17:53

地球自转一圈才是一天，绕太阳是一年。:D

[回复](http://www.wowotech.net/timer_subsystem/clock-id-in-linux.html#comment-5768)

**[linuxer](http://www.wowotech.net/)**  
2017-07-04 09:18

@mouse：多谢指正，哈哈，犯了一个低级错误。

[回复](http://www.wowotech.net/timer_subsystem/clock-id-in-linux.html#comment-5769)

**felix**  
2017-05-25 13:42

刚上网站，看到你的回复，再次对你详尽的回答表示致敬~  
问题1：对于SMP系统，每个cpu上是不是都会有周期性tick，而只有一个CPU负责更新时间和jiffies？  
问题2：负责更新时间的cpu是否能够进入睡眠，甚至C3+state？  
问题3：SMP系统中，所有的CPU在处理时间任务上是否都一样？有没有区别对待？

[回复](http://www.wowotech.net/timer_subsystem/clock-id-in-linux.html#comment-5602)

**[linuxer](http://www.wowotech.net/)**  
2017-05-25 19:13

@felix：问题一：是的  
问题二：可以，当该cpu进入idle之后，其他的cpu会在tick handler中接管global tick的任务。这个和C3STOP无关，有C3STOP的话，需要broadcast tick device来协助处理。  
问题三：我觉得是一样的。

[回复](http://www.wowotech.net/timer_subsystem/clock-id-in-linux.html#comment-5603)

**felix**  
2017-05-25 19:22

@linuxer：Dear linuxer  
今天在看时间子系统的时候，又想到一个问题，一直困惑  
之前您回答了我高精度模式下tickless的工作过程，只会从Timer Wheel中选择最先到期的Timer的到期时间设置clockevent，而不会从红黑树中选择，这个我收获很大；  
现在，我想问的是，在低精度模式下，进入tickless的时候，会从Timer Wheel和红黑树中选择最先到期的Timer的到期时间作为Clockevent的触发时间，这里也为什么也会考虑红黑树呢？

[回复](http://www.wowotech.net/timer_subsystem/clock-id-in-linux.html#comment-5604)

**[linuxer](http://www.wowotech.net/)**  
2017-05-25 19:34

@felix：在高精度模式下，底层运作机制是hrtimer，Timer Wheel是基于hrtimer的。在低精度模式下，hrtimer仍然可以运作，但是精度无法达到ns的精度，和Timer Wheel的精度是一样的，即hrtimer也是频率是HZ的tick来推动的（这时候Timer Wheel和hrtimer是平级的），因此这时候也需要考虑hrtimer的设定情况。

[回复](http://www.wowotech.net/timer_subsystem/clock-id-in-linux.html#comment-5605)

**[schedule](http://www.wowotech.net/)**  
2017-05-22 18:54

tickless就只是考虑周期性tick，对于杂乱的hrtimer的中断不予考虑  
==========  
当然考虑啊，linuxer 已经回答了，合并归一后取最近要到期的timer set到clockevent device中。但是如果系统确实很多时间段接近的timer， idle 系统可能就不会进no hz 状态了， 因为进出一次开销也是比较大的。繁忙的服务器系统很少进no hz状态。看看手机系统，插入usb 后灭屏，cpu 上很少有任务running，这时候基本都进入 C3stop状态，使用broadcast clock。  
  
周期性tick存在的必要性是什么？  
====================================  
启动的时候，hrtimer 没准备好，用 periodic 模式工作（但是，原始支持hrtimer 也可以在启动时跑起来，linux 没这样做）  
  
周期tick到了，给系统某些模块提供一个仲裁的时机， 比如性能剖析， 进程计时，任务调度等等

[回复](http://www.wowotech.net/timer_subsystem/clock-id-in-linux.html#comment-5585)

**felix**  
2017-05-23 07:27

@schedule：感谢你的回答  
对cpu idle框架了解的不多，对cpu在何种情况下会进入idle不是很清楚。这也是引起我对时间子系统理解瓶颈的原因

[回复](http://www.wowotech.net/timer_subsystem/clock-id-in-linux.html#comment-5590)

**wpch**  
2017-05-19 15:04

@felix  
插两句，CPU是因为无事可做才进入idle状态的，既然有hrtimer到期了自然就要唤醒处理，感觉你有点本末倒置了；系统的调度、时间的更新这些都依赖于周期性tick，你看一下tick中的处理应该就理解了

[回复](http://www.wowotech.net/timer_subsystem/clock-id-in-linux.html#comment-5576)

**[linuxer](http://www.wowotech.net/)**  
2017-05-19 15:12

@wpch：多谢wpch网友的回复，技术问题越多人讨论越清晰。  
  
由于工作和家庭的原因，其实我很难做到快速回复的，见谅！felix同学，我非常喜欢你的问题，很有意思，我是这么想的：如果你要自己设计一个OS，那么你是考虑如何设计时间子系统呢？其实是可以完全抛弃tick的概念，让整个系统的计时都是以ns为单位的，timer也都是基于ns的。内核的各个模块以及userspace的posix timer请求（最终也是内核的hrtimer）都最终被统一挂入一个数据结构中（也许是红黑树），在这样的思路指导下，调度模块自己设定自己的hrtimer，来处理各个进程的时间统计和调度，kernel profiling模块设定自己的hrtimer，来完成profile_tick函数中的各种任务。总之，就是各个模块设定自己的hrtimer，没有什么tick的东西，原来在tick中处理的所有的事情可以推回给各个模块，让他们自己不断的设定周期性timer来解决，这样的系统好不好呢？当然好，简洁而美丽。  
  
虽然软件结构好了，但是，这样实现导致的一个问题就是hw timer的中断太多了，一方面，在系统运行状态，很多模块可能都有每10ms需要处理日常性事务的需求（例如timekeeping模块需要更新系统时间、调度模块需要统计进程时间等），如果各自为政，那么各个模块的这种10ms的hrtimer都会各自触发中断（因为初始相位是不一样的），如果有tick的话，那么一次tick，可以统一为这些模块处理这些housekeeping task。另外一方面，实际上timer的需求还是有两种情况：一种是需要精确的时间（例如多媒体应用），另外一种就是简单的超时处理1S之后检查某种设备的状态，但是1.01秒之后检查也没有什么关系。如果对第二种场景也使用hrtimer来应对，那么也会造成过多timer中断，使用tick来推动第二种timer需求会更好一些。  
  
Timer中太多有什么坏处呢？一方面，timer interrupt handler消耗了额外的cpu处理时间，另外，更重要的是会造成processor cache thrashing的问题。这些都会造成性能和功耗方面的问题。

[回复](http://www.wowotech.net/timer_subsystem/clock-id-in-linux.html#comment-5577)

**felix**  
2017-05-23 07:09

@linuxer：感谢linuxer能够在百忙之中关注我的问题，并深入回答。对您这段话的理解如下：  
加入内核中有N个模块，N个模块有个共同的需求或者说规律：10ms需要一次动作（timekeeping的时间更新，进程调度以及某种设备状态的查询），如果没有统一的tick作为准则，这N个模块就会在自己的领域里以10ms间隔更新，这样显得杂乱无章，也不利于内核的稳定和性能，因此设计出周期性tick，定义了一个准则，这N个模块就在这个统一的tick中做自己想要做的事情。  
深深的感觉出您对内核的认识程度之深，顶礼膜拜~~~

[回复](http://www.wowotech.net/timer_subsystem/clock-id-in-linux.html#comment-5587)

**felix**  
2017-05-23 07:09

@felix：假如，刚才手误

[回复](http://www.wowotech.net/timer_subsystem/clock-id-in-linux.html#comment-5588)

**felix**  
2017-05-23 07:33

@wpch：谢谢的回答！！！我需要学习的东西还很多，希望多指教

[回复](http://www.wowotech.net/timer_subsystem/clock-id-in-linux.html#comment-5591)

**felix**  
2017-05-18 12:24

有个问题想请教一下：  
boot time具体是指什么时间到什么时间？系统加电-->boot finished?还是什么？

[回复](http://www.wowotech.net/timer_subsystem/clock-id-in-linux.html#comment-5567)

**[linuxer](http://www.wowotech.net/)**  
2017-05-18 15:16

@felix：我们这里讨论的都是clock，boottime clock不是从A到B的概念，而是一条time line，原点就是系统启动的那个时间点，然后在任何时间点，针对该clock id，调用clock_gettime都可以获取当前时间到原点的距离（即系统启动那个时间点到当前时间点的时间长度）。

[回复](http://www.wowotech.net/timer_subsystem/clock-id-in-linux.html#comment-5568)

**felix**  
2017-05-18 15:29

@linuxer：Got it!!!  
另外想请教你另外一个问题，关于动态时钟机制：  
针对高精度模式：一旦CPU idle就会启动动态始终机制，这时候cpu会停到周期性tick，选择time wheel中最先到期的timer的到期时间作为下次触发周期性tick的时机。问题是：这里为什么不考虑红黑树中高精度timer的最早到期的timer的到期时间呢？按道理不是应该比较time wheel 和 高精度timer，寻找这两种timer下最早到期的timer吗？为什么忽略掉了高精度tiimer?

[回复](http://www.wowotech.net/timer_subsystem/clock-id-in-linux.html#comment-5569)

**[linuxer](http://www.wowotech.net/)**  
2017-05-18 16:49

@felix：你问的应该是dynamic tick的问题吧，动态时钟是dynamic clock。  
  
你是如何得到进入idle，停掉tick的时候只考虑低精度timer，而没有考虑高精度timer这个结论的？

[回复](http://www.wowotech.net/timer_subsystem/clock-id-in-linux.html#comment-5570)

**felix**  
2017-05-18 17:30

@linuxer：发邮件给你了，不知道你能收到不？

[回复](http://www.wowotech.net/timer_subsystem/clock-id-in-linux.html#comment-5571)

**felix**  
2017-05-18 17:31

@linuxer：（1）切换到高精度模式，过程如下：  
                  run_timer_softirq  
                          hrtimer_run_pending  
                                   hrtimer_switch_to_hres  
1>    base->hres_active = 1 这个标志会在进入tickless时，判断是否要查找红黑树中最近到期的hrtimer；  
2>    tick_setup_sched_timer() 这里用一个hrtiemr实现了以一个HZ为间隔的周期性tick，假设当前时间为now，一个HZ的时间间隔为tick_period：a)初始，该hrtimer会以now+tick_period为到期时间，并挂到红黑树中；b)系统一旦检测到该hrtimer到期，就会将其从红黑树中删除，调用该hrtimer的回调函数tick_sched_timer（该回调函数主要完成jiffies的更新、时间的更新等，并且将该hrtiemr的下次触发时间设置为now+tick_period），之后再次将该hrtiemr挂到红黑树中；重复b)，模拟出一个以HZ为间隔的周期性tick。  
（2）进入tickless，过程如下：  
                  cpu_idle_loop  
                          tick_nohz_idle_enter  
                                   __tick_nohz_idle_enter  
                                            tick_nohz_stop_sched_tick  
1>    get_next_timer_interrupt，低精度模式下，该函数实现了从高精度timer和低精度timer中寻找最近到期的timer;但在高精度模式下，只在低精度timer中寻找最近到期的timer，过程如下：  
a>__next_timer_interrupt，该函数实现从time wheel中找到最近到期的低精度timer  
b>cmp_next_hrtimer_event  
hrtimer_get_next_event，这里会根据hrtimer_hres_active()判断系统是否处于高精度模式，如果系统没有处于高精度模式，则从红黑树中寻找最近到期的timer，之后再与最近到期的低精度timer比较，哪个最早到期；但是目前系统处于高精度模式（标志base->hres_active = 1），就不会从红黑树中寻最近到期的高精度timer。  
2>    hrtimer_start，这里将1>找到的最近到期timer的到期时间设置到（1）模拟出周期tick的hrtimer上，用于停止周期性tick。  
问题：高精度模式下，进入tickless之后，只从time wheel中寻找最早到期的低精度timer，没有去寻找最近到期的高精度timer，这里是为什么？最近到期的高精度timer如果早于最近到期的低精度timer，该怎么处理？

[回复](http://www.wowotech.net/timer_subsystem/clock-id-in-linux.html#comment-5572)

**[linuxer](http://www.wowotech.net/)**  
2017-05-19 10:47

@felix：多谢你的解释，我也按照你的思路走读了一遍代码，我是这样考虑的：hrtimer本身是不依赖于tick的，因此它的逻辑很简单，hrtimer到期后执行callback，然后就是从红黑树中摘下最近的那个hrtimer，设定下一次触发的时间点，如此周而复始。因此，hw timer的中断触发时间是和红黑树中的hrtimer有关，如果红黑树是空的，那么hw timer就静默了。  
  
当然，实际情况没有那么简单，因为目前内核运行了高精度和低精度timer两套系统，而低精度timer是基于tick的（具体为何如此，这说来话长，网上资料也很多，这里就不赘述了）。在这样的情况下，内核用一个hrtiemr实现了以一个HZ为间隔的周期性tick，而这个tick又推动了低精度timer的运作。  
  
OK，说道tickless，其实就是停掉没有必要的tick，实际上只有低精度timer是依赖于tick的，因此也就不难理解为何在tick_nohz_stop_sched_tick函数中，只从time wheel中寻找最早到期的低精度timer来决定是否stop模拟tick的那个hrtimer。  
  
假设time wheel中寻找最早到期的低精度timer是10秒，那么模拟tick的那个sched_timer就会被停止运作10秒，当然，在10秒内可能有非常多的hr timer在红黑树中，他们怎么办？没有什么怎么办，该触发的还是触发，hrtimer仍然按照其既定的规律运作，hw timer中断仍然按照hrtimer模块设定的规律触发。停掉tick并不是意味着停到hw timer的interrupt。

[回复](http://www.wowotech.net/timer_subsystem/clock-id-in-linux.html#comment-5573)

**felix**  
2017-05-19 11:17

@linuxer：十分感谢你的回复，昨天到现在一直等待你的回复，终于等到了！！  
低精度timer依赖周期性tick，而hrtimer不依赖周期性tick，这个对我帮助很大！！  
另外，对这段话：<<假设time wheel中寻找最早到期的低精度timer是10秒，那么模拟tick的那个sched_timer就会被停止运作10秒，当然，在10秒内可能有非常多的hr timer在红黑树中，他们怎么办？没有什么怎么办，该触发的还是触发，hrtimer仍然按照其既定的规律运作，hw timer中断仍然按照hrtimer模块设定的规律触发。停掉tick并不是意味着停到hw timer的interrupt。>>  
我还存在疑问，tickless的初衷是为了节能的考虑，所以停止没有必要的周期性tick。如果10s内有很多hrtimer不断的打断cpu睡眠，这个就和系统节能的本意不符合了。tickless就只是考虑周期性tick，对于杂乱的hrtimer的中断不予考虑？另外，想请教您一点，周期性tick存在的必要性是什么？

**felix**  
2017-05-23 07:22

@linuxer：Dear linuxer，  
还是对这段还有一些疑问，还请您多指教  
《假设time wheel中寻找最早到期的低精度timer是10秒，那么模拟tick的那个sched_timer就会被停止运作10秒，当然，在10秒内可能有非常多的hr timer在红黑树中，他们怎么办？没有什么怎么办，该触发的还是触发，hrtimer仍然按照其既定的规律运作，hw timer中断仍然按照hrtimer模块设定的规律触发。停掉tick并不是意味着停到hw timer的interrupt。》  
cpu进入idle状态后（未进入C3+state），会将Time Wheel中最先到期的timer的到期时间设定为sched_timer的到期时间，假设是10s，在这10s中可能有很多hrtimer在红黑树中，这些hrtimer到期后均会触发cpu处理，那这时候，cpu退出了idle，处理了hrtimer的中断，在中断退出的时候再次进入tickless，这样理解是否准确？

**[linuxer](http://www.wowotech.net/)**  
2017-05-23 16:58

@linuxer：假设目前cpu上的hrtimer的情况如下（按时间顺序）：  
1、hrtimer A （2ms）  
2、hrtimer B （5ms ）  
3、sched hrtimer （10ms ）－－－假设HZ=100  
4、hrtimer C （100ms ）  
........  
  
当CPU上没有任务执行，进入idle loop的时候，会根据time wheel中的情况来更新sched hrtimer超期时间，假设是7S，因此，这时候，cpu上的hrtimer的情况如下（按时间顺序）：  
1、hrtimer A （2ms）  
2、hrtimer B （5ms ）  
3、hrtimer C （100ms ）  
........  
x、sched hrtimer （7s ）  
........  
  
2ms后，hrtimer A到期触发，唤醒CPU，这时候有两种情况：其一是这个hrtimer A没有唤醒任何进程，这时候，系统保持tickless状态，cpu上的hrtimer的情况如下（按时间顺序）：  
1、hrtimer B （5ms ）  
2、hrtimer C （100ms ）  
........  
x、sched hrtimer （7s ）  
........  
5ms后，hrtimer B触发，这时候我们假设是情况二、该hrtimer的handler中唤醒了某个进程P，从而使得该cpu退出tickless状态（tick_nohz_idle_exit），重置sched timer的超期时间为10ms，这时候，cpu上的hrtimer的情况如下（按时间顺序）：  
1、sched hrtimer （10ms ）  
2、hrtimer C （100ms ）  
........  
  
也许进程P执行非常快，1ms之后系统又重回idle进程，这时候会调用tick_nohz_idle_enter函数，sched hrtimer的行为同上。

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
    
    - [DRAM 原理 1 ：DRAM Storage Cell](http://www.wowotech.net/basic_tech/307.html)
    - [Debian8 内核升级实验](http://www.wowotech.net/linux_application/debian8-upgrade-kernel.html)
    - [mellanox的网卡故障分析](http://www.wowotech.net/linux_kenrel/485.html)
    - [perfbook memory barrier（14.2章节）的中文翻译（上）](http://www.wowotech.net/kernel_synchronization/memory-barrier-1.html)
    - [u-boot启动流程分析(2)_板级(board)部分](http://www.wowotech.net/u-boot/boot_flow_2.html)
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
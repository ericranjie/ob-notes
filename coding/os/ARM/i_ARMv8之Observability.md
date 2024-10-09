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

作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2016-5-25 18:22 分类：[ARMv8A Arch](http://www.wowotech.net/sort/armv8a_arch)

一、前言

在ARMv8关于memory order描述章节中，大量使用了observer、observed、completion等术语，本文主要是澄清这些术语，为后续描述memory order和memory barrier相关指令打下基础。另外，在几个星期前，和codingbelief同学讨论DMB指令的时候，他提出了一个尖锐的问题：什么是PE observes memory access，是指 cpu 执行了 memory access 指令么？当时我对这些概念也比较模糊，未能回答他的疑问，现在希望这份文档可以解决这个问题。

二、observer

原文定义：

> An observer is a master in the system that is capable of observing memory accesses

从字面的意思看，能够“观察”到内存访问行为的master角色的硬件组件就是observer。这里需要解释两个关键词：一个是master，另外一个是观察（observing）。一个observer首先需要是一个总线master，可以通过总线发出读写操作。而所谓“观察”，实际上也通过read或者write操作来感知其他的内存操作，例如：通过read操作，observer可以感知到内存值的数据变化。当然，observer不仅仅能够读，还可以执行写操作，一个PE中可能的observer包括：

（1）执行load或者store操作的CPU组件（能够对内存发起读或者写操作）

（2）执行取指操作的CPU组件（仅仅能够发起对内存的read操作）

（3）加载页表的CPU组件（仅仅能够发起对内存的read操作）

从上面的描述可知，并非系统中有多少个CPU core（或者称为PE）就有多少个observer，一个PE往往包括多个observer。实际上，系统中的observer的数目是和具体的实现相关，除了PE之外，GPU，DMA controller等都可以发起对memory的read或者write操作，也都是observer。

三、什么是Observability？

可以用下面通俗的语言来描述Observability：

> I have observed your write when I can read what you wrote and I have observed your read when I can no longer change the value you read

这里的I和you都是指的系统中的observer。上面的这句话是针对两个场景：一个是对写动作的观察，I have observed your write when I can read what you wrote，当A observer通过对X地址的读操作获取了另外B observer对X地址的写入数值的时候，那么，可以认为A observer观察到了B observer的写入动作。另外一个场景是对读动作的观察，I have observed your read when I can no longer change the value you read，当A observer无法通过对地址X的值的写入操作来影响B observer的对X地址的read操作结果的时候，我们认为A observer观察到了B observer的read动作。

四、observed write

原文定义：

> A write to a location in memory is said to be observed by an observer when:\
> — A subsequent read of the location by the same observer returns the value written by the observed write, or written by a write to that location by any observer that is sequenced in the Coherence order of the location after the observed write.\
> — A subsequent write of the location by the same observer is sequenced in the Coherence order of the location after the observed write.

什么叫一次写入的内存操作被某个observer观察到（observed）？这事不能从单个写入操作看，还是让我们拉高一些，从一个写入序列来看。我们还是通过一个经典的例子来说明observed write这个概念。假设系统中有四个cpu core，分别执行同样的代码：cpux给一个全局变量A赋值为x，然后不断对A进行观察（即load操作）。在这个例子中A分别被四个CPU设定了1、 2、3、4的值，当然，先赋值的操作结果会被后来赋值操作覆盖，最后那个执行的write操作则决定了A变量最后的赋值。假设一次运行后，cpu 1看到的序列是{1,2}，cpu 2看到的序列是{2}，cpu 3看到的序列是{3,2}，cpu 4看到的序列是{4,2}，具体如下图所示：

![](http://www.wowotech.net/content/uploadfile/201512/133813bd99a9d17a3c4ecd41982a305420151225041816.gif)

我们先选定一个观察对象：cpu4的写入操作。怎么观察呢？通过read或者write来观察（英文原文的两段解释就是分别针对read和write的）。我们先看read吧，这是最直观的感受（或者称为观察）cpu4的写入动作的方法。有两种情况都可以认为本observer已经观察到了cpu4的写入操作：

（1）该observer的read操作返回了“4”。例如在cpu4上，read返回了之前write的数值“4”

（2）该observer的read操作没有返回“4”，但是返回了Coherence order 排在4之后的数值。例如，对于cpu1而言，刚开始，其read操作返回“1”（红色条带），大概在300ns的时间左右，read操作返回了“2”，我们知道，对于全局变量A，其coherent order是{3,1,4,2}，只要返回了coherent order序列中，4之后的数值，我们就认为该observer已经观察到了cpu4的对全局变量A写入数值4的操作。

通过write来“观察”另外一个write操作有点不是那么符合人类的逻辑思维，但是可以慢慢适应的，毕竟我们的目标是理解multiprocessing environment。对于write而言，只有一种情况认为本observer已经观察到了cpu4的写入操作：即做为观察者的那个observer的写入动作在coherent order序列中排在“4”之后。对于cpu 2而言，当执行写入2的操作的时候，我们认为，cpu 2已经观察到了cpu4的写入操作。

需要强调的是，本节描述的observed write都是以一个特定的observer为视角的，不是全局的概念。

五、globally observed write

原文定义：

> A write to a location in memory is said to be globally observed for a shareability domain or set of observers when:\
> — A subsequent read of the location by any observer in that shareability domain returns the value written by the globally observed write, or written by a write to that location by any observer that is sequenced in the Coherence order of the location after the globally observed write.\
> — A subsequent write of the location by any observer in that shareability domain is sequenced in the\
> Coherence order of the location after the globally observed write.

理解了上一节的内容，这里几乎不需要过多的解释，只需要强调两点：

（1）globally observed write是限定在一个shareability domain内部，或者指定的一个observer的集合。

（2）当一个shareability domain（observer集合）内所有的observer都观察到了一次write操作，那么就是globally observed write。

六、observed read和globally observed read

原文定义：

> • A read of a location in memory is said to be observed by an observer when a subsequent write to the location by the same observer has no effect on the value returned by the read.\
> • A read of a location in memory is said to be globally observed for a shareability domain when a subsequent write to the location by any observer in that shareability domain has no effect on the value returned by the read.

怎么观察（感知）其他observer的读操作的确是一个技术活，通过read毫无疑问是不可能感知的，唯有通过write来感知。当某个observer无法通过write操作来影响被观察者的read操作的时候，我们就认为该observer已经感知到了该read操作。类似的，globally observed read就是一个shareability domain内所有的observer们都观察到了该read操作。

七、内存访问指令的执行完成（completion）

原文定义：

> A read or write is complete for a shareability domain when all of the following are true:\
> — The read or write is globally observed for that shareability domain.\
> — Any translation table walks associated with the read or write are complete for that shareability domain.

内存访问指令被观察到和内存访问指令执行完毕是两个不同的概念，对于内存访问指令执行完毕，需要满足下面的两个条件：

（1）该内存访问操作被特定的shareability domain内的所有的observer globally observed

（2）和该内存访问指令相关的translation table walks（也会引发内存访问操作）必须执行完毕。

这里又牵扯出另外一个问题：什么叫做translation table walks执行完毕？原文定义如下：

> A translation table walk is complete for a shareability domain when the memory accesses associated with the translation table walk are globally observed for that shareability domain, and the TLB is updated.

需要满足两个条件：

（1）这个translation table walks而引起的内存访问操作被该shareability domain内的所有的observer globally observed

（2）TLB已经完成更新

八、参考文献

1、Computer Architecture, A Quantitative Approach 5th

2、Programmer’s Guide for ARMv8-A

原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/)。

标签: [ARMv8](http://www.wowotech.net/tag/ARMv8) [observer](http://www.wowotech.net/tag/observer) [observed](http://www.wowotech.net/tag/observed)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [蓝牙协议分析(5)\_BLE广播通信相关的技术分析](http://www.wowotech.net/bluetooth/ble_broadcast.html) | [u-boot启动流程分析(1)\_平台相关部分](http://www.wowotech.net/u-boot/boot_flow_1.html)»

**评论：**

**xiejin**\
2020-12-17 09:30

我做过ARMv8 core RTL design，谈一下我的理解吧。\
其实在真实的硬件里面，observe 这个动作，远远不止load/store 这些cpu 指令。\
基本上core 和 core 之间相互observe，是通过snoop 协议相互snoop，最终是在interconnect 上完成排序，来达到globally observed 的目的。

[回复](http://www.wowotech.net/armv8a_arch/Observability.html#comment-8159)

**z13**\
2020-01-10 11:40

不是观察的意思，也不只是read和write动作来感知的意思，而是一系列操作过程中，每一个操作的边界互不重叠，使得这些操作顺序的唯一性，得到的结果也是唯一的，无论何时何地。

[回复](http://www.wowotech.net/armv8a_arch/Observability.html#comment-7828)

**oldlinux**\
2018-09-17 12:02

您好，读了您的文章，感觉非常的有用，但是因为理解一部分，但是当我在看4个CPU赋值的时候有点不明白，\
CPU1 |                         1                         |                            2                   |\
这个1表示CPU1往memory写值么，如果是写值需要等待一会才会把1写入A变量中么？\
那么CPU1和CPU2同时赋值的时候总线不是冲突的么？\
而，“对于全局变量A，其coherent order是{3,1,4,2}”中A的序列3，1，4，2是我们在外部观察到A值变化的先后顺序么？

[回复](http://www.wowotech.net/armv8a_arch/Observability.html#comment-6958)

**Codingbelief**\
2016-05-31 02:47

所以我理解的 PE observes memory access，是 PE 执行 memory access。而 observed，则是指执行的 memory access 可以认定为完成。

[回复](http://www.wowotech.net/armv8a_arch/Observability.html#comment-4005)

**[linuxer](http://www.wowotech.net/)**\
2016-05-31 11:55

@Codingbelief：“PE observes memory access”这句话，我的理解是observes是一个动词，表示PE正在“感知”来自自己或者其他PE的memory access的操作结果。

observed是形容词，用来描述一个memory指令的执行结果是否被感知到。指令完成和指令被感知到是不同的概念，如果一条内存访问的指令已经完成，那么：\
1、该memory access被系统中所有的observer感知到（就是globally observed）\
2、和该内存访问指令相关的translation table walks（也会引发内存访问操作）必须执行完毕。\
因此observed和内存指令执行完成也不是同一个概念。

[回复](http://www.wowotech.net/armv8a_arch/Observability.html#comment-4009)

**Codingbelief**\
2016-05-31 02:40

个人感觉，observer 刚开始接触时，让人费解的一个原因是把它翻译为“观察者”。

在中文上，观察有着“看”的操作，进而会让人联想到"读"，再而让人联想到 observers 之间通过“读”感知到对方的动作，然后后面的一些 observed read write 解释起来就变得很别扭。

我觉得，在上面 coherent order {3,1,4,2} 的例子中，每一个 observer 实际上是只能够确定自己的 write 和 read 的发生的时间、结束的时间以及执行的结果，并不能具体的”感知"到其他 observer 所进行的操作，observers 之间也没有进行"观察"对方的动作。

当然，并不是说 observer 翻译成“观察者”是错误的，只是翻译成“观察者”会在前期理解的时候比较费力。

换个角度，如果把 observer 翻译为“执行者”，可能会省力一些。

observer 是“执行者”，它拥有“执行”读或者写内存的能力。那么对于 observed write 和 observed read 的可以解释如下：

observed write 描述了这样一个事实： “执行者” 完成了特定的 write memory 的操作的执行。

在上面经典例子的图中，我们可以很清楚的看到，在时间坐标轴上 500ns 时，CPU4 完成了 write 4 的操作。然而，在 PE 中，我们并没有这个关键的时间轴，这时候，我们就需要找其他的方法，来肯定"CPU4 完成了 write 4 的操作”这一事实。在 ARMv8 中，确定 write 操作已经完成执行(即 observed write)的方法是通过以下的两个事件：

- CPU4 对同样的内存地址执行的下一个读操作，读回了刚刚写入的值，或者读回  coherent order 4 之后的值。
- CPU4 对同样的内存地址执行下一个写操作，该写操作写入的值在 coherent order 4 之后。

也就是说，当发生了上面的两个事件中的一个时，我们就可以认为 CPU4 的 write 4 操作已经执行完成，即 observed。

observed read 则描述了这样一个事实： “执行者” 完成了特定的  read memory 的操作的执行。

同样在上面经典例子的图中，在时间坐标轴上 500ns 后，CPU4 执行了 read 的操作，并且返回了 2。什么时候我们可以说 CPU4 已经完成了 read 2 的执行呢？在 ARMv8 中，如下定义：

- 如果 CPU4 对同样内存地址执行了下一个 write 操作，并且该 write 操作没有影响 read 操作返回的值。

上面的条件一旦达成，就意味着 CPU4 读回 2 已经是铁板钉钉的事实了，就可以说 CPU4  read  2 已经完成，即 observed read。

observed 给我的感觉更多的是一个“边界”的概念，用于定义某一时刻某个执行者是否已经完成了特定的操作。

[回复](http://www.wowotech.net/armv8a_arch/Observability.html#comment-4004)

**[linuxer](http://www.wowotech.net/)**\
2016-05-31 12:05

@Codingbelief：看我之前写的文档，我和你一样被“忽悠”了，以为PE都是通过read来“观察”内存访问指令的结果。当然，具体的翻译而言，我觉得执行者更容易误导啊，因为很容易联想成“指令执行者”

另外，你上面这段话我不是很认同，基本上，CPU 4上的对某个共享遍历的读或者写操作是否完成不能从CPU 4的视角来判断，应该是从系统的角度来看，即其他的CPU是如何看待CPU4 上的读写操作的。

[回复](http://www.wowotech.net/armv8a_arch/Observability.html#comment-4010)

**[linuxer](http://www.wowotech.net/)**\
2016-05-31 12:05

@Codingbelief：遍历--->变量

[回复](http://www.wowotech.net/armv8a_arch/Observability.html#comment-4011)

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

  - [DRAM 原理 4 ：DRAM Timing](http://www.wowotech.net/basic_tech/330.html)
  - [SLUB DEBUG原理](http://www.wowotech.net/memory_management/427.html)
  - [支持wowo，有这样一块小天地享受技术，感觉棒棒哒！](http://www.wowotech.net/147.html)
  - [linux kernel的中断子系统之（三）：IRQ number和中断描述符](http://www.wowotech.net/irq_subsystem/interrupt_descriptor.html)
  - [Linux Regulator Framework(2)\_regulator driver](http://www.wowotech.net/pm_subsystem/regulator_driver.html)

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

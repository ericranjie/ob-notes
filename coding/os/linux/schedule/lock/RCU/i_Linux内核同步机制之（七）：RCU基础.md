作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2015-12-3 12:57 分类：[内核同步机制](http://www.wowotech.net/sort/kernel_synchronization)

# 一、前言

关于RCU的文档包括两份，一份讲基本的原理（也就是本文了），一份讲linux kernel中的实现。第二章描述了为何有RCU这种同步机制，特别是在cpu core数目不断递增的今天，一个性能更好的同步机制是如何解决问题的，当然，再好的工具都有其适用场景，本章也给出了RCU的一些应用限制。第三章的第一小节描述了RCU的设计概念，其实RCU的设计概念比较简单，比较容易理解，比较困难的是产品级别的RCU实现，我们会在下一篇文档中描述。第三章的第二小节描述了RCU的相关操作，其实就是对应到了RCU的外部接口API上来。最后一章是参考文献，perfbook是一本神奇的数，喜欢并行编程的同学绝对不能错过的一本书，强烈推荐。和perfbook比起来，本文显得非常的丑陋（主要是有些RCU的知识还是理解不深刻，可能需要再仔细看看linux kernel中的实现才能了解其真正含义），除了是中文表述之外，没有任何的优点，英语比较好的同学可以直接参考该书。

# 二、为何有RCU这种同步机制呢？

前面我们讲了[spin lock](http://www.wowotech.net/kernel_synchronization/spinlock.html)，[rw spin lock](http://www.wowotech.net/kernel_synchronization/rw-spinlock.html)和[seq lock](http://www.wowotech.net/kernel_synchronization/seqlock.html)，为何又出现了RCU这样的同步机制呢？这个问题类似于问：有了刀枪剑戟这样的工具，为何会出现流星锤这样的兵器呢？每种兵器都有自己的适用场合，内核同步机制亦然。RCU在一定的应用场景下，解决了过去同步机制的问题，这也是它之所以存在的基石。本章主要包括两部分内容：一部分是如何解决其他内核机制的问题，另外一部分是受限的场景为何？

## 1、性能问题

我们先回忆一下spin lcok、RW spin lcok和seq lock的基本原理。对于spin lock而言，临界区的保护是通过next和owner这两个共享变量进行的。线程调用spin_lock进入临界区，这里包括了三个动作：

（1）获取了自己的号码牌（也就是next值）和允许哪一个号码牌进入临界区（owner）
（2）设定下一个进入临界区的号码牌（next++）
（3）判断自己的号码牌是否是允许进入的那个号码牌（next == owner），如果是，进入临界区，否者spin（不断的获取owner的值，判断是否等于自己的号码牌，对于ARM64处理器而言，可以使用WFE来降低功耗）。

注意：（1）是取值，（2）是更新并写回，因此（1）和（2）必须是原子操作，中间不能插入任何的操作。

线程调用spin_unlock离开临界区，执行owner++，表示下一个线程可以进入。

RW spin lcok和seq lock都类似spin lock，它们都是基于一个memory中的共享变量（对该变量的访问是原子的）。我们假设系统架构如下：
!\[\[Pasted image 20241008205719.png\]\]

当线程在多个cpu上争抢进入临界区的时候，都会操作那个在多个cpu之间共享的数据lock（玫瑰色的block）。cpu 0操作了lock，为了数据的一致性，cpu 0的操作会导致其他cpu的L1中的lock变成无效，在随后的来自其他cpu对lock的访问会导致L1 cache miss（更准确的说是communication cache miss），必须从下一个level的cache中获取，同样的，其他cpu的L1 cache中的lock也被设定为invalid，从而引起下一次其他cpu上的communication cache miss。

RCU的read side不需要访问这样的“共享数据”，从而极大的提升了reader侧的性能。

## 2、reader和writer可以并发执行

spin lock是互斥的，任何时候只有一个thread（reader or writer）进入临界区，rw spin lock要好一些，允许多个reader并发执行，提高了性能。不过，reader和updater不能并发执行，RCU解除了这些限制，允许一个updater（不能多个updater进入临界区，这可以通过spinlock来保证）和多个reader并发执行。我们可以比较一下rw spin lock和RCU，参考下图：

[![rw-rcu](http://www.wowotech.net/content/uploadfile/201512/581bf648cf35429a18b43bab7070651a20151203045709.gif "rw-rcu")](http://www.wowotech.net/content/uploadfile/201512/8180fbe55bc857b2d0208565308c932420151203045708.gif)

rwlock允许多个reader并发，因此，在上图中，三个rwlock reader愉快的并行执行。当rwlock writer试图进入的时候（红色虚线），只能spin，直到所有的reader退出临界区。一旦有rwlock writer在临界区，任何的reader都不能进入，直到writer完成数据更新，立刻临界区。绿色的reader thread们又可以进行愉快玩耍了。rwlock的一个特点就是确定性，白色的reader一定是读取的是old data，而绿色的reader一定获取的是writer更新之后的new data。RCU和传统的锁机制不同，当RCU updater进入临界区的时候，即便是有reader在也无所谓，它可以长驱直入，不需要spin。同样的，即便有一个updater正在临界区里面工作，这并不能阻挡RCU reader的步伐。由此可见，RCU的并发性能要好于rwlock，特别如果考虑cpu的数目比较多的情况，那些处于spin状态的cpu在无谓的消耗，多么可惜，随着cpu的数目增加，rwlock性能不断的下降。RCU reader和updater由于可以并发执行，因此这时候的被保护的数据有两份，一份是旧的，一份是新的，对于白色的RCU reader，其读取的数据可能是旧的，也可能是新的，和数据访问的timing相关，当然，当RCU update完成更新之后，新启动的RCU reader（绿色block）读取的一定是新的数据。

## 3、适用的场景

我们前面说过，每种锁都有自己的适用的场景：spin lock不区分reader和writer，对于那些读写强度不对称的是不适合的，RW spin lcok和seq lock解决了这个问题，不过seq lock倾向writer，而RW spin lock更照顾reader。看起来一切都已经很完美了，但是，随着计算机硬件技术的发展，CPU的运算速度越来越快，相比之下，存储器件的速度发展较为滞后。在这种背景下，获取基于counter（需要访问存储器件）的锁（例如spin lock，rwlock）的机制开销比较大。而且，目前的趋势是：CPU和存储器件之间的速度差别在逐渐扩大。因此，那些基于一个multi-processor之间的共享的counter的锁机制已经不能满足性能的需求，在这种情况下，RCU机制应运而生（当然，更准确的说RCU一种内核同步机制，但不是一种lock，本质上它是lock-free的），它克服了其他锁机制的缺点，但是，甘蔗没有两头甜，RCU的使用场景比较受限，主要适用于下面的场景：

（1）RCU只能保护动态分配的数据结构，并且必须是通过指针访问该数据结构
（2）受RCU保护的临界区内不能sleep（SRCU不是本文的内容）
（3）读写不对称，对writer的性能没有特别要求，但是reader性能要求极高。
（4）reader端对新旧数据不敏感。

# 三、RCU的基本思路

## 1、原理

RCU的基本思路可以通过下面的图片体现：

[![rcu](http://www.wowotech.net/content/uploadfile/201512/76efb6df52d164f2e5f49e121a61e57420151203045710.gif "rcu")](http://www.wowotech.net/content/uploadfile/201512/015011aa8857e3c6b7a37825ec01a6cf20151203045710.gif)

RCU涉及的数据有两种，一个是指向要保护数据的指针，我们称之RCU protected pointer。另外一个是通过指针访问的共享数据，我们称之RCU protected data，当然，这个数据必须是动态分配的  。对共享数据的访问有两种，一种是writer，即对数据要进行更新，另外一种是reader。如果在有reader在临界区内进行数据访问，对于传统的，基于锁的同步机制而言，reader会阻止writer进入（例如spin lock和rw spin lock。seqlock不会这样，因此本质上seqlock也是lock-free的），因为在有reader访问共享数据的情况下，write直接修改data会破坏掉共享数据。怎么办呢？当然是移除了reader对共享数据的访问之后，再让writer进入了（writer稍显悲剧）。对于RCU而言，其原理是类似的，为了能够让writer进入，必须首先移除reader对共享数据的访问，怎么移除呢？创建一个新的copy是一个不错的选择。因此RCU writer的动作分成了两步：

（1）removal。write分配一个new version的共享数据进行数据更新，更新完毕后将RCU protected pointer指向新版本的数据。一旦把RCU protected pointer指向的新的数据，也就意味着将其推向前台，公布与众（reader都是通过pointer访问数据的）。通过这样的操作，原来read 0、1、2对共享数据的reference被移除了（对于新版本的受RCU保护的数据而言），它们都是在旧版本的RCU protected data上进行数据访问。

（2）reclamation。共享数据不能有两个版本，因此一定要在适当的时机去回收旧版本的数据。当然，不能太着急，不能reader线程还访问着old version的数据的时候就强行回收，这样会让reader crash的。reclamation必须发生在所有的访问旧版本数据的那些reader离开临界区之后再回收，而这段等待的时间被称为grace period。

顺便说明一下，reclamation并不需要等待read3和4，因为write端的为RCU protected pointer赋值的语句是原子的，乱入的reader线程要么看到的是旧的数据，要么是新的数据。对于read3和4，它们访问的是新的共享数据，因此不会reference旧的数据，因此reclamation不需要等待read3和4离开临界区。

## 2、基本RCU操作

对于reader，RCU的操作包括：

（1）rcu_read_lock，用来标识RCU read side临界区的开始。
（2）rcu_dereference，该接口用来获取RCU protected pointer。reader要访问RCU保护的共享数据，当然要获取RCU protected pointer，然后通过该指针进行dereference的操作。
（3）rcu_read_unlock，用来标识reader离开RCU read side临界区

对于writer，RCU的操作包括：

（1）rcu_assign_pointer。该接口被writer用来进行removal的操作，在witer完成新版本数据分配和更新之后，调用这个接口可以让RCU protected pointer指向RCU protected data。
（2）synchronize_rcu。writer端的操作可以是同步的，也就是说，完成更新操作之后，可以调用该接口函数等待所有在旧版本数据上的reader线程离开临界区，一旦从该函数返回，说明旧的共享数据没有任何引用了，可以直接进行reclaimation的操作。
（3）call_rcu。当然，某些情况下（例如在softirq context中），writer无法阻塞，这时候可以调用call_rcu接口函数，该函数仅仅是注册了callback就直接返回了，在适当的时机会调用callback函数，完成reclaimation的操作。这样的场景其实是分开removal和reclaimation的操作在两个不同的线程中：updater和reclaimer。

# 四、参考文档

1、perfbook
2、linux-4.1.10\\Documentation\\RCU\*

_原创文章，转发请注明出处。蜗窝科技_

标签: [RCU](http://www.wowotech.net/tag/RCU)

______________________________________________________________________

« [Why Memory Barriers？中文翻译（上）](http://www.wowotech.net/kernel_synchronization/Why-Memory-Barriers.html) | [显示技术介绍(2)\_电子显示的前世今生](http://www.wowotech.net/display/display_tech_intro.html)»

**评论：**

**luke**\
2019-08-20 17:44

@linuxer\
想请教一下， 为什么多个写者之间需要同步？\
假设有A B 两个写者， 原数据块指针为P， A B应该都是基于P copy 做update,\
rcu_assign_pointer(p, v)能否理解成是个原子操作？\
假设A首先做了rcu_assign_pointer(P, PA)\
B再做rcu_assign_pointer(P, PB) 的时候替换的应该就是B的指针了吧\
有可能A, B两者都得到原来P的指针， 而导致double free吗？

[回复](http://www.wowotech.net/kernel_synchronization/rcu_fundamentals.html#comment-7601)

**[linuxer](http://www.wowotech.net/)**\
2022-02-23 09:03

@luke：假设RCU保护的是一个链表：\
1、A线程向链表插入元素X\
2、B线程向链表插入元素Y\
如果不使用类似spinlock的同步机制，那么上面的两个操作必然会丢掉一个，从而引起逻辑问题。

[回复](http://www.wowotech.net/kernel_synchronization/rcu_fundamentals.html#comment-8562)

**orangeboyye**\
2022-04-29 00:29

@luke：两个写者不能同时执行，因为一个写者要基于另一个写者的结果去操作才行，否则就会丢失一个写者的操作。\
假设RCU保护的是int * p = &i，i=1，两个写者的操作都是要i++，按照你的逻辑操作，最后i==2，和预期的i==3不符，丢失了一个写者的操作。

[回复](http://www.wowotech.net/kernel_synchronization/rcu_fundamentals.html#comment-8607)

**[linuxer](http://www.wowotech.net/)**\
2022-04-29 06:14

@orangeboyye：这位同学，有没有兴趣来OPPO搞内核优化，O(∩_∩)O\
要不要加个微信详聊一下，哈哈。

[回复](http://www.wowotech.net/kernel_synchronization/rcu_fundamentals.html#comment-8608)

**orangeboyye**\
2022-04-29 21:32

@linuxer：非常感谢，之前加过你的微信，微信聊。

[回复](http://www.wowotech.net/kernel_synchronization/rcu_fundamentals.html#comment-8610)

**Linuxkernel**\
2018-06-27 15:28

有没有写RCU gpnum 方面的文章？

[回复](http://www.wowotech.net/kernel_synchronization/rcu_fundamentals.html#comment-6821)

**georgeguo**\
2018-05-03 23:24

"因为write端的为RCU protected pointer赋值的语句是原子的"\
您是说\
#define rcu_assign_pointer(p, v)        ({ \\
smp_wmb(); \\
(p) = (v); \\
})\
这里p=v是原子操作吗？ 为什么？

[回复](http://www.wowotech.net/kernel_synchronization/rcu_fundamentals.html#comment-6730)

**[linuxer](http://www.wowotech.net/)**\
2018-05-04 09:51

@georgeguo：每个CPU都有自己关于atomic的详细描述。对于ARM体系结构，你可以参考ARM ARM文档关于single-copy atomic的说明。\
这里p=v实际上是通过条store指令实现的，把一个ARM通用寄存器的值（v）赋值给一个memory（指针p），只要满足对齐要求，这个操作就是原子操作。例如：如果赋值操作是8B size，那么要求地址是8B对齐的，只有在这种情况下才是原子操作。而把4字节数据写入0xc0008721这个地址（没有四字节对齐），这个操作就不是原子操作。

[回复](http://www.wowotech.net/kernel_synchronization/rcu_fundamentals.html#comment-6731)

**hony**\
2017-04-30 19:06

初看发现和自己解决的一个问题思路很像：有一块内存有时需要更新，但更新时间较长，同时有大量对这块内存的读。于是我就设计了主内存和备内存，用一个指针指向其中一块读内存。更新时更新另外一块内存不影响读，更新完成后直接将指针指向这块更新后的内存。这样读写不用锁就可以解决读写冲突问题。

[回复](http://www.wowotech.net/kernel_synchronization/rcu_fundamentals.html#comment-5506)

**RaulXiong**\
2016-11-01 14:46

RCU的原理这一块相对来说容易理解，个人感觉最精髓的在于如何寻找判断GP结束的时机。笔者在这方面有什么计划吗？

[回复](http://www.wowotech.net/kernel_synchronization/rcu_fundamentals.html#comment-4823)

**[linuxer](http://www.wowotech.net/)**\
2016-11-01 22:54

@RaulXiong：本来呢我打算把RCU和memory barrier、memory order什么的搞的一清二楚，不过放弃了，这些东西比较绕，有时候会把自己绕晕了，我目前的计划是：先把内存管理和进程管理搞清楚，然后回头啃这些“硬骨头”。

[回复](http://www.wowotech.net/kernel_synchronization/rcu_fundamentals.html#comment-4832)

**温柔海洋**\
2016-11-02 09:38

@linuxer：热切期待你这方面的文章，我们可以贡献出自己对内存管理和进程调度的理解。

[回复](http://www.wowotech.net/kernel_synchronization/rcu_fundamentals.html#comment-4836)

**[Daniel Shieh](http://www.wowotech.net/)**\
2015-12-07 15:54

网络子系统有没有什么规划？嘻嘻......

[回复](http://www.wowotech.net/kernel_synchronization/rcu_fundamentals.html#comment-3196)

**[wowo](http://www.wowotech.net/)**\
2015-12-08 09:17

@Daniel Shieh：精力实在有限的，估计一时半会不会涉及这方面的东西。

[回复](http://www.wowotech.net/kernel_synchronization/rcu_fundamentals.html#comment-3198)

**[Daniel Shieh](http://www.wowotech.net/)**\
2015-12-08 17:22

@wowo：感觉网络这方面的内容相比其他内容，实用性更强一些，就技术角度来说，也是最重要最复杂的一个子系统，或许什么时候有精力了，wowo和linuxer就可以下笔了，哈哈~~

[回复](http://www.wowotech.net/kernel_synchronization/rcu_fundamentals.html#comment-3202)

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

  - [ARM Linux上的系统调用代码分析](http://www.wowotech.net/process_management/syscall-arm.html)
  - [eMMC 原理 2 ：eMMC 简介](http://www.wowotech.net/basic_tech/emmc_intro.html)
  - [mellanox的网卡故障分析](http://www.wowotech.net/linux_kenrel/485.html)
  - [Linux kernel scatterlist API介绍](http://www.wowotech.net/memory_management/scatterlist.html)
  - [蜗窝流量地域统计](http://www.wowotech.net/168.html)

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

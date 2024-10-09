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

作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2016-5-13 19:18 分类：[ARMv8A Arch](http://www.wowotech.net/sort/armv8a_arch)

一、前言

本文主要解析ARMv8手册中的Atomicity这个概念。首先给出为何定义这样的概念，定义这个概念的作用为何？然后介绍Atomicity相关的概念，很多时候我们引用了手册的原文，但是由于这些原文象天书一样难懂（可读性比较差），因此，我们使用程序员可理解的一些语言来描述这些概念。最后给出ARMv8上，各种内存操作指令，针对各种memory type，其Atomicity的特性为何。

二、Atomicity概述

1、什么是Atomicity？

Atomicity是用来描述系统中的memory access的特性的一个术语。在单核系统上，我们用Single-copy atomicity这个术语来描述，也就是说该内存访问操作是否是原子的，是否是可以被打断的。在多核系统中，用Single-copy atomicity来描述一次内存访问的原子性是不够的，因为即便是在执行该内存访问指令的CPU CORE上是Single-copy atomicity的，也只不过说明该指令不会被本CPU CORE的异常或者中断之类的异步事件打断，它并不能阻止其他CPU core上的内存访问操作对同一地址上的memory location进行操作，这时候，我们使用Multi-copy atomicity来描述多个CPU CORE发起对同一个地址进行写入的时候，这些内存访问表现出来的特性是怎样的。

2、为何定义Atomicity？

无它，主要是为了软件和硬件能够和谐工作在一起。对于软件工程师而言，我们必须了解硬件和软件的“接口”，即那些是HW完成，那些是需要软件完成的，只有这样，软件和CPU的硬件才能愉快的一起玩耍。对于硬件，其architecture reference maual需要定义这些接口。对于ARM处理器而言（并非SOC，主要指CPU core），接口分成两个大类，第一类是指CPU定义的各种通用寄存器、状态寄存器、各种协处理器寄存器，CPU支持的指令集等，这些是属于比较好理解的那一部分，另外一类是关于行为或者说是操作的定义，这部分的接口不是那么明显，但是也是非常重要的。Atomicity即属于第二类接口定义。

三、基本概念解释

1、Coherent order

由于在Atomicity的定义中大量引用了该术语，因此我们这里需要先解释一下，关于coherent order，原文定义如下：

> Data accesses from a set of observers to a byte in memory are coherent if accesses to that byte in memory by the members of that set of observers are consistent with there being a single total order of all writes to that byte in memory by all members of the set of observers. This single total order of all to writes to that memory location is the coherence order for that byte in memory.

从这里的英文原文，我们可以得出下面的结论：

（1）coherent不是漫无边际的，而是受限于“a set of observers”，用ARMv8的术语就是shareability domain。属于同一个shareability domain的observers共享memory space，并且能够对同一个地址的memory进行操作。

（2）是否coherent这里是从shareability domain中的一个或者多个observers的视角来观察的。观察什么？观察的是写入的动作，具体的说就是该shareability domain中的多个observers对某个内存位置进行写入的动作。观察的结果是什么？如果是coherent的，那么shareability domain中的各个observers看到的是一个一致的、全局写入顺序。

（3）强调一下，这里的write serialization有一个前提条件就是写入的是同一个memory location。

（4）下面我们用一个具体的例子来说明什么是“single total order”。假设系统中有四个cpu core，分别执行同样的代码：cpux给一个全局变量A赋值为x，然后不断对A进行观察（即load操作）。在这个例子中A分别被四个CPU设定了1、 2、3、4的值，当然，先赋值的操作结果会被后来赋值操作覆盖，最后那个执行的write操作则决定了A变量最后的赋值。假设一次运行后，cpu 1看到的序列是{1,2}，cpu 2看到的序列是{2}，cpu 3看到的序列是{3,2}，cpu 4看到的序列是{4,2}，那么所有的cpu看到的顺序都是符合一个全局的顺序{3,1,4,2}，而各个CPU并没有能够观察到全部的中间过程，但是没 有关系，至少各个cpu观察的结果和那个全局顺序是一致的（consistent）。如果cpu 1看到的序列是{2,1}，那么就不存在一个一致性的全局顺序了，也就不是coherent order了。

（5）原文定义使用了“byte in memory”，实际上我的理解是要求内存访问是原子操作的，对于ARM体系，只有byte的访问是always保证是原子性的（single-copy atomicity），因此使用了byte这样的内存操作特例。

2、Single-copy atomicity

Single-copy atomicity英文原文定义如下：

> A read or write operation is single-copy atomic only if it meets the following conditions:
>
> 1. For a single-copy atomic store, if the store overlaps another single-copy atomic store, then all of the writes from one of the stores are inserted into the Coherence order of each overlapping byte before any of the writes of the other store are inserted into the Coherence orders of the overlapping bytes.
> 1. If a single-copy atomic load overlaps a single-copy atomic store and for any of the overlapping bytes the load returns the data written by the write inserted into the Coherence order of that byte by the single-copy atomic store then the load must return data from a point in the Coherence order no earlier than the writes inserted into the Coherence order by the single-copy atomic store of all of the overlapping bytes.

基本上来说，工程师可以知道上面这段话的每一个单词的含义，但是组合起来就是不知道这段话表达什么意思，为了方便理解，我们首先对几个单词进行解析：

（1）首先解释Coherence order ，就是上一章描述coherent时候的那个被所有observer观察到的全局的，一致的写入动作序列。

（2）对overlap的解释。基本上一个指令overlap另外一个指令其实就是说这两条指令被同时执行的意思。而“overlapping byte”则指内存操作有重叠的部分。例如加载0x000地址的4-Byte到寄存器和加载0x02地址2-Byte有2个字节的重叠。

（3）single-copy中copy到底是什么意思呢？我的理解是这样的：当PE访问内存的时候，例如load指令，这时候会有数据从memory copy到寄存器的动作，如果该指令的内存访问只会触发一次copy的动作，那么就是single-copy。对于加载奇数地址开始的2Byte load指令，其实该指令实际在执行的时候会触发两次的copy动作，那么就不是single-copy，而是multi-copy的（注意：这里的multi-copy并非Multi-copy atomicity中的Multi-copy，后文会描述）。

（4）“all of the writes from one of the stores ”这里all of the writes是指本次store操作中所涉及的每一个bit，这些bits是一个不可分隔的整体，插入到Coherence order操作序列中

理解了上面的几个英文单词之后，我们来看看整段的英文表述。整段表述分成两个部分，第一部分描述store overlap store，第二部分描述的是load overlap store。对于store overlap store而言，每一个store操作的bits都是不可分割的整体，而这个store连同其操作的所有bits做为一个原子的、不可被打断的操作，插入到Coherence order操作序列中。当然，插入时机也很重要，不能随便插入，不能在其他store中的中间过程中插入。如果操作的bits有交叠，例如有8个bit在A B两个store操作中都有涉及，那么这8个比特要么是A store的结果，要么是B store的结果，不能是一个综合A B store操作的结果。

理解了store overlap store之后，load overlap store就很容易了。它主要是从其他观察者的角度看：如果load和store操作的memory区域有交叠，那么那些交叠区域的返回值（对load操作而言）要么是全部bit被store写入，要么没有任何写入，不会是一个中间结果。

3、Multi-copy atomicity

Multi-copy atomicity英文原文定义如下：

> In a multiprocessing system, writes to a memory location are multi-copy atomic if the following conditions are both true:\
> 1、All writes to the same location are serialized, meaning they are observed in the same order by all observers, although some observers might not observe all of the writes.\
> 2、A read of a location does not return the value of a write until all observers observe that write.

Single-copy atomicity描述的是内存访问指令操作的原子性，而Multi-copy atomicity定义的是multiprocessing 环境下，多个store操作的顺序问题以及多个observer之间的交互问题，因此Single-copy atomicity和Multi-copy atomicity定义的内容是不一样的，或者说Multi-copy atomicity并不是站在Single-copy atomicity的对立面，它们就是不同的东西而已。那么，你可能会问：到底Multi-copy atomicity中的Multi-copy是什么意思呢？我理解是这样的：系统中有多个CPU core，每一个core都可以对内存系统中的某个特定的地址发起写入操作，系统中有n个CORE，那么就有可能有n个寄存器到memory的copy动作。

对Multi-copy atomicity的定义解释倒是比较简单：

（1）系统中对同一个地址的memory的store操作是串行化的，也就是说，对于所有的observer而言，它们观察到的写入操作顺序就是相同的一个序列。这个串行化要求比较狠，高于coherent的要求，也就是说，如果系统中的write操作不是coherent的，那么也就是不是Multi-copy atomicity的。

（2）对一个地址进行的load操作会被block，直到该地址的值对所有的observer都是可见的。

显然，根据定义，Multi-copy atomicity要求比较严格，对cpu performance伤害很大。

四、ARMv8的规则

1、Single-copy atomicity

显示内存访问（通过load或者store指令进行的内存访问）的规则如下：

（1）对齐的load或者store操作是Single-copy atomicity的。针对byte的内存操作总是Single-copy atomicity的，2B的load或者store操作如果地址对齐在2上，那么也是Single-copy atomicity的。其他的可以以此类推。

（2）Load Pair和Store Pair指令不是Single-copy atomicity的，但是可以被分拆成2个Single-copy atomicity指令。

（3）Load-Exclusive Pair（加载2个32-bit）指令和Store-Exclusive Pair（写入2个32-bit数据）指令是Single-copy atomicity的

（4）……更多的规则可以参考ARM ARM，这里不再描述。

2、Multi-copy atomicity规则如下：

（1）对于normal memory，写入操作不需要具备Multi-copy atomicity的特性。

（2）如果是Device类型的memory，并且具备non-Gathering的属性，所有符合Single-copy atomicity要求的write操作指令也都是Multi-copy atomicity的

（3）如果是Device类型的memory，并且具备Gathering的属性，写入操作不需要具备Multi-copy atomicity的特性。

五、参考文献

\[1\]     ARMv8 Architecture Reference Manual

原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/)。

Changelog：

2016-5-18：自己重新review了整份文档，让文档的表述更合理一些。

标签: [Coherent](http://www.wowotech.net/tag/Coherent) [Single-copy](http://www.wowotech.net/tag/Single-copy) [atomicity](http://www.wowotech.net/tag/atomicity) [Multi-copy](http://www.wowotech.net/tag/Multi-copy)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [ARMv8之memory model](http://www.wowotech.net/armv8a_arch/memory-model.html) | [X-002-HW-S900芯片boot from USB有关的硬件描述](http://www.wowotech.net/x_project/s900_hw_adfu.html)»

**评论：**

**anonymous**\
2018-11-24 12:02

其实copy不是“复制”的意思。它是名词，是“份”的意思。就像“I heard about your article. May I have a copy (of it)?”以及“I need two copies of your application form.”一样，是“一份”、“一式两份”里的“份”的意思。

[回复](http://www.wowotech.net/armv8a_arch/atomicity.html#comment-7055)

**[linuxer](http://www.wowotech.net/)**\
2018-11-30 09:10

@anonymous：你这个说挺有道理的，有空的时候再把这部分梳理一下。

[回复](http://www.wowotech.net/armv8a_arch/atomicity.html#comment-7064)

**yupluo**\
2017-11-17 17:19

做IC的人也不清楚，这个应该是语言的差距。和内部人交流后。下面的答案比较靠谱：

copy 类似 ”observed values“，这样就比较清楚了

It's using copy as a noun, as in "I have a copy of the Lord Of The Rings in paperback, and I have another copy in hardback".

So, for "single-copy atomic", any one of the copies (i.e. observed value) is guaranteed to be atomically updated; for "multi-copy atomic", all copies are atomically updated.

[回复](http://www.wowotech.net/armv8a_arch/atomicity.html#comment-6214)

**yupluo**\
2017-11-16 15:10

这个文章写得不错。

几个疑问：\
1）Multi-copy atomicity对cpu performance伤害很大,我的理解是这个不一定。 Intel架构是支持Multi-copy atomicity的。 ARM和Power不支持。

对ARM架构来说来说，issuing write的那个core A做了一个store后， A可以直接看到这个write，但是其他的core未必能看到。感觉主要是是否能snoop到write buffer。

对于bus， fabric设计来说。支持Multi-copy atomicity，要求某个store要到endpoint，假如bus上有个buffer，一个core写后，其他的core未必能看到，这样需要把barrier的transaction发送到外面。 现在ARMv8 cpu不建议barrier外发。所有bus设计必须考虑到这个要求。

2） 我也比较好奇这个copy在 single-copy atomic里面的意思。 估计需要 $313.85 买那本书：

from one article "A Tutorial Introduction to the ARM and POWER Relaxed Memory Models", it says ":

A memory read or write by an instruction is access-atomic (or single-copy atomic, in the terminology of Collier \[Col92\]—though note that single-copy atomic is not the opposite of multiple-copy atomic) if it gives rise to a single access to the memory.

Maybe only the book "Reasoning About Parallel Architectures , February 1, 1992" could tell ?

[回复](http://www.wowotech.net/armv8a_arch/atomicity.html#comment-6207)

**[linuxer](http://www.wowotech.net/)**\
2017-11-17 15:57

@yupluo：ARM ARM太难理解了，我也是把自己的疑问和自己的看法表达出来，不一定对。估计真正能给解答的是IC设计人员吧。^\_^

[回复](http://www.wowotech.net/armv8a_arch/atomicity.html#comment-6213)

**[狂奔的蜗牛](http://www.wowotech.net/)**\
2017-02-12 20:22

---------那么所有的cpu看到的顺序都是符合一个全局的顺序{3,1,4,2}-----\
这个可以是{4，3，1，2}吗？四个CPU的执行顺序也是乱序的么？single total order的字面意思好像是顺序是唯一的？

[回复](http://www.wowotech.net/armv8a_arch/atomicity.html#comment-5214)

**[linuxer](http://www.wowotech.net/)**\
2017-02-13 09:22

@狂奔的蜗牛：我的理解是可以的，当然只是我的看法，仅供参考。

假设一次运行后，\
cpu 1看到的序列是{1,2}，\
cpu 2看到的序列是{2}，\
cpu 3看到的序列是{3,2}，\
cpu 4看到的序列是{4,2}，\
{4，3，1，2}或者{3，4，1，2}或者{3，1，4，2}都可以，因为这些全局顺序和各个cpu的观察结果是coherent的。

[回复](http://www.wowotech.net/armv8a_arch/atomicity.html#comment-5216)

**xuwukong**\
2018-11-27 10:11

@linuxer：可以这样总结吗，各个CPU看到的最后一个值一定是同一个值，比如这里举例说的，四个CPU看到的最后一个值都是2，这样的顺序一定是coherent order。

[回复](http://www.wowotech.net/armv8a_arch/atomicity.html#comment-7056)

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

  - [Linux vm运行参数之（一）：overcommit相关的参数](http://www.wowotech.net/memory_management/overcommit.html)
  - [Linux读写锁逻辑解析](http://www.wowotech.net/kernel_synchronization/rwsem.html)
  - [蜗窝微信群问题整理](http://www.wowotech.net/tech_discuss/question_set_1.html)
  - [Linux内核同步机制之（四）：spin lock](http://www.wowotech.net/kernel_synchronization/spinlock.html)
  - [蓝牙协议分析(11)\_BLE安全机制之SM](http://www.wowotech.net/bluetooth/le_security_manager.html)

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

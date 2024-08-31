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

作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2014-12-10 22:43 分类：[ARMv8A Arch](http://www.wowotech.net/sort/armv8a_arch)

#### 1. 前言

蜗蜗很早以前就知道有WFI和WFE这两个指令存在，但一直似懂非懂。最近准备研究CPU idle framework，由于WFI是让CPU进入idle状态的一种方法，就下决心把它们弄清楚。

WFI(Wait for interrupt)和WFE(Wait for event)是两个让ARM核进入low-power standby模式的指令，由ARM architecture定义，由ARM core实现。听着挺简单，但怎么会有两个指令？它们的区别是什么？使用场景是什么？深究起来，还挺有意思，例如：能想象WFE和spinlock的关系吗？

#### 2. WFI和WFE

1）共同点

WFI和WFE的功能非常类似，以ARMv8-A为例（参考DDI0487A_d_armv8_arm.pdf的描述），主要是“将ARMv8-A PE(Processing Element, 处理单元)设置为low-power standby state”。

需要说明的是，ARM architecture并没有规定“low-power standby state”的具体形式，因而可以由ARM core自行发挥，根据ARM的建议，一般可以实现为standby（关闭clock、保持供电）、dormant、shutdown等等。但有个原则，不能造成内存一致性的问题。以[Cortex-A57](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0488g/DDI0488G_cortex_a57_mpcore_trm.pdf) ARM core为例，它把WFI和WFE实现为“put the core in a low-power state by disabling the clocks in the core while keeping the core powered up”，即我们通常所说的standby模式，保持供电，关闭clock。

2）不同点

那它们的区别体现在哪呢？主要体现进入和退出的方式上。

对WFI来说，执行WFI指令后，ARM core会立即进入low-power standby state，直到有WFI Wakeup events发生。

而WFE则稍微不同，执行WFE指令后，根据Event Register（一个单bit的寄存器，每个PE一个）的状态，有两种情况：如果Event Register为1，该指令会把它清零，然后执行完成（不会standby）；如果Event Register为0，和WFI类似，进入low-power standby state，直到有WFE Wakeup events发生。

WFI wakeup event和WFE wakeup event可以分别让Core从WFI和WFE状态唤醒，这两类Event大部分相同，如任何的IRQ中断、FIQ中断等等，一些细微的差别，可以参考“DDI0487A_d_armv8_arm.pdf“的描述。而最大的不同是，WFE可以被任何PE上执行的SEV指令唤醒。

所谓的SEV指令，就是一个用来改变Event Register的指令，有两个：SEV会修改所有PE上的寄存器；SEVL，只修改本PE的寄存器值。下面让我们看看WFE这种特殊设计的使用场景。

#### 3. 使用场景

1）WFI

WFI一般用于cpuidle。

2）WFE

WFE的一个典型使用场景，是用在spinlock中（可参考arch_spin_lock，对arm64来说，位于arm64/include/asm/spinlock.h中）。spinlock的功能，是在不同CPU core之间，保护共享资源。使用WFE的流程是：

> a）资源空闲
> 
> b）Core1访问资源，acquire lock，获得资源
> 
> c）Core2访问资源，此时资源不空闲，执行WFE指令，让core进入low-power state
> 
> d）Core1释放资源，release lock，释放资源，同时执行SEV指令，唤醒Core2
> 
> e）Core2获得资源

以往的spinlock，在获得不到资源时，让Core进入busy loop，而通过插入WFE指令，可以节省功耗，也算是因祸（损失了性能）得福（降低了功耗）吧。

_原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/armv8a_arch/wfe_wfi.html)。_

标签: [Architecture](http://www.wowotech.net/tag/Architecture) [aarch64](http://www.wowotech.net/tag/aarch64) [ARM](http://www.wowotech.net/tag/ARM) [wfe](http://www.wowotech.net/tag/wfe) [wfi](http://www.wowotech.net/tag/wfi)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Linux时间子系统之（十二）：periodic tick](http://www.wowotech.net/timer_subsystem/periodic-tick.html) | [ARM概念梳理：Architecture, Core, CPU，SOC](http://www.wowotech.net/armv8a_arch/arm_concept.html)»

**评论：**

**育**  
2024-05-31 16:25

@wowo 想請問大神，我的board有兩個Kernel 但只有其中一個可以輸入指令另一個不能，這樣我要怎麼輸入WFI command

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-8903)

**我是来点赞的**  
2021-12-04 22:28

赞

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-8390)

**Rafe**  
2018-04-19 10:20

@wowo @linuxer  
Hi 大神们，想请教你们一个问题：系统suspend调用的是WFI, 那么WFI 执行后，系统进入suspend, 那么接下来如果中断产生了，系统是如何恢复到原有的C状态的？中间理应会有一段汇编执行保存恢复跳转才对，请问对该部分是否有了解过? Thanks!

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-6682)

**gnx**  
2017-12-15 11:43

@wowo，请教一个问题哈，如果在关中断之后，wfi之前收到了一个中断，执行wfi指令会发生怎样的行为？

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-6366)

**gnx**  
2017-12-15 12:48

@gnx：在手册找到答案了，如果有中断处于pending状态，wfi不会进入睡眠，伪代码如下  
if ConditionPassed() then  
    EncodingSpecificOperations();  
    if !InterruptPending() then  
        if PSTATE.EL == EL0 then  
            AArch32.CheckForWFxTrap(EL1, FALSE);  
        if HaveEL(EL2) && !IsSecure() && PSTATE.EL IN {EL0,EL1} then  
            AArch32.CheckForWFxTrap(EL2, FALSE);  
        if HaveEL(EL3) && PSTATE.M != M32_Monitor then  
            AArch32.CheckForWFxTrap(EL3, FALSE);  
        WaitForInterrupt();

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-6367)

**lamaboy**  
2016-08-17 23:01

你好，，  
请问 spin_lock  只是用来在不同CPU core之间，保护共享资源吗？？  
  
对于那些单cpu 的spin_lock   有什么不同之处呢？？

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-4415)

**[wowo](http://www.wowotech.net/)**  
2016-08-18 14:54

@lamaboy：spin的意思，就是让某个CPU在原地打转，这样就可以避免它去访问共享资源。  
对于单个CPU来说，它只能顺序执行（除非被中断、异常打断），因此没有spin的必要，也就没有spin_lock的概念。  
单CPU需要做同步的话，就把自己的中断关掉就行了。

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-4416)

**randy**  
2016-09-01 19:15

@wowo：单核下spin lock也是有意义的：用于线程之间的同步时禁止内核抢占，保证被中断后仍然回到原先的线程中；用于线程上下文和中断上下文之间的同步时根据上半部和下半部分别关中断和关下半部

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-4489)

**[linuxer](http://www.wowotech.net/)**  
2016-09-01 19:45

@randy：我想wowo同学的本意是：在单核情况下，spin lock退化成了禁止抢占，其实已经脱离了spin lock的本意，因此，他才说“对于单个CPU来说，它只能顺序执行（除非被中断、异常打断），因此没有spin的必要”  
  
我其实比较挑战的是他的另外一句话“单CPU需要做同步的话，就把自己的中断关掉就行了”，一言不合就使用关中断这种大杀器我是反对的，可以选择使用最适合的同步方法。

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-4490)

**[wowo](http://www.wowotech.net/)**  
2016-09-02 08:55

@randy：是的，randy和linuxer的说法比我说的“精细”多了，多谢~~  
我的风格是比较直接，不太希望把话题延伸的太宽，大家多包涵哈~~~

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-4495)

**坚持到底**  
2015-11-17 11:32

arm64 smp 中，并没有直接在 arch_spin_unlock() 里使用 sev 来唤醒执行 wfe 的 CPU，请问楼主有没研究过执行 wfe 的 CPU 怎么被唤醒的？

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-3065)

**[wowo](http://www.wowotech.net/)**  
2015-11-17 13:45

@坚持到底：当前ARM Kernel的spin_lock是不是没有利用WFE节省功耗？  
请教linuxer，ldaxrh让CPU进入低功耗吗？如果不会，现在仅仅用了busy loop的方式。

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-3067)

**坚持到底**  
2015-11-17 14:11

@wowo：static inline void arch_spin_lock(arch_spinlock_t *lock)  
{  
    unsigned int tmp;  
  
    asm volatile(  
    "    sevl\n"  
    "1:    wfe\n"  
    "2:    ldaxr    %w0, %1\n"  
    "    cbnz    %w0, 1b\n"  
    "    stxr    %w0, %w2, %1\n"  
    "    cbnz    %w0, 2b\n"  
    : "=&r" (tmp), "+Q" (lock->lock)  
    : "r" (1)  
    : "cc", "memory");  
}  
  
static inline void arch_spin_unlock(arch_spinlock_t *lock)  
{  
    asm volatile(  
    "    stlr    %w1, %0\n"  
    : "=Q" (lock->lock) : "r" (0) : "memory");  
}  
  
我的 linux kernel  3.10.61，是有用到 wfe 的。

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-3068)

**[wowo](http://www.wowotech.net/)**  
2015-11-17 14:33

@坚持到底：这两个配合：  
"    sevl\n"  
    "1:    wfe\n"  
wfe不会进入低功耗模式的。

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-3069)

**坚持到底**  
2015-11-17 14:58

@wowo：会的。  
  
假设 cpu0 执行下面伪代码：  
  
"    sevl\n"  //set event register 1  
    "1:    wfe\n"  //clear event register 为 0，执行 wfe 完毕，不睡眠。  
    "2:    ldaxr    %w0, %1\n" //tmp = lock->lock, 测试 lock->lock是否有 CPU 占用，1 为有 CPU 占用，0 为没 CPU 占用。假设此时已经有 CPU 占用被设置为 1, 则非零，跳转到 1b, wfe.  
    "    cbnz    %w0, 1b\n"

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-3070)

**[linuxer](http://www.wowotech.net/)**  
2015-11-17 15:05

@坚持到底：当没有获取spinlock的时候，cpu core当然会调用wfe，等待其他cpu使用sev来唤醒自己。linux kernel的ARM64代码当然不会busy loop了，那太业余了。具体可以参考：  
http://www.wowotech.net/kernel_synchronization/spinlock.html  
  
在ARM64中，arch_spin_unlock并没有显示的调用sev来唤醒其他cpu，而是通过stlr指令完成的。在ARM ARM文档中有说：在执行store操作的时候，如果要操作的地址被标记为exclusive的，那么global monitor的状态会从exclusive access变成open access，同时会触发一个事件，唤醒wfe中的cpu。

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-3071)

**[wowo](http://www.wowotech.net/)**  
2015-11-17 15:42

@linuxer：嗯嗯，这就对了，我也一直怀疑stlr，去memory barrier文档中找了很久没找到信息，原来在spin lock中。  
  
@坚持到底：抱歉误导您了啊，好在有大神在，哈哈。

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-3073)

**[linuxer](http://www.wowotech.net/)**  
2015-11-17 18:27

@wowo：大神不敢当啊，我还是喜欢华南区（深圳，珠海除外）首席xxx这样的称号，^_^

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-3078)

**坚持到底**  
2015-11-17 15:54

@linuxer：是的，感谢啊。在 6000 页的 pdf 中找到：  
  
The ARMv8 architecture adds the acquire and release semantics to Load-Exclusive and Store-Exclusive instructions, which allows them to gain ordering acquire and/or release semantics.  
  
The Load-Exclusive instruction can be specified to have acquire semantics, and the Store-Exclusive instruction can be specified to have release semantics. These can be arbitrarily combined to allow the atomic update created by a successful Load-Exclusive and Store-Exclusive pair to have any of:  
  
•  
No Ordering semantics (using LDREX and STREX).  
•  
Acquire only semantics (using LDAEX and STREX).  
•  
Release only semantics (using LDREX and STLEX).  
•  
Sequentially consistent semantics (using LDAEX and STLEX).  
  
In addition, the ARMv8 specification requires that the clearing of a global monitor will generate an event for the PE associated with the global monitor, which can simplify the use of WFE, by removing the need for a DSB barrier and SEV instruction.

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-3075)

**[linuxer](http://www.wowotech.net/)**  
2015-11-17 18:29

@坚持到底：关于这个问题可以参考ARM ARM文档中的Figure B2-5，这个图是PE（n）的global monitor的状态迁移图。当PE（n）对x地址发起了exclusive操作的时候，PE（n）的global monitor从open access迁移到exclusive access状态，来自其他PE上针对x（该地址已经被mark for PE（n））的store操作会导致PE（n）的global monitor从exclusive access迁移到open access状态，这时候，PE（n）的Event register会被写入event，就好象生成一个event，将该PE唤醒，从而可以省略一个SEV的指令。

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-3079)

**[坚持到底](http://www.wowotech.net/)**  
2015-11-19 01:01

@linuxer：这里我有个疑问。  
  
假设 spinlock 场景：  
  
1. cpu0 exclusive load, and exclusive store 操作 lock->lock.此时 global monitor 对于 lock->lock 的 status 依然是 exclusive.  
  
2. cpu1 exclusive load, and wfe.  
  
3. cpu2 exclusive load, and wfe.  
  
4. cpu3 exclusive load, and wfe.  
  
5. 此时只有 cpu0 能去 arch_spin_unlock()， exclusive store 操作 lock->lock, 本应该 excl -> open，会触发 global monitor generate an event for the PE.  
  
但是从你上一段描述和 Figure B2-5 的意思又不一致。 上面 spinlock 的场景根本不会有其他 PE 上针对 x( mark for exclusive ) 的操作了。Figure B2-5 图中也表示 StoreExcl(Marked_address,n) 进行 excl access 操作，global monitor status 是不变的。

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-3085)

**[linuxer](http://www.wowotech.net/)**  
2015-11-19 10:34

@坚持到底：好吧，看来我表述的不是那么的清楚，针对你说的场景，我重新组织一下：  
在cpu0调用unlock之前，各个状态机的状态如下：  
1、cpu0上针对spink lockmemory的状态机是open access状态。当然，这个状态和具体实现相关，不过，在这个场景中，没有人关注它的状态。  
2、cpu1上针对spink lockmemory的状态机是exclusive access状态，这时候cpu1处于wfe  
3、cpu2状态机和cpu1相同  
4、cpu3状态机和cpu1相同  
  
  
cpu0执行了spin_unlock操作，各个状态机的迁移情况如下：  
1、cpu0上针对spink lockmemory的状态机是怎样的呢？有人在乎吗？需要知道它的状态吗？当然不需要。  
2、cpu1的状态迁移：这时候实际上产生的事件是有其他cpu（指cpu0）执行了针对marked address（指共享的spin lock memory地址）的store操作，在状态图上对应Store(Marked_address,!n)事件，因此，该状态机迁移到open access状态，这时候cpu1的Event register会被写入event，就好象生成一个event，将cpu1唤醒  
3、cpu2类似cpu1  
4、cpu3类似cpu1  
  
  
上面是我的理解，请参考

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-3087)

**[坚持到底](http://www.wowotech.net/)**  
2015-11-19 23:51

@linuxer：对的，谢谢解释啊。^_^

**[omind](http://omind.org/)**  
2015-05-28 11:48

都写到armv8了，先进啊

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-1893)

**[shoujixiaodao](http://www.wowotech.net/)**  
2015-04-29 22:41

我也一直疑惑这两条指令，今日才明白。如拨云见日。感谢！

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-1641)

**[wowo](http://www.wowotech.net/)**  
2015-04-29 23:13

@shoujixiaodao：不必客气，大家多交流~~

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-1646)

**rockwu**  
2015-01-26 13:44

核心是WFI只能由cpu硬件来唤醒，WFE除了硬件唤醒之外还支持软件唤醒。所以WFE的自由度更好，只要WFE和SEV成对使用。不知道理解是否正确。

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-1133)

**[wowo](http://www.wowotech.net/)**  
2015-01-26 22:32

@rockwu：应该就是您说理解的那个样子。  
不过是不是必须“由CPU硬件唤醒”，这个不能太武断，因为到底什么是硬件的范畴，不太好界定。

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-1136)

**[haichunzhao](http://www.wowotech.net/)**  
2014-12-11 18:10

刚注册了个账号  
不论cpuidle和平台sleep最后走的都是WFI。WFE没研究过，看到你写的，在spinlock中使用的话，感觉和信号量差不多了。

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-889)

**wowo**  
2014-12-11 18:26

@haichunzhao：sleep应该是ARM core或者ARM cpu的低功耗状态，估计会在WFI之后，再关闭一些其它的东西。不知道是不是所有CPU都一样。  
行为确实和信号量类似，但信号量依赖软件的调度，而使用WFE的spinlock是硬件行为。

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-890)

**sea**  
2015-01-15 11:30

@wowo：wowo，“cpuidle和平台sleep最后走的都是WFI”，那sleep会调用cpuidle framework来走WFI吗？如果不是，那不就意味着sleep也就相当于使用的是《Linux cpuidle framework(1)_概述和软件架构》中“曾经有过一段时间的cpuidle过程”。而且像android这种移动端os，能够进入到idle进程的情况下，应该早就进行suspend操作了吧。这样的话cpuidle framework不就没什么意义了。或者说suspend也是使用cpuidle framework来走WFI，又或者有其它的应用场景?

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-1090)

**[wowo](http://www.wowotech.net/)**  
2015-01-15 13:18

@sea：1. sleep这个词可能过于含糊，不同平台的实现可能不同，因此是否会使用WFI，是不一定的。  
2. WFI只是ARM体系结构的一个指令，谁会使用这个指令，spinlock？cpuidle？还是sleep？是没有限制的，因此也不一定非要走cpuidle framework。  
3. 你说的很对，Android选择了autosleep作为自身电源管理的主要手段，cpuidle framework就没有太多用处了，甚至，大多数的Android设备，没有启用cpuidle framework功能。  
4. 个人意见，cpuidle framework的主要使用场景，是在服务器系统中，这些系统的cpu core动辄就几十个，对系统响应能力的要求又比较高，通过cpuidle，既可以保证性能，也可以节省很多power。

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-1093)

**[shoujixiaodao](http://www.wowotech.net/)**  
2015-04-29 22:52

@sea：说说我的见解。只限于手机场景。不论cpuide和sleep虽都调用wfi，但流程不同。很多情况下是cpu idle状态。比如你把手机亮屏不做任何操作有可能进入cpu idle，灭屏听fm也可能会进入。

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-1643)

**[wowo](http://www.wowotech.net/)**  
2015-04-29 23:12

@shoujixiaodao：我接触的手机平台不多，不知道它们是否使能了cpuidle功能。从道理上来说，确实是这样的。

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-1645)

**[shoujixiaodao](http://www.wowotech.net/)**  
2015-04-29 22:45

@wowo：我的理解是：信号量依赖软件的调度，所以cpu还干活。而使用WFE的spinlock。则cpu进入低功耗模式。不干活了。

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-1642)

**[wowo](http://www.wowotech.net/)**  
2015-04-29 23:11

@shoujixiaodao：是的，完全正确，所以从这个角度看，spinlock并不是那么可怕了。

[回复](http://www.wowotech.net/armv8a_arch/wfe_wfi.html#comment-1644)

1 [2](http://www.wowotech.net/armv8a_arch/wfe_wfi.html/comment-page-2#comments)

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
    
    - Shiina  
        [一个电路（circuit）中，由于是回路，所以用电势差的概念...](http://www.wowotech.net/basic_subject/voltage.html#8926)
    - Shiina  
        [其中比较关键的点是相对位置概念和点电荷的静电势能计算。](http://www.wowotech.net/basic_subject/voltage.html#8925)
    - leelockhey  
        [你这是哪个内核版本](http://www.wowotech.net/pm_subsystem/generic_pm_architecture.html#8924)
    - ja  
        [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
    - 元神高手  
        [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
    - 十七  
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
    
    - [linux kernel内存回收机制](http://www.wowotech.net/memory_management/233.html)
    - [CFS调度器（6）-总结](http://www.wowotech.net/process_management/452.html)
    - [以太网驱动的流程浅析(一)-Ifconfig主要流程](http://www.wowotech.net/linux_kenrel/464.html)
    - [X-019-KERNEL-串口驱动开发之数据收发](http://www.wowotech.net/x_project/serial_driver_porting_4.html)
    - [蓝牙协议分析(2)_协议架构](http://www.wowotech.net/bluetooth/bt_protocol_arch.html)
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
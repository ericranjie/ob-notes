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

作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2015-4-22 12:22 分类：[内核同步机制](http://www.wowotech.net/sort/kernel_synchronization)

一、前言

在linux kernel的实现中，经常会遇到这样的场景：共享数据被中断上下文和进程上下文访问，该如何保护呢？如果只有进程上下文的访问，那么可以考虑使用semaphore或者mutex的锁机制，但是现在中断上下文也参和进来，那些可以导致睡眠的lock就不能使用了，这时候，可以考虑使用spin lock。本文主要介绍了linux kernel中的spin lock的原理以及代码实现。由于spin lock是architecture dependent代码，因此，我们在第四章讨论了ARM32和ARM64上的实现细节。

注：本文需要进程和中断处理的基本知识作为支撑。

二、工作原理

1、spin lock的特点

我们可以总结spin lock的特点如下：

（1）spin lock是一种死等的锁机制。当发生访问资源冲突的时候，可以有两个选择：一个是死等，一个是挂起当前进程，调度其他进程执行。spin lock是一种死等的机制，当前的执行thread会不断的重新尝试直到获取锁进入临界区。

（2）只允许一个thread进入。semaphore可以允许多个thread进入，spin lock不行，一次只能有一个thread获取锁并进入临界区，其他的thread都是在门口不断的尝试。

（3）执行时间短。由于spin lock死等这种特性，因此它使用在那些代码不是非常复杂的临界区（当然也不能太简单，否则使用原子操作或者其他适用简单场景的同步机制就OK了），如果临界区执行时间太长，那么不断在临界区门口“死等”的那些thread是多么的浪费CPU啊（当然，现代CPU的设计都会考虑同步原语的实现，例如ARM提供了WFE和SEV这样的类似指令，避免CPU进入busy loop的悲惨境地）

（4）可以在中断上下文执行。由于不睡眠，因此spin lock可以在中断上下文中适用。

2、 场景分析

对于spin lock，其保护的资源可能来自多个CPU CORE上的进程上下文和中断上下文的中的访问，其中，进程上下文包括：用户进程通过系统调用访问，内核线程直接访问，来自workqueue中work function的访问（本质上也是内核线程）。中断上下文包括：HW interrupt context（中断handler）、软中断上下文（soft irq，当然由于各种原因，该softirq被推迟到softirqd的内核线程中执行的时候就不属于这个场景了，属于进程上下文那个分类了）、timer的callback函数（本质上也是softirq）、tasklet（本质上也是softirq）。

先看最简单的单CPU上的进程上下文的访问。如果一个全局的资源被多个进程上下文访问，这时候，内核如何交错执行呢？对于那些没有打开preemptive选项的内核，所有的系统调用都是串行化执行的，因此不存在资源争抢的问题。如果内核线程也访问这个全局资源呢？本质上内核线程也是进程，类似普通进程，只不过普通进程时而在用户态运行、时而通过系统调用陷入内核执行，而内核线程永远都是在内核态运行，但是，结果是一样的，对于non-preemptive的linux kernel，只要在内核态，就不会发生进程调度，因此，这种场景下，共享数据根本不需要保护（没有并发，谈何保护呢）。如果时间停留在这里该多么好，单纯而美好，在继续前进之前，让我们先享受这一刻。

当打开premptive选项后，事情变得复杂了，我们考虑下面的场景：

（1）进程A在某个系统调用过程中访问了共享资源R

（2）进程B在某个系统调用过程中也访问了共享资源R

会不会造成冲突呢？假设在A访问共享资源R的过程中发生了中断，中断唤醒了沉睡中的，优先级更高的B，在中断返回现场的时候，发生进程切换，B启动执行，并通过系统调用访问了R，如果没有锁保护，则会出现两个thread进入临界区，导致程序执行不正确。OK，我们加上spin lock看看如何：A在进入临界区之前获取了spin lock，同样的，在A访问共享资源R的过程中发生了中断，中断唤醒了沉睡中的，优先级更高的B，B在访问临界区之前仍然会试图获取spin lock，这时候由于A进程持有spin lock而导致B进程进入了永久的spin……怎么破？linux的kernel很简单，在A进程获取spin lock的时候，禁止本CPU上的抢占（上面的永久spin的场合仅仅在本CPU的进程抢占本CPU的当前进程这样的场景中发生）。如果A和B运行在不同的CPU上，那么情况会简单一些：A进程虽然持有spin lock而导致B进程进入spin状态，不过由于运行在不同的CPU上，A进程会持续执行并会很快释放spin lock，解除B进程的spin状态。

多CPU core的场景和单核CPU打开preemptive选项的效果是一样的，这里不再赘述。

我们继续向前分析，现在要加入中断上下文这个因素。访问共享资源的thread包括：

（1）运行在CPU0上的进程A在某个系统调用过程中访问了共享资源R

（2）运行在CPU1上的进程B在某个系统调用过程中也访问了共享资源R

（3）外设P的中断handler中也会访问共享资源R

在这样的场景下，使用spin lock可以保护访问共享资源R的临界区吗？我们假设CPU0上的进程A持有spin lock进入临界区，这时候，外设P发生了中断事件，并且调度到了CPU1上执行，看起来没有什么问题，执行在CPU1上的handler会稍微等待一会CPU0上的进程A，等它立刻临界区就会释放spin lock的，但是，如果外设P的中断事件被调度到了CPU0上执行会怎么样？CPU0上的进程A在持有spin lock的状态下被中断上下文抢占，而抢占它的CPU0上的handler在进入临界区之前仍然会试图获取spin lock，悲剧发生了，CPU0上的P外设的中断handler永远的进入spin状态，这时候，CPU1上的进程B也不可避免在试图持有spin lock的时候失败而导致进入spin状态。为了解决这样的问题，linux kernel采用了这样的办法：如果涉及到中断上下文的访问，spin lock需要和禁止本CPU上的中断联合使用。

linux kernel中提供了丰富的bottom half的机制，虽然同属中断上下文，不过还是稍有不同。我们可以把上面的场景简单修改一下：外设P不是中断handler中访问共享资源R，而是在的bottom half中访问。使用spin lock+禁止本地中断当然是可以达到保护共享资源的效果，但是使用牛刀来杀鸡似乎有点小题大做，这时候disable bottom half就OK了。

最后，我们讨论一下中断上下文之间的竞争。同一种中断handler之间在uni core和multi core上都不会并行执行，这是linux kernel的特性。如果不同中断handler需要使用spin lock保护共享资源，对于新的内核（不区分fast handler和slow handler），所有handler都是关闭中断的，因此使用spin lock不需要关闭中断的配合。bottom half又分成softirq和tasklet，同一种softirq会在不同的CPU上并发执行，因此如果某个驱动中的sofirq的handler中会访问某个全局变量，对该全局变量是需要使用spin lock保护的，不用配合disable CPU中断或者bottom half。tasklet更简单，因为同一种tasklet不会多个CPU上并发，具体我就不分析了，大家自行思考吧。

三、通用代码实现

1、文件整理

和体系结构无关的代码如下：

（1）include/linux/spinlock_types.h。这个头文件定义了通用spin lock的基本的数据结构（例如spinlock_t）和如何初始化的接口（DEFINE_SPINLOCK）。这里的“通用”是指不论SMP还是UP都通用的那些定义。

（2）include/linux/spinlock_types_up.h。这个头文件不应该直接include，在include/linux/spinlock_types.h文件会根据系统的配置（是否SMP）include相关的头文件，如果UP则会include该头文件。这个头文定义UP系统中和spin lock的基本的数据结构和如何初始化的接口。当然，对于non-debug版本而言，大部分struct都是empty的。

（3）include/linux/spinlock.h。这个头文件定义了通用spin lock的接口函数声明，例如spin_lock、spin_unlock等，使用spin lock模块接口API的驱动模块或者其他内核模块都需要include这个头文件。

（4）include/linux/spinlock_up.h。这个头文件不应该直接include，在include/linux/spinlock.h文件会根据系统的配置（是否SMP）include相关的头文件。这个头文件是debug版本的spin lock需要的。

（5）include/linux/spinlock_api_up.h。同上，只不过这个头文件是non-debug版本的spin lock需要的

（6）linux/spinlock_api_smp.h。SMP上的spin lock模块的接口声明

（7）kernel/locking/spinlock.c。SMP上的spin lock实现。

头文件有些凌乱，我们对UP和SMP上spin lock头文件进行整理：

|   |   |
|---|---|
|UP需要的头文件|SMP需要的头文件|
|linux/spinlock_type_up.h:  <br>linux/spinlock_types.h:  <br>linux/spinlock_up.h:  <br>linux/spinlock_api_up.h:  <br>linux/spinlock.h|asm/spinlock_types.h  <br>linux/spinlock_types.h:  <br>asm/spinlock.h  <br>linux/spinlock_api_smp.h:  <br>linux/spinlock.h|

2、数据结构

根据第二章的分析，我们可以基本可以推断出spin lock的实现。首先定义一个spinlock_t的数据类型，其本质上是一个整数值（对该数值的操作需要保证原子性），该数值表示spin lock是否可用。初始化的时候被设定为1。当thread想要持有锁的时候调用spin_lock函数，该函数将spin lock那个整数值减去1，然后进行判断，如果等于0，表示可以获取spin lock，如果是负数，则说明其他thread的持有该锁，本thread需要spin。

内核中的spinlock_t的数据类型定义如下：

> typedef struct spinlock {\
> struct raw_spinlock rlock; \
> } spinlock_t;
>
> typedef struct raw_spinlock {\
> arch_spinlock_t raw_lock;\
> } raw_spinlock_t;

由于各种原因（各种锁的debug、锁的validate机制，多平台支持什么的），spinlock_t的定义没有那么直观，为了让事情简单一些，我们去掉那些繁琐的成员。struct spinlock中定义了一个struct raw_spinlock的成员，为何会如此呢？好吧，我们又需要回到kernel历史课本中去了。在旧的内核中（比如我熟悉的linux 2.6.23内核），spin lock的命令规则是这样：

通用（适用于各种arch）的spin lock使用spinlock_t这样的type name，各种arch定义自己的struct raw_spinlock。听起来不错的主意和命名方式，直到linux realtime tree（PREEMPT_RT）提出对spinlock的挑战。real time linux是一个试图将linux kernel增加硬实时性能的一个分支（你知道的，linux kernel mainline只是支持soft realtime），多年来，很多来自realtime branch的特性被merge到了mainline上，例如：高精度timer、中断线程化等等。realtime tree希望可以对现存的spinlock进行分类：一种是在realtime kernel中可以睡眠的spinlock，另外一种就是在任何情况下都不可以睡眠的spinlock。分类很清楚但是如何起名字？起名字绝对是个技术活，起得好了事半功倍，可维护性好，什么文档啊、注释啊都素那浮云，阅读代码就是享受，如沐春风。起得不好，注定被后人唾弃，或者拖出来吊打（这让我想起给我儿子起名字的那段不堪回首的岁月……）。最终，spin lock的命名规范定义如下：

（1）spinlock，在rt linux（配置了PREEMPT_RT）的时候可能会被抢占（实际底层可能是使用支持PI（优先级翻转）的mutext）。

（2）raw_spinlock，即便是配置了PREEMPT_RT也要顽强的spin

（3）arch_spinlock，spin lock是和architecture相关的，arch_spinlock是architecture相关的实现

对于UP平台，所有的arch_spinlock_t都是一样的，定义如下：

> typedef struct { } arch_spinlock_t;

什么都没有，一切都是空啊。当然，这也符合前面的分析，对于UP，即便是打开的preempt选项，所谓的spin lock也不过就是disable preempt而已，不需定义什么spin lock的变量。

对于SMP平台，这和arch相关，我们在下一节描述。

3、spin lock接口API

我们整理spin lock相关的接口API如下：

|   |   |   |
|---|---|---|
|接口API的类型|spinlock中的定义|raw_spinlock的定义|
|定义spin lock并初始化|DEFINE_SPINLOCK|DEFINE_RAW_SPINLOCK|
|动态初始化spin lock|spin_lock_init|raw_spin_lock_init|
|获取指定的spin lock|spin_lock|raw_spin_lock|
|获取指定的spin lock同时disable本CPU中断|spin_lock_irq|raw_spin_lock_irq|
|保存本CPU当前的irq状态，disable本CPU中断并获取指定的spin lock|spin_lock_irqsave|raw_spin_lock_irqsave|
|获取指定的spin lock同时disable本CPU的bottom half|spin_lock_bh|raw_spin_lock_bh|
|释放指定的spin lock|spin_unlock|raw_spin_unlock|
|释放指定的spin lock同时enable本CPU中断|spin_unlock_irq|raw_spin_unock_irq|
|释放指定的spin lock同时恢复本CPU的中断状态|spin_unlock_irqstore|raw_spin_unlock_irqstore|
|获取指定的spin lock同时enable本CPU的bottom half|spin_unlock_bh|raw_spin_unlock_bh|
|尝试去获取spin lock，如果失败，不会spin，而是返回非零值|spin_trylock|raw_spin_trylock|
|判断spin lock是否是locked，如果其他的thread已经获取了该lock，那么返回非零值，否则返回0|spin_is_locked|raw_spin_is_locked|
||||

在具体的实现面，我们不可能把每一个接口函数的代码都呈现出来，我们选择最基础的spin_lock为例子，其他的读者可以自己阅读代码来理解。

spin_lock的代码如下：

> static inline void spin_lock(spinlock_t \*lock)\
> {\
> raw_spin_lock(&lock->rlock);\
> }

当然，在linux mainline代码中，spin_lock和raw_spin_lock是一样的，在realtime linux patch中，spin_lock应该被换成可以sleep的版本，当然具体如何实现我没有去看（也许直接使用了Mutex，毕竟它提供了优先级继承特性来解决了优先级翻转的问题），有兴趣的读者可以自行阅读，我们这里重点看看（本文也主要focus这个主题）真正的，不睡眠的spin lock，也就是是raw_spin_lock，代码如下：

> #define raw_spin_lock(lock)    \_raw_spin_lock(lock)
>
> UP中的实现：
>
> #define \_raw_spin_lock(lock)            \_\_LOCK(lock)
>
> #define \_\_LOCK(lock) \\
> do { preempt_disable(); \_\_\_LOCK(lock); } while (0)
>
> SMP的实现：
>
> void \_\_lockfunc \_raw_spin_lock(raw_spinlock_t \*lock)\
> {\
> \_\_raw_spin_lock(lock);\
> }
>
> static inline void \_\_raw_spin_lock(raw_spinlock_t \*lock)\
> {\
> preempt_disable();\
> spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);\
> LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock);\
> }

UP中很简单，本质上就是一个preempt_disable而已，和我们在第二章中分析的一致。SMP中稍显复杂，preempt_disable当然也是必须的，spin_acquire可以略过，这是和运行时检查锁的有效性有关的，如果没有定义CONFIG_LOCKDEP其实就是空函数。如果没有定义CONFIG_LOCK_STAT（和锁的统计信息相关），LOCK_CONTENDED就是调用do_raw_spin_lock而已，如果没有定义CONFIG_DEBUG_SPINLOCK，它的代码如下：

> static inline void do_raw_spin_lock(raw_spinlock_t \*lock) \_\_acquires(lock)\
> {\
> \_\_acquire(lock);\
> arch_spin_lock(&lock->raw_lock);\
> }

\_\_acquire和静态代码检查相关，忽略之，最终实际的获取spin lock还是要靠arch相关的代码实现。

四、ARM平台的细节

代码位于arch/arm/include/asm/spinlock.h和spinlock_type.h，和通用代码类似，spinlock_type.h定义ARM相关的spin lock定义以及初始化相关的宏；spinlock.h中包括了各种具体的实现。

1、回忆过去

在分析新的spin lock代码之前，让我们先回到2.6.23版本的内核中，看看ARM平台如何实现spin lock的。和arm平台相关spin lock数据结构的定义如下（那时候还是使用raw_spinlock_t而不是arch_spinlock_t）：

> typedef struct {\
> volatile unsigned int lock;\
> } raw_spinlock_t;

一个整数就OK了，0表示unlocked，1表示locked。配套的API包括\_\_raw_spin_lock和\_\_raw_spin_unlock。\_\_raw_spin_lock会持续判断lock的值是否等于0，如果不等于0（locked）那么其他thread已经持有该锁，本thread就不断的spin，判断lock的数值，一直等到该值等于0为止，一旦探测到lock等于0，那么就设定该值为1，表示本thread持有该锁了，当然，这些操作要保证原子性，细节和exclusive版本的ldr和str（即ldrex和strexeq）相关，这里略过。立刻临界区后，持锁thread会调用\_\_raw_spin_unlock函数是否spin lock，其实就是把0这个数值赋给lock。

这个版本的spin lock的实现当然可以实现功能，而且在没有冲突的时候表现出不错的性能，不过存在一个问题：不公平。也就是所有的thread都是在无序的争抢spin lock，谁先抢到谁先得，不管thread等了很久还是刚刚开始spin。在冲突比较少的情况下，不公平不会体现的特别明显，然而，随着硬件的发展，多核处理器的数目越来越多，多核之间的冲突越来越剧烈，无序竞争的spinlock带来的performance issue终于浮现出来，根据Nick Piggin的描述：

> On an 8 core (2 socket) Opteron, spinlock unfairness is extremely noticable, with a userspace test having a difference of up to 2x runtime per thread, and some threads are starved or "unfairly" granted the lock up to 1 000 000 (!) times.

多么的不公平，有些可怜的thread需要饥饿的等待1000000次。本质上无序竞争从概率论的角度看应该是均匀分布的，不过由于硬件特性导致这么严重的不公平，我们来看一看硬件block：

[![lock](http://www.wowotech.net/content/uploadfile/201504/a5f79a99b31a8ce21c2454333653987220150422042122.gif "lock")](http://www.wowotech.net/content/uploadfile/201504/b2275647efecb80a93c00ab40d560be620150422042020.gif)

lock本质上是保存在main memory中的，由于cache的存在，当然不需要每次都有访问main memory。在多核架构下，每个CPU都有自己的L1 cache，保存了lock的数据。假设CPU0获取了spin lock，那么执行完临界区，在释放锁的时候会调用smp_mb invalide其他忙等待的CPU的L1 cache，这样后果就是释放spin lock的那个cpu可以更快的访问L1cache，操作lock数据，从而大大增加的下一次获取该spin lock的机会。

2、回到现在：arch_spinlock_t

ARM平台中的arch_spinlock_t定义如下（little endian）：

> typedef struct {\
> union {\
> u32 slock;\
> struct \_\_raw_tickets {\
> u16 owner;\
> u16 next;\
> } tickets;\
> };\
> } arch_spinlock_t;

本来以为一个简单的整数类型的变量就搞定的spin lock看起来没有那么简单，要理解这个数据结构，需要了解一些ticket-based spin lock的概念。如果你有机会去九毛九去排队吃饭（声明：不是九毛九的饭托，仅仅是喜欢面食而常去吃而已）就会理解ticket-based spin lock。大概是因为便宜，每次去九毛九总是无法长驱直入，门口的笑容可掬的靓女会给一个ticket，上面写着15号，同时会告诉你，当前状态是10号已经入席，11号在等待。

回到arch_spinlock_t，这里的owner就是当前已经入席的那个号码，next记录的是下一个要分发的号码。下面的描述使用普通的计算机语言和在九毛九就餐（假设九毛九只有一张餐桌）的例子来进行描述，估计可以让吃货更有兴趣阅读下去。最开始的时候，slock被赋值为0，也就是说owner和next都是0，owner和next相等，表示unlocked。当第一个个thread调用spin_lock来申请lock（第一个人就餐）的时候，owner和next相等，表示unlocked，这时候该thread持有该spin lock（可以拥有九毛九的唯一的那个餐桌），并且执行next++，也就是将next设定为1（再来人就分配1这个号码让他等待就餐）。也许该thread执行很快（吃饭吃的快），没有其他thread来竞争就调用spin_unlock了（无人等待就餐，生意惨淡啊），这时候执行owner++，也就是将owner设定为1（表示当前持有1这个号码牌的人可以就餐）。姗姗来迟的1号获得了直接就餐的机会，next++之后等于2。1号这个家伙吃饭巨慢，这是不文明现象（thread不能持有spin lock太久），但是存在。又来一个人就餐，分配当前next值的号码2，当然也会执行next++，以便下一个人或者3的号码牌。持续来人就会分配3、4、5、6这些号码牌，next值不断的增加，但是owner岿然不动，直到欠扁的1号吃饭完毕（调用spin_unlock），释放饭桌这个唯一资源，owner++之后等于2，表示持有2那个号码牌的人可以进入就餐了。

3、接口实现

同样的，这里也只是选择一个典型的API来分析，其他的大家可以自行学习。我们选择的是arch_spin_lock，其ARM32的代码如下：

> static inline void arch_spin_lock(arch_spinlock_t \*lock)\
> {\
> unsigned long tmp;\
> u32 newval;\
> arch_spinlock_t lockval;
>
> prefetchw(&lock->slock);－－－－－－－－－－－－－－－－－－－－－－－－（1）\
> __asm__ __volatile__(\
> "1:    ldrex    %0, \[%3\]\\n"－－－－－－－－－－－－－－－－－－－－－－－－－（2）\
> "    add    %1, %0, %4\\n"\
> "    strex    %2, %1, \[%3\]\\n"－－－－－－－－－－－－－－－－－－－－－－－－（3）\
> "    teq    %2, #0\\n"－－－－－－－－－－－－－－－－－－－－－－－－－－－－（4）\
> "    bne    1b"\
> : "=&r" (lockval), "=&r" (newval), "=&r" (tmp)\
> : "r" (&lock->slock), "I" (1 \<\< TICKET_SHIFT)\
> : "cc");
>
> while (lockval.tickets.next != lockval.tickets.owner) {－－－－－－－－－－－－（5）\
> wfe();－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－（6）\
> lockval.tickets.owner = ACCESS_ONCE(lock->tickets.owner);－－－－－－（7）\
> }
>
> smp_mb();－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－（8）\
> }

（1）和preloading cache相关的操作，主要是为了性能考虑

（2）将slock的值保存在lockval这个临时变量中

（3）将spin lock中的next加一

（4）判断是否有其他的thread插入。更具体的细节参考[Linux内核同步机制之（一）：原子操作](http://www.wowotech.net/kernel_synchronization/atomic.html)中的描述

（5）判断当前spin lock的状态，如果是unlocked，那么直接获取到该锁

（6）如果当前spin lock的状态是locked，那么调用wfe进入等待状态。更具体的细节请参考[ARM WFI和WFE指令](http://www.wowotech.net/armv8a_arch/wfe_wfi.html)中的描述。

（7）其他的CPU唤醒了本cpu的执行，说明owner发生了变化，该新的own赋给lockval，然后继续判断spin lock的状态，也就是回到step 5。

（8）memory barrier的操作，具体可以参考[memory barrier](http://www.wowotech.net/kernel_synchronization/memory-barrier.html)中的描述。

arch_spin_lock函数ARM64的代码（来自4.1.10内核）如下：

> static inline void arch_spin_lock(arch_spinlock_t \*lock)\
> {\
> unsigned int tmp;\
> arch_spinlock_t lockval, newval;
>
> asm volatile(\
> /\* Atomically increment the next ticket. */\
> "    prfm    pstl1strm, %3\\n"\
> "1:    ldaxr    %w0, %3\\n"－－－－－（A）－－－－－－－－－－－lockval = lock\
> "    add    %w1, %w0, %w5\\n"－－－－－－－－－－－－－newval ＝ lockval + (1 \<\< 16)，相当于next++\
> "    stxr    %w2, %w1, %3\\n"－－－－－－－－－－－－－－lock ＝ newval\
> "    cbnz    %w2, 1b\\n"－－－－－－－－－－－－－－是否有其他PE的执行流插入？有的话，重来。\
> /* Did we get the lock? */\
> "    eor    %w1, %w0, %w0, ror #16\\n"－－lockval中的next域就是自己的号码牌，判断是否等于owner\
> "    cbz    %w1, 3f\\n"－－－－－－－－－－－－－－－－如果等于，持锁进入临界区\
> /*\
> \* No: spin on the owner. Send a local event to avoid missing an\
> \* unlock before the exclusive load.\
> */\
> "    sevl\\n"\
> "2:    wfe\\n"－－－－－－－－－－－－－－－－－－－－否则进入spin\
> "    ldaxrh    %w2, %4\\n"－－－－（A）－－－－－－－－－其他cpu唤醒本cpu，获取当前owner值\
> "    eor    %w1, %w2, %w0, lsr #16\\n"－－－－－－－－－自己的号码牌是否等于owner？\
> "    cbnz    %w1, 2b\\n"－－－－－－－－－－如果等于，持锁进入临界区，否者回到2，即继续spin\
> /* We got the lock. Critical section starts here. \*/\
> "3:"\
> : "=&r" (lockval), "=&r" (newval), "=&r" (tmp), "+Q" (\*lock)\
> : "Q" (lock->owner), "I" (1 \<\< TICKET_SHIFT)\
> : "memory");\
> }

基本的代码逻辑的描述都已经嵌入代码中，这里需要特别说明的有两个知识点：

（1）Load-Acquire/Store-Release指令的应用。Load-Acquire/Store-Release指令是ARMv8的特性，在执行load和store操作的时候顺便执行了memory barrier相关的操作，在spinlock这个场景，使用Load-Acquire/Store-Release指令代替dmb指令可以节省一条指令。上面代码中的（A）就标识了使用Load-Acquire指令的位置。Store-Release指令在哪里呢？在arch_spin_unlock中，这里就不贴代码了。Load-Acquire/Store-Release指令的作用如下：

－Load-Acquire可以确保系统中所有的observer看到的都是该指令先执行，然后是该指令之后的指令（program order）再执行

－Store-Release指令可以确保系统中所有的observer看到的都是该指令之前的指令（program order）先执行，Store-Release指令随后执行

（2）第二个知识点是关于在arch_spin_unlock代码中为何没有SEV指令？关于这个问题可以参考ARM ARM文档中的Figure B2-5，这个图是PE（n）的global monitor的状态迁移图。当PE（n）对x地址发起了exclusive操作的时候，PE（n）的global monitor从open access迁移到exclusive access状态，来自其他PE上针对x（该地址已经被mark for PE（n））的store操作会导致PE（n）的global monitor从exclusive access迁移到open access状态，这时候，PE（n）的Event register会被写入event，就好象生成一个event，将该PE唤醒，从而可以省略一个SEV的指令。

注：

（1）+表示在嵌入的汇编指令中，该操作数会被指令读取（也就是说是输入参数）也会被汇编指令写入（也就是说是输出参数）。\
（2）=表示在嵌入的汇编指令中，该操作数会是write only的，也就是说只做输出参数。\
（3）I表示操作数是立即数

_原创文章，转发请注明出处。蜗窝科技_

_Change log：_

_1、2015/11/5，加入ARM64的代码实现部分的分析_

\__2、2015/11/17，增加ARM64代码中的两个知识点的描述_\
\_

标签: [spin](http://www.wowotech.net/tag/spin) [lock](http://www.wowotech.net/tag/lock) [自旋锁](http://www.wowotech.net/tag/%E8%87%AA%E6%97%8B%E9%94%81)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [蜗窝流量地域统计](http://www.wowotech.net/168.html) | [spin_lock最简单用法。](http://www.wowotech.net/164.html)»

**评论：**

**Harry Song**\
2024-01-13 21:34

文中对 ticket 的机制进行举例和实际的汇编实现有偏差：

"最开始的时候，slock被赋值为0，也就是说owner和next都是0，owner和next相等，表示unlocked。"

--> 实际上应该是默认值是1，第一位来排队的，先 next+1 = 1,然后 当前owner == 当前next，拿到了锁；如果不是第一位来排队的，那就是等待 owner何时 == 自己的next值；\
而不是来排队的先 next+1 把号码给后来的人。

我通过 ARM32 的arch_spin_lock 分析的，你看对不对。

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-8861)

**[bsp](http://www.wowotech.net/)**\
2023-01-31 16:31

ticket-based-spinlock有个问题，\
CPU0获取了spinlock，CPU1、CPU2、CPU3...在等锁时，CPU0通过spinunlock中的sev唤醒其它所有所有CPU去check是否轮到自己；既然我们通过ticket知道了lock->next是哪个CPU，其实只需要唤醒对应CPU上的task即可。\
也许这就是qspinlock应运而生的一个原因？

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-8734)

**icy_river**\
2023-04-06 09:56

@bsp：1. sev只有sev/sevl两种, 无法给指定core发event;\
2\. 同时自旋的core太多的话, cache同步开销就不能忽略了;\
3\. kernel 5.10, ticket spinlock已经换成qspinlock;

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-8765)

**[bsp](http://www.wowotech.net/)**\
2024-04-12 17:36

@icy_river：嗯，我其实就在说qspinlock 替代ticket-based-spinlock的原因。\
通过qspinlock，只需要唤醒lock->next所在的cpu，其它cpu继续自旋。

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-8881)

**[bsp](http://www.wowotech.net/)**\
2024-04-12 17:36

@icy_river：嗯，我其实就在说qspinlock 替代ticket-based-spinlock的原因。\
通过qspinlock，只需要唤醒lock->next所在的cpu，其它cpu继续自旋。

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-8882)

**[bsp](http://www.wowotech.net/)**\
2024-04-12 17:38

@icy_river：bsp\
2023-02-01 13:11\
ticket-based-spinlock有个重大问题：\
假如当CPU0获取了spinlock，而CPU1、CPU2、CPU3...在等锁，在CPU0 spin_unlock时通过sev/stlr唤醒其它所有所有CPU去check是否轮到自己；我们通过ticket可以知道lock->next是哪个CPU，其实只需要唤醒对应CPU上的task即可。\
于是qspinlock应运而生：\
1：当CPU0获取了spinlockA，而CPU1-->CPU2-->CPU3依次在等锁（lock->next是CPU1上的task）时，只需让CPU1自旋在spinlockA上（即qspinlock中的pending）、CPU2/CPU3自旋在percpu上的另外一把锁B（即qspinlock中的mcs锁），percpu的锁可以减少cache-boucing（具体怎么减 自行脑补）；\
2：当CPU0释放了spinlockA（armv8的stlr可以只唤醒对应地址上的waiter即CPU1，这点比armv7的sev好用），只会唤醒自旋在spinlockA上的CPU1，其它CPU则暂时不会被唤醒；\
3：同理CPU1获取了spinlockA，需要把把lock->next标记为CPU2，并让CPU2从自旋在percpu的spinlockB转换到spinlockA；即qspinlock中对应的mcs队列；\
4：同理CPU1释放了spinlockA，stlr（load-acquire/store-release）只会唤醒CPU2,；CPU2获取了spinlockA，把lock->next标记为CPU3，并让CPU3从自旋在percpu的spinlockB转换到spinlockA；即qspinlock中对应的mcs队列；\
依次往下，这便是qspinlock的中心思想吧。至于怎么设计和实现，我觉得有不同的方法。

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-8883)

**[dream](http://https//blog.csdn.net/lovemengx)**\
2022-07-22 23:23

linuxer:

看到这段话，有些不理解，还请解惑，非常感谢。

"A在进入临界区之前获取了spin lock，同样的，在A访问共享资源R的过程中发生了中断，中断唤醒了沉睡中的，优先级更高的B，B在访问临界区之前仍然会试图获取spin lock，这时候由于A进程持有spin lock而导致B进程进入了永久的spin……怎么破？linux的kernel很简单，在A进程获取spin lock的时候，禁止本CPU上的抢占（上面的永久spin的场合仅仅在本CPU的进程抢占本CPU的当前进程这样的场景中发生）。"

即使当前进程在持有锁的时候，被高优先级进程抢占也访问了锁，进入自旋状态，那么内核不是支持时间片轮转的调度嘛，当产生 tick 中断的时候，高优先级的进程时间片用完，那么低优先级的进程就有再次执行的机会，只要低优先级有执行的机会，自然就能完成临界区的代码执行，然后释放锁。释放锁，高优先级进程也就自然也能获取锁。

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-8649)

**Yaksama**\
2022-12-15 19:49

@dream：个人认为spinlock本身就是一种轻量级的锁，设计初衷除了考虑中断访问临界区资源的情况，还有一个就是避免过多进程上下文切换带来的开销。\
如果不disable preempt，的确拥有锁的线程早晚有得到执行的一天，但这种情况为什么不直接用mutex呢，而且线程获取spinlock之后执行的任务都是轻量级的，如果让其调度出去再调度回来才能执行完，这和disable preempt相比未免有些得不偿失。\
另外个人对调度器内容不太了解，但如果调度器本身也是个线程，可以获取spinlock的话...

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-8723)

**[bsp](http://www.wowotech.net/)**\
2023-01-31 15:16

@dream：B如果为rt线程，SCHED_FIFO时，是不考虑时间片的；

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-8733)

**icy_river**\
2023-04-06 10:00

@dream：在你假设的场景是,功能是没问题的; 但是这样B的时间片不是完全浪费了吗?

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-8766)

**ja**\
2024-08-14 02:57

@dream：我看完這段也有相同的想法，引用 @dream 最後一段的說法

即使当前进程在持有锁的时候，被高优先级进程抢占也访问了锁，进入自旋状态，那么内核不是支持时间片轮转的调度嘛，当产生 tick 中断的时候，高优先级的进程时间片用完，那么低优先级的进程就有再次执行的机会，只要低优先级有执行的机会，自然就能完成临界区的代码执行，然后释放锁。释放锁，高优先级进程也就自然也能获取锁

這樣B进程不會进入"永久"的spin，而只是暫時的spin，使用"永久"這個詞是因為上述做法會因為某些原因而無法達成嗎?

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-8922)

**linuxxx**\
2021-01-30 18:38

linuxer你好:\
请教一个问题:进程上下文中在临界区有cooy_to_user的操作,中断上下文也需要写同样的buffer, 使用了copy_to_user就无法使用spinlock,但是中断里锁机制只能使用spinlock该如何解决这种冲突呢?谢谢

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-8192)

**[linuxer](http://www.wowotech.net/)**\
2022-02-23 07:47

@linuxxx：建议不要这么设计软件，可以考虑中断中queue一个work，然后在其callback中使用copy_to_user

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-8560)

**[bsp](http://www.wowotech.net/)**\
2020-11-27 10:30

（1）和preloading cache相关的操作，主要是为了性能考虑\
做一点补充：\
prefetch mem 指令其工作主要是通知memory系统（cache line file）在后台执行，然后立马返回，而不会阻塞在等待prefetch完成后（linefill complete））才返回，所以会有效加速性能（10%以上）。\
from: https://developer.arm.com/documentation/100403/0200/functional-description/level-1-memory-system/data-prefetching\
The Cortex-A75 core supports the AArch64 Prefetch Memory (PRFM) instructions and the AArch32 Prefetch Data (PLD) and Preload Data With Intent To Write (PLDW) instructions. These instructions signal to the memory system that memory accesses from a specified address are likely to occur soon. The memory system acts by taking actions that aim to reduce the latency of the memory access when they occur. PRFM instructions perform a lookup in the cache, and if they miss and are to a cacheable address, a linefill starts. However, the PRFM instruction retires when its linefill is started, rather than waiting for the linefill to complete. This enables other instructions to execute while the linefill continues in the background.

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-8146)

**linuxzz**\
2020-08-14 09:48

大神，加了你们的官方QQ群，求通过，一起学习

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-8096)

**crash**\
2019-08-20 08:51

spin_lock获取不到锁资源spin等待期间会被中断切走吗

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-7597)

**[linuxer](http://www.wowotech.net/)**\
2022-02-23 07:50

@crash：会的，spin lock在spin的时候其实是处于WFE状态，中断来了之后，cpu会从WFE状态解除，从而跳转到中断上下文去执行。

当然，前提是你的spin lock是单纯的spin lock，没有附件disable interrupt

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-8561)

**cp**\
2019-07-26 16:32

请教一个问题\
T1                   T2\
next +1 | owner

next+1 | owner+1\
WFE\
这种情况下，T1如何才能被唤醒？

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-7552)

**cp**\
2019-07-26 16:39

@cp：意思就是  一个cpu的SEV 在另一个CPU的next++ 和WFE之间完成会怎么样呢？\
多谢

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-7553)

**[bsp](http://www.wowotech.net/)**\
2020-10-20 15:13

@cp：这就是linuxer说的：有其它thread/PE的插入，strex会执行失败，重新ldrex。

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-8126)

**白白**\
2020-10-20 15:30

@bsp：大佬有没有联系方式 想咨询一下EMMC读写方面的问题

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-8127)

**randy**\
2019-01-30 17:33

想请问一个arm32实现版本的spin unlock的问题，我注意到spin_unlock的实现中仅仅是owner++，没有ldrex/strex的实现，请问是否有问题:\
cpu0             cpu1\
ldrex\
next | owner+1

next+1 | owner

strex\
所以在cpu0完成strex后，把老的owner值覆盖回去了，cpu1的对owner++操作被覆盖，关键在于unlock操作没有原子，请问Linux内核为何没有问题？

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-7168)

**[smcdef](http://www.wowotech.net/)**\
2019-01-31 11:07

@randy：你可以看到在ARM64上，arch_spin_unlock()同样也是类似owner++的操作。我就根据ARM64的认知，推测以下ARM32的原理吧。

CPU0 执行ldrex后，monitor的状态是Exclusive Access。当CPU1把owner写成功以后，就会导致monitor的状态切换到Open Access。然后，CPU0的strex操作就会失败。因此，不会出现你担心的情况。

虽然owner++是RMW操作，但是由于spin_unlock()在任意时刻都只有一个进程调用。spin_unlock()和spin_lock()之间也是安全的。

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-7169)

**randy**\
2019-01-31 11:24

@smcdef：我理解普通的ldr/str指令不会导致open/exclusive状态机的切换，除非把Unlock的实现为ldrex/strex指令对，而非普通的owner++。

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-7170)

**[smcdef](http://www.wowotech.net/)**\
2019-01-31 11:31

@randy：在ARM64上，普通的str操作也会clear monitor。你既然说不会，可能你已经查阅了相关ARM的文档。我只是说出了ARM64的设计，供你参考。因为，我确实不了解ARM的设计（一直在搞ARM64）。

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-7171)

**randy**\
2019-02-01 13:57

@smcdef：sorry,你的理解是对的，重新读了下文档，arm32下str指令也会导致exclusive->open态的切换，我之前一直理解为只有strex这样的指令才会，多谢！

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-7176)

**fy**\
2017-11-02 19:58

hi 说到spin lock\
说一个老生常谈的问题吧。\
如果一个临界区X是有进程上下文和中断上下文都会访问的。那按理 spin lock应该做到\
1.禁止本地中断 2.禁止抢占 3.自旋

那又由于内核抢占真正能运行的前提是 从一个中断返回时(A) 并且当前允许抢占(B)并且当前中断开启（C)。。。\
因为上面1.中已经禁止本地中断了，即使2.禁止抢占不做也没关系了。\
我个人觉得确实是不需要2.禁止抢占的。只要1禁止本地中断和3.自旋 一起就可以达到保护临界区X\
如果真的不加上 2.禁止抢占，会不会有问题，我想了一些case，暂时没想到。\
当然，我是默认临界区X内不会主动放弃CPU为前提的。（临界区也确实不应该睡眠才是)

不知道linuxer对此什么看法。

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-6165)

**[wowo](http://www.wowotech.net/)**\
2017-11-06 17:07

@fy：我有点看不太明白这个问题，不过可以试着说一下我的理解：\
spin lock有两大类：\
一类是spin_lock，做的事情就是禁止抢占+自旋，这样可以保护那些没有中断参与的临界区，代价也小一些。\
一类是spin_lock_irqsave，做的事情是禁止中断+（禁止抢占+自旋），这样在第一类的基础上也可以保护有中断参与的临界区。\
你好像在质疑第二类中禁止抢占的必要性，确实，没什么必要。可spin_lock_irqsave要调用spin_lock啊，再封出来一个接口？岂不是有点画蛇添足了？

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-6175)

**fy**\
2017-11-11 16:08

@wowo：是的。我只是就这里的“禁止抢占”非必要，问问wowo的个人理解。\
确实是非必要的。\
非常感谢。

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-6190)

**LOONG**\
2017-11-29 13:10

@fy：前来膜拜郭大侠:)\
那天看文嘉转了郭大侠在另一个群里讨论这个问题（关抢占有没有必要），觉得受益匪浅，学习了。\
当时就想：或者作者当初也并不是一气呵成呢？这样定然有commit log可以查询。试着搜了一下，发现目前的git库已经没有相关记录了。于是上网搜到了这个问题：\
https://stackoverflow.com/questions/13263538/linux-kernel-spinlock-smp-why-there-is-a-preempt-disable-in-spin-lock-irq-sm\
基本跟郭大侠分析的一样:)\
不知@fy有何理解？或许可以直接问Dave Miller，说不定他跟郭大侠一样nice:)

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-6259)

**[linuxer](http://www.wowotech.net/)**\
2017-11-30 08:38

@LOONG：多谢支持！问题是一样的问题，但是在stackoverflow上也没有得到回答，不过在前几天的微信群里，这里问题已经讨论的比较清楚了。

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-6261)

**[linuxer](http://www.wowotech.net/)**\
2017-11-30 08:50

@fy：非常好的问题，非常精彩的思考点。

禁止了中断的确等于了禁止抢占，但是并不意味着它们两个完全等同，因为在preempt disable－－－preempt enable这个的调用过程中，在打开抢占的时候有一个抢占点，内核控制路径会在这里检查抢占，如果满足抢占条件，那么会立刻调度schedule函数进行进程切换，但是local irq disable－－－local irq enable的调用中，并没有显示的抢占检查点，当然，中断有点特殊，因为一旦打开中断，那么pending的中断会进来，并且在返回中断点的时候会检查抢占，但是也许下面的这个场景就无能为力了。进程上下文中调用如下序列：\
（1）local irq disable\
（2）wake up high level priority task\
（3）local irq enable\
当唤醒的高优先级进程被调度到本CPU执行的时候，按理说这个高优先级进程应该立刻抢占当前进程，但是这个场景无法做到。在调用try_to_wake_up的时候会设定need resched flag并检查抢占，但是由于中断disable，因此不会立刻调用schedule，但是在step （3）的时候，由于没有检查抢占，这时候本应立刻抢占的高优先级进程会发生严重的调度延迟.....直到下一个抢占点到来。

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-6262)

**[wowo](http://www.wowotech.net/)**\
2017-11-30 10:06

@linuxer：这个场景还是挺有意思的，不过话说回来了，spin_lock_xxx的目的是保护临界区，关preempt的抢占点什么事呢？爱抢占不抢占啊～\
本质上还是spin_lock需要通过开关抢占进行临界区保护，到spin_lock_irqxxx的时候，顺便做一下，再顺便帮preempt一个忙而已。\
所以这个场景并不能构成“spin_lock_irqsave关抢占是否有必要的理由”，最后的答案还是没必要（因为“保护”的目的已经达到），至于是不是可以增加一个抢占点，在local_irq_restore的时候，调用一下preempt_check_resched岂不是更直接？

[回复](http://www.wowotech.net/kernel_synchronization/spinlock.html#comment-6264)

1 [2](http://www.wowotech.net/kernel_synchronization/spinlock.html/comment-page-2#comments) [3](http://www.wowotech.net/kernel_synchronization/spinlock.html/comment-page-3#comments)

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

  - [GIC驱动代码分析（废弃）](http://www.wowotech.net/irq_subsystem/gic-irq-chip-driver.html)
  - [Linux电源管理(7)\_Wakeup events framework](http://www.wowotech.net/pm_subsystem/wakeup_events_framework.html)
  - [Linux调度器：用户空间接口](http://www.wowotech.net/process_management/scheduler-API.html)
  - [schedutil governor情景分析](http://www.wowotech.net/process_management/schedutil_governor.html)
  - [计算机科学基础知识（四）: 动态库和位置无关代码](http://www.wowotech.net/basic_subject/pic.html)

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

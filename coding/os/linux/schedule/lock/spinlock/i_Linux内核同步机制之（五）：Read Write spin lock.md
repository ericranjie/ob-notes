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

作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2015-5-22 18:38 分类：[内核同步机制](http://www.wowotech.net/sort/kernel_synchronization)

一、为何会有rw spin lock？

在有了强大的spin lock之后，为何还会有rw spin lock呢？无他，仅仅是为了增加内核的并发，从而增加性能而已。spin lock严格的限制只有一个thread可以进入临界区，但是实际中，有些对共享资源的访问可以严格区分读和写的，这时候，其实多个读的thread进入临界区是OK的，使用spin lock则限制一个读thread进入，从而导致性能的下降。

本文主要描述RW spin lock的工作原理及其实现。需要说明的是[Linux内核同步机制之（四）：spin lock](http://www.wowotech.net/kernel_synchronization/spinlock.html)是本文的基础，请先阅读该文档以便保证阅读的畅顺。

二、工作原理

1、应用举例

我们来看一个rw spinlock在文件系统中的例子：

> static struct file_system_type *file_systems;  
> static DEFINE_RWLOCK(file_systems_lock);

linux内核支持多种文件系统类型，例如EXT4，YAFFS2等，每种文件系统都用struct file_system_type来表示。内核中所有支持的文件系统用一个链表来管理，file_systems指向这个链表的第一个node。访问这个链表的时候，需要用file_systems_lock来保护，场景包括：

（1）register_filesystem和unregister_filesystem分别用来向系统注册和注销一个文件系统。

（2）fs_index或者fs_name等函数会遍历该链表，找到对应的struct file_system_type的名字或者index。

这些操作可以分成两类，第一类就是需要对链表进行更新的动作，例如向链表中增加一个file system type（注册）或者减少一个（注销）。另外一类就是仅仅对链表进行遍历的操作，并不修改链表的内容。在不修改链表的内容的前提下，多个thread进入这个临界区是OK的，都能返回正确的结果。但是对于第一类操作则不然，这样的更新链表的操作是排他的，只能是同时有一个thread在临界区中。

2、基本的策略

使用普通的spin lock可以完成上一节中描述的临界区的保护，但是，由于spin lock的特定就是只允许一个thread进入，因此这时候就禁止了多个读thread进入临界区，而实际上多个read thread可以同时进入的，但现在也只能是不停的spin，cpu强大的运算能力无法发挥出来，如果使用不断retry检查spin lock的状态的话（而不是使用类似ARM上的WFE这样的指令），对系统的功耗也是影响很大的。因此，必须有新的策略来应对：

我们首先看看加锁的逻辑：

（1）假设临界区内没有任何的thread，这时候任何read thread或者write thread可以进入，但是只能是其一。

（2）假设临界区内有一个read thread，这时候新来的read thread可以任意进入，但是write thread不可以进入

（3）假设临界区内有一个write thread，这时候任何的read thread或者write thread都不可以进入

（4）假设临界区内有一个或者多个read thread，write thread当然不可以进入临界区，但是该write thread也无法阻止后续read thread的进入，他要一直等到临界区一个read thread也没有的时候，才可以进入，多么可怜的write thread。

unlock的逻辑如下：

（1）在write thread离开临界区的时候，由于write thread是排他的，因此临界区有且只有一个write thread，这时候，如果write thread执行unlock操作，释放掉锁，那些处于spin的各个thread（read或者write）可以竞争上岗。

（2）在read thread离开临界区的时候，需要根据情况来决定是否让其他处于spin的write thread们参与竞争。如果临界区仍然有read thread，那么write thread还是需要spin（注意：这时候read thread可以进入临界区，听起来也是不公平的）直到所有的read thread释放锁（离开临界区），这时候write thread们可以参与到临界区的竞争中，如果获取到锁，那么该write thread可以进入。

三、实现

1、通用代码文件的整理

rw spin lock的头文件的结构和spin lock是一样的。include/linux/rwlock_types.h文件中定义了通用rw spin lock的基本的数据结构（例如rwlock_t）和如何初始化的接口（DEFINE_RWLOCK）。include/linux/rwlock.h。这个头文件定义了通用rw spin lock的接口函数声明，例如read_lock、write_lock、read_unlock、write_unlock等。include/linux/rwlock_api_smp.h文件定义了SMP上的rw spin lock模块的接口声明。

需要特别说明的是：用户不需要include上面的头文件，基本上普通spinlock和rw spinlock使用统一的头文件接口，用户只需要include一个include/linux/spinlock.h文件就OK了。

2、数据结构。rwlock_t数据结构定义如下：

> typedef struct {  
>     arch_rwlock_t raw_lock;  
> } rwlock_t;

rwlock_t依赖arch对rw spinlock相关的定义。

3、API

我们整理RW spinlock的接口API如下表：

|   |   |
|---|---|
|接口API描述|rw spinlock API|
|定义rw spin lock并初始化|DEFINE_RWLOCK|
|动态初始化rw spin lock|rwlock_init|
|获取指定的rw spin lock|read_lock  <br>write_lock|
|获取指定的rw spin lock同时disable本CPU中断|read_lock_irq  <br>write_lock_irq|
|保存本CPU当前的irq状态，disable本CPU中断并获取指定的rw spin lock|read_lock_irqsave  <br>write_lock_irqsave|
|获取指定的rw spin lock同时disable本CPU的bottom half|read_lock_bh  <br>write_lock_bh|
|释放指定的spin lock|read_unlock  <br>write_unlock|
|释放指定的rw spin lock同时enable本CPU中断|read_unlock_irq  <br>write_unlock_irq|
|释放指定的rw spin lock同时恢复本CPU的中断状态|read_unlock_irqrestore  <br>write_unlock_irqrestore|
|获取指定的rw spin lock同时enable本CPU的bottom half|read_unlock_bh  <br>write_unlock_bh|
|尝试去获取rw spin lock，如果失败，不会spin，而是返回非零值|read_trylock  <br>write_trylock|

在具体的实现面，如何将archtecture independent的代码转到具体平台的代码的思路是和spin lock一样的，这里不再赘述。

2、ARM上的实现

对于arm平台，rw spin lock的代码位于arch/arm/include/asm/spinlock.h和spinlock_type.h（其实普通spin lock的代码也是在这两个文件中），和通用代码类似，spinlock_type.h定义ARM相关的rw spin lock定义以及初始化相关的宏；spinlock.h中包括了各种具体的实现。我们先看arch_rwlock_t的定义：

> typedef struct {  
>     u32 lock;  
> } arch_rwlock_t;

毫无压力，就是一个32-bit的整数。从定义就可以看出rw spinlock不是ticket-based spin lock。我们再看看arch_write_lock的实现：

> static inline void arch_write_lock(arch_rwlock_t *rw)  
> {  
>     unsigned long tmp;
> 
>     prefetchw(&rw->lock); －－－－－－－知道后面需要访问这个内存，先通知hw进行preloading cache  
>     __asm__ __volatile__(  
> "1:    ldrex    %0, [%1]\n" －－－－－获取lock的值并保存在tmp中  
> "    teq    %0, #0\n" －－－－－－－－判断是否等于0  
>     WFE("ne") －－－－－－－－－－如果tmp不等于0，那么说明有read 或者write的thread持有锁，那么还是静静的等待吧。其他thread会在unlock的时候Send Event来唤醒该CPU的  
> "    strexeq    %0, %2, [%1]\n" －－－－如果tmp等于0，将0x80000000这个值赋给lock  
> "    teq    %0, #0\n" －－－－－－－－是否str成功，如果有其他thread在上面的过程插入进来就会失败  
> "    bne    1b" －－－－－－－－－如果不成功，那么需要重新来过，否则持有锁，进入临界区  
>     : "=&r" (tmp) －－－－％0  
>     : "r" (&rw->lock), "r" (0x80000000)－－－－－－－％1和％2  
>     : "cc");
> 
>     smp_mb(); －－－－－－－memory barrier的操作  
> }

对于write lock，只要临界区有一个thread进行读或者写的操作（具体判断是针对32bit的lock进行，覆盖了writer和reader thread），该thread都会进入spin状态。如果临界区没有任何的读写thread，那么writer进入临界区，并设定lock＝0x80000000。我们再来看看write unlock的操作：

> static inline void arch_write_unlock(arch_rwlock_t *rw)  
> {  
>     smp_mb(); －－－－－－－memory barrier的操作
> 
>     __asm__ __volatile__(  
>     "str    %1, [%0]\n"－－－－－－－－－－－恢复0值  
>     :  
>     : "r" (&rw->lock), "r" (0) －－－－－－－－％0和％1  
>     : "cc");
> 
>     dsb_sev();－－－－－－－memory barrier的操作加上send event，wakeup其他 thread（那些cpu处于WFE状态）  
> }

write unlock看起来很简单，就是一个lock＝0x0的操作。了解了write相关的操作后，我们再来看看read的操作：

> static inline void arch_read_lock(arch_rwlock_t *rw)  
> {  
>     unsigned long tmp, tmp2;
> 
>     prefetchw(&rw->lock);  
>     __asm__ __volatile__(  
> "1:    ldrex    %0, [%2]\n"－－－－－－－－获取lock的值并保存在tmp中  
> "    adds    %0, %0, #1\n"－－－－－－－－tmp = tmp + 1  
> "    strexpl    %1, %0, [%2]\n"－－－－如果tmp结果非负值，那么就执行该指令，将tmp值存入lock  
>     WFE("mi")－－－－－－－－－如果tmp是负值，说明有write thread，那么就进入wait for event状态  
> "    rsbpls    %0, %1, #0\n"－－－－－判断strexpl指令是否成功执行  
> "    bmi    1b"－－－－－－－－－－如果不成功，那么需要重新来过，否则持有锁，进入临界区  
>     : "=&r" (tmp), "=&r" (tmp2)－－－－－－－－－－％0和％1  
>     : "r" (&rw->lock)－－－－－－－－－－－－－－－％2  
>     : "cc");
> 
>     smp_mb();  
> }

上面的代码比较简单，需要说明的是adds指令更新了状态寄存器（指令中s那个字符就是这个意思），strexpl会根据adds指令的执行结果来判断是否执行。pl的意思就是positive or zero，也就是说，如果结果是正数或者0（没有thread在临界区或者临界区内有若干read thread），该指令都会执行，如果是负数（有write thread在临界区），那么就不执行。OK，最后我们来看read unlock的函数：

> static inline void arch_read_unlock(arch_rwlock_t *rw)  
> {  
>     unsigned long tmp, tmp2;
> 
>     smp_mb();
> 
>     prefetchw(&rw->lock);  
>     __asm__ __volatile__(  
> "1:    ldrex    %0, [%2]\n"－－－－－－－－获取lock的值并保存在tmp中  
> "    sub    %0, %0, #1\n"－－－－－－－－tmp = tmp - 1  
> "    strex    %1, %0, [%2]\n"－－－－－－将tmp值存入lock中  
> "    teq    %1, #0\n"－－－－－－是否str成功，如果有其他thread在上面的过程插入进来就会失败  
> "    bne    1b"－－－－－－－如果不成功，那么需要重新来过，否则离开临界区  
>     : "=&r" (tmp), "=&r" (tmp2)－－－－－－－－－－－－％0和％1  
>     : "r" (&rw->lock)－－－－－－－－－－－－－－－－－％2  
>     : "cc");
> 
>     if (tmp == 0)  
>         dsb_sev();－－－－－如果read thread已经等于0，说明是最后一个离开临界区的reader，那么调用sev去唤醒WFE的cpu core  
> }

最后，总结一下：

[![rwspinlock](http://www.wowotech.net/content/uploadfile/201505/90ea73866588062607b2d2a642d64dd820150522103709.gif "rwspinlock")](http://www.wowotech.net/content/uploadfile/201505/93739f21f48bca7eedf618e87569d53b20150522103501.gif)

32个bit的lock，0～30的bit用来记录进入临界区的read thread的数目，第31个bit用来记录write thread的数目，由于只允许一个write thread进入临界区，因此1个bit就OK了。在这样的设计下，read thread的数目最大就是2的30次幂减去1的数值，超过这个数值就溢出了，当然这个数值在目前的系统中已经足够的大了，姑且认为它是安全的吧。

四、后记

read/write spinlock对于read thread和write thread采用相同的优先级，read thread必须等待write thread完成离开临界区才可以进入，而write thread需要等到所有的read thread完成操作离开临界区才能进入。正如我们前面所说，这看起来对write thread有些不公平，但这就是read/write spinlock的特点。此外，在内核中，已经不鼓励对read/write spinlock的使用了，RCU是更好的选择。如何解决read/write spinlock优先级问题？RCU又是什么呢？我们下回分解。

  

_原创文章，转发请注明出处。蜗窝科技，www.wowotech.net_

[http://www.wowotech.net/kernel_synchronization/rw-spinlock.html](http://www.wowotech.net/kernel_synchronization/rw-spinlock.html)

标签: [rw](http://www.wowotech.net/tag/rw) [spinlock](http://www.wowotech.net/tag/spinlock)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [msm8994 热插拔sim卡导致modem重新启动的流程](http://www.wowotech.net/190.html) | [Linux时间子系统之（十四）：tick broadcast framework](http://www.wowotech.net/timer_subsystem/tick-broadcast-framework.html)»

**评论：**

**garystone**  
2020-03-04 20:46

上文写到"read thread的数目最大就是2的30次幂减去1的数值"，read thread的数目最多应该是2的31次幂减去1吧。

[回复](http://www.wowotech.net/kernel_synchronization/rw-spinlock.html#comment-7904)

**crash**  
2018-11-14 13:07

arch_write_lock中的这段汇编不太理解：WFE("ne") －－－－－－－－－－如果tmp不等于0，那么说明有read 或者write的thread持有锁，那么还是静静的等待吧。其他thread会在unlock的时候Send Event来唤醒该CPU的  
  
唤醒后不是应该跳到"1:    ldrex    %0, [%2]\n"重新获取lock新值吗，为什么代码中没有跳转动作，醒来后直接往下走了

[回复](http://www.wowotech.net/kernel_synchronization/rw-spinlock.html#comment-7034)

**CXK**  
2022-08-23 13:59

@crash：ldrex本身是一个原子读指令，如果在这个期间这块内存被修改了，那么，就算醒来以后执行的strex指令也是会失败，也会重新跳转到ldrex，这段代码利用了原子操作的机制来保证不会出错。

[回复](http://www.wowotech.net/kernel_synchronization/rw-spinlock.html#comment-8669)

**[heziq](http://www.wowotech.net/)**  
2015-05-25 09:55

大哥，你终于放大招了，我等了好久了。回去研究一下。

[回复](http://www.wowotech.net/kernel_synchronization/rw-spinlock.html#comment-1860)

**[wowo](http://www.wowotech.net/)**  
2015-05-22 21:48

怪不得这么久没动静，原来在憋大招啊，哈哈~~  
记得添“原创文章，转发请注明出处。蜗窝科技，www.wowotech.net。”的链接哈，这对google排名很有帮助。

[回复](http://www.wowotech.net/kernel_synchronization/rw-spinlock.html#comment-1854)

**[linuxer](http://www.wowotech.net/)**  
2015-05-22 23:17

@wowo：哪里，这份文档和spinlock一起写的，几乎快完成了，但是我又开始写broadcast tick framework那份文档，一发不可收拾，就没有回来写这份，昨天，终于完成了broadcast tick framework（这份文档是真正耗费精力的），所以，今天中午，简单改一下，rwspinlock就OK了。内核同步中，RCU才是真正的大招，我估计得憋会儿  
  
---------------  
BTW，为何添加“原创文章，转发请注明出处。蜗窝科技，www.wowotech.net。”的链接会对google排名很有帮助？

[回复](http://www.wowotech.net/kernel_synchronization/rw-spinlock.html#comment-1856)

**[wowo](http://www.wowotech.net/)**  
2015-05-23 10:16

@linuxer：这种深奥的事情，我也不知道为什么，大家都是这么说的，可能谷歌对原创、知识产权比较在意。

[回复](http://www.wowotech.net/kernel_synchronization/rw-spinlock.html#comment-1857)

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
    
    - [一个较复杂dcache问题](http://www.wowotech.net/linux_kenrel/484.html)
    - [致驱动工程师的一封信](http://www.wowotech.net/device_model/429.html)
    - [/proc/meminfo分析（一）](http://www.wowotech.net/memory_management/meminfo_1.html)
    - [关于numa loadbance的死锁分析](http://www.wowotech.net/linux_kenrel/482.html)
    - [ARM64的启动过程之（一）：内核第一个脚印](http://www.wowotech.net/armv8a_arch/arm64_initialize_1.html)
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
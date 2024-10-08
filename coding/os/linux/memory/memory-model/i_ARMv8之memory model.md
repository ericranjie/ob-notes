作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2016-5-18 12:09 分类：[ARMv8A Arch](http://www.wowotech.net/sort/armv8a_arch)
# 一、前言

从阅读ARMv8手册的第一天起，我就饱受memory order、memory barrier、coherent、consistency等概念的残害，各种痛苦，各种迷茫，各种试图放弃，各种欲罢不能……，现在，终于收拾心情，再次出发，希望这次能把近期关于ARMv8上的memory model相关的知识点整理出来，让自己更清楚一些，也顺便希望能够和大家一起探讨。

本文主要关注shared-memory system，其他的系统不在本文的考虑范围。
# 二、什么是memory model？

想要理解ARMv8的memory model，首先需要知道什么是memory model，或者说memory consistency model。

当cpu从memory中的某个位置发起一次读操作的时候，该操作的返回值应该是什么样子的呢？对于程序员，直觉就是当然返回上次写入的数值了。不过，怎么定义“上次”呢？对于单核的执行环境，“上次”比较容易定义，从program order中可以轻易的得出上一次对该地址的写入操作。对于share-memory的多核环境而言，如何定义“上次”呢？那么多Hardware Thread在并发执行程序（每个cpu core上都有自己的指令流，都有自己的program order），想找出“上次”还不是一个显而易见的事情啊（当从两个不同CPU core上发起了对同一个memory location的访问，谁先谁后是无法通过program order来定义的）。为了搞清楚这个问题，我们必须要引入memory consistency model这个概念。所谓memory consistency model，其实就是就是定义系统中对内存访问需要遵守的规则（本质上就是read操作返回什么样的值），了解了这些规则，软件和硬件才能和谐工作，软件工程师才能写出逻辑正确的程序。

memory consistency model的规则影响了一大票人：使用高级语言的程序员、写编译器的软件工程师、CPU设计人员。做为一个程序员，一个有情怀的c程序员，我期望的memory model符合c语言的基本逻辑，内存操作是按照c程序中的program order进行的，而且是有序的进行，符合c程序的逻辑推导的，一言以蔽之，程序员当然希望能够编程简单，不要考虑什么memory order啊，memory barrier什么的。不过，从系统设计人员的角度看呢，需要找到一个各方面都OK的memory model。你看，系统设计人员就是高瞻远瞩啊，可以从多方面考虑，在性能和编程易用性方面平衡。
# 三、什么是sequential consistency？

我们从ARMv8的手册中知道它的memory consistency model是属于Relaxed Memory Consistency Models，这里的“Relaxed”是针对sequential consistency而言的，因此我们先解释什么是sequential consistency，定义如下：

> “the result of any execution is the same as if the operations of all the processors were executed in some sequential order, and the operations of each individual processor appear in this sequence in the order specified by its program”.

通俗的讲，如果shared memory multicore系统符合sequential consistency模型，那么在多个cpu core上执行的程序就好象在一个单一cpu core上顺序执行一样。例如如果系统有两个cpu，分别是core 0和core 1，core 0执行了A B C D内存访问指令，core 1执行了a b c d内存访问指令，对于程序员而言，sequential consistency的系统上执行这8条指令的效果就好象在一个core上顺序执行了A a B b C c D d的指令流。当然，上面只是给出一个示例的程序执行流，实际上也可以是A B a b C c D d，也可以是A  B  C  D a b c d，也可以是……只要你熟悉排列组合的原理，就能把所有可能的顺序罗列出来。当然，也不是任意的排组合都OK，因为这个综合效果的指令流必须符合core 0的program order（即单独从core 0的角度看，其program order必须是A->B->C->D），必须符合core 1的program order（即单独从core 1的角度看，其program order必须是a->b->c->d）。我们可以总结sequential consistency模型的两条规则：

（1）各个cpu core按照其program order来执行程序，执行完一条，然后启动下一条指令的执行，不会有local reordering。为了理解这条规则，我们来看看著名的Dekker algorithm，代码如下：

|   |   |
|---|---|
|core 0|core 1|
|flag0 = 1;  <br>if (flag1 == 0)  <br>    enter critical section|flag1 = 1;  <br>if (flag0 == 0)  <br>    enter critical section|

上面代码的初始条件是flag0 = flag1 = 0。在core 0上，当想要进入临界区的时候，会设定flag0等于1，并且判断flag1的值是否等于0，如果等于0，则说明core 1没有进入临界区，还没有执行flag1＝1这一条指令，同时也就隐含了flag0=1的赋值已经执行完毕。对于sequential consistency模型，各个CPU core能够保证自己的local program order被严格执行，从而保证了不会有多个thread同时进入临界区。

（2）从全局来看，每一个写的操作，都被系统中的所有cpu core同时感知到。注意上面句子中的“同时”，这也就意味着cpu 系统（包括所有的cpu core们）和存储系统之间有一个开关，一次只会连接一个CPU core和存储系统，因此对memory的访问都是原子的、串行化的。例如：某个cpu core对某个内存操作进行write操作，该操作完成之后，那个开关才会放下一个内存操作进入存储系统。我们用下面的代码示例来解释这一条规则：

|   |   |   |
|---|---|---|
|core 0|core 1|core 2|
|A = 1;|if (A == 1)  <br>    B = 1;|if (B == 1)  <br>    load A|

上面代码的初始条件是A = B = 0。对于sequential consistency模型，我们可以进行下面的推导：对于core 1，如果该处理器执行了B=1的赋值语句的时候，说明core 1已经感知到了core 0上对A变量的write操作，因此core 2必须也知道了A等于1这个事实。因此，在core 2上执行load A操作的时候，这时候A值必须要等于1。

程序员当然喜欢简单、直观的sequential consistency模型，但是，这也限制了CPU硬件和编译器的优化，从而影响了整个系统的性能。怎么办？答案是显而易见的，云计算、大数据等等这些真实用户的需求（要求更好的performance）无情的碾压了程序员的“小确幸”，因此，骚年们，直面“惨淡的人生”吧，这也就是relaxed consistency model。

四、什么是relaxed consistency model?

通过上一节，我们了解到sequential consistency模型中的各种要求，这些对内存操作顺序的要求实际上是对性能的限制和束缚，为了性能，能不能放松（relax）这些要求呢？当然没有问题，否则也就不会有relaxed consistency model的商业化应用了。这些发送内存操作的要求可以被分成两个类别：

（1）CPU core是否可以不按照program order的顺序来执行代码？进一步又可以分成RAW（Read after Write），WAW（write after write）和WAR（write after read）三种情况。需要特别强调的是这些内存操作对（RW，WR或者WW）必须操作不同的memory location，否则，这些操作是有数据依赖关系，CPU必须保证起操作顺序，否则在单CPU core上的程序逻辑就不对了。对于ARMv8而言，如果没有memory barrier指令，program order中的RAW，WAW和WAR的内存操作对都可以被reorder。

（2）CPU core的write操作是否必须是全局原子性的。对于sequential consistency，一个写的动作，会被其他CPU core同时观察到，即要么全部观察到新值，要么全部观察到旧值，不会有中间情况的存在。有的内存模型可以允许某个cpu core的read操作返回其他cpu core写入的新值，而其他的cpu core仍然观察到旧值。这里实际上说的是Multi-copy atomicity，对于ARMv8而言，normal memory的写入操作不具备Multi-copy atomicity的特性。

当然，无论如何的放松要求，下面的要求不能放松：

（1）在单核上，程序中的数据依赖和控制依赖必须保证，即有依赖关系的内存操作必须保证操作顺序，不能out of order。

（2）对同一地址的memory location的写入操作必须是串行化的。如果系统中有cache，那么需要cache coherence protocol来保证这一点。
# 五、参考文献

1、Computer Architecture, A Quantitative Approach 5th
2、Shared Memory Consistency Models: A  Tutorial
3、Memory Consistency Models for Shared-Memory Multiprocessors
4、ARMv8 ARM

原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/)。

标签: [Model](http://www.wowotech.net/tag/Model) [ARMv8](http://www.wowotech.net/tag/ARMv8) [sequential](http://www.wowotech.net/tag/sequential) [consistency](http://www.wowotech.net/tag/consistency) [relaxed](http://www.wowotech.net/tag/relaxed)

---

« [玩转BLE(2)_使用bluepy扫描BLE的广播数据](http://www.wowotech.net/bluetooth/bluepy_scan.html) | [ARMv8之Atomicity](http://www.wowotech.net/armv8a_arch/atomicity.html)»

**评论：**

**zyqin**  
2019-06-10 15:23

写的好详细，目前有打算支持一个armv8架构的cortex-A53核到freertos上，还不知道从何入手

[回复](http://www.wowotech.net/armv8a_arch/memory-model.html#comment-7463)

**ARMer**  
2017-06-18 10:08

最近ARMv8 memory model重新定义了 other-multi-copy atomic

[回复](http://www.wowotech.net/armv8a_arch/memory-model.html#comment-5687)

**綠色時間**  
2016-12-22 17:38

有的内存模型可以允许某个cpu core的read操作返回其他cpu core写入的新值，而其他的cpu core仍然观察到旧值。 <- 請問什麼情況會允許這種現象發生呢? 一般而言不都會想要一起取得新值或一起取得舊值嗎? 有人新有人舊不就不一致了嗎?

[回复](http://www.wowotech.net/armv8a_arch/memory-model.html#comment-5057)

**Codingbelief**  
2016-05-21 00:17

好过瘾的文字，一直都是看着这些术语，但是没有认真去思考和深入理解其本质~  
Linuxer 的一词一句都在提醒我：不要在思考上怠惰，不要在疑问上妥协，哈哈~~

[回复](http://www.wowotech.net/armv8a_arch/memory-model.html#comment-3962)

**[linuxer](http://www.wowotech.net/)**  
2016-05-23 17:03

@Codingbelief：我的目标是看懂ARMv8的手册，越是困难的东西，征服后就越有快感，与Codingbelief同学共勉。当然，现在我也还不是非常理解，不过可以慢慢来，现在对CPU architecture越来越着迷了，不可否认，驱动的趣味性的确是小了一点......  
  
BTW，有空研究一下ARMv8的地址翻译过程，可以给大家讲讲啊。

[回复](http://www.wowotech.net/armv8a_arch/memory-model.html#comment-3964)

**Codingbelief**  
2016-05-31 02:59

@linuxer：越接近本源的东西都越是迷人的~~  
驱动的趣味性的话，估计大部分来自各种意想不到的技巧上了~  
D4 已经基本过了一遍了~  但是写作比翻译慢多了 ~~  
这周应该会先发 eMMC 的内容，估计 wowo 的 boot 很快就要盯上这块了。

[回复](http://www.wowotech.net/armv8a_arch/memory-model.html#comment-4006)

**[linuxer](http://www.wowotech.net/)**  
2016-05-31 11:37

@Codingbelief：好啊！期待中......

[回复](http://www.wowotech.net/armv8a_arch/memory-model.html#comment-4008)

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
    
    - [u-boot启动流程分析(1)_平台相关部分](http://www.wowotech.net/u-boot/boot_flow_1.html)
    - [linux kernel的中断子系统之（四）：High level irq event handler](http://www.wowotech.net/irq_subsystem/High_level_irq_event_handler.html)
    - [Linux kernel debug技巧----开启DEBUG选项](http://www.wowotech.net/linux_application/kernel_debug_enable.html)
    - [Binder从入门到放弃（上）](http://www.wowotech.net/linux_kenrel/binder1.html)
    - [Linux设备模型(8)_platform设备](http://www.wowotech.net/device_model/platform_device.html)
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
作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2017-11-9 22:37 分类：[内存管理](http://www.wowotech.net/sort/memory_management)
## 1. 前言

内存（memory）在Linux系统中是一种牵涉面极广的资源，上至应用程序、下至kernel和driver，无不为之魂牵梦绕。加上它天然的稀缺性，导致内存管理（Memory Management，简称MM）是linux kernel中非常重要又非常复杂的一个子系统。

重要性就不多说了，Kernel自有分寸。关于复杂性（鉴于Linux kernel优秀的抽象能力），应该不会被普通人（Linux系统的使用者、应用工程师、驱动工程师、轻量级的内核工程师）感知到才对。事实确实如此，Kernel屏蔽掉了大多数的实现细节，尽量以简单、易用的方式向其它模块提供memory服务。

不过呢，这个世界上没有完美的存在，kernel的内存管理也是如此，由于两方面的原因：一、众口难调，内存管理有关的需求实在太复杂了；二、CPU、Device和Memory之间纠结的三角恋（参考下面图片），导致它也（不得不）提供了很多啰里啰唆的、不易理解的功能（困扰了很多从入门级到资深级的linux软件工程师）。

![[Pasted image 20241008091330.png]]
图片1 CPU, Device and Memory

基于上面的原因，本站[内存管理子系统](http://www.wowotech.net/sort/memory_management)发布了很多分析文章，以帮助大家理解内存管理有关的概念。不过到目前为止，还缺少一篇索引类的文章，从整体出发，理解Kernel内存管理所需要面对的软硬件局面、所要解决的问题，以及各个内存管理子模块的功能和意义。这就是本文的目的。
## 2. 内存有关的需求总结

在嵌入式系统中，从需求的角度看内存，是非常简单的，可以总结为两大类（参考上面的图片1）：

1）CPU有访问内存的需求，包括从内存中取指、从内存中取数据、向内存中写数据。相应的数据流为：

> CPU<-------------->MMU(Optional)<----------->Memory

2）其它外部设备有访问内存的需求，包括从内存“读取”数据、向内存“写入”数据。这里的“读取”和“写入”加了引号，是因为在大部分情况下，设备不像CPU那样有智能，不具备直接访问内存的能力。总结来说，设备有三种途径访问内存：

> a）由CPU从中中转，数据流如下（本质上是CPU访问内存）：  
> Device<---------------->CPU<--------------------Memory
> 
> b）由第三方具有智能的模块（如DMA控制器）中转，数据流如下：  
> Device<----------->DMA Controller<--------->Memory
> 
> c）直接访问内存，数据流如下：  
> Device<---------->IOMMU(Optional)--------->Memory

那么Linux kernel的内存管理模块怎么理解并满足上面的需求呢？接下来我们将一一梳理。
## 3. 软件（Linux kernel内存管理模块）的角度看内存

#### 3.1 CPU视角

我们先从CPU的需求说起（以当前具有MMU功能的嵌入式Linux平台为例），看看会向kernel的内存管理模块提出哪些需求。

⬛ 看到内存

> 关于内存以及内存管理，最初始的需求是：Linux kernel的核心代码（主要包括启动和内存管理），要能看到物理内存。
> 
> 在MMU使能之前，该需求很容易满足。
> 
> 但MMU使能之后、Kernel的内存管理机制ready之前，Kernel看到的是虚拟地址，此时需要一些简单且有效的机制，建立虚拟内存到物理内存的映射（可以是部分的映射，但要能够满足kenrel此时的需要）。

⬛ 管理内存

> 看到内存之后，下一步就是将它们管理起来。根据不同的内存形态（在物理地址上是否连续、是否具有NUMA内存、是否具有可拔插的内存、等等），可能有不同的管理模型和管理方法。

⬛ 向内核线程/用户进程提供服务

> 将内存管理起来之后，就可以向其它人（kernel的其它模块、内核线程、用户空间进程、等等）提供服务了，主要包括：  
> ✓ 以虚拟地址（VA）的形式，为应用程序提供远大于物理内存的虚拟地址空间（Virtual Address Space）  
> ✓ 每个进程都有独立的虚拟地址空间，不会相互影响，进而可提供非常好的内存保护（memory protection）  
> ✓ 提供内存映射（Memory Mapping）机制，以便把物理内存、I/O空间、Kernel Image、文件等对象映射到相应进程的地址空间中，方便进程的访问  
> ✓ 提供公平、高效的物理内存分配（Physical Memory Allocation）算法  
> ✓ 提供进程间内存共享的方法（以虚拟内存的形式），也称作Shared Virtual Memory  
> ✓ 等等

⬛ 更为高级的内存管理需求

> 欲望是无止境的，在内存管理模块提供了基础的内存服务之后，Linux系统（包括kernel线程和用户进程）已经可以正常work了，更为高级的需求也产生了，例如：  
> ✓ 内存的热拔插（memory hotplug）  
> ✓ 内存的size超过了虚拟地址可寻址的空间怎么办（high memory）  
> ✓ 超大页（hugetlbpage）的支持  
> ✓ 利用磁盘作为交换页以扩大可用内存（各种swap机制和算法）  
> ✓ 在NUMA系统中通过移动物理页面位置的方法提升内存的访问效率（Page migration）  
> ✓ 内存泄漏的检查  
> ✓ 内存碎片的整理  
> ✓ 内存不足时的处理（oom kill机制）  
> ✓ 等等
#### 3.2 Device视角

正常情况下，当软件活动只需要CPU参与时（例如简单的数学运算、图像处理等），上面3.1 所涉及内容已经足够了，无论是用户空间程序，还是内核空间程序，都可以欢快的运行了。

不过，当软件操作一些特殊的、可以以自己的方式访问memory的硬件设备的时候，麻烦就出现了：软件通过CPU视角获得memory，并不能直接被这些硬件设备访问。于是这些硬件设备就提出了需求：

> 内存管理模块需要为这些设备提供一些特殊的获取内存的接口，这些接口可以按照设备所期望的形式组织内存（因而可以被设备访问），也可以重新映射成CPU视角的形式，以便CPU可以访问。
> 
> 这就是我们在编写设备驱动的时候会经常遇到的DMA mapping功能，其中DMA是Direct Memory Access的所需，表示（memory）可以被设备直接访问。

另外，在某些应用场景下，内存数据可能会在多个设备间流动，为了提高效率，不能为每个设备都提供一份拷贝，因此内存管理模块需要提供设备间内存共享（以及相互转换）的功能。

## 4. 软件结构

基于上面章节的需求，Linux kernel从虚拟内存（VM）、DMA mapping以及DMA buffer sharing三个角度，对内存进行管理，如下图所示：
![[Pasted image 20241008091516.png]]
图片2 VM、DMA mapping和DMA buffer sharing

其中VM是内存管理的主要模块，也是我们通常意义上所讲的狭义“内存管理”，代码主要分布在mm/以及arch/xxx/mm/两个目录下，其中arch/xxx/mm/*提供平台相关部分的实现，mm/*提供平台无关部分的实现。

DMA mapping是内存管理的辅助模块，注要提供dma_alloc_xxx（申请可供设备直接访问的内存----dma_addr）和dma_map_xxx（是在CPU视角的虚拟内存和dma_addr之间转换）两类接口。该模块的具体实现依赖于设备访问内存的方式，代码主要分别在drivers/base/*（通用实现）以及arch/xxx/mm/（平台相关的实现）。

最后是DMA buffer sharing的机制，用于在不同设备之间共享内存，一般包括两种方法：

> 传统的、利用CPU虚拟地址中转的方法，例如scatterlist；
> dma buffer sharing framework，位于drivers/dma-buf/dma-buf.c中。      

有关这些模块的具体描述,本文后续的文章将会一一展开(有些已经有了),这里就先结束吧.

原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/memory_management/concept.html)。

标签: [Linux](http://www.wowotech.net/tag/Linux) [Kernel](http://www.wowotech.net/tag/Kernel) [内核](http://www.wowotech.net/tag/%E5%86%85%E6%A0%B8) [内存管理](http://www.wowotech.net/tag/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86) [mm](http://www.wowotech.net/tag/mm) [概念](http://www.wowotech.net/tag/%E6%A6%82%E5%BF%B5)

---

« [逆向映射的演进](http://www.wowotech.net/memory_management/reverse_mapping.html) | [Linux kernel scatterlist API介绍](http://www.wowotech.net/memory_management/scatterlist.html)»

**评论：**

**jeff-lv**  
2020-07-20 15:37

请问下wowo大神，对于准备系统学习内存管理的人来说，可否给出一个大概的文章，推荐阅读顺序呢？

[回复](http://www.wowotech.net/memory_management/concept.html#comment-8056)

**jiebin**  
2018-11-13 14:41

wowo大神，好久没更了

[回复](http://www.wowotech.net/memory_management/concept.html#comment-7026)

**一二三**  
2018-07-09 11:09

>> b）由第三方具有智能的模块（如DMA控制器）中转，数据流如下：  
>> Device<----------->DMA Controller<--------->Memory  
>> c）直接访问内存，数据流如下：  
>> Device<---------->IOMMU(Optional)--------->Memory  
这点貌似不对，dma发起内存访问，iommu提供地址映射，相当于一个是发动机，一个是马路，看起来不是并列的关系，是不是应该改成下面这样？  
b）由第三方具有智能的模块（如DMA控制器）中转，数据流如下：  
Device<----------->DMA Controller<--------->IOMMU(Optional)<--------->Memory

[回复](http://www.wowotech.net/memory_management/concept.html#comment-6839)

**toofat**  
2018-04-16 17:48

突然就结束了。。。

[回复](http://www.wowotech.net/memory_management/concept.html#comment-6671)

**看看**  
2018-01-25 12:24

有点虎头蛇尾呀

[回复](http://www.wowotech.net/memory_management/concept.html#comment-6497)

**[wowo](http://www.wowotech.net/)**  
2018-01-25 18:36

@看看：您说的太对了，不是有点，我自己也觉得就是虎头蛇尾:-( 应该多花点时间好好写写。  
谢谢提醒哈~

[回复](http://www.wowotech.net/memory_management/concept.html#comment-6501)

**在路上**  
2017-11-14 20:46

每次看wowo的文章都有新的体会。

[回复](http://www.wowotech.net/memory_management/concept.html#comment-6197)

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
    
    - [Windows系统结合MinGW搭建软件开发环境](http://www.wowotech.net/soft/6.html)
    - [一例centos7.6内核hardlock的解析](http://www.wowotech.net/linux_kenrel/479.html)
    - [Linux设备模型(6)_Bus](http://www.wowotech.net/device_model/bus.html)
    - [X-015-KERNEL-ARM generic timer driver的移植](http://www.wowotech.net/x_project/generic_timer_porting.html)
    - [Linux调度器：用户空间接口](http://www.wowotech.net/process_management/scheduler-API.html)
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
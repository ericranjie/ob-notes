作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2016-9-9 12:44 分类：[内存管理](http://www.wowotech.net/sort/memory_management)
# 一、前言

某天，wowo同学突然来了一句：如果要在start_kernel中点LED，ioremap在什么时间点才能调用呢？我想他应该是想通过点LED灯来调试start_kernel之后的初始化的代码（例如DTB解析部分的代码）。那天，我们两个花了二十分钟的时间，讨论相关的问题，我觉得很有意思，因此决定写fix mapped address这样的一份文档。

在汇编代码中，由于没有打开MMU，想怎么访问外设都很简单，直接使用物理地址即可，然而，进入start kernel之后（打开了MMU），想要访问硬件都是那么的不方便，至少需要通过ioremap获取了虚拟地址之后才可以访问。但是，实际上，在内核的启动的初始阶段，内存管理子系统还没有ready，ioremap还不能调用（在mm_init之后可以正常使用）。

实际上，这个需求是和early ioremap模块相关，此外，还有一些其他的需求，内核合并了这些需求并提出了fix mapped address的概念。本文就是描述关于fix mapped address的方方面面，BTW，本文的代码来自4.4.6内核，体系结构相关的代码依然选择的是ARM64。
# 二、什么是fixmap？

Fix map中的fix指的是固定的意思，那么固定什么东西呢？其实就是虚拟地址是固定的，也就是说，有些虚拟地址在编译（compile-time）的时候就固定下来了，而这些虚拟地址对应的物理地址不是固定的，是在kernel启动过程中被确定的。为了方便大家理解fixmap，我们提供一个具体的例子：DTB image的处理。

实际上DTB的处理一开始并没有使用fixmap，而是放在了kernel image附近。更具体的要求是必须放在kernel image起始地址的512M内，8字节对齐，dtb image不能越过2M section size的边界。之所以这么要求，主要是想借用kernel image页表的“东风”，反正建立kernel image所需要的PGD/PUD/PMD页表都已经静态分配了，对dtb image的要求可以不需要建立新的页表，只是在原有的页表中增加一个entry而已，不过这样的设计存在下面的问题：

（1）对dtb image的位置有限制，不是很灵活（谁不想自由自在呢？bootload copy dtb img的时候当然想了无牵挂啊）。
（2）对dtb image的位置检查在head.S中，需要汇编实现（谁想写汇编啊，用c实现多买易懂啊，可维护性，可移植性多么好啊）。

现在，既然内核有了early fixmap的支持，那么上面的限制都可以放宽了。整个dtb image的处理过程如下：

（1）bootloader copy dtb image到memory的某个位置上。具体的位置随便，当然还是要满足8字节对齐，dtb image不能越过2M section size的边界的要求，毕竟我们也想一条section mapping就搞定dtb image。
（2）bootload通过寄存器x0传递dtb image的物理地址，dtb image的虚拟地址在编译kernel image的时候就确定了。
（3）汇编初始化阶段不对dtb image做任何处理
（4）在start kernel之后的初始化代码中（具体在setup_arch--->setup_machine_fdt中），创建dtb image的相关Translation tables，之后就可以自由的访问dtb image了。
# 三、为何有fixmap这个概念？

动态分配虚拟地址以及建立地址映射是一个复杂的过程，在内核完全启动之后，内存管理可以提供各种丰富的API让内核的其他模块可以完成虚拟地址分配和建立地址映射的功能，但是，在内核的启动过程中，有些模块需要使用虚拟内存并mapping到指定的物理地址上，而且，这些模块也没有办法等待完整的内存管理模块初始化之后再进行地址映射。因此，linux kernel固定分配了一些fixmap的虚拟地址，这些地址有固定的用途，使用该地址的模块在初始化的时候，讲这些固定分配的地址mapping到指定的物理地址上去。

最直观的需求来自初始化代码的调试，想一想当我们来到start_kernel的时候我们面临的处境：

（1）我们不能访问全部的内存，只能访问kernel image附近的memory。
（2）我们不能访问任何的硬件，所有的io memory还没有mapping

想要通过串口控制台输出错误信息？sorry，现在离console驱动的初始化还早着呢。想点个LED灯看看内核运行情况？sorry，ioremp函数需要kmalloc分配内存，但是伙伴系统还没有初始化呢。怎么办？一个最简洁的方法就是简化虚拟内存的分配和管理（ioremp中使用的管理虚拟内存地址的方法太重了），而最简单的方法就是fix virtual address。
# 四、fixmap的具体位置在那里？

fixmap的地址区域位于FIXADDR_START和FIXADDR_TOP之间，具体可以参考下图：
![[Pasted image 20241007233004.png]]

上图中，红色框的block就是fixmap address的具体位置。

五、fixmap具体应用在哪些场景？

fixmap的地址区域有被进一步细分，如下：
```cpp
enum fixed_addresses {  
FIX_HOLE,   
FIX_FDT_END,  
FIX_FDT = FIX_FDT_END + FIX_FDT_SIZE / PAGE_SIZE - 1,

FIX_EARLYCON_MEM_BASE,  
FIX_TEXT_POKE0,  
end_of_permanent_fixed_addresses,

FIX_BTMAP_END = __end_of_permanent_fixed_addresses,  
FIX_BTMAP_BEGIN = FIX_BTMAP_END + TOTAL_FIX_BTMAPS - 1,  
__end_of_fixed_addresses  
};
```
由定义可知，fixmap地址区域又分成了两个部分，一部分叫做permanent fixed address，是用于具体的某个内核模块的，使用关系是永久性的。另外一个叫做temporary fixed address，各个内核模块都可以使用，用完之后就释放，模块和虚拟地址之间是动态的关系。

permanent fixed address主要涉及的模块包括：

（1）dtb解析模块。
（2）early console模块。标准的串口控制台驱动的初始化在整个kernel初始化过程中是很靠后的事情了，如果能够在kernel启动阶段的初期就有一个console，能够输出各种debug信息是多买美妙的事情啊，early console就能满足你的这个愿望，这个模块是使用early param来初始化该模块的功能的，因此可以很早就投入使用，从而协助工程师了解内核的启动过程。
（3）动态打补丁的模块。正文段一般都被映射成read only的，该模块可以使用fix mapped address来映射RW的正文段，从动态修改程序正文段，从而完成动态打补丁的功能。

temporary fixed address主要用于early ioremap模块。linux kernel在fix map区域的虚拟地址空间中开了FIX_BTMAPS_SLOTS个的slot（每个slot的size是NR_FIX_BTMAPS），内核中的模块都能够通过early_ioremap、early_iounmap的接口来申请或者释放对某个slot 虚拟地址的使用。

_原创文章，转发请注明出处。蜗窝科技_

标签: [fixmap](http://www.wowotech.net/tag/fixmap)

---

« [X-011-UBOOT-使用bootm命令启动kernel(Bubblegum-96平台)](http://www.wowotech.net/x_project/bubblegum_uboot_bootm.html) | [u-boot FIT image介绍](http://www.wowotech.net/u-boot/fit_image_overview.html)»

**评论：**

**crystal736**  
2017-12-25 09:19

你好：  
   有一个问题，在发生tlb miss的时候，会进行tlb充填工作，但是tlb充填时候会通过pgd和虚拟地址找到页表项，但是如果此时页表还没有被分配，是不是会发生page fault？此时的page fault流程是什么呢？

[回复](http://www.wowotech.net/memory_management/fixmap.html#comment-6416)

**加西亚将军**  
2020-12-11 21:41

@crystal736：arm64会发生转换表错误的异常，发生缺页异常分配各级页表的页，然后填充页表项，处理完毕，重新执行发生页错误的指令。

[回复](http://www.wowotech.net/memory_management/fixmap.html#comment-8154)

**timor**  
2017-02-27 10:42

有些虚拟地址在编译（compile-time）的时候就固定下来了，而这些虚拟地址对应的物理地址不是固定的，是在kernel启动过程中被确定的.  
  
---虚拟地址编译的时候就固定下来了,这就话怎么理解？  是在哪里设置的，还是？  
---还有“反正建立kernel image所需要的PGD/PUD/PMD页表都已经静态分配了” 是在什么时候静态分配的。 谢谢。

[回复](http://www.wowotech.net/memory_management/fixmap.html#comment-5256)

**[linuxer](http://www.wowotech.net/)**  
2017-02-27 19:12

@timor：---虚拟地址编译的时候就固定下来了,这就话怎么理解？  是在哪里设置的，还是？  
＝＝＝＝＝＝＝＝＝＝＝＝  
我们一dtb为例子来说吧：当完成内核编译之后，dtb的虚拟地址是确定的，不存在动态分配的问题，这个固定的虚拟地址就是__fix_to_virt(FIX_FDT)  
  
---还有“反正建立kernel image所需要的PGD/PUD/PMD页表都已经静态分配了” 是在什么时候静态分配的。  
＝＝＝＝＝＝＝＝＝＝＝＝  
请参考内核的链接脚本，如下：  
......  
    BSS_SECTION(0, 0, 0)  
  
    . = ALIGN(PAGE_SIZE);  
    idmap_pg_dir = .;  
    . += IDMAP_DIR_SIZE;  
    swapper_pg_dir = .;  
    . += SWAPPER_DIR_SIZE;  
  
    _end = .;  
......

[回复](http://www.wowotech.net/memory_management/fixmap.html#comment-5266)

**[electrlife](http://www.wowotech.net/)**  
2016-12-21 17:36

这段时间一直跟着Linuxer学习mmu管理，再次读完本节，发表下我对于fixmap的理解或是读书笔记，请linuxer指正。  
在内核初始化阶段，内核仅仅映射kernel image & identify map 两个地方的虚拟地址空间。固定映射所需的页表都静态定义在kernel image中，因此可以方便的时行访问，这样可很轻松的映射一些虚拟地址。但是这是对虚拟地址的限制的，虚拟地址不能太大，其所需的PGD描述符应该可以在一个全局的PGD页表中找到，因为全局的页表仅使用一个页来保存,所以fixed map的虚拟地址应该限制在一定范围内。fixed map映射所需要的页表按文章所述应该是静态的存在kernel image bss段.  
  
以上是个人浅显的理解，不知是否正确，请Linuxer指正。  
  
另，对于fixed map映射所需要的页表，及其定义能否简单讲解下？谢谢！

[回复](http://www.wowotech.net/memory_management/fixmap.html#comment-5046)

**[electrlife](http://www.wowotech.net/)**  
2016-12-21 17:38

@electrlife：对于fixed map感觉arm 32中似乎没有使用，至少fdt的映射我没有找到相关的信息？不知是否这样？

[回复](http://www.wowotech.net/memory_management/fixmap.html#comment-5047)

**[wowo](http://www.wowotech.net/)**  
2017-01-04 08:56

@electrlife：我看手上linux-4.0.5的代码，arm 32也是可以使用fixmap的。

[回复](http://www.wowotech.net/memory_management/fixmap.html#comment-5085)

**andyshrk**  
2016-11-06 11:21

linuxer 您好，  
我去内核里面搜了下，ARM32的FIXADDR_START是从0xffc00000开始的，从一些内核的启动log上也可以看到 fixmap  : 0xffc00000 - 0xfff00000   (3072 kB)，但是在ARM32下面有一个DEBUG_LL的功能，  
我看下面很多平台定义的对应DEBUG_UART_VIRT（即DEBUG串口对应的虚拟地址）却不在这个fixmap的范围内，而且我手边的某些平台，我尝试去修改这个虚拟地址（kernel hacking里面打开low level debug选项，和early printk选项，然后在command line里面加入earlyprintk参数），我发现有些值可以用，有些值却无会导致串口没有任何输出。不知道他这个是按照什么规则去映射的。  
另外，在比较低版本内核里面，很多platform相关的代码都会通过medsc->map_io接口调用iotable_init接口对很多外设去做静态映射（即使现在最想的linux 4.9内核，在devicemaps_init函数中也还保留了对这个接口的兼容），我也一直没搞明白，他这个静态映射的虚拟地址是根据什么规则去定下来的。  
不知道博主对这方面有没有研究。

[回复](http://www.wowotech.net/memory_management/fixmap.html#comment-4847)

**[linuxer](http://www.wowotech.net/)**  
2016-11-08 09:23

@andyshrk：我对ARM32的代码反而没有那么熟悉，呵呵～～不过基本原理是不会变的，在打开MMU之前，可以直接操作UART的物理地址，在打开MMU之后，一定要进行map，然后操作UART的虚拟地址来访问串口。当然，具体的流程可以自己看看代码，我就偷一下懒了，哈哈  
  
machine中的map_io接口应该是对io地址空间进行映射。在过去，倾向在这里完成所有io空间的映射，而现在的内核，把这些io空间的映射放到了各个驱动模块中。当然，现在的机制会合理一些，毕竟谁的资源谁管理嘛。

[回复](http://www.wowotech.net/memory_management/fixmap.html#comment-4851)

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
    
    - [Linux common clock framework(2)_clock provider](http://www.wowotech.net/pm_subsystem/clock_provider.html)
    - [irq wakeup in linux](http://www.wowotech.net/pm_subsystem/491.html)
    - [CFS任务放置代码详解](http://www.wowotech.net/process_management/task_placement_detail.html)
    - [这是一篇发布在讨论区的测试文章](http://www.wowotech.net/test.html)
    - [RCU synchronize原理分析](http://www.wowotech.net/kernel_synchronization/223.html)
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
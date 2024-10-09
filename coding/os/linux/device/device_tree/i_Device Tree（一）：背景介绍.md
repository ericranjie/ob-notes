作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2014-5-22 16:46 分类：[统一设备模型](http://www.wowotech.net/sort/device_model)

一、前言

作为一个多年耕耘在linux 2.6.23内核的开发者，各个不同项目中各种不同周边外设驱动的开发以及各种琐碎的、扯皮的俗务占据了大部分的时间。当有机会下载3.14的内核并准备学习的时候，突然发现linux kernel对于我似乎变得非常的陌生了，各种新的机制，各种framework、各种新的概念让我感到阅读内核代码变得举步维艰。 还好，剖析内核的热情还在，剩下的就交给时间的。首先进入视线的是Device Tree机制，这是和porting内核非常相关的机制，如果想让将我们的硬件平台迁移到高版本的内核上，Device Tree是一个必须要扫清的障碍。

我想从下面三个方面来了解Device Tree：

1、为何要引入Device Tree，这个机制是用来解决什么问题的？（这是本文的主题）
2、Device Tree的基础概念（请参考[DT基础概念](http://www.wowotech.net/linux_kenrel/dt_basic_concept.html)）
3、ARM linux中和Device Tree相关的代码分析（请参考[DT代码分析](http://www.wowotech.net/linux_kenrel/dt-code-analysis.html)）

阅读linux内核代码就像欣赏冰山，有看得到的美景（各种内核机制及其代码），也有埋在水面之下看不到的基础（机制背后的源由和目的）。沉醉于各种内核机制的代码固然有无限乐趣，但更重要的是注入更多的思考，思考其背后的机理，真正理解软件抽象。这样才能举一反三，并应用在具体的工作和生活中。

本文主要从下面几个方面阐述为何ARM linux会引入Device Tree：

1、没有Device Tree的ARM linux是如何运转的？
2、混乱的ARM architecture代码和存在的问题
3、新内核的解决之道

二、没有Device Tree的ARM linux是如何运转的？

我曾经porting内核到两个ARM-based的平台上。一个是小的芯片公司的应用处理器，公司自己购买了CPU core，该CPU core使用ARM兼容的指令集（但不是ARM）加上各种公司自行设计的多媒体外设整合成公司的产品进行销售。而我的任务就是porting 2.4.18内核到该平台上。在黑白屏幕的手机时代，那颗AP（application process）支持了彩屏、camera、JPEG硬件加速、2D/3D加速、MMC/SD卡、各种音频加速（内置DSP）等等特性，功能强大到无法直视。另外一次移植经历是让2.6.23内核跑在一个大公司的冷门BP（baseband processor）上。具体porting的方法是很简单的：

1、自己撰写一个bootloader并传递适当的参数给kernel。除了传统的command line以及tag list之类的，最重要的是申请一个machine type，当拿到属于自己项目的machine type ID的时候，当时心情雀跃，似乎自己已经是开源社区的一份子了（其实当时是有意愿，或者说有目标是想将大家的代码并入到linux kernel main line的）。

2、在内核的arch/arm目录下建立mach-xxx目录，这个目录下，放入该SOC的相关代码，例如中断controller的代码，时间相关的代码，内存映射，睡眠相关的代码等等。此外，最重要的是建立一个board specific文件，定义一个machine的宏：

```cpp
MACHINE_START(project name, "xxx公司的xxx硬件平台")  
.phys_io    = 0x40000000,  
.boot_params    = 0xa0000100,    
.io_pg_offst    = (io_p2v(0x40000000) >> 18) & 0xfffc,  
.map_io        = xxx_map_io,  
.init_irq    = xxx_init_irq,  
.timer        = &xxx_timer,  
.init_machine    = xxx_init,  
MACHINE_END
```

在xxx_init函数中，一般会加入很多的platform device。因此，伴随这个board specific文件中是大量的静态table，描述了各种硬件设备信息。

3、调通了system level的driver（timer，中断处理，clock等）以及串口terminal之后，linux kernel基本是可以起来了，后续各种driver不断的添加，直到系统软件支持所有的硬件。

综上所述，在linux kernel中支持一个SOC平台其实是非常简单的，让linux kernel在一个特定的平台上“跑”起来也是非常简单的，问题的重点是如何优雅的”跑”。

三、混乱的ARM architecture代码和存在的问题

每次正式的linux kernel release之后都会有两周的merge window，在这个窗口期间，kernel各个部分的维护者都会提交各自的patch，将自己测试稳定的代码请求并入kernel main line。每到这个时候，Linus就会比较繁忙，他需要从各个内核维护者的分支上取得最新代码并merge到自己的kernel source tree中。Tony Lindgren，内核OMAP development tree的维护者，发送了一个邮件给Linus，请求提交OMAP平台代码修改，并给出了一些细节描述：

1、简单介绍本次改动
2、关于如何解决merge conficts。有些git mergetool就可以处理，不能处理的，给出了详细介绍和解决方案

一切都很平常，也给出了足够的信息，然而，正是这个pull request引发了一场针对ARM linux的内核代码的争论。我相信Linus一定是对ARM相关的代码早就不爽了，ARM的merge工作量较大倒在其次，主要是他认为ARM很多的代码都是垃圾，代码里面有若干愚蠢的table，而多个人在维护这个table，从而导致了冲突。因此，在处理完OMAP的pull request之后（Linus并非针对OMAP平台，只是Tony Lindgren撞在枪口上了），他发出了怒吼：

> Gaah. Guys, this whole ARM thing is a f\*cking pain in the ass.

负责ARM linux开发的Russell King脸上挂不住，进行了反驳：事情没有那么严重，这次的merge conficts就是OMAP和IMX/MXC之间一点协调的问题，不能抹杀整个ARM linux团队的努力。其他的各个ARM平台维护者也加入讨论：ARM平台如何复杂，如何庞大，对于arm linux code我们已经有一些思考，正在进行中……一时间，讨论的气氛有些尖锐，但总体是坦诚和友好的。

对于一件事情，不同层次的人有不同层次的思考。这次争论涉及的人包括：

1、内核维护者（CPU体系结构无关的代码）
2、维护ARM系统结构代码的人
3、维护ARM sub architecture的人（来自各个ARM SOC vendor）

维护ARM sub architecture的人并没有强烈的使命感，作为公司的一员，他们最大的目标是以最快的速度支持自己公司的SOC，尽快的占领市场。这些人的软件功力未必强，对linux kernel的理解未必深入（有些人可能很强，但是人在江湖身不由己）。在这样的情况下，很多SOC specific的代码都是通过copy and paste，然后稍加修改代码就提交了。此外，各个ARM vendor的SOC family是一长串的CPU list，每个CPU多多少少有些不同，这时候＃ifdef就充斥了各个源代码中，让ARM mach-和plat-目录下的代码有些不忍直视。

作为维护ARM体系结构的人，其能力不容置疑。以Russell King为首的team很好的维护了ARM体系结构的代码。基本上，除了mach-和plat-目录，其他的目录中的代码和目录组织是很好的。作为ARM linux的维护者，维护一个不断有新的SOC加入的CPU architecture code的确是一个挑战。在Intel X86的架构一统天下的时候，任何想正面攻击Intel的对手都败下阵来。想要击倒巨人（或者说想要和巨人并存）必须另辟蹊径。ARM的策略有两个，一个是focus在嵌入式应用上，也就意味着要求低功耗，同时也避免了和Intel的正面对抗。另外一个就是博采众家之长，采用license IP的方式，让更多的厂商加入ARM建立的生态系统。毫无疑问，ARM公司是成功的，但是这种模式也给ARM linux的维护者带来了噩梦。越来越多的芯片厂商加入ARM阵营，越来越多的ARM platform相关的代码被加入到内核，不同厂商的周边HW block设计又各不相同……

内核维护者是真正对操作系统内核软件有深入理解的人，他们往往能站在更高的层次上去观察问题，发现问题。Linus注意到每次merge window中，ARM的代码变化大约占整个ARCH目录的60％，他认为这是一个很明显的符号，意味着ARM linux的代码可能存在问题。其实，60％这个比率的确很夸张，因为unicore32是在2.6.39 merge window中第一次全新提交，它的代码是全新的，但是其代码变化大约占整个ARCH目录的9.6％（需要提及的是unicore32是一个中国芯）。有些维护ARM linux的人认为这是CPU市场占用率的体现，不是问题，直到内核维护者贴出实际的代码并指出问题所在。内核维护者当然想linux kernel支持更多的硬件平台，但是他们更愿意为linux kernel制定更长远的规划。例如：对于各种繁杂的ARM平台，用一个kernel image来支持。

经过争论，确定的问题如下：

1、ARM linux缺少platform（各个ARM sub architecture，或者说各个SOC）之间的协调，导致arm linux的代码有重复。值得一提的是在本次争论之前，ARM维护者已经进行了不少相关的工作（例如PM和clock tree）来抽象相同的功能模块。
2、ARM linux中大量的board specific的源代码应该踢出kernel，否则这些垃圾代码和table会影响linux kernel的长期目标。
3、各个sub architecture的维护者直接提交给Linux并入主线的机制缺乏层次。

四、新内核的解决之道

针对ARM linux的现状，最需要解决的是人员问题，也就是如何整合ARM sub architecture（各个ARM Vendor）的资源。因此，内核社区成立了一个ARM sub architecture的team，该team主要负责协调各个ARM厂商的代码（not ARM core part），Russell King继续负责ARM core part的代码。此外，建立一个ARM platform consolidation tree。ARM sub architecture team负责review各个sub architecture维护者提交的代码，并在ARM platform consolidation tree上维护。在下一个merge window到来的时候，将patch发送给Linus。

针对重复的代码问题，如果不同的SOC使用了相同的IP block（例如I2C controller），那么这个driver的code要从各个arch/arm/mach-xxx中独立出来，变成一个通用的模块供各个SOC specific的模块使用。移动到哪个目录呢？对于I2C或者USB OTG而言，这些HW block的驱动当然应该移动到kernel/drivers目录。因为，对于这些外设，可能是in-chip，也可能是off-chip的，但是对于软件而言，它们是没有差别的（或者说好的软件抽象应该掩盖底层硬件的不同）。对于那些system level的code呢？例如clock control、interrupt control。其实这些也不是ARM-specific，应该属于linux kernel的核心代码，应该放到linux/kernel目录下，属于core-Linux-kernel frameworks。当然对于ARM平台，也需要保存一些和framework交互的code，这些code叫做ARM SoC core architecture code。OK，总结一下：

1、ARM的核心代码仍然保存在arch/arm目录下
2、ARM SoC core architecture code保存在arch/arm目录下
3、ARM SOC的周边外设模块的驱动保存在drivers目录下
4、ARM SOC的特定代码在arch/arm/mach-xxx目录下
5、ARM SOC board specific的代码被移除，由Device Tree机制来负责传递硬件拓扑和硬件资源信息。

OK，终于来到了Device Tree了。本质上，Device Tree改变了原来用hardcode方式将HW 配置信息嵌入到内核代码的方法，改用bootloader传递一个DB的形式。对于基于ARM CPU的嵌入式系统，我们习惯于针对每一个platform进行内核的编译。但是随着ARM在消费类电子上的广泛应用（甚至桌面系统、服务器系统），我们期望ARM能够象X86那样用一个kernel image来支持多个platform。在这种情况下，如果我们认为kernel是一个black box，那么其输入参数应该包括：

1、识别platform的信息

2、runtime的配置参数

3、设备的拓扑结构以及特性

对于嵌入式系统，在系统启动阶段，bootloader会加载内核并将控制权转交给内核，此外，还需要把上述的三个参数信息传递给kernel，以便kernel可以有较大的灵活性。在linux kernel中，Device Tree的设计目标就是如此。

_原创文章，转发请注明出处。蜗窝科技_，[www.wowotech.net](http://www.wowotech.net/linux_kenrel/why-dt.html)。

标签: [Device](http://www.wowotech.net/tag/Device) [tree](http://www.wowotech.net/tag/tree)

______________________________________________________________________

« [蓝牙协议分析(1)\_基本概念](http://www.wowotech.net/bluetooth/bt_overview.html) | [Linux电源管理(3)\_Generic PM之Reboot过程](http://www.wowotech.net/pm_subsystem/reboot.html)»

**评论：**

**DyalnW**\
2022-10-12 14:06

正在学习device tree，感谢大佬的文章

[回复](http://www.wowotech.net/device_model/why-dt.html#comment-8686)

**wolf2blood**\
2021-05-08 17:05

想请教蜗窝一个问题:我现在对海思HI3536进行内核升级，从3.10升级到4.9.88，uboot是2010版本的，我拿的海思最新的补丁是HI3559的linux-4.9.37.patch，里面有关于HI3536的配置文件！我编译成功生成uImage，加载的时候卡在了Uncompressing Linux... done, booting the kernel，我追踪内核代码发现是在init/main的函数setup_arch-->paging_init-->devicemaps_init-->local_tlb_flush_all这里卡住了，清除TLB页表缓存不动作！这个uboot本来是传递机器码的，我现在按照网上那种方式传递dtb还是不行，是不是uboot不支持传递设备树所以配置了也没有用啊！我的配置是这样的bootcmd=ping 192.168.1.19; ping 192.168.1.19;tftp 0x42000000 uImage;tftp 0x45500000 hi3536dv100.dtb;bootm 0x42000000 - 0x45500000，内核大小为3.1M，我折腾了一个半月依然卡在这里，跪求蜗窝能给一些建议！

[回复](http://www.wowotech.net/device_model/why-dt.html#comment-8235)

**kein**\
2021-07-19 13:30

@wolf2blood：老哥，问题有没有解决了，是什么原因？

[回复](http://www.wowotech.net/device_model/why-dt.html#comment-8254)

**Gonglja**\
2022-04-28 20:32

@wolf2blood：uboot中 添加对扁平化设备树的支持，CONFIG_OF_LIBFDT，然后机器码传递0xffffffff即可。

[回复](http://www.wowotech.net/device_model/why-dt.html#comment-8605)

**Jack**\
2019-02-25 16:06

請教蝸窩兩幾個問題：

1. 目前再研究如何將新的Kernel 移植到前人所交接的版子，原先是可以動作。(版本為Kernel 4.15左右)\
   但我去下載kernel 4.19版本來編譯後，根據其他網站及論壇所講解的方式及步驟，有成功將zImage所編譯出來。\
   放入燒錄後，但始終卡住在Starting kernel位置，不知蝸窩是否有什麼方向或是建議可提供。謝謝蝸窩。
1. 而我編譯的步驟為\
   a. 下載源碼\
   b. 設定環境變數export ARCH=arm;export CROSS_COMPILE=arm-linux-gnueabihf-;\
   c. 開始編譯 make menuconfig;make;之後就會出現zImage\
   不知蝸窩有沒有什麼方向可以提供。謝謝。

[回复](http://www.wowotech.net/device_model/why-dt.html#comment-7204)

**Downey**\
2019-07-11 16:39

@Jack：可能是image地址的问题

[回复](http://www.wowotech.net/device_model/why-dt.html#comment-7528)

**bigbigtree**\
2018-04-16 07:34

wowo大哥，您好！请问一下您上面讲的移植内核“调通了system level的driver（timer，中断处理，clock等）以及串口terminal之后，linux kernel基本是可以起来了”，这个是怎么做的呢？有没有一些文章介绍呢？就是让linux最小系统跑起来，可以进入终端，谢谢您

[回复](http://www.wowotech.net/device_model/why-dt.html#comment-6667)

**[wowo](http://www.wowotech.net/)**\
2018-04-16 08:47

@bigbigtree：你可以看看这个专题（不知道能否帮到你）：\
http://www.wowotech.net/sort/x_project

[回复](http://www.wowotech.net/device_model/why-dt.html#comment-6669)

**wason**\
2018-02-27 10:48

你好，看了你的文章，感到非常惊讶，请问你是如何对linux设备树的历史背景了解得那么清楚呢？

[回复](http://www.wowotech.net/device_model/why-dt.html#comment-6567)

**black**\
2017-12-28 14:04

逻辑上flat device tree实现了adv cfg & pwr management(acpi)中的cfg部分，但是power management部分怎么办？

[回复](http://www.wowotech.net/device_model/why-dt.html#comment-6428)

**风露**\
2017-12-21 23:05

请教蜗窝两个问题：\
1、对于platform匹配，我的理解是这样的：\
过去采用的方式是，Linux静态定义板级信息，通过bootloader传递的machine type ID 进行静态匹配，匹配上之后得到相关硬件资源和定义的初始化接口。\
dt采用的方式是，Linux同样定义一些静态信息（DT_MACHINE_START和MACHINE_END），然后根据dts根目录中的“compatible”来进行匹配。\
这样理解对吗？如果是这样，在platform匹配上面，这两种方式好像换汤不换药啊。\
2、对于硬件信息的初始化。我的理解是这样的：\
过去的方式是：通过静态配置的方式，把各种控制器相关的信息（如寄存器）写死，内核中配置上要使用的驱动，并给驱动传入这些寄存器等参数，完成硬件的初始化。\
dt的方式是，片上的硬件信息都写到dts中，内核配置上可能用到的驱动（会产生冗余？），然后Linux启动时会解析dtb，根据node来进行硬件设备的注册。这种理解对吗？另外，这是否就是dt引入的根本原因？

[回复](http://www.wowotech.net/device_model/why-dt.html#comment-6406)

**[linuxer](http://www.wowotech.net/)**\
2017-12-22 16:36

@风露：基本上我是认同你的看法的。\
DTS之前，bootloader可以通过tag list传递参数，以便实现内核的动态配置，但是，这时候设备往往是通过静态定义的xxx_device hardcode在代码中，DTS主要解决这个问题，顺便接管了原理tag list的功能，使之统一在dts的框架中。

[回复](http://www.wowotech.net/device_model/why-dt.html#comment-6408)

**风露**\
2017-12-22 17:35

@linuxer：关于tag list，bootargs可以通过在 chosen node中指定，但对于一些自定义tag在哪里进行传递

[回复](http://www.wowotech.net/device_model/why-dt.html#comment-6410)

**Isara**\
2017-10-27 10:59

字里行间体现出的风采，和背后沉淀的底蕴，这都是装不出来的

[回复](http://www.wowotech.net/device_model/why-dt.html#comment-6146)

**zin**\
2017-07-17 18:29

写的真好，学习了

[回复](http://www.wowotech.net/device_model/why-dt.html#comment-5817)

**光阴的事故**\
2017-02-08 10:21

感谢，我是新手，有空就过来学习哈

[回复](http://www.wowotech.net/device_model/why-dt.html#comment-5211)

1 [2](http://www.wowotech.net/device_model/why-dt.html/comment-page-2#comments) [3](http://www.wowotech.net/device_model/why-dt.html/comment-page-3#comments)

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

  - [KASAN实现原理](http://www.wowotech.net/memory_management/424.html)
  - [eMMC 原理 1 ：Flash Memory 简介](http://www.wowotech.net/basic_tech/flash_memory_intro.html)
  - [irq wakeup in linux](http://www.wowotech.net/pm_subsystem/491.html)
  - [Linux2.6.11版本：classic RCU的实现](http://www.wowotech.net/kernel_synchronization/linux2-6-11-RCU.html)
  - [Linux时间子系统之（十二）：periodic tick](http://www.wowotech.net/timer_subsystem/periodic-tick.html)

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

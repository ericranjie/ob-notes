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

作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2015-7-7 22:31 分类：[ARMv8A Arch](http://www.wowotech.net/sort/armv8a_arch)

#### 1. 前言

ARMv8（当前只有A系列，即ARMv8-A）架构，是ARM公司为满足新需求而重新设计的一个架构，是近20年来，ARM架构变动最大的一次。它引入的Execution State、Exception Level、Security State等新特性，已经和我们对旧的ARM架构的认知，有很大差距了。

因此，本文从ARMv8-A产生的背景开始，对它进行一个简单的介绍，使大家从整体上，对ARMv8有一个简单的了解。

#### 2. 背景

本节参考自“[http://www.arm.com/zh/files/downloads/ARMv8_white_paper_v5.pdf](http://www.arm.com/zh/files/downloads/ARMv8_white_paper_v5.pdf)”，感兴趣的同学可以自行阅读。

有一点是可以确定的，ARM诞生时，对Intel主导的PC市场，没有（也不敢有）一点点的非分之想。最初的ARMv4（ARM7系列），到最近的ARMv7（Cortex-A,-M,-R系列），都是针对功耗比较敏感的移动设备的，就性能而言，基于ARM处理器的设备，始终无法和PC相提并论。

但从ARMv7开始，情况开始有些转变，ARM的市场开始扩展到移动设备之外的其它领域，这也是ARMv7划分为A（Application）、R（Real-time）和M（Microcontroller）三个系列的原因，其实质就是三个细分市场，其中的A系列，就是针对性能要求较高的应用。

特别是在Cortex-A9之后，ARM的处理性能有很大的提高，渐渐的吸引了一些PC用户。因此基于ARM的类PC产品，如平板电脑，开始大量涌现。此时，ARM的处理能力，已经有机会应用于其它领域了，如企业设备、服务器等，当然，其优势依然是低功耗。

与此同时，新的趋势正在酝酿，主要包括大内存（Large Memory）、虚拟化（Virtualization）和安全（Security）。Virtualization在ARMv7上已经有简单的硬件实现，Security也有可能基于当前架构扩展，唯有Large memory的需求，有点棘手。

由于处理器性能越来越强，运行于其上的软件也来越复杂，复杂到单一应用对内存的需求可能超出32-bit架构所能支持的最大内存（4G），这就是Large memory需求的起因。不过，后来的Cortex-A15（ARMv7架构）通过Large Physical Address Extensions (LPAE) 技术，可以支持高达40bits的物理地址空间。但受限于32-bit的指令集，虚拟地址空间依旧只有32bits（4G），如果有应用需要更大的虚拟内存，怎么办？只能定义一个新的架构，使用64-bit的指令集（也即我们常说的ARM64）。

毫无疑问，在现阶段，需要超过4G虚拟内存的应用场景，是非常少的。但ARM还是定义了一个新的架构--ARMv8，为什么呢？下面是ARM的解释（只有伟大的公司才有伟大的理念！）：

> Trends. That’s really what ARM has to look at when defining a new architecture. That is the nature of our business, we need to look a long way forward, and plan.

当然，ARMv8并不仅仅是为了解决虚拟地址的问题，它也要解决现有架构的一些问题。不过，新的问题又来了：一个新的架构？用户为什么要使用新的架构？因此，ARMv8的定义，必须先满足如下前提条件：

> 1）对上兼容。
> 
> 2）能解决现存架构的已知问题。
> 
> 3）相比现存架构，必须具备优势明显的新特性，哪怕软件从来不使用这些新特性。

以上就是ARMv8-a产生的背景，也是ARMv8-a架构之所以是“这个”样子的直接原因。那么到底是什么样子呢？我们继续介绍。

#### 3. ARMv8-a架构简介

基于上面的前提条件，ARMv8-a架构的主要特性包括：

1）新增一套64-bit的指令集，称作A64。

2）由于需要向前兼容ARMv7，所以同时支持现存的32-bit指令集，称作A32和T32（也即我们熟悉的ARM和Thumb指令集）。

3）定义AArch64和AArch32两套运行环境（称作Execution state），分别执行64-bit和32-bit指令集。软件可以在需要的时候，切换Execution state。

4）AArch64最大的改动，使用新的概念（exception level），重新解释了processor mode、privilege level等概念，具体可参考第4章的介绍。

5）在ARMv7 security extension的基础上，新增security model，支持安全相关的应用需求。

6）在ARMv7 virtualization extension的基础上，提供完整的virtualization框架，从硬件上支持虚拟化。

#### 4. AArch64 Exception level

Exception level，是ARMv8-a引入的一个新概念，用于整合之前架构中processor mode和privilege level相关的功能。

**4.1 ARMv7之前的实现**

我们知道，以前的ARM架构，处理器可以工作在多种模式（称作processor mode）下，包括User、FIQ、IRQ、Abort、Undefined、System等，之所以存在不同的模式，主要有2个方面的考虑：

> 1）不同的处理器模式，有不同的硬件访问权限，称作privilege level。
> 
> 主要有2个level，privilege和non-privilege。其中只有User模式属于non-privilege level，其它均是privilege level。
> 
> 安全起见，大多数时候，软件都运行在User mode。一旦需要其它操作，则需要切换到相应的privilege模式下。这是最原始、最朴素的安全思想，当然，只防君子，不防小人。
> 
> 2）这些处理器模式，除User模式外，其它模式基本上和各类异常一一对应。而不同的模式，都有一些自己独有的寄存器，例如R13(SP)、R14(LR)等等，可以使模式切换过程（也是异常处理过程）更为高效、便利。

**4.2 ARMv7-a的实现**

ARMv7-a基本保留了之前的设计，不同之处，将privilege level命名了，称作PL0和PL1（也许您猜到了，后来出现了PL2，用于虚拟化扩展（Virtualization Extension）。

另外，增加了两个模式：Monitor和Supervisor，分别用于security扩展和virtualization扩展。

**4.3 ARMv8-a的实现**

可能ARMv8-a的设计者觉得之前的设计有些啰嗦，就把processor mode的概念去掉（或者说淡化）了，取而代之的是4个固定的Exception level，简称EL0-EL3。同时，也淡化了privilege level的概念。Exception level本身就已经包好了privilege的信息，即ELn的privilege随着n的增大而增大。类似地，可以将EL0归属于non-privilege level，EL1/2/3属于privilege level。

这些Exception level的现实意义是（如下图，先忽略Secure model有关的内容）：

[![aarch32_exception_levels](http://www.wowotech.net/content/uploadfile/201507/90b488a2378e7792f7cf5067541c0b2b20150707143059.gif "aarch32_exception_levels")](http://www.wowotech.net/content/uploadfile/201507/7ffa71881f6356c740ef254b01cecf1520150707143059.gif)

> ARMv8-a Exception level有关的说明如下：
> 
> 1）首先需要注意的是，AArch64中，已经没有User、SVC、ABT等处理器模式的概念，但ARMv8需要向前兼容，在AArch32中，就把这些处理器模式map到了4个Exception level。
> 
> 2）Application位于特权等级最低的EL0，Guest OS（Linux kernel、window等）位于EL1，提供虚拟化支持的Hypervisor位于EL2（可以不实现），提供Security支持的Seurity Monitor位于EL3（可以不实现）。
> 
> 3）只有在异常发生时（或者异常处理返回时），才能切换Exception level（这也是Exception level的命名原因，为了处理异常）。当异常发生时，有两种选择，停留在当前的EL，或者跳转到更高的EL，EL不能降级。同样，异常处理返回时，也有两种选择，停留在当前EL，或者调到更低的EL。
> 
> 注1：有关ARMv8-a异常处理的具体细节，会在其它文章中描述。

#### 5. security model

ARMv8-a的security模型基本沿用了ARMv7 security extension的思路，主要目的保护一些安全应用的数据，例如支付等。它不同于privilege level等软件逻辑上的保护，而是一种物理上的区隔，即不同security状态下，可以访问的物理内存是不同的。

ARMv8-a架构有两个security state（参考上面图片），Security和non-Security。主要的功效是物理地址的区隔，以及一些system control寄存器的访问控制：

> 在Security状态下，处理器可以访问所有的Secure physical address space以及Non-secure physical address space；
> 
> 在Non-security状态下，只能访问Non-secure physical address space，且不能访问Secure system control resources。

#### 6. virtualization

硬件虚拟化包括指令集虚拟化、异常处理虚拟化、MMU虚拟化、IO虚拟化等多个议题，比较复杂，这里先不描述了。

#### 7. 总结

本文简单介绍了ARMv8-a中的一些概念，后续文章将会重点关注异常处理模型、security模型、virtualization模型。

_原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/armv8a_arch/armv8-a_overview.html)。_

标签: [arm64](http://www.wowotech.net/tag/arm64) [armv8-a](http://www.wowotech.net/tag/armv8-a) [exception_level;security;virtualization](http://www.wowotech.net/tag/exception_level%3Bsecurity%3Bvirtualization)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Concurrency Managed Workqueue之（一）：workqueue的基本概念](http://www.wowotech.net/irq_subsystem/workqueue.html) | [linux kernel的中断子系统之（九）：tasklet](http://www.wowotech.net/irq_subsystem/tasklet.html)»

**评论：**

**Roy**  
2022-12-08 15:54

拜阅  
  
Guest OS（Linux kernel、window等）位于EL1  
是否大致逻辑如下：  
若实现Hypervisor（EL2），则EL1可运行若干GuestOS，否则EL1运行OS。  
  
只有在异常发生时（或者异常处理返回时），才能切换Exception level（...）  
SMC或HVC 是否可直接触发EL切换，或者这类超级调用是先触发异常，在切换EL？没有具体深入研究过。

[回复](http://www.wowotech.net/armv8a_arch/armv8-a_overview.html#comment-8715)

**[lethe](http://android%20develope/)**  
2020-04-30 18:07

工作已经快三年了，刚毕业做driver的时后经常会看大佬技术blog和留言讨论，说真的很佩服十年如一日对技术的热情研究和心得分享，非常喜欢蜗窝，希望可以做一个像wowo一样真在的追求实质的developer，特此激励自己。

[回复](http://www.wowotech.net/armv8a_arch/armv8-a_overview.html#comment-7974)

**霍宏鹏**  
2017-12-13 11:20

通过LINUX DTS搜索，结识了您的博客，发现网上的文章都是基于您的文章来描述。后来看了您的其他文章，您的文章每一篇都是高水平，高质量，像是在写作、创作，真的很用心。您向技术致敬，我们向您致敬。

[回复](http://www.wowotech.net/armv8a_arch/armv8-a_overview.html#comment-6349)

**[wowo](http://www.wowotech.net/)**  
2017-12-13 13:44

@霍宏鹏：多谢鼓励，大家的认同就是我们最大的动力啊:-)

[回复](http://www.wowotech.net/armv8a_arch/armv8-a_overview.html#comment-6355)

**夏伟**  
2017-10-10 21:31

后来的Cortex-A15（ARMv7架构）通过Large Physical Address Extensions (LPAE) 技术，可以支持高达40bits的物理地址空间。但受限于32-bit的指令集，虚拟地址空间依旧只有32bits（4G）  
============================================================================  
@wowo  您好，最近刚好在研究LPAE，有个问题向您请教一下。  
LPAE能够扩大物理地址空间，但是在ARMv7架构（linux平台）中，虚拟地址空间只有4G，按照3:1的划分，内核空间只划分到了1G的虚拟地址空间。假如要在内核空间申请大片内存（>1G）,那内核虚拟地址空间肯定是不够用了。这样一来，扩大的物理地址空间如何能够得到有效合理利用？ 换句话说，在打开LPAE的情况下，如何扩大的物理地址空间进行充分利用呢？

[回复](http://www.wowotech.net/armv8a_arch/armv8-a_overview.html#comment-6099)

**[wowo](http://www.wowotech.net/)**  
2017-10-11 09:38

@夏伟：简单来说：每个进程的VA都是32bit，物理内存是40bit，不同进程的32bit可以映射到物理内存40bit的不同位置啊，所以肯定可以利用的。

[回复](http://www.wowotech.net/armv8a_arch/armv8-a_overview.html#comment-6100)

**夏伟**  
2017-10-11 23:04

@wowo：感谢！您的回答令我茅塞顿开，是我自己把思维局限于内核态的1G虚拟地址空间了。

[回复](http://www.wowotech.net/armv8a_arch/armv8-a_overview.html#comment-6102)

**buhui**  
2015-11-04 08:47

多么好的文章，期待蜗蜗一直写下去，技术本来是美好的，我们要享受它。

[回复](http://www.wowotech.net/armv8a_arch/armv8-a_overview.html#comment-2994)

**[wowo](http://www.wowotech.net/)**  
2015-11-04 09:25

@buhui：多谢鼓励！没错，享受技术，和享受一杯咖啡，从本质上是没区别的。

[回复](http://www.wowotech.net/armv8a_arch/armv8-a_overview.html#comment-2996)

**[jserv](http://https//about.me/jserv)**  
2015-09-15 15:45

感謝一直以來熱心分享，我是台灣的一位大學教師，日前指導學生彙整 ARMv8-A 相關題材，請多多指教: http://wiki.csie.ncku.edu.tw/embedded/ARMv8

[回复](http://www.wowotech.net/armv8a_arch/armv8-a_overview.html#comment-2580)

**wowo**  
2015-09-15 22:46

@jserv：感謝分享，已經閱讀，寫的還是蠻細緻的。ARMv8的內容是非常多的，希望你們能多研究一下這方面的內容。謝謝。

[回复](http://www.wowotech.net/armv8a_arch/armv8-a_overview.html#comment-2588)

**游客1**  
2020-05-16 19:48

@jserv：感谢您，我是本科生，最近在做ARMv8相关研究，您这个wiki帮了我大忙，做的真好。

[回复](http://www.wowotech.net/armv8a_arch/armv8-a_overview.html#comment-7995)

**老蛮熊**  
2015-09-02 10:23

hi , 楼主，不错，很好，  
  
另外：正因为有MMU的存在，内存寻址才可以和指令集剥离开来。 这句的表达容易误解，没有MMU，数据访问和指令访问也是分开的。编译链接的时候就是不同的有效地址。  
  
有效地址：程序编译链接时候设定的地址。  
虚拟地址：ＣＰＵ　内部把有效地址映射成的一种中间地址。  
物理地址： MMU把有效地址转为虚拟地址后最后放到总线上的地址（理解为DDR的地址线）

[回复](http://www.wowotech.net/armv8a_arch/armv8-a_overview.html#comment-2522)

**[wowo](http://www.wowotech.net/)**  
2015-09-02 17:26

@老蛮熊：多谢指正，确实，这个表达欠妥当。谢谢~~

[回复](http://www.wowotech.net/armv8a_arch/armv8-a_overview.html#comment-2525)

**wenky**  
2015-08-11 00:03

版主，A64指令集可不是指ARMv8指令集是64位的，"Each instruction in A64 is defined within a fixed length 32-bit instruction."。实际上指令集只是对指令的一种编码方式，32位的编码对目前来说已经够用了。64位主要是指寄存器的宽度是64位，另外内存寻址范围跟寄存器的宽度也没关系，取决于处理器内部的总线宽度，实际上ARMv8寻址也只用到了48位。Intel32位的处理器内部寻址总线也远远不止32位。阅读量越来越大、请及时修正~~

[回复](http://www.wowotech.net/armv8a_arch/armv8-a_overview.html#comment-2444)

**[wowo](http://www.wowotech.net/)**  
2015-08-11 09:15

@wenky：非常感谢wenky的建议，由于对ARM64的理解不是很深刻，有些表述不清或者错误的地方，还请多指正、讨论。  
关于A64指令集的定义，确实不太好表述，因此本文只是如实引用了ARM白皮书（ARMv8_white_paper_v5.pdf）中的说法：“Does ARMv8 need to be considering a full 64-bit instruction set architecture with its larger virtual address space architecture or not?”，也即64-bit指令集。至于这里的64-bit到底指的是什么，由于不想在这个介绍性的文档中带入太多信息，就没有提。是的，您的说法是正确的，指令编码是32位，通用寄存器等，是64位。  
关于内存寻址，您说的也很正确，和总线以及内存管理单元有关，和指令集无关，但这并不是A64关心的地方。对于物理内存，ARMv7+LPAE是可以解决4GB的限制，但虚拟内存怎么办？这才是A64关心的地方。  
正因为有MMU的存在，内存寻址才可以和指令集剥离开来，CPU访问虚拟地址，MMU将之转换为物理地址（理论上可以是任何范围）。但虚拟地址的范围由什么决定的呢？可以参考下面的表述（希望后续能以单独的文章和大家讨论AArch64的内存模型，这里就不再细述了）：    
The A64 instruction set supports 64-bit addresses. The valid address range is determined by the following factors:  
• The size of the implemented virtual address space.  
• Memory Management Unit (MMU) configuration settings.

[回复](http://www.wowotech.net/armv8a_arch/armv8-a_overview.html#comment-2448)

**wenky**  
2015-08-15 23:28

@wowo：O(∩_∩)O我也是接触不久一知半解、多多指教~~在这儿收获很多、谢谢~~

[回复](http://www.wowotech.net/armv8a_arch/armv8-a_overview.html#comment-2465)

**[daisy](http://www.wowotech.net/)**  
2015-07-27 16:03

有没有V8（A53/A57)流水线技术的描述文档，找了infocenter，没找到，想看下pc值是当前执行值的原因  
  
只看到这些，实在是太晦涩了  
Simple sequential execution  
The behavior of an implementation that fetches, decodes and completely executes each instruction before proceeding to the next instruction. Such an implementation performs no speculative accesses to memory, including to instruction memory. The implementation does not pipeline any phase of execution. In practice, this is the theoretical execution model that the architecture is based on, and ARM does not expect this model to correspond to a realistic implementation of the architecture.

[回复](http://www.wowotech.net/armv8a_arch/armv8-a_overview.html#comment-2323)

**[wowo](http://www.wowotech.net/)**  
2015-07-27 19:57

@daisy：抱歉，没有接触过这些:(

[回复](http://www.wowotech.net/armv8a_arch/armv8-a_overview.html#comment-2326)

**邵靖杰**  
2019-06-24 22:08

@daisy：看到这个为之一振，我也非常想找到这方面的资料，网络上现在都是关于老版本ARM8,9,11之类的流水线技术细节。今天在ARM官网上逛了很久仍然没有大收获。希望您如果找到相关资料可以通过邮件告知一声。感激不尽。

[回复](http://www.wowotech.net/armv8a_arch/armv8-a_overview.html#comment-7487)

**邵靖杰**  
2019-06-24 22:19

@邵靖杰：邮箱地址2248831103@qq.com

[回复](http://www.wowotech.net/armv8a_arch/armv8-a_overview.html#comment-7488)

**[shoujixiaodao](http://www.wowotech.net/)**  
2015-07-09 13:59

牛逼。

[回复](http://www.wowotech.net/armv8a_arch/armv8-a_overview.html#comment-2184)

1 [2](http://www.wowotech.net/armv8a_arch/armv8-a_overview.html/comment-page-2#comments)

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
    
    - [CPU 多核指令 —— WFE 原理](http://www.wowotech.net/armv8a_arch/499.html)
    - [教育漫谈(1)_《乌合之众》摘录](http://www.wowotech.net/tech_discuss/jymt1.html)
    - [智慧家庭之我见_多媒体篇](http://www.wowotech.net/tech_discuss/zhjt_dmt.html)
    - [PELT算法浅析](http://www.wowotech.net/process_management/pelt.html)
    - [Linux设备模型(7)_Class](http://www.wowotech.net/device_model/class.html)
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
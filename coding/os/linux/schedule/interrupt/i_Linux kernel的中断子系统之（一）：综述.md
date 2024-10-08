 作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2014-8-14 19:12 分类：[中断子系统](http://www.wowotech.net/sort/irq_subsystem)

一、前言

一个合格的linux驱动工程师需要对kernel中的中断子系统有深刻的理解，只有这样，在写具体driver的时候才能：

1、正确的使用linux kernel提供的的API，例如最著名的request_threaded_irq（request_irq）接口

2、正确使用同步机制保护驱动代码中的临界区

3、正确的使用kernel提供的softirq、tasklet、workqueue等机制来完成具体的中断处理

基于上面的原因，我希望能够通过一系列的文档来描述清楚linux kernel中的中断子系统方方面面的知识。一方面是整理自己的思绪，另外一方面，希望能够对其他的驱动工程师（或者想从事linux驱动工作的工程师）有所帮助。

二、中断系统相关硬件描述

中断硬件系统主要有三种器件参与，各个外设、中断控制器和CPU。各个外设提供irq request line，在发生中断事件的时候，通过irq request line上的电气信号向CPU系统请求处理。外设的irq request line太多，CPU需要一个小伙伴帮他，这就是Interrupt controller。Interrupt Controller是连接外设中断系统和CPU系统的桥梁。根据外设irq request line的多少，Interrupt Controller可以级联。CPU的主要功能是运算，因此CPU并不处理中断优先级，那是Interrupt controller的事情。对于CPU而言，一般有两种中断请求，例如：对于ARM，是IRQ和FIQ信号线，分别让ARM进入IRQ mode和FIQ mode。对于X86，有可屏蔽中断和不可屏蔽中断。

本章节不是描述具体的硬件，而是使用了HW block这样的概念。例如CPU HW block是只ARM core或者X86这样的实际硬件block的一个逻辑描述，实际中，可能是任何可能的CPU block。

  

1、HW中断系统的逻辑block图

我对HW中断系统之逻辑block diagram的理解如下图所示：

[![irq_thumb1](http://www.wowotech.net/content/uploadfile/201408/3c022613b452b88b4d1e2dc76ccd7a3a20140816061633.gif "irq_thumb1")](file:///C:/Documents%20and%20Settings/unigz/Local%20Settings/Temp/WindowsLiveWriter-429641856/supfiles134F60E/irq3.gif)

系统中有若干个CPU block用来接收中断事件并进行处理，若干个Interrupt controller形成树状的结构，汇集系统中所有外设的irq request line，并将中断事件分发给某一个CPU block进行处理。从接口层面看，主要有两类接口，一种是中断接口。有的实现中，具体中断接口的形态就是一个硬件的信号线，通过电平信号传递中断事件（ARM以及GIC组成的中断系统就是这么设计的）。有些系统采用了其他的方法来传递中断事件，比如x86＋APIC（Advanced Programmable Interrupt Controller）组成的系统，每个x86的核有一个Local APIC，这些Local APIC们通过ICC（Interrupt Controller Communication）bus连接到IO APIC上。IO APIC收集各个外设的中断，并翻译成总线上的message，传递给某个CPU上的Local APIC。因此，上面的红色线条也是逻辑层面的中断信号，可能是实际的PCB上的铜线（或者SOC内部的铜线），也可能是一个message而已。除了中断接口，CPU和Interrupt Controller之间还需要有控制信息的交流。Interrupt Controller会开放一些寄存器让CPU访问、控制。

  

2、多个Interrupt controller和多个cpu之间的拓扑结构

Interrupt controller有的是支持多个CPU core的（例如GIC、APIC等），有的不支持（例如S3C2410的中断控制器，X86平台的PIC等）。如果硬件平台中只有一个GIC的话，那么通过控制该GIC的寄存器可以将所有的外设中断，分发给连接在该interrupt controller上的CPU。如果有多个GIC呢（或者级联的interrupt controller都支持multi cpu core）？假设我们要设计一个非常复杂的系统，系统中有8个CPU，有2000个外设中断要处理，这时候你如何设计系统中的interrupt controller？如果使用GIC的话，我们需要两个GIC（一个GIC最多支持1024个中断源），一个是root GIC，另外一个是secondary GIC。这时候，你有两种方案：

（1）把8个cpu都连接到root GIC上，secondary GIC不接CPU。这时候原本挂接在secondary GIC的外设中断会输出到某个cpu，现在，只能是（通过某个cpu interface的irq signal）输到root GIC的某个SPI上。对于软件而言，这是一个比较简单的设计，secondary GIC的cpu interface的设定是固定不变的，永远是从一个固定的CPU interface输出到root GIC。这种方案的坏处是：这时候secondary GIC的PPI和SGI都是没有用的了。此外，在这种设定下，所有连接在secondary GIC上的外设中断要送达的target CPU是统一处理的，要么送去cpu0，要么cpu 5，不能单独控制。

（2）当然，你也可以让每个GIC分别连接4个CPU core，root GIC连接CPU0~CPU3，secondary GIC连接CPU4~CPU7。这种状态下，连接在root GIC的中断可以由CPU0~CPU3分担处理，连接在secondary GIC的中断可以由CPU4~CPU7分担处理。但这样，在中断处理方面看起来就体现不出8核的威力了。

注：上一节中的逻辑block示意图采用的就是方案一。

3、Interrupt controller把中断事件送给哪个CPU？

毫无疑问，只有支持multi cpu core的中断控制器才有这种幸福的烦恼。一般而言，中断控制器可以把中断事件上报给一个CPU或者一组CPU（包括广播到所有的CPU上去）。对于外设类型的中断，当然是送到一个cpu上就OK了，我看不出来要把这样的中断送给多个CPU进行处理的必要性。如果送达了多个cpu，实际上，也应该只有一个handler实际和外设进行交互，另外一个cpu上的handler的动作应该是这样的：发现该irq number对应的中断已经被另外一个cpu处理了，直接退出handler，返回中断现场。IPI的中断不存在这个限制，IPI更像一个CPU之间通信的机制，对这种中断广播应该是毫无压力。

实际上，从用户的角度看，其需求是相当复杂的，我们的目标可能包括：

（1）让某个IRQ number的中断由某个特定的CPU处理

（2）让某个特定的中断由几个CPU轮流处理

……

当然，具体的需求可能更加复杂，但是如何区分软件和硬件的分工呢？让硬件处理那么复杂的策略其实是不合理的，复杂的逻辑如果由硬件实现，那么就意味着更多的晶体管，更多的功耗。因此，最普通的做法就是为Interrupt Controller支持的每一个中断设定一个target cpu的控制接口（当然应该是以寄存器形式出现，对于GIC，这个寄存器就是Interrupt processor target register）。系统有多个cpu，这个控制接口就有多少个bit，每个bit代表一个CPU。如果该bit设定为1，那么该interrupt就上报给该CPU，如果为0，则不上报给该CPU。这样的硬件逻辑比较简单，剩余的控制内容就交给软件好了。例如如果系统有两个cpu core，某中断想轮流由两个CPU处理。那么当CPU0相应该中断进入interrupt handler的时候，可以将Interrupt processor target register中本CPU对应的bit设定为0，另外一个CPU的bit设定为1。这样，在下次中断发生的时候，interupt controller就把中断送给了CPU1。对于CPU1而言，在执行该中断的handler的时候，将Interrupt processor target register中CPU0的bit为设置为1，disable本CPU的比特位，这样在下次中断发生的时候，interupt controller就把中断送给了CPU0。这样软件控制的结果就是实现了特定中断由2个CPU轮流处理的算法。

  

4、更多的思考

面对这个HW中断系统之逻辑block diagram，我们其实可以提出更多的问题：

（1）中断控制器发送给CPU的中断是否可以收回？重新分发给另外一个CPU？

（2）系统中的中断如何分发才能获得更好的性能呢？

（3）中断分发的策略需要考虑哪些因素呢？

……

很多问题其实我也没有答案，慢慢思考，慢慢逼近真相吧。

二、中断子系统相关的软件框架

linux kernel的中断子系统相关的软件框架图如下所示：

[![isbs](http://www.wowotech.net/content/uploadfile/201408/3ec48c362ba8d614358dfdb779fd6c3020140814111248.gif "isbs")](http://www.wowotech.net/content/uploadfile/201408/4a6be3bcb547cdcd1c2d787a15db689720140814111245.gif) 

由上面的block图，我们可知linux kernel的中断子系统分成4个部分：

（1）硬件无关的代码，我们称之Linux kernel通用中断处理模块。无论是哪种CPU，哪种controller，其中断处理的过程都有一些相同的内容，这些相同的内容被抽象出来，和HW无关。此外，各个外设的驱动代码中，也希望能用一个统一的接口实现irq相关的管理（不和具体的中断硬件系统以及CPU体系结构相关）这些“通用”的代码组成了linux kernel interrupt subsystem的核心部分。

（2）CPU architecture相关的中断处理。 和系统使用的具体的CPU architecture相关。

（3）Interrupt controller驱动代码 。和系统使用的Interrupt controller相关。

（4）普通外设的驱动。这些驱动将使用Linux kernel通用中断处理模块的API来实现自己的驱动逻辑。

三、中断子系统文档规划

中断相关的文档规划如下：

1、linux kernel的中断子系统之（一），也就是本文，其实是一个导论，没有实际的内容，主要是给读者一个大概的软硬件框架。

2、[linux kernel的中断子系统之（二）：irq domain介绍](http://www.wowotech.net/linux_kenrel/irq-domain.html)。主要描述如何将一个HW interrupt ID转换成IRQ number。

3、[linux kernel的中断子系统之（三）：IRQ number和中断描述符](http://www.wowotech.net/linux_kenrel/interrupt_descriptor.html)。主要描述中断描述符相关的数据结构和接口API。

4、[linux kernel的中断子系统之（四）：high level irq event handler](http://www.wowotech.net/linux_kenrel/High_level_irq_event_handler.html)。

5、[linux kernel的中断子系统之（五）：driver API](http://www.wowotech.net/linux_kenrel/request_threaded_irq.html)。主要以一个普通的驱动程序为视角，看待linux interrupt subsystem提供的API，如何利用这些API，分配资源，是否资源，如何处理中断相关的同步问题等等。

6、[linux kernel的中断子系统之（六）：ARM中断处理过程](http://www.wowotech.net/linux_kenrel/irq_handler.html)，这份文档以ARM CPU为例，描述ARM相关的中断处理过程

7、[linux kernel的中断子系统之（七）：](http://www.wowotech.net/linux_kenrel/gic-irq-chip-driver.html)GIC代码分析，这份文档是以一个具体的interrupt controller为例，描述irq chip driver的代码构成情况。

8、[linux kernel的中断子系统之（八）：softirq](http://www.wowotech.net/linux_kenrel/soft-irq.html)

9、[linux kernel的中断子系统之（九）：tasklet](http://www.wowotech.net/irq_subsystem/tasklet.html)

  

  

_原创文章，转发请注明出处。蜗窝科技。[http://www.wowotech.net/linux_kenrel/interrupt_subsystem_architecture.html](http://www.wowotech.net/linux_kenrel/interrupt_subsystem_architecture.html)_

标签: [软件框架](http://www.wowotech.net/tag/%E8%BD%AF%E4%BB%B6%E6%A1%86%E6%9E%B6) [中断子系统](http://www.wowotech.net/tag/%E4%B8%AD%E6%96%AD%E5%AD%90%E7%B3%BB%E7%BB%9F)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [DMB DSB ISB以及SMP CPU 乱序](http://www.wowotech.net/77.html) | [开源的RF硬件平台](http://www.wowotech.net/75.html)»

**评论：**

**cc**  
2023-03-23 16:33

博主你好，说起来是在其他博文里看的东西，但是有个问题一直没有解决。问题是这样的，检测充电器的插拔，插和拔两个动作产生两种中断，但是只有一个检测中断的引脚。我看到文章的示例中，设备节点的interrupts属性中只有一个中断向量，但是获取中断号时却可以得到两个中断号，很疑惑，如果博主或者其他读者看到我的问题，可以解答一下，不胜感激。博文的地址：blog.csdn.net/qq_37858386/article/details/125298804，该文章下面的评论也是我的，可以参考，

[回复](http://www.wowotech.net/irq_subsystem/interrupt_subsystem_architecture.html#comment-8758)

**songxin8466419**  
2019-05-06 15:39

博主要是把linux内核分析这部分做成pdf文档就好了。

[回复](http://www.wowotech.net/irq_subsystem/interrupt_subsystem_architecture.html#comment-7394)

**Youthzzq**  
2019-10-16 19:49

@songxin8466419：已经有完整wowo全网站的pdf了

[回复](http://www.wowotech.net/irq_subsystem/interrupt_subsystem_architecture.html#comment-7698)

**伊斯科明**  
2019-10-17 20:25

@Youthzzq：在哪里能获取？

[回复](http://www.wowotech.net/irq_subsystem/interrupt_subsystem_architecture.html#comment-7701)

**Linux小白**  
2017-11-23 18:36

wowo你好，我目前遇到一个问题，我在调试的TP中断经常发生偶然性的实效，一般开机的时候我触摸TP是能有event事件上报的，但是当我不去触摸它的时候，也就是搁置一段时间再去触摸就会实效了，log如下：  
irq 72: nobody cared (try booting with the "irqpoll" option)  
[  904.066378] CPU: 0 PID: 0 Comm: swapper/0 Tainted: G        W      3.18.31 #3  
[  904.066391] Hardware name: Qualcomm Technologies, Inc. MSM8953 + PMI8950 QRD SKU3 (DT)  
[  904.066403] Call trace:  
[  904.066442] [<ffffffc000089b14>] dump_backtrace+0x0/0x270  
[  904.066460] [<ffffffc000089d98>] show_stack+0x14/0x1c  
[  904.066484] [<ffffffc000e74dcc>] dump_stack+0x80/0xa4  
[  904.066508] [<ffffffc0000f7ccc>] __report_bad_irq+0x38/0xdc  
[  904.066527] [<ffffffc0000f8194>] note_interrupt+0x1d0/0x274  
[  904.066543] [<ffffffc0000f5c74>] handle_irq_event_percpu+0x1d4/0x25c  
[  904.066558] [<ffffffc0000f5d48>] handle_irq_event+0x4c/0x78  
[  904.066575] [<ffffffc0000f8e68>] handle_level_irq+0xe0/0x114  
[  904.066591] [<ffffffc0000f519c>] generic_handle_irq+0x2c/0x3c  
[  904.066612] [<ffffffc00036869c>] msm_gpio_irq_handler+0x114/0x170  
[  904.066627] [<ffffffc0000f519c>] generic_handle_irq+0x2c/0x3c  
[  904.066642] [<ffffffc0000f5500>] __handle_domain_irq+0xb0/0xec  
[  904.066657] [<ffffffc000082560>] gic_handle_irq+0x68/0xa8  
[  904.066670] Exception stack(0xffffffc001773cd0 to 0xffffffc001773df0)  
[  904.066687] 3cc0:                                     00000001 00000000 000007be 00000000  
[  904.066706] 3ce0: 01773e20 ffffffc0 0094f7f4 ffffffc0 60000145 00000000 000001c0 00000000  
[  904.066724] 3d00: 001e3fbc 00000000 ffffffff 00ffffff 34155555 00000000 00000018 00000000  
[  904.066743] 3d20: 2cf60e00 002de543 155f19cc 00000004 00027b40 00000000 00000000 00000000  
[  904.066761] 3d40: b5193519 00000032 00085000 ffffffc0 00000000 00000000 00000001 00000000  
[  904.066779] 3d60: 04c5d91d 00000000 00000000 00000000 00000000 00000000 00000000 00000000  
[  904.066796] 3d80: 00000000 00000000 00000000 00000000 00000000 00000000 00000001 00000000  
[  904.066815] 3da0: 000007be 00000000 000003e8 00000000 a3542118 ffffffc0 00000001 00000000  
[  904.066832] 3dc0: 0000018c 00000000 b24549b0 ffffffc0 a3542168 ffffffc0 00000000 00000000  
[  904.066846] 3de0: a3542168 ffffffc0 01773e20 ffffffc0  
[  904.066861] [<ffffffc000085da8>] el1_irq+0x68/0xd4  
[  904.066887] [<ffffffc00094a82c>] cpuidle_enter_state+0xd0/0x224  
[  904.066902] [<ffffffc00094aa58>] cpuidle_enter+0x18/0x20  
[  904.066923] [<ffffffc0000e72ac>] cpu_startup_entry+0x2a4/0x3a0  
[  904.066939] [<ffffffc000e6df58>] rest_init+0x84/0x8c  
[  904.066964] [<ffffffc0016299c4>] start_kernel+0x3e8/0x410  
[  904.066974] handlers:  
[  904.066989] [<ffffffc0000f5d74>] irq_default_primary_handler threaded [<ffffffc0007836d0>] mxt_interrupt  
[  904.067020] Disabling IRQ #72

[回复](http://www.wowotech.net/irq_subsystem/interrupt_subsystem_architecture.html#comment-6236)

**[wowo](http://www.wowotech.net/)**  
2017-11-24 09:03

@Linux小白：这个irq没人处理了，例如没有调用request_irq。从你描述的场景里面推测，可能是某个driver（TP？）在一些场景（动态电源管理？）下把自己的irq free了？（可以查查相应的request_irq and free_irq组合，看看是否有不合理的地方）。

[回复](http://www.wowotech.net/irq_subsystem/interrupt_subsystem_architecture.html#comment-6237)

**bigpillow**  
2017-11-24 11:04

@Linux小白：可能没人register 这个irq  
也可能是share的时候，所有的IRQ function 都返回"这不是我的irq"

[回复](http://www.wowotech.net/irq_subsystem/interrupt_subsystem_architecture.html#comment-6238)

**wellxin**  
2017-10-22 09:59

各位，大大楼主，想问下你们这些知识是怎样获取的，我好佩服你们一个知识能掌握这么透彻。你们一般是看那些资料，书籍，，，才能获得这些知识的。好想跟着你们的步伐走。另外各位大大，有什么好书推荐吗。  
目前自己刚入门就只看过《LKD》,《宋宝华》，内核的一点点Documentation.想问下各位大大的学习路线

[回复](http://www.wowotech.net/irq_subsystem/interrupt_subsystem_architecture.html#comment-6131)

**Marcus**  
2018-11-20 22:32

@wellxin：同问

[回复](http://www.wowotech.net/irq_subsystem/interrupt_subsystem_architecture.html#comment-7044)

**wesley**  
2017-04-14 17:31

因此，上面的红色线条也是逻辑层面的中断信号，可能是实际的PCB上的铜线（或者SOC内部的铜线），也可能是一个message而已。除了中断接口，CPU和Interrupt Controller之间还需要有控制信息的交流。Interrupt Controller会开放一些寄存器让CPU访问、控制。  
  
这里的MESSAGE，是指PCIE的MSI吗？

[回复](http://www.wowotech.net/irq_subsystem/interrupt_subsystem_architecture.html#comment-5471)

**[linuxer](http://www.wowotech.net/)**  
2017-04-21 10:39

@wesley：这里的message是一个通用的概念而已，不是特指某种具体实现的东西。当然PCIE的MSI应该通过message传递interrupt信息的，应该是属于message这种类型的。

[回复](http://www.wowotech.net/irq_subsystem/interrupt_subsystem_architecture.html#comment-5477)

**maze**  
2016-08-01 15:09

窝窝们好。我是新手，不大理解gpio_to_irq和irq_of_parse_and_map这两个函数的区别，他们的返回值都一样。是获取irq_num的方式有区别么?

[回复](http://www.wowotech.net/irq_subsystem/interrupt_subsystem_architecture.html#comment-4359)

**jueduilingdu27**  
2016-05-20 11:23

大神，请教一个问题，小弟现在遇到这个一个情况，中断上升沿无效，下降沿又是可以的，能正确读出引脚的电平，但就是上升沿触发不了，麻烦帮我分析分析，这个问题要在哪方面入手去找，谢谢了，纠结了好几天了！！！！

[回复](http://www.wowotech.net/irq_subsystem/interrupt_subsystem_architecture.html#comment-3955)

**[wowo](http://www.wowotech.net/)**  
2016-05-20 14:59

@jueduilingdu27：你这里的中断，应该指的是GPIO外部中断吧？  
既然下降沿没有问题，则说明GPIO到GIC的通路是正确的。  
如果通过波形测量，也看到有上升沿出现的话（需要仔细测量，检查高电平是否够高，是否是没有拉上去，否则沿太平缓），说明硬件信号是没有问题的。  
那很有可能是GPIO控制器外部中断有关的寄存器没有配置好，可以检查一下，一般可以配置为：  
高电平触发、低电平触发、上升沿触发、下降沿触发、双边沿触发等等。

[回复](http://www.wowotech.net/irq_subsystem/interrupt_subsystem_architecture.html#comment-3956)

**jueduilingdu27**  
2016-05-20 15:29

@wowo：谢谢你的回答，对，是ｇｐｉｏ外部中断，而且其他的引脚都没问题，一样的代码，就把引脚换一下，就这个引脚不行，而且就是它的上升沿不行（但跟踪到驱动中的代码，它读有个地址ioread32(regs_umsk_isr_gpa + reg_ofst)）里的值总是０，所以到不了我们自己定义中断处理函数，而且也灭有中断计数，它是引脚复用的，另一个功能是ｉ２ｃ，不知道这会不会有什么影响，需要怎么处理一下不，我怀疑会不会是硬件的问题。

[回复](http://www.wowotech.net/irq_subsystem/interrupt_subsystem_architecture.html#comment-3957)

**[wowo](http://www.wowotech.net/)**  
2016-05-20 16:11

@jueduilingdu27：这个寄存器判断的是什么东西呢？  
和I2C功能复用？可以检查一下pinmux是否配置正确。  
另外，这个管脚有上拉或者下拉电阻吗？建议你们用示波器量一下波形。

[回复](http://www.wowotech.net/irq_subsystem/interrupt_subsystem_architecture.html#comment-3959)

**jueduilingdu27**  
2016-04-25 20:08

博主要是能把中断的设备树配置详细讲解一下就完美了

[回复](http://www.wowotech.net/irq_subsystem/interrupt_subsystem_architecture.html#comment-3855)

**[linuxer](http://www.wowotech.net/)**  
2016-04-26 10:51

@jueduilingdu27：在设备树相关的文档中有描述，但是还是不够系统，不过可以参考一下！

[回复](http://www.wowotech.net/irq_subsystem/interrupt_subsystem_architecture.html#comment-3859)

**jueduilingdu27**  
2016-04-26 14:31

@linuxer：博主是又问必答啊，非常感谢咯，感觉以后有什么搞不定的问题都可以向您讨教啊！！！

[回复](http://www.wowotech.net/irq_subsystem/interrupt_subsystem_architecture.html#comment-3862)

**南山南**  
2016-03-04 15:25

因此，最普通的做法就是为Interrupt Controller支持的每一个中断设定一个target cpu的控制接口（当然应该是以寄存器形式出现，对于GIC，这个寄存器就是Interrupt processor target register）  
  
  
假设CPU个数是x，中断有N个，那岂不得分配N*x个寄存器？要是内存可以理解，寄存器没有这么多吧？

[回复](http://www.wowotech.net/irq_subsystem/interrupt_subsystem_architecture.html#comment-3582)

**[lianghao](http://www.wowotech.net/)**  
2016-03-27 11:43

@南山南：每一个bit代表一个CPU，有x个CPU，需要(x+8-1)/8个字节代表所有CPU，有N个中断，那么总共需要N*(x+8-1)/8个字节，假设一个寄存器4字节，总共需要N*(x+8-1)/32个寄存器

[回复](http://www.wowotech.net/irq_subsystem/interrupt_subsystem_architecture.html#comment-3734)

**[wowo](http://www.wowotech.net/)**  
2015-12-24 14:02

@linux_emb：@linuxer：这个提议好阿，@wowo 怎么看。  
===================================================  
我也觉得这个提议很好啊。  
只不过，天朝地大物博，聚在一起也不容易啊（人力、财力）。  
而且，大家都是一帮技术宅啊，不太擅长搞活动。  
总结来说，蜗窝缺一个COO啊，且缺乏资金来源，不太好操作。

[回复](http://www.wowotech.net/irq_subsystem/interrupt_subsystem_architecture.html#comment-3279)

**linux_emb**  
2015-12-24 14:39

@wowo：那目前这种情况下可以约一个时间在群里一起聊聊，联络联络感情也好啊。  
很想和志同道合的同学们交流。  
可以每次探讨一个主题，就是思想交流的那种，不涉及具体细节。  
也可以探讨一些非技术的话题阿。。。。  
反正很喜欢思想交流带给人那种眼前一亮的感觉。

[回复](http://www.wowotech.net/irq_subsystem/interrupt_subsystem_architecture.html#comment-3281)

**[wowo](http://www.wowotech.net/)**  
2015-12-24 15:27

@linux_emb：个人愚见，谈人生谈理想不太适合在群里面啊，哪一个感情丰富的人愿意对着冷冰冰的机器袒露心声啊。我一直有个理想，开一个连锁的咖啡店----蜗窝咖啡馆，不为挣钱，最好会员制，每一个进来的人，都像已经认识的一样，可以畅所欲言，谈人生，聊技术……然后某一个天，突然发现，唉！您就是linux_emb兄啊？岂不快哉！  
至于技术交流，我觉得，一起做一个开源项目比较有效，为一个目标，享受过程！

[回复](http://www.wowotech.net/irq_subsystem/interrupt_subsystem_architecture.html#comment-3282)

**linux_emb**  
2015-12-24 16:11

@wowo：我也只是想表达交流的需求，也是觉得群里很不合适。但是想不出更好的办法。  
交流还是面对面的最有效，记得之前在学校，有一个人每次和他谈话总是很开心。可是后来毕业了，偶尔的电话联系好像也没有当年的激情。  
不知道wowo 对窝窝的规划是则样的？愿参与其中！

[回复](http://www.wowotech.net/irq_subsystem/interrupt_subsystem_architecture.html#comment-3285)

**[wowo](http://www.wowotech.net/)**  
2015-12-24 18:41

@linux_emb：不瞒你说，没有什么大的规划，因为财富不自由，大部分时间都要花在工作、挣钱上，仅余的一点精力，没办法做太多。不知道你有什么好的建议不？大家可以探讨一下。

[回复](http://www.wowotech.net/irq_subsystem/interrupt_subsystem_architecture.html#comment-3288)

**[blessed](http://www.wowotech.net/)**  
2016-01-24 21:08

@wowo：非常棒的想法，如果组织起来，我愿意参与到其中！

[回复](http://www.wowotech.net/irq_subsystem/interrupt_subsystem_architecture.html#comment-3439)

1 [2](http://www.wowotech.net/irq_subsystem/interrupt_subsystem_architecture.html/comment-page-2#comments) [3](http://www.wowotech.net/irq_subsystem/interrupt_subsystem_architecture.html/comment-page-3#comments)

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
    
    - [Linux Regulator Framework(1)_概述](http://www.wowotech.net/pm_subsystem/regulator_framework_overview.html)
    - [图解slub](http://www.wowotech.net/memory_management/426.html)
    - [Linux kernel的中断子系统之（一）：综述](http://www.wowotech.net/irq_subsystem/interrupt_subsystem_architecture.html)
    - [ARM64的启动过程之（四）：打开MMU](http://www.wowotech.net/armv8a_arch/turn-on-mmu.html)
    - [蓝牙协议中LQ和RSSI的原理及应用场景](http://www.wowotech.net/bluetooth/lqi_rssi.html)
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
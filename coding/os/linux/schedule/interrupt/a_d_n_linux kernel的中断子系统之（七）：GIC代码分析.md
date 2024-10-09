作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2014-9-4 16:59 分类：[中断子系统](http://www.wowotech.net/sort/irq_subsystem)

# 一、前言

GIC（Generic Interrupt Controller）是ARM公司提供的一个通用的中断控制器，其architecture specification目前有四个版本，V1～V4(V2最多支持8个ARM core，V3/V4支持更多的ARM core，主要用于ARM64服务器系统结构）。目前在ARM官方网站只能下载到Version 2的GIC architecture specification，因此，本文主要描述符合V2规范的GIC硬件及其驱动。

具体GIC硬件的实现形态有两种，一种是在ARM vensor研发自己的SOC的时候，会向ARM公司购买GIC的IP，这些IP包括的型号有：PL390，GIC-400，GIC-500。其中GIC-500最多支持128个 cpu core，它要求ARM core必须是ARMV8指令集的（例如Cortex-A57），符合GIC architecture specification version 3。另外一种形态是ARM vensor直接购买ARM公司的Cortex A9或者A15的IP，Cortex A9或者A15中会包括了GIC的实现，当然，这些实现也是符合GIC V2的规格。

本文在进行硬件描述的时候主要是以GIC-400为目标，当然，也会顺便提及一些Cortex A9或者A15上的GIC实现。

本文主要分析了linux kernel中GIC中断控制器的驱动代码（位于drivers/irqchip/irq-gic.c和irq-gic-common.c）。 irq-gic-common.c中是GIC V2和V3的通用代码，而irq-gic.c是V2 specific的代码，irq-gic-v3.c是V3 specific的代码，不在本文的描述范围。本文主要分成三个部分：第二章描述了GIC V2的硬件；第三章描述了GIC V2的初始化过程；第四章描述了底层的硬件call back函数。

注：具体的linux kernel的版本是linux-3.17-rc3。

# 二、GIC-V2的硬件描述

## 1、GIC-V2的输入和输出信号

### （1）GIC-V2的输入和输出信号示意图

要想理解一个building block（无论软件还是硬件），我们都可以先把它当成黑盒子，只是研究其input，output。GIC-V2的输入和输出信号的示意图如下（注：我们以GIC-400为例，同时省略了clock，config等信号）：
!\[\[Pasted image 20241009100741.png\]\]

（2）输入信号

上图中左边就是来自外设的interrupt source输入信号。分成两种类型，分别是PPI（Private Peripheral Interrupt）和SPI（Shared Peripheral Interrupt）。其实从名字就可以看出来两种类型中断信号的特点，PPI中断信号是CPU私有的，每个CPU都有其特定的PPI信号线。而SPI是所有CPU之间共享的。通过寄存器GICD_TYPER可以配置SPI的个数（最多480个）。GIC-400支持多少个SPI中断，其输入信号线就有多少个SPI interrupt request signal。同样的，通过寄存器GICD_TYPER也可以配置CPU interface的个数（最多8个），GIC-400支持多少个CPU interface，其输入信号线就提供多少组PPI中断信号线。一组PPI中断信号线包括6个实际的signal：

（a）nLEGACYIRQ信号线。对应interrupt ID 31，在bypass mode下（这里的bypass是指bypass GIC functionality，直接连接到某个processor上），nLEGACYIRQ可以直接连到对应CPU的nIRQCPU信号线上。在这样的设置下，该CPU不参与其他属于该CPU的PPI以及SPI中断的响应，而是特别为这一根中断线服务。
（b）nCNTPNSIRQ信号线。来自Non-secure physical timer的中断事件，对应interrupt ID 30。
（c）nCNTPSIRQ信号线。来自secure physical timer的中断事件，对应interrupt ID 29。
（d）nLEGACYFIQ信号线。对应interrupt ID 28。概念同nLEGACYIRQ信号线，不再描述。
（e）nCNTVIRQ信号线。对应interrupt ID 27。Virtual Timer Event，和虚拟化相关，这里不与描述。
（f）nCNTHPIRQ信号线。对应interrupt ID 26。Hypervisor Timer Event，和虚拟化相关，这里不与描述。

对于Cortex A15的GIC实现，其PPI中断信号线除了上面的6个，还有一个叫做Virtual Maintenance Interrupt，对应interrupt ID 25。

对于Cortex A9的GIC实现，其PPI中断信号线包括5根：

（a）nLEGACYIRQ信号线和nLEGACYFIQ信号线。对应interrupt ID 31和interrupt ID 28。这部分和上面一致。
（b）由于Cortext A9的每个处理器都有自己的Private timer和watch dog timer，这两个HW block分别使用了ID 29和ID 30
（c）Cortext A9内嵌一个global timer为系统内的所有processor共享，对应interrupt ID 27

关于private timer和global timer的描述，请参考时间子系统的相关文档。

关于一系列和虚拟化相关的中断，请参考虚拟化的系列文档。

（3）输出信号

所谓输出信号，其实就是GIC和各个CPU直接的接口，这些接口包括：

（a）触发CPU中断的信号。nIRQCPU和nFIQCPU信号线，熟悉ARM CPU的工程师对这两个信号线应该不陌生，主要用来触发ARM cpu进入IRQ mode和FIQ mode。

（b）Wake up信号。nFIQOUT和nIRQOUT信号线，去ARM CPU的电源管理模块，用来唤醒CPU的

（c）AXI slave interface signals。AXI（Advanced eXtensible Interface）是一种总线协议，属于AMBA规范的一部分。通过这些信号线，ARM CPU可以和GIC硬件block进行通信（例如寄存器访问）。

（4）中断号的分配

GIC-V2支持的中断类型有下面几种：

（a）外设中断（Peripheral interrupt）。有实际物理interrupt request signal的那些中断，上面已经介绍过了。

（b）软件触发的中断（SGI，Software-generated interrupt）。软件可以通过写GICD_SGIR寄存器来触发一个中断事件，这样的中断，可以用于processor之间的通信。

（c）虚拟中断（Virtual interrupt）和Maintenance interrupt。这两种中断和本文无关，不再赘述。

为了标识这些interrupt source，我们必须要对它们进行编码，具体的ID分配情况如下：

（a）ID0~ID31是用于分发到一个特定的process的interrupt。标识这些interrupt不能仅仅依靠ID，因为各个interrupt source都用同样的ID0~ID31来标识，因此识别这些interrupt需要interrupt ID ＋ CPU interface number。ID0~ID15用于SGI，ID16~ID31用于PPI。PPI类型的中断会送到其私有的process上，和其他的process无关。SGI是通过写GICD_SGIR寄存器而触发的中断。Distributor通过processor source ID、中断ID和target processor ID来唯一识别一个SGI。

（b）ID32~ID1019用于SPI。 这是GIC规范的最大size，实际上GIC-400最大支持480个SPI，Cortex-A15和A9上的GIC最多支持224个SPI。

2、GIC-V2的内部逻辑

（1）GIC的block diagram

GIC的block diagram如下图所示：

[![gic](http://www.wowotech.net/content/uploadfile/201409/049e88993db95cd9b35b6d9e8c1fa99920140904115854.gif "gic")](http://www.wowotech.net/content/uploadfile/201409/e1fbd5278ddff2a62b06d22df15848e920140904115852.gif)

GIC可以清晰的划分成两个block，一个block是Distributor（上图的左边的block），一个是CPU interface。CPU interface有两种，一种就是和普通processor接口，另外一种是和虚拟机接口的。Virtual CPU interface在本文中不会详细描述。

（2）Distributor 概述

Distributor的主要的作用是检测各个interrupt source的状态，控制各个interrupt source的行为，分发各个interrupt source产生的中断事件分发到指定的一个或者多个CPU interface上。虽然Distributor可以管理多个interrupt source，但是它总是把优先级最高的那个interrupt请求送往CPU interface。Distributor对中断的控制包括：

（1）中断enable或者disable的控制。Distributor对中断的控制分成两个级别。一个是全局中断的控制（GIC_DIST_CTRL）。一旦disable了全局的中断，那么任何的interrupt source产生的interrupt event都不会被传递到CPU interface。另外一个级别是对针对各个interrupt source进行控制（GIC_DIST_ENABLE_CLEAR），disable某一个interrupt source会导致该interrupt event不会分发到CPU interface，但不影响其他interrupt source产生interrupt event的分发。
（2）控制将当前优先级最高的中断事件分发到一个或者一组CPU interface。当一个中断事件分发到多个CPU interface的时候，GIC的内部逻辑应该保证只assert 一个CPU。
（3）优先级控制。
（4）interrupt属性设定。例如是level-sensitive还是edge-triggered
（5）interrupt group的设定

Distributor可以管理若干个interrupt source，这些interrupt source用ID来标识，我们称之interrupt ID。

（3）CPU interface

CPU interface这个block主要用于和process进行接口。该block的主要功能包括：

（a）enable或者disable CPU interface向连接的CPU assert中断事件。对于ARM，CPU interface block和CPU之间的中断信号线是nIRQCPU和nFIQCPU。如果disable了中断，那么即便是Distributor分发了一个中断事件到CPU interface，但是也不会assert指定的nIRQ或者nFIQ通知processor。

（b）ackowledging中断。processor会向CPU interface block应答中断（应答当前优先级最高的那个中断），中断一旦被应答，Distributor就会把该中断的状态从pending状态修改成active或者pending and active（这是和该interrupt source的信号有关，例如如果是电平中断并且保持了该asserted电平，那么就是pending and active）。processor ack了中断之后，CPU interface就会deassert nIRQCPU和nFIQCPU信号线。

（c）中断处理完毕的通知。当interrupt handler处理完了一个中断的时候，会向写CPU interface的寄存器从而通知GIC CPU已经处理完该中断。做这个动作一方面是通知Distributor将中断状态修改为deactive，另外一方面，CPU interface会priority drop，从而允许其他的pending的interrupt向CPU提交。

（d）设定priority mask。通过priority mask，可以mask掉一些优先级比较低的中断，这些中断不会通知到CPU。

（e）设定preemption的策略

（f）在多个中断事件同时到来的时候，选择一个优先级最高的通知processor

（4）实例

我们用一个实际的例子来描述GIC和CPU接口上的交互过程，具体过程如下：

[![xxx](http://www.wowotech.net/content/uploadfile/201409/61d3821d475611cca3ba2fd66232a80820140904115900.gif "xxx")](http://www.wowotech.net/content/uploadfile/201409/f37480c353855439540d8fd7f514eb3320140904115856.gif)

（注：图片太长，因此竖着放，看的时候有点费劲，就当活动一下脖子吧）

首先给出前提条件：

（a）N和M用来标识两个外设中断，N的优先级大于M
（b）两个中断都是SPI类型，level trigger，active-high
（c）两个中断被配置为去同一个CPU
（d）都被配置成group 0，通过FIQ触发中断

下面的表格按照时间轴来描述交互过程：

|   |   |
|---|---|
|时间|交互动作的描述|
|T0时刻|Distributor检测到M这个interrupt source的有效触发电平|
|T2时刻|Distributor将M这个interrupt source的状态设定为pending|
|T17时刻|大约15个clock之后，CPU interface拉低nFIQCPU信号线，向CPU报告M外设的中断请求。这时候，CPU interface的ack寄存器（GICC_IAR）的内容会修改成M interrupt source对应的ID|
|T42时刻|Distributor检测到N这个优先级更高的interrupt source的触发事件|
|T43时刻|Distributor将N这个interrupt source的状态设定为pending。同时，由于N的优先级更高，因此Distributor会标记当前优先级最高的中断|
|T58时刻|大约15个clock之后，CPU interface拉低nFIQCPU信号线，向CPU报告N外设的中断请求。当然，由于T17时刻已经assert CPU了，因此实际的电平信号仍然保持asserted。这时候，CPU interface的ack寄存器（GICC_IAR）的内容会被更新成N interrupt source的ID|
|T61时刻|软件通过读取ack寄存器的内容，获取了当前优先级最高的，并且状态是pending的interrupt ID（也就是N interrupt source对应的ID），通过读该寄存器，CPU也就ack了该interrupt source N。这时候，Distributor将N这个interrupt source的状态设定为pending and active（因为是电平触发，只要外部仍然有asserted的电平信号，那么一定就是pending的，而该中断是正在被CPU处理的中断，因此状态是pending and active）  <br>注意：T61标识CPU开始服务该中断|
|T64时刻|3个clock之后，由于CPU已经ack了中断，因此GIC中CPU interface模块 deassert nFIQCPU信号线，解除发向该CPU的中断请求|
|T126时刻|由于中断服务程序操作了N外设的控制寄存器（ack外设的中断），因此N外设deassert了其interrupt request signal|
|T128时刻|Distributor解除N外设的pending状态，因此N这个interrupt source的状态设定为active|
|T131时刻|软件操作End of Interrupt寄存器（向GICC_EOIR寄存器写入N对应的interrupt ID），标识中断处理结束。Distributor将N这个interrupt source的状态修改为idle  <br>注意：T61～T131是CPU服务N外设中断的的时间区域，这个期间，如果有高优先级的中断pending，会发生中断的抢占（硬件意义的），这时候CPU interface会向CPU assert 新的中断。|
|T146时刻|大约15个clock之后，Distributor向CPU interface报告当前pending且优先级最高的interrupt source，也就是M了。漫长的pending之后，M终于迎来了春天。CPU interface拉低nFIQCPU信号线，向CPU报告M外设的中断请求。这时候，CPU interface的ack寄存器（GICC_IAR）的内容会修改成M interrupt source对应的ID|
|T211时刻|CPU ack M中断（通过读GICC_IAR寄存器），开始处理低优先级的中断。|

# 三、GIC-V2 irq chip driver的初始化过程

在linux-3.17-rc3\\drivers\\irqchip目录下保存在各种不同的中断控制器的驱动代码，这个版本的内核支持了GICV3。irq-gic-common.c是通用的GIC的驱动代码，可以被各个版本的GIC使用。irq-gic.c是用于V2版本的GIC controller，而irq-gic-v3.c是用于V3版本的GIC controller。

1、GIC的device node和GIC irq chip driver的匹配过程

（1）irq chip driver中的声明

在linux-3.17-rc3\\drivers\\irqchip目录下的irqchip.h文件中定义了IRQCHIP_DECLARE宏如下：

```cpp
#define IRQCHIP_DECLARE(name, compat, fn) OF_DECLARE_2(irqchip, name, compat, fn)

#define OF_DECLARE_2(table, name, compat, fn) \  
OF_DECLARE(table, name, compat, fn, of_init_fn_2)

#define _OF_DECLARE(table, name, compat, fn, fn_type)            \  
static const struct of_device_id of_table##name        \  
__used __section(##table##of_table)            \  
= { .compatible = compat,                \  
.data = (fn == (fn_type)NULL) ? fn : fn  }
```

这个宏其实就是初始化了一个struct of_device_id的静态常量，并放置在\_\_irqchip_of_table section中。irq-gic.c文件中使用IRQCHIP_DECLARE来定义了若干个静态的struct of_device_id常量，如下：

```cpp
IRQCHIP_DECLARE(gic_400, "arm,gic-400", gic_of_init);  
IRQCHIP_DECLARE(cortex_a15_gic, "arm,cortex-a15-gic", gic_of_init);  
IRQCHIP_DECLARE(cortex_a9_gic, "arm,cortex-a9-gic", gic_of_init);  
IRQCHIP_DECLARE(cortex_a7_gic, "arm,cortex-a7-gic", gic_of_init);  
IRQCHIP_DECLARE(msm_8660_qgic, "qcom,msm-8660-qgic", gic_of_init);  
IRQCHIP_DECLARE(msm_qgic2, "qcom,msm-qgic2", gic_of_init);
```

兼容GIC-V2的GIC实现有很多，不过其初始化函数都是一个。在linux kernel编译的时候，你可以配置多个irq chip进入内核，编译系统会把所有的IRQCHIP_DECLARE宏定义的数据放入到一个特殊的section中（section name是\_\_irqchip_of_table），我们称这个特殊的section叫做irq chip table。这个table也就保存了kernel支持的所有的中断控制器的ID信息（最重要的是驱动代码初始化函数和DT compatible string）。我们来看看struct of_device_id的定义：

> struct of_device_id\
> {\
> char    name\[32\];－－－－－－要匹配的device node的名字\
> char    type\[32\];－－－－－－－要匹配的device node的类型\
> char    compatible\[128\];－－－匹配字符串（DT compatible string），用来匹配适合的device node\
> const void \*data;－－－－－－－－对于GIC，这里是初始化函数指针\
> };

这个数据结构主要被用来进行Device node和driver模块进行匹配用的。从该数据结构的定义可以看出，在匹配过程中，device name、device type和DT compatible string都是考虑的因素。更细节的内容请参考\_\_of_device_is_compatible函数。

（2）device node

不同的GIC-V2的实现总会有一些不同，这些信息可以通过Device tree的机制来传递。Device node中定义了各种属性，其中就包括了memory资源，IRQ描述等信息，这些信息需要在初始化的时候传递给具体的驱动，因此需要一个Device node和driver模块的匹配过程。在Device Tree模块中会包括系统中所有的device node，如果我们的系统使用了GIC-400，那么系统的device node数据库中会有一个node是GIC-400的，一个示例性的GIC-400的device node（我们以瑞芯微的RK3288处理器为例）定义如下：

> gic: interrupt-controller@ffc01000 {\
> compatible = "arm,gic-400";\
> interrupt-controller;\
> #interrupt-cells = \<3>;\
> #address-cells = \<0>;
>
> reg = \<0xffc01000 0x1000="">,－－－－Distributor address range\
> \<0xffc02000 0x1000="">,－－－－－CPU interface address range\
> \<0xffc04000 0x2000="">,－－－－－Virtual interface control block\
> \<0xffc06000 0x2000="">;－－－－－Virtual CPU interfaces\
> interrupts = ;\
> };

（3）device node和irq chip driver的匹配

在machine driver初始化的时候会调用irqchip_init函数进行irq chip driver的初始化。在driver/irqchip/irqchip.c文件中定义了irqchip_init函数，如下：

```cpp
void init irqchip_init(void) {  
of_irq_init(__irqchip_begin);  
}
```

`__irqchip_begin` 就是内核irq chip table的首地址，这个table也就保存了kernel支持的所有的中断控制器的ID信息（用于和device node的匹配）。of_irq_init函数执行之前，系统已经完成了device tree的初始化，因此系统中的所有的设备节点都已经形成了一个树状结构，每个节点代表一个设备的device node。of_irq_init是在所有的device node中寻找中断控制器节点，形成树状结构（系统可以有多个interrupt controller，之所以形成中断控制器的树状结构，是为了让系统中所有的中断控制器驱动按照一定的顺序进行初始化）。之后，从root interrupt controller节点开始，对于每一个interrupt controller的device node，扫描irq chip table，进行匹配，一旦匹配到，就调用该interrupt controller的初始化函数，并把该中断控制器的device node以及parent中断控制器的device node作为参数传递给irq chip driver。。具体的匹配过程的代码属于Device Tree模块的内容，更详细的信息可以参考[Device Tree代码分析文档](http://www.wowotech.net/linux_kenrel/dt-code-analysis.html)。

2、GIC driver初始化代码分析

（1）gic_of_init的代码如下：

> int \_\_init gic_of_init(struct device_node \*node, struct device_node \*parent)\
> {\
> void \_\_iomem \*cpu_base;\
> void \_\_iomem \*dist_base;\
> u32 percpu_offset;\
> int irq;
>
> dist_base = of_iomap(node, 0);----------------映射GIC Distributor的寄存器地址空间
>
> cpu_base = of_iomap(node, 1);----------------映射GIC CPU interface的寄存器地址空间
>
> if (of_property_read_u32(node, "cpu-offset", &percpu_offset))--------处理cpu-offset属性。\
> percpu_offset = 0;
>
> gic_init_bases(gic_cnt, -1, dist_base, cpu_base, percpu_offset, node);))-----主处理过程，后面详述\
> if (!gic_cnt)\
> gic_init_physaddr(node); -----对于不支持big.LITTLE switcher（CONFIG_BL_SWITCHER）的系统，该函数为空。
>
> if (parent) {--------处理interrupt级联\
> irq = irq_of_parse_and_map(node, 0); －－－解析second GIC的interrupts属性，并进行mapping，返回IRQ number\
> gic_cascade_irq(gic_cnt, irq);\
> }\
> gic_cnt++;\
> return 0;\
> }

我们首先看看这个函数的参数，node参数代表需要初始化的那个interrupt controller的device node，parent参数指向其parent。在映射GIC-400的memory map I/O space的时候，我们只是映射了Distributor和CPU interface的寄存器地址空间，和虚拟化处理相关的寄存器没有映射，因此这个版本的GIC driver应该是不支持虚拟化的（不知道后续版本是否支持，在一个嵌入式平台上支持虚拟化有实际意义吗？最先支持虚拟化的应该是ARM64+GICV3/4这样的平台）。

要了解cpu-offset属性，首先要了解什么是banked register。所谓banked register就是在一个地址上提供多个寄存器副本。比如说系统中有四个CPU，这些CPU访问某个寄存器的时候地址是一样的，但是对于banked register，实际上，不同的CPU访问的是不同的寄存器，虽然它们的地址是一样的。如果GIC没有banked register，那么需要提供根据CPU index给出一系列地址偏移，而地址偏移=cpu-offset * cpu-nr。

interrupt controller可以级联。对于root GIC，其传入的parent是NULL，因此不会执行级联部分的代码。对于second GIC，它是作为其parent（root GIC）的一个普通的irq source，因此，也需要注册该IRQ的handler。由此可见，非root的GIC的初始化分成了两个部分：一部分是作为一个interrupt controller，执行和root GIC一样的初始化代码。另外一方面，GIC又作为一个普通的interrupt generating device，需要象一个普通的设备驱动一样，注册其中断handler。理解irq_of_parse_and_map需要irq domain的知识，请参考[linux kernel的中断子系统之（二）：irq domain介绍](http://www.wowotech.net/linux_kenrel/irq-domain.html)。

（2）gic_init_bases的代码如下：

> void \_\_init gic_init_bases(unsigned int gic_nr, int irq_start,\
> void \_\_iomem \*dist_base, void \_\_iomem \*cpu_base,\
> u32 percpu_offset, struct device_node \*node)\
> {\
> irq_hw_number_t hwirq_base;\
> struct gic_chip_data \*gic;\
> int gic_irqs, irq_base, i;
>
> gic = &gic_data\[gic_nr\]; \
> gic->dist_base.common_base = dist_base; －－－－省略了non banked的情况\
> gic->cpu_base.common_base = cpu_base; \
> gic_set_base_accessor(gic, gic_get_common_base);
>
> for (i = 0; i \< NR_GIC_CPU_IF; i++) －－－后面会具体描述gic_cpu_map的含义\
> gic_cpu_map\[i\] = 0xff;
>
> if (gic_nr == 0 && (irq_start & 31) > 0) { －－－－－－－－－－－－－－－－－－－－（a）\
> hwirq_base = 16;\
> if (irq_start != -1)\
> irq_start = (irq_start & ~31) + 16;\
> } else {\
> hwirq_base = 32;\
> }
>
> gic_irqs = readl_relaxed(gic_data_dist_base(gic) + GIC_DIST_CTR) & 0x1f; －－－－（b）\
> gic_irqs = (gic_irqs + 1) * 32;\
> if (gic_irqs > 1020)\
> gic_irqs = 1020;\
> gic->gic_irqs = gic_irqs;
>
> gic_irqs -= hwirq_base;－－－－－－－－－－－－－－－－－－－－－－－－－－－－（c）
>
> if (of_property_read_u32(node, "arm,routable-irqs",－－－－－－－－－－－－－－－－（d）\
> &nr_routable_irqs)) {\
> irq_base = irq_alloc_descs(irq_start, 16, gic_irqs,  numa_node_id()); －－－－－－－（e）\
> if (IS_ERR_VALUE(irq_base)) {\
> WARN(1, "Cannot allocate irq_descs @ IRQ%d, assuming pre-allocated\\n",\
> irq_start);\
> irq_base = irq_start;\
> }
>
> gic->domain = irq_domain_add_legacy(node, gic_irqs, irq_base, －－－－－－－（f）\
> hwirq_base, &gic_irq_domain_ops, gic);\
> } else {\
> gic->domain = irq_domain_add_linear(node, nr_routable_irqs, －－－－－－－－（f）\
> &gic_irq_domain_ops,\
> gic);\
> }
>
> if (gic_nr == 0) { －－－只对root GIC操作，因为设定callback、注册Notifier只需要一次就OK了\
> #ifdef CONFIG_SMP\
> set_smp_cross_call(gic_raise_softirq);－－－－－－－－－－－－－－－－－－（g）\
> register_cpu_notifier(&gic_cpu_notifier);－－－－－－－－－－－－－－－－－－（h）\
> #endif\
> set_handle_irq(gic_handle_irq); －－－这个函数名字也不好，实际上是设定arch相关的irq handler\
> }
>
> gic_chip.flags |= gic_arch_extn.flags;\
> gic_dist_init(gic);---------具体的硬件初始代码，参考下节的描述\
> gic_cpu_init(gic);\
> gic_pm_init(gic);\
> }

（a）gic_nr标识GIC number，等于0就是root GIC。hwirq的意思就是GIC上的HW interrupt ID，并不是GIC上的每个interrupt ID都有map到linux IRQ framework中的一个IRQ number，对于SGI，是属于软件中断，用于CPU之间通信，没有必要进行HW interrupt ID到IRQ number的mapping。变量hwirq_base表示该GIC上要进行map的base ID，hwirq_base = 16也就意味着忽略掉16个SGI。对于系统中其他的GIC，其PPI也没有必要mapping，因此hwirq_base = 32。

在本场景中，irq_start ＝ -1，表示不指定IRQ number。有些场景会指定IRQ number，这时候，需要对IRQ number进行一个对齐的操作。

（b）变量gic_irqs保存了该GIC支持的最大的中断数目。该信息是从GIC_DIST_CTR寄存器（这是V1版本的寄存器名字，V2中是GICD_TYPER，Interrupt Controller Type Register,）的低五位ITLinesNumber获取的。如果ITLinesNumber等于N，那么最大支持的中断数目是32(N+1)。此外，GIC规范规定最大的中断数目不能超过1020，1020-1023是有特别用户的interrupt ID。

（c）减去不需要map（不需要分配IRQ）的那些interrupt ID，OK，这时候gic_irqs的数值终于和它的名字一致了。gic_irqs从字面上看不就是该GIC需要分配的IRQ number的数目吗？

（d）of_property_read_u32函数把arm,routable-irqs的属性值读出到nr_routable_irqs变量中，如果正确返回0。在有些SOC的设计中，外设的中断请求信号线不是直接接到GIC，而是通过crossbar/multiplexer这个的HW block连接到GIC上。arm,routable-irqs这个属性用来定义那些不直接连接到GIC的中断请求数目。

（e）对于那些直接连接到GIC的情况，我们需要通过调用irq_alloc_descs分配中断描述符。如果irq_start大于0，那么说明是指定IRQ number的分配，对于我们这个场景，irq_start等于-1，因此不指定IRQ 号。如果不指定IRQ number的，就需要搜索，第二个参数16就是起始搜索的IRQ number。gic_irqs指明要分配的irq number的数目。如果没有正确的分配到中断描述符，程序会认为可能是之前已经准备好了。

（f）这段代码主要是向系统中注册一个irq domain的数据结构。为何需要struct irq_domain这样一个数据结构呢？从linux kernel的角度来看，任何外部的设备的中断都是一个异步事件，kernel都需要识别这个事件。在内核中，用IRQ number来标识某一个设备的某个interrupt request。有了IRQ number就可以定位到该中断的描述符（struct irq_desc）。但是，对于中断控制器而言，它不并知道IRQ number，它只是知道HW interrupt number（中断控制器会为其支持的interrupt source进行编码，这个编码被称为Hardware interrupt number ）。不同的软件模块用不同的ID来识别interrupt source，这样就需要映射了。如何将Hardware interrupt number 映射到IRQ number呢？这需要一个translation object，内核定义为struct irq_domain。

每个interrupt controller都会形成一个irq domain，负责解析其下游的interrut source。如果interrupt controller有级联的情况，那么一个非root interrupt controller的中断控制器也是其parent irq domain的一个普通的interrupt source。struct irq_domain定义如下：

> struct irq_domain {\
> ……\
> const struct irq_domain_ops \*ops;\
> void \*host_data;
>
> ……\
> };

这个数据结构是属于linux kernel通用中断子系统的一部分，我们这里只是描述相关的数据成员。host_data成员是底层interrupt controller的私有数据，linux kernel通用中断子系统不应该修改它。对于GIC而言，host_data成员指向一个struct gic_chip_data的数据结构，定义如下：

> struct gic_chip_data {\
> union gic_base dist_base;－－－－－－－－－－－－－－－－－－GIC Distributor的基地址空间\
> union gic_base cpu_base;－－－－－－－－－－－－－－－－－－GIC CPU interface的基地址空间\
> #ifdef CONFIG_CPU_PM－－－－－－－－－－－－－－－－－－－－GIC 电源管理相关的成员\
> u32 saved_spi_enable\[DIV_ROUND_UP(1020, 32)\];\
> u32 saved_spi_conf\[DIV_ROUND_UP(1020, 16)\];\
> u32 saved_spi_target\[DIV_ROUND_UP(1020, 4)\];\
> u32 \_\_percpu \*saved_ppi_enable;\
> u32 \_\_percpu \*saved_ppi_conf;\
> #endif\
> struct irq_domain \*domain;－－－－－－－－－－－－－－－－－该GIC对应的irq domain数据结构\
> unsigned int gic_irqs;－－－－－－－－－－－－－－－－－－－GIC支持的IRQ的数目\
> #ifdef CONFIG_GIC_NON_BANKED\
> void \_\_iomem \*(\*get_base)(union gic_base \*);\
> #endif\
> };

对于GIC支持的IRQ的数目，这里还要赘述几句。实际上并非GIC支持多少个HW interrupt ID，其就支持多少个IRQ。对于SGI，其处理比较特别，并不归入IRQ number中。因此，对于GIC而言，其SGI（从0到15的那些HW interrupt ID）不需要irq domain进行映射处理，也就是说SGI没有对应的IRQ number。如果系统越来越复杂，一个GIC不能支持所有的interrupt source（目前GIC支持1020个中断源，这个数目已经非常的大了），那么系统还需要引入secondary GIC，这个GIC主要负责扩展外设相关的interrupt source，也就是说，secondary GIC的SGI和PPI都变得冗余了（这些功能，primary GIC已经提供了）。这些信息可以协助理解代码中的hwirq_base的设定。

在注册GIC的irq domain的时候还有一个重要的数据结构gic_irq_domain_ops，其类型是struct irq_domain_ops ，对于GIC，其irq domain的操作函数是gic_irq_domain_ops，定义如下：

> static const struct irq_domain_ops gic_irq_domain_ops = {\
> .map = gic_irq_domain_map,\
> .unmap = gic_irq_domain_unmap,\
> .xlate = gic_irq_domain_xlate,\
> };

irq domain的概念是一个通用中断子系统的概念，在具体的irq chip driver这个层次，我们需要一些解析GIC binding，创建IRQ number和HW interrupt ID的mapping的callback函数，更具体的解析参考后文的描述。

漫长的准备过程结束后，具体的注册比较简单，调用irq_domain_add_legacy或者irq_domain_add_linear进行注册就OK了。关于这两个接口请参考[linux kernel的中断子系统之（二）：irq domain介绍](http://www.wowotech.net/linux_kenrel/irq-domain.html)。

（g） 一个函数名字是否起的好足可以看出工程师的功力。set_smp_cross_call这个函数看名字也知道它的含义，就是设定一个多个CPU直接通信的callback函数。当一个CPU core上的软件控制行为需要传递到其他的CPU上的时候（例如在某一个CPU上运行的进程调用了系统调用进行reboot），就会调用这个callback函数。对于GIC，这个callback定义为gic_raise_softirq。这个函数名字起的不好，直观上以为是和softirq相关，实际上其实是触发了IPI中断。

（h）在multi processor环境下，当processor状态发送变化的时候（例如online，offline），需要把这些事件通知到GIC。而GIC driver在收到来自CPU的事件后会对cpu interface进行相应的设定。

# 3、GIC硬件初始化

（1）Distributor初始化，代码如下：

> static void \_\_init gic_dist_init(struct gic_chip_data \*gic)\
> {\
> unsigned int i;\
> u32 cpumask;\
> unsigned int gic_irqs = gic->gic_irqs;－－－－－－－－－获取该GIC支持的IRQ的数目\
> void \_\_iomem \*base = gic_data_dist_base(gic); －－－－获取该GIC对应的Distributor基地址
>
> writel_relaxed(0, base + GIC_DIST_CTRL); －－－－－－－－－－－（a）
>
> cpumask = gic_get_cpumask(gic);－－－－－－－－－－－－－－－（b）\
> cpumask |= cpumask \<\< 8;\
> cpumask |= cpumask \<\< 16;－－－－－－－－－－－－－－－－－－（c）\
> for (i = 32; i \< gic_irqs; i += 4)\
> writel_relaxed(cpumask, base + GIC_DIST_TARGET + i * 4 / 4); －－（d）
>
> gic_dist_config(base, gic_irqs, NULL); －－－－－－－－－－－－－－－（e）
>
> writel_relaxed(1, base + GIC_DIST_CTRL);－－－－－－－－－－－－－（f）\
> }

（a）Distributor Control Register用来控制全局的中断forward情况。写入0表示Distributor不向CPU interface发送中断请求信号，也就disable了全部的中断请求（group 0和group 1），CPU interace再也收不到中断请求信号了。在初始化的最后，step（f）那里会进行enable的动作（这里只是enable了group 0的中断）。在初始化代码中，并没有设定interrupt source的group（寄存器是GIC_DIST_IGROUP），我相信缺省值就是设定为group 0的。

（b）我们先看看gic_get_cpumask的代码：

> static u8 gic_get_cpumask(struct gic_chip_data \*gic)\
> {\
> void \_\_iomem \*base = gic_data_dist_base(gic);\
> u32 mask, i;
>
> for (i = mask = 0; i \< 32; i += 4) {\
> mask = readl_relaxed(base + GIC_DIST_TARGET + i);\
> mask |= mask >> 16;\
> mask |= mask >> 8;\
> if (mask)\
> break;\
> }
>
> return mask;\
> }

这里操作的寄存器是Interrupt Processor Targets Registers，该寄存器组中，每个GIC上的interrupt ID都有8个bit来控制送达的target CPU。我们来看看下面的图片：

[![cpu mask](http://www.wowotech.net/content/uploadfile/201409/b82a9c1eec7bfc66eed6ed4086d9d80420140904115902.gif "cpu mask")](http://www.wowotech.net/content/uploadfile/201409/eb40e557055dc9efc3633f73ff0ad3b520140904115901.gif)

GIC_DIST_TARGETn（Interrupt Processor Targets Registers）位于Distributor HW block中，能控制送达的CPU interface，并不是具体的CPU，如果具体的实现中CPU interface和CPU是严格按照上图中那样一一对应，那么GIC_DIST_TARGET送达了CPU Interface n，也就是送达了CPU n。当然现实未必如你所愿，那么怎样来获取这个CPU的mask呢？我们知道SGI和PPI不需要使用GIC_DIST_TARGET控制target CPU。SGI送达目标CPU有自己特有的寄存器来控制（Software Generated Interrupt Register），对于PPI，其是CPU私有的，因此不需要控制target CPU。GIC_DIST_TARGET0～GIC_DIST_TARGET7是控制0～31这32个interrupt ID（SGI和PPI）的target CPU的，但是实际上SGI和PPI是不需要控制target CPU的，因此，这些寄存器是read only的，读取这些寄存器返回的就是cpu mask值。假设CPU0接在CPU interface 4上，那么运行在CPU 0上的程序在读GIC_DIST_TARGET0～GIC_DIST_TARGET7的时候，返回的就是0b00010000。

当然，由于GIC-400只支持8个CPU，因此CPU mask值只需要8bit，但是寄存器GIC_DIST_TARGETn返回32个bit的值，怎么对应？很简单，cpu mask重复四次就OK了。了解了这些知识，回头看代码就很简单了。

（c）step （b）中获取了8个bit的cpu mask值，通过简单的copy，扩充为32个bit，每8个bit都是cpu mask的值，这么做是为了下一步设定所有IRQ（对于GIC而言就是SPI类型的中断）的CPU mask。
（d）设定每个SPI类型的中断都是只送达该CPU。
（e）配置GIC distributor的其他寄存器，代码如下：

```cpp
void init gic_dist_config(void __iomem base, int gic_irqs,  void (*sync_access)(void)) {  
unsigned int i;

/* Set all global interrupts to be level triggered, active low.    */  
for (i = 32; i < gic_irqs; i += 16)  
writel_relaxed(0, base + GIC_DIST_CONFIG + i / 4);

/ Set priority on all global interrupts.   /  
for (i = 32; i < gic_irqs; i += 4)  
writel_relaxed(0xa0a0a0a0, base + GIC_DIST_PRI + i);

/ Disable all interrupts.  Leave the PPI and SGIs alone as they are enabled by redistributor registers.    /  
for (i = 32; i < gic_irqs; i += 32)  
writel_relaxed(0xffffffff, base + GIC_DIST_ENABLE_CLEAR + i / 8);

if (sync_access)  
sync_access();  
}
```

程序的注释已经非常清楚了，这里就不细述了。需要注意的是：这里设定的都是缺省值，实际上，在各种driver的初始化过程中，还是有可能改动这些设置的（例如触发方式）。

（2）CPU interface初始化，代码如下：

> static void gic_cpu_init(struct gic_chip_data \*gic)\
> {\
> void \_\_iomem \*dist_base = gic_data_dist_base(gic);－－－－－－－Distributor的基地址空间\
> void \_\_iomem \*base = gic_data_cpu_base(gic);－－－－－－－CPU interface的基地址空间\
> unsigned int cpu_mask, cpu = smp_processor_id();－－－－－－获取CPU的逻辑ID\
> int i;
>
> cpu_mask = gic_get_cpumask(gic);－－－－－－－－－－－－－（a）\
> gic_cpu_map\[cpu\] = cpu_mask;
>
> for (i = 0; i \< NR_GIC_CPU_IF; i++)\
> if (i != cpu)\
> gic_cpu_map\[i\] &= ~cpu_mask; －－－－－－－－－－－－（b）
>
> gic_cpu_config(dist_base, NULL); －－－－－－－－－－－－－－（c）
>
> writel_relaxed(0xf0, base + GIC_CPU_PRIMASK);－－－－－－－（d）\
> writel_relaxed(1, base + GIC_CPU_CTRL);－－－－－－－－－－－（e）\
> }

（a）系统软件实际上是使用CPU 逻辑ID这个概念的，通过smp_processor_id可以获得本CPU的逻辑ID。gic_cpu_map这个全部lookup table就是用CPU 逻辑ID作为所以，去寻找其cpu mask，后续通过cpu mask值来控制中断是否送达该CPU。在gic_init_bases函数中，我们将该lookup table中的值都初始化为0xff，也就是说不进行mask，送达所有的CPU。这里，我们会进行重新修正。

（b）清除lookup table中其他entry中本cpu mask的那个bit。

（c）设定SGI和PPI的初始值。具体代码如下：

> void gic_cpu_config(void \_\_iomem \*base, void (\*sync_access)(void))\
> {\
> int i;
>
> /\* Deal with the banked PPI and SGI interrupts - disable all\
> \* PPI interrupts, ensure all SGI interrupts are enabled.     \*/\
> writel_relaxed(0xffff0000, base + GIC_DIST_ENABLE_CLEAR);\
> writel_relaxed(0x0000ffff, base + GIC_DIST_ENABLE_SET);
>
> /\* Set priority on PPI and SGI interrupts    \*/\
> for (i = 0; i \< 32; i += 4)\
> writel_relaxed(0xa0a0a0a0, base + GIC_DIST_PRI + i * 4 / 4);
>
> if (sync_access)\
> sync_access();\
> }

程序的注释已经非常清楚了，这里就不细述了。

（d）通过Distributor中的寄存器可以控制送达CPU interface，中断来到了GIC的CPU interface是否可以真正送达CPU呢？也不一定，还有一道关卡，也就是CPU interface中的Interrupt Priority Mask Register。这个寄存器设定了一个中断优先级的值，只有中断优先级高过该值的中断请求才会被送到CPU上去。我们在前面初始化的时候，给每个interrupt ID设定的缺省优先级是0xa0，这里设定的priority filter的优先级值是0xf0。数值越小，优先级越过。因此，这样的设定就是让所有的interrupt source都可以送达CPU，在CPU interface这里不做控制了。

（e）设定CPU interface的control register。enable了group 0的中断，disable了group 1的中断，group 0的interrupt source触发IRQ中断（而不是FIQ中断）。

（3）GIC电源管理初始化，代码如下：

> static void \_\_init gic_pm_init(struct gic_chip_data \*gic)\
> {\
> gic->saved_ppi_enable = \_\_alloc_percpu(DIV_ROUND_UP(32, 32) * 4, sizeof(u32));
>
> gic->saved_ppi_conf = \_\_alloc_percpu(DIV_ROUND_UP(32, 16) * 4,  sizeof(u32));
>
> if (gic == &gic_data\[0\])\
> cpu_pm_register_notifier(&gic_notifier_block);\
> }

这段代码前面主要是分配两个per cpu的内存。这些内存在系统进入sleep状态的时候保存PPI的寄存器状态信息，在resume的时候，写回寄存器。对于root GIC，需要注册一个和电源管理的事件通知callback函数。不得不吐槽一下gic_notifier_block和gic_notifier这两个符号的命名，看不出来和电源管理有任何关系。更优雅的名字应该包括pm这样的符号，以便让其他工程师看到名字就立刻知道是和电源管理相关的。

# 四、GIC callback函数分析

1、irq domain相关callback函数分析

irq domain相关callback函数包括：

（1）gic_irq_domain_map函数：创建IRQ number和GIC hw interrupt ID之间映射关系的时候，需要调用该回调函数。具体代码如下：

> static int gic_irq_domain_map(struct irq_domain \*d, unsigned int irq, irq_hw_number_t hw)\
> {\
> if (hw \< 32) {－－－－－－－－－－－－－－－－－－SGI或者PPI\
> irq_set_percpu_devid(irq);－－－－－－－－－－－－－－－－－－－－－－－－－－（a）\
> irq_set_chip_and_handler(irq, &gic_chip, handle_percpu_devid_irq);－－－－－－－（b）\
> set_irq_flags(irq, IRQF_VALID | IRQF_NOAUTOEN);－－－－－－－－－－－－－－（c）\
> } else {\
> irq_set_chip_and_handler(irq, &gic_chip, handle_fasteoi_irq);－－－－－－－－－－（d）\
> set_irq_flags(irq, IRQF_VALID | IRQF_PROBE);
>
> gic_routable_irq_domain_ops->map(d, irq, hw);－－－－－－－－－－－－－－－－（e）\
> }\
> irq_set_chip_data(irq, d->host_data);－－－－－设定irq chip的私有数据\
> return 0;\
> }

（a）SGI或者PPI和SPI最大的不同是per cpu的，SPI是所有CPU共享的，因此需要分配per cpu的内存，设定一些per cpu的flag。

（b）设定该中断描述符的irq chip和high level的handler

（c）设定irq flag是有效的（因为已经设定好了chip和handler了），并且request后不是auto enable的。

（d）对于SPI，设定的high level irq event handler是handle_fasteoi_irq。对于SPI，是可以probe，并且request后是auto enable的。

（e）有些SOC会在各种外设中断和GIC之间增加cross bar（例如TI的OMAP芯片），这里是为那些ARM SOC准备的

（2）gic_irq_domain_unmap是gic_irq_domain_map的逆过程也就是解除IRQ number和GIC hw interrupt ID之间映射关系的时候，需要调用该回调函数。

（3）gic_irq_domain_xlate函数：除了标准的属性之外，各个具体的interrupt controller可以定义自己的device binding。这些device bindings都需在irq chip driver这个层面进行解析。要给定某个外设的device tree node 和interrupt specifier，该函数可以解码出该设备使用的hw interrupt ID和linux irq type value 。具体的代码如下：

> static int gic_irq_domain_xlate(struct irq_domain \*d,\
> struct device_node \*controller,\
> const u32 \*intspec, unsigned int intsize,－－－－－－－－输入参数\
> unsigned long \*out_hwirq, unsigned int \*out_type)－－－－输出参数\
> {\
> unsigned long ret = 0; \
> \*out_hwirq = intspec\[1\] + 16; －－－－－－－－－－－－－－－－－－－－－（a）
>
> \*out_type = intspec\[2\] & IRQ_TYPE_SENSE_MASK; －－－－－－－－－－－（b）
>
> return ret;\
> }

（a）根据gic binding文档的描述，其interrupt specifier包括3个cell，分别是interrupt type（0 表示SPI，1表示PPI），interrupt number（对于PPI，范围是\[0-15\]，对于SPI，范围是\[0-987\]），interrupt flag（触发方式）。GIC interrupt specifier中的interrupt number需要加上16（也就是加上SGI的那些ID号），才能转换成GIC的HW interrupt ID。

（b）取出bits\[3:0\]的信息，这些bits保存了触发方式的信息

2、电源管理的callback函数

TODO

3、irq chip回调函数分析

（1）gic_mask_irq函数

这个函数用来mask一个interrupt source。代码如下：

> static void gic_mask_irq(struct irq_data \*d)\
> {\
> u32 mask = 1 \<\< (gic_irq(d) % 32);
>
> raw_spin_lock(&irq_controller_lock);\
> writel_relaxed(mask, gic_dist_base(d) + GIC_DIST_ENABLE_CLEAR + (gic_irq(d) / 32) * 4);\
> if (gic_arch_extn.irq_mask)\
> gic_arch_extn.irq_mask(d);\
> raw_spin_unlock(&irq_controller_lock);\
> }

GIC有若干个叫做Interrupt Clear-Enable Registers（具体数目是和GIC支持的hw interrupt数目相关，我们前面说过的，GIC是一个高度可配置的interrupt controller）。这些Interrupt Clear-Enable Registers寄存器的每个bit可以控制一个interrupt source是否forward到CPU interface，写入1表示Distributor不再forward该interrupt，因此CPU也就感知不到该中断，也就是mask了该中断。特别需要注意的是：写入0无效，而不是unmask的操作。

由于不同的SOC厂商在集成GIC的时候可能会修改，也就是说，也有可能mask的代码要微调，这是通过gic_arch_extn这个全局变量实现的。在gic-irq.c中这个变量的全部成员都设定为NULL，各个厂商在初始中断控制器的时候可以设定其特定的操作函数。

（2）gic_unmask_irq函数

这个函数用来unmask一个interrupt source。代码如下：

> static void gic_unmask_irq(struct irq_data \*d)\
> {\
> u32 mask = 1 \<\< (gic_irq(d) % 32);
>
> raw_spin_lock(&irq_controller_lock);\
> if (gic_arch_extn.irq_unmask)\
> gic_arch_extn.irq_unmask(d);\
> writel_relaxed(mask, gic_dist_base(d) + GIC_DIST_ENABLE_SET + (gic_irq(d) / 32) * 4);\
> raw_spin_unlock(&irq_controller_lock);\
> }

GIC有若干个叫做Interrupt Set-Enable Registers的寄存器。这些寄存器的每个bit可以控制一个interrupt source。当写入1的时候，表示Distributor会forward该interrupt到CPU interface，也就是意味这unmask了该中断。特别需要注意的是：写入0无效，而不是mask的操作。

（3）gic_eoi_irq函数

当processor处理中断的时候就会调用这个函数用来结束中断处理。代码如下：

> static void gic_eoi_irq(struct irq_data \*d)\
> {\
> if (gic_arch_extn.irq_eoi) {\
> raw_spin_lock(&irq_controller_lock);\
> gic_arch_extn.irq_eoi(d);\
> raw_spin_unlock(&irq_controller_lock);\
> }
>
> writel_relaxed(gic_irq(d), gic_cpu_base(d) + GIC_CPU_EOI);\
> }

对于GIC而言，其中断状态有四种：

|   |   |
|---|---|
|中断状态|描述|
|Inactive|中断未触发状态，该中断即没有Pending也没有Active|
|Pending|由于外设硬件产生了中断事件（或者软件触发）该中断事件已经通过硬件信号通知到GIC，等待GIC分配的那个CPU进行处理|
|Active|CPU已经应答（acknowledge）了该interrupt请求，并且正在处理中|
|Active and Pending|当一个中断源处于Active状态的时候，同一中断源又触发了中断，进入pending状态|

processor ack了一个中断后，该中断会被设定为active。当处理完成后，仍然要通知GIC，中断已经处理完毕了。这时候，如果没有pending的中断，GIC就会将该interrupt设定为inactive状态。操作GIC中的End of Interrupt Register可以完成end of interrupt事件通知。

（4）gic_set_type函数

这个函数用来设定一个interrupt source的type，例如是level sensitive还是edge triggered。代码如下：

> static int gic_set_type(struct irq_data \*d, unsigned int type)\
> {\
> void \_\_iomem \*base = gic_dist_base(d);\
> unsigned int gicirq = gic_irq(d);\
> u32 enablemask = 1 \<\< (gicirq % 32);\
> u32 enableoff = (gicirq / 32) * 4;\
> u32 confmask = 0x2 \<\< ((gicirq % 16) * 2);\
> u32 confoff = (gicirq / 16) * 4;\
> bool enabled = false;\
> u32 val;
>
> /\* Interrupt configuration for SGIs can't be changed \*/\
> if (gicirq \< 16)\
> return -EINVAL;
>
> if (type != IRQ_TYPE_LEVEL_HIGH && type != IRQ_TYPE_EDGE_RISING)\
> return -EINVAL;
>
> raw_spin_lock(&irq_controller_lock);
>
> if (gic_arch_extn.irq_set_type)\
> gic_arch_extn.irq_set_type(d, type);
>
> val = readl_relaxed(base + GIC_DIST_CONFIG + confoff);\
> if (type == IRQ_TYPE_LEVEL_HIGH)\
> val &= ~confmask;\
> else if (type == IRQ_TYPE_EDGE_RISING)\
> val |= confmask;
>
> /\*\
> \* As recommended by the spec, disable the interrupt before changing\
> \* the configuration\
> \*/\
> if (readl_relaxed(base + GIC_DIST_ENABLE_SET + enableoff) & enablemask) {\
> writel_relaxed(enablemask, base + GIC_DIST_ENABLE_CLEAR + enableoff);\
> enabled = true;\
> }
>
> writel_relaxed(val, base + GIC_DIST_CONFIG + confoff);
>
> if (enabled)\
> writel_relaxed(enablemask, base + GIC_DIST_ENABLE_SET + enableoff);
>
> raw_spin_unlock(&irq_controller_lock);
>
> return 0;\
> }

对于SGI类型的interrupt，是不能修改其type的，因为GIC中SGI固定就是edge-triggered。对于GIC，其type只支持高电平触发（IRQ_TYPE_LEVEL_HIGH）和上升沿触发（IRQ_TYPE_EDGE_RISING）的中断。另外需要注意的是，在更改其type的时候，先disable，然后修改type，然后再enable。

（5）gic_retrigger

这个接口用来resend一个IRQ到CPU。

> static int gic_retrigger(struct irq_data \*d)\
> {\
> if (gic_arch_extn.irq_retrigger)\
> return gic_arch_extn.irq_retrigger(d);
>
> /\* the genirq layer expects 0 if we can't retrigger in hardware \*/\
> return 0;\
> }

看起来这是功能不是通用GIC拥有的功能，各个厂家在集成GIC的时候，有可能进行功能扩展。

（6）gic_set_affinity

在多处理器的环境下，外部设备产生了一个中断就需要送到一个或者多个处理器去，这个设定是通过设定处理器的affinity进行的。具体代码如下：

> static int gic_set_affinity(struct irq_data \*d, const struct cpumask \*mask_val,    bool force)\
> {\
> void \_\_iomem \*reg = gic_dist_base(d) + GIC_DIST_TARGET + (gic_irq(d) & ~3);\
> unsigned int cpu, shift = (gic_irq(d) % 4) * 8;\
> u32 val, mask, bit;
>
> if (!force)\
> cpu = cpumask_any_and(mask_val, cpu_online_mask);－－－随机选取一个online的cpu\
> else\
> cpu = cpumask_first(mask_val); －－－－－－－－选取mask中的第一个cpu，不管是否online
>
> raw_spin_lock(&irq_controller_lock);\
> mask = 0xff \<\< shift;\
> bit = gic_cpu_map\[cpu\] \<\< shift;－－－－－－－将CPU的逻辑ID转换成要设定的cpu mask\
> val = readl_relaxed(reg) & ~mask;\
> writel_relaxed(val | bit, reg);\
> raw_spin_unlock(&irq_controller_lock);
>
> return IRQ_SET_MASK_OK;\
> }

GIC Distributor中有一个寄存器叫做Interrupt Processor Targets Registers，这个寄存器用来设定制定的中断送到哪个process去。由于GIC最大支持8个process，因此每个hw interrupt ID需要8个bit来表示送达的process。每一个Interrupt Processor Targets Registers由32个bit组成，因此每个Interrupt Processor Targets Registers可以表示4个HW interrupt ID的affinity，因此上面的代码中的shift就是计算该HW interrupt ID在寄存器中的偏移。

（7）gic_set_wake

这个接口用来设定唤醒CPU的interrupt source。对于GIC，代码如下：

> static int gic_set_wake(struct irq_data \*d, unsigned int on)\
> {\
> int ret = -ENXIO;
>
> if (gic_arch_extn.irq_set_wake)\
> ret = gic_arch_extn.irq_set_wake(d, on);
>
> return ret;\
> }

设定唤醒的interrupt和具体的厂商相关，这里不再赘述。

4、BSP（bootstrap processor）之外，其他CPU的callback函数

对于multi processor系统，不可能初始化代码在所有的processor上都执行一遍，实际上，系统的硬件会选取一个processor作为引导处理器，我们称之BSP。这个processor会首先执行，其他的CPU都是处于reset状态，等到BSP初始化完成之后，release所有的non-BSP，这时候，系统中的各种外设硬件条件和软件条件（例如per CPU变量）都准备好了，各个non-BSP执行自己CPU specific的初始化就OK了。

上面描述的都是BSP的初始化过程，具体包括：

> ……\
> gic_dist_init(gic);－－－－－－初始化GIC的Distributor\
> gic_cpu_init(gic);－－－－－－初始化BSP的CPU interface\
> gic_pm_init(gic);－－－－－－初始化GIC的Power management\
> ……

对于GIC的Distributor和Power management，这两部分是全局性的，BSP执行初始化一次就OK了。对于CPU interface，每个processor负责初始化自己的连接的那个CPU interface HW block。我们用下面这个图片来描述这个过程：

[![booting](http://www.wowotech.net/content/uploadfile/201409/cdec2bc26e9fb0e1a7b3d29389896b5420140909083806.gif "booting")](http://www.wowotech.net/content/uploadfile/201409/f9b5b82dcfa88ba9503bb8c835da63c420140909083805.gif)

假设CPUx被选定为BSP，那么第三章描述的初始化过程在该CPU上欢畅的执行。这时候，被初始化的GIC硬件包括：root GIC的Distributor、root GIC CPU Interface x（连接BSP的那个CPU interface）以及其他的级联的非root GIC（上图中绿色block，当然，我偷懒，没有画non-root GIC）。

BSP初始化完成之后，各个其他的CPU运行起来，会发送CPU_STARTING消息给关注该消息的模块。毫无疑问，GIC driver模块当然要关注这样的消息，在初始化过程中会注册callback函数如下：

> register_cpu_notifier(&gic_cpu_notifier);

GIC相关的回调函数定义如下：

> static struct notifier_block gic_cpu_notifier = {\
> .notifier_call = gic_secondary_init,\
> .priority = 100,\
> };
>
> static int gic_secondary_init(struct notifier_block \*nfb, unsigned long action,  void \*hcpu)\
> {\
> if (action == CPU_STARTING || action == CPU_STARTING_FROZEN)\
> gic_cpu_init(&gic_data\[0\]);－－－－－－－－－初始化那些非BSP的CPU interface\
> return NOTIFY_OK;\
> }

因此，当non-BSP booting up的时候，发送CPU_STARTING消息，调用GIC的callback函数，对上图中的紫色的CPU Interface HW block进行初始化，这样，就完成了全部GIC硬件的初始化过程。

Change log：\
11月3号，修改包括：\
1、使用GIC-V2这样更通用的描述，而不是仅仅GIC-400

_原创文章，转发请注明出处。蜗窝科技，[http://www.wowotech.net/linux_kenrel/gic_driver.html](http://www.wowotech.net/linux_kenrel/gic_driver.html)_

标签: [GIC](http://www.wowotech.net/tag/GIC) [代码分析](http://www.wowotech.net/tag/%E4%BB%A3%E7%A0%81%E5%88%86%E6%9E%90)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Linux电源管理(7)\_Wakeup events framework](http://www.wowotech.net/pm_subsystem/wakeup_events_framework.html) | [linux kernel的中断子系统之（四）：High level irq event handler](http://www.wowotech.net/irq_subsystem/High_level_irq_event_handler.html)»

**评论：**

**qixi**\
2020-04-09 15:19

谢谢作者的分享，请问文中描述GIC和CPU接口上的交互过程的时序图是参考的哪个文档？多谢

[回复](http://www.wowotech.net/irq_subsystem/gic_driver.html#comment-7943)

**无非**\
2019-11-09 16:02

在GIC硬件初始化这一章节里：  设定CPU interface的control register。enable了group 0的中断，disable了group 1的中断，group 0的interrupt source触发IRQ中断（而不是FIQ中断）。\
请问：既然没有打开FIQ，哪系统是怎样触发FIQ类型中断的，还是不能触发FIQ类型的中断。

[回复](http://www.wowotech.net/irq_subsystem/gic_driver.html#comment-7742)

**yz**\
2022-03-06 17:14

@无非：group0和group1其中一个可以产生fiq，如果想要触发fiq，就需要把中断的group更改一下，并且开启gic中fiq的使能，就可以产生fiq中断

[回复](http://www.wowotech.net/irq_subsystem/gic_driver.html#comment-8568)

**soc**\
2019-06-18 13:56

博主："（3）gic_irq_domain_xlate函数" 下方的注释 "GIC interrupt specifier中的interrupt number需要加上16（也就是加上SGI的那些ID号），才能转换成GIC的HW interrupt ID。" 上面的步骤只是跳过了SGI,转换了PPI，如果是SPI的话还要+16，我查了下源码发现是有的：\
/\* For SPIs, we need to add 16 more to get the GIC irq ID number \*/\
if (!intspec\[0\])\
\*out_hwirq += 16;

[回复](http://www.wowotech.net/irq_subsystem/gic_driver.html#comment-7476)

**BeDook**\
2019-06-04 17:16

博主大侠，请教个关于GIC的入门问题，在ARM V8系统中，如何实现一个SPI中断同时给多个Core响应么？貌似手册上说对于SPI类型的中断，不论是Targeted distribution model模式还是1 of N model模式，其一次只能指定由一个核响应改变该中断的状态？而其他核上中断状态仍为Pending，菜鸟认为这样的后果是各个核一次轮流响应SPI z中断？

[回复](http://www.wowotech.net/irq_subsystem/gic_driver.html#comment-7455)

**威点零**\
2019-05-22 16:18

研究了一下：\
为gic中断分配cpu, GIC_DIST_TARGET寄存器为gic中断提供8-bits的目标CPU，一个GIC_DIST_TARGET寄存器可服务于4个gic中断。为什么一个CPU target只需要8bit呢？因为cortex_a9_gic只支持8个CPU。若CPU targets=0xff，表示任何一个cpu都可处理该中断，如果CPU targets=0x01，则表示只有CPU0可处理该中断。GIC_DIST_TARGET结构如下：\
b31-b24        b23-b16        b15-b8         b7-b0\
CPU targets    CPU targets    CPU targets    CPU targets\
offset3        offset2        offset1        offset0

[回复](http://www.wowotech.net/irq_subsystem/gic_driver.html#comment-7437)

**威点零**\
2019-05-22 15:54

这里操作的寄存器是Interrupt Processor Targets Registers，该寄存器组中，每个GIC上的interrupt ID都有8个bit来控制送达的target CPU。我们来看看下面的图片：\
GIC_DIST_TARGETn（Interrupt Processor Targets Registers）位于Distributor HW block中，能控制送达的CPU interface，并不是具体的CPU，如果具体的实现中CPU interface和CPU是严格按照上图中那样一一对应，那么GIC_DIST_TARGET送达了CPU Interface n，也就是送达了CPU n。当然现实未必如你所愿，那么怎样来获取这个CPU的mask呢？我们知道SGI和PPI不需要使用GIC_DIST_TARGET控制target CPU。SGI送达目标CPU有自己特有的寄存器来控制（Software Generated Interrupt Register），对于PPI，其是CPU私有的，因此不需要控制target CPU。GIC_DIST_TARGET0～GIC_DIST_TARGET7是控制0～31这32个interrupt ID（SGI和PPI）的target CPU的，但是实际上SGI和PPI是不需要控制target CPU的，因此，这些寄存器是read only的，读取这些寄存器返回的就是cpu mask值。假设CPU0接在CPU interface 4上，那么运行在CPU 0上的程序在读GIC_DIST_TARGET0～GIC_DIST_TARGET7的时候，返回的就是0b00010000。

文章写得不错，但是上面这段，我是看了又看，有很多地方没明白。\
1.上面提到每个interrupt Num由8bit控制送达的target cpu，后面又提到Cpu Interface和CPUx不是对应接的。不太明白，distributor到底是控制到cpu interface还是到Cpu的。\
2.GIC_DIST_TARGET0～GIC_DIST_TARGET7是控制0～31这32个interrupt ID（SGI和PPI）的target CPU的。按第8bit来算，位数应该是32\*32个bit，GIC_DIST_TARGET0～GIC_DIST_TARGET7，每个target是多少位呢？\
3.假设CPU0接在CPU interface 4上，那么运行在CPU 0上的程序在读GIC_DIST_TARGET0～GIC_DIST_TARGET7的时候，返回的就是0b00010000。这里是无论读GIC_DIST_TARGET0\\、GIC_DIST_TARGET1...GIC_DIST_TARGET7都是0b00010000吗？\
总得看下来，看得有点晕晕的。

[回复](http://www.wowotech.net/irq_subsystem/gic_driver.html#comment-7436)

**Mike_wu**\
2019-05-09 11:23

您好，请问个问题。假设CPU0接在CPU interface 4上，那么运行在CPU 0上的程序在读GIC_DIST_TARGET0～GIC_DIST_TARGET7的时候，返回的就是0b00010000。

[回复](http://www.wowotech.net/irq_subsystem/gic_driver.html#comment-7403)

**Mike_wu**\
2019-05-09 11:24

@Mike_wu：这个不是太懂

[回复](http://www.wowotech.net/irq_subsystem/gic_driver.html#comment-7404)

**zoro**\
2018-06-26 18:24

@linuxer:\
这里有个疑惑\
if (gic_nr == 0 && (irq_start & 31) > 0) {\
hwirq_base = 16;\
if (irq_start != -1)\
irq_start = (irq_start & ~31) + 16;\
} else {\
hwirq_base = 32;\
}\
irq_start = -1\
应该是进入上面的if语句吧，即PPI也mapping吧？

[回复](http://www.wowotech.net/irq_subsystem/gic_driver.html#comment-6819)

**[callme_friend](http://www.wowotech.net/)**\
2018-02-05 16:55

请教郭大侠：\
关于gic初始化distributor时：\
1、为何要使用0-31(SGI/PPI)中断id中的一个来初始化所有的SPI中断？

2、在获取0-31中某个中断时，实际上代码是获取0-4-8-...-28中断mask，代码如下所示，如果这些id对应的mask刚好全部无效，岂不很奇怪？\
static u8 gic_get_cpumask(struct gic_chip_data \*gic)\
{\
void \_\_iomem \*base = gic_data_dist_base(gic);\
u32 mask, i;\
for (i = mask = 0; i \< 32; i += 4) {   // 因为i += 4,因此获取的mask实际上是按4跳跃\
mask = readl_relaxed(base + GIC_DIST_TARGET + i);\
mask |= mask >> 16;\
mask |= mask >> 8;\
if (mask)\
break;\
}\
return mask;\
}

[回复](http://www.wowotech.net/irq_subsystem/gic_driver.html#comment-6531)

**[linuxer](http://www.wowotech.net/)**\
2018-02-07 09:25

@callme_friend：第二个问题：gic_get_cpumask是获取cpumask值，通过它可以设定该中断送达的CPU。之所以按照0-4-8-...-28进行操作，那是因为0、1、2、3是一样的。原文中有一段话清楚的说明了这个情况：\
－－－－－－－－－－－－－－－－\
GIC_DIST_TARGETn（Interrupt Processor Targets Registers）位于Distributor HW block中，能控制送达的CPU interface，并不是具体的CPU，如果具体的实现中CPU interface和CPU是严格按照上图中那样一一对应，那么GIC_DIST_TARGET送达了CPU Interface n，也就是送达了CPU n。当然现实未必如你所愿，那么怎样来获取这个CPU的mask呢？我们知道SGI和PPI不需要使用GIC_DIST_TARGET控制target CPU。SGI送达目标CPU有自己特有的寄存器来控制（Software Generated Interrupt Register），对于PPI，其是CPU私有的，因此不需要控制target CPU。GIC_DIST_TARGET0～GIC_DIST_TARGET7是控制0～31这32个interrupt ID（SGI和PPI）的target CPU的，但是实际上SGI和PPI是不需要控制target CPU的，因此，这些寄存器是read only的，读取这些寄存器返回的就是cpu mask值。假设CPU0接在CPU interface 4上，那么运行在CPU 0上的程序在读GIC_DIST_TARGET0～GIC_DIST_TARGET7的时候，返回的就是0b00010000。\
－－－－－－－－－－－－－－－－－－－－－－－－－－

第一个问题我没有特别搞清楚你的意思，能不能给出对应的代码？

[回复](http://www.wowotech.net/irq_subsystem/gic_driver.html#comment-6536)

**[callme_friend](http://www.wowotech.net/)**\
2018-02-07 10:04

@linuxer：这两个问题都是关于中断的mask初始化。\
第一个问题，我的意思是，为何通过0-31中断的cpu mask来初始化其他的SPI中断的cpu mask？

第二个问题，为何0、1、2、3号中断的cpu mask一样？

[回复](http://www.wowotech.net/irq_subsystem/gic_driver.html#comment-6538)

**[linuxer](http://www.wowotech.net/)**\
2018-02-07 10:24

@callme_friend：第二个问题：\
0-31的中断需要cpu mask吗？不需要啊，SGI是软件设定target，PPI是一对一的，因此，0-31的中断号不需要设定target cpu，也就是说它们的GIC_DIST_TARGET是没有意义的。不过ARM仍然提供了这些寄存器，是readonly的，通过它们可以读出8个CPU interface的mask值。8个CPU Interface如何对应到32个中断号上，那么就0、1、2、3对应cpu interface 0，4、5、6、7对应cpu interface 1 ........

第一个问题\
通过这样子可以把所有SPI中断送达系统中所有的CPU。

[回复](http://www.wowotech.net/irq_subsystem/gic_driver.html#comment-6539)

**甲乙丙丁**\
2016-12-16 23:53

既然GIC只支持高电平和上升沿种中断模式，那么外设的低电平触发中断和下降沿触发中断是怎么处理，是由什么硬件将其翻转为GIC支持的高电平和上升沿信号吗？

[回复](http://www.wowotech.net/irq_subsystem/gic_driver.html#comment-5036)

**[wowo](http://www.wowotech.net/)**\
2017-01-04 08:39

@甲乙丙丁：文章中只是说“SGI只支持高电平触发”，其它的还是没有问题的。

[回复](http://www.wowotech.net/irq_subsystem/gic_driver.html#comment-5084)

1 [2](http://www.wowotech.net/irq_subsystem/gic_driver.html/comment-page-2#comments) [3](http://www.wowotech.net/irq_subsystem/gic_driver.html/comment-page-3#comments)

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

  - [ARMv8之Atomicity](http://www.wowotech.net/armv8a_arch/atomicity.html)
  - [建立讨论区的原因](http://www.wowotech.net/73.html)
  - [Linux power supply class(1)\_软件架构及API汇整](http://www.wowotech.net/pm_subsystem/psy_class_overview.html)
  - [vim使用技巧摘录](http://www.wowotech.net/linux_application/vim_skill.html)
  - [Linux调度器：进程优先级](http://www.wowotech.net/process_management/process-priority.html)

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

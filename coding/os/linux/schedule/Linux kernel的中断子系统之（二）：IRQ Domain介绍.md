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

作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2014-8-19 18:46 分类：[中断子系统](http://www.wowotech.net/sort/irq_subsystem)

一、概述

在linux kernel中，我们使用下面两个ID来标识一个来自外设的中断：

1、IRQ number。CPU需要为每一个外设中断编号，我们称之IRQ Number。这个IRQ number是一个虚拟的interrupt ID，和硬件无关，仅仅是被CPU用来标识一个外设中断。

2、HW interrupt ID。对于interrupt controller而言，它收集了多个外设的interrupt request line并向上传递，因此，interrupt controller需要对外设中断进行编码。Interrupt controller用HW interrupt ID来标识外设的中断。在interrupt controller级联的情况下，仅仅用HW interrupt ID已经不能唯一标识一个外设中断，还需要知道该HW interrupt ID所属的interrupt controller（HW interrupt ID在不同的Interrupt controller上是会重复编码的）。

这样，CPU和interrupt controller在标识中断上就有了一些不同的概念，但是，对于驱动工程师而言，我们和CPU视角是一样的，我们只希望得到一个IRQ number，而不关系具体是那个interrupt controller上的那个HW interrupt ID。这样一个好处是在中断相关的硬件发生变化的时候，驱动软件不需要修改。因此，linux kernel中的中断子系统需要提供一个将HW interrupt ID映射到IRQ number上来的机制，这就是本文主要的内容。

二、历史

关于HW interrupt ID映射到IRQ number上 这事，在过去系统只有一个interrupt controller的时候还是很简单的，中断控制器上实际的HW interrupt line的编号可以直接变成IRQ number。例如我们大家都熟悉的SOC内嵌的interrupt controller，这种controller多半有中断状态寄存器，这个寄存器可能有64个bit（也可能更多），每个bit就是一个IRQ number，可以直接进行映射。这时候，GPIO的中断在中断控制器的状态寄存器中只有一个bit，因此所有的GPIO中断只有一个IRQ number，在该通用GPIO中断的irq handler中进行deduplex，将各个具体的GPIO中断映射到其相应的IRQ number上。如果你是一个足够老的工程师，应该是经历过这个阶段的。

随着linux kernel的发展，将interrupt controller抽象成irqchip这个概念越来越流行，甚至GPIO controller也可以被看出一个interrupt controller chip，这样，系统中至少有两个中断控制器了，一个传统意义的中断控制器，一个是GPIO controller type的中断控制器。随着系统复杂度加大，外设中断数据增加，实际上系统可以需要多个中断控制器进行级联，面对这样的趋势，linux kernel工程师如何应对？答案就是irq domain这个概念。

我们听说过很多的domain，power domain，clock domain等等，所谓domain，就是领域，范围的意思，也就是说，任何的定义出了这个范围就没有意义了。系统中所有的interrupt controller会形成树状结构，对于每个interrupt controller都可以连接若干个外设的中断请求（我们称之interrupt source），interrupt controller会对连接其上的interrupt source（根据其在Interrupt controller中物理特性）进行编号（也就是HW interrupt ID了）。但这个编号仅仅限制在本interrupt controller范围内。

三、接口

1、向系统注册irq domain

具体如何进行映射是interrupt controller自己的事情，不过，有软件架构思想的工程师更愿意对形形色色的interrupt controller进行抽象，对如何进行HW interrupt ID到IRQ number映射关系上进行进一步的抽象。因此，通用中断处理模块中有一个irq domain的子模块，该模块将这种映射关系分成了三类：

（1）线性映射。其实就是一个lookup table，HW interrupt ID作为index，通过查表可以获取对应的IRQ number。对于Linear map而言，interrupt controller对其HW interrupt ID进行编码的时候要满足一定的条件：hw ID不能过大，而且ID排列最好是紧密的。对于线性映射，其接口API如下：

> static inline struct irq_domain *irq_domain_add_linear(struct device_node *of_node,  
>                      unsigned int size,－－－－－－－－－该interrupt domain支持多少IRQ  
>                      const struct irq_domain_ops *ops,－－－callback函数  
>                      void *host_data)－－－－－driver私有数据  
> {  
>     return __irq_domain_add(of_node, size, size, 0, ops, host_data);  
> }

（2）Radix Tree map。建立一个Radix Tree来维护HW interrupt ID到IRQ number映射关系。HW interrupt ID作为lookup key，在Radix Tree检索到IRQ number。如果的确不能满足线性映射的条件，可以考虑Radix Tree map。实际上，内核中使用Radix Tree map的只有powerPC和MIPS的硬件平台。对于Radix Tree map，其接口API如下：

> static inline struct irq_domain *irq_domain_add_tree(struct device_node *of_node,  
>                      const struct irq_domain_ops *ops,  
>                      void *host_data)  
> {  
>     return __irq_domain_add(of_node, 0, ~0, 0, ops, host_data);  
> }

（3）no map。有些中断控制器很强，可以通过寄存器配置HW interrupt ID而不是由物理连接决定的。例如PowerPC 系统使用的MPIC (Multi-Processor Interrupt Controller)。在这种情况下，不需要进行映射，我们直接把IRQ number写入HW interrupt ID配置寄存器就OK了，这时候，生成的HW interrupt ID就是IRQ number，也就不需要进行mapping了。对于这种类型的映射，其接口API如下：

> static inline struct irq_domain *irq_domain_add_nomap(struct device_node *of_node,  
>                      unsigned int max_irq,  
>                      const struct irq_domain_ops *ops,  
>                      void *host_data)  
> {  
>     return __irq_domain_add(of_node, 0, max_irq, max_irq, ops, host_data);  
> }

这类接口的逻辑很简单，根据自己的映射类型，初始化struct irq_domain中的各个成员，调用__irq_domain_add将该irq domain挂入irq_domain_list的全局列表。

2、为irq domain创建映射

上节的内容主要是向系统注册一个irq domain，具体HW interrupt ID和IRQ number的映射关系都是空的，因此，具体各个irq domain如何管理映射所需要的database还是需要建立的。例如：对于线性映射的irq domain，我们需要建立线性映射的lookup table，对于Radix Tree map，我们要把那个反应IRQ number和HW interrupt ID的Radix tree建立起来。创建映射有四个接口函数：

（1）调用irq_create_mapping函数建立HW interrupt ID和IRQ number的映射关系。该接口函数以irq domain和HW interrupt ID为参数，返回IRQ number（这个IRQ number是动态分配的）。该函数的原型定义如下：

> extern unsigned int irq_create_mapping(struct irq_domain *host,  
>                        irq_hw_number_t hwirq);

驱动调用该函数的时候必须提供HW interrupt ID，也就是意味着driver知道自己使用的HW interrupt ID，而一般情况下，HW interrupt ID其实对具体的driver应该是不可见的，不过有些场景比较特殊，例如GPIO类型的中断，它的HW interrupt ID和GPIO有着特定的关系，driver知道自己使用那个GPIO，也就是知道使用哪一个HW interrupt ID了。

（2）irq_create_strict_mappings。这个接口函数用来为一组HW interrupt ID建立映射。具体函数的原型定义如下：

> extern int irq_create_strict_mappings(struct irq_domain *domain,  
>                       unsigned int irq_base,  
>                       irq_hw_number_t hwirq_base, int count);

（3）irq_create_of_mapping。看到函数名字中的of（open firmware），我想你也可以猜到了几分，这个接口当然是利用device tree进行映射关系的建立。具体函数的原型定义如下：

> extern unsigned int irq_create_of_mapping(struct of_phandle_args *irq_data);

通常，一个普通设备的device tree node已经描述了足够的中断信息，在这种情况下，该设备的驱动在初始化的时候可以调用irq_of_parse_and_map这个接口函数进行该device node中和中断相关的内容（interrupts和interrupt-parent属性）进行分析，并建立映射关系，具体代码如下：

> unsigned int irq_of_parse_and_map(struct device_node *dev, int index)  
> {  
>     struct of_phandle_args oirq;
> 
>     if (of_irq_parse_one(dev, index, &oirq))－－－－分析device node中的interrupt相关属性  
>         return 0;
> 
>     return irq_create_of_mapping(&oirq);－－－－－创建映射，并返回对应的IRQ number  
> }

对于一个使用Device tree的普通驱动程序（我们推荐这样做），基本上初始化需要调用irq_of_parse_and_map获取IRQ number，然后调用request_threaded_irq申请中断handler。

（4）irq_create_direct_mapping。这是给no map那种类型的interrupt controller使用的，这里不再赘述。

四、数据结构描述

1、irq domain的callback接口

struct irq_domain_ops抽象了一个irq domain的callback函数，定义如下：

> struct irq_domain_ops {  
>     int (*match)(struct irq_domain *d, struct device_node *node);  
>     int (*map)(struct irq_domain *d, unsigned int virq, irq_hw_number_t hw);  
>     void (*unmap)(struct irq_domain *d, unsigned int virq);  
>     int (*xlate)(struct irq_domain *d, struct device_node *node,  
>              const u32 *intspec, unsigned int intsize,  
>              unsigned long *out_hwirq, unsigned int *out_type);  
> };

我们先看xlate函数，语义是翻译（translate）的意思，那么到底翻译什么呢？在DTS文件中，各个使用中断的device node会通过一些属性（例如interrupts和interrupt-parent属性）来提供中断信息给kernel以便kernel可以正确的进行driver的初始化动作。这里，interrupts属性所表示的interrupt specifier只能由具体的interrupt controller（也就是irq domain）来解析。而xlate函数就是将指定的设备（node参数）上若干个（intsize参数）中断属性（intspec参数）翻译成HW interrupt ID（out_hwirq参数）和trigger类型（out_type）。

match是判断一个指定的interrupt controller（node参数）是否和一个irq domain匹配（d参数），如果匹配的话，返回1。实际上，内核中很少定义这个callback函数，实际上struct irq_domain中有一个of_node指向了对应的interrupt controller的device node，因此，如果不提供该函数，那么default的匹配函数其实就是判断irq domain的of_node成员是否等于传入的node参数。

map和unmap是操作相反的函数，我们描述其中之一就OK了。调用map函数的时机是在创建（或者更新）HW interrupt ID（hw参数）和IRQ number（virq参数）关系的时候。其实，从发生一个中断到调用该中断的handler仅仅调用一个request_threaded_irq是不够的，还需要针对该irq number设定：

（1）设定该IRQ number对应的中断描述符（struct irq_desc）的irq chip

（2）设定该IRQ number对应的中断描述符的highlevel irq-events handler

（3）设定该IRQ number对应的中断描述符的 irq chip data

这些设定不适合由具体的硬件驱动来设定，因此在Interrupt controller，也就是irq domain的callback函数中设定。

2、irq domain

在内核中，irq domain的概念由struct irq_domain表示：

> struct irq_domain {  
>     struct list_head link;  
>     const char *name;  
>     const struct irq_domain_ops *ops; －－－－callback函数  
>     void *host_data;
> 
>     /* Optional data */  
>     struct device_node *of_node; －－－－该interrupt domain对应的interrupt controller的device node  
>     struct irq_domain_chip_generic *gc; －－－generic irq chip的概念，本文暂不描述
> 
>     /* reverse map data. The linear map gets appended to the irq_domain */  
>     irq_hw_number_t hwirq_max; －－－－该domain中最大的那个HW interrupt ID  
>     unsigned int revmap_direct_max_irq; －－－－  
>     unsigned int revmap_size; －－－线性映射的size，for Radix Tree map和no map，该值等于0  
>     struct radix_tree_root revmap_tree; －－－－Radix Tree map使用到的radix tree root node  
>     unsigned int linear_revmap[]; －－－－－线性映射使用的lookup table  
> };

linux内核中，所有的irq domain被挂入一个全局链表，链表头定义如下：

> static LIST_HEAD(irq_domain_list);

struct irq_domain中的link成员就是挂入这个队列的节点。通过irq_domain_list这个指针，可以获取整个系统中HW interrupt ID和IRQ number的mapping DB。host_data定义了底层interrupt controller使用的私有数据，和具体的interrupt controller相关（对于GIC，该指针指向一个struct gic_chip_data数据结构）。

对于线性映射：

（1）linear_revmap保存了一个线性的lookup table，index是HW interrupt ID，table中保存了IRQ number值

（2）revmap_size等于线性的lookup table的size。

（3）hwirq_max保存了最大的HW interrupt ID

（4）revmap_direct_max_irq没有用，设定为0。revmap_tree没有用。

对于Radix Tree map：

（1）linear_revmap没有用，revmap_size等于0。

（2）hwirq_max没有用，设定为一个最大值。

（3）revmap_direct_max_irq没有用，设定为0。

（4）revmap_tree指向Radix tree的root node。

五、中断相关的Device Tree知识回顾

想要进行映射，首先要了解interrupt controller的拓扑结构。系统中的interrupt controller的拓扑结构以及其interrupt request line的分配情况（分配给哪一个具体的外设）都在Device Tree Source文件中通过下面的属性给出了描述。这些内容在Device Tree的三份文档中给出了一些描述，这里简单总结一下：

对于那些产生中断的外设，我们需要定义interrupt-parent和interrupts属性：

（1）interrupt-parent。表明该外设的interrupt request line物理的连接到了哪一个中断控制器上

（2）interrupts。这个属性描述了具体该外设产生的interrupt的细节信息（也就是传说中的interrupt specifier）。例如：HW interrupt ID（由该外设的device node中的interrupt-parent指向的interrupt controller解析）、interrupt触发类型等。

对于Interrupt controller，我们需要定义interrupt-controller和#interrupt-cells的属性：

（1）interrupt-controller。表明该device node就是一个中断控制器

（2）#interrupt-cells。该中断控制器用多少个cell（一个cell就是一个32-bit的单元）描述一个外设的interrupt request line。？具体每个cell表示什么样的含义由interrupt controller自己定义。

（3）interrupts和interrupt-parent。对于那些不是root 的interrupt controller，其本身也是作为一个产生中断的外设连接到其他的interrupt controller上，因此也需要定义interrupts和interrupt-parent的属性。

六、Mapping DB的建立

1、概述

系统中HW interrupt ID和IRQ number的mapping DB是在整个系统初始化的过程中建立起来的，过程如下：

（1）DTS文件描述了系统中的interrupt controller以及外设IRQ的拓扑结构，在linux kernel启动的时候，由bootloader传递给kernel（实际传递的是DTB）。

（2）在Device Tree初始化的时候，形成了系统内所有的device node的树状结构，当然其中包括所有和中断拓扑相关的数据结构（所有的interrupt controller的node和使用中断的外设node）

（3）在machine driver初始化的时候会调用of_irq_init函数，在该函数中会扫描所有interrupt controller的节点，并调用适合的interrupt controller driver进行初始化。毫无疑问，初始化需要注意顺序，首先初始化root，然后first level，second level，最好是leaf node。在初始化的过程中，一般会调用上节中的接口函数向系统增加irq domain。有些interrupt controller会在其driver初始化的过程中创建映射

（4）在各个driver初始化的过程中，创建映射

2、 interrupt controller初始化的过程中，注册irq domain

我们以GIC的代码为例。具体代码在gic_of_init->gic_init_bases中，如下：

> void __init gic_init_bases(unsigned int gic_nr, int irq_start,  
>                void __iomem *dist_base, void __iomem *cpu_base,  
>                u32 percpu_offset, struct device_node *node)  
> {  
>     irq_hw_number_t hwirq_base;  
>     struct gic_chip_data *gic;  
>     int gic_irqs, irq_base, i;
> 
> ……  
> 对于root GIC  
>         hwirq_base = 16;  
>         gic_irqs = 系统支持的所有的中断数目－16。之所以减去16主要是因为root GIC的0～15号HW interrupt 是for IPI的，因此要去掉。也正因为如此hwirq_base从16开始
> 
>   
>     irq_base = irq_alloc_descs(irq_start, 16, gic_irqs, numa_node_id());申请gic_irqs个IRQ资源，从16号开始搜索IRQ number。由于是root GIC，申请的IRQ基本上会从16号开始
> 
>   
>     gic->domain = irq_domain_add_legacy(node, gic_irqs, irq_base,  
>                     hwirq_base, &gic_irq_domain_ops, gic);－－－向系统注册irq domain并创建映射
> 
> ……  
> }

很遗憾，在GIC的代码中没有调用标准的注册irq domain的接口函数。要了解其背后的原因，我们需要回到过去。在旧的linux kernel中，ARM体系结构的代码不甚理想。在arch/arm目录充斥了很多board specific的代码，其中定义了很多具体设备相关的静态表格，这些表格规定了各个device使用的资源，当然，其中包括IRQ资源。在这种情况下，各个外设的IRQ是固定的（如果作为驱动程序员的你足够老的话，应该记得很长篇幅的针对IRQ number的宏定义），也就是说，HW interrupt ID和IRQ number的关系是固定的。一旦关系固定，我们就可以在interupt controller的代码中创建这些映射关系。具体代码如下：

> struct irq_domain *irq_domain_add_legacy(struct device_node *of_node,  
>                      unsigned int size,  
>                      unsigned int first_irq,  
>                      irq_hw_number_t first_hwirq,  
>                      const struct irq_domain_ops *ops,  
>                      void *host_data)  
> {  
>     struct irq_domain *domain;
> 
>     domain = __irq_domain_add(of_node, first_hwirq + size,－－－－注册irq domain  
>                   first_hwirq + size, 0, ops, host_data);  
>     if (!domain)  
>         return NULL;
> 
>     irq_domain_associate_many(domain, first_irq, first_hwirq, size); －－－创建映射
> 
>     return domain;  
> }

这时候，对于这个版本的GIC driver而言，初始化之后，HW interrupt ID和IRQ number的映射关系已经建立，保存在线性lookup table中，size等于GIC支持的中断数目，具体如下：

index 0～15对应的IRQ无效

16号IRQ  <------------------>16号HW interrupt ID

17号IRQ  <------------------>17号HW interrupt ID

……

如果想充分发挥Device Tree的威力，3.14版本中的GIC 代码需要修改。

3、在各个硬件外设的驱动初始化过程中，创建HW interrupt ID和IRQ number的映射关系

我们上面的描述过程中，已经提及：设备的驱动在初始化的时候可以调用irq_of_parse_and_map这个接口函数进行该device node中和中断相关的内容（interrupts和interrupt-parent属性）进行分析，并建立映射关系，具体代码如下：

> unsigned int irq_of_parse_and_map(struct device_node *dev, int index)  
> {  
>     struct of_phandle_args oirq;
> 
>     if (of_irq_parse_one(dev, index, &oirq))－－－－分析device node中的interrupt相关属性  
>         return 0;
> 
>     return irq_create_of_mapping(&oirq);－－－－－创建映射  
> }

我们再来看看irq_create_of_mapping函数如何创建映射：

> unsigned int irq_create_of_mapping(struct of_phandle_args *irq_data)  
> {  
>     struct irq_domain *domain;  
>     irq_hw_number_t hwirq;  
>     unsigned int type = IRQ_TYPE_NONE;  
>     unsigned int virq;
> 
>     domain = irq_data->np ? irq_find_host(irq_data->np) : irq_default_domain;－－A  
>     if (!domain) {  
>         return 0;  
>     }
> 
>   
>     if (domain->ops->xlate == NULL)－－－－－－－－－－－－－－B  
>         hwirq = irq_data->args[0];  
>     else {  
>         if (domain->ops->xlate(domain, irq_data->np, irq_data->args,－－－－C  
>                     irq_data->args_count, &hwirq, &type))  
>             return 0;  
>     }
> 
>     /* Create mapping */  
>     virq = irq_create_mapping(domain, hwirq);－－－－－－－－D  
>     if (!virq)  
>         return virq;
> 
>     /* Set type if specified and different than the current one */  
>     if (type != IRQ_TYPE_NONE &&  
>         type != irq_get_trigger_type(virq))  
>         irq_set_irq_type(virq, type);－－－－－－－－－E  
>     return virq;  
> }

A：这里的代码主要是找到irq domain。这是根据传递进来的参数irq_data的np成员来寻找的，具体定义如下：

> struct of_phandle_args {  
>     struct device_node *np;－－－指向了外设对应的interrupt controller的device node  
>     int args_count;－－－－－－－该外设定义的interrupt相关属性的个数  
>     uint32_t args[MAX_PHANDLE_ARGS];－－－－具体的interrupt相当属性的定义  
> };

B：如果没有定义xlate函数，那么取interrupts属性的第一个cell作为HW interrupt ID。

C：解铃还需系铃人，interrupts属性最好由interrupt controller（也就是irq domain）解释。如果xlate函数能够完成属性解析，那么将输出参数hwirq和type，分别表示HW interrupt ID和interupt type（触发方式等）。

D：解析完了，最终还是要调用irq_create_mapping函数来创建HW interrupt ID和IRQ number的映射关系。

E：如果有需要，调用irq_set_irq_type函数设定trigger type

irq_create_mapping函数建立HW interrupt ID和IRQ number的映射关系。该接口函数以irq domain和HW interrupt ID为参数，返回IRQ number。具体的代码如下：

> unsigned int irq_create_mapping(struct irq_domain *domain,  
>                 irq_hw_number_t hwirq)  
> {  
>     unsigned int hint;  
>     int virq;
> 
> 如果映射已经存在，那么不需要映射，直接返回  
>     virq = irq_find_mapping(domain, hwirq);  
>     if (virq) {  
>         return virq;  
>     }
> 
>   
>     hint = hwirq % nr_irqs;－－－－－－－分配一个IRQ 描述符以及对应的irq number  
>     if (hint == 0)  
>         hint++;  
>     virq = irq_alloc_desc_from(hint, of_node_to_nid(domain->of_node));  
>     if (virq <= 0)  
>         virq = irq_alloc_desc_from(1, of_node_to_nid(domain->of_node));  
>     if (virq <= 0) {  
>         pr_debug("-> virq allocation failed\n");  
>         return 0;  
>     }
> 
>     if (irq_domain_associate(domain, virq, hwirq)) {－－－建立mapping  
>         irq_free_desc(virq);  
>         return 0;  
>     }
> 
>     return virq;  
> }

对于分配中断描述符这段代码，后续的文章会详细描述。这里简单略过，反正，指向完这段代码，我们就可以或者一个IRQ number以及其对应的中断描述符了。程序注释中没有使用IRQ number而是使用了virtual interrupt number这个术语。virtual interrupt number还是重点理解“virtual”这个词，所谓virtual，其实就是说和具体的硬件连接没有关系了，仅仅是一个number而已。具体建立映射的函数是irq_domain_associate函数，代码如下：

> int irq_domain_associate(struct irq_domain *domain, unsigned int virq,  
>              irq_hw_number_t hwirq)  
> {  
>     struct irq_data *irq_data = irq_get_irq_data(virq);  
>     int ret;
> 
>     mutex_lock(&irq_domain_mutex);  
>     irq_data->hwirq = hwirq;  
>     irq_data->domain = domain;  
>     if (domain->ops->map) {  
>         ret = domain->ops->map(domain, virq, hwirq);－－－调用irq domain的map callback函数  
>     }
> 
>     if (hwirq < domain->revmap_size) {  
>         domain->linear_revmap[hwirq] = virq;－－－－填写线性映射lookup table的数据  
>     } else {  
>         mutex_lock(&revmap_trees_mutex);  
>         radix_tree_insert(&domain->revmap_tree, hwirq, irq_data);－－向radix tree插入一个node  
>         mutex_unlock(&revmap_trees_mutex);  
>     }  
>     mutex_unlock(&irq_domain_mutex);
> 
>     irq_clear_status_flags(virq, IRQ_NOREQUEST); －－－该IRQ已经可以申请了，因此clear相关flag
> 
>     return 0;  
> }

七、将HW interrupt ID转成IRQ number

创建了庞大的HW interrupt ID到IRQ number的mapping DB，最终还是要使用。具体的使用场景是在CPU相关的处理函数中，程序会读取硬件interrupt ID，并转成IRQ number，调用对应的irq event handler。在本章中，我们以一个级联的GIC系统为例，描述转换过程

1、GIC driver初始化

上面已经描述了root GIC的的初始化，我们再来看看second GIC的初始化。具体代码在gic_of_init->gic_init_bases中，如下：

> void __init gic_init_bases(unsigned int gic_nr, int irq_start,  
>                void __iomem *dist_base, void __iomem *cpu_base,  
>                u32 percpu_offset, struct device_node *node)  
> {  
>     irq_hw_number_t hwirq_base;  
>     struct gic_chip_data *gic;  
>     int gic_irqs, irq_base, i;
> 
> ……  
> 对于second GIC  
>         hwirq_base = 32;   
>         gic_irqs = 系统支持的所有的中断数目－32。之所以减去32主要是因为对于second GIC，其0～15号HW interrupt 是for IPI的，因此要去掉。而16～31号HW interrupt 是for PPI的，也要去掉。也正因为如此hwirq_base从32开始
> 
>   
>     irq_base = irq_alloc_descs(irq_start, 16, gic_irqs, numa_node_id());申请gic_irqs个IRQ资源，从16号开始搜索IRQ number。由于是second GIC，申请的IRQ基本上会从root GIC申请的最后一个IRQ号＋1开始
> 
>   
>     gic->domain = irq_domain_add_legacy(node, gic_irqs, irq_base,  
>                     hwirq_base, &gic_irq_domain_ops, gic);－－－向系统注册irq domain并创建映射
> 
> ……  
> }

second GIC初始化之后，该irq domain的HW interrupt ID和IRQ number的映射关系已经建立，保存在线性lookup table中，size等于GIC支持的中断数目，具体如下：

index 0～32对应的IRQ无效

root GIC申请的最后一个IRQ号＋1  <------------------>32号HW interrupt ID

root GIC申请的最后一个IRQ号＋2  <------------------>33号HW interrupt ID

……

OK，我们回到gic的初始化函数，对于second GIC，还有其他部分的初始化内容：

> int __init gic_of_init(struct device_node *node, struct device_node *parent)  
> {
> 
> ……
> 
>     if (parent) {  
>         irq = irq_of_parse_and_map(node, 0);－－解析second GIC的interrupts属性，并进行mapping，返回IRQ number  
>         gic_cascade_irq(gic_cnt, irq);－－－设置handler  
>     }  
> ……  
> }

上面的初始化函数去掉和级联无关的代码。对于root GIC，其传入的parent是NULL，因此不会执行级联部分的代码。对于second GIC，它是作为其parent（root GIC）的一个普通的irq source，因此，也需要注册该IRQ的handler。由此可见，非root的GIC的初始化分成了两个部分：一部分是作为一个interrupt controller，执行和root GIC一样的初始化代码。另外一方面，GIC又作为一个普通的interrupt generating device，需要象一个普通的设备驱动一样，注册其中断handler。

irq_of_parse_and_map函数相信大家已经熟悉了，这里不再描述。gic_cascade_irq函数如下：

> void __init gic_cascade_irq(unsigned int gic_nr, unsigned int irq)  
> {  
>     if (irq_set_handler_data(irq, &gic_data[gic_nr]) != 0)－－－设置handler data  
>         BUG();  
>     irq_set_chained_handler(irq, gic_handle_cascade_irq);－－－设置handler  
> }

2、具体如何在中断处理过程中，将HW interrupt ID转成IRQ number

在系统的启动过程中，经过了各个interrupt controller以及各个外设驱动的努力，整个interrupt系统的database（将HW interrupt ID转成IRQ number的数据库，这里的数据库不是指SQL lite或者oracle这样通用数据库软件）已经建立。一旦发生硬件中断，经过CPU architecture相关的中断代码之后，会调用irq handler，该函数的一般过程如下：

（1）首先找到root interrupt controller对应的irq domain。

（2）根据HW 寄存器信息和irq domain信息获取HW interrupt ID

（3）调用irq_find_mapping找到HW interrupt ID对应的irq number

（4）调用handle_IRQ（对于ARM平台）来处理该irq number

对于级联的情况，过程类似上面的描述，但是需要注意的是在步骤4中不是直接调用该IRQ的hander来处理该irq number因为，这个irq需要各个interrupt controller level上的解析。举一个简单的二阶级联情况：假设系统中有两个interrupt controller，A和B，A是root interrupt controller，B连接到A的13号HW interrupt ID上。在B interrupt controller初始化的时候，除了初始化它作为interrupt controller的那部分内容，还有初始化它作为root interrupt controller A上的一个普通外设这部分的内容。最重要的是调用irq_set_chained_handler设定handler。这样，在上面的步骤4的时候，就会调用13号HW interrupt ID对应的handler（也就是B的handler），在该handler中，会重复上面的（1）～（4）。

_原创文章，转发请注明出处。蜗窝科技。[http://www.wowotech.net/linux_kenrel/irq-domain.html](http://www.wowotech.net/linux_kenrel/irq-domain.html)_

标签: [irq_domain](http://www.wowotech.net/tag/irq_domain)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Linux电源管理(6)_Generic PM之Suspend功能](http://www.wowotech.net/pm_subsystem/suspend_and_resume.html) | [DMB DSB ISB以及SMP CPU 乱序](http://www.wowotech.net/77.html)»

**评论：**

**sleep**  
2023-08-02 11:34

我是初学者，首先感觉大佬很强，文章写的也很细，但少了点框架性的概述，感觉文章结构，内容组织和语句上可以改进下。文章开始我觉得可以先描述下interrupt-controller，irq-domain，irq-desc等几个对象的组织方式和对应关系或所属关系，稍稍具象化一点，不然一直说添加domain，添加映射其实是比较难以理解这个概念的，脑子里没有这几个对象的数据结构组织关系。  
  
还有比如说：  
“在second GIC初始化之后，该irq domain的HW interrupt ID和IRQ number的映射关系已经建立”  
紧接着：  
“3、在各个硬件外设的驱动初始化过程中，创建HW interrupt ID和IRQ number的映射关系”  
  
这个应该意思是HW interrupt ID和IRQ number在不同的阶段都可以进行映射(gic初始化或者device初始化）对吧。这里没有相关说明，单纯2各生硬句子让人有点混乱。  
  
大佬可能是对从中断子系统和设备子系统初始化过程都很熟悉，所以有些高估我这种菜鸟的知识储备程度吧。当然都是菜鸡发言哈，还是感谢和敬仰大佬实力和分享精神的。

[回复](http://www.wowotech.net/irq_subsystem/irq-domain.html#comment-8800)

**test**  
2021-11-29 15:29

问个问题,级联的irq如何设置affinity?  直接在父节irq号上使用affinity吗?

[回复](http://www.wowotech.net/irq_subsystem/irq-domain.html#comment-8386)

**WL__Linux**  
2018-05-24 17:34

博主的技术已经达到了敬仰的水平了：  
我问两个简单的问题呢  
（1）对于级联的中断控制器，您在上面的问题回答中说道  
“同样的，irq_find_mapping将hw interrupt ID转变成IRQ Number（当然是在second gic对应的irq domain上了，传递给irq_find_mapping的domain参数是和上次不一样的）”  
那么第二次的IRQ Number会和第一次的IRQ Num重合吗，还是在上一级的末尾进行继续申请。  
（2）最近做开发遇到这样的问题，我通过gpio_to_irq()这个函数得到gpio0-gpio5-31的中断号，gpio1-0与gpio1-0的差值为33，多个那个中断号使谁的呢  
（3）获取到gpio5_31的中断号为232  而  芯片手册中的网卡ID分别为（73,74）通过函数读到的中断号为238，写的中断号为239，那么他之前的中断号不是连续的吗

[回复](http://www.wowotech.net/irq_subsystem/irq-domain.html#comment-6762)

**melo**  
2019-02-26 11:56

@WL__Linux：这个我可以做个尝试解释下。  
驱动里最常见gpio_to_irq函数，追下去也是会跑到irq_create_mapping函数处，而内部这里wowo也讲了，会用irq_find_mapping来判断是否已经映射。  
那么这里为什么会这么处理呢，什么情况会提前在你驱动申请前就映射，什么情况是在你驱动申请时才需要映射？  
  
这个根据我的理解，是因为设备树使用习惯导致的。  
内核起来的时候,start_kernel->init_IRQ->irqchip_init->of_irq_init  
of_irq_init会扫描设备树，并根据里面配置的级联中断信息按顺序做好映射的工作。(这个wowo也提到啦)。  
但是我们还有的驱动，在使用gpio中断的时候，没有使用interrupts属性，而是在设备里简单的自定了一个中断gpio号，然后在驱动里取出gpio号，再手动的申请。  
  
这两者的区别，会导致再分配的虚拟中断号中间可能被填充了包括pmic等其他申请的虚拟中断号。所以这里就是使用了interrupts属性，就会提前分配。如果没有使用，那么就只能根据驱动加载顺序依次动态申请了

[回复](http://www.wowotech.net/irq_subsystem/irq-domain.html#comment-7205)

**[linuxer_fy](http://no%20home%20page%20yet/)**  
2017-10-13 14:57

"  
（1）首先找到root interrupt controller对应的irq domain。  
（2）根据HW 寄存器信息和irq domain信息获取HW interrupt ID  
"  
--------------  
（2)中的表述，有点绕人。  
HW interrupt ID不是在CPU收到中断信号时，就获取到了吗？  
(2)中CPU根据irq domain获取HW interrupt ID如何理解？

[回复](http://www.wowotech.net/irq_subsystem/irq-domain.html#comment-6107)

**[linuxer_fy](http://no%20home%20page%20yet/)**  
2017-11-28 14:30

@linuxer_fy：自己回答：  
发生中断，首先从root interrupt controller的寄存器得到HW interrupt ID  
经过irq_find_mapping得到IRQ number,再得到IRQ handler 进行处理  
如果该IRQ handler是对应一个串级的interrupt controller，则继续上述1-4  
每次都从当前interrupt controller的寄存器得到HW interrupt ID  
  
如有不对，请指正。

[回复](http://www.wowotech.net/irq_subsystem/irq-domain.html#comment-6249)

**kernel_123**  
2017-09-11 21:13

假设系统中就只有一个gic-400  
gic:interrupt-controller@1000000 {  
    compatible = "arm,cortex-a15-gic", "arm,cortex-a9-gic";  
    #interrupt-cells = <3>;  
    #address-cells = <0>;  
    interrupt-controller;  
    reg =   <0x0 0x10201000 0x1000>,  
        <0x0 0x10202000 0x1000>,  
        <0x0 0x10204000 0x2000>,  
        <0x0 0x10206000 0x2000>;  
    interrupts = <1 9 0xf04>;  
};  
比如上述的gic是这样的配置的，然后在pin-ctrl中引用gic作为parent  
pinctrl@10000000 {  
    gpe0: gpe0 {  
        gpio-controller;  
        #gpio-cells = <2>;  
  
        interrupt-controller;  
        #interrupt-cells = <2>;  
        interrupt-parent = <&gic>;  
        interrupts = <0 0 0>, <0 1 0>, <0 2 0>, <0 3 0>,  
                 <0 4 0>, <0 5 0>, <0 6 0>, <0 7 0>;  
};  
  
那这样的话，系统中的中断树就是如下样子：  
　    　　　　　　　    gic  
            |  
             32 33 ... 67  68 69  ...  
                |  
                      xxx_irq_chip  
                |            　  
              连接设备  
  
问题1: 上面描述的也是一种级联情况，gic初始化的时候只会调用一次gic_of_init函数，分配irq_domain, 建立映射。  
    　　那gic_of_init中的这种情况会不会调用到？  
    　　　if (parent) {  
        　irq = irq_of_parse_and_map(node, 0);  
        　gic_cascade_irq(gic_cnt, irq);  
    　　　}  
  
问题2: 在pin_ctrl驱动中也看到了，先分配每个bank的irq domain, 建立映射。然后接着去初始化pin ctrl所对应的gpio, 去建立hwirq到vir irq之间的映射。  
　　　但是每个bank设置了__irq_do_set_handler，　同时pin ctrl也设置了对应的__irq_do_set_handler，同时gic controller也有对应的irq_handler.  
      那这样一来，当接在pin_ctrl上的外设产生中断后，中断的处理流程是怎么样的？　是从gic的irq_handler -> pin_ctrl　-> bank -> 对应驱动的中断处理函数？  
  
  
　　

[回复](http://www.wowotech.net/irq_subsystem/irq-domain.html#comment-6025)

**hxj**  
2017-02-17 19:47

您好，我想问一下，在ARM架构下，我已经知道了HW interrupt ID 为26，在驱动程序中通过哪个函数得到irq number呢？  
还是说在3.14内核中不推荐用resource的方式来定义硬件中断号？

[回复](http://www.wowotech.net/irq_subsystem/irq-domain.html#comment-5228)

**[linuxer](http://www.wowotech.net/)**  
2017-02-21 09:17

@hxj：1、在DTS中描述该设备的HW Interrupt相关信息  
2、如果是platform device，那么直接调用platform_get_irq_byname或者platform_get_irq即可获得IRQ number  
3、如果不是platform device，那么需要调用更底层一些的接口函数。

[回复](http://www.wowotech.net/irq_subsystem/irq-domain.html#comment-5233)

**[cracker](http://www.wowotech.net/)**  
2016-09-02 10:04

Hi,linuxer:  
  
目前遇到如下WARNING和error，但不知道如何去分析根本，可以指点一下吗？  
  
log如下：  
  
[    2.213782] Driver 'mmcblk' needs updating - please use bus_type methods  
[    2.213879] sdhci: Secure Digital Host Controller Interface driver  
[    2.213894] sdhci: Copyright(c) Pierre Ossman  
[    2.213997] sdhci-pci 0000:00:01.0: SDHCI controller found [8086:1190] (rev 1)  
[    2.214050] ------------[ cut here ]------------  
[    2.214085] WARNING: CPU: 1 PID: 1 at /home/andrei/work/resin/resin-edison/build/tmp/work-shared/edison/kernel-source/kernel/irq/irqdomain.c:284 irq_domain_associate+0x1a3/0x200()  
[    2.214100] error: virq0 is already associated  
[    2.214111] Modules linked in:  
  
[    2.214142] CPU: 1 PID: 1 Comm: swapper/0 Not tainted 3.19.5-yocto-standard #1  
[    2.214156] Hardware name: Intel Corporation Merrifield/BODEGA BAY, BIOS 466 2014.06.23:19.20.05  
[    2.214169]  00000000 00000000 f6e0fc00 c19f08d7 f6e0fc40 f6e0fc30 c1253157 c1c24014  
[    2.214219]  f6e0fc5c 00000001 c1c23f40 0000011c c12a4563 c12a4563 f6c05e00 f675e480  
[    2.214267]  00000000 f6e0fc48 c12531c3 00000009 f6e0fc40 c1c24014 f6e0fc5c f6e0fc7c  
[    2.214314] Call Trace:  
[    2.214349]  [<c19f08d7>] dump_stack+0x4b/0x75  
[    2.214375]  [<c1253157>] warn_slowpath_common+0x87/0xc0  
[    2.214402]  [<c12a4563>] ? irq_domain_associate+0x1a3/0x200  
[    2.214426]  [<c12a4563>] ? irq_domain_associate+0x1a3/0x200  
[    2.214449]  [<c12531c3>] warn_slowpath_fmt+0x33/0x40  
[    2.214475]  [<c12a4563>] irq_domain_associate+0x1a3/0x200  
[    2.214502]  [<c12a45ff>] irq_domain_associate_many+0x3f/0xa0  
[    2.214528]  [<c19f559d>] ? mutex_unlock+0xd/0x10  
[    2.214551]  [<c19ecb88>] ? __irq_alloc_descs+0xe8/0x1b0  
[    2.214578]  [<c12a46a2>] irq_create_strict_mappings+0x42/0x50  
[    2.214605]  [<c12338fa>] mp_map_pin_to_irq+0x1ca/0x200  
[    2.214631]  [<c1233f40>] mp_map_gsi_to_irq+0x90/0xc0  
[    2.214658]  [<c184fb59>] intel_mid_pci_irq_enable+0x59/0x80  
[    2.214683]  [<c1850390>] pcibios_enable_device+0x30/0x40  
[    2.214708]  [<c1618f6f>] do_pci_enable_device+0x4f/0xd0  
[    2.214732]  [<c1619e4d>] pci_enable_device_flags+0x9d/0xe0  
[    2.214757]  [<c1619ee2>] pci_enable_device+0x12/0x20  
[    2.214782]  [<c17dc4b2>] sdhci_pci_probe+0xd2/0x950  
[    2.214810]  [<c161bb5f>] pci_device_probe+0x6f/0xd0  
[    2.214834]  [<c13ba2a5>] ? sysfs_create_link+0x25/0x50  
[    2.214860]  [<c16ada0f>] driver_probe_device+0x7f/0x3c0  
[    2.214885]  [<c161ba92>] ? pci_match_device+0xd2/0x100  
[    2.214910]  [<c16ade09>] __driver_attach+0x79/0x80  
[    2.214933]  [<c16add90>] ? __device_attach+0x40/0x40  
[    2.214955]  [<c16abdaf>] bus_for_each_dev+0x4f/0x80  
[    2.214979]  [<c16ad51e>] driver_attach+0x1e/0x20  
[    2.215001]  [<c16add90>] ? __device_attach+0x40/0x40  
[    2.215024]  [<c16ad157>] bus_add_driver+0x157/0x240  
[    2.215049]  [<c1e01c8f>] ? sdhci_drv_init+0x1b/0x1b  
[    2.215072]  [<c1e01c8f>] ? sdhci_drv_init+0x1b/0x1b  
[    2.215095]  [<c16ae56d>] driver_register+0x5d/0xf0  
[    2.215119]  [<c1351cce>] ? kfree+0x9e/0x1d0  
[    2.215144]  [<c15eb7d0>] ? kvasprintf+0x40/0x50  
[    2.215168]  [<c161ab9a>] __pci_register_driver+0x4a/0x50  
[    2.215192]  [<c1e01ca3>] sdhci_driver_init+0x14/0x16  
[    2.215216]  [<c120044a>] do_one_initcall+0xaa/0x1e0  
[    2.215239]  [<c1e01c8f>] ? sdhci_drv_init+0x1b/0x1b  
[    2.215263]  [<c1dcc4e3>] ? repair_env_string+0x12/0x54  
[    2.215288]  [<c126d0a9>] ? parse_args+0x299/0x510  
[    2.215317]  [<c1dccbeb>] kernel_init_freeable+0x147/0x1ef  
[    2.215345]  [<c1273a63>] ? preempt_count_sub+0xb3/0x110  
[    2.215368]  [<c127262d>] ? finish_task_switch+0x5d/0xd0  
[    2.215392]  [<c19eb630>] kernel_init+0x10/0xe0  
[    2.215415]  [<c19f7441>] ret_from_kernel_thread+0x21/0x30  
[    2.215436]  [<c19eb620>] ? rest_init+0x80/0x80  
[    2.215476] ---[ end trace 96410e869c63de19 ]---  
[    2.215566] flis_addr mapped addr: f8498900  
[    2.215694] sdhci-pci 0000:00:01.0: rte_addr mapped addr: f849c000  
[    2.317076] mmc0: BKOPS_EN bit is not set  
[    2.324368] mmc0: new HS200 MMC card at address 0001  
[    2.325192] mmcblk0: mmc0:0001 H4G1d� 3.64 GiB  
[    2.332236]  mmcblk0: p1 p2 p3 p4 p5 p6 p7 p8 p9 p10 p11  
[    2.337906] mmc0: SDHCI controller on PCI [0000:00:01.0] using ADMA

[回复](http://www.wowotech.net/irq_subsystem/irq-domain.html#comment-4496)

**[linuxer](http://www.wowotech.net/)**  
2016-09-02 15:24

@cracker：错误信息已经给出来了，就是virq等于0的那个中断描述符已经和某个irq domain的hw interrupt id绑定了，不能再次绑定。

[回复](http://www.wowotech.net/irq_subsystem/irq-domain.html#comment-4497)

**[cracker](http://www.wowotech.net/)**  
2016-09-13 11:05

@linuxer：Hi，linuxer，  
你说的很对。但我还是有疑问。  
virq = irq_find_mapping(domain, hwirq);  
    if (virq) {  
        return virq;  
    }  
unsigned int irq_find_mapping(struct irq_domain *domain,  
552                   irq_hw_number_t hwirq)  
{  
554     struct irq_data *data;  
555    
              。。。。。。。。。。。  
561    
562     if (hwirq < domain->revmap_direct_max_irq) {  
563         data = irq_domain_get_irq_data(domain, hwirq);  
564         if (data && data->hwirq == hwirq)    
565             return hwirq;  
566     }  
            。。。。。。。。。。。。。。。  
  
573     data = radix_tree_lookup(&domain->revmap_tree, hwirq);  
574     rcu_read_unlock();    
575     return data ? data->irq : 0;  
  
这里如果virq为0说明data为NULL还是data->irq = 0？如果data->irq = 0是不是配置的mapping映射是不对的？

[回复](http://www.wowotech.net/irq_subsystem/irq-domain.html#comment-4562)

**[linuxer](http://www.wowotech.net/)**  
2016-09-14 23:25

@cracker：看你上面栈的回溯，错误应该是发生在驱动初始化的过程中（irq_domain_associate），和irq_find_mapping无关。

[回复](http://www.wowotech.net/irq_subsystem/irq-domain.html#comment-4563)

**[electrlife](http://www.wowotech.net/)**  
2016-03-18 14:51

请教下啊，在我的arch/arm目录中，为什么找不到vector_irq的相关定义，这个函数定义在哪里？

[回复](http://www.wowotech.net/irq_subsystem/irq-domain.html#comment-3687)

**[electrlife](http://www.wowotech.net/)**  
2016-03-18 16:01

@electrlife：哈哈， 找到了，原来是宏vector_stub定义的

[回复](http://www.wowotech.net/irq_subsystem/irq-domain.html#comment-3688)

**z**  
2016-03-16 22:38

@linuxer：你好，  
有个问题没想明白，对于domain级联以及使用radix tree的情况，比如中断1,2,3连在root domain A，中断4,5,6连在domain B，那么domain A和B它所对应的revmap_tree是同一个还是各自有自己的revmap_tree呢？  
因为我看gic_handle_irq中通过hwid去找irq的时候直接去root domain中找domain->revmap_tree；  
如果是这样的话他们的domain->revmap_tree是在哪里赋值的呢？  
谢谢

[回复](http://www.wowotech.net/irq_subsystem/irq-domain.html#comment-3675)

**[郭健](http://www.wowotech.net/)**  
2016-03-17 18:26

@z：那么domain A和B它所对应的revmap_tree是同一个还是各自有自己的revmap_tree呢？  
－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－  
各自有各自的  
  
因为我看gic_handle_irq中通过hwid去找irq的时候直接去root domain中找domain->revmap_tree；  
－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－  
gic_handle_irq函数是什么？是真个系统的中断处理函数，因此，当中断发生后调用gic_handle_irq函数的时候，当然要从root irq domain开始了，你说是吧。  
  
secondary gic和root gic有级联关系，那么实际上secondary gic有双重身份，，一个是做为连接到root gic上的普通设备（因此有IRQ number，有其对应的handler），另外一个是interrupt controller，对应irq domain，负责翻译其负责的HW interrupt ID。  
  
我们假设场景如下：设备D连接到secondary gic的125号硬件中断上，而secondary gic连接到root gic的75号硬件中断上。当设备D产生中断的时候，CPU首先看到的是root gic上的75号中断（在gic_handle_irq函数中处理），然后才进入secondary gic的irq doamin（在gic_handle_cascade_irq函数中处理）。

[回复](http://www.wowotech.net/irq_subsystem/irq-domain.html#comment-3684)

**菜鸟**  
2016-02-24 11:30

hi linuxer,  
如果系统中有对个domain ， 他们的关系是怎么的呢？ 如gic 的一个domain， 和 gpio对应生成的中断 domain。  
是彼此独立还是 gic 为root ？ 其他为级联？ 在中断发生时我只能追踪到gic domain handler，而无法看到调用其他domain的 handler 。

[回复](http://www.wowotech.net/irq_subsystem/irq-domain.html#comment-3521)

**[郭健](http://www.wowotech.net/)**  
2016-02-24 12:46

@菜鸟：irq domain的结构类似linux系统中文件系统的结构，是树状结构，其根节点连接到了CPU，而叶节点连接到了设备。当设备发生了中断事件后，会通过各级的中断控制器反馈给CPU，而kernel的中断子系统会处理这个中断事件，通过各级的irq domain将HW interrupt ID翻译成virtual interrupt ID，也就是IRQ number。  
  
看你的场景，涉及两个domain，一个是GIC irq domain，应该是root，另外一个是GPIO controler的irq domain。对于GIC irq domain而言，他完成的任务就是根据当前的硬件情况将GPIO controller（也是一个interrupt controller）对应的HW interrupt ID翻译成virtual interrupt ID从而调用其handler。  
  
在GPIO controller驱动初始化的时候，会建立其irq domain并且通过irq_set_chained_handler设定该设备的中断处理函数。而GPIO domain的处理都是在这个GPIO controller（或者说GPIO type的中断控制器）中的中断处理函数完成的。  
  
换句话说：中间节点的中断控制器（IRQ DOMAIN）都是有两个角色，一个是做为连接到上一级的中断控制器的普通设备（因此有IRQ number，有其对应的handler），另外一个是interrupt controller，对应irq domain，负责翻译其负责的HW interrupt ID。

[回复](http://www.wowotech.net/irq_subsystem/irq-domain.html#comment-3522)

**菜鸟**  
2016-02-25 13:14

@郭健：我能这样理解吗， 暂且不知道gpio 的中断 domain 和 GIC 的domain在物理上是不是连级，但是在逻辑上可以看成连级？  
对gpio domain中的handler处理也是从这里irq_domain_associate调用的吗？

[回复](http://www.wowotech.net/irq_subsystem/irq-domain.html#comment-3524)

**[郭健](http://www.wowotech.net/)**  
2016-02-25 19:09

@菜鸟：GPIO中断控制器物理上是连接到GIC中断控制器的某一个interrupt request line上，因此，物理上也是级联的。除非你的硬件拓扑结构是所有的GPIO中断都是分别连接到若干GIC中断控制器的若干的interrupt request line上，我估计一般的硬件设计不会这样的。  
  
irq_domain_associate和handler处理无关，这个函数只是建立一个HW interrupt ID和virtual interrupt ID之间的映射，在一个中断触发之前，其mapping就建立了。

[回复](http://www.wowotech.net/irq_subsystem/irq-domain.html#comment-3529)

1 [2](http://www.wowotech.net/irq_subsystem/irq-domain.html/comment-page-2#comments) [3](http://www.wowotech.net/irq_subsystem/irq-domain.html/comment-page-3#comments)

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
    
    - [Linux内核同步机制之（八）：mutex](http://www.wowotech.net/kernel_synchronization/504.html)
    - [调试手段之sys节点](http://www.wowotech.net/linux_application/15.html)
    - [DRAM 原理 3 ：DRAM Device](http://www.wowotech.net/basic_tech/321.html)
    - [Device Tree（三）：代码分析](http://www.wowotech.net/device_model/dt-code-analysis.html)
    - [linux内核中的GPIO系统之（1）：软件框架](http://www.wowotech.net/gpio_subsystem/io-port-control.html)
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
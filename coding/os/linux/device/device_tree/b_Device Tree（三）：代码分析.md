作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2014-6-6 16:03 分类：[统一设备模型](http://www.wowotech.net/sort/device_model)

一、前言

Device Tree总共有三篇，分别是：

1、为何要引入Device Tree，这个机制是用来解决什么问题的？（请参考[引入Device Tree的原因](http://www.wowotech.net/linux_kenrel/why-dt.html)）
2、Device Tree的基础概念（请参考[DT基础概念](http://www.wowotech.net/linux_kenrel/dt_basic_concept.html)）
3、ARM linux中和Device Tree相关的代码分析（这是本文的主题）

本文主要内容是：以Device Tree相关的数据流分析为索引，对ARM linux kernel的代码进行解析。主要的数据流包括：

1、初始化流程。也就是扫描dtb并将其转换成Device Tree Structure。
2、传递运行时参数传递以及platform的识别流程分析
3、如何将Device Tree Structure并入linux kernel的设备驱动模型。

注：本文中的linux kernel使用的是3.14版本。

二、如何通过Device Tree完成运行时参数传递以及platform的识别功能？

1、汇编部分的代码分析

linux/arch/arm/kernel/head.S文件定义了bootloader和kernel的参数传递要求：

> MMU = off, D-cache = off, I-cache = dont care, r0 = 0, r1 = machine nr, r2 = atags or dtb pointer.

目前的kernel支持旧的tag list的方式，同时也支持device tree的方式。r2可能是device tree binary file的指针（bootloader要传递给内核之前要copy到memory中），也可以能是tag list的指针。在ARM的汇编部分的启动代码中（主要是head.S和head-common.S），machine type ID和指向DTB或者atags的指针被保存在变量\_\_machine_arch_type和\_\_atags_pointer中，这么做是为了后续c代码进行处理。

2、和device tree相关的setup_arch代码分析

具体的c代码都是在setup_arch中处理，这个函数是一个总的入口点。具体代码如下（删除了部分无关代码）：

```cpp
void init setup_arch(char **cmdline_p)  
{  
const struct machine_desc mdesc;

……

mdesc = setup_machine_fdt(__atags_pointer);  
if (!mdesc)  
mdesc = setup_machine_tags(__atags_pointer, __machine_arch_type);  
machine_desc = mdesc;  
machine_name = mdesc->name;

……  
}
```

对于如何确定HW platform这个问题，旧的方法是静态定义若干的machine描述符（struct machine_desc ），在启动过程中，通过machine type ID作为索引，在这些静态定义的machine描述符中扫描，找到那个ID匹配的描述符。在新的内核中，首先使用setup_machine_fdt来setup machine描述符，如果返回NULL，才使用传统的方法setup_machine_tags来setup machine描述符。传统的方法需要给出\_\_machine_arch_type（bootloader通过r1寄存器传递给kernel的）和tag list的地址（用来进行tag parse）。\_\_machine_arch_type用来寻找machine描述符；tag list用于运行时参数的传递。随着内核的不断发展，相信有一天linux kernel会完全抛弃tag list的机制。

3、匹配platform（machine描述符）

setup_machine_fdt函数的功能就是根据Device Tree的信息，找到最适合的machine描述符。具体代码如下：

```cpp
const struct machine_desc * init setup_machine_fdt(unsigned int dt_phys)  
{  
const struct machine_desc mdesc, *mdesc_best = NULL;

if (!dt_phys || !early_init_dt_scan(phys_to_virt(dt_phys)))  
return NULL;

mdesc = of_flat_dt_match_machine(mdesc_best, arch_get_next_mach);

if (!mdesc) {   
出错处理  
}

/ Change machine number to match the mdesc we're using /  
__machine_arch_type = mdesc->nr;

return mdesc;  
}
```

early_init_dt_scan函数有两个功能，一个是为后续的DTB scan进行准备工作，另外一个是运行时参数传递。具体请参考下面一个section的描述。

of_flat_dt_match_machine是在machine描述符的列表中scan，找到最合适的那个machine描述符。我们首先看如何组成machine描述符的列表。和传统的方法类似，也是静态定义的。DT_MACHINE_START和MACHINE_END用来定义一个machine描述符。编译的时候，compiler会把这些machine descriptor放到一个特殊的段中（.arch.info.init），形成machine描述符的列表。machine描述符用下面的数据结构来标识（删除了不相关的member）：

> struct machine_desc {\
> unsigned int        nr;        /\* architecture number    \*/\
> const char \*const     *dt_compat;    /* array of device tree 'compatible' strings    \*/
>
> ……
>
> };

nr成员就是过去使用的machine type ID。内核machine描述符的table有若干个entry，每个都有自己的ID。bootloader传递了machine type ID，指明使用哪一个machine描述符。目前匹配machine描述符使用compatible strings，也就是dt_compat成员，这是一个string list，定义了这个machine所支持的列表。在扫描machine描述符列表的时候需要不断的获取下一个machine描述符的compatible字符串的信息，具体的代码如下：

> static const void * \_\_init arch_get_next_mach(const char \*const \*\*match)\
> {\
> static const struct machine_desc \*mdesc = \_\_arch_info_begin;\
> const struct machine_desc \*m = mdesc;
>
> if (m >= \_\_arch_info_end)\
> return NULL;
>
> mdesc++;\
> \*match = m->dt_compat;\
> return m;\
> }

\_\_arch_info_begin指向machine描述符列表第一个entry。通过mdesc++不断的移动machine描述符指针（Note：mdesc是static的）。match返回了该machine描述符的compatible string list。具体匹配的算法倒是很简单，就是比较字符串而已，一个是root node的compatible字符串列表，一个是machine描述符的compatible字符串列表，得分最低的（最匹配的）就是我们最终选定的machine type。

4、运行时参数传递

运行时参数是在扫描DTB的chosen node时候完成的，具体的动作就是获取chosen node的bootargs、initrd等属性的value，并将其保存在全局变量（boot_command_line，initrd_start、initrd_end）中。使用tag list方法是类似的，通过分析tag list，获取相关信息，保存在同样的全局变量中。具体代码位于early_init_dt_scan函数中：

> bool \_\_init early_init_dt_scan(void \*params)\
> {\
> if (!params)\
> return false;
>
> /\* 全局变量initial_boot_params指向了DTB的header\*/\
> initial_boot_params = params;
>
> /\* 检查DTB的magic，确认是一个有效的DTB \*/\
> if (be32_to_cpu(initial_boot_params->magic) != OF_DT_HEADER) {\
> initial_boot_params = NULL;\
> return false;\
> }
>
> /\* 扫描 /chosen node，保存运行时参数（bootargs）到boot_command_line，此外，还处理initrd相关的property，并保存在initrd_start和initrd_end这两个全局变量中 \*/\
> of_scan_flat_dt(early_init_dt_scan_chosen, boot_command_line);
>
> /\* 扫描根节点，获取 {size,address}-cells信息，并保存在dt_root_size_cells和dt_root_addr_cells全局变量中 \*/\
> of_scan_flat_dt(early_init_dt_scan_root, NULL);
>
> /\* 扫描DTB中的memory node，并把相关信息保存在meminfo中，全局变量meminfo保存了系统内存相关的信息。\*/\
> of_scan_flat_dt(early_init_dt_scan_memory, NULL);
>
> return true;\
> }

设定meminfo（该全局变量确定了物理内存的布局）有若干种途径：

1、通过tag list（tag是ATAG_MEM）传递memory bank的信息。

2、通过command line（可以用tag list，也可以通过DTB）传递memory bank的信息。

3、通过DTB的memory node传递memory bank的信息。

目前当然是推荐使用Device Tree的方式来传递物理内存布局信息。

三、初始化流程

在系统初始化的过程中，我们需要将DTB转换成节点是device_node的树状结构，以便后续方便操作。具体的代码位于setup_arch->unflatten_device_tree中。

> void \_\_init unflatten_device_tree(void)\
> {\
> \_\_unflatten_device_tree(initial_boot_params, &of_allnodes,\
> early_init_dt_alloc_memory_arch);
>
> /\* Get pointer to "/chosen" and "/aliases" nodes for use everywhere \*/\
> of_alias_scan(early_init_dt_alloc_memory_arch);\
> }

我们用struct device_node 来抽象设备树中的一个节点，具体解释如下：

> struct device_node {\
> const char \*name;－－－－－－－－－－－－－－－－－－－－－－device node name\
> const char \*type;－－－－－－－－－－－－－－－－－－－－－－－对应device_type的属性\
> phandle phandle;－－－－－－－－－－－－－－－－－－－－－－－对应该节点的phandle属性\
> const char \*full_name; －－－－－－－－－－－－－－－－从“/”开始的，表示该node的full path
>
> struct    property \*properties;－－－－－－－－－－－－－该节点的属性列表\
> struct    property \*deadprops; －－－－－－－－－－如果需要删除某些属性，kernel并非真的删除，而是挂入到deadprops的列表\
> struct    device_node \*parent;－－－－－－parent、child以及sibling将所有的device node连接起来\
> struct    device_node \*child;\
> struct    device_node \*sibling;\
> struct    device_node \*next;  －－－－－－－－通过该指针可以获取相同类型的下一个node\
> struct    device_node \*allnext;－－－－－－－通过该指针可以获取node global list下一个node\
> struct    proc_dir_entry \*pde;－－－－－－－－开放到userspace的proc接口信息\
> struct    kref kref;－－－－－－－－－－－－－该node的reference count\
> unsigned long \_flags;\
> void    \*data;\
> };

unflatten_device_tree函数的主要功能就是扫描DTB，将device node被组织成：

1、global list。全局变量struct device_node \*of_allnodes就是指向设备树的global list

2、tree。

这些功能主要是在\_\_unflatten_device_tree函数中实现，具体代码如下（去掉一些无关紧要的代码）：

> static void \_\_unflatten_device_tree(struct boot_param_header \*blob,－－－需要扫描的DTB\
> struct device_node \*\*mynodes,－－－－－－－－－global list指针\
> void * (\*dt_alloc)(u64 size, u64 align))－－－－－－内存分配函数\
> {\
> unsigned long size;\
> void \*start, \*mem;\
> struct device_node \*\*allnextp = mynodes;
>
> 此处删除了health check代码，例如检查DTB header的magic，确认blob的确指向一个DTB。
>
> /\* scan过程分成两轮，第一轮主要是确定device-tree structure的长度，保存在size变量中 \*/\
> start = ((void \*)blob) + be32_to_cpu(blob->off_dt_struct);\
> size = (unsigned long)unflatten_dt_node(blob, 0, &start, NULL, NULL, 0);\
> size = ALIGN(size, 4);
>
> /\* 初始化的时候，并不是扫描到一个node或者property就分配相应的内存，实际上内核是一次性的分配了一大片内存，这些内存包括了所有的struct device_node、node name、struct property所需要的内存。\*/\
> mem = dt_alloc(size + 4, __alignof__(struct device_node));\
> memset(mem, 0, size);
>
> \*(\_\_be32 \*)(mem + size) = cpu_to_be32(0xdeadbeef);   //用来检验后面unflattening是否溢出
>
> /\* 这是第二轮的scan，第一次scan是为了得到保存所有node和property所需要的内存size，第二次就是实打实的要构建device node tree了 \*/\
> start = ((void \*)blob) + be32_to_cpu(blob->off_dt_struct);\
> unflatten_dt_node(blob, mem, &start, NULL, &allnextp, 0);
>
> 此处略去校验溢出和校验OF_DT_END。\
> }

具体的scan是在unflatten_dt_node函数中，如果已经清楚地了解DTB的结构，其实代码很简单，这里就不再细述了。

四、如何并入linux kernel的设备驱动模型

在linux kernel引入统一设备模型之后，bus、driver和device形成了设备模型中的铁三角。在驱动初始化的时候会将代表该driver的一个数据结构（一般是xxx_driver）挂入bus上的driver链表。device挂入链表分成两种情况，一种是即插即用类型的bus，在插入一个设备后，总线可以检测到这个行为并动态分配一个device数据结构（一般是xxx_device，例如usb_device），之后，将该数据结构挂入bus上的device链表。bus上挂满了driver和device，那么如何让device遇到“对”的那个driver呢？那么就要靠缘分了，也就是bus的match函数。

上面是一段导论，我们还是回到Device Tree。导致Device Tree的引入ARM体系结构的代码其中一个最重要的原因的太多的静态定义的表格。例如：一般代码中会定义一个static struct platform_device \*xxx_devices的静态数组，在初始化的时候调用platform_add_devices。这些静态定义的platform_device往往又需要静态定义各种resource，这导致静态表格进一步增大。如果ARM linux中不再定义这些表格，那么一定需要一个转换的过程，也就是说，系统应该会根据Device tree来动态的增加系统中的platform_device。当然，这个过程并非只是发生在platform bus上（具体可以参考[“Platform Device”的设备](http://www.wowotech.net/linux_kenrel/platform_device.html)），也可能发生在其他的非即插即用的bus上，例如AMBA总线、PCI总线。一言以蔽之，如果要并入linux kernel的设备驱动模型，那么就需要根据device_node的树状结构（root是of_allnodes）将一个个的device node挂入到相应的总线device链表中。只要做到这一点，总线机制就会安排device和driver的约会。

当然，也不是所有的device node都会挂入bus上的设备链表，比如cpus node，memory node，choose node等。

1、cpus node的处理

这部分的处理可以参考setup_arch->arm_dt_init_cpu_maps中的代码，具体的代码如下：

> void \_\_init arm_dt_init_cpu_maps(void)\
> {\
> scan device node global list，寻找full path是“/cpus”的那个device node。cpus这个device node只是一个容器，其中包括了各个cpu node的定义以及所有cpu node共享的property。\
> cpus = of_find_node_by_path("/cpus");
>
> for_each_child_of_node(cpus, cpu) {           遍历cpus的所有的child node\
> u32 hwid;
>
> if (of_node_cmp(cpu->type, "cpu"))        我们只关心那些device_type是cpu的node\
> continue;
>
> if (of_property_read_u32(cpu, "reg", &hwid)) {    读取reg属性的值并赋值给hwid\
> return;\
> }
>
> reg的属性值的8 MSBs必须设置为0，这是ARM CPU binding定义的。\
> if (hwid & ~MPIDR_HWID_BITMASK)  \
> return;
>
> 不允许重复的CPU id，那是一个灾难性的设定\
> for (j = 0; j \< cpuidx; j++)\
> if (WARN(tmp_map\[j\] == hwid, "Duplicate /cpu reg "\
> "properties in the DT\\n"))\
> return;
>
> 数组tmp_map保存了系统中所有CPU的MPIDR值（CPU ID值），具体的index的编码规则是： tmp_map\[0\]保存了booting CPU的id值，其余的CPU的ID值保存在1～NR_CPUS的位置。\
> if (hwid == mpidr) {\
> i = 0;\
> bootcpu_valid = true;\
> } else {\
> i = cpuidx++;\
> }
>
> tmp_map\[i\] = hwid;\
> }
>
> 根据DTB中的信息设定cpu logical map数组。
>
> for (i = 0; i \< cpuidx; i++) {\
> set_cpu_possible(i, true);\
> cpu_logical_map(i) = tmp_map\[i\];\
> }\
> }

要理解这部分的内容，需要理解ARM CUPs binding的概念，可以参考linux/Documentation/devicetree/bindings/arm目录下的CPU.txt文件的描述。

2、memory的处理

这部分的处理可以参考setup_arch->setup_machine_fdt->early_init_dt_scan->early_init_dt_scan_memory中的代码。具体如下：

> int \_\_init early_init_dt_scan_memory(unsigned long node, const char \*uname,\
> int depth, void \*data)\
> {\
> char \*type = of_get_flat_dt_prop(node, "device_type", NULL); 获取device_type属性值\
> \_\_be32 \*reg, \*endp;\
> unsigned long l;
>
> 在初始化的时候，我们会对每一个device node都要调用该call back函数，因此，我们要过滤掉那些和memory block定义无关的node。和memory block定义有的节点有两种，一种是node name是memory@形态的，另外一种是node中定义了device_type属性并且其值是memory。\
> if (type == NULL) {\
> if (depth != 1 || strcmp(uname, "memory@0") != 0)\
> return 0;\
> } else if (strcmp(type, "memory") != 0)\
> return 0;
>
> 获取memory的起始地址和length的信息。有两种属性和该信息有关，一个是linux,usable-memory，不过最新的方式还是使用reg属性。
>
> reg = of_get_flat_dt_prop(node, "linux,usable-memory", &l);\
> if (reg == NULL)\
> reg = of_get_flat_dt_prop(node, "reg", &l);\
> if (reg == NULL)\
> return 0;
>
> endp = reg + (l / sizeof(\_\_be32));
>
> reg属性的值是address，size数组，那么如何来取出一个个的address/size呢？由于memory node一定是root node的child，因此dt_root_addr_cells（root node的#address-cells属性值）和dt_root_size_cells（root node的#size-cells属性值）之和就是address，size数组的entry size。
>
> while ((endp - reg) >= (dt_root_addr_cells + dt_root_size_cells)) {\
> u64 base, size;
>
> base = dt_mem_next_cell(dt_root_addr_cells, ®);\
> size = dt_mem_next_cell(dt_root_size_cells, ®);
>
> early_init_dt_add_memory_arch(base, size);  将具体的memory block信息加入到内核中。\
> }
>
> return 0;\
> }

3、interrupt controller的处理

初始化是通过start_kernel->init_IRQ->machine_desc->init_irq()实现的。我们用S3C2416为例来描述interrupt controller的处理过程。下面是machine描述符的定义。

> DT_MACHINE_START(S3C2416_DT, "Samsung S3C2416 (Flattened Device Tree)")\
> ……\
> .init_irq    = irqchip_init,\
> ……\
> MACHINE_END

在driver/irqchip/irq-s3c24xx.c文件中定义了两个interrupt controller，如下：

> IRQCHIP_DECLARE(s3c2416_irq, "samsung,s3c2416-irq", s3c2416_init_intc_of);
>
> IRQCHIP_DECLARE(s3c2410_irq, "samsung,s3c2410-irq", s3c2410_init_intc_of);

当然，系统中可以定义更多的irqchip，不过具体用哪一个是根据DTB中的interrupt controller node中的compatible属性确定的。在driver/irqchip/irqchip.c文件中定义了irqchip_init函数，如下：

> void \_\_init irqchip_init(void)\
> {\
> of_irq_init(\_\_irqchip_begin);\
> }

\_\_irqchip_begin就是所有的irqchip的一个列表，of_irq_init函数是遍历Device Tree，找到匹配的irqchip。具体的代码如下：

> void \_\_init of_irq_init(const struct of_device_id \*matches)\
> {\
> struct device_node \*np, \*parent = NULL;\
> struct intc_desc \*desc, \*temp_desc;\
> struct list_head intc_desc_list, intc_parent_list;
>
> INIT_LIST_HEAD(&intc_desc_list);\
> INIT_LIST_HEAD(&intc_parent_list);
>
> 遍历所有的node，寻找定义了interrupt-controller属性的node，如果定义了interrupt-controller属性则说明该node就是一个中断控制器。
>
> for_each_matching_node(np, matches) {\
> if (!of_find_property(np, "interrupt-controller", NULL) ||\
> !of_device_is_available(np))\
> continue;
>
> 分配内存并挂入链表，当然还有根据interrupt-parent建立controller之间的父子关系。对于interrupt controller，它也可能是一个树状的结构。\
> desc = kzalloc(sizeof(\*desc), GFP_KERNEL);\
> if (WARN_ON(!desc))\
> goto err;
>
> desc->dev = np;\
> desc->interrupt_parent = of_irq_find_parent(np);\
> if (desc->interrupt_parent == np)\
> desc->interrupt_parent = NULL;\
> list_add_tail(&desc->list, &intc_desc_list);\
> }
>
> 正因为interrupt controller被组织成树状的结构，因此初始化的顺序就需要控制，应该从根节点开始，依次递进到下一个level的interrupt controller。\
> while (!list_empty(&intc_desc_list)) {  intc_desc_list链表中的节点会被一个个的处理，每处理完一个节点就会将该节点删除，当所有的节点被删除，整个处理过程也就是结束了。\
> \
> list_for_each_entry_safe(desc, temp_desc, &intc_desc_list, list) {\
> const struct of_device_id \*match;\
> int ret;\
> of_irq_init_cb_t irq_init_cb;
>
> 最开始的时候parent变量是NULL，确保第一个被处理的是root interrupt controller。在处理完root node之后，parent变量被设定为root interrupt controller，因此，第二个循环中处理的是所有parent是root interrupt controller的child interrupt controller。也就是level 1（如果root是level 0的话）的节点。
>
> if (desc->interrupt_parent != parent)\
> continue;
>
> list_del(&desc->list);      －－－－－从链表中删除\
> match = of_match_node(matches, desc->dev);－－－－－匹配并初始化\
> if (WARN(!match->data,－－－－－－－－－－match->data是初始化函数\
> "of_irq_init: no init function for %s\\n",\
> match->compatible)) {\
> kfree(desc);\
> continue;\
> }
>
> irq_init_cb = (of_irq_init_cb_t)match->data;\
> ret = irq_init_cb(desc->dev, desc->interrupt_parent);－－－－－执行初始化函数\
> if (ret) {\
> kfree(desc);\
> continue;\
> }
>
> 处理完的节点放入intc_parent_list链表，后面会用到\
> list_add_tail(&desc->list, &intc_parent_list);\
> }
>
> 对于level 0，只有一个root interrupt controller，对于level 1，可能有若干个interrupt controller，因此要遍历这些parent interrupt controller，以便处理下一个level的child node。\
> desc = list_first_entry_or_null(&intc_parent_list,\
> typeof(\*desc), list);\
> if (!desc) {\
> pr_err("of_irq_init: children remain, but no parents\\n");\
> break;\
> }\
> list_del(&desc->list);\
> parent = desc->dev;\
> kfree(desc);\
> }
>
> list_for_each_entry_safe(desc, temp_desc, &intc_parent_list, list) {\
> list_del(&desc->list);\
> kfree(desc);\
> }\
> err:\
> list_for_each_entry_safe(desc, temp_desc, &intc_desc_list, list) {\
> list_del(&desc->list);\
> kfree(desc);\
> }\
> }

只有该node中有interrupt-controller这个属性定义，那么linux kernel就会分配一个interrupt controller的描述符（struct intc_desc）并挂入队列。通过interrupt-parent属性，可以确定各个interrupt controller的层次关系。在scan了所有的Device Tree中的interrupt controller的定义之后，系统开始匹配过程。一旦匹配到了interrupt chip列表中的项次后，就会调用相应的初始化函数。如果CPU是S3C2416的话，匹配到的是irqchip的初始化函数是s3c2416_init_intc_of。

OK，我们已经通过compatible属性找到了适合的interrupt controller，那么如何解析reg属性呢？我们知道，对于s3c2416的interrupt controller而言，其#interrupt-cells的属性值是4，定义为。每个域的解释如下：

（1）ctrl_num表示使用哪一种类型的interrupt controller，其值的解释如下：

\- 0 ... main controller\
\- 1 ... sub controller\
\- 2 ... second main controller

（2）parent_irq。对于sub controller，parent_irq标识了其在main controller的bit position。

（3）ctrl_irq标识了在controller中的bit位置。

（4）type标识了该中断的trigger type，例如：上升沿触发还是电平触发。

为了更顺畅的描述后续的代码，我需要简单的介绍2416的中断控制器，其block diagram如下：

[![2416intc](http://www.wowotech.net/content/uploadfile/201406/af17f85d8a8d3d689caf5c62fcb8ea1420140606080340.gif "2416intc")](http://www.wowotech.net/content/uploadfile/201406/1fae6636c0a85be88e3177559936bad420140606080337.gif)

53个Samsung2416的中断源被分成两种类型，一种是需要sub寄存器进行控制的，例如DMA，系统中的8个DMA中断是通过两级识别的，先在SRCPND寄存器中得到是DMA中断的信息，具体是哪一个channel的DMA中断需要继续查询SUBSRC寄存器。那些不需要sub寄存器进行控制的，例如timer，5个timer的中断可以直接从SRCPND中得到。\
中断MASK寄存器可以控制产生的中断是否要报告给CPU，当一个中断被mask的时候，虽然SRCPND寄存器中，硬件会set该bit，但是不会影响到INTPND寄存器，从而不会向CPU报告该中断。对于SUBMASK寄存器，如果该bit被set，也就是该sub中断被mask了，那么即便产生了对应的sub中断，也不会修改SRCPND寄存器的内容，只是修改SUBSRCPND中寄存器的内容。

不过随着硬件的演化，更多的HW block加入到SOC中，这使得中断源不够用了，因此中断寄存器又被分成两个group，一个是group 1（开始地址是0X4A000000，也就是main controller了），另外一个是group2（开始地址是0X4A000040，叫做second main controller）。group 1中的sub寄存器的起始地址是0X4A000018（也就是sub controller）。

了解了上面的内容后，下面的定义就比较好理解了：

> static struct s3c24xx_irq_of_ctrl s3c2416_ctrl\[\] = {\
> {\
> .name = "intc", －－－－－－－－－－－main controller\
> .offset = 0,\
> }, {\
> .name = "subintc", －－－－－－－－－sub controller\
> .offset = 0x18,\
> .parent = &s3c_intc\[0\],\
> }, {\
> .name = "intc2", －－－－－－－－－－second main controller\
> .offset = 0x40,\
> }\
> };

对于s3c2416而言，irqchip的初始化函数是s3c2416_init_intc_of，s3c2416_ctrl作为参数传递给了s3c_init_intc_of，大部分的处理都是在s3c_init_intc_of函数中完成的，由于这个函数和中断子系统非常相关，这里就不详述了，后续会有一份专门的文档描述之。

4、GPIO controller的处理

暂不描述，后续会有一份专门的文档描述GPIO sub system。

5、machine初始化

machine初始化的代码可以沿着start_kernel->rest_init->kernel_init->kernel_init_freeable->do_basic_setup->do_initcalls路径寻找。在do_initcalls函数中，kernel会依次执行各个initcall函数，在这个过程中，会调用customize_machine，具体如下：

> static int \_\_init customize_machine(void)\
> {
>
> if (machine_desc->init_machine)\
> machine_desc->init_machine();\
> else\
> of_platform_populate(NULL, of_default_bus_match_table, NULL, NULL);
>
> return 0;\
> }\
> arch_initcall(customize_machine);

在这个函数中，一般会调用machine描述符中的init_machine callback函数来把各种Device Tree中定义的platform device设备节点加入到系统（即platform bus的所有的子节点，对于device tree中其他的设备节点，需要在各自bus controller初始化的时候自行处理）。如果machine描述符中没有定义init_machine函数，那么直接调用of_platform_populate把所有的platform device加入到kernel中。对于s3c2416，其machine描述符中的init_machine callback函数就是s3c2416_dt_machine_init，代码如下：

> static void \_\_init s3c2416_dt_machine_init(void)\
> {\
> of_platform_populate(NULL, --------传入NULL参数表示从root node开始scan
>
> of_default_bus_match_table, s3c2416_auxdata_lookup, NULL);
>
> s3c_pm_init(); －－－－－－－－power management相关的初始化\
> }

由此可见，最终生成platform device的代码来自of_platform_populate函数。该函数的逻辑比较简单，遍历device node global list中所有的node，并调用of_platform_bus_create处理，of_platform_bus_create函数代码如下：

> static int of_platform_bus_create(struct device_node \*bus,-------------要创建的那个device node\
> const struct of_device_id \*matches,-------要匹配的list\
> const struct of_dev_auxdata \*lookup,------附属数据\
> struct device \*parent, bool strict)---------------parent指向父节点。strict是否要求完全匹配\
> {\
> const struct of_dev_auxdata \*auxdata;\
> struct device_node \*child;\
> struct platform_device \*dev;\
> const char \*bus_id = NULL;\
> void \*platform_data = NULL;\
> int rc = 0;
>
> 删除确保device node有compatible属性的代码。
>
> auxdata = of_dev_lookup(lookup, bus);  在传入的lookup table寻找和该device node匹配的附加数据\
> if (auxdata) {\
> bus_id = auxdata->name;-----------------如果找到，那么就用附加数据中的静态定义的内容\
> platform_data = auxdata->platform_data;\
> }
>
> ARM公司提供了CPU core，除此之外，它设计了AMBA的总线来连接SOC内的各个block。符合这个总线标准的SOC上的外设叫做ARM Primecell Peripherals。如果一个device node的compatible属性值是arm,primecell的话，可以调用of_amba_device_create来向amba总线上增加一个amba device。
>
> if (of_device_is_compatible(bus, "arm,primecell")) {\
> of_amba_device_create(bus, bus_id, platform_data, parent);\
> return 0;\
> }
>
> 如果不是ARM Primecell Peripherals，那么我们就需要向platform bus上增加一个platform device了
>
> dev = of_platform_device_create_pdata(bus, bus_id, platform_data, parent);\
> if (!dev || !of_match_node(matches, bus))\
> return 0;
>
> 一个device node可能是一个桥设备，因此要重复调用of_platform_bus_create来把所有的device node处理掉。
>
> for_each_child_of_node(bus, child) {\
> pr_debug("   create child: %s\\n", child->full_name);\
> rc = of_platform_bus_create(child, matches, lookup, &dev->dev, strict);\
> if (rc) {\
> of_node_put(child);\
> break;\
> }\
> }\
> return rc;\
> }

具体增加platform device的代码在of_platform_device_create_pdata中，代码如下：

> static struct platform_device \*of_platform_device_create_pdata(\
> struct device_node \*np,\
> const char \*bus_id,\
> void \*platform_data,\
> struct device \*parent)\
> {\
> struct platform_device \*dev;
>
> if (!of_device_is_available(np))---------check status属性，确保是enable或者OK的。\
> return NULL;
>
> of_device_alloc除了分配struct platform_device的内存，还分配了该platform device需要的resource的内存（参考struct platform_device 中的resource成员）。当然，这就需要解析该device node的interrupt资源以及memory address资源。
>
> dev = of_device_alloc(np, bus_id, parent);\
> if (!dev)\
> return NULL;
>
> 设定platform_device 中的其他成员\
> dev->dev.coherent_dma_mask = DMA_BIT_MASK(32);\
> if (!dev->dev.dma_mask)\
> dev->dev.dma_mask = &dev->dev.coherent_dma_mask;\
> dev->dev.bus = &platform_bus_type;\
> dev->dev.platform_data = platform_data;
>
> if (of_device_add(dev) != 0) {------------------把这个platform device加入统一设备模型系统中\
> platform_device_put(dev);\
> return NULL;\
> }
>
> return dev;\
> }

_原创文章，转发请注明出处。蜗窝科技_，[www.wowotech.net。](http://www.wowotech.net/linux_kenrel/dt-code-analysis.html)

标签: [设备树](http://www.wowotech.net/tag/%E8%AE%BE%E5%A4%87%E6%A0%91)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Linux电源管理(5)\_Hibernate和Sleep功能介绍](http://www.wowotech.net/pm_subsystem/std_str_func.html) | [Device Tree（二）：基本概念](http://www.wowotech.net/device_model/dt_basic_concept.html)»

**评论：**

**aha**\
2023-07-21 17:27

从arm64的setup_arch来看，arm64架构不需要做platform匹配，只检查了root节点是否有model或者compatible。也就是说不同的arm64的soc都是无差异的setup arch过程，不需要实现struct machine_desc结构体中各种回调。这是因为arm64的soc真的不需要，还是这种差别化分散到其他驱动？

[回复](http://www.wowotech.net/device_model/dt-code-analysis.html#comment-8798)

**在渊**\
2021-01-08 16:13

wowo大佬，请教一下。设备总线驱动模型中，在总线创建时会注册设备，如文中platform总线从device_node遍历后会有device_add()动作，那为啥在驱动中，我们往往也还有注册class，注册device的动作。驱动中的device往往用于和用户态交互，这里是否就是为了这个原因创建，这里的设备如何和设备模型统一起来。

[回复](http://www.wowotech.net/device_model/dt-code-analysis.html#comment-8182)

**一一**\
2020-08-06 19:19

hi，wowo大神，想请教一下\
cpus node的中对compatible = "arm,cortex-a7"是如何处理的\
我大致追了一下源码：\
rest_init-->kernel_init-->smp_prepare_cpus-->init_cpu_topology-->parse_dt_topology中有一段

for (cpu_eff = table_efficiency; cpu_eff->compatible; cpu_eff++)\
if (of_device_is_compatible(cn, cpu_eff->compatible))\
break;

static const struct cpu_efficiency table_efficiency\[\] = {\
{"arm,cortex-a15", 3891},\
{"arm,cortex-a7",  2048},\
{NULL, },\
};

但是table_efficiency只有a15和a7 没看见a9等其他型号，所以不知道我找的是不是正确，以上是32位arm。\
64位arm分析半天，好像都没有看见比较compatible属性。

[回复](http://www.wowotech.net/device_model/dt-code-analysis.html#comment-8085)

**Warren**\
2020-01-19 17:50

补充一点：i2c和spi有点特殊，i2c和spi有主从设备之分，并且有自己的总线i2c bus和spi bus。挂载在platform bus下的只是i2c和spi的controller设备（即master），以iMX的i2c举例，在执行of_platform_populate()时创建的platform device只代表了I2C controller，在注册platform_driver(如i2c_imx_driver)时会执行platform_driver的probe函数，即i2c_imx_probe()，probe函数会分配一个i2c_adapter作为i2c controller的抽象设备挂接在i2c bus上。同时该probe函数会遍历该i2c controller设备树节点下的所有i2c子节点，创建i2c_client设备并将i2c_client->adapter指向这个i2c_adapter。随后在i2c client加入i2c bus时或者i2c client的驱动i2c_driver加入i2c bus时，会匹配i2c_client与i2c_driver，匹配成功会执行i2c_driver的probe函数来初始化i2c_client。i2c_client通过adapter指针指向自己的i2c_adapter，i2c_adapter实现i2c协议传输的方法。上层应用在使用i2c_client的字符设备时最终通过该指针来操作i2c_adapter发出正确的时序和数据。

[回复](http://www.wowotech.net/device_model/dt-code-analysis.html#comment-7852)

**Warren**\
2020-01-20 09:08

@Warren：回错地方了,本来是想回复这个兄弟的，无奈回复中出现了敏感词(sla-ve),改了敏感词后被重置到首页了：\
jinxin\
2015-12-22 10:49\
@wowo：请问那像一些i2c, spi设备呢？也是会挂在platform bus下么?\
这好象不合适把

[回复](http://www.wowotech.net/device_model/dt-code-analysis.html#comment-7854)

**勾越**\
2018-04-26 19:08

1、您好，文中的：全局变量struct device_node \*of_allnodes就是指向设备树的global list\
我在内核里面没有找到啊？？？我在system.map里面也找了。

[回复](http://www.wowotech.net/device_model/dt-code-analysis.html#comment-6706)

**hello**\
2018-04-26 19:14

@勾越：文章中有这样一句话\
注：本文中的linux kernel使用的是3.14版本。\
可能不同的内核版本会有不同

[回复](http://www.wowotech.net/device_model/dt-code-analysis.html#comment-6707)

**勾越**\
2018-04-26 19:25

@hello：您好，我的是kernel4.1版本，请问有什么方法去找到我这个global list的全局指针变量呢？

[回复](http://www.wowotech.net/device_model/dt-code-analysis.html#comment-6708)

**jake**\
2020-03-30 09:38

@勾越：你找找&of_root

[回复](http://www.wowotech.net/device_model/dt-code-analysis.html#comment-7933)

**勾越**\
2018-04-26 19:04

1、您好，我没在内核中找到全局变量struct device_node \*of_allnodes（就是指向设备树的global list）啊？system.map也找了，也还是没有。

[回复](http://www.wowotech.net/device_model/dt-code-analysis.html#comment-6705)

**小明**\
2018-01-17 16:17

wowo大神你好，看你的文章受益匪浅，真的是用嵌入式工程师的思维讲解，很容易理解接受。\
最近遇到个问题，我想在用户空间获取并解析整个设备树，来获取一些配置信息。但用户空间改怎么获取呢？有现成的方案吗？谢谢

[回复](http://www.wowotech.net/device_model/dt-code-analysis.html#comment-6482)

**maze**\
2018-11-01 14:59

@小明：这个应该是有的/proc下面是有一个device tree的东西吧。不过印象是通过defconfig配置出来的选项

[回复](http://www.wowotech.net/device_model/dt-code-analysis.html#comment-7010)

**mikhail**\
2019-01-22 11:22

@小明：kernel_path/drivers/of/Kconfig

config PROC_DEVICETREE

重新编译内核，系统就会生成文件 /proc/device-tree

[回复](http://www.wowotech.net/device_model/dt-code-analysis.html#comment-7146)

**yellow**\
2017-08-22 20:01

@wowo:\
我结合代码重新看了这篇文章，有个地方想不明白：\
phandle 这个属性值，根据什么初始化产生的呢？（我的理解是每个节点都有这个属性，因为其他节点需要引用）\
比如 interrupt-map = \<0 0 0 0 &intc 0 0 405 0> ，这个&intc，从代码看是根据它的phandle，来找到intc所在的节点\
但从dtsi上，intc这个节点没看到有定义 “phandle”"linux,phandle"的属性。那么intc这个节点的phandle从哪里产生的？

劳烦解答一下，谢谢。

[回复](http://www.wowotech.net/device_model/dt-code-analysis.html#comment-5940)

**[wowo](http://www.wowotech.net/)**\
2017-08-24 09:40

@yellow：phandle是编译器（dtc）自行添加的，具体你可以去扒扒dtc的source code。

[回复](http://www.wowotech.net/device_model/dt-code-analysis.html#comment-5944)

**yellow**\
2017-08-24 18:38

@wowo：谢谢。。看您的文章越读越觉得受益匪浅~

[回复](http://www.wowotech.net/device_model/dt-code-analysis.html#comment-5948)

**yellow**\
2017-06-16 18:01

@wowo： 请教下，我打log发现，扫描device tree添加device的时候，并不是所有node都会注册成设备的，除了soc,已经下一层的soc/\*\*\*/，就是说只会注册device tree根节点已经第二层节点，请问对吗，代码从哪里体现出来呢？

\<3>\[    2.032465\]  \[4:      swapper/0:    1\]  create bus: /soc\
\<3>\[    2.038388\]  \[4:      swapper/0:    1\]  create child: /soc/sec_hw_param\
\<3>\[    2.043990\]  \[4:      swapper/0:    1\]  create child: /soc/qcom,smp2p-modem@17911008\
\<3>\[    2.052261\]  \[4:      swapper/0:    1\]  create child: /soc/qcom,smp2p-adsp@17911008\
.....

[回复](http://www.wowotech.net/device_model/dt-code-analysis.html#comment-5684)

**[wowo](http://www.wowotech.net/)**\
2017-06-19 09:00

@yellow：是的，你可以往下翻翻之前的留言，其中有同学讨论过这个事情：\
http://www.wowotech.net/device_model/dt-code-analysis.html/comment-page-2\
（搜索simple-bus这个关键字）

[回复](http://www.wowotech.net/device_model/dt-code-analysis.html#comment-5691)

**helloworld**\
2017-01-11 10:49

请教wowo，编译的内核中有很多IRQCHIP_DECLARE定义的of_device_id结构体，对于interrupt controller的匹配，of_irq_init会去遍历所有的of_device_id结构体吗？\
这里的匹配是根据dts的interrupt-controller节点和of_device_id的compatible属性进行匹配的吗？我看了很久好像没有看到，能指点下具体在哪里吗？

[回复](http://www.wowotech.net/device_model/dt-code-analysis.html#comment-5127)

**helloworld**\
2017-01-11 11:45

@helloworld：是我太粗心了，已经找到了，原来是在\_\_of_match_node对matches轮询匹配root interrupt controller.

[回复](http://www.wowotech.net/device_model/dt-code-analysis.html#comment-5129)

1 [2](http://www.wowotech.net/device_model/dt-code-analysis.html/comment-page-2#comments) [3](http://www.wowotech.net/device_model/dt-code-analysis.html/comment-page-3#comments) [4](http://www.wowotech.net/device_model/dt-code-analysis.html/comment-page-4#comments) [5](http://www.wowotech.net/device_model/dt-code-analysis.html/comment-page-5#comments) [6](http://www.wowotech.net/device_model/dt-code-analysis.html/comment-page-6#comments)

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

  - [X-020-ROOTFS-initramfs的制作和测试](http://www.wowotech.net/x_project/simple_initramfs.html)
  - [ACCESS_ONCE宏定义的解释](http://www.wowotech.net/process_management/access-once.html)
  - [Common Clock Framework系统结构](http://www.wowotech.net/pm_subsystem/ccf-arch.html)
  - [load_balance函数代码详解](http://www.wowotech.net/process_management/load_balance_function.html)
  - [USB-C(USB Type-C)规范的简单介绍和分析](http://www.wowotech.net/usb/usb_type_c_overview.html)

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

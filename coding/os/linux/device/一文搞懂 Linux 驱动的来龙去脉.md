
布道师Peter Linux内核之旅

 _2022年02月06日 19:24_

驱动相关的学习资料网上很多，但大部分都是碎片化的记录，很少有系统化的总结整理。本文旨在系统化的讲清楚 Linux 驱动的来龙去脉。先从总线，设备，驱动介绍内核对于驱动的模型设计；然后引入设备树的概念，打通设备和驱动的关系；最后手把手和你一起定制自己的开发板。这也是拿到芯片后，如何 bringup 板子的入门工作。详细交流可以加我微信 rrjike，一起学习进步！

# Linux 总线、设备、驱动模型的探究

## 设备驱动模型的需求

总线、设备和驱动模型，如果把它们之间的关系比喻成生活中的例子是比较容易理解的。举个例子，充电墙壁插座安静的嵌入在墙面上，无论设备是电脑还是手机，插座都能依然不动的完成它的使命——充电，没有说为了满足各种设备充电而去更换插座的。其实这就是软件工程强调的高内聚、低耦合概念。

所谓高内聚低耦合是模块内各元素联系越紧密就代表内聚性就越高，模块间联系越不紧密就代表耦合性低。所以高内聚、低耦合强调的就是内部要紧紧抱团。设备和驱动就是基于这种模型去实现彼此隔离不相干的。这里，有的读者就要问了，高内聚、低耦合的软件模型理解，可设备和驱动为什么要采用这种模型呢？没错，好问题。下面进入今天的话题——总线、设备和驱动模型的探究。

设想一个叫 GITCHAT 的网卡，它需要接在 CPU 的内部总线上，需要地址总线、数据总线和控制总线，以及中断 pin 脚等。

![图片](https://mmbiz.qpic.cn/mmbiz_png/9sNwsXcN68oBXaZWDS3PIzNKxMPABZx5MEHplBu54dFL8f8bDPXm28bxSzJOt7YQ8tBk8EaB5YKxAXCVctl3WQ/640?wx_fmt=png&wxfrom=13&tp=wxpic)

那么在 GITCHAT 的驱动里需要定义 GITCHAT 的基地址、中断号等信息。假设 GITCHAT 的地址为0x0001，中断号是 2，那么：

`#define GITCHAT_BASE 0x0001   #define GITCHAT_INTERRUPT 2      int gitchat_send()   {       writel(GITCHAT_BASE + REG, 1);       ...   }      int gitchat_init()   {       request_init(GITCHAT_INTERRUPT, ...);       ...   }   `

但是世界上的板子千千万，有三星、华为、飞思卡尔……每个板子的信息也都不一样，站在驱动的角度看，当每次重新换板子的时候，GITCHAT_BASE 和 GITCHAT_INTERRUPT 就不再一样，那驱动代码也要随之改变。这样的话一万个开发板要写一万个驱动了，这就是文章刚开始提到的高内聚、低耦合的应用场景。

驱动想以不变应万变的姿态适配各种设备连接的话就要实现设备驱动模型。基本上我们可以认为驱动不会因为 CPU 的改变而改变，所以它应该是跨平台的。自然像 “#define GITCHAT_BASE 0x0001，#define GITCHAT_INTERRUPT 2” 这样描述和 CPU 相关信息的代码不应该出现在驱动里。

## 设备驱动模型的实现

现在 CPU 板级信息和驱动分开的需求已经刻不容缓。但是基地址、中断号等板级信息始终和驱动是有一定联系的，因为驱动毕竟要取出基地址、中断号等。怎么取？有一种方法是 GITCHAT 驱动满世界去询问各个板子：请问你的基地址是多少？中断号是几？细心的读者会发现这仍然是一个耦合的情况。

![图片](https://mmbiz.qpic.cn/mmbiz_png/9sNwsXcN68oBXaZWDS3PIzNKxMPABZx5RUZLqRIQbNKsCtdbiabglGPibQWHYYzJuAWibbDRXf1lFelGDpH32WqqQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

对软件工程熟悉的读者肯定立刻想到能不能设计一个类似接口适配器的类（adapter）去适配不同的板级信息，这样板子上的基地址、中断号等信息都在一个 adapter 里去维护，然后驱动通过这个 adapter 不同的 API 去获取对应的硬件信息。没错，Linux 内核里就是运用了这种设计思想去对设备和驱动进行适配隔离的，只不过在内核里我们不叫做适配层，而取名为总线，意为通过这个总线去把驱动和对应的设备绑定一起，如图：

![图片](https://mmbiz.qpic.cn/mmbiz_png/9sNwsXcN68oBXaZWDS3PIzNKxMPABZx5GEhMWHajY8INMyfYULUwXW9QC8laVN1EAnBWeOV632fKeHFChf840w/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

基于这种设计思想，Linux 把设备驱动分为了总线、设备和驱动三个实体，这三个实体在内核里的职责分别如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/9sNwsXcN68oBXaZWDS3PIzNKxMPABZx50OpH128v6ILg8icFIh1KVuIqKt7KjaTEBU9ib8jiby9hdm3kzjeXpicyJQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

模型设计好后，下面来看一下具体驱动的实践，首先把板子的硬件信息填入设备端，然后让设备向总线注册，这样总线就间接的知道了设备的硬件信息。比如一个板子上有一个 GITCHAT，首先向总线注册：

`static struct resource gitchat_resource[] = {       {               .start = ...,               .end = ...,               .flags = IORESOURCE_MEM       }...   };      static struct platform_device gitchat_device = {       .name = "gitchat";       .id = 0;       .num_resources = ARRAY_SIZE(gitchat_resource);       .resource = gitchat_resource,   };      static struct platform_device *ip0x_device __initdata = {       &gitchat_device,       ...   };      static ini __init ip0x_init(void)   {       platform_add_devices(ip0x_device, ARRAY_SIZE(ip0x_device));   }   `

现在 platform 总线上自然知道了板子上关于 GITCHAT 设备的硬件信息，一旦 GITCHAT 的驱动也被注册的话，总线就会把驱动和设备绑定起来，从而驱动就获得了基地址、中断号等板级信息。总线存在的目的就是把设备和对应的驱动绑定起来，让内核成为该是谁的就是谁的和谐世界，有点像我们生活中红娘的角色，把有缘人通过红线牵在一起。设备注册总线的代码例子看完了，下面看下驱动注册总线的代码示例：

`static int gitchat_probe(struct platform_device *pdev)   {       ...       db->addr_res = platform_get_resource(pdev, IORESOURCE_MEM, 0);       db->data_res = platform_get_resource(pdev, IORESOURCE_MEM, 1);       db->irq_res  = platform_get_resource(pdev, IORESOURCE_IRQ, 2);       ...   }   `

从代码中看到驱动是通过总线 API 接口 platform_get_resource 取得板级信息，这样驱动和设备之间就实现了高内聚、低耦合的设计，无论你设备怎么换，我驱动就可以岿然不动。

看到这里，可能有些喜欢探究本质的读者又要问了，设备向总线注册了板级信息，驱动也向总线注册了驱动模块，但总线是怎么做到驱动和设备匹配的呢？接下来就讲下设备和驱动是怎么通过总线进行“联姻”的。

总线里有很多匹配方式，比如：

`static int platform_match(struct device *dev, struct device_driver *drv)   {           struct platform_device *pdev = to_platform_device(dev);           struct platform_driver *pdrv = to_platform_driver(drv);              /* When driver_override is set, only bind to the matching driver */           if (pdev->driver_override)                    return !strcmp(pdev->driver_override, drv->name);              /* Attempt an OF style match first */           if (of_driver_match_device(dev, drv))                   return 1;              /* Then try ACPI style match */           if (acpi_driver_match_device(dev, drv))                   return 1;              /* Then try to match against the id table */           if (pdrv->id_table)                   return platform_match_id(pdrv->id_table, pdev) != NULL;              /* fall-back to driver name match */           return (strcmp(pdev->name, drv->name) == 0);   }   `

从上面可知 platform 总线下的设备和驱动是通过名字进行匹配的，先去匹配 platform_driver 中的 id_table 表中的各个名字与 platform_device->name 名字是否相同，如果相同则匹配。

## 设备驱动模型的改善

相信通过上面的学习，相信对于设备、驱动通过总线来匹配的模型已经有所了解。如果写代码的话应该是下面结构图所示：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

最底层是不同板子的板级文件代码，中间层是内核的总线，最上层是对应的驱动，现在描述板级的代码已经和驱动解耦了，这也是 Linux 设备驱动模型最早的实现机制，但随着时代的发展，就像是人类的贪婪促进了社会的进步一样，开发人员对这种模型有了更高的要求，虽然驱动和设备解耦了，但是天下设备千千万，每次设备的需求改动都要去修改 board-xxx.c 设备文件的话，这样下去，有太多的板级文件需要维护。完美的 Linux 怎么会允许这样的事情存在，于是乎，设备树（DTS）就登向了历史舞台，下一篇内容将探讨设备树的实现原理和用法。

# Linux 设备树（DTS）的深入理解

## 设备树的出现

上面说过设备树的出现是为了解决内核中大量的板级文件代码，通过 DTS 可以像应用程序里的 XML 语言一样很方便的对硬件信息进行配置。关于设备树的出现其实在 2005 年时候就已经在 PowerPC Linux 里出现了，由于 DTS 的方便性，慢慢地被广泛应用到 ARM、MIPS、X86 等架构上。为了理解设备树的出现的好处，先来看下在使用设备树之前是采用什么方式的。

关于硬件的描述信息之前一般放在一个个类似 arch/xxx/mach-xxx/board-xxx.c 的文件中，如：

`static struct resource gitchat_resource[] = {       {           .start = 0x20100000 ,           .end = 0x20100000 +1,           .flags = IORESOURCE_MEM            …           .start = IRQ_PF IRQ_PF 15 ,           .end = IRQ_PF IRQ_PF 15 ,           .flags = IORESOURCE_IRQ | IORESOURCE_IRQ_HIGHEDGE       }   };      static struct platform_device gitchat_device = {       .name name ="gitchat",       .id = 0,       .num_resources num_resources = ARRAY_SIZE(gitchat_resource),       .resource = gitchat_resource,   };      static struct platform_device *ip0x_devices[] __initdata ={       &gitchat_device,   };      static int __init ip0x_init(void)   {       platform_add_devices(ip0x_devices, ARRAY_SIZE(ip0x_devices));    }   `

一个很少的地址获取，我们就要写大量的类似代码，当年 Linus 看到内核里有大量的类似代码，很是生气并且在 Linux 邮件列表里发了份邮件，才有了现在的设备树概念，至于设备树的出现到底带来了哪些好处，先看一下设备树的文件：

`eth:eth@ 4,c00000 {       compatible ="csdn, gitchat";       reg =<           4 0x00c00000 0x2           4 0x00c00002 0x2       >;       interrupt-parent =<&gpio 2>;       interrupts=<14 IRQ_TYPE_LEVEL_LOW>;       …   };   `

从代码中可看到对于 GITCHAT 这个网卡驱动、一些寄存器、中断号和上一层 gpio 节点都很清晰的被描述。比上一图的代码优化了很多，也容易维护了很多。这样就形成了设备在脚本，驱动在 c 文件里的关系图：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

从图中可以看出 A、B、C 三个板子里都含有 GITCHAT 设备树文件，这样对于 GITCHAT 驱动写一份就可以在 A、B、C 三个板子里共用。从上幅图里不难看出，其实设备树的出现在软件模型上相对于之前并没有太大的改变，设备树的出现主要在设备维护上有了更上一层楼的提高，此外在内核编译上使内核更精简，镜像更小。

## 设备树的文件结构和剖析

设备树和设备树之间到底是什么关系，有着哪些依赖和联系，先看下设备树之间的关系图：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

除了设备树（DTS）外，还存有 dtsi 文件，就像代码里的头文件一样，是不同设备树共有的设备文件，这不难理解，但是值得注意的是如果 dts 和 dtsi 里都对某个属性进行定义的话，底层覆盖上层的属性定义。这样的好处是什么呢？假如你要做一块电路板，电路板里有很多模块是已经存在的，这样就可以直接像包含头文件一样把共性的 dtsi 文件包含进来，大大减少工作量，后期也可以对类似模块再次利用。

设备树文件的格式是 dts，包含的头文件格式是 dtsi，dts 文件是一种程序员可以看懂的格式，但是 Uboot 和 Linux 只能识别二进制文件，不能直接识别。所以就需要把 dts 文件编译成 dtb 文件。把 dts 编译成 dtb 文件的工具是 dtc，位于内核目录下 scripts/dtc，也可以手动安装：sudo apt-get install device-tree-compiler 工具。具体 dts 是如何转换成机器码并在内存里供 kernel 识别的，请看下图：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 设备树的应用

有了理论，在具体的工程里如何做设备树呢？这里介绍三大法宝：文档、脚本、代码。文档是对各种 node 的描述，位于内核 documentation/devicetree/bingdings/arm/ 下，脚本就是设备树 dts，代码就是你要写的设备代码，一般位于 arch/arm/ 下，以后在写设备的时候可以用这种方法，绝对的事半功倍。很多上层应用开发者没有做过内核开发的经验，对内核一直觉得很神秘，其实可以换一种思路来看内核，相信上层应用开发者最熟悉的就是各种 API，工作中可以说就是和 API 打交道，对于内核也可以想象是各种 API，只不过是内核态的 API。这里设备文件就是根据各种内核态的 API 来调用设备树里的板级信息：

- struct device_node *of_find_node_by_phandle(phandle handle);
    
- struct device_node *of_get_parent(const struct device_node_ *node);
    
- of_get_child_count()
    
- of_property_read_u32_array()
    
- of_property_read_u64()
    
- of_property_read_string()
    
- of_property_read_string_array()
    
- of_property_read_bool()
    

具体的用法这里不做进一步的解释，大家可以查询资料或者看官网解释。

这里对设备树做个总结，设备树可以总结为三大作用：一是平台标识，所谓平台标识就是板级识别，让内核知道当前使用的是哪个开发板，这里识别的方式是根据 root 节点下的 compatible 字段来匹配。二是运行时配置，就是在内核启动的时候 ramdisk 的配置，比如 bootargs 的配置，ramdisk 的起始和结束地址。三是设备信息集合，这也是最重要的信息，集合了各种设备控制器，接下来的实践课会对这一作用重点应用。这里有张图对大家理解设备树作用有一定的帮助：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

# Linux 设备和驱动的相遇

## 一个开发板

上面的最后我们讲到设备树的三大作用，其最后一个作用也是最重要的作用：设备信息集合。这一节结合设备信息集合的详细讲解来认识一下设备和驱动是如何绑定的。所谓设备信息集合，就是根据不同的外设寻找各自的外设信息，我们知道一个完整的开发板有 CPU 和各种控制器（如 I2C 控制器、SPI 控制器、DMA 控制器等），CPU 和控制器可以统称为 SOC，除此之外还有各种外设 IP，如 LCD、HDMI、SD、CAMERA 等，如下图：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

我们看到一个开发板有很多的设备，这些设备是如何一层一层展开的呢？设备和驱动又是如何绑定的呢？我们带着这些疑问进入本节的主题。

## 各级设备的展开

内核启动的时候是一层一层展开地去寻找设备，设备树之所以叫设备树也是因为设备在内核中的结构就像树一样，从根部一层一层的向外展开，为了更形象的理解来看一张图：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

大的圆圈中就是我们常说的 soc，里面包括 CPU 和各种控制器 A、B、I2C、SPI，soc 外面接了外设 E 和 F。IP 外设有具体的总线，如 I2C 总线、SPI 总线，对应的 I2C 设备和 SPI 设备就挂在各自的总线上，但是在 soc 内部只有系统总线，是没有具体总线的。

第一节中讲了总线、设备和驱动模型的原理，即任何驱动都是通过对应的总线和设备发生联系的，故虽然 soc 内部没有具体的总线，但是内核通过 platform 这条虚拟总线，把控制器一个一个找到，一样遵循了内核高内聚、低耦合的设计理念。下面我们按照 platform 设备、i2c 设备、spi 设备的顺序探究设备是如何一层一层展开的。

### 1.展开 platform 设备

上图中可以看到红色字体标注的 simple-bus，这些就是连接各类控制器的总线，在内核里即为 platform 总线，挂载的设备为 platform 设备。下面看下 platform 设备是如何展开的。

还记得上一节讲到在内核初始化的时候有一个叫做 init_machine() 的回调函数吗？如果你在板级文件里注册了这个函数，那么在系统启动的时候这个函数会被调用，如果没有定义，则会通过调用 of_platform_populate() 来展开挂在“simple-bus”下的设备，如图（分别位于 kernel/arch/arm/kernel/setup.c，kernel/drivers/of/platform.c）：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这样就把 simple-bus 下面的节点一个一个的展开为 platform 设备。

### 2.展开 i2c 设备

有经验的小伙伴知道在写 i2c 控制器的时候肯定会调用 i2c_register_adapter() 函数，该函数的实现如下（kernel/drivers/i2c/i2c-core.c）：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

注册函数的最后有一个函数 of_i2c_register_devices(adap)，实现如下：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

of_i2c_register_devices()函数中会遍历控制器下的节点，然后通过of_i2c_register_device()函数把 i2c 控制器下的设备注册进去。

### 3.展开 spi 设备

spi 设备的注册和 i2c 设备一样，在 spi 控制器下遍历 spi 节点下的设备，然后通过相应的注册函数进行注册，只是和 i2c 注册的 api 接口不一样，下面看一下具体的代码（kernel/drivers/spi/spi.c)：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

当通过 spi_register_master 注册 spi 控制器的时候会通过 of_register_spi_devices 来遍历 spi 总线下的设备，从而注册。这样就完成了 spi 设备的注册。

## 各级设备的展开

学到这里相信应该了解设备的硬件信息是从设备树里获取的，如寄存器地址、中断号、时钟等等。接下来我们一起看下这些信息在设备树里是怎么记录的，为下一节动手定制开发板做好准备。

### 1.reg 寄存器

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

我们先看设备树里的 soc 描述信息，红色标注的代表着寄存器地址用几个数据量来表述，绿色标注的代表着寄存器空间大小用几个数据量来表述。图中的含义是中断控制器的基地址是 0xfec00000，空间大小是 0x1000。如果 address-cells 的值是 2 的话表示需要两个数量级来表示基地址，比如寄存器是 64 位的话就需要两个数量级来表示，每个代表着 32 位的数。

### 2.ranges 取值范围

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

ranges 代表了 local 地址向 parent 地址的转换，如果 ranges 为空的话代表着与 cpu 是 1:1 的映射关系，如果没有 range 的话表示不是内存区域。

# 动手定制一个开发板案例

前面通过学习总线、设备、驱动模型知识后，知道了设备和驱动之间都是通过总线进行绑定而匹配的；然后通过设备树的深入探究，知道了设备树的出现大大增加了驱动的通用性；接着我们一起看了 Linux 的启动流程和设备在内核里一层一层的展开。

通过前面的课程学习，相信内核初学者对内核和驱动有了一个大概的感性认识，有内核经验的小伙伴相信也有了系统化的梳理。都说“实践是检验真理的唯一标准”，为了检验前面的理论性，更为了加深理解，这里我们手把手一起从设备树入手，模拟一个电路板，上面有中断控制器、GPIO 控制器、I2C 控制器、SPI 控制器、以太网控制器等，并根据这个电路板从头到尾构建一个 DTS 文件，使内核支持我们自己定制的开发板。

## 添加一个新的 SOC

我们假设 csdn 是一家芯片厂商，研发了一块叫做 gitchat 的芯片，对应的开发板叫做 evb。需要做什么工作才能定制一套开发板并且使 Linux 支持我们这块电路板呢？

首先我们在 arch/arm 目录下创建一个 mach-gitchat 的目录，然后创建 arch/arm/machgitchat/Kconfig，Makefile，common.c。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

其中，Kconfig 里做了一个叫 ARCH_GITCHAT 的假芯片，common.c 是对应的驱动，Makefile 里对这个驱动做了编译。我们来简单的看一下这个驱动

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

就像前几节课讲的那样，我们是通过 DT_MACHINE_START 来定义一个电路板的，里面有很多回调函数，这里简单的写了 init_late 和 dt_compat，其中 dt_compat 会根据设备树里的字段进行匹配，如果字段一致就说明匹配正确，这里的字段是“csdn, gitchat”。

## 添加对应的 DTS

现在已经在 Linux 里添加了我们定义的 soc，接下来需要添加 soc 对应的设备树，即具体的板级文件信息。

在 arch/arm/boot/dts 下创建 csdn-gitchat.dtsi 和 csdn-gitchat-evb.dts。csdn-gitchat.dtsi 是对芯片的描述，csdn-gitchat-evb.dts 是针对这个芯片的 evb 板子描述，其内容分别如下：

`/*    * DTS file for demo platform for Dec20,2017 CSDN/Gitchat course    *    * Copyright (c) 2017 xxx <xxx@gmail.com>    *    * Licensed under GPLv2.    */      / {       compatible = "csdn,gitchat";       #address-cells = <1>;       #size-cells = <1>;       interrupt-parent = <&intc>;          aliases {           serial0 = &uart0;       };          cpus {           #address-cells = <1>;           #size-cells = <0>;              cpu@0 {               compatible = "arm,cortex-a9";               device_type = "cpu";               reg = <0x0>;           };           cpu@1 {               compatible = "arm,cortex-a9";               device_type = "cpu";               reg = <0x1>;           };       };          axi@40000000 {           compatible = "simple-bus";           #address-cells = <1>;           #size-cells = <1>;           ranges = <0x40000000 0x40000000 0x80000000>;              l2-cache-controller@80040000 {               compatible = "arm,pl310-cache";               reg = <0x80040000 0x1000>;               interrupts = <59>;               arm,tag-latency = <1 1 1>;               arm,data-latency = <1 1 1>;               arm,filter-ranges = <0 0x40000000>;           };              intc: interrupt-controller@80020000 {               #interrupt-cells = <1>;               interrupt-controller;               compatible = "csdn,gitchat";               reg = <0x80020000 0x1000>;           };              peri-iobg@b0000000 {               compatible = "simple-bus";               #address-cells = <1>;               #size-cells = <1>;               ranges = <0x0 0xb0000000 0x180000>;                  ethernet@10000 {                   compatible = "davicom,dm9000";                   reg = <0x10000 0x2 0x10004 0x2>;                   interrupts = <7>;                   davicom,no-eeprom;               };                  timer@20000 {                   compatible = "csdn,gitchat-tick";                   reg = <0x20000 0x1000>;                   interrupts = <0>;                   clocks = <&clks 11>;               };                  clks: clock-controller@30000 {                   compatible = "csdn,gitchat-clkc";                   reg = <0x30000 0x1000>;                   #clock-cells = <1>;               };                  gpio0: goio@40000 {                   #gpio-cells = <2>;                   #interrupt-cells = <2>;                   compatible = "csdn,gitchat-gpio";                   reg = <0x40000 0x200>;                   interrupts = <43>;                   gpio-controller;                   interrupt-controller;               };                  uart0: uart@50000 {                   cell-index = <0>;                   compatible = "csdn,gitchat-uart";                   reg = <0x50000 0x1000>;                   interrupts = <17>;                   status = "disabled";               };                  spi0: spi@d0000 {                   cell-index = <0>;                   compatible = "csdn,gitchat-spi";                   reg = <0xd0000 0x10000>;                   interrupts = <15>;                   #address-cells = <1>;                   #size-cells = <0>;                   status = "okay";               };                  i2c0: i2c@e0000 {                   cell-index = <0>;                   compatible = "csdn,gitchat-i2c";                   reg = <0xe0000 0x10000>;                   interrupts = <24>;                   #address-cells = <1>;                   #size-cells = <0>;                   status = "okay";               };           };       };   };   `

`/dts-v1/;      /include/ "csdn-gitchat.dtsi"      / {       model = "Csdn Gitchat EVB Board";       compatible = "csdn,gitchat-evb", "csdn,gitchat";          memory {           device_type = "memory";           reg = <0x00000000 0x20000000>;       };          chosen {           bootargs = "console=ttyS0,115200 root=/dev/mtdblock5 rw rootfstype=ubifs";       };          axi@40000000 {           peri-iobg@b0000000 {               spi@d0000 {                   status = "disabled";               };               i2c@e0000 {                   pixcir_ts@5c {                       compatible = "pixcir,pixcir_tangoc";                       reg = <0x5c>;                       interrupt-parent = <&gpio0>;                       interrupts = <17 0>;                       attb-gpio = <&gpio0 22 0>;                       touchscreen-size-x = <1024>;                       touchscreen-size-y = <600>;                   };               };           };       };      };      &uart0 {       status = "okay";   };   `

由于篇幅限制，这里我们只写了部分代码。

## DTS 的编译

在 arch/arm/boot/dts/Makefile 里添加我们的板子：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

刚刚平台目录下我们已经对 MACH_GITCHAT 进行了定义，这里就会把设备树编译成 dtb 文件。我们也可以通过命令手动来编译生成 csdn-gitchat-evb.dtb 文件：

> $ dtc -I dts -O dtb csdn-gitchat-evb.dts -o csdn-gitchat-evb.dtb

至此，一个自己定制的开发板就已经搞定，这里虚拟了一个开发板，如果定制一个真实的电路板的话需要根据芯片手册把设备树里的信息完善起来。但是通过这个系列学习完全可以理解从零是如何定制一个开发板的。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "虚线阴影分割线")

![](http://mmbiz.qpic.cn/mmbiz_png/9sNwsXcN68pL55XIyzTrCHZTbIUdTibQcuzuCaYeGTXNMyn6ACmicUrpoDC0xZSap46XJ59sKysPg9Rg379f32cA/300?wx_fmt=png&wxfrom=19)

**人人极客社区**

工程师们自己的Linux底层技术社区，分享体系架构、内核、网络、安全和驱动。

316篇原创内容

公众号

5T技术资源大放送！包括但不限于：C/C++，Arm, Linux，Android，人工智能，单片机，树莓派，等等。在上面的【人人都是极客】公众号内回复「peter」，即可免费获取！！  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E) **记得点击****分享****、****赞****和****在看****，给我充点儿电吧**

阅读 4351

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/SeWfibBcBT0EibtIWVNvshnuWMN1AoJw3poFIsbpaIVyZibCCqwBUR21rcDfrQgoqYzaYNdS14IIXzvmzvibdDa5Rw/300?wx_fmt=png&wxfrom=18)

Linux内核之旅

27115

写留言

写留言

**留言**

暂无留言
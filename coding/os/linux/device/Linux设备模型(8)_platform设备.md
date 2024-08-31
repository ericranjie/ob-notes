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

作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2014-4-28 10:24 分类：[统一设备模型](http://www.wowotech.net/sort/device_model)

#### 1. 前言

在Linux设备模型的抽象中，存在着一类称作“Platform Device”的设备，内核是这样描述它们的（Documentation/driver-model/platform.txt）：

> Platform devices are devices that typically appear as autonomous entities in the system. This includes legacy port-based devices and host bridges to peripheral buses, and most controllers integrated into system-on-chip platforms.  What they usually have in common is direct addressing from a CPU bus.  Rarely, a platform_device will be connected through a segment of some other kind of bus; but its registers will still be directly addressable.

概括来说，Platform设备包括：基于端口的设备（已不推荐使用，保留下来只为兼容旧设备，legacy）；连接物理总线的桥设备；集成在SOC平台上面的控制器；连接在其它bus上的设备（很少见）。等等。

这些设备有一个基本的特征：可以通过CPU bus直接寻址（例如在嵌入式系统常见的“寄存器”）。因此，由于这个共性，内核在设备模型的基础上（device和device_driver），对这些设备进行了更进一步的封装，抽象出paltform bus、platform device和platform driver，以便驱动开发人员可以方便的开发这类设备的驱动。

可以说，paltform设备对Linux驱动工程师是非常重要的，因为我们编写的大多数设备驱动，都是为了驱动plaftom设备。本文我们就来看看Platform设备在内核中的实现。

#### 2. Platform模块的软件架构

内核中Platform设备有关的实现位于include/linux/platform_device.h和drivers/base/platform.c两个文件中，它的软件架构如下：

[![Platform设备软件架构](http://www.wowotech.net/content/uploadfile/201405/17339e2f323c0379fb16bdbea9aaf29720140507093251.gif "Platform设备软件架构")](http://www.wowotech.net/content/uploadfile/201405/8cf2e12b367205a9c1c7888e2dcb818520140507093250.gif)

由图片可知，Platform设备在内核中的实现主要包括三个部分：

Platform Bus，基于底层bus模块，抽象出一个虚拟的Platform bus，用于挂载Platform设备；  
Platform Device，基于底层device模块，抽象出Platform Device，用于表示Platform设备；  
Platform Driver，基于底层device_driver模块，抽象出Platform Driver，用于驱动Platform设备。

其中Platform Device和Platform Driver会会其它Driver提供封装好的API，具体可参考后面的描述。

#### 3. Platform模块向其它模块提供的API汇整

Platform提供的接口包括：Platform Device和Platform Driver两个数据结构，以及它们的操作函数。

##### 3.1 数据结构

1. 用于抽象Platform设备的数据结构----“struct platform_device”：

   1: /* include/linux/platform_device.h, line 22 */

   2: struct platform_device {

   3:         const char      *name;

   4:         int             id;

   5:         bool            id_auto; 

   6:         struct device   dev;

   7:         u32             num_resources;

   8:         struct resource *resource;

   9:  

  10:         const struct platform_device_id *id_entry;

  11:         

  12:         /* MFD cell pointer */

  13:         struct mfd_cell *mfd_cell;

  14:  

  15:         /* arch specific additions */

  16:         struct pdev_archdata    archdata;

  17: };

> 该结构的解释如下：
> 
> dev，真正的设备（Platform设备只是一个特殊的设备，因此其核心逻辑还是由底层的模块实现）。
> 
> name，设备的名称，和struct device结构中的init_name（"[Linux设备模型(5)_device和device driver](http://www.wowotech.net/linux_kenrel/device_and_driver.html)”）意义相同。实际上，该名称在设备注册时，会拷贝到dev.init_name中。
> 
> id，用于标识该设备的ID。  
> 在“[Linux设备模型(6)_Bus](http://www.wowotech.net/linux_kenrel/bus.html)”中有提过，内核允许存在多个名称相同的设备。而设备驱动的probe，依赖于名称，Linux采取的策略是：在bus的设备链表中查找device，和对应的device_driver比对name，如果相同，则查看该设备是否已经绑定了driver（查看其dev->driver指针是否为空），如果已绑定，则不会执行probe动作，如果没有绑定，则以该device的指针为参数，调用driver的probe接口。  
> 因此，在driver的probe接口中，通过判断设备的ID，可以知道此次驱动的设备是哪个。
> 
> id_auto，指示在注册设备时，是否自动赋予ID值（不需要人为指定啦，可以懒一点啦）。
> 
> num_resources、resource，该设备的资源描述，由struct resource（include/linux/ioport.h）结构抽象。  
> 在Linux中，系统资源包括I/O、Memory、Register、IRQ、DMA、Bus等多种类型。这些资源大多具有独占性，不允许多个设备同时使用，因此Linux内核提供了一些API，用于分配、管理这些资源。  
> 当某个设备需要使用某些资源时，只需利用struct resource组织这些资源（如名称、类型、起始、结束地址等），并保存在该设备的resource指针中即可。然后在设备probe时，设备需求会调用资源管理接口，分配、使用这些资源。而内核的资源管理逻辑，可以判断这些资源是否已被使用、是否可被使用等等。
> 
> id_entry，和内核模块相关的内容，暂不说明。
> 
> mfd_cell，和MFD设备相关的内容，暂不说明。
> 
> archdata，一个奇葩的存在！！它的目的是为了保存一些architecture相关的数据，去看看arch/arm/include/asm/device.h中struct pdev_archdata结构的定义，就知道这种放纵的设计有多么垃圾了。不管它了！！  

2. 用于抽象Platform设备驱动的数据结构----“struct platform_driver”：

   1: /* include/linux/platform_device.h, line 173 */

   2: struct platform_driver {

   3:         int (*probe)(struct platform_device *);

   4:         int (*remove)(struct platform_device *);

   5:         void (*shutdown)(struct platform_device *);

   6:         int (*suspend)(struct platform_device *, pm_message_t state);

   7:         int (*resume)(struct platform_device *);

   8:         struct device_driver driver;

   9:         const struct platform_device_id *id_table;

  10: };

> struct platform_driver结构和struct device_driver非常类似，无非就是提供probe、remove、suspend、resume等回调函数，这里不再细说。
> 
> 另外这里有一个id_table的指针，该指针和"[Linux设备模型(5)_device和device driver](http://www.wowotech.net/linux_kenrel/device_and_driver.html)”所描述的of_match_table、acpi_match_table的功能类似：提供其它方式的设备probe。  
> 我们在"[Linux设备模型(5)_device和device driver](http://www.wowotech.net/linux_kenrel/device_and_driver.html)”讲过，内核会在合适的时机检查device和device_driver的名字，如果匹配，则执行probe。其实除了名称之外，还有一些宽泛的匹配方式，例如这里提到的各种match table，具体原理就先不罗嗦了，徒添烦恼！就当没看见，呵呵。  

##### 3.2 Platform Device提供的API

Platform Device主要提供设备的分配、注册等接口，供其它driver使用，具体包括：

   1: /* include/linux/platform_device.h */

   2: extern int platform_device_register(struct platform_device *);

   3: extern void platform_device_unregister(struct platform_device *);

   4:  

   5: extern void arch_setup_pdev_archdata(struct platform_device *);

   6: extern struct resource *platform_get_resource(struct platform_device *,

   7:                                               unsigned int, unsigned int);

   8: extern int platform_get_irq(struct platform_device *, unsigned int);

   9: extern struct resource *platform_get_resource_byname(struct platform_device *,

  10:                                                      unsigned int,

  11:                                                      const char *);

  12: extern int platform_get_irq_byname(struct platform_device *, const char *);

  13: extern int platform_add_devices(struct platform_device **, int);

  14:  

  15: extern struct platform_device *platform_device_register_full(

  16:                 const struct platform_device_info *pdevinfo);

  17:  

  18: static inline struct platform_device *platform_device_register_resndata(

  19:                 struct device *parent, const char *name, int id,

  20:                 const struct resource *res, unsigned int num,

  21:                 const void *data, size_t size)

  22:  

  23: static inline struct platform_device *platform_device_register_simple(

  24:                 const char *name, int id,

  25:                 const struct resource *res, unsigned int num)

  26:  

  27: static inline struct platform_device *platform_device_register_data(

  28:                 struct device *parent, const char *name, int id,

  29:                 const void *data, size_t size)

  30:  

  31: extern struct platform_device *platform_device_alloc(const char *name, int id);

  32: extern int platform_device_add_resources(struct platform_device *pdev,

  33:                                          const struct resource *res,

  34:                                          unsigned int num);

  35: extern int platform_device_add_data(struct platform_device *pdev,

  36:                                     const void *data, size_t size);

  37: extern int platform_device_add(struct platform_device *pdev);

  38: extern void platform_device_del(struct platform_device *pdev);

  39: extern void platform_device_put(struct platform_device *pdev);

> platform_device_register、platform_device_unregister，Platform设备的注册/注销接口，和底层的device_register等接口类似。
> 
> arch_setup_pdev_archdata，设置platform_device变量中的archdata指针。
> 
> platform_get_resource、platform_get_irq、platform_get_resource_byname、platform_get_irq_byname，通过这些接口，可以获取platform_device变量中的resource信息，以及直接获取IRQ的number等等。
> 
> platform_device_register_full、platform_device_register_resndata、platform_device_register_simple、platform_device_register_data，其它形式的设备注册。调用者只需要提供一些必要的信息，如name、ID、resource等，Platform模块就会自动分配一个struct platform_device变量，填充内容后，注册到内核中。
> 
> platform_device_alloc，以name和id为参数，动态分配一个struct platform_device变量。
> 
> platform_device_add_resources，向platform device中增加资源描述。
> 
> platform_device_add_data，向platform device中添加自定义的数据（保存在pdev->dev.platform_data指针中）。
> 
> platform_device_add、platform_device_del、platform_device_put，其它操作接口。

##### 3.3 Platform Driver提供的API

Platform Driver提供struct platform_driver的分配、注册等功能，具体如下：

   1: /* include/linux/platform_device.h */

   2:  

   3: extern int platform_driver_register(struct platform_driver *);

   4: extern void platform_driver_unregister(struct platform_driver *);

   5:  

   6: /* non-hotpluggable platform devices may use this so that probe() and

   7:  * its support may live in __init sections, conserving runtime memory.

   8:  */

   9: extern int platform_driver_probe(struct platform_driver *driver,

  10:                 int (*probe)(struct platform_device *));

  11:  

  12: static inline void *platform_get_drvdata(const struct platform_device *pdev)

  13:  

  14: static inline void platform_set_drvdata(struct platform_device *pdev,

  15:                                         void *data)

> platform_driver_registe、platform_driver_unregister，platform driver的注册、注销接口。
> 
> platform_driver_probe，主动执行probe动作。
> 
> platform_set_drvdata、platform_get_drvdata，设置或者获取driver保存在device变量中的私有数据。

##### 3.4 懒人API

又是注册platform device，又是注册platform driver，看着挺啰嗦的。不过内核想到了这点，所以提供一个懒人API，可以同时注册platform driver，并分配一个platform device：

   1: extern struct platform_device *platform_create_bundle(

   2:         struct platform_driver *driver, int (*probe)(struct platform_device *),

   3:         struct resource *res, unsigned int n_res,

   4:         const void *data, size_t size);

> 只要提供一个platform_driver（要把driver的probe接口显式的传入），并告知该设备占用的资源信息，platform模块就会帮忙分配资源，并执行probe操作。对于那些不需要热拔插的设备来说，这种方式是最省事的了。  

##### 3.5 Early platform device/driver

内核启动时，要完成一定的初始化操作之后，才会处理device和driver的注册及probe，因此在这之前，常规的platform设备是无法使用的。但是在Linux中，有些设备需要尽早使用（如在启动过程中充当console输出的serial设备），所以platform模块提供了一种称作Early platform device/driver的机制，允许驱动开发人员，在开发驱动时，向内核注册可在内核早期启动过程中使用的driver。这些机制提供了如下接口：

   1: extern int early_platform_driver_register(struct early_platform_driver *epdrv,

   2:                                           char *buf);

   3: extern void early_platform_add_devices(struct platform_device **devs, int num);

   4:  

   5: static inline int is_early_platform_device(struct platform_device *pdev)

   6: {

   7:         return !pdev->dev.driver;

   8: }

   9:  

  10: extern void early_platform_driver_register_all(char *class_str);

  11: extern int early_platform_driver_probe(char *class_str,

  12:                                        int nr_probe, int user_only);

  13: extern void early_platform_cleanup(void);

> early_platform_driver_register，注册一个用于Early device的driver。
> 
> early_platform_add_devices，添加一个Early device。
> 
> is_early_platform_device，判断指定的device是否是Early device。
> 
> early_platform_driver_register_all，将指定class的所有driver注册为Early device driver。
> 
> early_platform_driver_probe，probe指定class的Early device。
> 
> early_platform_cleanup，清除所有的Early device/driver。

#### 4. Platform模块的内部动作解析

##### 4.1 Platform模块的初始化

Platform模块的初始化是由drivers/base/platform.c中platform_bus_init接口完成的，该接口的实现和动作如下：

   1: int __init platform_bus_init(void)

   2: {

   3:         int error;

   4:  

   5:         early_platform_cleanup();

   6:  

   7:         error = device_register(&platform_bus);

   8:         if (error)

   9:                 return error;

  10:         error =  bus_register(&platform_bus_type);

  11:         if (error)

  12:                 device_unregister(&platform_bus);

  13:         return error;

  14: }

> early_platform_cleanup，清除所有和Early device/driver相关的代码。因为执行到这里的时候，证明系统已经完成了Early阶段的启动，转而进行正常的设备初始化、启动操作，所以不再需要Early Platform相关的东西。
> 
> device_register，注册一个名称为platform_bus的设备，该设备的定义非常简单，只包含init_name（为“platform”）。该步骤会在sysfs中创建“/sys/devices/platform/”目录，所有的Platform设备，都会包含在此目录下。
> 
> bus_register，注册一个名称为platform_bus_type的bus，该bus的定义如下：  
> struct bus_type platform_bus_type = {  
>         .name           = "platform",  
>         .dev_attrs      = platform_dev_attrs,  
>         .match          = platform_match,  
>         .uevent         = platform_uevent,  
>         .pm             = &platform_dev_pm_ops,  
> };  
> 该步骤会在sysfs中创建“/sys/bus/platform/”目录，同时，结合“[Linux设备模型(6)_Bus](http://www.wowotech.net/linux_kenrel/bus.html)”的描述，会在“/sys/bus/platform/”目录下，创建uevent attribute（/sys/bus/platform/uevent）、devices目录、drivers目录、drivers_probe和drivers_autoprobe两个attribute（/sys/bus/platform/drivers_probe和/sys/bus/platform/drivers_autoprobe）。

##### 4.2 platform device和platform driver的注册

结合第3章的描述，platform device和platform driver的注册，由platform_device_add和platform_driver_register两个接口实际实现。其内部动作分别如下。

platform_device_add的内部动作：

> 如果该设备没有指定父设备，将其父设备设置为platform_bus，即“/sys/devices/platform/”所代表的设备，这时该设备的sysfs目录即为“/sys/devices/platform/xxx_device”。
> 
> 将该设备的bus指定为platform_bus_type（pdev->dev.bus = &platform_bus_type）。
> 
> 根据设备的ID，修改或者设置设备的名称。对于多个同名的设备，可以使用ID区分，在这里将实际名称修改为“name.id”的形式。
> 
> 调用resource模块的insert_resource接口，将该设备需要使用的resource统一管理起来（我们知道，在这之前，只是声明了本设备需要使用哪些resource，但resource模块并不知情，也就无从管理，因此需要告知）。
> 
> 调用device_add接口，将内嵌的struct device变量添加到内核中。

platform_driver_register的内部动作：

> 将该driver的bus指定为platform_bus_type（drv->driver.bus = &platform_bus_type）。
> 
> 如果该platform driver提供了probe、remove、shutdown等回调函数，将该它内嵌的struct driver变量的probe、remove、shutdown等指针，设置为platform模块提供函数，包括platform_drv_probe、platform_drv_remove和platform_drv_shutdown。因为probe等动作会从struct driver变量开始，经过platform_drv_xxx等接口的转接，就可以到达platform diver自身的回调函数中。
> 
> 调用driver_register接口，将内嵌的struct driver变量添加到内核中。

##### 4.3 platform设备的probe

我们在“[Linux设备模型(6)_Bus](http://www.wowotech.net/linux_kenrel/bus.html)”中讲过，设备的probe，都发生在向指定的bus添加device或者device_driver时，由bus模块的bus_probe_device，或者device_driver模块driver_attach接口触发。这里就不再详细描述了。

_原创文章，转发请注明出处。蜗窝科技_，[www.wowotech.net](http://www.wowotech.net/linux_kenrel/platform_device.html)。

标签: [Linux](http://www.wowotech.net/tag/Linux) [Kernel](http://www.wowotech.net/tag/Kernel) [设备模型](http://www.wowotech.net/tag/%E8%AE%BE%E5%A4%87%E6%A8%A1%E5%9E%8B) [platform设备](http://www.wowotech.net/tag/platform%E8%AE%BE%E5%A4%87)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Process Creation（二）](http://www.wowotech.net/process_management/process-creation-2.html) | [Linux设备模型(7)_Class](http://www.wowotech.net/device_model/class.html)»

**评论：**

**独寻**  
2017-04-22 10:23

在“Linux设备模型(6)_Bus”中有提过，内核允许存在多个名称相同的设备。而设备驱动的probe，依赖于名称，Linux采取的策略是：在bus的设备链表....  
======  
查看bus_type没有看到指向保存设备的链表？

[回复](http://www.wowotech.net/device_model/platform_device.html#comment-5479)

**[wowo](http://www.wowotech.net/)**  
2017-04-24 14:58

@独寻：在“struct subsys_private”指针中。

[回复](http://www.wowotech.net/device_model/platform_device.html#comment-5481)

**[linuxer](http://www.wowotech.net/)**  
2015-03-26 23:03

既然cdev和device是独立的，那cdev里面为啥要包含kobject，kobject不是只服务于统一设备模型吗？  
----------------------------------------------------------------  
kobject这个基类从来都不是只服务于统一设备模型，当然，在设备模型中用的比较多，内核中很多地方也都使用了kobject这个基类，有兴趣的话可以自己查查内核代码。  
  
至于洋葱问题，那cdev和device是不是属于两个不同的洋葱？  
--------------------------------------------------------  
对于一个字符设备，cdev和device位于一个洋葱的不同层次，cdev（和文件系统比较近）算是洋葱的最外层，和用户接口，而device（属于设备模型）属于比较内层

[回复](http://www.wowotech.net/device_model/platform_device.html#comment-1427)

**callme_friend**  
2015-04-09 11:31

@linuxer：本想完整看懂tty驱动再回复，但遇到问题多多。  
突然有个建议提给博主：  
   能否仿照侯捷的《深入浅出MFC》来阐述linux内核？  
   去掉旁支，留下主干，然后用非常精简的一个示例代码展示内核；对于理解内核来说，这应该是最容易的。

[回复](http://www.wowotech.net/device_model/platform_device.html#comment-1483)

**[linuxer](http://www.wowotech.net/)**  
2015-04-10 04:59

@callme_friend：多谢你的建议，作为一个linuxer，我其实是没有看过深入浅出MFC这本书的，当然知道它是一本好书。如果你确实需要，那我就写一份tty driver的文档，尽量使用你说的方法，看看是否可以达到良好的效果。

[回复](http://www.wowotech.net/device_model/platform_device.html#comment-1488)

**[callme_friend](http://www.wowotech.net/)**  
2015-04-12 22:07

@linuxer：要是有tty分析文档，那真是太感谢啦！  
我又阅读了相关子系统，问题多多，有没有专门提问的板块？

[回复](http://www.wowotech.net/device_model/platform_device.html#comment-1495)

**hony**  
2017-01-14 17:22

@linuxer：这里节选了一段kobject设计意图的一段话：“The "kobject" structure first made its appearance in the 2.5.45 development kernel. It was initially meant as a simple way of unifying kernel code which manages reference counted objects. The kobject has since encountered a bit of "mission creep," however; it is now the glue that holds much of the device model and its sysfs interface together. ”——可以解释为什么cdev虽然不属于设备模型但是也用到了它。  
  
而且仅仅用到了kobject的两个功能：  
1、Reference counts  
2、name  
  
另外一段关于cdev的节选：“Often, much of the initialization of a kobject is handled by the layer that manages the containing kset. Thus, to get back to our old example, a char driver might create a struct cdev, but it need not worry about setting any of the fields in the embedded kobject - except for the name. Everything else is handled by the char device layer.”——这段话说明了kobject的功能和设备模型并没有什么直接关系（甚至都可以不用使用sysfs的功能）。  
  
因此cdev和device本质是两种东西,只是它们都使用了kobject仅此而已。

[回复](http://www.wowotech.net/device_model/platform_device.html#comment-5146)

**[wowo](http://www.wowotech.net/)**  
2017-01-16 09:11

@hony：分析的很好，赞！～

[回复](http://www.wowotech.net/device_model/platform_device.html#comment-5154)

**callme_friend**  
2015-03-24 17:30

博主，我现在还真遇到一个of_match_table驱动了：  
static struct platform_driver cdns_uart_platform_driver = {  
    .probe   = cdns_uart_probe,  
    .remove  = cdns_uart_remove,  
    .driver  = {  
        .name = CDNS_UART_NAME,  
        .of_match_table = cdns_uart_of_match,  
        .pm = &cdns_uart_dev_pm_ops,  
        },  
};  
在这个驱动中：  
     name=CDNS_UART_NAME="xuartps";  
     of_match_table = cdns_uart_of_match="xlnx,xuartps"  
而在dts文件中是这样的：  
     uart0: serial@e0000000 {  
    compatible = "xlnx,xuartps", "cdns,uart-r1p8";  
        ...  
我的问题是：这里在匹配设备和驱动的时候，是不是使用的是of_match_table 里的数据和DTB里的compatible 数据进行匹配？  
一般情况下，是匹配name？

[回复](http://www.wowotech.net/device_model/platform_device.html#comment-1398)

**[linuxer](http://www.wowotech.net/)**  
2015-03-24 18:30

@callme_friend：总线上的device和driver进行匹配的时候会调用bus的match函数，对于platform bus而言就是platform_match：  
  
static int platform_match(struct device *dev, struct device_driver *drv)  
{  
    struct platform_device *pdev = to_platform_device(dev);  
    struct platform_driver *pdrv = to_platform_driver(drv);  
  
    /* Attempt an OF style match first */  
    if (of_driver_match_device(dev, drv))  
        return 1;  
  
    /* Then try ACPI style match */  
    if (acpi_driver_match_device(dev, drv))  
        return 1;  
  
    /* Then try to match against the id table */  
    if (pdrv->id_table)  
        return platform_match_id(pdrv->id_table, pdev) != NULL;  
  
    /* fall-back to driver name match */  
    return (strcmp(pdev->name, drv->name) == 0);  
}  
  
代码注释已经非常的清楚了，我就不赘述了

[回复](http://www.wowotech.net/device_model/platform_device.html#comment-1400)

**callme_friend**  
2015-03-24 21:46

@linuxer：嗯，是很清楚，匹配时有先后次序，谢谢前辈。  
   现拜读前辈数篇博文，思路慢慢清晰；手头刚好有个新开发板，里面有官方给的串口驱动。想看看驱动，再自己写一个类似驱动出来；本以为有了驱动模型概念、bus概念（platformbus），做起来应该不来，但是看了一个下午，仍感到知识点众多。还在啃代码中  
    像tty串口驱动，官方初始化给的是两步：  
        // Register the cdns_uart driver with the serial core  
    uart_register_driver(&cdns_uart_uart_driver);  
    /* Register the platform driver */  
    platform_driver_register(&cdns_uart_platform_driver);  
    按照总线、设备、驱动的概念，不是只需要注册驱动即platform_driver_register()就可以了吗，为何还要uart_register_driver()？  
    uart_register_driver()注册的驱动跟总线驱动模型是什么层次关系？

[回复](http://www.wowotech.net/device_model/platform_device.html#comment-1404)

**[linuxer](http://www.wowotech.net/)**  
2015-03-25 12:01

@callme_friend：虽然代码是那样的，但是我的观点是：  
1、串口driver的初始化代码应该只有注册platform driver的内容  
2、在platform driver的probe函数中调用 uart_register_driver接口，向serial core注册UART driver

[回复](http://www.wowotech.net/device_model/platform_device.html#comment-1412)

**callme_friend**  
2015-03-25 18:22

@linuxer：现在那个代码中是驱动模块初始化时候是先uart_register_driver()；然后再platform_driver_register()；最后在驱动probe()函数中添加uart_add_one_port()。  
    tty core驱动跟设备驱动之间有什么联系和区别？

[回复](http://www.wowotech.net/device_model/platform_device.html#comment-1414)

**callme_friend**  
2015-03-25 21:34

@linuxer：看完关于bus、device、driver的博文，再看如下两个数据结构，有点晕：  
struct cdev {  
    struct kobject kobj;  
    struct module *owner;  
    const struct file_operations *ops;  
    struct list_head list;  
    dev_t dev;  
    unsigned int count;  
};  
struct device {  
    struct device        *parent;  
    struct device_private    *p;  
    struct kobject kobj;  
    const char        *init_name;  
    const struct device_type *type;  
    ... ...  
    struct bus_type    *bus;          
    struct device_driver *driver;  
    void        *platform_data;      
    void        *driver_data;      
    ... ...  
    }  
    cdev和device结构体中都直接包含kobject基类，如何理解呢？  
    能这样理解吗：内核从两个角度管理设备驱动；对于物理硬件的抽象使用的是总线、设备、驱动模型，而对于具体设备的功能，又分字符设备、块设备、网络设备等。我也不知道怎么表达了，还是想不通。

[回复](http://www.wowotech.net/device_model/platform_device.html#comment-1415)

**[wowo](http://www.wowotech.net/)**  
2015-03-25 22:08

@callme_friend：您可以参考下面的评论，大家对这个问题有讨论过的。

[回复](http://www.wowotech.net/device_model/platform_device.html#comment-1416)

**[linuxer](http://www.wowotech.net/)**  
2015-03-26 09:40

@callme_friend：使用USB串口可能会更好的解释你提出的问题，任何的设备都是两个方面：  
1、从设备拓扑的角度看  
2、从功能层面看  
“USB串口”中的串口描述其设备拓扑，而串口描述其功能。无论是连接到PCI总线，还是连接到platform这样的伪总线上的串口都是串口，都有共同的特点，这些内容都在串口这个软件层次处理。同样的道理，无论是是串口还是普通中断，还是网络终端，都有tty的属性，都要处理break信号、发送延迟等tty设备需要处理的内容，这些共同的内容当然不适合在各自的驱动中处理，因此放到tty core处理。一言以蔽之，软件就像洋葱，是有层次的，是需要抽象的。对应你说的这个场景，软件层次是：  
－－－－－－－－－－－－－－－－  
char device driver layer  
－－－－－－－－－－－－－－－－  
tty device driver layer  
－－－－－－－－－－－－－－－－  
serial device driver layer  
－－－－－－－－－－－－－－－  
platform device driver layer  
  
对于串口驱动的初始化，我认为最佳的方式是：  
1、设备树建立串口设备对应的platform device  
2、串口驱动调用platform_driver_register接口注册该串口的platform driver  
3、设备模型会调用probe函数进行该串口设备的初始化。  
4、probe函数中处理serial device driver layer的内容。例如你说的uart_register_driver或者uart_add_one_port什么的  
5、在serial的初始化过程中，调用tty layer的接口进行该层次的初始化。

[回复](http://www.wowotech.net/device_model/platform_device.html#comment-1417)

**callme_friend**  
2015-03-26 21:22

@linuxer：既然cdev和device是独立的，那cdev里面为啥要包含kobject，kobject不是只服务于统一设备模型吗？  
至于洋葱问题，那cdev和device是不是属于两个不同的洋葱？

**[puppypyb](http://www.wowotech.net/)**  
2014-12-03 16:04

这样理解很OK  
谢谢两位

[回复](http://www.wowotech.net/device_model/platform_device.html#comment-800)

**[puppypyb](http://www.wowotech.net/)**  
2014-12-03 11:13

有个问题，之前一直没有考虑到。  
我们写传统的字符设备驱动的时候，似乎并没有将驱动挂载到任何bus上，只有主、次设备号什么的。  
这些字符设备驱动也是属于设备模型范畴吗？

[回复](http://www.wowotech.net/device_model/platform_device.html#comment-793)

**[linuxer](http://www.wowotech.net/)**  
2014-12-03 12:14

@puppypyb：普通的字符设备也是需要设备模型的，例如一个lcd驱动，你会定义一个plaform driver，并注册到系统。通过device tree，将lcd的platform device加入系统。当匹配的时候，调用platform driver中的lcd probe函数，而在这个函数中会分配cdev并加入到系统。  
  
cdev和设备模型不冲突啊，还是我没有明白你的问题？

[回复](http://www.wowotech.net/device_model/platform_device.html#comment-794)

**[callme_friend](http://www.wowotech.net/)**  
2015-05-06 09:53

@linuxer：我现在是这样理解cdev和设备模型：  
按照与app最接近的次序，程序调用首先应该是经过vfs；然后由vfs分为cdev、块设备或fifo设备等。应该说从始至终，只有“文件设备”、字符设备、块设备等，没有统一设备模型。设备模型只是为以上设备服务而已。  
你怎么看？

[回复](http://www.wowotech.net/device_model/platform_device.html#comment-1724)

**[linuxer](http://www.wowotech.net/)**  
2015-05-07 00:04

@callme_friend：我怎么感觉我是元芳呢，呵呵  
  
在没有统一设备模型的时代，设备文件就存在了，并且也是unix操作系统中，一切皆文件中的重要一环。  
那么美妙的接口设计，当然要配美妙的内部设计。统一设备模型让系统中的所有设备有组织、有层次、有内涵。  
如果说：设备模型只是为以上设备服务而已，这多少有些藐视我”大“统一模型的功能，毕竟如果是这样，kernel为何要增加统一设备模型呢？毕竟“文件设备”一直都在kernel中愉快的工作着，即便在没有统一设备模型的日子里。  
  
本身，统一设备模型是为了热拔插、开关机、电源管理等复杂的系统动作而设计的，当然，由于sys文件系统开放了属性文件，实际上也担负了部分原来“文件设备”的接口功能。

[回复](http://www.wowotech.net/device_model/platform_device.html#comment-1729)

**[callme_friend](http://www.wowotech.net/)**  
2015-05-07 08:45

@linuxer：还没有细看这部分。  
形而上学的通过“热拔插、开关机、电源管理”这些字眼猜测：这些东西都是跟总线联系紧密的东西。有了总线，自然而然的就会有次序的插拔设备、电源管理。  
但是linux的核心仍然是一切皆文件；如果哪天硬件拓扑不再是总线树形，而是一种截然不同的形式，那统一设备模型会被剔除内核，而vfs相关的东西仍然要保留在内核。  
一点猜测，欢迎大牛拍砖。

[回复](http://www.wowotech.net/device_model/platform_device.html#comment-1731)

**[wowo](http://www.wowotech.net/)**  
2015-05-07 09:49

@callme_friend：总结的很对啊，这些VFS/CDEV和设备模型，不是一个层面的事情：前者用于提供人机交互的API，是面向用户的；后者倾向于抽象、管理硬件设备，是面向设备的。不要混在一起，就容易理解了。  
不过呢，“那统一设备模型会被剔除内核”这句话不太准确，不会剔除的，可能会换一种形式。

[回复](http://www.wowotech.net/device_model/platform_device.html#comment-1733)

**[callme_friend](http://www.wowotech.net/)**  
2015-05-07 16:54

@wowo：是的，换种形势更准确。

**[linuxer](http://www.wowotech.net/)**  
2014-12-03 12:21

@puppypyb：驱动的逻辑可以说主要包括两个方面：  
1、对AP提供接口  
2、和HW block交互  
  
基本上cdev是属于向AP提供接口而设立的一个数据结构，并不是统一设备模型的一部分。因为在linux中“一切皆文件”，设备也是文件，因此cdev数据结构更多的是属于linux虚拟文件子系统而不是linux设备模型。

[回复](http://www.wowotech.net/device_model/platform_device.html#comment-795)

**[puppypyb](http://www.wowotech.net/)**  
2014-12-03 13:50

@linuxer：根据你的描述，是不是可以如下理解：  
平常我们喜闻乐见的字符设备驱动，其实是不属于linux设备模型范围的。所以我们注册的cdev可以在/dev目录下找到对应的设备节点，但/sys目录下却没有任何相关信息。  
当然，我们也可以将cdev驱动加入到设备模型（例如通过platform），这样cdev也纳入到sysfs进行统一管理。

[回复](http://www.wowotech.net/device_model/platform_device.html#comment-797)

**[linuxer](http://www.wowotech.net/)**  
2014-12-03 15:18

@puppypyb：“当然，我们也可以将cdev驱动加入到设备模型（例如通过platform），这样cdev也纳入到sysfs进行统一管理。”  
－－－－－－－－－－－－－－－－－－－－－－－  
上面这句话不对，感觉怪怪的，cdev无法加入到设备模型（加入设备模型的总是device或者driver、bus什么的），cdev仅仅说明你的driver的接口形态而以。  
  
如果你的driver想要提供传统字符设备的节点（/dev目录下），那么一定需要定义cdev这样的数据结构，并通过cdev_add（或者旧的register_chrdev）接口函数，向字符设备vfs层注册这个cdev。注意：这里虽然有dev，但是和设备模型无关，如果你不想提供传统的字符设备的节点，那么其实根本不需要cdev，统一设备模型的sys fs也提供了用户空间的接口。  
  
cdev不需要挂入任何总线，cdev需要向文件系统 layer注册。

[回复](http://www.wowotech.net/device_model/platform_device.html#comment-798)

**wowo**  
2014-12-03 15:52

@linuxer：这样理解会好点：字符设备里面的“设备”，和设备模型里面的“设备”，不是一个概念。字符设备或块设备，是在描述设备的的物理特性。设备模型是在描述设备的拓扑结构。所以这二者，没啥关系，谁离了谁都没有问题。

[回复](http://www.wowotech.net/device_model/platform_device.html#comment-799)

**[puppypyb](http://www.wowotech.net/)**  
2014-12-03 16:05

@wowo：谢谢两位！

[回复](http://www.wowotech.net/device_model/platform_device.html#comment-801)

**[Physh](http://www.wowotech.net/)**  
2015-01-15 14:38

@wowo：同意这个说法。有时候我喜欢注册虚拟的字符设备文件，以这个虚拟设备上的 read/write/ioctl 等接口与用户交互，不过虚拟字符设备并不描述任何物理特性就是了

[回复](http://www.wowotech.net/device_model/platform_device.html#comment-1097)

**[wowo](http://www.wowotech.net/)**  
2015-01-15 23:23

@Physh：这是个好习惯，一看就是个linux老手啊，呵呵。

**[callme_friend](http://www.wowotech.net/)**  
2015-05-07 08:46

@wowo：wowo如何看我上面的想法呢？

[回复](http://www.wowotech.net/device_model/platform_device.html#comment-1732)

**[VPS运维](http://www.sanduoyun.com/)**  
2014-05-02 15:07

博主你好，很欣赏你的网站，希望能跟你做个友情连接！

[回复](http://www.wowotech.net/device_model/platform_device.html#comment-112)

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
    
    - [CFS任务放置代码详解](http://www.wowotech.net/process_management/task_placement_detail.html)
    - [总结Linux kernel开发中0的巧妙使用](http://www.wowotech.net/201.html)
    - [Linux 2.5.43版本的RCU实现（废弃）](http://www.wowotech.net/kernel_synchronization/Linux-2-5-43-RCU.html)
    - [X-019-KERNEL-串口驱动开发之数据收发](http://www.wowotech.net/x_project/serial_driver_porting_4.html)
    - [tty驱动分析](http://www.wowotech.net/tty_framework/435.html)
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
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

作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2014-4-15 19:21 分类：[统一设备模型](http://www.wowotech.net/sort/device_model)

#### 1. 概述

在Linux设备模型中，Bus（总线）是一类特殊的设备，它是连接处理器和其它设备之间的通道（channel）。为了方便设备模型的实现，内核规定，系统中的每个设备都要连接在一个Bus上，这个Bus可以是一个内部Bus、虚拟Bus或者Platform Bus。

内核通过struct bus_type结构，抽象Bus，它是在include/linux/device.h中定义的。本文会围绕该结构，描述Linux内核中Bus的功能，以及相关的实现逻辑。最后，会简单的介绍一些标准的Bus（如Platform），介绍它们的用途、它们的使用场景。

#### 2. 功能说明

按照老传统，描述功能前，先介绍一下该模块的一些核心数据结构，对bus模块而言，核心数据结构就是struct bus_type，另外，还有一个sub system相关的结构，会一并说明。

##### 2.1 struct bus_type

 1: /* inlcude/linux/device.h, line 93 */

 2: struct bus_type {

 3:     const char *name;

 4:     const char *dev_name;

 5:     struct device       *dev_root;

 6:     struct bus_attribute    *bus_attrs;

 7:     struct device_attribute *dev_attrs;

 8:     struct driver_attribute *drv_attrs;

 9:  

 10:    int (*match)(struct device *dev, struct device_driver *drv);

 11:    int (*uevent)(struct device *dev, struct kobj_uevent_env *env);

 12:    int (*probe)(struct device *dev);

 13:    int (*remove)(struct device *dev);

 14:    void (*shutdown)(struct device *dev);

 15:  

 16:    int (*suspend)(struct device *dev, pm_message_t state);

 17:    int (*resume)(struct device *dev);

 18:  

 19:    const struct dev_pm_ops *pm;

 20:  

 21:    struct iommu_ops *iommu_ops;

 22:  

 23:    struct subsys_private *p;

 24:    struct lock_class_key lock_key;

 25: };

> name，该bus的名称，会在sysfs中以目录的形式存在，如platform bus在sysfs中表现为"/sys/bus/platform”。
> 
> dev_name，该名称和"[Linux设备模型(5)_device和device driver](http://www.wowotech.net/linux_kenrel/device_and_driver.html)”所讲述的struct device结构中的init_name有关。对有些设备而言（例如批量化的USB设备），设计者根本就懒得为它起名字的，而内核也支持这种懒惰，允许将设备的名字留空。这样当设备注册到内核后，设备模型的核心逻辑就会用"bus->dev_name+device ID”的形式，为这样的设备生成一个名称。
> 
> bus_attrs、dev_attrs、drv_attrs，一些默认的attribute，可以在bus、device或者device_driver添加到内核时，自动为它们添加相应的attribute。
> 
> dev_root，根据内核的注释，dev_root设备为bus的默认父设备（Default device to use as the parent），但在内核实际实现中，只和一个叫sub system的功能有关，随后会介绍。
> 
> match，一个由具体的bus driver实现的回调函数。当任何属于该Bus的device或者device_driver添加到内核时，内核都会调用该接口，如果新加的device或device_driver匹配上了自己的另一半的话，该接口要返回非零值，此时Bus模块的核心逻辑就会执行后续的处理。
> 
> uevent，一个由具体的bus driver实现的回调函数。当任何属于该Bus的device，发生添加、移除或者其它动作时，Bus模块的核心逻辑就会调用该接口，以便bus driver能够修改环境变量。
> 
> probe、remove，这两个回调函数，和[device_driver](http://www.wowotech.net/linux_kenrel/device_and_driver.html)中的非常类似，但它们的存在是非常有意义的。可以想象一下，如果需要probe（其实就是初始化）指定的device话，需要保证该device所在的bus是被初始化过、确保能正确工作的。这就要就在执行device_driver的probe前，先执行它的bus的probe。remove的过程相反。  
> 注1：并不是所有的bus都需要probe和remove接口的，因为对有些bus来说（例如platform bus），它本身就是一个虚拟的总线，无所谓初始化，直接就能使用，因此这些bus的driver就可以将这两个回调函数留空。
> 
> shutdown、suspend、resume，和probe、remove的原理类似，电源管理相关的实现，暂不说明。
> 
> pm，电源管理相关的逻辑，暂不说明。
> 
> iommu_ops，暂不说明。
> 
> p，一个struct subsys_private类型的指针，后面我们会用一个小节说明。

##### 2.2 struct subsys_private

该结构和[device_driver](http://www.wowotech.net/linux_kenrel/device_and_driver.html)中的struct driver_private类似，在"[Linux设备模型(5)_device和device driver](http://www.wowotech.net/linux_kenrel/device_and_driver.html)”章节中有提到它，但没有详细说明。

要说明subsys_private的功能，让我们先看一下该结构的定义：

 1: /* drivers/base/base.h, line 28 */

 2: struct subsys_private {

 3:     struct kset subsys;

 4:     struct kset *devices_kset;

 5:     struct list_head interfaces;

 6:     struct mutex mutex;

 7:  

 8:     struct kset *drivers_kset;

 9:     struct klist klist_devices;

 10:    struct klist klist_drivers;

 11:    struct blocking_notifier_head bus_notifier;

 12:    unsigned int drivers_autoprobe:1;

 13:    struct bus_type *bus;

 14:  

 15:    struct kset glue_dirs;

 16:    struct class *class;

 17: };

看到结构内部的字段，就清晰多了，没事不要乱起名字嘛！什么subsys啊，看的晕晕的！不过还是试着先理解一下为什么起名为subsys吧：

按理说，这个结构就是集合了一些bus模块需要使用的私有数据，例如kset啦、klist啦等等，命名为bus_private会好点（就像device_driver模块一样）。不过为什么内核没这么做呢？看看include/linux/device.h中的struct class结构（我们会在下一篇文章中介绍class）就知道了，因为class结构中也包含了一个一模一样的struct subsys_private指针，看来class和bus很相似啊。

想到这里，就好理解了，无论是bus，还是class，还是我们会在后面看到的一些虚拟的子系统，它都构成了一个“子系统（sub-system）”，该子系统会包含形形色色的device或device_driver，就像一个独立的王国一样，存在于内核中。而这些子系统的表现形式，就是/sys/bus（或/sys/class，或其它）目录下面的子目录，每一个子目录，都是一个子系统（如/sys/bus/spi/）。

好了，我们回过头来看一下struct subsys_private中各个字段的解释：

> subsys、devices_kset、drivers_kset是三个kset，由"[Linux设备模型(2)_Kobject](http://www.wowotech.net/linux_kenrel/kobject.html)”中对kset的描述可知，kset是一个特殊的kobject，用来集合相似的kobject，它在sysfs中也会以目录的形式体现。其中subsys，代表了本bus（如/sys/bus/spi），它下面可以包含其它的kset或者其它的kobject；devices_kset和drivers_kset则是bus下面的两个kset（如/sys/bus/spi/devices和/sys/bus/spi/drivers），分别包括本bus下所有的device和device_driver。
> 
> interface是一个list head，用于保存该bus下所有的interface。有关interface的概念后面会详细介绍。
> 
> klist_devices和klist_drivers是两个链表，分别保存了本bus下所有的device和device_driver的指针，以方便查找。
> 
> drivers_autoprobe，用于控制该bus下的drivers或者device是否自动probe，"[Linux设备模型(5)_device和device driver](http://www.wowotech.net/linux_kenrel/device_and_driver.html)”中有提到。
> 
> bus和class指针，分别保存上层的bus或者class指针。

##### 2.3 功能总结

根据上面的核心数据结构，可以总结出bus模块的功能包括：

- bus的注册和注销
- 本bus下有device或者device_driver注册到内核时的处理
- 本bus下有device或者device_driver从内核注销时的处理
- device_drivers的probe处理
- 管理bus下的所有device和device_driver

#### 3. 内部执行逻辑分析

##### 3.1 bus的注册

bus的注册是由bus_register接口实现的，该接口的原型是在include/linux/device.h中声明的，并在drivers/base/bus.c中实现，其原型如下：

 1: /* include/linux/device.h, line 118 */

 2: extern int __must_check bus_register(struct bus_type *bus);

> 该功能的执行逻辑如下：
> 
> - 为bus_type中struct subsys_private类型的指针分配空间，并更新priv->bus和bus->p两个指针为正确的值
> - 初始化priv->subsys.kobj的name、kset、ktype等字段，启动name就是该bus的name（它会体现在sysfs中），kset和ktype由bus模块实现，分别为bus_kset和bus_ktype
> - 调用kset_register将priv->subsys注册到内核中，该接口同时会向sysfs中添加对应的目录（如/sys/bus/spi）
> - 调用bus_create_file向bus目录下添加一个uevent attribute（如/sys/bus/spi/uevent）
> - 调用kset_create_and_add分别向内核添加devices和device_drivers kset，同时会体现在sysfs中
> - 初始化priv指针中的mutex、klist_devices和klist_drivers等变量
> - 调用add_probe_files接口，在bus下添加drivers_probe和drivers_autoprobe两个attribute（如/sys/bus/spi/drivers_probe和/sys/bus/spi/drivers_autoprobe），其中drivers_probe允许用户空间程序主动出发指定bus下的device_driver的probe动作，而drivers_autoprobe控制是否在device或device_driver添加到内核时，自动执行probe
> - 调用bus_add_attrs，添加由bus_attrs指针定义的bus的默认attribute，这些attributes最终会体现在/sys/bus/xxx目录下

##### 3.2 device和device_driver的添加

我们有在"[Linux设备模型(5)_device和device driver](http://www.wowotech.net/linux_kenrel/device_and_driver.html)”中讲过，内核提供了device_register和driver_register两个接口，供各个driver模块使用。而这两个接口的核心逻辑，是通过bus模块的bus_add_device和bus_add_driver实现的，下面我们看看这两个接口的处理逻辑。

这两个接口都是在drivers/base/base.h中声明，在drivers/base/bus.c中实现，其原型为：

 1: /* drivers/base/base.h, line 106 */

 2: extern int bus_add_device(struct device *dev);

 3:  

 4: /* drivers/base/base.h, line 110 */ 

 5: extern int bus_add_driver(struct device_driver *drv);

> bus_add_device的处理逻辑：
> 
> - 调用内部的device_add_attrs接口，将由bus->dev_attrs指针定义的默认attribute添加到内核中，它们会体现在/sys/devices/xxx/xxx_device/目录中
> - 调用sysfs_create_link接口，将该device在sysfs中的目录，链接到该bus的devices目录下，例如：  
>     
>     xxx# ls /sys/bus/spi/devices/spi1.0 -l                                                          
>     lrwxrwxrwx root     root              2014-04-11 10:46 spi1.0 -> ../../../devices/platform/s3c64xx-spi.1/spi_master/spi1/spi1.0  
>     其中/sys/devices/…/spi1.0，为该device在sysfs中真正的位置，而为了方便管理，内核在该设备所在的bus的xxx_bus/devices目录中，创建了一个符号链接
>     
> - 调用sysfs_create_link接口，在该设备的sysfs目录中（如/sys/devices/platform/alarm/）中，创建一个指向该设备所在bus目录的链接，取名为subsystem，例如：  
>     
>     xxx # ls /sys/devices/platform/alarm/subsystem -l                                                  
>     lrwxrwxrwx root     root              2014-04-11 10:28 subsystem -> ../../../bus/platform
>     
> - 最后，毫无疑问，要把该设备指针保存在bus->priv->klist_devices中
> 
> bus_add_driver的处理逻辑：
> 
> - 为该driver的struct driver_private指针（priv）分配空间，并初始化其中的priv->klist_devices、priv->driver、priv->kobj.kset等变量，同时将该指针保存在device_driver的p处
> - 将driver的kset（priv->kobj.kset）设置为bus的drivers kset（bus->p->drivers_kset），这就意味着所有driver的kobject都位于bus->p->drivers_kset之下（寄/sys/bus/xxx/drivers目录下）
> - 以driver的名字为参数，调用kobject_init_and_add接口，在sysfs中注册driver的kobject，体现在/sys/bus/xxx/drivers/目录下，如/sys/bus/spi/drivers/spidev
> - 将该driver保存在bus的klist_drivers链表中，并根据drivers_autoprobe的值，选择是否调用driver_attach进行probe
> - 调用driver_create_file接口，在sysfs的该driver的目录下，创建uevent attribute
> - 调用driver_add_attrs接口，在sysfs的该driver的目录下，创建由bus->drv_attrs指针定义的默认attribute
> - 同时根据suppress_bind_attrs标志，决定是否在sysfs的该driver的目录下，创建bind和unbind attribute（具体可参考"[Linux设备模型(5)_device和device driver](http://www.wowotech.net/linux_kenrel/device_and_driver.html)”中的介绍）  
>     

##### 3.3 driver的probe

我们在"[Linux设备模型(5)_device和device driver](http://www.wowotech.net/linux_kenrel/device_and_driver.html)”中，我们已经介绍过driver的probe时机及过程，其中大部分的逻辑会依赖bus模块的实现，主要为bus_probe_device和driver_attach接口。同样，这两个接口都是在drivers/base/base.h中声明，在drivers/base/bus.c中实现。

这两个结构的行为类似，逻辑也很简单，既：搜索所在的bus，比对是否有同名的device_driver（或device），如果有并且该设备没有绑定Driver（注：这一点很重要，通过它，可以使同一个Driver，驱动相同名称的多个设备，后续在Platform设备的描述中会提及）则调用device_driver的probe接口。

#### 4. 杂项

##### 4.1 再谈Subsystem

在旧的Linux内核版本中（以蜗蜗使用的linux2.6.23版本的内核为例），sysfs下所有的顶层目录（包括一些二级目录）都是以调用subsystem_register接口，以sub-system的形式注册到内核的，如：

> /sys/bus/
> 
> /sys/devices/
> 
> /sys/devices/system/
> 
> /sys/block
> 
> /sys/kernel/
> 
> /sys/slab/
> 
> …

那时的subsystem_register的实现很简单，就是调用kset_register，创建一个kset。我们知道，kset就是一堆kobject的集合，并会在sysfs中以目录的形式呈现出来。

在新版本的内核中（如[“Linux内核分析”系列文章](http://www.wowotech.net/linux_kenrel/11.html)所参考的linux3.10.29），subsystem的实现有了很大变化，例如：去掉了subsystem_register接口（但为了兼容/sys/device/system子系统，在drivers/base/bus.c中，增加了一个subsys_register的内部接口，用于实现相应的功能）。根据这些变化，现在注册subsystem有两种方式：

方式一，在各自的初始化函数中，调用kset_create_and_add接口，创建对应的子系统，包括：

- bus子系统，/sys/bus/，buses_init（drivers/base/bus.c）
- class子系统，/sys/class
- kernel子系统，/sys/kernel
- firmware子系统，/sys/firmware
- 等等

其中bus子系统就是本文所讲的Bus模块，而其它的，我们会在后续的文章中陆续讲述。这个方式和旧版本内核使用kset_register接口的方式基本一样。

方式二，在bus模块中，利用subsys_register接口，封装出两个API：subsys_system_register和subsys_virtual_register，分别用于注册system设备(/sys/devices/system/*)和virtual设备(/sys/devices/virtual/*)。 而该方式和方式一的区别是：它不仅仅创建了sysfs中的目录，同时会注册同名的bus和device。

##### 4.2 system/virtual/platform

在Linux内核中，有三种比较特殊的bus（或者是子系统），分别是system bus、virtual bus和platform bus。它们并不是一个实际存在的bus（像USB、I2C等），而是为了方便设备模型的抽象，而虚构的。

system bus是旧版内核提出的概念，用于抽象系统设备（如CPU、Timer等等）。而新版内核认为它是个坏点子，因为任何设备都应归属于一个普通的子系统（New subsystems should use plain subsystems, drivers/base/bus.c, line 1264），所以就把它抛弃了（不建议再使用，它的存在只为兼容旧有的实现）。

virtaul bus是一个比较新的bus，主要用来抽象那些虚拟设备，所谓的虚拟设备，是指不是真实的硬件设备，而是用软件模拟出来的设备，例如虚拟机中使用的虚拟的网络设备（有关该bus的描述，可参考该链接处的解释：[https://lwn.net/Articles/326540/](https://lwn.net/Articles/326540/ "https://lwn.net/Articles/326540/")）。

platform bus就比较普通，它主要抽象集成在CPU（SOC）中的各种设备。这些设备直接和CPU连接，通过总线寻址和中断的方式，和CPU交互信息。 

我们会在后续的文章中，进一步分析这些特殊的bus，这里就暂时不详细描述了。

  

**4.3 subsys interface**

##### 

subsys interface是一个很奇怪的东西，除非有一个例子，否则很难理解。代码中是这样注释的：

> ##### 
> 
> Interfaces usually represent a specific functionality of a subsystem/class of devices.

##### 

字面上理解，它抽象了bus下所有设备的一些特定功能。

kernel使用struct subsys_interface结构抽象subsys interface，并提供了subsys_interface_register/subsys_interface_unregister用于注册/注销subsys interface，bus下所有的interface都挂载在struct subsys_private变量的“interface”链表上（具体可参考2.2小节的描述）。struct subsys_interface的定义如下：

1. /**
2.  * struct subsys_interface - interfaces to device functions
3.  * @name:       name of the device function
4.  * @subsys:     subsytem of the devices to attach to
5.  * @node:       the list of functions registered at the subsystem
6.  * @add_dev:    device hookup to device function handler
7.  * @remove_dev: device hookup to device function handler
8.  *
9.  * Simple interfaces attached to a subsystem. Multiple interfaces can
10.  * attach to a subsystem and its devices. Unlike drivers, they do not
11.  * exclusively claim or control devices. Interfaces usually represent
12.  * a specific functionality of a subsystem/class of devices.
13.  */     
14. struct subsys_interface {
15.         const char *name;
16.         struct bus_type *subsys;
17.         struct list_head node;
18.         int (*add_dev)(struct device *dev, struct subsys_interface *sif);
19.         int (*remove_dev)(struct device *dev, struct subsys_interface *sif);
20. };

> ##### 
> 
> name，interface的名称。
> 
> ##### 
> 
> subsys，interface所属的bus。
> 
> ##### 
> 
> node，用于将interface挂到bus中。
> 
> ##### 
> 
> add_dev/remove_dev，两个回调函数，subsys interface的核心功能。当bus下有设备增加或者删除的时候，bus core会调用它下面所有subsys interface的add_dev或者remove_dev回调。设计者可以在这两个回调函数中实现所需功能，例如绑定该“specific functionality”所对应的driver，等等。

##### 

subsys interface的实现逻辑比较简单，这里不再详细描述了，具体可参考“drivers/base/bus.c”中相应的代码。另外，后续分析cpufreq framework的时候，会遇到使用subsys interface的例子，到时候我们再进一步理解它的现实意义。

_原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/linux_kenrel/bus.html)。_

标签: [Linux](http://www.wowotech.net/tag/Linux) [Kernel](http://www.wowotech.net/tag/Kernel) [内核](http://www.wowotech.net/tag/%E5%86%85%E6%A0%B8) [设备模型](http://www.wowotech.net/tag/%E8%AE%BE%E5%A4%87%E6%A8%A1%E5%9E%8B) [bus](http://www.wowotech.net/tag/bus)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [教育漫谈(1)_《乌合之众》摘录](http://www.wowotech.net/tech_discuss/jymt1.html) | [Linux设备模型(5)_device和device driver](http://www.wowotech.net/device_model/device_and_driver.html)»

**评论：**

**kernfs**  
2020-02-13 17:58

wowo，您好，请教个问题。本章讲到的bus_add_device函数，有个功能是创建链接，如下：  
“调用sysfs_create_link接口，将该device在sysfs中的目录，链接到该bus的devices目录下，例如：  
  
xxx# ls /sys/bus/spi/devices/spi1.0 -l                                                          
lrwxrwxrwx root     root              2014-04-11 10:46 spi1.0 -> ../../../devices/platform/s3c64xx-spi.1/spi_master/spi1/spi1.0  
其中/sys/devices/…/spi1.0，为该device在sysfs中真正的位置，而为了方便管理，内核在该设备所在的bus的xxx_bus/devices目录中，创建了一个符号链接 ”  
此处有个疑问，就是从sys/devices/platform/s3c64xx-spi.1/spi_master/spi1/spi1.0是不是可以断定spi1属于platform device？从platform device的初始化流程来看，应该属于platform bus，那为什么创建链接的时候会放在/sys/bus/spi/devices下面呢，这个目录不应该是spi bus的目录吗？

[回复](http://www.wowotech.net/device_model/bus.html#comment-7877)

**kernfs**  
2020-02-18 14:59

@kernfs：该问题，重新整理了下code，基本明白spi和i2c设备的特殊性。  
引用其他人的回复：“i2c和spi有点特殊，i2c和spi有主从设备之分，并且有自己的总线i2c bus和spi bus。挂载在platform bus下的只是i2c和spi的controller设备（即master），以iMX的i2c举例，在执行of_platform_populate()时创建的platform device只代表了I2C controller，在注册platform_driver(如i2c_imx_driver)时会执行platform_driver的probe函数，即i2c_imx_probe()，probe函数会分配一个i2c_adapter作为i2c controller的抽象设备挂接在i2c bus上。同时该probe函数会遍历该i2c controller设备树节点下的所有i2c子节点，创建i2c_client设备并将i2c_client->adapter指向这个i2c_adapter。随后在i2c client加入i2c bus时或者i2c client的驱动i2c_driver加入i2c bus时，会匹配i2c_client与i2c_driver，匹配成功会执行i2c_driver的probe函数来初始化i2c_client。i2c_client通过adapter指针指向自己的i2c_adapter，i2c_adapter实现i2c协议传输的方法。上层应用在使用i2c_client的字符设备时最终通过该指针来操作i2c_adapter发出正确的时序和数据。”

[回复](http://www.wowotech.net/device_model/bus.html#comment-7887)

**huanglei104**  
2019-11-11 10:49

"为了方便设备模型的实现，内核规定，系统中的每个设备都要连接在一个Bus上" , 我想请问一下， 一个字符设备连接到的是哪个Bus呢？

[回复](http://www.wowotech.net/device_model/bus.html#comment-7745)

**h233**  
2018-12-22 11:09

@wowo:请教一下，搜索所在的bus，比对是否有同名的device_driver（或device），如果有并且该设备没有绑定Driver（注：这一点很重要，通过它，可以使同一个Driver，驱动相同名称的多个设备，后续在Platform设备的描述中会提及）则调用device_driver的probe接口。  
  
我这边现在有2颗同样的IC，都是i2c设备，现在使用了这个机制，共用一套driver,之前访问它们都是通过各自的sys节点(尽管设备名称相同，但是i2c地址不同嘛，所以好区分），但是后来发现这样访问，数据若有变化时会有滞后的问题，所以想跟放在TP的中断线程里，和TP坐标一起上报上去。  
  
现在有个困难是，我发现如果用这种方式的话，我不知道如何取出各自的 i2c_client 变量，毕竟是同一套代码加载了两次而已。虽然我也明白模块之间尽量不要有耦合，或者说这种工作在用户空间做也许更好？但是我想在这个方向尝试一下，不知道有什么建议？谢谢

[回复](http://www.wowotech.net/device_model/bus.html#comment-7099)

**h233**  
2019-04-12 14:11

@h233：全局变量对于两个device而言是共享的同一个，只不过是probe加载了两次而已。  
只需要创建一个全局变量，在每次probe时进行记录，之后的驱动接口做好保护即可。

[回复](http://www.wowotech.net/device_model/bus.html#comment-7349)

**[wowo](http://www.wowotech.net/)**  
2019-04-14 11:42

@h233：之前你的问题没有看到，就漏掉了，抱歉啊！好在你自己想到解决方法了，赞一个！！！

[回复](http://www.wowotech.net/device_model/bus.html#comment-7350)

**小学生**  
2017-06-06 14:07

楼主，我想请问下PCI子系统的struct pci_bus 和 pci_bus_type（类型为struct bus_type） 是什么关系？两者分别用在哪些场景呢？  
我的理解是：我们在使用pci_read_config_byte()读取某个PCI设备的config space信息时，最终会调用到该pci device所在pci_bus的ops->read函数；但是在使用pci_get_device()来查找当前是否有对应vendorID,deviceID的pci device时，最终会调到bus_find_device()上，而该函数的第一个参数类型是struct bus_type，相当于是在pci_bus_type上查找是否有需要查找的pci device。感觉这两个地方不大对应啊？  
请楼主赐教

[回复](http://www.wowotech.net/device_model/bus.html#comment-5628)

**[wowo](http://www.wowotech.net/)**  
2017-06-06 16:49

@小学生：不是很熟悉PCI，不过从逻辑上来说，pci_bus_types应该是是挂各种PCI控制器的bus，而struct pci_bus是每个PCI控制器抽象出来的bus，用来挂PCI设备。如下图（具体细节你要自己看看）：  
=========================pci_bus_type====================  
    +           +  
    |        |  
    |        |  
    +        +  
pci driver pci device X--------struct pci_bus------------  
    \            /                              |  
     \          /                               |  
    probe                                   +  
                                      devices

[回复](http://www.wowotech.net/device_model/bus.html#comment-5630)

**REGA**  
2017-01-06 10:59

hi wowo,在看code的时候  
  
priv->drivers_kset = kset_create_and_add("drivers", NULL,  
                         &priv->subsys.kobj);  
这个/sys/bus/drivers对应的kset的kset_uevet_ops是NULL，之后driver注册的时候，bus_add_driver会把driver对应的kobject注册到/sys/bus/drivers下面？那这个kset_uevet_ops是NULL不会有影响吗？

[回复](http://www.wowotech.net/device_model/bus.html#comment-5108)

**[wowo](http://www.wowotech.net/)**  
2017-01-06 14:45

@REGA：没有问题，uevent_ops不是必须的，它的功能为（可参考http://www.wowotech.net/linux_kenrel/kobject.html）：  
uevent_ops，该kset的uevent操作函数集。当任何Kobject需要上报uevent时，都要调用它所从属的kset的uevent_ops，添加环境变量，或者过滤event（kset可以决定哪些event可以上报）。因此，如果一个kobject不属于任何kset时，是不允许发送uevent的。

[回复](http://www.wowotech.net/device_model/bus.html#comment-5109)

**REGA**  
2017-01-09 09:58

@wowo：@wowo，那就是说在bus上挂某个driver的时候不需要uevent操作吗？还是说只要从属于kset就可以做uevent操作，只是有一些限制（无法添加环境变量，无法过滤event）？  
  
  
  
    priv->devices_kset = kset_create_and_add("devices", NULL,  
                         &priv->subsys.kobj);  
  
而且/sys/bus/devices这个kset的uevent_ops也为null,我理解挂device的时候是需要uevent的？

[回复](http://www.wowotech.net/device_model/bus.html#comment-5113)

**[wowo](http://www.wowotech.net/)**  
2017-01-09 10:19

@REGA：你理解的是对的，device肯定是需要uevent的（因为device是操作的主题，可能会产生各种事件），但driver，还真不一定。另外我也没有看这么细，因此也不一定理解正确哈～～

[回复](http://www.wowotech.net/device_model/bus.html#comment-5116)

**REGA**  
2017-01-11 13:58

@wowo：@wowo:我看了code，在driver_register中bus_add_driver后会有kobject_uevent(&drv->p->kobj, KOBJ_ADD);  
但是这个driver所属的kset中的uevent_ops又为NULL，这个让我很费解

[回复](http://www.wowotech.net/device_model/bus.html#comment-5131)

**[wowo](http://www.wowotech.net/)**  
2017-01-12 13:43

@REGA：uevent_ops的作用，是为了统一处理该kset下所有kobject上报的uevent，如果为NULL，意味着不再帮忙处理，driver上报的uevent，会原封不动的报上去。没有毛病吧？所以不费解的:-)

[回复](http://www.wowotech.net/device_model/bus.html#comment-5139)

**REGA**  
2017-01-13 09:35

@wowo：@wowo，感谢wowo的解答~

**REGA**  
2017-01-09 10:06

@wowo：我理解/sys/bus/devices应该是一个链接，device真正的object操作是对应在/sys/devices下的  
  
devices_kset = kset_create_and_add("devices", &device_uevent_ops, NULL);  
  
这个kset是有对应的uevent_ops的，但是driver对应的kobject，我只在bus_add_driver下找到，driver的注册不需要uevent操作吗？  
  
问题比较小白，请wowo赐教。。。

[回复](http://www.wowotech.net/device_model/bus.html#comment-5115)

**linuxxx**  
2016-10-14 16:46

想问下wowo,Linux设备模型这个系列代码分析是基于哪个内核版本

[回复](http://www.wowotech.net/device_model/bus.html#comment-4711)

**[wowo](http://www.wowotech.net/)**  
2016-10-14 17:24

@linuxxx：抱歉啊，当初觉得设备模型比较通用，就没有太在意版本，应该是3.10左右的。

[回复](http://www.wowotech.net/device_model/bus.html#comment-4712)

**lucifer**  
2016-08-28 21:57

在3.12 内核上 看到subsys_interface  ，不过对于这个sif->add_dev(dev, sif); 还不明白什么用途  
void bus_probe_device(struct device *dev)  
{  
         struct bus_type *bus = dev->bus;  
         struct subsys_interface *sif;  
         int ret;  
  
         if (!bus)  
                 return;  
  
        if (bus->p->drivers_autoprobe) {  
                 ret = device_attach(dev);  
                 WARN_ON(ret < 0);  
         }  
  
        mutex_lock(&bus->p->mutex);  
         list_for_each_entry(sif, &bus->p->interfaces, node)  
                if (sif->add_dev)  
                        sif->add_dev(dev, sif);  
        mutex_unlock(&bus->p->mutex);  
}

[回复](http://www.wowotech.net/device_model/bus.html#comment-4462)

**[wowo](http://www.wowotech.net/)**  
2016-08-28 22:19

@lucifer：cpufreq framework中有add_dev的例子，这两篇文章特意提到了，你可以看一下：  
http://www.wowotech.net/pm_subsystem/cpufreq_overview.html  
http://www.wowotech.net/pm_subsystem/cpufreq_core.html

[回复](http://www.wowotech.net/device_model/bus.html#comment-4463)

**lucifer**  
2016-08-28 23:16

@wowo：非常谢谢wowo ~~ 我会去学习的。自己基础太差了。。。

[回复](http://www.wowotech.net/device_model/bus.html#comment-4464)

**狼花1997**  
2016-08-12 20:12

我想问一下sys/bus/spi/devices/spi1.0   这个spi1.0 是如何注册得来的啊

[回复](http://www.wowotech.net/device_model/bus.html#comment-4391)

**[wowo](http://www.wowotech.net/)**  
2016-08-12 21:58

@狼花1997：大致的过程，是：spi_add_device-->device_add，具体可以拔拔spi的代码：  
https://github.com/wowotechX/linux/blob/x_integration/drivers/spi/spi.c  
刚好可以借助理解设备模型的概念~

[回复](http://www.wowotech.net/device_model/bus.html#comment-4392)

**凌云**  
2016-07-25 08:39

能不能解释一下struct subsys_private结构中struct kset glue_dirs成员的意义？

[回复](http://www.wowotech.net/device_model/bus.html#comment-4307)

**[wowo](http://www.wowotech.net/)**  
2016-07-25 09:06

@凌云：它主要在下面的场景使用：  
device_add的时候，如果这个device属于某一个class，同时它又是某一个device子设备，设备模型会利用glue_dirs，在parent device的目录下，创建一个名称为class、kset为glue_dirs的目录。  
根据内核的注释，这是为了解决namespace的冲突。再进一步，我就没有去理解了。  
另外，你可以写一个小的测试代码，创建一个这样的设备，看看最终的结果如何。这样可以加深理解。

[回复](http://www.wowotech.net/device_model/bus.html#comment-4308)

**凌云**  
2016-07-25 10:04

@wowo：恢复得真快，谢谢！

[回复](http://www.wowotech.net/device_model/bus.html#comment-4309)

**凌云**  
2016-07-25 10:05

@凌云：回复的真快，谢谢！

[回复](http://www.wowotech.net/device_model/bus.html#comment-4310)

**vedic**  
2015-10-25 09:50

Thanks a lot :)

[回复](http://www.wowotech.net/device_model/bus.html#comment-2828)

1 [2](http://www.wowotech.net/device_model/bus.html/comment-page-2#comments)

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
    
    - [ARM64的启动过程之（三）：为打开MMU而进行的CPU初始化](http://www.wowotech.net/armv8a_arch/__cpu_setup.html)
    - [Linux vm运行参数之（一）：overcommit相关的参数](http://www.wowotech.net/memory_management/overcommit.html)
    - [O(n)、O(1)和CFS调度器](http://www.wowotech.net/process_management/scheduler-history.html)
    - [linux kernel的中断子系统之（九）：tasklet](http://www.wowotech.net/irq_subsystem/tasklet.html)
    - [vim使用技巧摘录](http://www.wowotech.net/linux_application/vim_skill.html)
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

作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2014-4-2 19:28 分类：[统一设备模型](http://www.wowotech.net/sort/device_model)

# **1. 前言**

device和device driver是Linux驱动开发的基本概念。Linux kernel的思路很简单：驱动开发，就是要开发指定的软件（driver）以驱动指定的设备，所以kernel就为设备和驱动它的driver定义了两个数据结构，分别是device和device_driver。因此本文将会围绕这两个数据结构，介绍Linux设备模型的核心逻辑，包括：

设备及设备驱动在kernel中的抽象、使用和维护；

设备及设备驱动的注册、加载、初始化原理；

设备模型在实际驱动开发过程中的使用方法。

注：在介绍device和device_driver的过程中，会遇到很多额外的知识点，如Class、Bus、DMA、电源管理等等，这些知识点都很复杂，任何一个都可以作为一个单独的专题区阐述，因此本文不会深入解析它们，而会在后续的文章中专门描述。

# **2. struct device和struct device_driver**

在阅读Linux内核源代码时，通过核心数据结构，即可理解某个模块60%以上的逻辑，设备模型部分尤为明显。

在include/linux/device.h中，Linux内核定义了设备模型中最重要的两个数据结构，struct device和struct device_driver。

- struct device

```cpp
1: /\* include/linux/device.h, line 660 \*/
2: struct device {
3:     struct device       \*parent;
4:
5:     struct device_private   \*p;
6:
7:     struct kobject kobj;
8:     const char *init_name; /* initial name of the device \*/
9:     const struct device_type \*type;
10:
11:    struct mutex        mutex; /\* mutex to synchronize calls to
12:                             * its driver.
13:                             \*/
14:
15:    struct bus_type *bus; /* type of bus device is on \*/
16:    struct device_driver *driver; /* which driver has allocated this
17:                                 device \*/
18:    void *platform_data; /* Platform specific data, device
19:                         core doesn't touch it \*/
20:    struct dev_pm_info  power;
21:    struct dev_pm_domain    \*pm_domain;
22:
23: #ifdef CONFIG_PINCTRL
24:    struct dev_pin_info \*pins;
25: #endif
26:
27: #ifdef CONFIG_NUMA
28:    int numa_node; /\* NUMA node this device is close to \*/
29: #endif
30:    u64     *dma_mask; /* dma mask (if dma'able device) \*/
31:    u64     coherent_dma_mask;/\* Like dma_mask, but for
32:                             alloc_coherent mappings as
33:                             not all hardware supports
34:                             64 bit addresses for consistent
35:                             allocations such descriptors. \*/
36:
37:    struct device_dma_parameters \*dma_parms;
38:
39:    struct list_head    dma_pools; /\* dma pools (if dma'ble) \*/
40:
41:    struct dma_coherent_mem *dma_mem; /* internal for coherent mem
42:                            override \*/
43: #ifdef CONFIG_CMA
44:    struct cma *cma_area; /* contiguous memory area for dma
45:                            allocations \*/
46: #endif
47:    /\* arch specific additions \*/
48:    struct dev_archdata archdata;
49:
50:    struct device_node  *of_node; /* associated device tree node \*/
51:    struct acpi_dev_node    acpi_node; /\* associated ACPI device node \*/
52:
53:    dev_t           devt; /\* dev_t, creates the sysfs "dev" \*/
54:    u32         id; /\* device instance \*/
55:
56:    spinlock_t      devres_lock;
57:    struct list_head    devres_head;
58:
59:    struct klist_node   knode_class;
60:    struct class \*class;
61:    const struct attribute_group \**groups; /* optional groups \*/
62:
63:    void (\*release)(struct device \*dev);
64:    struct iommu_group  \*iommu_group;
65: };
```

> device结构很复杂（不过linux内核的开发人员素质是很高的，该接口的注释写的非常详细，感兴趣的同学可以参考内核源代码），这里将会选一些对理解设备模型非常关键的字段进行说明。
>
> parent，该设备的父设备，一般是该设备所从属的bus、controller等设备。
>
> p，一个用于struct device的私有数据结构指针，该指针中会保存子设备链表、用于添加到bus/driver/prent等设备中的链表头等等，具体可查看源代码。
>
> kobj，该数据结构对应的struct kobject。
>
> init_name，该设备的名称。
>
> 注1：在设备模型中，名称是一个非常重要的变量，任何注册到内核中的设备，都必须有一个合法的名称，可以在初始化时给出，也可以由内核根据“bus name + device ID”的方式创造。
>
> type，struct device_type结构是新版本内核新引入的一个结构，它和struct device关系，非常类似[stuct kobj_type和struct kobject](http://www.wowotech.net/linux_kenrel/kobject.html)之间的关系，后续会再详细说明。
>
> bus，该device属于哪个总线（后续会详细描述）。
>
> driver，该device对应的device driver。
>
> platform_data，一个指针，用于保存具体的平台相关的数据。具体的driver模块，可以将一些私有的数据，暂存在这里，需要使用的时候，再拿出来，因此设备模型并不关心该指针得实际含义。
>
> power、pm_domain，电源管理相关的逻辑，后续会由电源管理专题讲解。
>
> pins，"PINCTRL”功能，暂不描述。
>
> numa_node，"NUMA”功能，暂不描述。
>
> dma_mask~archdata，DMA相关的功能，暂不描述。
>
> devt，dev_t是一个32位的整数，它由两个部分（Major和Minor）组成，在需要以设备节点的形式（字符设备和块设备）向用户空间提供接口的设备中，当作设备号使用。在这里，该变量主要用于在sys文件系统中，为每个具有设备号的device，创建/sys/dev/\* 下的对应目录，如下：
>
> 1|root@android:/storage/sdcard0 #ls /sys/dev/char/1:                                                                     \
> 1:1/  1:11/ 1:13/ 1:14/ 1:2/  1:3/  1:5/  1:7/  1:8/  1:9/ \
> 1|root@android:/storage/sdcard0 #ls /sys/dev/char/1:1                                                                    \
> 1:1/  1:11/ 1:13/ 1:14/\
> 1|root@android:/storage/sdcard0 # ls /sys/dev/char/1:1 \
> /sys/dev/char/1:1
>
> class，该设备属于哪个class。
>
> groups，该设备的默认attribute集合。将会在设备注册时自动在sysfs中创建对应的文件。

- struct device_driver

```cpp
1: /\* include/linux/device.h, line 213 \*/
2: struct device_driver {
3:     const char \*name;
4:     struct bus_type     \*bus;
5:
6:     struct module       \*owner;
7:     const char *mod_name; /* used for built-in modules \*/
8:
9:     bool suppress_bind_attrs; /\* disables bind/unbind via sysfs \*/
10:
11:    const struct of_device_id   \*of_match_table;
12:    const struct acpi_device_id \*acpi_match_table;
13:
14:    int (\*probe) (struct device \*dev);
15:    int (\*remove) (struct device \*dev);
16:    void (\*shutdown) (struct device \*dev);
17:    int (\*suspend) (struct device \*dev, pm_message_t state);
18:    int (\*resume) (struct device \*dev);
19:    const struct attribute_group \*\*groups;
20:
21:    const struct dev_pm_ops \*pm;
22:
23:    struct driver_private \*p;
24: };
```

> device_driver就简单多了（在早期的内核版本中driver的数据结构为"struct driver”，不知道从哪个版本开始，就改成device_driver了）：
>
> name，该driver的名称。和device结构一样，该名称非常重要，后面会再详细说明。
>
> bus，该driver所驱动设备的总线设备。为什么driver需要记录总线设备的指针呢？因为内核要保证在driver运行前，设备所依赖的总线能够正确初始化。
>
> owner、mod_name，內核module相关的变量，暂不描述。
>
> suppress_bind_attrs，是不在sysfs中启用bind和unbind attribute，如下：root@android:/storage/sdcard0 # ls /sys/bus/platform/drivers/switch-gpio/                                                  \
> bind   uevent unbind在kernel中，bind/unbind是从用户空间手动的为driver绑定/解绑定指定的设备的机制。这种机制是在bus.c中完成的，后面会详细解释。
>
> probe、remove，这两个接口函数用于实现driver逻辑的开始和结束。Driver是一段软件code，因此会有开始和结束两个代码逻辑，就像PC程序，会有一个main函数，main函数的开始就是开始，return的地方就是结束。而内核driver却有其特殊性：在设备模型的结构下，只有driver和device同时存在时，才需要开始执行driver的代码逻辑。这也是probe和remove两个接口名称的由来：检测到了设备和移除了设备（就是为热拔插起的！）。
>
> shutdown、suspend、resume、pm，电源管理相关的内容，会在电源管理专题中详细说明。
>
> groups，和struct device结构中的同名变量类似，driver也可以定义一些默认attribute，这样在将driver注册到内核中时，内核设备模型部分的代码（driver/base/driver.c）会自动将这些attribute添加到sysfs中。
>
> p，driver core的私有数据指针，其它模块不能访问。

# **3. 设备模型框架下驱动开发的基本步骤**

在设备模型框架下，设备驱动的开发是一件很简单的事情，主要包括2个步骤：

步骤1：分配一个struct device类型的变量，填充必要的信息后，把它注册到内核中。

步骤2：分配一个struct device_driver类型的变量，填充必要的信息后，把它注册到内核中。

这两步完成后，内核会在合适的时机（后面会讲），调用struct device_driver变量中的probe、remove、suspend、resume等回调函数，从而触发或者终结设备驱动的执行。而所有的驱动程序逻辑，都会由这些回调函数实现，此时，驱动开发者眼中便不再有“设备模型”，转而只关心驱动本身的实现。

> 以上两个步骤的补充说明：
>
> 1. 一般情况下，Linux驱动开发很少直接使用device和device_driver，因为内核在它们之上又封装了一层，如soc device、platform device等等，而这些层次提供的接口更为简单、易用（也正是因为这个原因，本文并不会过多涉及device、device_driver等模块的实现细节）。
>
> 1. 内核提供很多struct device结构的操作接口（具体可以参考include/linux/device.h和drivers/base/core.c的代码），主要包括初始化（device_initialize）、注册到内核（device_register）、分配存储空间+初始化+注册到内核（device_create）等等，可以根据需要使用。
>
> 1. device和device_driver必须具备相同的名称，内核才能完成匹配操作，进而调用device_driver中的相应接口。这里的同名，作用范围是同一个bus下的所有device和device_driver。
>
> 1. device和device_driver必须挂载在一个bus之下，该bus可以是实际存在的，也可以是虚拟的。
>
> 1. driver开发者可以在struct device变量中，保存描述设备特征的信息，如寻址空间、依赖的GPIOs等，因为device指针会在执行probe等接口时传入，这时driver就可以根据这些信息，执行相应的逻辑操作了。

# **4. 设备驱动probe的时机**

所谓的"probe”，是指在Linux内核中，如果存在相同名称的device和device_driver（注：还存在其它方式，我们先不关注了），内核就会执行device_driver中的probe回调函数，而该函数就是所有driver的入口，可以执行诸如硬件设备初始化、字符设备注册、设备文件操作ops注册等动作（"remove”是它的反操作，发生在device或者device_driver任何一方从内核注销时，其原理类似，就不再单独说明了）。

设备驱动prove的时机有如下几种（分为自动触发和手动触发）：

- 将struct device类型的变量注册到内核中时自动触发（device_register，device_add，device_create_vargs，device_create）
- 将struct device_driver类型的变量注册到内核中时自动触发（driver_register）
- 手动查找同一bus下的所有device_driver，如果有和指定device同名的driver，执行probe操作（device_attach）
- 手动查找同一bus下的所有device，如果有和指定driver同名的device，执行probe操作（driver_attach）
- 自行调用driver的probe接口，并在该接口中将该driver绑定到某个device结构中----即设置dev->driver（device_bind_driver）

> 注2：probe动作实际是由bus模块（会在下一篇文章讲解）实现的，这不难理解：device和device_driver都是挂载在bus这根线上，因此只有bus最清楚应该为哪些device、哪些driver配对。
>
> 注3：每个bus都有一个drivers_autoprobe变量，用于控制是否在device或者driver注册时，自动probe。该变量默认为1（即自动probe），bus模块将它开放到sysfs中了，因而可在用户空间修改，进而控制probe行为。

**5. 其它杂项**

**5.1 device_attribute和driver_attribute**

在"[Linux设备模型(4)\_sysfs](http://www.wowotech.net/linux_kenrel/dm_sysfs.html)”中，我们有讲到，大多数时候，attribute文件的读写数据流为：vfs---->sysfs---->kobject---->attibute---->kobj_type---->sysfs_ops---->xxx_attribute，其中kobj_type、sysfs_ops和xxx_attribute都是由包含kobject的上层数据结构实现。

Linux内核中关于该内容的例证到处都是，device也不无例外的提供了这种例子，如下：

1: /\* driver/base/core.c, line 118 \*/

2: static ssize_t dev_attr_show(struct kobject \*kobj, struct attribute \*attr,

3: char \*buf)

4: {

5:     struct device_attribute \*dev_attr = to_dev_attr(attr);

6:     struct device \*dev = kobj_to_dev(kobj);

7:     ssize_t ret = -EIO;

8:

9:     if (dev_attr->show)

10:        ret = dev_attr->show(dev, dev_attr, buf);

11:        if (ret >= (ssize_t)PAGE_SIZE) {

12:            print_symbol("dev_attr_show: %s returned bad count\\n",

13:                        (unsigned long)dev_attr->show);

14:    }

15:    return ret;

16: }

17:

18: static ssize_t dev_attr_store(struct kobject \*kobj, struct attribute \*attr,

19: const char \*buf, size_t count)

20: {

21:    struct device_attribute \*dev_attr = to_dev_attr(attr);

22:    struct device \*dev = kobj_to_dev(kobj);

23:    ssize_t ret = -EIO;

24:

25:    if (dev_attr->store)

26:        ret = dev_attr->store(dev, dev_attr, buf, count);

27:    return ret;

28: }

29:

30: static const struct sysfs_ops dev_sysfs_ops = {

31:    .show   = dev_attr_show,

32:    .store  = dev_attr_store,

33: };

34:

35: /\* driver/base/core.c, line 243 \*/

36: static struct kobj_type device_ktype = {

37:    .release    = device_release,

38:    .sysfs_ops  = &dev_sysfs_ops,

39:    .namespace = device_namespace,

40: };

41:

42: /\* include/linux/device.h, line 478 \*/

43: /\* interface for exporting device attributes \*/

44: struct device_attribute {

45:    struct attribute    attr;

46:    ssize_t (\*show)(struct device \*dev, struct device_attribute \*attr,

47:                    char \*buf);

48:    ssize_t (\*store)(struct device \*dev, struct device_attribute \*attr,

49:                    const char \*buf, size_t count);

50: };

至于driver的attribute，则要简单的多，其数据流为：vfs---->sysfs---->kobject---->attribute---->driver_attribute，如下：

1: /\* include/linux/device.h, line 247 \*/

2: /\* sysfs interface for exporting driver attributes \*/

3:

4: struct driver_attribute {

5:     struct attribute attr;

6:     ssize_t (\*show)(struct device_driver \*driver, char \*buf);

7:     ssize_t (\*store)(struct device_driver \*driver, const char \*buf,

8:                     size_t count);

9: };

10:

11: #define DRIVER_ATTR(\_name, \_mode, \_show, \_store)    \\

12: struct driver_attribute driver_attr\_##\_name =       \\

13:    \_\_ATTR(\_name, \_mode, \_show, \_store)

**5.2 device_type**

device_type是内嵌在struct device结构中的一个数据结构，用于指明设备的类型，并提供一些额外的辅助功能。它的的形式如下：

1: /\* include/linux/device.h, line 467 \*/

2: struct device_type {

3:     const char \*name;

4:     const struct attribute_group \*\*groups;

5:     int (\*uevent)(struct device \*dev, struct kobj_uevent_env \*env);

6:     char \*(\*devnode)(struct device \*dev, umode_t \*mode,

7:                     kuid_t \*uid, kgid_t \*gid);

8:     void (\*release)(struct device \*dev);

9:

10:    const struct dev_pm_ops \*pm;

11: };

> device_type的功能包括：
>
> - name表示该类型的名称，当该类型的设备添加到内核时，内核会发出"DEVTYPE=‘name’”类型的uevent，告知用户空间某个类型的设备available了
> - groups，该类型设备的公共attribute集合。设备注册时，会同时注册这些attribute。这就是面向对象中“继承”的概念
> - uevent，同理，所有相同类型的设备，会有一些共有的uevent需要发送，由该接口实现
> - devnode，devtmpfs有关的内容，暂不说明
> - release，如果device结构没有提供release接口，就要查询它所属的type是否提供。用于释放device变量所占的空间

**5.3 root device**

在sysfs中有这样一个目录：/sys/devices，系统中所有的设备，都归集在该目录下。有些设备，是通过device_register注册到Kernel并体现在/sys/devices/xxx/下。但有时候我们仅仅需要在/sys/devices/下注册一个目录，该目录不代表任何的实体设备，这时可以使用下面的接口：

1: /\* include/linux/device.h, line 859 \*/

2: /\*

3:  * Root device objects for grouping under /sys/devices

4:  \*/

5: extern struct device \*\_\_root_device_register(const char \*name,

6: struct module \*owner);

7:

8: /\*

9:  * This is a macro to avoid include problems with THIS_MODULE,

10:  * just as per what is done for device_schedule_callback() above.

11:  \*/

12: #define root_device_register(name) \\

13: \_\_root_device_register(name, THIS_MODULE)

14:

15: extern void root_device_unregister(struct device \*root);

该接口会调用device_register函数，向内核中注册一个设备，但是（你也想到了），没必要注册与之对应的driver（顺便提一下，内核中有很多不需要driver的设备，这是之一）。

_原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/linux_kenrel/device_and_driver.html)。_

标签: [Linux](http://www.wowotech.net/tag/Linux) [内核](http://www.wowotech.net/tag/%E5%86%85%E6%A0%B8) [设备模型](http://www.wowotech.net/tag/%E8%AE%BE%E5%A4%87%E6%A8%A1%E5%9E%8B) [Device](http://www.wowotech.net/tag/Device) [driver](http://www.wowotech.net/tag/driver)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Linux设备模型(6)\_Bus](http://www.wowotech.net/device_model/bus.html) | [process credentials相关的用户空间文件](http://www.wowotech.net/linux_application/24.html)»

**评论：**

**xiaotonga**\
2023-09-21 10:55

了解device和device_driver相关概念和结构，但对应device和device_driver之间怎么匹配或建立联系呢？\
1.内核解析dts(device tree source)文件获知系统存在的硬件设备及其配置；\
2.device driver 通过设备树来获取系统中硬件设备，并与其通信。\
3.device driver功能，读写设备寄存器，或调用封装设备寄存器读写的接口，实现改变设备寄存器状态、数据。\
4.device driver 通过实现vfs提供标准接口结构file_operations{.read=xxx_read;.write=xxx_write}给用户使用。

[回复](http://www.wowotech.net/device_model/device_and_driver.html#comment-8828)

**Thomas**\
2023-03-20 00:57

前辈你好，阅读你的文章受益良多。linuxer新手前来讨论。对以下两段话还是感觉有不对的地方。\
1.在"Linux设备模型(4)\_sysfs”中，我们有讲到，大多数时候，attribute文件的读写数据流为：vfs---->sysfs---->kobject---->attribute---->kobj_type---->sysfs_ops---->xxx_attribute，其中kobj_type、sysfs_ops和xxx_attribute都是由包含kobject的上层数据结构实现。\
2.至于driver的attribute，则要简单的多，其数据流为：vfs---->sysfs---->kobject---->attribute---->driver_attribute，\
PS:我阅读的是3.18的源码。\
首先1，kobject里没有attribute结构体，只能通过kobj_type---->sysfs_ops---->xxx_attribute调用show/store\
其次2，在源码drivers/base/bus.c 中，int bus_add_driver(struct device_driver *drv)函数在添加driver的时候会带入driver_ktype，driver_ktype是kobj_type，且driver也有同样的sysfs_ops 。这说明device和driver的attribute的处理方式是一样的，且所有的属性的的ops方法都是在openfile里面这样获取的\
/* every kobject with an attribute needs a ktype assigned \*/\
15:    if (kobj->ktype && kobj->ktype->sysfs_ops)\
16:        ops = kobj->ktype->sysfs_ops;\
所以driver的kobject读取设备属性的时候和device是一样的，都应该为：vfs---->sysfs---->kobject---->kobj_type---->sysfs_ops---->xxx_attribute

望赐教。

[回复](http://www.wowotech.net/device_model/device_and_driver.html#comment-8757)

**tyy**\
2020-05-06 01:28

@wowo 有个地方我不明白，在注册device时，device_add会调用发送uevent消息，让用户空间的mdev创建/dev下的文件。\
但是按照下面的描述，字符设备注册是在driver的probe里面做的。那么在注册device的时候，字符设备还没注册，字符设备都没注册，mdev怎么创建/dev下的文件呢？

所谓的"probe”，是指在Linux内核中，如果存在相同名称的device和device_driver（注：还存在其它方式，我们先不关注了），内核就会执行device_driver中的probe回调函数，而该函数就是所有driver的入口，可以执行诸如硬件设备初始化、字符设备注册、设备文件操作ops注册等动作（"remove”是它的反操作，发生在device或者device_driver任何一方从内核注销时，其原理类似，就不再单独说明了）

[回复](http://www.wowotech.net/device_model/device_and_driver.html#comment-7975)

**luffy**\
2019-07-09 21:52

感谢博主，博主能否分享一些媒体设备驱动心得。刚入门的小白对里面pipeline，dts节点图，数据流看的头大

[回复](http://www.wowotech.net/device_model/device_and_driver.html#comment-7524)

**essaypinglun 作家**\
2018-05-19 17:12

Linux设备驱动程序的平台，您可以从中检查此区域中的更好条件。您也可以用....替换最好的想法，并在这里用struct device的文章维护更好的博客。

[回复](http://www.wowotech.net/device_model/device_and_driver.html#comment-6759)

**小白**\
2017-08-15 23:40

指正一个明显错误：\
struct device_driver中成员p，私有数据的指针，具体的driver代码可以把任何需要的内容放在这里，反正设备模型代码不关心。\
这个变量是驱动模型很重要的，非常重要，因为里面包含了kobject是挂到sysfs的，不能使用

[回复](http://www.wowotech.net/device_model/device_and_driver.html#comment-5911)

**[wowo](http://www.wowotech.net/)**\
2017-08-16 08:38

@小白：多谢指正，是的，这个是driver core的私有指针，其它人不能动。已修改。多谢～～～

[回复](http://www.wowotech.net/device_model/device_and_driver.html#comment-5912)

**LINUX**\
2017-06-09 16:53

@蜗蜗 你好，现在struct device不直接使用，有被封装了一层，如soc device、platform device等等，然后在写驱动的时候，怎么都不用对这个struct device结构体进行初始化了？是不是linux内部帮我们初始化好了？

[回复](http://www.wowotech.net/device_model/device_and_driver.html#comment-5647)

**[wowo](http://www.wowotech.net/)**\
2017-06-10 14:28

@LINUX：大部分初始化好了，不过还有一些东西，driver还能看到，比如drv_data等。

[回复](http://www.wowotech.net/device_model/device_and_driver.html#comment-5650)

**Parchilor**\
2017-01-03 10:41

能不能顺便给个练手的练习，作为小白根本不知道如何下手啊！

[回复](http://www.wowotech.net/device_model/device_and_driver.html#comment-5076)

**[wowo](http://www.wowotech.net/)**\
2017-01-03 13:18

@Parchilor：这里有个例子，你可以参考一下自己写一个:-)\
https://github.com/wowotech/sparrow

[回复](http://www.wowotech.net/device_model/device_and_driver.html#comment-5077)

**虎妞**\
2017-11-16 17:53

@wowo：新手求教：\
这个例子卸载模块的时候出现的警告是不是因为sparrow是一个platform_device,但是初始化的时候没有指定底层的device，也就没有指定device的release函数。而没有release系统认为资源未释放所以报警？\
还有那个test是不是用来测试线层同步的。麻烦简单说明一下。我运行起来报错。

[回复](http://www.wowotech.net/device_model/device_and_driver.html#comment-6209)

**[wowo](http://www.wowotech.net/)**\
2017-11-16 21:33

@虎妞：警告的原因是没有指定device的release函数，而不是没有指定device。至于为什么没指定release函数，因为platform device一般都是不remove的。我们这个例子比较特殊。\
那个测试，就是随手写写，没什么目的，报错的话，你试着自己解决一下吧。

[回复](http://www.wowotech.net/device_model/device_and_driver.html#comment-6211)

**heng**\
2016-12-22 10:48

@wowo\
窝窝，这个probe我大致理解了，之前都是内核检测到有相应的设备的时候，又匹配到有相应的平台驱动。但是最近linux不是更新了设备树吗？就是用dts文件这个，我在《ARM Device Tree设备树.pdf》这个文章里面看到这样一句话，有点不理解。。：（以下是原文）\
使用 Device Tree 后,驱动需要与.dts 中描述的设备结点进行匹配,从而引发驱动的\
probe()函数执行。\
1.这段话的理解可不可以是现在就算没有设备，plattform_driver的probe也可以执行？\
2.如果不是，那么这个of_match_table里面匹配dts中的东西的时候，是什么原理？\
谢谢窝窝～

[回复](http://www.wowotech.net/device_model/device_and_driver.html#comment-5048)

**[wowo](http://www.wowotech.net/)**\
2016-12-22 10:57

@heng：driver一定要等到device才能匹配。\
dts是一种描述设备的语法，kernel在启动的时候会把dts的描述转换为实际的device。

[回复](http://www.wowotech.net/device_model/device_and_driver.html#comment-5049)

**heng**\
2016-12-22 11:32

@wowo：明白了，实际上probe的原理还是没变的。那么，如果没有实际的物理设备呢？比如，dts里面有一个wifi device的配置，但是这个wifi芯片并没有连到板子上面。\
因为我之前写了一个platform_driver,然后在dts里面加了相应的配置，最后这个driver的probe也执行了呢。很明确的是platform_driver没有真正的物理设备

[回复](http://www.wowotech.net/device_model/device_and_driver.html#comment-5051)

**[wowo](http://www.wowotech.net/)**\
2016-12-22 11:34

@heng：换个思路想一下：检查是不是有真正的物理设备，不就是driver要干的事情吗？

[回复](http://www.wowotech.net/device_model/device_and_driver.html#comment-5052)

**heng**\
2016-12-22 11:54

@wowo：啊！我傻了！对阿～driver可以先加载的。。。不一定先要有设备0.0。。

(◡¸◡✿)\
谢谢窝窝～用了dts过后纠结了好久呢～

[回复](http://www.wowotech.net/device_model/device_and_driver.html#comment-5053)

**sheeaza**\
2020-06-27 12:25

@wowo：想问下arm64 把dts转为实际device是在哪里，我找了下源码，没有找到

[回复](http://www.wowotech.net/device_model/device_and_driver.html#comment-8036)

**HHP_Low**\
2016-12-16 16:01

4. 设备驱动probe的时机 :7: Error:'prove' you mean 'probe'?\
   设备驱动prove的时机有如下几种（分为自动触发和手动触发）：\
   ^\
   2333\
   菜鸟只能找这种错误了

PS:感觉技术教程太严肃看不下去，太浮夸又营养。wowo写的感觉就很舒服～  谢谢wowo

[回复](http://www.wowotech.net/device_model/device_and_driver.html#comment-5031)

**[wowo](http://www.wowotech.net/)**\
2016-12-16 16:42

@HHP_Low：多谢指正～～\
话说能被发现的错误都不算错，哈哈:-)

[回复](http://www.wowotech.net/device_model/device_and_driver.html#comment-5033)

1 [2](http://www.wowotech.net/device_model/device_and_driver.html/comment-page-2#comments) [3](http://www.wowotech.net/device_model/device_and_driver.html/comment-page-3#comments)

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

  - [zRAM内存压缩技术原理与应用](http://www.wowotech.net/memory_management/zram.html)
  - [Linux电源管理(6)\_Generic PM之Suspend功能](http://www.wowotech.net/pm_subsystem/suspend_and_resume.html)
  - [我的bash shell学习笔记](http://www.wowotech.net/linux_application/134.html)
  - [Linux PM QoS framework(2)\_PM QoS class](http://www.wowotech.net/pm_subsystem/pm_qos_class.html)
  - [Linux cpuidle framework(4)\_menu governor](http://www.wowotech.net/pm_subsystem/cpuidle_menu_governor.html)

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

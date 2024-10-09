作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2014-3-7 0:25 分类：[统一设备模型](http://www.wowotech.net/sort/device_model)

#### 1. 前言

Kobject是Linux设备模型的基础，也是设备模型中最难理解的一部分（可参考Documentation/kobject.txt的表述）。因此有必要先把它分析清楚。

#### 2. 基本概念

由“[Linux设备模型(1)\_基本概念](http://www.wowotech.net/linux_kenrel/13.html)”可知，Linux设备模型的核心是使用Bus、Class、Device、Driver四个核心数据结构，将大量的、不同功能的硬件设备（以及驱动该硬件设备的方法），以树状结构的形式，进行归纳、抽象，从而方便Kernel的统一管理。

而硬件设备的数量、种类是非常多的，这就决定了Kernel中将会有大量的有关设备模型的数据结构。这些数据结构一定有一些共同的功能，需要抽象出来统一实现，否则就会不可避免的产生冗余代码。这就是Kobject诞生的背景。

目前为止，Kobject主要提供如下功能：

1. 通过parent指针，可以将所有Kobject以层次结构的形式组合起来。
1. 使用一个引用计数（reference count），来记录Kobject被引用的次数，并在引用次数变为0时把它释放（这是Kobject诞生时的唯一功能）。
1. 和sysfs虚拟文件系统配合，将每一个Kobject及其特性，以文件的形式，开放到用户空间（有关sysfs，会在其它文章中专门描述，本文不会涉及太多内容）。

注1：在Linux中，Kobject几乎不会单独存在。它的主要功能，就是内嵌在一个大型的数据结构中，为这个数据结构提供一些底层的功能实现。\
注2：Linux driver开发者，很少会直接使用Kobject以及它提供的接口，而是使用构建在Kobject之上的设备模型接口。

#### 3. 代码解析

##### 3.1 在Linux Kernel source code中的位置

在Kernel源代码中，Kobject由如下两个文件实现：

- include/linux/kobject.h
- lib/kobject.c

其中kobject.h为Kobject的头文件，包含所有的数据结构定义和接口声明。kobject.c为核心功能的实现。

##### 3.2 主要的数据结构

在描述数据结构之前，有必要说明一下Kobject, Kset和Ktype这三个概念。

Kobject是基本数据类型，每个Kobject都会在"/sys/“文件系统中以目录的形式出现。

Ktype代表Kobject（严格地讲，是包含了Kobject的数据结构）的属性操作集合（由于通用性，多个Kobject可能共用同一个属性操作集，因此把Ktype独立出来了）。\
注3：在设备模型中，ktype的命名和解释，都非常抽象，理解起来非常困难，后面会详细说明。

Kset是一个特殊的Kobject（因此它也会在"/sys/“文件系统中以目录的形式出现），它用来集合相似的Kobject（这些Kobject可以是相同属性的，也可以不同属性的）。

- 首先看一下Kobject的原型

```cpp
 1: /* Kobject: include/linux/kobject.h line 60 */
 2: struct kobject {
 3:     const char *name;
 4:     struct list_head    entry;
 5:     struct kobject      *parent;
 6:     struct kset     *kset;
 7:     struct kobj_type    *ktype;
 8:     struct sysfs_dirent *sd;
 9:     struct kref     kref;
 10:    unsigned int state_initialized:1;
 11:    unsigned int state_in_sysfs:1;
 12:    unsigned int state_add_uevent_sent:1;
 13:    unsigned int state_remove_uevent_sent:1;
 14:    unsigned int uevent_suppress:1;
 15: };
```

> name，该Kobject的名称，同时也是sysfs中的目录名称。由于Kobject添加到Kernel时，需要根据名字注册到sysfs中，之后就不能再直接修改该字段。如果需要修改Kobject的名字，需要调用kobject_rename接口，该接口会主动处理sysfs的相关事宜。
>
> entry，用于将Kobject加入到Kset中的list_head。
>
> parent，指向parent kobject，以此形成层次结构（在sysfs就表现为目录结构）。
>
> kset，该kobject属于的Kset。可以为NULL。如果存在，且没有指定parent，则会把Kset作为parent（别忘了Kset是一个特殊的Kobject）。
>
> ktype，该Kobject属于的kobj_type。每个Kobject必须有一个ktype，或者Kernel会提示错误。
>
> sd，该Kobject在sysfs中的表示。
>
> kref，"struct kref”类型（在include/linux/kref.h中定义）的变量，为一个可用于原子操作的引用计数。
>
> state_initialized，指示该Kobject是否已经初始化，以在Kobject的Init，Put，Add等操作时进行异常校验。
>
> state_in_sysfs，指示该Kobject是否已在sysfs中呈现，以便在自动注销时从sysfs中移除。
>
> state_add_uevent_sent/state_remove_uevent_sent，记录是否已经向用户空间发送ADD uevent，如果有，且没有发送remove uevent，则在自动注销时，补发REMOVE uevent，以便让用户空间正确处理。
>
> uevent_suppress，如果该字段为1，则表示忽略所有上报的uevent事件。
>
> 注4：Uevent提供了“用户空间通知”的功能实现，通过该功能，当内核中有Kobject的增加、删除、修改等动作时，会通知用户空间。有关该功能的具体内容，会在其它文章详细描述。

- Kset的原型为

```cpp
 1: /* include/linux/kobject.h, line 159 */
 2: struct kset {
 3:     struct list_head list;
 4:     spinlock_t list_lock;
 5:     struct kobject kobj;
 6:     const struct kset_uevent_ops *uevent_ops;
 7: };
```

list/list_lock，用于保存该kset下所有的kobject的链表。

kobj，该kset自己的kobject（kset是一个特殊的kobject，也会在sysfs中以目录的形式体现）。

uevent_ops，该kset的uevent操作函数集。当任何Kobject需要上报uevent时，都要调用它所从属的kset的uevent_ops，添加环境变量，或者过滤event（kset可以决定哪些event可以上报）。因此，如果一个kobject不属于任何kset时，是不允许发送uevent的。

- Ktype的原型为

```cpp
 1: /* include/linux/kobject.h, line 108 */
 2: struct kobj_type {
 3:     void (*release)(struct kobject *kobj);
 4:     const struct sysfs_ops *sysfs_ops;
 5:     struct attribute **default_attrs;
 6:     const struct kobj_ns_type_operations *(*child_ns_type)(struct kobject *kobj);
 7:     const void *(*namespace)(struct kobject *kobj);
 8: };
```

> release，通过该回调函数，可以将包含该种类型kobject的数据结构的内存空间释放掉。
>
> sysfs_ops，该种类型的Kobject的sysfs文件系统接口。
>
> default_attrs，该种类型的Kobject的atrribute列表（所谓attribute，就是sysfs文件系统中的一个文件）。将会在Kobject添加到内核时，一并注册到sysfs中。
>
> child_ns_type/namespace，和文件系统（sysfs）的命名空间有关，这里不再详细说明。

|                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| \*\*总结，Ktype以及整个Kobject机制的理解。  <br>\*\*Kobject的核心功能是：保持一个引用计数，当该计数减为0时，自动释放（由本文所讲的kobject模块负责） Kobject所占用的meomry空间。这就决定了Kobject必须是动态分配的（只有这样才能动态释放）。  <br>  <br>而Kobject大多数的使用场景，是内嵌在大型的数据结构中（如Kset、device_driver等），因此这些大型的数据结构，也必须是动态分配、动态释放的。那么释放的时机是什么呢？是内嵌的Kobject释放时。但是Kobject的释放是由Kobject模块自动完成的（在引用计数为0时），那么怎么一并释放包含自己的大型数据结构呢？  <br>  <br>这时Ktype就派上用场了。我们知道，Ktype中的release回调函数负责释放Kobject（甚至是包含Kobject的数据结构）的内存空间，那么Ktype及其内部函数，是由谁实现呢？是由上层数据结构所在的模块！因为只有它，才清楚Kobject嵌在哪个数据结构中，并通过Kobject指针以及自身的数据结构类型，找到需要释放的上层数据结构的指针，然后释放它。  <br>  <br>讲到这里，就清晰多了。所以，每一个内嵌Kobject的数据结构，例如kset、device、device_driver等等，都要实现一个Ktype，并定义其中的回调函数。同理，sysfs相关的操作也一样，必须经过ktype的中转，因为sysfs看到的是Kobject，而真正的文件操作的主体，是内嵌Kobject的上层数据结构！  <br>  <br>  <br>顺便提一下，Kobject是面向对象的思想在Linux kernel中的极致体现，但C语言的优势却不在这里，所以Linux kernel需要用比较巧妙（也很啰嗦）的手段去实现， |

##### 3.3 功能分析

**3.3.1 Kobject使用流程**

Kobject大多数情况下（有一种例外，下面会讲）会嵌在其它数据结构中使用，其使用流程如下：

1. 定义一个struct kset类型的指针，并在初始化时为它分配空间，添加到内核中
1. 根据实际情况，定义自己所需的数据结构原型，该数据结构中包含有Kobject
1. 定义一个适合自己的ktype，并实现其中回调函数
1. 在需要使用到包含Kobject的数据结构时，动态分配该数据结构，并分配Kobject空间，添加到内核中
1. 每一次引用数据结构时，调用kobject_get接口增加引用计数；引用结束时，调用kobject_put接口，减少引用计数
1. 当引用计数减少为0时，Kobject模块调用ktype所提供的release接口，释放上层数据结构以及Kobject的内存空间

上面有提过，有一种例外，Kobject不再嵌在其它数据结构中，可以单独使用，这个例外就是：开发者只需要在sysfs中创建一个目录，而不需要其它的kset、ktype的操作。这时可以直接调用kobject_create_and_add接口，分配一个kobject结构并把它添加到kernel中。

**3.3.2 Kobject的分配和释放**

前面讲过，Kobject必须动态分配，而不能静态定义或者位于堆栈之上，它的分配方法有两种。

1. 通过kmalloc自行分配（一般是跟随上层数据结构分配），并在初始化后添加到kernel。这种方法涉及如下接口：

```cpp
 1: /* include/linux/kobject.h, line 85 */
 2: extern void kobject_init(struct kobject *kobj, struct kobj_type *ktype);
 3: extern __printf(3, 4) __must_check
 4: int kobject_add(struct kobject *kobj, struct kobject *parent,
 5:                 const char *fmt, ...);
 6: extern __printf(4, 5) __must_check
 7: int kobject_init_and_add(struct kobject *kobj,
 8:             struct kobj_type *ktype, struct kobject *parent,
 9:             const char *fmt, ...);
```

> kobject_init，初始化通过kmalloc等内存分配函数获得的struct kobject指针。主要执行逻辑为：
>
> - 确认kobj和ktype不为空
> - 如果该指针已经初始化过（判断kobj->state_initialized），打印错误提示及堆栈信息（但不是致命错误，所以还可以继续）
> - 初始化kobj内部的参数，包括引用计数、list、各种标志等
> - 根据输入参数，将ktype指针赋予kobj->ktype
>
> kobject_add，将初始化完成的kobject添加到kernel中，参数包括需要添加的kobject、该kobject的parent（用于形成层次结构，可以为空）、用于提供kobject name的格式化字符串。主要执行逻辑为：
>
> - 确认kobj不为空，确认kobj已经初始化，否则错误退出
> - 调用内部接口kobject_add_varg，完成添加操作
>
> kobject_init_and_add，是上面两个接口的组合，不再说明。
>
> ==========================内部接口======================================
>
> kobject_add_varg，解析格式化字符串，将结果赋予kobj->name，之后调用kobject_add_internal接口，完成真正的添加操作。
>
> kobject_add_internal，将kobject添加到kernel。主要执行逻辑为：
>
> - 校验kobj以及kobj->name的合法性，若不合法打印错误信息并退出
> - 调用kobject_get增加该kobject的parent的引用计数（如果存在parent的话）
> - 如果存在kset（即kobj->kset不为空），则调用kobj_kset_join接口加入kset。同时，如果该kobject没有parent，却存在kset，则将它的parent设为kset（kset是一个特殊的kobject），并增加kset的引用计数
> - 通过create_dir接口，调用sysfs的相关接口，在sysfs下创建该kobject对应的目录
> - 如果创建失败，执行后续的回滚操作，否则将kobj->state_in_sysfs置为1
>
> kobj_kset_join，负责将kobj加入到对应kset的链表中。

这种方式分配的kobject，会在引用计数变为0时，由kobject_put调用其ktype的release接口，释放内存空间，具体可参考后面有关kobject_put的讲解。

2. 使用kobject_create创建

Kobject模块可以使用kobject_create自行分配空间，并内置了一个ktype（dynamic_kobj_ktype），用于在计数为0是释放空间。代码如下：

```cpp
 1: /* include/linux/kobject.h, line 96 */
 2: extern struct kobject * __must_check kobject_create(void);
 3: extern struct kobject * __must_check kobject_create_and_add(const char *name,
 4:             struct kobject *parent);
 1: /* lib/kobject.c, line 605 */
 2: static void dynamic_kobj_release(struct kobject *kobj)
 3: {
 4:     pr_debug("kobject: (%p): %s\n", kobj, __func__);
 5:     kfree(kobj);
 6: }
 7:  
 8: static struct kobj_type dynamic_kobj_ktype = {
 9:     .release    = dynamic_kobj_release,
 10:    .sysfs_ops  = &kobj_sysfs_ops,
 11: };
```

> kobject_create，该接口为kobj分配内存空间，并以dynamic_kobj_ktype为参数，调用kobject_init接口，完成后续的初始化操作。
>
> kobject_create_and_add，是kobject_create和kobject_add的组合，不再说明。
>
> dynamic_kobj_release，直接调用kfree释放kobj的空间。

**3.3.3 Kobject引用计数的修改**

通过kobject_get和kobject_put可以修改kobject的引用计数，并在计数为0时，调用ktype的release接口，释放占用空间。

```cpp
 1: /* include/linux/kobject.h, line 103 */
 2: extern struct kobject *kobject_get(struct kobject *kobj);
 3: extern void kobject_put(struct kobject *kobj);
```

> kobject_get，调用kref_get，增加引用计数。
>
> kobject_put，以内部接口kobject_release为参数，调用kref_put。kref模块会在引用计数为零时，调用kobject_release。
>
> ==========================内部接口======================================
>
> kobject_release，通过kref结构，获取kobject指针，并调用kobject_cleanup接口继续。
>
> kobject_cleanup，负责释放kobject占用的空间，主要执行逻辑如下：
>
> - 检查该kobject是否有ktype，如果没有，打印警告信息
> - 如果该kobject向用户空间发送了ADD uevent但没有发送REMOVE uevent，补发REMOVE uevent
> - 如果该kobject有在sysfs文件系统注册，调用kobject_del接口，删除它在sysfs中的注册
> - 调用该kobject的ktype的release接口，释放内存空间
> - 释放该kobject的name所占用的内存空间

**3.3.4 Kset的初始化、注册**

Kset是一个特殊的kobject，因此其初始化、注册等操作也会调用kobject的相关接口，除此之外，会有它特有的部分。另外，和Kobject一样，kset的内存分配，可以由上层软件通过kmalloc自行分配，也可以由Kobject模块负责分配，具体如下。

```cpp
 1: /* include/linux/kobject.h, line 166 */
 2: extern void kset_init(struct kset *kset);
 3: extern int __must_check kset_register(struct kset *kset);
 4: extern void kset_unregister(struct kset *kset);
 5: extern struct kset * __must_check kset_create_and_add(const char *name,
 6:             const struct kset_uevent_ops *u,
 7:             struct kobject *parent_kobj);
```

> kset_init，该接口用于初始化已分配的kset，主要包括调用kobject_init_internal初始化其kobject，然后初始化kset的链表。需要注意的时，如果使用此接口，上层软件必须提供该kset中的kobject的ktype。
>
> kset_register，先调用kset_init，然后调用kobject_add_internal将其kobject添加到kernel。
>
> kset_unregister，直接调用kobject_put释放其kobject。当其kobject的引用计数为0时，即调用ktype的release接口释放kset占用的空间。
>
> kset_create_and_add，会调用内部接口kset_create动态创建一个kset，并调用kset_register将其注册到kernel。
>
> ==========================内部接口======================================
>
> kset_create，该接口使用kzalloc分配一个kset空间，并定义一个kset_ktype类型的ktype，用于释放所有由它分配的kset空间。

_原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/linux_kenrel/kobject.html)。_

标签: [Linux](http://www.wowotech.net/tag/Linux) [内核](http://www.wowotech.net/tag/%E5%86%85%E6%A0%B8) [设备模型](http://www.wowotech.net/tag/%E8%AE%BE%E5%A4%87%E6%A8%A1%E5%9E%8B) [kobject](http://www.wowotech.net/tag/kobject) [ktype](http://www.wowotech.net/tag/ktype) [kset](http://www.wowotech.net/tag/kset)

______________________________________________________________________

« [调试手段之sys节点](http://www.wowotech.net/linux_application/15.html) | [Linux设备模型(1)\_基本概念](http://www.wowotech.net/device_model/13.html)»

**评论：**

**xiaotongs**\
2023-09-14 13:18

kobject,内核基础设施，怎么和device（抽象设备）关联？怎么和driver关联？内核怎么通过kobject调用到对应的device？

1. kobject会设备文件的形式在“sys”下出现，ket是kobject的集合，会将相似的kobj集合起来，在sys下也会以设备文件形式显示。\
   2.ktype包含kobj数据结构中属性的集合，有释放kobject数据结构内存占用接口release。\
   3.每个包含kobject结构，如bus、device、devie_driver都需实现ktype,并实现其回调函数release，而sysfs_ops相关操作必须通过ktype中接口中转。因为sysfs看到是kobject对应的设备文件，真正操作文件的主体是包含kobj结构的上层数据结构，它才知道什么时候释放和操作设备文件，并调用kobj中ktype相关接口实现释放内存和调用文件接口。
1. syscall->sysfs->fs_ops->callback->kobject->device

[回复](http://www.wowotech.net/device_model/kobject.html#comment-8821)

**devin**\
2023-03-18 15:06

动态创建\
"前面讲过，Kobject必须动态分配，而不能静态定义或者位于堆栈之上，它的分配方法有两种。"这里笔误了吗？

[回复](http://www.wowotech.net/device_model/kobject.html#comment-8755)

**hulianqing**\
2023-04-10 18:03

@devin：我的理解是：1、把kobject定义在你的自定义结构体内，然后自定义结构体动态分配 2、直接单独分配kobject

[回复](http://www.wowotech.net/device_model/kobject.html#comment-8768)

**xiaotongs**\
2023-09-14 15:12

@devin：1.动态创建：在内核空间动态分配内存。\
2.而不能静态定义或者位于堆栈之上：静态变量区、堆、栈是用户空间，用户空间分配内存需用通过C库或系统调用分配内存，实际上，用户空间分配的物理内存需通过内核来分配；

[回复](http://www.wowotech.net/device_model/kobject.html#comment-8822)

**xpo**\
2020-12-18 12:03

kset是 chain of responsibity 设计模式组织的kobject. 和 SEH, message proc是相同的设计,只不过要简陋的多.

[回复](http://www.wowotech.net/device_model/kobject.html#comment-8160)

**jams-cui**\
2019-08-13 09:33

个人觉得kset正如wowo所言，是后续添加的。它属于一个大的子项的共性“行为”接口，用于底层和应用层互动的行为。也可以理解为热插拔之类。。。kobject和ktype偏向于个体，描述一个对象自身的行为，如创立，释放，以及这个个体对应于sysfs这种设备模型下各种属性的预留管理接口

[回复](http://www.wowotech.net/device_model/kobject.html#comment-7586)

**Mike_5**\
2019-07-19 22:29

同时，如果该kobject没有parent，却存在kset，则将它的parent设为kset（kset是一个特殊的kobject），并增加kset的引用计数

这句话后面是不是说错了，应该是kset设为它的parent

[回复](http://www.wowotech.net/device_model/kobject.html#comment-7542)

**Xavior**\
2020-04-21 17:38

@Mike_5：If a kset is associated with a kobject, then the parent for the kobject can be set to NULL in the call to kobject_add() and then the kobject's parent will be the kset itself.

不需要设置吧？

[回复](http://www.wowotech.net/device_model/kobject.html#comment-7966)

**ask_dir**\
2019-07-01 14:52

@wowo :文中讲到注册kobject的时候，其中kobj_type中有一个default_attrs，该种类型的Kobject的atrribute列表（所谓attribute，就是sysfs文件系统中的一个文件）。这个attribute属性好像只有name和mode两个成员，但是我实际在sysfs目录下的属性好像是可以实现读写操作的啊，请问这里面读写操作该文件的时候是调用的是kobject的读写函数吗？

[回复](http://www.wowotech.net/device_model/kobject.html#comment-7498)

**[wowo](http://www.wowotech.net/)**\
2019-07-02 14:02

@ask_dir：设备模型其他的文章里面有讲，你可以再看看~

[回复](http://www.wowotech.net/device_model/kobject.html#comment-7504)

**noobwang**\
2020-11-18 16:57

@ask_dir：驱动里定义一个str，echo会调用ktype->sysfs_ops->store，可以给str赋值；cat会调用ktype->sysfs_ops->show, 显示str的值。我说错了的话大家指正我喔：）

[回复](http://www.wowotech.net/device_model/kobject.html#comment-8139)

**tmmdh**\
2019-03-04 15:08

入行1年发现这个博客，真是相见恨晚，文章好，评论也好

[回复](http://www.wowotech.net/device_model/kobject.html#comment-7216)

**kobj**\
2017-06-07 22:55

“调用kobject_get增加该kobject的parent的引用计数（如果存在parent的话）。”\
调用kobject_get不应该是增加参数自己的引用计数么，为什么是parent的呢？源码：kobject_get(struct kobject \*kobj) -> kref_get(&kobj->kref) -> WARN_ON_ONCE(atomic_inc_return(&kref->refcount) \< 2)；

[回复](http://www.wowotech.net/device_model/kobject.html#comment-5635)

**[wowo](http://www.wowotech.net/)**\
2017-06-08 09:25

@kobj：可以看kobject_add_internal的代码：\
parent = kobject_get(kobj->parent);

为什么要增加parent的引用计数呢？加入该obj在使用，能删除parent吗？不能，所以要确保parent的引用计数不为0（不被删除）。

[回复](http://www.wowotech.net/device_model/kobject.html#comment-5638)

**kobj**\
2017-06-08 09:55

@wowo：哦，的确是这样的。感谢wowo，祝福本站。

[回复](http://www.wowotech.net/device_model/kobject.html#comment-5641)

**loren**\
2017-05-09 20:34

楼主，在3.3.2节的第一种方法中，如果我定义了一个ktype,但是其中的release回调函数没有定义（或者设置为NULL），这种case下使用这种方法来创建kobject是否会有问题呢？因为我看kobject_init()和kobject_add()两个函数都没有去检查ktype.release是否为空，而只是检查了ktype是否为空，不知道是不是我哪里理解有问题。\
BTW：我看的是3.16.0的内核

[回复](http://www.wowotech.net/device_model/kobject.html#comment-5530)

**[wowo](http://www.wowotech.net/)**\
2017-05-10 08:27

@loren：release的检查，只有在kobject_cleanup时，做了一次，如果为空，会给出一个debug信息，要求fix：\
if (t && !t->release)                                                  \
pr_debug("kobject: '%s' (%p): does not have a release() "      \
"function, it is broken and must be fixed.\\n",        \
kobject_name(kobj), kobj);    \
至于其它的，要有开发者自己把握了。

[回复](http://www.wowotech.net/device_model/kobject.html#comment-5531)

**[tinylaker](http://www.wowotech.net/)**\
2016-05-03 20:07

好文点赞，如果可以添加设备模型的结构图进行说明就更完美了

[回复](http://www.wowotech.net/device_model/kobject.html#comment-3879)

**[wowo](http://www.wowotech.net/)**\
2016-05-03 21:41

@tinylaker：多谢建议，下次整理的时候，一定填上！

[回复](http://www.wowotech.net/device_model/kobject.html#comment-3881)

**maze**\
2016-07-27 09:30

@wowo：http://pic002.cnblogs.com/images/2012/314660/2012012909402335.gif\
http://pic002.cnblogs.com/images/2012/314660/2012012909435638.gif\
不知道这两张图对您是否有帮助

[回复](http://www.wowotech.net/device_model/kobject.html#comment-4324)

**[wowo](http://www.wowotech.net/)**\
2016-07-27 22:03

@maze：多谢分享，图画的很好:-)

[回复](http://www.wowotech.net/device_model/kobject.html#comment-4328)

**ymj**\
2019-08-03 22:35

@wowo：maze发的图2,ksetB 向右的箭头方向反了吧,因该都指向ksetA.

[回复](http://www.wowotech.net/device_model/kobject.html#comment-7571)

1 [2](http://www.wowotech.net/device_model/kobject.html/comment-page-2#comments) [3](http://www.wowotech.net/device_model/kobject.html/comment-page-3#comments)

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

  - [支持wowo，有这样一块小天地享受技术，感觉棒棒哒！](http://www.wowotech.net/147.html)
  - [Linux的时钟](http://www.wowotech.net/timer_subsystem/clock-id-in-linux.html)
  - [arm64 linux移植](http://www.wowotech.net/100.html)
  - [Dynamic DMA mapping Guide](http://www.wowotech.net/memory_management/DMA-Mapping-api.html)
  - [Linux时间子系统之（十三）：Tick Device layer综述](http://www.wowotech.net/timer_subsystem/tick-device-layer.html)

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

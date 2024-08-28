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

作者：[callme_friend](http://www.wowotech.net/author/148) 发布于：2018-1-9 18:37 分类：[统一设备模型](http://www.wowotech.net/sort/device_model)

kobj/kset作为统一设备模型的基础，到底提供了哪些功能，在具体应用过程中，如device、bus甚至platform_device等是如何使用kobj/kset的，这是本文的主要阐述内容。

作为阅读wowo相关文章后的笔记，本文纰漏之处，欢迎各位大侠拍砖。

**1 kobj****实现**

**1.1 kobject**

> struct kobject {
> 
>          const char                  *name;
> 
>          struct list_head        entry;
> 
>          struct kobject           *parent;
> 
>          struct kset                  *kset;
> 
>          struct kobj_type       *ktype;
> 
>          struct kernfs_node  *sd;
> 
>          struct kref                  kref;
> 
>          unsigned int   state_initialized:1;
> 
>          unsigned int   state_in_sysfs:1;
> 
>          unsigned int   state_add_uevent_sent:1;
> 
>          unsigned int   state_remove_uevent_sent:1;
> 
>          unsigned int uevent_suppress:1;
> 
> };

name：对应sysfs的目录名。

entry：用于将kobj挂在kset->list中。

parent：指向kobj的父结构，形成层次结构，在sysfs中表现为父子目录的关系。

kset：表征该kobj所属的kset。kset可以作为parent的“候补”：当注册时，传入的parent为空时，可以让kset来担当。

ktype：该kobj对应的kobj_type。每个kobj或其嵌入的结构对象应该都对应一个kobj_type。

sd：对应sysfs对象。在3.14以后的内核中，sysfs基于kernfs来实现。

kref：引用计数对象，支撑kobj的引用计数功能。

state_initialized:1-------------------记录初始化与否。调用kobject_init()后，会置位。

state_in_sysfs:1---------------------记录kobj是否注册到sysfs，在kobject_add_internal()中置位。

state_add_uevent_sent:1--------当发送KOBJ_ADD消息时，置位。提示已经向用户空间发送ADD消息。

state_remove_uevent_sent:1---当发送KOBJ_REMOVE消息时，置位。提示已经向用户空间发送REMOVE消息。

uevent_suppress:1-----------------如果该字段为1，则表示忽略所有上报的uevent事件。

**1.2 kobj_type**

> struct kobj_type {
> 
>          void (*release)(struct kobject *kobj);
> 
>          const struct sysfs_ops *sysfs_ops;
> 
>          struct attribute **default_attrs;
> 
>          const struct kobj_ns_type_operations *(*child_ns_type)(struct kobject *kobj);
> 
>          const void *(*namespace)(struct kobject *kobj);
> 
> };

release：处理对象终结的回调函数。该接口应该由具体对象负责填充。

sysfs_ops：该类型kobj的sysfs操作接口。

default_attrs：该类型kobj自带的缺省属性（文件），这些属性文件在注册kobj时，直接pop为该目录下的文件。

child_ns_type/namespace：文件系统命名空间相关，不分析。

**2 kset****实现**

> struct kset {
> 
>          struct list_head list;
> 
>          spinlock_t list_lock;
> 
>          struct kobject kobj;
> 
>          const struct kset_uevent_ops *uevent_ops;
> 
> };

head_list：与kobj->entry对应，用来组织本kset管理的kobj。

kobj：kset内部包含一个kobj对象。

uevent_ops：kset用于发送消息的操作函数集。需要指出的是，kset能够发送它所包含的各种子kobj、孙kobj的消息，即kobj或其父辈、爷爷辈，都可以发送消息；优先父辈，然后是爷爷辈，以此类推。 

**3 kobj/kset****功能特性**

**3.1****对象生命周期管理**

在创建一个kobj对象时，kobj中的引用计数管理成员kref被初始化为1；从此kobj可以使用下面的API函数来进行生命周期管理：

> struct kobject *kobject_get(struct kobject *kobj)
> 
> void kobject_put(struct kobject *kobj)

对于kobject_get()，它就是直接使用kref_get()接口来对引用计数进行加1操作；

而对于kobject_put()，它的实现复杂。其不仅要使用kref_put()接口来对引用计数进行减1操作，还要对生命终结的对象执行release()操作。然而kobject是高度抽象的实体，导致kobject不会单独使用，而是嵌在具体对象中。反过来也可以这样理解：**凡是需要做对象生命周期管理的对象，都可以通过内嵌****kobject****来实现需求**。

回到kobject_put()，它通常被具体对象做一个简单包装，如bus_put()，它直接调用kset_put()，然后调用到kobject_put()。那对于这个bus_type对象而言，仅仅通过kobject_put()，如何来达到释放整个bus_type的目的呢？这里就需要kobject另一个成员struct kobj_type * ktype来完成。

回到kobject_put()的release()操作。当引用计数为0时，kobject核心会调用kobject_release()，最后会调用kobj_type->release(kobj)来完成对象的释放。可是具体对象的释放，最后却通过kobj->kobj_type->release()来释放，那这个release()函数，就必须得由具体的对象来指定。还是拿bus_type举例，在通过bus_register(struct bus_type *bus)进行总线注册时，该API内部会执行priv->subsys.kobj.ktype = &bus_ktype操作，有了该操作，那么前面的bus_put()在执行bus_type->p-> subsys.kobj->ktype->release()时，就会执行上面注册的bus_ktype.release = bus_release函数，由于bus_release()函数由具体的bus子系统提供，它必定知道如何释放包括kobj在内的bus_type对象。 

**3.2sysfs****文件系统的层次组织**

kobject的另一个功能就是完成sysfs文件系统的组织功能。该功能依靠kobj的parent、kset、sd等成员来完成。sysfs文件系统能够以友好的界面，将kobj所刻画的对象层次、所属关系、属性值，展现在用户空间。 

  

**3.3****用户空间事件投递**

这方面的内容可参考《http://www.wowotech.net/device_model/uevent.html》，该博文已经详细的说明了用户空间事件投递。

具体对象有事件发生时，相应的操作函数中（如device_add()），会调用事件消息接口kobject_uevent()，在该接口中，首先会添加一些共性的消息（路径、子系统名等），然后会找寻合适的kset->uevent_ops，并调用该ops->uevent(kset, kobj, env)来添加该对象特有的事件消息（如device对象的设备号、设备名、驱动名、DT信息），一切准备完毕，就会通过两种可能的方法向用户空间发送消息：1.netlink广播；2. uevent_helper程序。

因此，上面具体对象的事件消息填充函数，应该由特定对象来填充；对于device对象来说，在初始化的时候，通过下面这行代码：

**devices_kset = kset_create_and_add("devices", &device_uevent_ops, NULL);**

来完成kset->uevent_ops的指定，今后所有设备注册时，调用device_register()-->device_initialize()后，都将导致：

**dev->kobj.kset = devices_kset;**

因此通过device_register()注册的设备，在调用kobject_uevent()接口发送事件消息时，就自动会调用devices_kset的device_uevent_ops。该ops的uevent()方法定义如下：

> static const struct kset_uevent_ops device_uevent_ops = {
> 
>          .filter =     dev_uevent_filter,
> 
>          .name =             dev_uevent_name,
> 
>          .uevent = dev_uevent,
> 
> };                ||
> 
>                   \/
> 
> dev_uevent(struct kset *kset, struct kobject *kobj, struct kobj_uevent_env *env)
> 
>          add_uevent_var(env, "xxxxxxx", ....)
> 
>          add_uevent_var(env, "xxxxxxx", ....)
> 
>          add_uevent_var(env, "xxxxxxx", ....)
> 
>          ....
> 
>          if (dev->bus && dev->bus->uevent)
> 
>                    dev->bus->uevent(dev, env)       //通过总线的uevent()方法，发送设备状态改变的事件消息
> 
>          if (dev->class && dev->class->dev_uevent)
> 
>                    dev->class->dev_uevent(dev, env)
> 
>          if (dev->type && dev->type->uevent)
> 
>                    dev->type->uevent(dev, env)

在该ops的uevent()方法中，会分别调用bus、class、device type的uevent方法来生成事件消息。 

**3.4kobj****、****kset****关系总结**

**kobj****和kset****并不是完全的父子关系**。kset算是kobj的“接盘侠”，当kobj没有所属的parent时，才让kset来接盘当parent；如果连kset也没有，那该kobj属于顶层对象，其sysfs目录将位于/sys/下。正因为kobj和kset并不是完全的父子关系，因此在注册kobj时，将同时对parent及其所属的kset增加引用计数。若parent和kset为同一对象，则会对kset增加两次引用计数。

**kset****内部本身也包含一个kobj****对象，在sysfs****中也表现为目录**；所不同的是，**kset****要承担kobj****状态变动消息的发送任务**。因此，首先kset会将所属的kobj组织在kset.list下，同时，通过uevent_ops在合适时候发送消息。

对于**_kobject_add()_**来说，它的输入信息是：kobj-parent、kobj-name，**_kobject_add()_**优先使用传入的**_parent_**作为**_kobj->parent_**；其次，使用**_kset_**作为**_kobj->parent_**。

kobj状态变动后，必须依靠所关联的kset来向用户空间发送消息；若无关联kset（**该****kobj****向上组成的树中，任何成员都无所属的kset**），则kobj无法发送用户消息。 

**4 kset****和****kobj****的注册总结**

**4.1 API**

> **kobject_add(struct kobject *kobj, struct kobject *parent, const char *fmt, ...)**

说明：（1）kobj->parent = parent；（2）kobject_add_internal(kobj)。其中，kobject_add_internal(kobj)会根据kobj.parent和kobj.kset来综合决定父目录。详见下面总结。

> **kset_register(struct kset *k)**

说明：直接调用kobject_add_internal()进行注册，然后用kobject_uevent(&k->kobj, KOBJ_ADD)发送KOBJ_ADD事件。

> **kobject_uevent(struct kobject *kobj, enum kobject_action action)**

说明：向用户空间发送事件消息。

**4.2****总结**

**关于kobject_add()****的总结：**

在进行kobj注册时，调用**_kobject_add(struct kobject *kobj, struct kobject *parent, const char *fmt, ...)_**

**输入信息**：_kobj__对象、kobj-parent__、kobj-name_

> 当kobj，无parent、无kset时，将在sysfs的根目录(即/sys/)下创建目录；
> 
> 当kobj，无parent、有kset时，kobject_add()会设置kobj->parent为kset->kobj；因此会在该kset下创建目录；该kobj会加入kset.list；同时，会对kobj->kset->kobj增加两次引用：1.增加对kobj->parent的引用；2.增加对kobj->kset的引用。
> 
> 当kobj，有parent、无kset时，kobject_add()会在该parent下创建目录；同时，会对parent增加引用。
> 
> 当kobj，有parent、有kset时，优先在parent下创建目录；该kobj会加入kset.list；同时，会分别对parent、kset增加引用计数。

（1）kobj/kset的引用计数示例

下图展示了一个顶层kobj/kset，通过不断的添加子节点，导致的引用计数变化情况。

[![1](http://www.wowotech.net/content/uploadfile/201801/b04d1fab75dac2f6b1e003d0235aee3c20180110005611.gif "1")](http://www.wowotech.net/content/uploadfile/201801/cba745ebb78753759f3af4dcb9b10bc720180110005611.gif)

首先在（I）中建立了顶层对象A，其kref初始化为1；在第（II）步中，建立了A的子对象B，此时A对象的kref引用计数加1变为2；在第（III）步中，继续在A下建立子对象C，此时A对象的kref引用计数加1变为3；在第（IV）步中，建立B对象的子对象D，此时，只会对D的父对象进行引用计数加1；而对更上层的父对象，则不进行引用加1操作。

（2）platform_bus_type的注册示例

内核启动时，会在初始化阶段进行platform_bus_type总线的注册：bus_register(&platform_bus_type)。在该函数里，会设置platform_bus_type->p->subsys.kobj.kset = bus_kset、platform_bus_type->p->subsys.kobj.ktype = &bus_ktype，因此按照前面的总结，由于platform_bus_type有kset、无parent，导致该bus注册进系统后，其kset引用计数将被增加两次，同时，在bus_kset对应的目录（/sys/bus/）下建立platform_bus_type对应的子目录（/sys/bus/platform/）。同理，当该总线的引用计数为0时，将导致该对象生命的终结，会触发release()操作，显然该操作将导致bus_kset的引用计数减少两次。

**5** **对外接口的总结**

[![2](http://www.wowotech.net/content/uploadfile/201801/dbc59bfffd0315275d9a4d7b5e89349120180110012115.gif "2")](http://www.wowotech.net/content/uploadfile/201801/c18ad38a53a1acc60e6aa74fde9aaee620180110012111.gif)  示意图如下图所示。

[![4](http://www.wowotech.net/content/uploadfile/201801/c839cae3fea493c433a3619e8b3ff04420180110012116.gif "4")](http://www.wowotech.net/content/uploadfile/201801/9615d05276ad0faf7b77d81c6e82cf5f20180110012115.gif)

  **6** **小结**

> l  kobj对应一个目录；
> 
> l  kobj实现对象的生命周期管理（计数为0即清除）；
> 
> l  kset包含一个kobj，相当于也是一个目录；
> 
> l  kset是一个容器，可以包容不同类型的kobj（甚至父子关系的两个kobj都可以属于一个kset）；
> 
> l  注册kobj（如kobj_add()函数）时，优先以kobj->parent作为自身的parent目录；
> 
> 其次，以kset作为parent目录；若都没有，则是sysfs的顶层目录；
> 
> 同时，若设置了kset，还会将kobj加入kset.list。举例：
> 
> 1、无parent、无kset，则将在sysfs的根目录（即/sys/）下创建目录；
> 
> 2、无parent、有kset，则将在kset下创建目录；并将kobj加入kset.list；
> 
> 3、有parent、无kset，则将在parent下创建目录；
> 
> 4、有parent、有kset，则将在parent下创建目录，并将kobj加入kset.list；
> 
> l  注册kset时，如果它的kobj->parent为空，则设置它所属的kset（kset.kobj->kset.kobj）为parent，即优先使用kobj所属的parent；然后再使用kset作为parent。（如platform_bus_type的注册，如某些input设备的注册）
> 
> l  注册device时，dev->kobj.kset = devices_kset；然而kobj->parent的选取有优先次序：

[![3](http://www.wowotech.net/content/uploadfile/201801/127eba1d7a6fcf6f5d03921b28e5321320180110011146.gif "3")](http://www.wowotech.net/content/uploadfile/201801/31df6298c85112ffdf9fe93ef412a12d20180110011146.gif)

标签: [kset](http://www.wowotech.net/tag/kset) [kobj](http://www.wowotech.net/tag/kobj)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Meltdown论文翻译](http://www.wowotech.net/basic_subject/meltdown.html) | [O(n)、O(1)和CFS调度器](http://www.wowotech.net/process_management/scheduler-history.html)»

**评论：**

**伊斯科明**  
2021-12-24 17:16

说实话，还是没懂为什么有的目录是kset对象，而有的是kobject对象

[回复](http://www.wowotech.net/device_model/421.html#comment-8443)

**xyy**  
2022-07-20 21:51

@伊斯科明：有通知和没通知的区别

[回复](http://www.wowotech.net/device_model/421.html#comment-8646)

**kevin**  
2021-01-19 16:50

请教一下：kobject_add向kset.list中添加kobj->entry的事件是不是不会调用uevent通知用户kset的变化，只有kobject添加之后的状态修改才会调用uevent

[回复](http://www.wowotech.net/device_model/421.html#comment-8186)

**anlin**  
2019-03-07 16:49

"当引用计数为0时，kobject核心会调用kobject_release()，最后会调用kobj_type->release(kobj)来完成对象的释放。可是具体对象的释放，最后却通过kobj->kobj_type->release()来释放",这句话里面，前面的一个“kobj_type->release(kobj)”，是不是应该为“kobj->release(kobj)”？

[回复](http://www.wowotech.net/device_model/421.html#comment-7242)

**renyule**  
2018-04-18 17:06

似乎明白了点，总结的很棒

[回复](http://www.wowotech.net/device_model/421.html#comment-6677)

**[callme_friend](http://www.wowotech.net/)**  
2018-04-20 11:05

@renyule：谢谢。一起交流

[回复](http://www.wowotech.net/device_model/421.html#comment-6685)

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
    
    - ja  
        [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
    - 元神高手  
        [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
    - 十七  
        [内核空间的映射在系统启动时就已经设定好，并且在所有进程的页表...](http://www.wowotech.net/process_management/context-switch-arch.html#8920)
    - lw  
        [sparse模型和disconti模型没看出来有什么本质区别...](http://www.wowotech.net/memory_management/memory_model.html#8919)
    - 肥饶  
        [一个没设置好就出错](http://www.wowotech.net/linux_kenrel/516.html#8918)
    - orange  
        [点赞点赞，对linuxer的文章总结到位](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html#8917)
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
    
    - [Linux graphic subsystem(2)_DRI介绍](http://www.wowotech.net/graphic_subsystem/dri_overview.html)
    - [Windows系统结合MinGW搭建软件开发环境](http://www.wowotech.net/soft/6.html)
    - [spin_lock最简单用法。](http://www.wowotech.net/164.html)
    - [计算机科学基础知识之（六）：理解栈帧](http://www.wowotech.net/basic_subject/stack-frame.html)
    - [内存一致性模型](http://www.wowotech.net/memory_management/456.html)
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
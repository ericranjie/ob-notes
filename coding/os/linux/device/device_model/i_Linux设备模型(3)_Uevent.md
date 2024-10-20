
作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2014-3-10 20:39 分类：[统一设备模型](http://www.wowotech.net/sort/device_model)

# 1. Uevent的功能

Uevent是Kobject的一部分，用于在Kobject状态发生改变时，例如增加、移除等，通知用户空间程序。用户空间程序收到这样的事件后，会做相应的处理。

该机制通常是用来支持热拔插设备的，例如U盘插入后，USB相关的驱动软件会动态创建用于表示该U盘的device结构（相应的也包括其中的kobject），并告知用户空间程序，为该U盘动态的创建/dev/目录下的设备节点，更进一步，可以通知其它的应用程序，将该U盘设备mount到系统中，从而动态的支持该设备。

# 2. Uevent在kernel中的位置

下面图片描述了Uevent模块在内核中的位置：

![[Pasted image 20241019223333.png]]

由此可知，Uevent的机制是比较简单的，设备模型中任何设备有事件需要上报时，会触发Uevent提供的接口。Uevent模块准备好上报事件的格式后，可以通过两个途径把事件上报到用户空间：一种是通过kmod模块，直接调用用户空间的可执行文件；另一种是通过netlink通信机制，将事件从内核空间传递给用户空间。

注1：有关kmod和netlink，会在其它文章中描述，因此本文就不再详细说明了。

# 3. Uevent的内部逻辑解析

## 3.1 Source Code位置

Uevent的代码比较简单，主要涉及kobject.h和kobject_uevent.c两个文件，如下：

- include/linux/kobject.h
- lib/kobject_uevent.c

## 3.2 数据结构描述

kobject.h定义了uevent相关的常量和数据结构，如下：

- kobject_action

```cpp
 1: /* include/linux/kobject.h, line 50 */
 2: enum kobject_action {   
 3:     KOBJ_ADD,
 4:     KOBJ_REMOVE,    
 5:     KOBJ_CHANGE, 
 6:     KOBJ_MOVE,
 7:     KOBJ_ONLINE, 
 8:     KOBJ_OFFLINE,
 9:     KOBJ_MAX 
 10: };
```

kobject_action定义了event的类型，包括：

ADD/REMOVE，Kobject（或上层数据结构）的添加/移除事件。

ONLINE/OFFLINE，Kobject（或上层数据结构）的上线/下线事件，其实是是否使能。

CHANGE，Kobject（或上层数据结构）的状态或者内容发生改变。

MOVE，Kobject（或上层数据结构）更改名称或者更改Parent（意味着在sysfs中更改了目录结构）。

CHANGE，如果设备驱动需要上报的事件不再上面事件的范围内，或者是自定义的事件，可以使用该event，并携带相应的参数。

- kobj_uevent_env

```cpp
 1: /* include/linux/kobject.h, line 31 */
 2: #define UEVENT_NUM_ENVP         32 /* number of env pointers */
 3: #define UEVENT_BUFFER_SIZE      2048 /* buffer for the variables */
 4:  
 5: /* include/linux/kobject.h, line 116 */
 6: struct kobj_uevent_env {
 7:     char *envp[UEVENT_NUM_ENVP];
 8:     int envp_idx;
 9:     char buf[UEVENT_BUFFER_SIZE];
 10:    int buflen;
 11: }
```

前面有提到过，在利用Kmod向用户空间上报event事件时，会直接执行用户空间的可执行文件。而在Linux系统，可执行文件的执行，依赖于环境变量，因此kobj_uevent_env用于组织此次事件上报时的环境变量。

> envp，指针数组，用于保存每个环境变量的地址，最多可支持的环境变量数量为UEVENT_NUM_ENVP。
>
> envp_idx，用于访问环境变量指针数组的index。
>
> buf，保存环境变量的buffer，最大为UEVENT_BUFFER_SIZE。
>
> buflen，访问buf的变量。

- kset_uevent_ops

```cpp
 1: /* include/linux/kobject.h, line 123 */
 2: struct kset_uevent_ops {
 3:     int (* const filter)(struct kset *kset, struct kobject *kobj);
 4:     const char *(* const name)(struct kset *kset, struct kobject *kobj);
 5:     int (* const uevent)(struct kset *kset, struct kobject *kobj,
 6:                         struct kobj_uevent_env *env);
 7: };
```

kset_uevent_ops是为kset量身订做的一个数据结构，里面包含filter和uevent两个回调函数，用处如下：

> filter，当任何Kobject需要上报uevent时，它所属的kset可以通过该接口过滤，阻止不希望上报的event，从而达到从整体上管理的目的。
>
> name，该接口可以返回kset的名称。如果一个kset没有合法的名称，则其下的所有Kobject将不允许上报uvent
>
> uevent，当任何Kobject需要上报uevent时，它所属的kset可以通过该接口统一为这些event添加环境变量。因为很多时候上报uevent时的环境变量都是相同的，因此可以由kset统一处理，就不需要让每个Kobject独自添加了。

## 3.3 内部动作

通过kobject.h，uevent模块提供了如下的API（这些API的实现是在"lib/kobject_uevent.c”文件中）：

```cpp
 1: /* include/linux/kobject.h, line 206 */
 2: int kobject_uevent(struct kobject *kobj, enum kobject_action action);
 3: int kobject_uevent_env(struct kobject *kobj, enum kobject_action action,
 4:                         char *envp[]);
 5:  
 6: __printf(2, 3)
 7: int add_uevent_var(struct kobj_uevent_env *env, const char *format, ...);
 8:  
 9: int kobject_action_type(const char *buf, size_t count,
 10:                         enum kobject_action *type);
```

> kobject_uevent_env，以envp为环境变量，上报一个指定action的uevent。环境变量的作用是为执行用户空间程序指定运行环境。具体动作如下：
>
> - 查找kobj本身或者其parent是否从属于某个kset，如果不是，则报错返回（注2：由此可以说明，如果一个kobject没有加入kset，是不允许上报uevent的）
> - 查看kobj->uevent_suppress是否设置，如果设置，则忽略所有的uevent上报并返回（注3：由此可知，可以通过Kobject的uevent_suppress标志，管控Kobject的uevent的上报）
> - 如果所属的kset有uevent_ops->filter函数，则调用该函数，过滤此次上报（注4：这佐证了3.2小节有关filter接口的说明，kset可以通过filter接口过滤不希望上报的event，从而达到整体的管理效果）
> - 判断所属的kset是否有合法的名称（称作subsystem，和前期的内核版本有区别），否则不允许上报uevent
> - 分配一个用于此次上报的、存储环境变量的buffer（结果保存在env指针中），并获得该Kobject在sysfs中路径信息（用户空间软件需要依据该路径信息在sysfs中访问它）
> - 调用add_uevent_var接口（下面会介绍），将Action、路径信息、subsystem等信息，添加到env指针中
> - 如果传入的envp不空，则解析传入的环境变量中，同样调用add_uevent_var接口，添加到env指针中
> - 如果所属的kset存在uevent_ops->uevent接口，调用该接口，添加kset统一的环境变量到env指针
> - 根据ACTION的类型，设置kobj->state_add_uevent_sent和kobj->state_remove_uevent_sent变量，以记录正确的状态
> - 调用add_uevent_var接口，添加格式为"SEQNUM=%llu”的序列号
> - 如果定义了"CONFIG_NET”，则使用netlink发送该uevent
> - 以uevent_helper、subsystem以及添加了标准环境变量（HOME=/，PATH=/sbin:/bin:/usr/sbin:/usr/bin）的env指针为参数，调用kmod模块提供的call_usermodehelper函数，上报uevent。\
>   其中uevent_helper的内容是由内核配置项CONFIG_UEVENT_HELPER_PATH(位于./drivers/base/Kconfig)决定的(可参考lib/kobject_uevent.c, line 32)，该配置项指定了一个用户空间程序（或者脚本），用于解析上报的uevent，例如"/sbin/hotplug”。\
>   call_usermodehelper的作用，就是fork一个进程，以uevent为参数，执行uevent_helper。
>
> kobject_uevent，和kobject_uevent_env功能一样，只是没有指定任何的环境变量。
>
> add_uevent_var，以格式化字符的形式（类似printf、printk等），将环境变量copy到env指针中。
>
> kobject_action_type，将enum kobject_action类型的Action，转换为字符串。

|                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **说明：怎么指定处理uevent的用户空间程序(简称uevent helper)？  <br>  <br>上面介绍kobject_uevent_env的内部动作时，有提到，Uevent模块通过Kmod上报Uevent时，会通过call_usermodehelper函数，调用用户空间的可执行文件（或者脚本，简称**uevent helper\*\* ）处理该event。而该uevent helper的路径保存在\*\*uevent_helper数组中。  <br>  <br>可以在编译内核时，通过CONFIG_UEVENT_HELPER_PATH配置项，静态指定uevent helper。但这种方式会为每个event fork一个进程，随着内核支持的设备数量的增多，这种方式在系统启动时将会是致命的（可以导致内存溢出等）。因此只有在早期的内核版本中会使用这种方式，现在内核不再推荐使用该方式。因此内核编译时，需要把该配置项留空。  <br>  <br>在系统启动后，大部分的设备已经ready，可以根据需要，重新指定一个uevent helper，以便检测系统运行过程中的热拔插事件。这可以通过把helper的路径写入到"/sys/kernel/uevent_helper”文件中实现。实际上，内核通过sysfs文件系统的形式，将****uevent_helper数组开放到用户空间，供用户空间程序修改访问，具体可参考"****./kernel/ksysfs.c”中相应的代码，这里不再详细描述。\*\*\*\* |

_原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/linux_kenrel/uevent.html)。_

标签: [Linux](http://www.wowotech.net/tag/Linux) [Kernel](http://www.wowotech.net/tag/Kernel) [内核](http://www.wowotech.net/tag/%E5%86%85%E6%A0%B8) [设备模型](http://www.wowotech.net/tag/%E8%AE%BE%E5%A4%87%E6%A8%A1%E5%9E%8B) [Uevent](http://www.wowotech.net/tag/Uevent)

______________________________________________________________________

« [蓝牙协议中LQ和RSSI的原理及应用场景](http://www.wowotech.net/bluetooth/lqi_rssi.html) | [调试手段之sys节点](http://www.wowotech.net/linux_application/15.html)»

**评论：**

**xiaotonga**\
2023-09-15 21:31

@wowo 昵称被限制，要换个马甲，ip被限制咋办呢，总不能再换设备吧

[回复](http://www.wowotech.net/device_model/uevent.html#comment-8824)

**xiaotonga**\
2023-09-15 21:28

1.个人理解kset_uevent_ops的逻辑和vfs中struct file_operations {}结构很类似（linux/v2.6.39.4/source/include/linux/fs.h#L1537）；\
2.用户只需读写设备文件，不用关心底层设备驱动，通过文件系统vfs的标准接口调用设备文件对应device来调用具体的设备驱动程序实现。\
3.用户空间对读device的调用逻辑：\
usr->syscall(read)->file_operations.read()->driver.read_impl();\
4.对于kset_uevent_ops{}，文中说了filter、name、event便于内核同一管理。\
对于回调函数filter等接口，不同device需要在自己回调函数中实现具体功能，如（linux/v2.6.39.4/source/drivers/base/bus.c#L157）（linux/v2.6.39.4/source/fs/gfs2/sys.c#L630）\
（linux/v2.6.39.4/source/mm/slub.c#L4674）\
即内核提供像file_operations{}类似的标准接口，不同device或sys需要实现kset_uevent_ops{}中回调函数具体功能。

[回复](http://www.wowotech.net/device_model/uevent.html#comment-8823)

**poipoi**\
2020-07-28 23:00

wowo您好，我有一个疑问，kset_uevent_ops里 const char *(* const name)(struct kset \*kset, struct kobject \*kobj); 这个name回调函数是用来获取对应kset的名称的。但是kset的结构体中包含有kobject，而kobject中有name变量储存名称。那么在kset_uevent_ops里的name的回调函数，作用只是获取名称，内核直接写一个简单的函数，从kset->kobject->name获取名称就好了，为什么还需要将回调函数的实现交给使用者？

[回复](http://www.wowotech.net/device_model/uevent.html#comment-8072)

**qq**\
2020-03-08 21:19

对于不同的操作方法是怎样响应的呢？\
比如kobject-action有多种操作方式，那是所有的动作都响应么？还是需要注册某一种呢？谢谢

[回复](http://www.wowotech.net/device_model/uevent.html#comment-7909)

**poolloop**\
2019-12-19 17:36

您好，请教一个问题\
文章中介绍了2种将uevent上报给user space的方式：①kmod ②netlink\
这两种方式中，任何其中一种方式就可以独立实现预期的‘上报’行为了吗？\
还是说两中上报方式缺一不可，均是通知用户空间必要条件？

个人理解，\
对于kmod，文章中指出，所有与事件相关的信息似乎均已保存在，特别的环境变量中了。通过kmod机制启动的用户进程，似乎只需要读取自己的环境变量就可以了解，uevent的全部信息。

对于netlink，如果将uevent转换成netlink message，message应该已经包含了事件完整的信息。此时，netlink可以独立完成通知的前提就是，userspace要有一个正在运行着的daemon监听着netlink就好。

这么理解是对的吗？ 多谢！！！！

[回复](http://www.wowotech.net/device_model/uevent.html#comment-7789)

**威点零**\
2020-01-08 17:33

@poolloop：我觉得你的理解是对的。你去看代码，netlink/kmod都有各自的宏包着。我的kernel里面是netlink/kmod都有开。不过我的代码在上层有监听netlink，kmod userspace的路径并未设置，一样可以工作。

[回复](http://www.wowotech.net/device_model/uevent.html#comment-7825)

**Edward**\
2019-10-18 15:25

感谢博主分享，请问有kmod和netlink相关的文章吗，最近在研究usb热插拔，请问uevent只能通过编写程序读netlink才能监听吗？有没有更简单直接的办法？感谢

[回复](http://www.wowotech.net/device_model/uevent.html#comment-7704)

**Edward**\
2019-10-18 17:54

@Edward：已经找到了，用udevadm monitor命令即可观察uevent事件

[回复](http://www.wowotech.net/device_model/uevent.html#comment-7705)

**Jet**\
2019-04-02 23:28

谢谢\
最近在看 ueventd，这篇文章对我帮助很大\
已进行转载https://www.jianshu.com/p/10653d83909d\
不方便的话可撤下(*^\_\_^*)

[回复](http://www.wowotech.net/device_model/uevent.html#comment-7341)

**[wowo](http://www.wowotech.net/)**\
2019-04-03 10:38

@Jet：欢迎转载~~

[回复](http://www.wowotech.net/device_model/uevent.html#comment-7342)

**const**\
2018-03-13 12:53

wowo 你好，有个问题请教一下， kset_uevent_ops 里面的函数指针都进行了 const 修饰，这样除了确保第一次初始化后不会被再次修改，还有其他作用嘛？

[回复](http://www.wowotech.net/device_model/uevent.html#comment-6606)

**[wowo](http://www.wowotech.net/)**\
2018-03-13 16:13

@const：你可以去kernel里面看看，基本上所有的函数指针都是const的。

[回复](http://www.wowotech.net/device_model/uevent.html#comment-6607)

**Jackie**\
2017-12-07 11:07

@wowo @linuxer:\
请教一个问题，kernel发送uevent给用户空间，\
可这个时候用户空间处于freeze状态（suspend后还未resume回来），\
请问这时候发送的uevent是会丢掉，还是会存在某个buffer中？\
当用户空间resume回来的时候，还能收到这个uevent吗？\
谢谢!

[回复](http://www.wowotech.net/device_model/uevent.html#comment-6298)

**[wowo](http://www.wowotech.net/)**\
2017-12-08 10:06

@Jackie：现在的uevent基本上都使用netlink传递msg，kernel上报的msg会保存在一个socket queue中，等待user醒来并读取。所以resume回来的时候，还是能收到的。\
当然，如果保存的数据太多了（超过netlink的限制），也是会失败的，不过这种情况可以忽略（因为netlink的数目很大）。

[回复](http://www.wowotech.net/device_model/uevent.html#comment-6313)

**慢慢想**\
2016-09-12 13:36

wowo您好，想请教一个问题。\
在android启动的时候init进程会进行coldboot操作，往所有uevent文件里写入add，如下：

fd = openat(dfd, "uevent", O_WRONLY);\
write(fd, "add\\n", 4);

但是在对某个uevent写入add的时候花费了很久的时间，导致系统起不来。（每次启动都是同一个uevent耗费时间）\
想请教一下write(fd, "add\\n", 4);操作之后相应的driver都做了啥？下面该如何加log分析，我在write前后加了log，可以确定时间全部耗费在write这个函数上了。

非常感谢！

[回复](http://www.wowotech.net/device_model/uevent.html#comment-4560)

**[wowo](http://www.wowotech.net/)**\
2016-09-12 20:02

@慢慢想：对Android的uvent机制不是很熟悉，我猜测这里的write，是init进程通过uevent向另外一个进程传递了一个信息（add），另一个进程收到该信息后，做出相应的处理。\
你要查为什么这么耗时，就要查这个uevent最终是谁处理的。\
write uvent的过程大致如下：\
init进程\
write uvent(ADD)\
drivers/base/core.c\
static struct device_attribute uevent_attr\
store_uevent\
kobject_action_type\
KOBJ_ADD\
kobject_uevent\
kobject_uevent_env\
...

其它进程，收到ADD event，进行处理（find out it）

[回复](http://www.wowotech.net/device_model/uevent.html#comment-4561)

**慢慢想**\
2016-09-22 10:20

@wowo：@wowo\
怎么回复会显示IP被限制？

[回复](http://www.wowotech.net/device_model/uevent.html#comment-4589)

**[wowo](http://www.wowotech.net/)**\
2016-09-22 10:29

@慢慢想：这一段时间反垃圾评论好像有点抽风，不知道怎么回事，昨天我已经把屏蔽的IP列表全部清空了，应该没有问题了，另外你这不是可以评论吗？

[回复](http://www.wowotech.net/device_model/uevent.html#comment-4591)

**慢慢想**\
2016-09-22 10:30

@wowo：现在又能回复了。。。

您好，“其他进程”应该还是本进程，处理函数为handle_device_fd();不过这个函数没有耗费时间，耗费时间的只是write(fd, "add\\n", 4);系统调用。我猜测问题还是出现在kernel space在上报netlink之前，请问这样理解对么？是不是注册这个uevent的driver有问题？

940    fd = openat(dfd, "uevent", O_WRONLY);\
941    if(fd >= 0) {\
942        write(fd, "add\\n", 4);\
943        close(fd);\
944        handle_device_fd();\
945    }

[回复](http://www.wowotech.net/device_model/uevent.html#comment-4592)

**[wowo](http://www.wowotech.net/)**\
2016-09-22 10:56

@慢慢想：先确认是不是用的netlink，如果是的话，就看看时间到底耗在哪里了（write耗时？我持怀疑态度）

[回复](http://www.wowotech.net/device_model/uevent.html#comment-4593)

**慢慢想**\
2016-09-22 14:34

@wowo：您好，多谢指点，我再查查看。

[回复](http://www.wowotech.net/device_model/uevent.html#comment-4595)

**shu**\
2022-04-14 08:52

@慢慢想：上述耗时的问题确定了吗？

[回复](http://www.wowotech.net/device_model/uevent.html#comment-8587)

1 [2](http://www.wowotech.net/device_model/uevent.html/comment-page-2#comments)

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

  - [linux kernel的中断子系统之（八）：softirq](http://www.wowotech.net/irq_subsystem/soft-irq.html)
  - [u-boot FIT image介绍](http://www.wowotech.net/u-boot/fit_image_overview.html)
  - [ARM64的启动过程之（六）：异常向量表的设定](http://www.wowotech.net/armv8a_arch/238.html)
  - [tty驱动分析](http://www.wowotech.net/tty_framework/435.html)
  - [Linux graphic subsystem(2)\_DRI介绍](http://www.wowotech.net/graphic_subsystem/dri_overview.html)

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

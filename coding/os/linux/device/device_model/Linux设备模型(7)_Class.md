
作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2014-4-23 15:17 分类：[统一设备模型](http://www.wowotech.net/sort/device_model)

# 1. 概述

在设备模型中，Bus、Device、Device driver等等，都比较好理解，因为它们对应了实实在在的东西，所有的逻辑都是围绕着这些实体展开的。而本文所要描述的Class就有些不同了，因为它是虚拟出来的，只是为了抽象设备的共性。

举个例子，一些年龄相仿、需要获取的知识相似的人，聚在一起学习，就构成了一个班级（Class）。这个班级可以有自己的名称（如295），但如果离开构成它的学生（device），它就没有任何存在意义。另外，班级存在的最大意义是什么呢？是由老师讲授的每一个课程！因为老师只需要讲一遍，一个班的学生都可以听到。不然的话（例如每个学生都在家学习），就要为每人请一个老师，讲授一遍。而讲的内容，大多是一样的，这就是极大的浪费。

设备模型中的Class所提供的功能也一样了，例如一些相似的device（学生），需要向用户空间提供相似的接口（课程），如果每个设备的驱动都实现一遍的话，就会导致内核有大量的冗余代码，这就是极大的浪费。所以，Class说了，我帮你们实现吧，你们会用就行了。

这就是设备模型中Class的功能，再结合内核的注释：A class is a higher-level view of a device that abstracts out low-level implementation details(include/linux/device.h line326)，就容易理解了。

# 2. 数据结构描述

## 2.1 struct class

struct class是class的抽象，它的定义如下：

```cpp
1: /\* include/linux/device.h, line 332 \*/
2: struct class {
3:         const char              \*name;
4:         struct module           \*owner;
5:
6:         struct class_attribute          \*class_attrs;
7:         struct device_attribute         \*dev_attrs;
8:         struct bin_attribute            \*dev_bin_attrs;
9:         struct kobject                  \*dev_kobj;
10:
11:         int (\*dev_uevent)(struct device \*dev, struct kobj_uevent_env \*env);
12:         char \*(\*devnode)(struct device \*dev, umode_t \*mode);
13:
14:         void (\*class_release)(struct class \*class);
15:         void (\*dev_release)(struct device \*dev);
16:
17:         int (\*suspend)(struct device \*dev, pm_message_t state);
18:         int (\*resume)(struct device \*dev);
19:
20:         const struct kobj_ns_type_operations \*ns_type;
21:         const void \*(\*namespace)(struct device \*dev);
22:
23:         const struct dev_pm_ops \*pm;
24:
25:         struct subsys_private \*p;
26: };
```

> 其实struct class和struct bus很类似，解释如下：
>
> name，class的名称，会在“/sys/class/”目录下体现。
>
> class_atrrs，该class的默认attribute，会在class注册到内核时，自动在“/sys/class/xxx_class”下创建对应的attribute文件。
>
> dev_attrs，该class下每个设备的attribute，会在设备注册到内核时，自动在该设备的sysfs目录下创建对应的attribute文件。
>
> dev_bin_attrs，类似dev_attrs，只不过是二进制类型attribute。
>
> dev_kobj，表示该class下的设备在/sys/dev/下的目录，现在一般有char和block两个，如果dev_kobj为NULL，则默认选择char。
>
> dev_uevent，当该class下有设备发生变化时，会调用class的uevent回调函数。
>
> class_release，用于release自身的回调函数。
>
> dev_release，用于release class内设备的回调函数。在device_release接口中，会依次检查Device、Device Type以及Device所在的class，是否注册release接口，如果有则调用相应的release接口release设备指针。
>
> p，和“[Linux设备模型(6)\_Bus](http://www.wowotech.net/linux_kenrel/bus.html)”中struct bus结构一样，不再说明。

## 2.2 struct class_interface

struct class_interface是这样的一个结构：它允许class driver在class下有设备添加或移除的时候，调用预先设置好的回调函数（add_dev和remove_dev）。那调用它们做什么呢？想做什么都行（例如修改设备的名称），由具体的class driver实现。

该结构的定义如下：

```cpp
1: /\* include/linux/device.h, line 434 \*/
2: struct class_interface {
3:         struct list_head        node;
4:         struct class            \*class;
5:
6:         int (\*add_dev)          (struct device \*, struct class_interface \*);

7:         void (\*remove_dev)      (struct device \*, struct class_interface \*);

8: };
```

#### 3. 功能及内部逻辑解析

##### 3.1 class的功能

看完上面的东西，蜗蜗依旧糊里糊涂的，class到底提供了什么功能？怎么使用呢？让我们先看一下现有Linux系统中有关class的状况（这里以input class为例）：

```cpp
root@android:/ # ls /sys/class/input/ -l                                                                                          
lrwxrwxrwx root     root              2014-04-23 03:39 event0 -> ../../devices/platform/i2c-gpio.17/i2c-17/17-0066/max77693-muic/input/input0/event0
lrwxrwxrwx root     root              2014-04-23 03:39 event1 -> ../../devices/platform/gpio-keys.0/input/input1/event1
lrwxrwxrwx root     root              2014-04-23 03:39 event10 -> ../../devices/virtual/input/input10/event10
lrwxrwxrwx root     root              2014-04-23 03:39 event2 -> ../../devices/platform/s3c2440-i2c.3/i2c-3/3-0048/input/input2/event2
…
lrwxrwxrwx root     root              2014-04-23 03:39 event8 -> ../../devices/platform/soc-audio/sound/card0/input8/event8\
lrwxrwxrwx root     root              2014-04-23 03:39 event9 -> ../../devices/platform/i2c-gpio.8/i2c-8/8-0020/input/input9/event9\
lrwxrwxrwx root     root              2014-04-23 03:39 input0 -> ../../devices/platform/i2c-gpio.17/i2c-17/17-0066/max77693-muic/input/input0\
…\
lrwxrwxrwx root     root              2014-04-23 03:39 mice -> ../../devices/virtual/input/mice

root@android:/mailto:root@android:/ # ls /sys/devices/platform/s3c2440-i2c.3/i2c-3/3-0048/input/input2/event2/ -l

-r--r--r-- root     root         4096 2014-04-23 04:08 dev\
lrwxrwxrwx root     root              2014-04-23 04:08 device -> ../../input2\
drwxr-xr-x root     root              2014-04-23 04:08 power\
lrwxrwxrwx root     root              2014-04-23 04:08 subsystem -> ../../../../../../../../class/input\
-rw-r--r-- root     root         4096 2014-04-23 04:08 uevent

root@android:/ # ls /sys/devices/virtual/input/mice/ -l                                      \
-r--r--r-- root     root         4096 2014-04-23 03:57 dev\
drwxr-xr-x root     root              2014-04-23 03:57 power\
lrwxrwxrwx root     root              2014-04-23 03:57 subsystem -> ../../../../class/input\
-rw-r--r-- root     root         4096 2014-04-23 03:57 uevent
```

看上面的例子，发现input class也没做什么实实在在的事儿，它（input class）的功能，仅仅是：

- 在/sys/class/目录下，创建一个本class的目录（input）
- 在本目录下，创建每一个属于该class的设备的符号链接（如，把“sys/devices/platform/s3c2440-i2c.3/i2c-3/3-0048/input/input2/event2”设备链接到”/sys/class/input/event2”），这样就可以在本class目录下，访问该设备的所有特性（即attribute）
- 另外，device在sysfs的目录下，也会创建一个subsystem的符号链接，链接到本class的目录

算了，我们还是先分析一下Class的核心逻辑都做了哪些事情，至于class到底有什么用处，可以在后续具体的子系统里面（如input子系统），更为细致的探讨。

## 3.2 class的注册

> class的注册，是由\_\_class_register接口（它的实现位于"drivers/base/class.c, line 609"）实现的，它的处理逻辑和bus的注册类似，主要包括：
>
> - 为class结构中的struct subsys_private类型的指针（cp）分配空间，并初始化其中的字段，包括cp->subsys.kobj.kset、cp->subsys.kobj.ktype等等
> - 调用kset_register，注册该class（回忆“[Linux设备模型(6)\_Bus](http://www.wowotech.net/linux_kenrel/bus.html)”中的描述，一个class就是一个子系统，因此注册class也是注册子系统）。该过程结束后，在/sys/class/目录下，就会创建对应该class（子系统）的目录
> - 调用add_class_attrs接口，将class结构中class_attrs指针所指向的attribute，添加到内核中。执行完后，在/sys/class/xxx_class/目录下，就会看到这些attribute对应的文件

## 3.3 device注册时，和class有关的动作

在"[Linux设备模型(5)\_device和device driver](http://www.wowotech.net/linux_kenrel/device_and_driver.html)”中，我们有讲过struct device和struct device_driver这两个数据结构，其中struct device结构会包含一个struct class指针（这从侧面说明了class是device的集合，甚至，class可以是device的driver）。当某个class driver向内核注册了一个class后，需要使用该class的device，通过把自身的class指针指向该class即可，剩下的事情，就由内核在注册device时处理了。

本节，我们讲一下在device注册时，和class有关的动作：

> device的注册最终是由device_add接口（drivers/base/core.c）实现了，该接口中和class有关的动作包括：
>
> - 调用device_add_class_symlinks接口，创建3.1小节描述的各种符号链接，即：在对应class的目录下，创建指向device的符号链接；在device的目录下，创建名称为subsystem、指向对应class目录的符号链接
> - 调用device_add_attrs，添加由class指定的attributes（class->dev_attrs）
> - 如果存在对应该class的add_dev回调函数，调用该回调函数

# 4. 结束语

其实在这篇文章结束后，蜗蜗依旧没有弄清楚class在内核到底是怎么使用的。不过没关系，在后续的子系统的分析中（如input子系统、RTC子系统等），我们会看到很多class的使用用例。到时候，再回过头总结，就会很清楚了。

_原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/linux_kenrel/class.html)。_

标签: [Linux](http://www.wowotech.net/tag/Linux) [Kernel](http://www.wowotech.net/tag/Kernel) [内核](http://www.wowotech.net/tag/%E5%86%85%E6%A0%B8) [设备模型](http://www.wowotech.net/tag/%E8%AE%BE%E5%A4%87%E6%A8%A1%E5%9E%8B) [class](http://www.wowotech.net/tag/class)

---

« [Linux设备模型(8)\_platform设备](http://www.wowotech.net/device_model/platform_device.html) | [Process Creation（一）](http://www.wowotech.net/process_management/Process-Creation-1.html)»

**评论：**

**sklli**\
2018-12-21 11:56

@wowo   大神，请问下input子系统什么时候上线了？

[回复](http://www.wowotech.net/device_model/class.html#comment-7095)

**js_wawayu**\
2016-07-14 09:17

请教一下，根目录下的dev和sys目录下的dev有什么关系？这几篇设备模型里的节点都是位于sys目录下的

[回复](http://www.wowotech.net/device_model/class.html#comment-4250)

**[wowo](http://www.wowotech.net/)**\
2016-07-14 09:51

@js_wawayu：/dev下面的东西，本质是“文件系统”有关的东西，不属于设备模型：\
mknod - make block or character special files

/sys/dev/下面的东西，就是系统所有device的一个符号连接：\
ls -l /sys/dev/\
可以看看是什么效果。

[回复](http://www.wowotech.net/device_model/class.html#comment-4254)

**czh**\
2016-01-05 11:48

非常感谢蜗蜗的好文章！\
有一个问题，如果在device_add()时，device的bus和class都不为空，device_add_class_symlinks(dev)会调用\
error = sysfs_create_link(&dev->kobj,&dev->class->p->subsys.kobj,"subsystem"); bus_add_device(dev);也会调用\
error = sysfs_create_link(&dev->kobj,&dev->bus->p->subsys.kobj, "subsystem");是不是会有冲突？

[回复](http://www.wowotech.net/device_model/class.html#comment-3344)

**[wowo](http://www.wowotech.net/)**\
2016-01-05 17:12

@czh：不会的，一个是在class目录，一个是在bus目录。

[回复](http://www.wowotech.net/device_model/class.html#comment-3346)

**czh**\
2016-01-06 10:44

@wowo：回复的这么快，太感动了！\
"subsystem"链接都是在devices目录下创建的，class目录和bus目录不会有啊\
是不是我的问题没有表述明白。如果一个device注册到kernel的时候bus指针和class指针都不为空的话，既要在device的目录下创建subsystem链接到bus，又要创建subsystem链接到class，这样不会又冲突吗？\
是不是device的bus指针和class指针不会同时有效？

[回复](http://www.wowotech.net/device_model/class.html#comment-3352)

**[wowo](http://www.wowotech.net/)**\
2016-01-06 16:38

@czh：抱歉，是我没有仔细看您的问题，并不是您没有表述清除。\
确实，device_add的时候，如果同时存在class和bus的时候，sysfs_create_link时，后一个（也就是bus）会报错，不能共存。\
对于bus和class，我的理解是：\
同一个bus下的设备，是一种“空间上（或物理上）”聚集，之所以加引号，可能是虚拟的；\
同一个class下的设备，是一种“文化上”的聚集，例如我们有共同的特征、共同的兴趣爱好等等。

那么，一个设备是否可能既从属于某一个bus，又从属于某一个class？是可以的。通常的做法是：\
该设备的device指针（由设备模型管理），和bus打交道，如某一个platform设备下的device指针；\
如果需要加入某一个class，则新添一个子设备，让这个设备加入到class。

[回复](http://www.wowotech.net/device_model/class.html#comment-3355)

**[callme_friend](http://www.wowotech.net/)**\
2016-05-12 01:04

@wowo：1、如果device_add的时候，dev同时存在class和bus，那么后一个（也就是bus）会报错。\
我觉得这好像不仅仅导致不能共存，而且影响了bus的管理。\
int bus_add_device(struct device \*dev)\
{\
...\
error = sysfs_create_link(&dev->kobj,\
&dev->bus->p->subsys.kobj, "subsystem");\
if (error)\
goto out_subsys;\
klist_add_tail(&dev->p->knode_bus, &bus->p->klist_devices);

return 0;\
}\
sysfs_create_link()失败，导致dev不能注册到bus的klist_devices。\
2、如果需要加入某一个class，则新添一个子设备，让这个设备加入到class。\
如果是子设备添加到class，那么该子设备的bus必定是NULL吧？这样的话，也不算同时包含bus和class吧？我看到的gpio_keys例子就是这样。wowo看到过子设备同时包含bus和class的例子吗？

[回复](http://www.wowotech.net/device_model/class.html#comment-3922)

**[wowo](http://www.wowotech.net/)**\
2016-05-12 09:05

@callme_friend：是的，子设备是parent--child的关系，不是总线上的从属关系。

[回复](http://www.wowotech.net/device_model/class.html#comment-3924)

**kevin**\
2016-05-28 11:44

@wowo：“device_add的时候，如果同时存在class和bus的时候，sysfs_create_link时，后一个（也就是bus）会报错，不能共存。”

这里应该有点问题，在 device_add 的时候，为device添加class软链接的时候，会先做以下判断：if(!dev->class) return 0;我们知道父设备是不会注册class的，所以这里会直接返回 0，然后继续执行 bus_add_device，在 sysfs_create_link 中不会失败报错。

[回复](http://www.wowotech.net/device_model/class.html#comment-3994)

**[wowo](http://www.wowotech.net/)**\
2016-05-30 08:55

@kevin：kevin同学，可能您没有看清楚我们上面的讨论。if(!dev->class) return 0;这个判断也是当前设备，不是父设备。\
“device_add的时候，如果同时存在class和bus的时候，sysfs_create_link时，后一个（也就是bus）会报错，不能共存。” 这个表述应该是对的~~~

[回复](http://www.wowotech.net/device_model/class.html#comment-3997)

**kevin**\
2016-05-30 18:37

@wowo：谢谢指正！我重新看了一下代码，我的理解确实有误，当 dev->class 存在的时候，在 bus_add_device 中，bus = bus_get(dev->bus) 的返回值为 NULL，所以并不会走到 sysfs_create_link，原提问前提应该不存在。至于为什么 dev->class 存在时，bus 会为 NULL，我暂时还没找到答案。我猜应该跟设备创建时 class_register 有关。

wowo 可否指点一二。

**wzw200**\
2015-03-25 09:28

class网上讲的很少，可以直接看代码，\
《分析过mdev(udev的BusyBox简化版)源码的都知道mdev的基本原理》\
执行mdev -s命令时，mdev扫描/sys/block(块设备保存在/sys/block目录下,内核2.6.25版本以后，块设备也保存在/sys/class/block目录下。mdev扫描/sys/block是为了实现向后兼容)和/sys/class两个目录下的dev属性文件，从该dev属性文件中获取到设备编号(dev属性文件以"major:minor\\n"形式保存设备编号)，并以包含该dev属性文件的目录名称作为设备名device_name(即包含dev属性文件的目录称为device_name，而/sys/class和device_name之间的那部分目录称为subsystem。也就是每个dev属性文件所在的路径都可表示为/sys/class/subsystem/device_name/dev)，在/dev目录下创建相应的设备文件。例如，cat /sys/class/tty/tty0/dev会得到4:0，subsystem为tty,device_name为tty0。

[回复](http://www.wowotech.net/device_model/class.html#comment-1410)

**lucifer**\
2015-05-31 16:30

@wzw200：多谢你的解答。我再看看代码。

[回复](http://www.wowotech.net/device_model/class.html#comment-1903)

**lucifer**\
2015-02-23 22:14

怎么理解比如input subsystem跟设备模型什么关系了？我自己看到设备模型是在讲driver跟device的绑定，但是input似乎是在说处理事件？应用层怎么应对了？不是说设备都是文件吗？另外最简单的驱动例子的的都是实现fops，似乎在设备模型和类似input subsystem中看不到。

[回复](http://www.wowotech.net/device_model/class.html#comment-1214)

**wowo**\
2015-02-25 10:56

@lucifer：input子系统和设备模型的关系有两个基本的体现：

1. input device（struct input_dev）本身就包含了struct device变量，并且在input device注册（input_register_device）时，会调用device_add等设备模型的接口，添加设备模型的元素。
1. 所谓的event机制，正是基于设备模型（uevent）的封装。

至于fops，就是字符设备层面的事情了，在input子系统也是存在的，只不过input subsystem core的实现比较简单，只规定了input字符设备的major number，其它的具体事情（如fops实现），则由具体的input driver负责实现，例如drivers/input/mousedev.c。

[回复](http://www.wowotech.net/device_model/class.html#comment-1218)

**lucifer**\
2015-02-26 22:03

@wowo：谢谢回复!我再看看.

[回复](http://www.wowotech.net/device_model/class.html#comment-1228)

**Bright-Ho**\
2023-06-30 11:10

@wowo：2. 所谓的event机制，正是基于设备模型（uevent）的封装。\
wowo，你好！\
这句话有依据吗？event事件作为input子系统的事件上报机制，与设备模型（uevent）有关系吗？

[回复](http://www.wowotech.net/device_model/class.html#comment-8793)

**袁波**\
2014-11-18 12:22

楼主大神你的《input子系统、RTC子系统分析》文章在哪里可以找到啊。很想看一下啊。谢谢了啊。我找了半天都没看得起。

[回复](http://www.wowotech.net/device_model/class.html#comment-749)

**wowo**\
2014-11-18 12:29

@袁波：非常抱歉，还没写呢。业余时间实在不多，很多想写的都没有写。抱歉~~

[回复](http://www.wowotech.net/device_model/class.html#comment-750)

**袁波**\
2014-11-17 15:27

楼主我发现你文章中有个错误。dev_kobj，代表本class的kobject，用于在sysfs中创建目录。

这句话有错误吧，class 在sysfs中创建的目录是由class 中subsys_private \*p 中的subsys这个kset决定的吧。

内核中在 \_\_class_register(struct class \*cls, struct lock_class_key \*key)中有这么一段代码\
struct subsys_private *cp;\
error = kset_register(&cp->subsys);这句就说明了class是注册的这个subsys这个kset而并非dev_kobj。\
而同样在注册这个函数中还有这么一段代码：\
/* set the default /sys/dev directory for devices of this class \*/\
if (!cls->dev_kobj)\
cls->dev_kobj = sysfs_dev_char_kobj;\
这说说明了dev_kobj这个成员代表的是这样一内设备的类型缺省为/sys/dev/char

[回复](http://www.wowotech.net/device_model/class.html#comment-741)

**wowo**\
2014-11-17 18:50

@袁波：非常感谢。文中的表述确实是错的。正像您说的，dev_kobj表示该class下的设备，在/sys/dev/下的目录，现在一般有char和block两个，如果dev_kobj为NULL，则默认选择char。\
一会儿我会更新文章，谢谢~~

[回复](http://www.wowotech.net/device_model/class.html#comment-743)

**袁波**\
2014-11-17 09:34

大神啊，看了楼主的文章我一下就明白过来了。谢谢。

[回复](http://www.wowotech.net/device_model/class.html#comment-738)

**非常响亮**\
2014-11-11 14:50

对的，我也是想表达这个意思。事实也应该如此，device_register()-->device_initialize（）中首先将dev的kset字段设为devices(dev->kobj.kset = devices_kset;)如果dev的class、parent字段都为空，那么后续将不会对dev->kset进行操作，那么dev将置于/sys/devices中。这样逆着推，/sys/devices中的所有子目录都应该是device_register（）的结果。但是目录中也有例外，如先前那个例子，（先纠正上面的输入错误，led_class/目录下是led_dev而不是myled）在/sys/devices/virtual/led_class/led_dev中,led_class仅仅只是一个kset实体，并不包括device实体。不过我想上面有这么一句话应该可以解释：Class-devices with a non class-device as parent, live in a "glue" directory to prevent namespace collisions.

[回复](http://www.wowotech.net/device_model/class.html#comment-713)

**非常响亮**\
2014-11-10 11:59

刚学linux不久，还是个菜鸟，看到大神们百忙之中抽出时间发贴授课为我们后辈指引迷津，对前辈们无私奉献的精神、炉火纯青的技术、严谨深刻的分析表示深深的敬佩，更让我明确了以后对技术追求的态度，专于技术，乐于奉献。\
这几天一直在看设备模型，摸清了大概，但是还有几个地方不是很明白，请帮忙分析分析下（用的是2.6.32内核）\
注册一个字符设备的时候我是这么做的：\
1.major = register_chrdev(0, "myled", &led_fops); 注册一个字符设备\
2.cls = class_create(THIS_MODULE, "led_class");  创建一个类\
3.device_create(cls, NULL, MKDEV(major, 0), NULL, "dev_led"); 在类下创建一个设备以供udev自动创建设备节点

这样的结果是在/sys/devices和/sys/class下都产生了led_class/myled/,（截了图，插不进来..）在class中产生这样一个类和类中的设备容易理解，但是在/sys/devices产生led_class这个一个目录就有点不懂了，追踪class_create()->\_\_class_create()->\_\_class_register(),在\_\_class_register（）中只找到kset_register(&cp->class_subsys)即可在class中创建led_class目录，但是没有找到关于在/sys/devices中能创建led_class的蛛丝马迹。  （ps）led_class是一个类，为什么会出现在/sys/devices中呢？类也是一个设备么？  \
再追踪device_create（）->...->device_register()，发现此函数传入的parent参数置为Null的话应该会在/sys/devices中创建dev_led的。

描述有点冗长，还忘不要介意，希望蜗蜗前辈能帮我理清这几个目录之间的关系，

[回复](http://www.wowotech.net/device_model/class.html#comment-696)

**wowo**\
2014-11-10 13:17

@非常响亮：我觉得你已经分析的很清楚了啊，device_create，总要在sysfs下某一个目录中创建该设备的目录啊，如果你不指定它的parent的话，难道让它在/sys/devices下创建吗？所以kernel采用了相对优雅的方式，帮你创建一个以class命名的目录。\
其实你可以看看kernel标准led class是怎么做的，例如：\
led_pwm_probe-->led_classdev_register:\
ret = led_classdev_register(&pdev->dev, &led_dat->cdev);

虽然某一个LED设备（led_classdev_register所注册的）依附于led class，但首先它是一个platform device，那么他就可以在/sys/devices/platform/下创建自己的目录了。

[回复](http://www.wowotech.net/device_model/class.html#comment-698)

**非常响亮**\
2014-11-11 13:21

@wowo：确实像wowo前辈说的，kernel创建目录的方式很优雅。昨天追踪子函数的还不够仔细深入，想当然的漏了一个函数没看。唉，幸好今天再仔细查看的时候发现了这样一个函数：==>device_register==>device_add==>setup_parent==>get_device_parent，在这个函数中有这么一段话If we have no parent, we live in "virtual". Class-devices with a non class-device as parent, live in a "glue" directory to prevent namespace collisions.     dev在有class而没有parent时，会遍历dev->class->class_dirs.list 寻找在virtual中的class_dirs（一个kset），找到了则返回此kobj当作parent,如果没找到则创建一个与class同名的kset当作parent . 这样一来，class中的设备在/sys/devices中存放的位置必定是在一个与class同名的kset下。

[回复](http://www.wowotech.net/device_model/class.html#comment-710)

**lucifer**\
2016-08-26 21:34

@wowo：wowo用lxr吗？很奇怪我怎么搜3.12内核中class结果是unused，结果但是代码中可以看到class，只是显示灰色。

[回复](http://www.wowotech.net/device_model/class.html#comment-4459)

**[wowo](http://www.wowotech.net/)**\
2016-08-27 16:41

@lucifer：我很少用lxr，很多是哈我是直接在github上看，可以关键字搜索。

[回复](http://www.wowotech.net/device_model/class.html#comment-4460)

**lucifer**\
2016-08-28 21:54

@wowo：谢谢哈~

[回复](http://www.wowotech.net/device_model/class.html#comment-4461)

**[linuxer](http://www.wowotech.net/)**\
2014-11-10 16:30

@非常响亮：可以从两个角度来看一个设备：\
1、物理拓扑方面。非常直观，物理上一个外设是如何连接到系统的。\
2、该设备属于那个类别的设备。从high level角度来看一个设备，抽象其特性（例如：I2C的声卡和PCI的声卡虽然low level的bus是不一样的，但是，作为声卡，无论是I2C还是PCI都是有着共同的设备属性。\
例如：一个device，是一个声卡设备，这个结论其实就是从上面的第二点来观察该外设的。我们不管他是什么总线设备，我关注的是设备类别。但是如果从物理拓扑方面来看，我们则关注其属于哪个总线上的设备。例如一个USB声卡，其拓扑结构可能是PCI总线－－－USB总线－－－－USB sound device。

sys文件系统做设备模型和user space的接口，毫无疑问也要提供两个层面的通道来描述一个设备。我们用你的例子来说明好了。对于一个LED设备，sysfs可以有两个节点来描述该外设：\
1、/sys/class/led_class目录下的节点。这是从high level角度看该LED设备的接口。这里我们可以把一个属于LED设备的属性放到这个目录下，例如：LED颜色、闪烁的频率、占空比等，只要是led class，任何设备都是有这些属性的（无论是接到platform bus还是接到I2C bus）。\
2、/sys/class/virtual/led_class目录下的节点。这个节点是和物理拓扑有关了。其实对于device_create(哪一类设备, 物理拓扑参数,....) 而言，如果你不传递物理拓扑参数（传入NULL），那么设备模型会统一放到virtual的目录下。这个目录保存的属性文件和/sys/class/led_class目录下的是不一样的，这里的属性文件更多的是关注作为某个总线外设的属性。我们可以把这个LED变得复杂些，例如是一个USB接口的LED设备（当然，实际上我是没有见过这样的设备，呵呵～～最多是I2C的LED设备），这时候，这个目录下就会保存这样的属性文件：该设备有多少个endpoint、interface的数目，类别等，而这些属性是和USB总线有直接的关系

[回复](http://www.wowotech.net/device_model/class.html#comment-701)

**非常响亮**\
2014-11-11 13:39

@linuxer：谢谢linuxer前辈详细的解说。受益良多。\
还想请教一个问题，sysfs中所有的目录都具kobj属性，bus目录中所有的目录都具有bus_type属性，class目录中所有的目录都具有class属性，同理推想，devices中应该也具有某个数据结构的属性吧？  不知道这样理解对不对

[回复](http://www.wowotech.net/device_model/class.html#comment-711)

**[linuxer](http://www.wowotech.net/)**\
2014-11-11 14:12

@非常响亮：我觉得这是使用“属性”这个词汇不是非常合适，更准确的说：\
sysfs中所有的目录以及目录下的文件都对应内核中的一个struct kobj实例。\
bus目录中所有的子目录都对应内核中的一个struct bus_type实例。\
class目录中所有的子目录对应内核中的一个struct class实例\
devices目录中的所有目录以及子目录都对应一个struct device实例

[回复](http://www.wowotech.net/device_model/class.html#comment-712)

**非常响亮**\
2014-11-11 14:52

@linuxer：@linuxer 对的，我也是想表达这个意思。事实也应该如此，device_register()-->device_initialize（）中首先将dev的kset字段设为devices(dev->kobj.kset = devices_kset;)如果dev的class、parent字段都为空，那么后续将不会对dev->kset进行操作，那么dev将置于/sys/devices中。这样逆着推，/sys/devices中的所有子目录都应该是device_register（）的结果。但是目录中也有例外，如先前那个例子，（先纠正上面的输入错误，led_class/目录下是led_dev而不是myled）在/sys/devices/virtual/led_class/led_dev中,led_class仅仅只是一个kset实体，并不包括device实体。不过我想上面有这么一句话应该可以解释：Class-devices with a non class-device as parent, live in a "glue" directory to prevent namespace collisions.

[回复](http://www.wowotech.net/device_model/class.html#comment-714)

**hony**\
2017-01-14 11:36

@linuxer："sysfs中所有的目录以及目录下的文件都对应内核中的一个struct kobj实例。"—— 目录下的属性文件不是kobject实体哦。

[回复](http://www.wowotech.net/device_model/class.html#comment-5144)

**[wowo](http://www.wowotech.net/)**\
2017-01-16 09:09

@hony：“目录下的属性文件”，指的是什么呢？

[回复](http://www.wowotech.net/device_model/class.html#comment-5152)

**hello**\
2017-01-16 10:50

@hony：多谢指正！应该是目录对应kobject，目录下的属性文件对应attribute

[回复](http://www.wowotech.net/device_model/class.html#comment-5159)

1 [2](http://www.wowotech.net/device_model/class.html/comment-page-2#comments)

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

  - [逆向映射的演进](http://www.wowotech.net/memory_management/reverse_mapping.html)
  - [Linux内核同步机制之（七）：RCU基础](http://www.wowotech.net/kernel_synchronization/rcu_fundamentals.html)
  - [显示技术介绍(3)\_CRT技术](http://www.wowotech.net/display/crt_intro.html)
  - [Linux TTY framework(4)\_TTY driver](http://www.wowotech.net/tty_framework/tty_driver.html)
  - [Linux时间子系统系列文章之目录](http://www.wowotech.net/timer_subsystem/time_subsystem_index.html)

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

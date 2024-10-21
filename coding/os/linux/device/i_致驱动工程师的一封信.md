
作者：[smcdef](http://www.wowotech.net/author/531) 发布于：2018-4-14 21:00 分类：[统一设备模型](http://www.wowotech.net/sort/device_model)

# **引言**

作为一个算是合格的驱动工程师，总是有很多话想说。代码看的多了总是有些小感悟。可能是吧。那就总结一下自己看的代码的一些感悟和技巧。如何利用你看的这些代码？如何体现在工作的调试中。作为驱动工程师，主要的工作就是移植各种驱动，接触各种硬件。接触最多的就是dts、中断、gpio、sysfs、proc fs。如何利用sysfs、proc fs及内核提供的接口为我们降低调试难度，快速解决问题呢？

> 注：部分代码分析举例基于linux-4.15。

# **如何利用dts**

首先我们关注的主要是两点，gpio和irq。其他的选择忽略。先展示一下我期望的gpio和irq的使用方法。dts如下。

```cpp
1. device {
2.     rst-gpio = <&gpioc_ctl 10 OF_GPIO_ACTIVE_LOW>;
3.     irq-gpio = <&gpioc_ctl 11 0>;
4.     interrupts-extended = <&vic 11 IRQF_TRIGGER_RISING>;
5. };
```

对于以上的dts你应该再熟悉不过，当然这里不是教你如何使用dts，而是关注gpio和irq最后一个数字可以如何利用。例如rst-gpio的OF_GPIO_ACTIVE_LOW代表什么意思呢？可以理解为低有效。什么意思呢？举个例子，正常情况下，我们需要一个gpio口控制灯，我们认为灯打开就是active状态。对于一个程序员来说，我们可以封装一个函数，写1就是打开灯，写0就是关灯。但是对于硬件来说，变化的是gpio口的电平状态。如果gpio输出高电平灯亮，那么这就是高有效。如果硬件设计是gpio输出低电平灯亮，那么就是低有效。对于一个软件工程师来说，我们的期望是写1就是亮灯，写0就是关灯。我可不管硬件工程师是怎么设计的。我们可以认为dts是描述具体的硬件。因此对于驱动来说，硬件的这种变化，只需要修改dts即可。软件不用任何修改。软件可以如下实现。

```cpp
1. int device_probe(struct platform_device *pdev)
2. {
3.     rst_gpio = of_get_named_gpio_flags(np, "rst-gpio", 0, &flags);
4.     if (flags & OF_GPIO_ACTIVE_LOW) {
5.         struct gpio_desc *desc;

7.         desc = gpio_to_desc(rst_gpio);
8.         set_bit(FLAG_ACTIVE_LOW, &desc->flags);
9.     }

11.     irq = of_irq_get(np, 0);
12.     trigger_type = irq_get_trigger_type(irq);
13.     request_threaded_irq(irq, NULL, irq_handler, trigger_type, "irq", NULL);
14. }
```

驱动按照以上代码实现的话，如果修改中断触发类型或者电平有效状态只需要修改dts即可。例如不同的IC复位电平是不一样的，有的IC是高电平复位，有的IC却是低电平复位。其实这就是一个电平有效状态的例子。

# **如何调试gpio**

移植驱动阶段或者调试阶段的工程中，难免想知道当前gpio的电平状态。当然很easy。万用表戳上去不就行了。是啊！硬件工程师的思维。作为软件工程师自然是要软件的方法。下面介绍两个api接口。自己摸索使用吧。点到为止。

```cpp
1. static inline int gpio_export(unsigned gpio, bool direction_may_change);
2. static inline int gpio_export_link(struct device *dev, const char *name,
3.            unsigned gpio);
```

在你的driver中调用以上api后，编译下载。去/sys/class/gpio目录看看有什么发现。

# **如何调试irq**

调试的时候也难免会确定硬件是否产生中断。我们该怎么办呢？也很easy。

1. cat /proc/interrupts

输出信息不想多余的介绍。看代码去。根据输出的irq num，假设是irq_num。请进入以下目录。看看下面都有什么文件。摸索这些文件可以为你调试带来哪些方便。

1. cd /proc/irq/irq_num

# **dts和sysfs有什么关联**

曾经写过一篇dts解析的文章[http://www.wowotech.net/device_model/dt-code-file-struct-parse.html](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html)。最后一节其实说了一个很有意思的东西。但是仅仅是寥寥结尾。并没有展开。因为他不关乎我写的文章的的主题。但是，他却和调试息息相关。

1. cd /sys/firmware/devicetree/base

该目录的信息就是整个dts。将整个dts的节点以及属性全部展现在sysfs中。他对我们的调试有什么用呢？Let me tell you。如果你的项目非常的复杂，例如一套代码兼容多种硬件配置。这些硬件配置的差异信息主要是依靠dts进行区分。当编译kernel的时候，你发现你很难确定就是你用的是哪个dts。可能你就会凭借自己多年的工作经验去猜测。是的，你的经验很厉害。但是，如何确定你的dts信息是否添加到kernel使用的dts呢？我觉得你应该got it。就到这个目录去查找是否包含你添加的ndoe和property。dts中有status属性可以设置某个devicec是否使能。当你发现你的driver的probe没有执行的时候，我觉得你就需要确定一遍status是不是“ok”、“okay”或者没有这个属性。

远不止我所说的这个功能，还可以判断当前的硬件是匹配哪个dts文件。你还不去探索一波源码的设计与实现吗？节点为什么出现在某些目录？为什么有些节点的属性cat却是乱码？就是乱码，你没看错。至于为什么。点到为止。

# **sysfs可以看出什么猫腻**

sysfs有什么用？sysfs可以看出device是否注册成功、存在哪些device、driver是否注册、device和driver是都匹配、device匹配的driver是哪个等等。先说第一项技能。deivce是否注册。

就以i2c设备为例说明。/sys/bus/i2c/devices该目录下面全是i2c总线下面的devices。如何确定自己的device是否注册呢？首选你需要确定自己的device挂接的总线是哪个i2c（i2c0, i2c1...）。假设设备挂接i2c3，从地址假设0x55。那么devices目录只需要查看是否有3-0055目录即可。如何确定device是否匹配了驱动？进入3-0055目录，其实你可以看到driver的符号链接。如果没有，那么就是没有driver。driver是否注册如何确定呢？方法类似。/sys/bus/i2c/drivers目录就是所有注册的i2c driver。方法如上。不再列举。

你以为就这些简单的功能了吗？其实不是，还有很多待你探讨。主要是各种目录之间的关系，device注册会出现什么目录？那么driver呢？各种符号链接会在那些目录？等等。

# **如何排查driver的probe没有执行问题**

我想这类问题是移植过程中极容易出现的第一个拦路虎。这里需要声明一点。可能有些驱动工程师认为probe没有执行就是driver没有注册成功。其实这两个没有半毛钱关系。probe是否执行只和device和driver的是否匹配成功有关。我们有什么思路排查这类问题呢？主要排查方法可以如下。

> 1. 通过sysfs确定对应的device是否注册成功。
> 1. 通过sysfs确定对应的driver是否注册成功。
> 1. 通过sysfs中dts的展开信息得到compatible属性的值，然后和driver的compatible进行对比。

如果发现device没有注册，如何确定问题。首先通过sysfs中dts展开文件查看是否有你添加的device信息。如果没有的话，没有device注册就是正常的，去找到正确的dts添加正确的device信息。如果发现sysfs有device的dts信息。那么就看看status属性的值是不是ok的。如果也是ok的。那么就非常奇怪了。这种情况一般出现在kernel的device注册路径出错了。曾经就有一个小伙伴提出问题，现象就是这样。最后我帮他确定问题原因是dts章reg属性的地址是0xc0。这是一个大于0x7f的值。在i2c总线注册驱动的时候会解析当前总线下的所有device。然后注册所有的从设备。就是这里的地址检查出了问题，因此这个device没有注册上。

如果发现driver没有注册，那么去看看对应的Makefile是否参与编译。如果参与了编译，就查看log信息，是不是驱动注册的时候有错误信息。

最后一点的compatible属性的值只需要cat一下，然后compare即可。不多说。

# **后记**

sysfs还有很多的其他的调试信息可以查看。因此，我建议驱动工程师都应该掌握sysfs的使用和原理。看看代码实现。我始终坚信只有更好地掌握技术的原理才能更好地利用技术。文章内容不想展开。我告诉你的结果，你永远记忆不深刻，而且我说的也不一定对。还是需要自己专研。我只是给你指条明路，剩下的就需要自己去走。最后说一句，代码不会骗你，还会告诉你别人不能告诉你的。

---

« [ftrace时间精度issue修复](http://www.wowotech.net/linux_kenrel/ftrace-clock-issue.html) | [SLUB DEBUG原理](http://www.wowotech.net/memory_management/427.html)»

**评论：**

**xiaotonga**\
2023-10-15 20:26

驱动节点通过sysfs暴露给用户，通过sysfs可以查看驱动节点信息，打call

[回复](http://www.wowotech.net/device_model/429.html#comment-8836)

**一只卤蛋**\
2023-04-11 20:02

作者现在不更新了么，还是换了平台了呢，有没有朋友知道的

[回复](http://www.wowotech.net/device_model/429.html#comment-8770)

**凉冰**\
2022-11-16 16:36

什么时候出一篇致入门小白的一封信啊，好多的点到为止，小白真就点到为止了。

[回复](http://www.wowotech.net/device_model/429.html#comment-8699)

**Since**\
2020-03-16 18:06

写的很棒，都是经验之谈

[回复](http://www.wowotech.net/device_model/429.html#comment-7921)

**Teddy**\
2018-07-31 14:44

谢谢，写的很棒，记住这些用法，调试驱动的时候就能事半功倍了。

[回复](http://www.wowotech.net/device_model/429.html#comment-6858)

**davidlamb**\
2018-05-30 14:41

这种文章很好！

[回复](http://www.wowotech.net/device_model/429.html#comment-6774)

**Vernon**\
2018-04-26 12:17

很喜欢这种讲解风格，点到为止，又给我们指条明路

[回复](http://www.wowotech.net/device_model/429.html#comment-6704)

**lcbj**\
2018-04-20 10:01

请教下，如果引脚配置为输入，那么类似gpios = \<2 GPIO_ACTIVE_XXX>中的GPIO_ACTIVE_XXX是不是无所谓配高配低？

[回复](http://www.wowotech.net/device_model/429.html#comment-6684)

**[smcdef](http://www.wowotech.net/)**\
2018-04-22 17:36

@lcbj：取决你调用什么接口以及如何配置。例如下面的函数。\
int gpiod_get_value(const struct gpio_desc \*desc)\
{\
int value;

VALIDATE_DESC(desc);\
/\* Should be using gpio_get_value_cansleep() \*/\
WARN_ON(desc->gdev->chip->can_sleep);

value = gpiod_get_raw_value_commit(desc);\
if (value \< 0)\
return value;

if (test_bit(FLAG_ACTIVE_LOW, &desc->flags))\
value = !value;

return value;\
}\
看看这个if (test_bit(FLAG_ACTIVE_LOW, &desc->flags))value = !value;你的疑问可以通过全局搜索FLAG_ACTIVE_LOW，或者你的driver调用的接口追踪函数是否用到该flag

[回复](http://www.wowotech.net/device_model/429.html#comment-6690)

**essaypinglun 作家**\
2018-04-17 20:02

有设备驱动程序，您可以在其中找到Linux驱动程序开发并获得Promer设备，通过该设备可以使用此模型享受核心逻辑设计。 你也可以从抽象模型中获得帮助。

[回复](http://www.wowotech.net/device_model/429.html#comment-6673)

**DarkCom**\
2018-04-16 15:15

我最近正在学习IIO子系统及其驱动的相关东西，但是学的不是很懂。看了您关于子系统的文章获益匪浅，请问您对IIO这方面有了解吗？以后能写一篇关于IIO的文章吗？还是已经有了但我没有找到...谢谢

[回复](http://www.wowotech.net/device_model/429.html#comment-6670)

1 [2](http://www.wowotech.net/device_model/429.html/comment-page-2#comments)

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

  - ja\
    [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
  - 元神高手\
    [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
  - 十七\
    [内核空间的映射在系统启动时就已经设定好，并且在所有进程的页表...](http://www.wowotech.net/process_management/context-switch-arch.html#8920)
  - lw\
    [sparse模型和disconti模型没看出来有什么本质区别...](http://www.wowotech.net/memory_management/memory_model.html#8919)
  - 肥饶\
    [一个没设置好就出错](http://www.wowotech.net/linux_kenrel/516.html#8918)
  - orange\
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

  - [Linux2.6.23 ：sleepable RCU的实现](http://www.wowotech.net/kernel_synchronization/linux2-6-23-RCU.html)
  - [中断唤醒系统流程](http://www.wowotech.net/irq_subsystem/irq_handle_procedure.html)
  - [eMMC 原理 4 ：总线协议](http://www.wowotech.net/basic_tech/emmc_bus_protocol.html)
  - [X-014-KERNEL-ARM GIC driver的移植](http://www.wowotech.net/x_project/gic_driver_porting.html)
  - [Linux vm运行参数之（二）：OOM相关的参数](http://www.wowotech.net/memory_management/oom.html)

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

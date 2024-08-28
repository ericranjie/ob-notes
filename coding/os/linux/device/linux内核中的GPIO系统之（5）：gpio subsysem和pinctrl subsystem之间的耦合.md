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

作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2017-8-10 22:17 分类：[GPIO子系统](http://www.wowotech.net/sort/gpio_subsystem)

## 1. 前言

按理说，kernel中gpio subsystem和pinctrl subsystem的关系应该非常清楚：

> pinctrl subsystem管理系统的所有管脚，GPIO是这些管脚的用途之一，因此gpio subsystem应该是pinctrl subsystem的client（也可叫做backend、consumer），基于pinctrl subsystem提供的功能，处理GPIO有关的逻辑。 

不过，实际情况却不是这么简单，它们之间有着较为紧密的耦合（看一看kernel中pinctrl和gpio相关的实现就知道了）。本文将对这种耦合进行一个简单的分析，解释为什么要这样设计。

## 2. Why

首先，无论硬件的架构如何（可参考[[2]](http://www.wowotech.net/gpio_subsystem/pin-control-subsystem.html/)中“五、和GPIO subsystem交互”），从逻辑上讲，有一点是很明确的（这一点和linuxer同学在[2]中的说明有出入，待讨论）：

> gpio subsystem永远是pinctrl的backend（或client，或consumer）。

基于这一点，规范的做法，任何一个gpio chip（相关的概念可参考本站GPIO子系统的文章[5]），在使用GPIO的时候（通常是gpio subsystem的consumer申请GPIO资源的时候），都需要向系统的pinctrl subsystem申请管脚，并将管脚配置为GPIO功能。

思路是简单、直接的，但实际操作起来，却有点棘手，下面以一个最简单的例子说明：

> 假设某一个gpio chip只包括2个gpio，这两个gpio分别和uart进行功能复用。
> 
> 如果这两个管脚是同时控制的，要么是gpio，要么是uart，就很好处理了，按照pinctrl subsystem的精神，抽象出两个function：gpio和uart，gpio chip在使用gpio功能的时候，调用pinctrl set state，将它们切换为gpio即可。
> 
> 但是，如果这两个gpio可以分开控制（很多硬件都是这样的设计的），麻烦就出现了，每个gpio要单独抽象为一个function，因此我们可以抽象出3个function：gpio1、gpio2和uart。
> 
> 然后考虑一下一个包含32个gpio的chip（一般硬件的gpio bank都是32个），如果它们都可以单独控制，则会出现32个function。而系统又不止有一个chip，灾难就发生了，我们的device tree文件将会被一坨坨的gpio functions撑爆！

规范是我们追求的目标，但有限度，不能让上面的事情发生，怎么办呢？在pinctrl subsystem的标准框架上打个洞，只要碰到这种情况，直接就走后门就是了。

## 3. pinctrl中和gpio有关的后门

后门是什么呢？其实很简单，参考下面的API：

|   |
|---|
|static inline int pinctrl_request_gpio(unsigned gpio) ;  <br>static inline void pinctrl_free_gpio(unsigned gpio) ;  <br>static inline int pinctrl_gpio_direction_input(unsigned gpio);  <br>static inline int pinctrl_gpio_direction_output(unsigned gpio);|

> 当gpio driver需要使用某个管脚的时候，直接调用pinctrl_request_gpio，向pinctrl subsystem申请。
> 
> pinctrl subsystem会维护一个gpio number到pin number的map，将gpio subsystem传来的gpio number转换为pin number之后，调用struct pinmux_ops中有关的回调函数即可：
> 
> struct pinmux_ops {  
>         ...  
>         int (*gpio_request_enable) (struct pinctrl_dev *pctldev,  
>                                      struct pinctrl_gpio_range *range,  
>                                      unsigned offset);  
>         void (*gpio_disable_free) (struct pinctrl_dev *pctldev,  
>                                     struct pinctrl_gpio_range *range,  
>                                     unsigned offset);  
>         int (*gpio_set_direction) (struct pinctrl_dev *pctldev,  
>                                     struct pinctrl_gpio_range *range,  
>                                     unsigned offset,  
>                                     bool input);  
> };
> 
> 对gpio driver来说，要做的事情就是提供gpio number到pin number的map。
> 
> 而pinctrl subsystem呢，做自己分内的事情即可：管理系统的pin资源，并根据gpio subsystem的请求，将某些pin设置为GPIO功能。

## 4. gpio range----gpio number到pin number的map

那么，怎么提供gpio number和pin number的map呢？是通过一个名称为gpio range的数据结构(pinctrl subsystem提供的），如下：

|   |
|---|
|/* include/linux/pinctrl/pinctrl.h */  <br>  <br>/**  <br>  * struct pinctrl_gpio_range - each pin controller can provide subranges of  <br>  * the GPIO number space to be handled by the controller  <br>  * @node: list node for internal use  <br>  * @name: a name for the chip in this range  <br>  * @id: an ID number for the chip in this range  <br>  * @base: base offset of the GPIO range  <br>  * @pin_base: base pin number of the GPIO range if pins == NULL  <br>  * @pins: enumeration of pins in GPIO range or NULL  <br>  * @npins: number of pins in the GPIO range, including the base number  <br>  * @gc: an optional pointer to a gpio_chip  <br>  */  <br>struct pinctrl_gpio_range {  <br>        struct list_head node;  <br>        const char *name;  <br>        unsigned int id;  <br>         unsigned int base;  <br>        unsigned int pin_base;  <br>        unsigned const *pins;  <br>        unsigned int npins;  <br>        struct gpio_chip *gc;  <br>};|

> 该数据结构很容易理解，总结来说，就是：gpio controller(gc)中的gpio(base)到gpio(base + npins - 1)，和pin controller中的pin(pin_base)到pin(pin_base + npins - 1)是一一对应的。
> 
> 有了这个对应关系之后，pinctrl subsystem就可以将任意一个gpio controller中的gpio number转换为相应的pin controller中的pin number。

最后，gpio subsystem为了方便gpio driver的开发，提供了一种简单的、可以通过device tree提供gpio range信息的方法，总结如下：

1）gpio range的dts格式示例（可参考[6]中的介绍）

>         qe_pio_e: gpio-controller@1460 {  
>                 #gpio-cells = <2>;  
>                 compatible = "fsl,qe-pario-bank-e", "fsl,qe-pario-bank";  
>                 reg = <0x1460 0x18="">;  
>                 gpio-controller;  
>                 gpio-ranges = <&pinctrl1 0 20 10>, <&pinctrl2 10 50 20>;  
>         };

上面dts节点中的gpio-ranges关键字指定了两个gpio range：gpio controller(qe_pio_e)中的gpio0~9和pinctrl1中的pin20~29对应；gpio controller(qe_pio_e)中的gpio10~29和pinctrl2中的pin50~69对应。

2）gpio range dts node的解析

解析的过程如下（具体不再详细介绍了，大家可参考相应的代码）：

> devm_gpiochip_add_data(drivers/gpio/gpiolib.c)  
>        gpiochip_add_data  
>               of_gpiochip_add(drivers/gpio/gpiolib-of.c)  
>                      of_gpiochip_add_pin_range  
>                             gpiochip_add_pin_range(drivers/gpio/gpiolib.c)  
>                                    pinctrl_find_and_add_gpio_range(drivers/pinctrl/core.c)  
>                                           pinctrl_add_gpio_range

以上过程的结果，就是在相应的pin controller device的数据结构中生成了一个gpio range的链表，如下：

> struct pinctrl_dev {  
>         struct list_head node;  
>         struct pinctrl_desc *desc;  
>         struct radix_tree_root pin_desc_tree;  
>         struct list_head gpio_ranges;  
>         ...  
> };

3）gpio range的使用

当gpio driver需要使用某一个gpio的时候，可以在gpiochip的request函数中，调用pinctrl core提供的pinctrl_request_gpio接口（参数是gpio编号），然后pinctrl core会查寻gpio ranges链表，将gpio编号转换成pin编号，然后调用pinctrl的相应接口（参数是pin编号），申请该pin的使用。

至此，gpio subsystem和pinctrl subsystem的耦合，就真相大白了。

## 5. 参考文档 

[1] [https://github.com/wowotechX/linux/blob/x_integration/Documentation/pinctrl.txt](https://github.com/wowotechX/linux/blob/x_integration/Documentation/pinctrl.txt "https://github.com/wowotechX/linux/blob/x_integration/Documentation/pinctrl.txt")

[2] [linux内核中的GPIO系统之（2）：pin control subsystem](http://www.wowotech.net/gpio_subsystem/pin-control-subsystem.html)

[3] [linux内核中的GPIO系统之（4）：pinctrl驱动的理解和总结](http://www.wowotech.net/gpio_subsystem/pinctrl-driver-summary.html)

[4] [X-023-KERNEL-Linux pinctrl driver的移植](http://www.wowotech.net/x_project/kernel_pinctrl_driver_porting.html)

[5] [http://www.wowotech.net/sort/gpio_subsystem](http://www.wowotech.net/sort/gpio_subsystem "http://www.wowotech.net/sort/gpio_subsystem")

[6] Documentation/devicetree/bindings/gpio/gpio.txt

原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/gpio_subsystem/pinctrl-and-gpio.html)。

标签: [GPIO](http://www.wowotech.net/tag/GPIO) [subsystem](http://www.wowotech.net/tag/subsystem) [pinctrl](http://www.wowotech.net/tag/pinctrl) [range](http://www.wowotech.net/tag/range)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [/proc/meminfo分析（一）](http://www.wowotech.net/memory_management/meminfo_1.html) | [《奔跑吧，Linux内核》已经上架预售了](http://www.wowotech.net/tech_discuss/running_kernel.html)»

**评论：**

**wulala**  
2023-08-28 17:41

wowo前辈 请问另有博客或者公众号吗

[回复](http://www.wowotech.net/gpio_subsystem/pinctrl-and-gpio.html#comment-8810)

**入行真的好难**  
2023-08-29 16:43

@wulala：期待wowo网站再次“忙碌起来”

[回复](http://www.wowotech.net/gpio_subsystem/pinctrl-and-gpio.html#comment-8811)

**tmmdh**  
2019-05-19 16:42

终于把gpio系列看完了，结合自己分析Linux源码，整整用了半个月，无法想象博主究竟是何许人也，能把一个gpio子系统剖析的如此详细，就像是在介绍自己设计的一个子系统一样，佩服的五体投地！

[回复](http://www.wowotech.net/gpio_subsystem/pinctrl-and-gpio.html#comment-7428)

**linux小鸟**  
2019-02-12 17:17

Hi 博主，我之前是做固件开发（偏向应用），对驱动部分不是很了解。关于GPIO部分，之前只是配置GPIO，并不了解详细原理。  
  
个人觉得，可能增加一些框图，少一些具体代码，可能会更加利于我这样的“驱动外行人”理解。  
  
谢谢！！

[回复](http://www.wowotech.net/gpio_subsystem/pinctrl-and-gpio.html#comment-7180)

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
    
    - [蓝牙协议中LQ和RSSI的原理及应用场景](http://www.wowotech.net/bluetooth/lqi_rssi.html)
    - [智慧家庭之我见_多媒体篇](http://www.wowotech.net/tech_discuss/zhjt_dmt.html)
    - [Linux CPU core的电源管理(5)_cpu control及cpu hotplug](http://www.wowotech.net/pm_subsystem/cpu_hotplug.html)
    - [MMC/SD/SDIO介绍](http://www.wowotech.net/basic_tech/mmc_sd_sdio_intro.html)
    - [Linux时间子系统之（六）：POSIX timer](http://www.wowotech.net/timer_subsystem/posix-timer.html)
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
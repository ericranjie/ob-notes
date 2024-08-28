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

作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2017-9-1 10:46 分类：[电源管理子系统](http://www.wowotech.net/sort/pm_subsystem)

## 1. 前言

大家都知道，复杂IC内部有很多具有独立功能的硬件模块，例如CPU cores、GPU cores、USB控制器、MMC控制器、等等，出于功耗、稳定性等方面的考虑，有些IC在内部为这些硬件模块设计了复位信号（reset signals），软件可通过寄存器（一般1个bit控制1个硬件）控制这些硬件模块的复位状态。

Linux kernel为了方便设备驱动的编写，抽象出一个简单的软件框架----reset framework，为reset的provider提供统一的reset资源管理手段，并为reset的consumer（各个硬件模块）提供便捷、统一的复位控制API。

reset framework的思路、实现和使用都非常简单、易懂（参考kernel有关的API--include/linux/reset-controller.h、include/linux/reset.h可知），不过麻雀虽小，五脏俱全，通过它可以加深对Linux kernel的设备模型、驱动框架、分层设计、provider/consumer等设计思想的理解，因此本文将对其进行一个简单的罗列和总结。

## 2. 从consumer的角度看

从某一个硬件模块的驱动设计者来看，他的要求很简单：我只是想复位我的硬件，而不想知道到底用什么手段才能复位（例如控制哪个寄存器的哪个bit位，等等）。

> 这个要求其实体现了软件设计（甚至是任何设计）中的一个最最质朴的设计理念：封装和抽象。对设备驱动来说，它期望看到是“reset”这个通用概念，用这个通用概念去发号施令的话，这个驱动就具备了通用性和可移植性（无论在周围的环境如何变化，“reset”本身不会变化）。而至于怎么reset，是通过寄存器A的bit m，还是寄存器B的bit n，则是平台维护者需要关心的事情（就是本文的reset provider）。

看到这样的要求，Linux kernel说：OK，于是reset framework出场，提供了如下的机制（基于device tree）：

1）首先，提供描述系统中reset资源的方法（参考下面第3章的介绍），这样consumer可以基于这种描述在自己的dts node中引用所需的reset信号。

2）然后，consumer设备在自己的dts node中使用“resets”、“reset-names”等关键字声明所需的reset的资源，例如[1]（“resets”字段的具体格式由reset provider决定”）：

> device {                                                                 
>         resets = <&rst 20>;                                              
>         reset-names = "reset";                                           
> };

3）最后，consumer driver在需要的时候，可以调用下面的API复位自己（具体可参考“include/linux/reset.h“）：

3-a）只有一个reset信号的话，可以使用最简单的device_reset API

> int device_reset(struct device *dev);

3-b）如果需要更为复杂的控制（例如有多个reset信号、需要控制处于reset状态的长度的等），可以使用稍微复杂的API

> /* 通过reset_control_get或者devm_reset_control_get获得reset句柄 */  
> struct reset_control *reset_control_get(struct device *dev, const char *id);     
> void reset_control_put(struct reset_control *rstc);                              
> struct reset_control *devm_reset_control_get(struct device *dev, const char *id);
> 
> /* 通过reset_control_reset进行复位，或者通过reset_control_assert使设备处于复位生效状态，通过reset_control_deassert使复位失效 */  
> int reset_control_reset(struct reset_control *rstc);                             
> int reset_control_assert(struct reset_control *rstc);                            
> int reset_control_deassert(struct reset_control *rstc);

## 3. 从provider的角度看

kernel为reset provider提供的API位于“include/linux/reset-controller.h”中，很简单，无非就是：创建并填充reset controller设备（struct reset_controller_dev），并调用相应的接口（reset_controller_register/reset_controller_unregister）注册或者注销之。

reset controller的抽象也很简单：

|   |
|---|
|struct reset_controller_dev {                                                    <br>        struct reset_control_ops *ops;                                           <br>        struct module *owner;                                                    <br>        struct list_head list;                                                   <br>        struct device_node *of_node;                                             <br>        int of_reset_n_cells;                                                    <br>        int (*of_xlate)(struct reset_controller_dev *rcdev,                      <br>                        const struct of_phandle_args *reset_spec);               <br>        unsigned int nr_resets;  <br>};|

> ops提供reset操作的实现，基本上是reset provider的所有工作量。
> 
> of_xlate和of_reset_n_cells用于解析consumer device dts node中的“resets = ; ”节点，如果reset controller比较简单（仅仅是线性的索引），可以不实现，使用reset framework提供的简单版本----of_reset_simple_xlate即可。
> 
> nr_resets，该reset controller所控制的reset信号的个数。
> 
> 其它字段内部使用，provider不需要关心。

struct reset_control_ops也比较单纯，如下：

|   |
|---|
|struct reset_control_ops {                                                       <br>        int (*reset)(struct reset_controller_dev *rcdev, unsigned long id);      <br>        int (*assert)(struct reset_controller_dev *rcdev, unsigned long id);     <br>        int (*deassert)(struct reset_controller_dev *rcdev, unsigned long id);  <br>};|

> reset可控制设备完成一次完整的复位过程。
> 
> assert和deassert分别控制设备reset状态的生效和失效。

## 4. 参考文档

[1] Documentation/devicetree/bindings/reset/reset.txt

_原创文章，转发请注明出处。蜗窝科技_，[www.wowotech.net](http://www.wowotech.net/pm_subsystem/reset_framework.html)。

标签: [Linux](http://www.wowotech.net/tag/Linux) [Kernel](http://www.wowotech.net/tag/Kernel) [内核](http://www.wowotech.net/tag/%E5%86%85%E6%A0%B8) [framework](http://www.wowotech.net/tag/framework) [reset](http://www.wowotech.net/tag/reset)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [蓝牙协议分析(11)_BLE安全机制之SM](http://www.wowotech.net/bluetooth/le_security_manager.html) | [页面回收的基本概念](http://www.wowotech.net/memory_management/page_reclaim_basic.html)»

**评论：**

**shousi**  
2024-06-09 20:18

挺巧妙的

[回复](http://www.wowotech.net/pm_subsystem/reset_framework.html#comment-8906)

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
    
    - [CMA模块学习笔记](http://www.wowotech.net/memory_management/cma.html)
    - [快讯：蓝牙5.0发布（新特性速览）](http://www.wowotech.net/bluetooth/bluetooth_5_0_overview.html)
    - [从“码农”说起](http://www.wowotech.net/tech_discuss/111.html)
    - [内存初始化代码分析（三）：创建系统内存地址映射](http://www.wowotech.net/memory_management/mem_init_3.html)
    - [防冲突机制介绍](http://www.wowotech.net/basic_tech/103.html)
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
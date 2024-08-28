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

作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2017-9-27 22:27 分类：[X Project](http://www.wowotech.net/sort/x_project)

## 1. 前言

我们在[1][2]中提到过，鉴于gpio的特殊性，pinctrl subsystem特意留了一个后门（gpio range），gpio driver可以通过这个后门直接向pinctrl subsystem申请将某个pin用作gpio功能。本文将根据一个简单的示例，介绍这个后门的使用方法，以加深对相关机制的理解。

注1：本文的测试方法和[3]中的一致，即：通过gpiolib sysfs api控制LED0（GPIOA19）的亮灭，因而不再罗列详细步骤。

## 2. 移植步骤

由[2]可知，gpio range的主要目的就是将gpio命名空间（gpio）转换为pinctrl命名空间（pin），并由pinctrl subsystem访问硬件实现gpio有关的功能控制。因此gpio range的移植步骤注要包括：

#### 2.1 pinctrl命名空间和gpio命名空间的定义

参考”[X-025-KERNEL-Linux gpio driver的移植之基本功能](http://www.wowotech.net/x_project/kernel_gpio_driver_porting_1.html)[3]”中有关gpiochip的实现，本例中的GPIOA19的gpio命名空间为：

> gpiochip：gpioa  
> 编号：19

同理，按照“[X-023-KERNEL-Linux pinctrl driver的移植](http://www.wowotech.net/x_project/kernel_pinctrl_driver_porting.html)[4]”中的方法，结合bubblegum-96的原理图，我们可以把GPIOA19所在的管脚编号为"F3"，它对应的pinctrl命名空间为：

> pin controller：pinctrl@0xe01b0000  
> 编号：52
> 
> @@ -68,6 +68,7 @@ static const struct pinctrl_pin_desc s900_pins[] = {  
>         PINCTRL_PIN(15, "B6"),  
>         PINCTRL_PIN(24, "C5"),  
>         PINCTRL_PIN(37, "D8"),  
> +       PINCTRL_PIN(52, "F3"),  
>         PINCTRL_PIN(60, "G1"),  
>          PINCTRL_PIN(61, "G2"),  
>   };

#### 2.2 将gpio number转换为pin number

命名空间定义完成后，可以按照[2]中的步骤，在dts中定义一个gpio range，将gpio number转换为pin number，如下：

> @@ -39,7 +39,7 @@  
>                 clock-frequency = <24000000>;  
>         };
> 
> -       pinctrl@0xe01b0000 {  
> +       pinctrl1: pinctrl@0xe01b0000 {  
>                  compatible = "actions,s900-pinctrl";  
>                  reg = <0 0="" 0xe01b0000="" 0x550="">;
> 
> @@ -56,6 +56,7 @@  
>         gpioa: gpio@0xe01b0000 {  
>                  compatible = "actions,s900-gpio";  
>                 reg = <0 0="" 12="" 0xe01b0000="">;  
>                 base = <0>;  
> +               gpio-ranges = <&pinctrl1 19 52 1>;  
>         };

其中黄色背景那一行的含义是：将gpioa中的19号gpio，和pinctrl1中的52号pin，对应。

#### 2.3 修改gpio driver和pinctrl driver，二者配合，完成gpio的request、free、direction_input以及direction_output等操作

1）修改pinctrl driver，实现pinmux_ops中gpio_request_enable、gpio_disable_free、gpio_set_direction等回调函数

这三个API的输入参数都是range指针和offset（如下所示），通过它们可以找到这个GPIO所在的gpiochip、GPIO bank、GPIO number、对应pin所在的pin controller、pin number等信息。基于这些信息，可以获得相应的硬件信息（寄存器、bit偏移等）。

> int (*gpio_request_enable) (struct pinctrl_dev *pctldev,  
>                              struct pinctrl_gpio_range *range,  
>                              unsigned offset);  
> void (*gpio_disable_free) (struct pinctrl_dev *pctldev,  
>                            struct pinctrl_gpio_range *range,  
>                            unsigned offset);  
> int (*gpio_set_direction) (struct pinctrl_dev *pctldev,  
>                            struct pinctrl_gpio_range *range,  
>                            unsigned offset,  
>                             bool input);

2）修改gpio chip的.request、.free、.direction_input、.direction_output等回调函数，让它们调用pinctrl subsystem提供的相关API，如下：

> int pinctrl_request_gpio(unsigned gpio) ;   
> void pinctrl_free_gpio(unsigned gpio) ;  
> int pinctrl_gpio_direction_input(unsigned gpio);  
> int pinctrl_gpio_direction_output(unsigned gpio);  

注2：有些硬件平台，在完成上面步骤1）的时候，可能会遇到一些困扰，例如怎么根据gpio和pin的信息，找到对应的硬件控制信息（寄存器、bit偏移等），这时我们可以灵活处理。例如在本文例子所使用的bubblegum-96平台上，gpio的pinmux功能并没有单独的寄存器控制，而是通过gpio的in或者out功能的使能，覆盖其它的pinmux功能。此时我们可以把硬件配置的操作交给gpio driver，而pinctrl driver只处理管脚的互斥。具体可参考下面patch的改动。

patch：[https://github.com/wowotechX/linux/commit/cf4d492855d0619772876bc8494b7e180c6a4232](https://github.com/wowotechX/linux/commit/cf4d492855d0619772876bc8494b7e180c6a4232 "https://github.com/wowotechX/linux/commit/cf4d492855d0619772876bc8494b7e180c6a4232")

## 3. 测试步骤

请参考[3]中的测试。测试结果如下（看到s900_gpio_request_enable中的打印，就说明我们的移植成功了）：

> / # mkdir /sys  
> / # mount -t sysfs /sys /sys
> 
> / # echo 19 > /sys/class/gpio/export  
> [   55.136218] owl_pinctrl e01b0000.pinctrl: s900_gpio_request_enable, range(19/52/1), offset 52
> 
> / # echo out > /sys/class/gpio/gpio19/direction  
> [   86.886343] owl_gpio e01b0000.gpio: offset 19, value 0  
>   
> / # echo 1 > /sys/class/gpio/gpio19/value  
> [  100.223812] owl_gpio e01b0000.gpio: offset 19, value 1  
>   
> / # echo 0 > /sys/class/gpio/gpio19/value  
> [  120.467468] owl_gpio e01b0000.gpio: offset 19, value 0

## 4. 参考文档

[1] [linux内核中的GPIO系统之（4）：pinctrl驱动的理解和总结](http://www.wowotech.net/gpio_subsystem/pinctrl-driver-summary.html)

[2] [linux内核中的GPIO系统之（5）：gpio subsysem和pinctrl subsystem之间的耦合](http://www.wowotech.net/gpio_subsystem/pinctrl-and-gpio.html)

[3] [X-025-KERNEL-Linux gpio driver的移植之基本功能](http://www.wowotech.net/x_project/kernel_gpio_driver_porting_1.html)

[4] [X-023-KERNEL-Linux pinctrl driver的移植](http://www.wowotech.net/x_project/kernel_pinctrl_driver_porting.html)

[5] Schematics_Bubblegum96.pdf

原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/x_project/kernel_gpio_driver_porting_2.html)。

标签: [driver](http://www.wowotech.net/tag/driver) [GPIO](http://www.wowotech.net/tag/GPIO) [porting](http://www.wowotech.net/tag/porting) [pinctrl](http://www.wowotech.net/tag/pinctrl) [range](http://www.wowotech.net/tag/range)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Linux kernel scatterlist API介绍](http://www.wowotech.net/memory_management/scatterlist.html) | [Device Tree（四）：文件结构解析](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html)»

**评论：**

**bigpillow**  
2017-11-10 10:54

刚写完一套GPIO driver.  
当时考虑到不同chip通用性使用了gpiochip_add_pin_range function  
现在看看在DTS里面原来这么方便。

[回复](http://www.wowotech.net/x_project/kernel_gpio_driver_porting_2.html#comment-6189)

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
    
    - [DRAM 原理 4 ：DRAM Timing](http://www.wowotech.net/basic_tech/330.html)
    - [Linux kernel的中断子系统之（六）：ARM中断处理过程](http://www.wowotech.net/irq_subsystem/irq_handler.html)
    - [ARM64的启动过程之（二）：创建启动阶段的页表](http://www.wowotech.net/armv8a_arch/create_page_tables.html)
    - [Linux common clock framework(1)_概述](http://www.wowotech.net/pm_subsystem/clk_overview.html)
    - [Concurrency Managed Workqueue之（三）：创建workqueue代码分析](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html)
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
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

作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2016-10-31 22:23 分类：[X Project](http://www.wowotech.net/sort/x_project)

## 1. 前言

“[X Project](http://www.wowotech.net/sort/x_project)”完成“[X-012-KERNEL-serial early console的移植](http://www.wowotech.net/x_project/kernel_earlycon_porting.html)”之后，终止在如下的kernel panic中：

> NR_IRQS:64 nr_irqs:64 0\
> Kernel panic - not syncing: No interrupt controller found.\
> ---\[ end Kernel panic - not syncing: No interrupt controller found.

结果很明显，系统中没有注册中断控制器。因此，本文将以“Bubbugum-96”平台为例，介绍ARM GIC驱动的移植步骤，顺便继续加深对device tree的理解和认识。

注1：由于“Bubbugum-96”的GIC符合ARM标准，Linux kernel中相关的驱动是现成的，因此GIC驱动的移植就非常简单了，只要配置一下device tree即可。

## 2. GIC硬件说明

由\[1\]中的关键字可知，bubblegum-96所使用的SOC（S900）符合GIC_400规范。根据\[3\]中GIC driver的分析可知，配置GIC驱动需要知道如下的信息：

> Distributor address range\
> CPU interface address range\
> Virtual interface control block\
> Virtual CPU interfaces

不过S900的公开资料却没有这方面的信息，没关系，我们可以从它们开放出来的源代码反推，如下\[4\]：

> gic: interrupt-controller@e00f1000 {\
> compatible = "arm,cortex-a15-gic", "arm,cortex-a9-gic";\
> #interrupt-cells = \<3>;\
> #address-cells = \<0>;\
> interrupt-controller;\
> reg = \<0 0="" 0xe00f1000="" 0x1000="">,\
> \<0 0="" 0xe00f2000="" 0x1000="">,\
> \<0 0="" 0xe00f4000="" 0x2000="">,\
> \<0 0="" 0xe00f6000="" 0x2000="">;               interrupts = ;\
> };

即：

> Distributor address range：0xe00f1000 ~ 0xe00f1000 + 0x1000\
> CPU interface address range：0xe00f2000 ~ 0xe00f1000 + 0x1000\
> Virtual interface control block：0xe00f4000 ~ 0xe00f4000 + 0x2000\
> Virtual CPU interfaces：0xe00f6000 ~ 0xe00f6000 + 0x2000

## 3. GIC driver移植

移植过程很简单，修改dts文件，加入gic的节点即可（可以抄上面的~~）：

> diff --git a/arch/arm64/boot/dts/actions/s900-bubblegum.dts b/arch/arm64/boot/dts/actions/s900-bubblegum.dts\
> index fb1351a..a3af064 100644\
> --- a/arch/arm64/boot/dts/actions/s900-bubblegum.dts\
> +++ b/arch/arm64/boot/dts/actions/s900-bubblegum.dts\
> @@ -10,4 +10,16 @@\
> / {\
> model = "Bubblegum-96 Development Board";\
> compatible = "actions,s900-bubblegum", "actions,s900";\
> +\
> +       #address-cells = \<2>;\
> +       #size-cells = \<2>;\
> +\
> +       gic: interrupt-controller@e00f1000 {\
> +               compatible = "arm,gic-400";\
> +               #interrupt-cells = \<3>;\
> +               #address-cells = \<0>;\
> +               interrupt-controller;\
> +               reg = \<0 0="" 0xe00f1000="" 0x1000="">,  /\* dist_base */\
> +                     \<0 0="" 0xe00f2000="" 0x1000="">;  /* cpu_base \*/\
> +       };\
> };

简单说明如下：

1）#address-cells和#size-cells

目前为止，我们的dts文件已经裸奔了很久了（因为没有任何的节点），不过现在就不行了。因为有了第一个节点[----interrupt-controller@e00f1000](mailto:----interrupt-controller@e00f1000)，该节点下面有“reg”关键字需要解析，解析的依据是什么呢？也就是说：

> reg关键字中，地址信息有多长？size信息又有多长呢？

这两个信息需要由#address-cells和#size-cells告诉kernel，这里都设置为2，含义是：

> address信息占用两个word的长度，即“0 0xe00f1000”组合成0x00000000e00f1000(Distributor address)，“0 0x1000”组合成0x0000000000001000（Distributor length）。

2）compatible名称为“arm,gic-400”，具体可参考“drivers/irqchip/irq-gic.c”中有关的定义。

3）interrupt-controller用于指示该节点是一个中断控制器。

4）#interrupt-cells用于指明该中断控制器怎么编码某一个中断源，这里的3是ARM GIC driver定义的，意义如下：

> 例如：interrupts = ;
>
> 第一个cell用于指示中断的类型，0是SPI，1是PPI；
>
> 第二个cell用于指示中断号；
>
> 第三个cell用于指示中断的flag，包括： bits\[3:0\]指示中断的触发类型，1 edge triggered、4 level triggered。

5）reg字段我没有写完全，剩下的后续真正用的时候再说了。

## 4. 编译验证

根据“[X Project](http://www.wowotech.net/sort/x_project)”的开发指南，编译并运行新的kernel：

> make kernel uImage
>
> make spl-run
>
> make kernel-run

出现如下的打印：

> clocksource_probe: no matching clocksources found\
> Kernel panic - not syncing: Unable to initialise architected timer.
>
> ---\[ end Kernel panic - not syncing: Unable to initialise architected timer.

panic的对象换了，看来GIC成功了，简单吧？

## 5. GIC的debug和使用

虽然编译、运行通过了，但能否正确使用呢？我们需要一个测试对象，因此，留到下一篇Generic Timer的移植里面，一起介绍吧。

## 6. 参考文档

\[1\] [bubblegum-96/SoC_bubblegum96.pdf](https://github.com/96boards/documentation/blob/master/ConsumerEdition/Bubblegum-96/AdditionalDocs/SoC_bubblegum96.pdf)

\[2\] [bubblegum-96/HardwareManual_Bubblegum96.pdf](https://github.com/uCRDev/documentation/blob/master/ConsumerEdition/Bubblegum-96/AdditionalDocs/HardwareManual_Bubblegum96_S900_V1.1.pdf)

\[3\] [http://www.wowotech.net/linux_kenrel/gic_driver.html](http://www.wowotech.net/linux_kenrel/gic_driver.html)

\[4\] [https://github.com/96boards-bubblegum/linux/blob/bubblegum96-3.10/arch/arm64/boot/dts/s900.dtsi](https://github.com/96boards-bubblegum/linux/blob/bubblegum96-3.10/arch/arm64/boot/dts/s900.dtsi "https://github.com/96boards-bubblegum/linux/blob/bubblegum96-3.10/arch/arm64/boot/dts/s900.dtsi")

原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/x_project/gic_driver_porting.html)。

标签: [Linux](http://www.wowotech.net/tag/Linux) [Kernel](http://www.wowotech.net/tag/Kernel) [内核](http://www.wowotech.net/tag/%E5%86%85%E6%A0%B8) [driver](http://www.wowotech.net/tag/driver) [porting](http://www.wowotech.net/tag/porting) [移植](http://www.wowotech.net/tag/%E7%A7%BB%E6%A4%8D)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Linux kernel debug技巧----开启DEBUG选项](http://www.wowotech.net/linux_application/kernel_debug_enable.html) | [Linux TTY framework(5)\_System console driver](http://www.wowotech.net/tty_framework/system_console_driver.html)»

**评论：**

**breako**\
2018-10-28 21:35

wowo:您好！请问一下，对于arm32没有gic的情况，需要怎么样移植中断这一块呢？

[回复](http://www.wowotech.net/x_project/gic_driver_porting.html#comment-7003)

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

  - [Linux内核同步机制之（三）：memory barrier](http://www.wowotech.net/kernel_synchronization/memory-barrier.html)
  - [X-010-UBOOT-使用booti命令启动kernel(Bubblegum-96平台)](http://www.wowotech.net/x_project/bubblegum_uboot_booti.html)
  - [支持wowo，有这样一块小天地享受技术，感觉棒棒哒！](http://www.wowotech.net/147.html)
  - [Linux电源管理(5)\_Hibernate和Sleep功能介绍](http://www.wowotech.net/pm_subsystem/std_str_func.html)
  - [Device Tree（三）：代码分析](http://www.wowotech.net/device_model/dt-code-analysis.html)

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

作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2016-12-25 21:52 分类：[基础技术](http://www.wowotech.net/sort/basic_tech)

## 1. 前言

熟悉Linux kernel的人都知道，kernel使用MMC subsystem统一管理MMC、SD、SDIO等设备，为什么呢？到底什么是MMC？SD和SDIO又是什么？为什么可以用MMC统称呢？

在分析Linux kernel的MMC subsystem之前，有必要先介绍一些概念，以便对MMC/SD/SDIO有一个大致的了解，这就是本文的目的。

## 2. 基本概念

MMC是MultiMediaCard的简称，从本质上看，它是一种用于固态非易失性存储的内存卡（memory card）规范\[1\]，定义了诸如卡的形态、尺寸、容量、电气信号、和主机之间的通信协议等方方面面的内容。

从1997年MMC规范发布至今，基于不同的考量（物理尺寸、电压范围、管脚数量、最大容量、数据位宽、clock频率、安全特性、是否支持SPI mode、是否支持DDR mode、等等），进化出了MMC、SD、microSD、SDIO、eMMC等不同的规范（如下面图片1所示）。虽然乱花迷人，其本质终究还是一样的，丝毫未变，这就是Linux kernel将它们统称为MMC的原因。

[![mmc_sd_sdio_history](http://www.wowotech.net/content/uploadfile/201612/95d6d6a51a757c21cdc3108e12d16d0320161225135202.gif "mmc_sd_sdio_history")](http://www.wowotech.net/content/uploadfile/201612/45b6e3aeed9e014036481cbdc767d96920161225135201.gif)

图片1 MMC/SD/SDIO evolution

关于该图片，这里强调几点（其它的，大家可参考\[1\]\[2\]，不再详细介绍）：

> MMC、SD、SDIO的技术本质是一样的（使用相同的总线规范，等等），都是从MMC规范演化而来；
>
> MMC强调的是多媒体存储（MM，MultiMedia）；
>
> SD强调的是安全和数据保护（S，Secure）；
>
> SDIO是从SD演化出来的，强调的是接口（IO，Input/Output），不再关注另一端的具体形态（可以是WIFI设备、Bluetooth设备、GPS等等）。

## 3. 规范简介

MMC分别从卡（Card Concept）、总线（Bus Concept）以及控制器（Host Controller）三个方面，定义MMC system的行为，如下面图片2所示：

[![mmc_sd_sdio_hw_block](http://www.wowotech.net/content/uploadfile/201612/fbcc70f4593e41a6f96a28c4667a9c3420161225135203.gif "mmc_sd_sdio_hw_block")](http://www.wowotech.net/content/uploadfile/201612/1dd98218bcd00111510a094902eccf4720161225135203.gif)

图片2 mmc_sd_sdio_hw_block

不同岗位的工程师，可以根据自己的工作性质，重点理解某一部分的规范，下面从嵌入式软件工程师的视角，简单的介绍一下。

#### 3.1 卡的规范

卡的规范主要规定卡的形状、物理尺寸、管脚，内部block组成、寄存器等等，以eMMC为例\[3\]：

[![Card Concept(eMMC)](http://www.wowotech.net/content/uploadfile/201612/4ca87abb20c96c2362ed22855c0fb89a20161225135205.gif "Card Concept(eMMC)")](http://www.wowotech.net/content/uploadfile/201612/78483df39a056808e374a54d62e5534e20161225135204.gif)

图片3 Card Concept(eMMC)

1）有关形状、尺寸的内容，这里不再介绍，感兴趣的同学可参考\[1\]。

2）卡的内部由如下几个block组成：

> Memory core，存储介质，一般是NAND flash、NOR flash等；
>
> Memory core interface，管理存储介质的接口，用于访问（读、写、擦出等操作）存储介质；
>
> Card interface（CMD、CLK、DATA），总线接口，外界访问卡内部存储介质的接口，和具体的管脚相连；
>
> Card interface controller，将总线接口上的协议转换为Memory core interface的形式，用于访问内部存储介质；
>
> Power模块，提供reset、上电检测等功能；
>
> 寄存器（图片1中位于Card interface controller的左侧，那些小矩形），用于提供卡的信息、参数、访问控制等功能。

3）卡的管脚有VDD、GND、RST、CLK、CMD和DATA等，VDD和GND提供power，RST用于复位，CLK、CMD和DATA为MMC总线协议（具体可参考3.2小节）的物理通道：

> CLK有一条，提供同步时钟，可以在CLK的上升沿（或者下降沿，或者上升沿和下降沿）采集数据；
>
> CMD有一条，用于传输双向的命令。
>
> DATA用于传说双向的数据，根据MMC的类型，可以有一条（1-bit）、四条（4-bit）或者八条（8-bit）。

4）以eMMC为例，规范定义了OCR, CID, CSD, EXT_CSD, RCA 以及DSR 6组寄存器，具体含义后面再介绍。

## 3.2 总线规范

前面我们提到过，MMC的本质是提供一套可以访问固态非易失性存储介质的通信协议，从产业化的角度看，这些存储介质一般集成在一个独立的外部模块中（卡、WIFI模组等），通过物理总线和CPU连接。对任何有线的通信协议来说，总线规范都是非常重要的。关于MMC总线规范，简单总结如下：

1）物理信号有CLK、CMD和DATA三类。

2）电压范围为1.65V和3.6V（参考上面图片2），根据工作电压的不同，MMC卡可以分为两类：

> High Voltage MultiMediaCard，工作电压为2.7V~3.6V。
>
> Dual Voltage MultiMediaCard，工作电压有两种，1.70V~1.95V和2.7V~3.6V，CPU可以根据需要切换。

3）数据传输的位宽（称作data bus width mode）是允许动态配置的，包括1-bit (默认)模式、4-bit模式和8-bit模式。

> 注1：不使用的数据线，需要保持上拉状态，这就是图片2中的DATA中标出上拉的原因。另外，由于数据线宽度是动态可配的，这要求CPU可以动态的enable/disable数据线的那些上拉电阻。

4）MMC规范定义了CLK的频率范围，包括0-20MHz、0-26MHz、0-52MHz等几种，结合数据线宽度，基本决定了MMC的访问速度。

5）总线规范定义了一种简单的、主从式的总线协议，MMC卡位从机（slave），CPU为主机（Host）。

6）协议规定了三种可以在总线上传输的信标（token）：

> Command，Host通过CMD线发送给Slave的，用于启动（或结束）一个操作（后面介绍）；
>
> Response，Slave通过CMD线发送给Host，用于回应Host发送的Command；
>
> Data，Host和Slave之间通过数据线传说的数据。方向可以是Host到Slave，也可以是Slave到Host。数据线的个数可以是1、4或者8。在每一个时钟周期，每根数据线上可以传输1bit或者2bits的数据。

7）一次数据传输过程，需要涉及所有的3个信标。一次数据传输的过程也称作Bus Operation，根据场景的不同，MMC协议规定了很多类型的Bus Operation（具体可参考相应的规范）。

#### 3.3 控制器规范

Host控制器是MMC总线规范在Host端的实现，也是Linux驱动工程师比较关注的地方，后面将会结合Linux MMC framework的有关内容，再详细介绍。

## 4. 总结

本文对MMC/SD/SDIO等做了一个简单的介绍，有了这些基本概念之后，在Linux kernel中编写MMC驱动将不再是一个困难的事情（因为MMC是一个协议，所有有关协议的事情，都很简单，因为协议是固定的），我们只需要如下步骤即可完成：

> 1）结合MMC的规范，阅读Host MMC controller的spec，理解有关的功能和操作方法。
>
> 2）根据Linux MMC framework的框架，将MMC bus有关的操作方法通过MMC controller实现。

具体可参考后续MMC framework的分析文档。

## 5. 参考文档

\[1\] [https://en.wikipedia.org/wiki/MultiMediaCard](https://en.wikipedia.org/wiki/MultiMediaCard "https://en.wikipedia.org/wiki/MultiMediaCard")

\[2\] [https://en.wikipedia.org/wiki/Secure_Digital](https://en.wikipedia.org/wiki/Secure_Digital "https://en.wikipedia.org/wiki/Secure_Digital")

\[3\] eMM spec（注册后可免费下载），[http://www.jedec.org/standards-documents/results/jesd84-b51](http://www.jedec.org/standards-documents/results/jesd84-b51 "http://www.jedec.org/standards-documents/results/jesd84-b51")

原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/basic_tech/mmc_sd_sdio_intro.html)。

标签: [emmc](http://www.wowotech.net/tag/emmc) [mmc](http://www.wowotech.net/tag/mmc) [sd](http://www.wowotech.net/tag/sd) [sdio](http://www.wowotech.net/tag/sdio)

______________________________________________________________________

« [X-022-OTHERS-git操作记录之合并远端分支的更新](http://www.wowotech.net/x_project/u_boot_merge_denx.html) | [基于Hikey的"Boot from USB"调试](http://www.wowotech.net/x_project/hikey_usb_boot.html)»

**评论：**

**111**\
2020-08-18 21:41

https://linux.codingbelief.com/zh/\
里面有flash和内存的帖子（和这个网站上的帖子一样）

[回复](http://www.wowotech.net/basic_tech/mmc_sd_sdio_intro.html#comment-8099)

**eric**\
2020-04-26 15:52

MMC卡位从机,MMC卡为从机

[回复](http://www.wowotech.net/basic_tech/mmc_sd_sdio_intro.html#comment-7970)

**[jinxinxin163](http://www.wowotech.net/)**\
2017-01-19 08:52

请教：\
https://en.wikipedia.org/wiki/MultiMediaCard\
里的Comparison of technical features of MMC and SD card variants\
为什么没有emmc？

[回复](http://www.wowotech.net/basic_tech/mmc_sd_sdio_intro.html#comment-5165)

**[wowo](http://www.wowotech.net/)**\
2017-01-20 08:35

@jinxinxin163：emmc和mmc的本质区别，就是封装的不同（以IP的形式嵌入到CPU中），其它的都差不多，所以wikipedia没有去更新吧（我猜测的）。

[回复](http://www.wowotech.net/basic_tech/mmc_sd_sdio_intro.html#comment-5169)

**maze**\
2018-01-16 13:59

@jinxinxin163：已经有了哦.现在

[回复](http://www.wowotech.net/basic_tech/mmc_sd_sdio_intro.html#comment-6480)

**King Jin**\
2016-12-28 16:09

大神写的真好。

[回复](http://www.wowotech.net/basic_tech/mmc_sd_sdio_intro.html#comment-5068)

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

  - [Linux时间子系统之（十二）：periodic tick](http://www.wowotech.net/timer_subsystem/periodic-tick.html)
  - [X-019-KERNEL-串口驱动开发之数据收发](http://www.wowotech.net/x_project/serial_driver_porting_4.html)
  - [串口通信技术浅析](http://www.wowotech.net/basic_tech/serial_intro.html)
  - [以太网驱动的流程浅析(四)-以太网驱动probe流程](http://www.wowotech.net/linux_kenrel/469.html)
  - [中断上下文中调度会怎样？](http://www.wowotech.net/process_management/schedule-in-interrupt.html)

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

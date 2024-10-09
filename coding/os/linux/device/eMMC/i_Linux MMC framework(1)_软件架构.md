作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2017-1-10 22:24 分类：[通信类协议](http://www.wowotech.net/sort/comm)

## 1. 前言

由\[1\]中MMC、SD、SDIO的介绍可知，这三种技术都是起源于MMC技术，有很多共性，因此Linux kernel统一使用MMC framework管理所有和这三种技术有关的设备。

本文将基于\[1\]对MMC技术的介绍，学习Linux kernel MMC framework的软件架构。

## 2. 软件架构

Linux kernel的驱动框架有两个要点（尽管本站前面的文章已经多次强调，本文还是要再说明一下，因为这样的设计思想，说一千遍都不会烦）：

> 1）抽象硬件（硬件架构是什么样子，驱动框架就应该是什么样子）。
>
> 2）向“客户”提供使用该硬件的API（之前我们提到最多的客户是“用户空间的Application”，不过也有其它“客户”，例如内核空间的其它driver、其它framework）。

以本文的描述对象为例，MMC framework的软件架构如下面“图片1”所示：

[![mmc_architecture](http://www.wowotech.net/content/uploadfile/201701/50b2a0c2bbf76a81a7a5fd9549b7daae20170110142443.gif "mmc_architecture")](http://www.wowotech.net/content/uploadfile/201701/d07215f202ec5295dd60b88a86e2660020170110142442.gif)

图片1 Linux MMC framework软件架构

MMC framework分别有“从左到右”和“从下到上”两种层次结构。

1） 从左到右

MMC协议是一个总线协议，因此包括Host controller、Bus、Card三类实体（从左到右）。相应的，MMC framework抽象出了host、bus、card三个软件实体，以便和硬件一一对应：

> host，负责驱动Host controller，提供诸如访问card的寄存器、检测card的插拔、读写card等操作方法。从设备模型的角度看，host会检测卡的插入，并向bus注册MMC card设备；
>
> bus，是MMC bus的虚拟抽象，以标准设备模型的方式，收纳MMC card（device）以及对应的MMC driver（driver）；
>
> card，抽象具体的MMC卡，由对应的MMC driver驱动（从这个角度看，可以忽略MMC的技术细节，只需关心一个个具有特定功能的卡设备，如存储卡、WIFI卡、GPS卡等等）。

2）从下到上

MMC framework从下到上也有3个层次（老生常谈了）：

> MMC core位于中间，是MMC framework的核心实现，负责抽象host、bus、card等软件实体，负责向底层提供统一、便利的编写Host controller driver的API；
>
> MMC host controller driver位于底层，基于MMC core提供的框架，驱动具体的硬件（MMC controller）；
>
> MMC card driver位于最上面，负责驱动MMC core抽象出来的虚拟的card设备，并对接内核其它的framework（例如块设备、TTY、wireless等），实现具体的功能。

## 3. 工作流程

基于图片1中的软件架构，Linux MMC framework的工作流程如下：

[![mmc_opt_flow](http://www.wowotech.net/content/uploadfile/201701/610a9ad18af8205f2c8d6e9c2d24c82d20170110142445.gif "mmc_opt_flow")](http://www.wowotech.net/content/uploadfile/201701/133ff2349cf343a5d07bccb6c266b5ca20170110142444.gif)

图片2 MMC操作流程

暂时不进行详细介绍，感兴趣的同学可以照着代码先看看。后续其它文章会逐一展开。

_原创文章，转发请注明出处。蜗窝科技_，[www.wowotech.net](http://www.wowotech.net/comm/mmc_framework_arch.html)。

标签: [Linux](http://www.wowotech.net/tag/Linux) [Kernel](http://www.wowotech.net/tag/Kernel) [内核](http://www.wowotech.net/tag/%E5%86%85%E6%A0%B8) [架构](http://www.wowotech.net/tag/%E6%9E%B6%E6%9E%84) [Architecture](http://www.wowotech.net/tag/Architecture) [framework](http://www.wowotech.net/tag/framework) [mmc](http://www.wowotech.net/tag/mmc)

______________________________________________________________________

« [eMMC 原理 2 ：eMMC 简介](http://www.wowotech.net/basic_tech/emmc_intro.html) | [eMMC 原理 1 ：Flash Memory 简介](http://www.wowotech.net/basic_tech/flash_memory_intro.html)»

**评论：**

**[董先生](http://dongni.work/)**\
2021-10-11 16:47

你好，不太理解这个“从下到上也有3个层次”

不太明白card_driver 与 host controller driver 这两个有什么区别？

请问大家，有没有一些相关文档，以供仔细阅读呢？

刚学习，有点迷糊，谢谢大家指教！

[回复](http://www.wowotech.net/comm/mmc_framework_arch.html#comment-8330)

**北海风云**\
2019-08-20 14:29

图挂了 。能补一下吗 ?

[回复](http://www.wowotech.net/comm/mmc_framework_arch.html#comment-7599)

**zoro**\
2017-03-25 23:55

请问框图是用什么软件画的啊

[回复](http://www.wowotech.net/comm/mmc_framework_arch.html#comment-5379)

**[wowo](http://www.wowotech.net/)**\
2017-03-27 08:49

@zoro：ppt

[回复](http://www.wowotech.net/comm/mmc_framework_arch.html#comment-5380)

**[江南书生](http://no/)**\
2017-05-04 17:30

@wowo：PPT 画的这么工整，牛逼

[回复](http://www.wowotech.net/comm/mmc_framework_arch.html#comment-5516)

**[wowo](http://www.wowotech.net/)**\
2017-05-05 08:23

@江南书生：哪里，过奖了。其实office2007之后，用ppt画图还是挺漂亮的，那些模板的配色，做到不错:-)

[回复](http://www.wowotech.net/comm/mmc_framework_arch.html#comment-5520)

**[江南书生](http://no/)**\
2017-03-01 14:38

赞

[回复](http://www.wowotech.net/comm/mmc_framework_arch.html#comment-5276)

**青春年少**\
2017-02-13 16:57

写的真好

[回复](http://www.wowotech.net/comm/mmc_framework_arch.html#comment-5218)

**[狂奔的蜗牛](http://www.wowotech.net/)**\
2017-01-25 10:37

清晰明了，赞

[回复](http://www.wowotech.net/comm/mmc_framework_arch.html#comment-5191)

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

  - [内存初始化代码分析（一）：identity mapping和kernel image mapping](http://www.wowotech.net/memory_management/__create_page_tables_code_analysis.html)
  - [Common Clock Framework系统结构](http://www.wowotech.net/pm_subsystem/ccf-arch.html)
  - [process credentials相关的用户空间文件](http://www.wowotech.net/linux_application/24.html)
  - [Linux时间子系统之（十六）：clockevent](http://www.wowotech.net/timer_subsystem/clock-event.html)
  - [蓝牙协议分析(11)\_BLE安全机制之SM](http://www.wowotech.net/bluetooth/le_security_manager.html)

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

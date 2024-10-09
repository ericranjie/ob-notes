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

作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2014-12-9 22:54 分类：[ARMv8A Arch](http://www.wowotech.net/sort/armv8a_arch)

#### 1. 前言

本文简单梳理一下ARM有关的概念，包括ARM architecture、ARM core、ARM CPU（或MCU）以及ARM Soc。我们这些以ARM平台为主的嵌入式工程师，几乎每天都会和这些概念打交道，也似乎非常理解它们。但仔细想想，却有些说不清道不明的感觉，因而有必要整理一下思路，也就顺手记录下来了。

#### 2. 概念梳理

1）ARM architecture

ARM architecture，是指ARM公司开发的、基于精简指令集架构（RISC, Reduced Instruction Set Computing architecture）的指令集架构（Instruction set architecture）。我们常说的ARMv7、ARMv8、ARMv8-A，就是指ARM architecture。类似的基于RISC的architecture也有很多，例如MIPS、AVR、Blackfin等等，都是这个概念。

2）ARM core

ARM core是基于ARM architecture开发出来的IP core，它是介于architecture和最终的CPU(MCU)之间的中间产品，这也是ARM商业模式的独特之处。

有两种类型的ARM core：一种是ARM公司自己发布的，如我们耳熟能详的ARM7、ARM9、ARM Cortex M3、ARM Cortex A57等等；另一种是ARM授权其它公司开发的Core，如苹果的A6/A6X等等。下面链接是维基百科上的ARM core的列表，共大家参考：

[http://en.wikipedia.org/wiki/List_of_ARM_microarchitectures](http://en.wikipedia.org/wiki/List_of_ARM_microarchitectures "http://en.wikipedia.org/wiki/List_of_ARM_microarchitectures")

3）ARM CPU（MCU）

其它的芯片厂商，如Phillips、ST、TI等，会基于ARM公司发布的Core，开发自己的ARM处理器，这称作ARM CPU（也可称为MCU）。这些是我们工作过程中接触最多的，如LPCxxxx、STM32xxx、OMAPxxxx、S3Cxxxx等等。

4）ARM Soc

对于一些比较专业的应用场景，如视频、音频等，为了追求更小的size、更低的功耗，厂商会在芯片上，集成除处理器之外的东西，如视频编解码器、DSP等。这些集成了其它功能的芯片，称作片上系统（SOC），如TI的DM37x Video SOC。

注1：其实ARM的技术和商业模式，正体现了软件工程中抽象和封装的思想。

#### 3. ARM 64bit

我们以一款64bit ARM CPU为例，反向阐述一下ARM处理的诞生过程，同时罗列一些学习、研究方向。

1）我们熟悉一个CPU（假设它的型号是WW9000）的第一手资料，是芯片厂家发布的Datasheet，例如WW9000_SPEC.pdf。

2）WW9000是基于ARM Cortex-A57 Core封装而来的，该ARM core的资料可以从下面链接下载

[http://infocenter.arm.com/help/topic/com.arm.doc.ddi0488g/DDI0488G_cortex_a57_mpcore_trm.pdf](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0488g/DDI0488G_cortex_a57_mpcore_trm.pdf "PDF version")

3）ARM Cortex-A57 Core又是基于ARMv8-A architecture，该结构的资料可以通过如下方式获取：

Go to [ARM Infocenter](http://infocenter.arm.com/) and navigate through _ARM architecture_ / _Reference Manuals_

注2：[ARM Infocenter](http://infocenter.arm.com/)中资料是非常全面的，没事时可以多逛逛。

_原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/armv8a_arch/arm_concept.html)。_

标签: [Architecture](http://www.wowotech.net/tag/Architecture) [ARM](http://www.wowotech.net/tag/ARM) [core](http://www.wowotech.net/tag/core) [soc](http://www.wowotech.net/tag/soc)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [ARM WFI和WFE指令](http://www.wowotech.net/armv8a_arch/wfe_wfi.html) | [Linux时间子系统之（十六）：clockevent](http://www.wowotech.net/timer_subsystem/clock-event.html)»

**评论：**

**Tech_fan**\
2015-12-10 07:26

ARM architecture，是指ARM公司开发的、基于精简指令集架构（RISC, Reduced Instruction Set Computing architecture）的指令集架构（Instruction set architecture）

其实ARM构架不仅仅包括指令集，还包括异常模式，寄存器集，内存管理等

[回复](http://www.wowotech.net/armv8a_arch/arm_concept.html#comment-3211)

**[wowo](http://www.wowotech.net/)**\
2015-12-10 09:11

@Tech_fan：确实如此，多谢指正~~

[回复](http://www.wowotech.net/armv8a_arch/arm_concept.html#comment-3216)

**[superm](http://www.wowotech.net/)**\
2015-10-30 11:08

高通会修改ARM IP的内部RTL代码吗？

[回复](http://www.wowotech.net/armv8a_arch/arm_concept.html#comment-2897)

**[wowo](http://www.wowotech.net/)**\
2015-10-30 13:50

@superm：抱歉，我也不是很懂RTL，可以的话，您可以给大家普及一下这个概念。

[回复](http://www.wowotech.net/armv8a_arch/arm_concept.html#comment-2898)

**[superm](http://www.wowotech.net/)**\
2015-10-30 17:19

@wowo：其实我就是觉得高通可能会自己修改了ARM的IP，优化和个性化IP，就好像你说的高通买的是IP core的授权，而不是ARM core的授权，这个是不是说明高通自己在修改ARM IP?\
二国内大部分都是直接集成了。

[回复](http://www.wowotech.net/armv8a_arch/arm_concept.html#comment-2900)

**[wowo](http://www.wowotech.net/)**\
2015-10-30 21:33

@superm：高通和苹果应该会自己设计、修改ARM IP的，不然比其它厂家就没有优势了。

[回复](http://www.wowotech.net/armv8a_arch/arm_concept.html#comment-2907)

**[tigger](http://www.wowotech.net/)**\
2014-12-10 13:05

所以高通的芯片是高通拿到ARM的IP core lincense然后自己做出来的一个ARM Core，指令集跟拿到的那IP core的指令集兼容。但是有自己的独特的东东\
其他的公司比如国内的芯片公司，都是拿的完整的ARM core，然后自己封装出来一个ARM CPU.所谓的CPU就是ARM CORE+其他总线啊，外设啊。等等。

[回复](http://www.wowotech.net/armv8a_arch/arm_concept.html#comment-872)

**[tigger](http://www.wowotech.net/)**\
2014-12-10 13:06

@tigger：当然对于高通比如8916,光有ARM core还不行，还是要有外设组成一个CPU.这个8916应该是个ARM CPU.

[回复](http://www.wowotech.net/armv8a_arch/arm_concept.html#comment-873)

**[tigger](http://www.wowotech.net/)**\
2014-12-10 13:19

@tigger：这里cpu 跟soc的区别我有点搞不懂了。。。

[回复](http://www.wowotech.net/armv8a_arch/arm_concept.html#comment-874)

**wowo**\
2014-12-10 14:05

@tigger：是的，你看维基上给出来的那个表格，高通比较有技术含量，买的是IP core的授权，而不是ARM core的授权。\
对于CPU和SOC，其实界限不是特别明显，这取决于厂商怎么定义。如果严格定义，CPU应该只负责取指、执行、存储、读取。从这个角度看，主流CPU都是SOC，因为里面包了很多之外的东西，但厂商却不那么叫。\
另外，有一些很明显的chip，它可以包含1个ARM core和1个DSP，这一定是SOC了。

[回复](http://www.wowotech.net/armv8a_arch/arm_concept.html#comment-875)

**[tigger](http://www.wowotech.net/)**\
2014-12-10 16:36

@wowo：领教了。

[回复](http://www.wowotech.net/armv8a_arch/arm_concept.html#comment-877)

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

  - [copy\_{to,from}\_user()的思考](http://www.wowotech.net/memory_management/454.html)
  - [内存初始化（上）](http://www.wowotech.net/memory_management/mm-init-1.html)
  - [Process Creation（一）](http://www.wowotech.net/process_management/Process-Creation-1.html)
  - [CFS任务的负载均衡（load balance）](http://www.wowotech.net/process_management/load_balance_detail.html)
  - [Linux电源管理(10)\_autosleep](http://www.wowotech.net/pm_subsystem/autosleep.html)

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

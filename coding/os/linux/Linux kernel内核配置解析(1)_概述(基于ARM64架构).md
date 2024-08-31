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

作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2016-8-10 23:20 分类：[Linux内核分析](http://www.wowotech.net/sort/linux_kenrel)

#### 1. 前言

对刚接触Linux kernel的同学来说，遇到的第一个问题就是：我该从哪里入手？、

话说Linux kernel的打开方式是多种多样的：从简单的设备驱动入手；从源代码的目录结构入手；从kernel的启动过程入手；从大的功能模块入手；等等。不管怎样，每条都是正途（条条大路通罗马嘛）。

而本文（以及随后的系列文章），将从Linux kernel的配置项入手，从整体上认识Linux kernel。之所以这么做，原因有二：

> 1）Linux kernel的配置项数目繁多，以至于进行kernel移植的时候，看到menuconfig界面后，会有深深的恐惧感（可参考下面图片1）。
> 
> 2）配置项的目的，是功能配置和功能开关，从一定程度上可以看出一个软件的功能模块划分。以Linux kernel为例，Kconfig所呈现出来的树状结构，从功能划分的角度看，比source code的目录结构还清晰。

注1：本系列文章使用的Linux kernel版本是“[X Project](http://www.wowotech.net/forum/viewforum.php?id=6)”所用的“[Linux 4.6-rc5](https://github.com/wowotechX/linux/commit/02da2d72174c61988eb4456b53f405e3ebdebce4)”，具体可参考“[https://github.com/wowotechX/linux.git](https://github.com/wowotechX/linux.git "https://github.com/wowotechX/linux.git")”。

#### 2. Kernel配置项初识

Linux kernel的配置项，是以架构（ARCH）为单位，通过Kconfig语言组织在一起的。以ARM64为例，其Kconfig的入口位于：

> arch/arm64/Kconfig

在Kernel根目录下以“ARCH=arm64”为参数，执行make menuconfig，可以得到如下的配置界面：

> make ARCH=arm64 menuconfig

[![kernel_menuconfig](http://www.wowotech.net/content/uploadfile/201608/09aa28c7104ce75c5d80ce602344219b20160810152007.gif "kernel_menuconfig")](http://www.wowotech.net/content/uploadfile/201608/a731b44c081c641cfd21bdba02a61f6220160810152005.gif)

图片1 Kernel_menuconfig

第一个画面，还可以接受，毕竟画风清爽。但点进到二级菜单，脑袋就大了。不过不着急，我们一层一层的分析。

开始之前，先交代一下分析的手段，很简单，要点有四：

> 结合Kconfig文件；
> 
> 跟随menuconfig的菜单项；
> 
> 加上强大的Google；
> 
> 必要时阅读source code。

另外，鉴于篇幅问题，本文只介绍Kconfig的一级菜单（就是图片1所能看到的部分），相当于一个索引，后续文章会一个一个展开描述。

#### 3. 一级菜单

本章我们将根据arch/arm64/Kconfig文件，对menuconfig的一级菜单进行简要的分析，目的是从实际的例子出发，理解Kconfig语言的语法，一级Linux kernel配置项的整体结构。具体请参考如下表格：

|   |   |   |
|---|---|---|
|配**置项**|**Kconfig文件位置**|**功能说明**|
|ARM64架构的默认配置项|[arch/arm64/Kconfig](https://github.com/wowotechX/linux/blob/x_integration/arch/arm64/Kconfig)|指定ARCH为ARM64之后，ARM64的Kconfig会默认帮我们确定众多的配置项，例如CONFIG_64BIT、CONFIG_MMU、CONFIG_OF等等。这些配置项不会体现在menuconfig的菜单中，但可以在最终生成的config文件中看到。|
|General setup|[init/Kconfig](https://github.com/wowotechX/linux/blob/x_integration/init/Kconfig)  <br>位于menu "General setup"和对应的endmenu之间|该配置项由“menu … endmenu”定义，是一个配置菜单，表示一类配置的集合（参考上面图片1，“--->”结尾的配置项都是菜单项，按Enter直接进入对应的菜单界面）；  <br>主要用于配置和功能无关的的通用选项，例如kernel的版本号、压缩方式、等等。|
|loadable module|[init/Kconfig](https://github.com/wowotechX/linux/blob/x_integration/init/Kconfig)  <br>由“menuconfig MODULES”定义|menuconfig和menu不同，是一个可以选择是否开启的菜单（参考图片1中的“[*]”）；  <br>用于配置内核“模块”有关的特性。|
|block device|[block/Kconfig](https://github.com/wowotechX/linux/blob/x_integration/block/Kconfig)  <br>由“menuconfig BLOCK”定义|内核块设备有关的特性。|
|Platform selection|[arch/arm64/Kconfig.platforms](https://github.com/wowotechX/linux/blob/x_integration/arch/arm64/Kconfig.platforms)  <br>位于menu "Platform selection"和endmenu之间|用于配置和具体平台有关的配置项，如SUNIX、HISI等；  <br>自从ARM64把“mach-xxx”目录抛弃之后，这里可能是各个平台可自行发挥的最后一个空间了。|
|PCI Bus support|[arch/arm64/Kconfig](https://github.com/wowotechX/linux/blob/x_integration/arch/arm64/Kconfig)  <br>    drivers/pci/Kconfi|PCI总线有关的特性。|
|ACPI support|drivers/acpi/Kconfig|ACPI总线有关的特性。|
|Kernel Features|[arch/arm64/Kconfig](https://github.com/wowotechX/linux/blob/x_integration/arch/arm64/Kconfig)  <br>    kernel/Kconfig.preempt  <br>    kernel/Kconfig.hz  <br>    mm/Kconfig  <br>位于menu "Kernel Features"和对应的endmenu之间|Linux kernel的核心功能的配置，如进程管理、内存管理、等等。是Linux kernel配置项中最复杂的一类。|
|Boot options|[arch/arm64/Kconfig](https://github.com/wowotechX/linux/blob/x_integration/arch/arm64/Kconfig)  <br>位于menu "Boot options"和对应的endmenu之间|用于配置和内核启动有关的功能，如默认的Command line、UEFI支持等。|
|Userspace binary formats|[arch/arm64/Kconfig](https://github.com/wowotechX/linux/blob/x_integration/arch/arm64/Kconfig)  <br>    fs/Kconfig.binfmt  <br>位于menu "Userspace binary formats"和对应的endmenu之间|用于配置用户空间二进制的格式。|
|Power management|[arch/arm64/Kconfig](https://github.com/wowotechX/linux/blob/x_integration/arch/arm64/Kconfig)  <br>    kernel/power/Kconfig  <br>位于menu "Power management options"和对应的endmenu之间|Linux kernel电源管理有关的特性。|
|CPU Power Management|[arch/arm64/Kconfig](https://github.com/wowotechX/linux/blob/x_integration/arch/arm64/Kconfig)  <br>    drivers/cpuidle/Kconfig  <br>    drivers/cpufreq/Kconfig  <br>位于menu "CPU Power Management"和对应的endmenu之间|CPU有关的电源管理特性，如cpuidle、cpufreq等；  <br>这是新版kernel的一大改进，将CPU有关的电源管理功能，抽象成一个顶层功能，和系统的电源管理并列。|
|Networking support|[net/Kconfig](https://github.com/wowotechX/linux/blob/x_integration/net/Kconfig)|网络有关的特性。|
|Device Drivers|[drivers/Kconfig](https://github.com/wowotechX/linux/blob/x_integration/drivers/Kconfig)|设备驱动有关的配置项。|
|Firmware Drivers|[drivers/firmware/Kconfig](https://github.com/wowotechX/linux/blob/x_integration/drivers/firmware/Kconfig)   <br>    …|Firmware有关的配置项。|
|File systems|[fs/Kconfig](https://github.com/wowotechX/linux/blob/x_integration/fs/Kconfig)|文件系统有关的配置项。|
|Virtualization|[arch/arm64/kvm/Kconfig](https://github.com/wowotechX/linux/blob/x_integration/arch/arm64/kvm/Kconfig)|虚拟化有关的配置项。|
|Kernel hacking|arch/arm64/Kconfig.debug|Kernel调试有关的配置项。|
|Security options|[security/Kconfig](https://github.com/wowotechX/linux/blob/x_integration/security/Kconfig)|安全特性有关的配置项。|
|Cryptographic API|[crypto/Kconfig](https://github.com/wowotechX/linux/blob/x_integration/crypto/Kconfig)  <br>[arch/arm64/crypto/Kconfig](https://github.com/wowotechX/linux/blob/x_integration/arch/arm64/crypto/Kconfig)|加密算法有关的配置项。|
|Library routines|[lib/Kconfig](https://github.com/wowotechX/linux/blob/x_integration/lib/Kconfig)|用于配置常用的library，如CRC16等。|

#### 4. 总结

本文只是一个引子，以ARM64平台为例，大概介绍了Linux内核配置的基本结构。后续将会以一个个专题为单位，展开介绍。

鉴于内核配置涉及到了Linux kernel的所有模块，因此该系列文档将是一个大工程，后续所有的文件将包括（此处先留位占空，后面会慢慢补充完善：

> Linux kernel内核配置解析(2)_ARM64架构特有的配置项
> 
> Linux kernel内核配置解析(3)_General setup
> 
> Linux kernel内核配置解析(4)_Kernel Features
> 
> Linux kernel内核配置解析(5)_Boot options
> 
> Linux kernel内核配置解析(6)_Kernel hacking
> 
> （等等）
> 
> Linux kernel内核配置解析(n)_配置项汇整。

_  
原创文章，转发请注明出处。蜗窝科技_，[www.wowotech.net](http://www.wowotech.net/sort/linux_kenrel/kernel_config_overview.html)。

标签: [Linux](http://www.wowotech.net/tag/Linux) [Kernel](http://www.wowotech.net/tag/Kernel) [内核](http://www.wowotech.net/tag/%E5%86%85%E6%A0%B8) [配置项](http://www.wowotech.net/tag/%E9%85%8D%E7%BD%AE%E9%A1%B9) [menuconfig](http://www.wowotech.net/tag/menuconfig)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [ARM64架构下地址翻译相关的宏定义](http://www.wowotech.net/memory_management/arm64-memory-addressing.html) | [DRAM 原理 3 ：DRAM Device](http://www.wowotech.net/basic_tech/321.html)»

**评论：**

**楚慕**  
2020-09-16 19:57

四年了，大佬还写吗？

[回复](http://www.wowotech.net/linux_kenrel/kernel_config_overview.html#comment-8114)

**123@**  
2016-08-22 13:56

赞

[回复](http://www.wowotech.net/linux_kenrel/kernel_config_overview.html#comment-4433)

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
    
    - Shiina  
        [一个电路（circuit）中，由于是回路，所以用电势差的概念...](http://www.wowotech.net/basic_subject/voltage.html#8926)
    - Shiina  
        [其中比较关键的点是相对位置概念和点电荷的静电势能计算。](http://www.wowotech.net/basic_subject/voltage.html#8925)
    - leelockhey  
        [你这是哪个内核版本](http://www.wowotech.net/pm_subsystem/generic_pm_architecture.html#8924)
    - ja  
        [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
    - 元神高手  
        [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
    - 十七  
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
    
    - [Linux调度器：用户空间接口](http://www.wowotech.net/process_management/scheduler-API.html)
    - [玩转BLE(2)_使用bluepy扫描BLE的广播数据](http://www.wowotech.net/bluetooth/bluepy_scan.html)
    - [Linux电源管理(9)_wakelocks](http://www.wowotech.net/pm_subsystem/wakelocks.html)
    - [Linux时间子系统系列文章之目录](http://www.wowotech.net/timer_subsystem/time_subsystem_index.html)
    - [Linux内核同步机制之（八）：mutex](http://www.wowotech.net/kernel_synchronization/504.html)
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
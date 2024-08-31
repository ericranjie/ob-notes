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

作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2016-8-12 22:46 分类：[Linux内核分析](http://www.wowotech.net/sort/linux_kenrel)

#### 1. 前言

本文将介绍ARM64架构下，Linux kernel和启动有关的配置项。

注1：本系列文章使用的Linux kernel版本是“[X Project](http://www.wowotech.net/forum/viewforum.php?id=6)”所用的“[Linux 4.6-rc5](https://github.com/wowotechX/linux/commit/02da2d72174c61988eb4456b53f405e3ebdebce4)”，具体可参考“[https://github.com/wowotechX/linux.git](https://github.com/wowotechX/linux.git)”。

#### 2. Kconfig文件

ARM64架构中和Boot有关的配置项，非常简单，主要包括ACPI、命令行参数和UEFI几种。这些配置项位于“ [arch/arm64/Kconfig](https://github.com/wowotechX/linux/blob/x_integration/arch/arm64/Kconfig)”中，具体如下：

  1: menu "Boot options"

  2: 

  3: config ARM64_ACPI_PARKING_PROTOCOL

  4: 	bool "Enable support for the ARM64 ACPI parking protocol"

  5: 	depends on ACPI

  6: 	help

  7: 	  Enable support for the ARM64 ACPI parking protocol. If disabled

  8: 	  the kernel will not allow booting through the ARM64 ACPI parking

  9: 	  protocol even if the corresponding data is present in the ACPI

 10: 	  MADT table.

 11: 

 12: config CMDLINE

 13: 	string "Default kernel command string"

 14: 	default ""

 15: 	help

 16: 	  Provide a set of default command-line options at build time by

 17: 	  entering them here. As a minimum, you should specify the the

 18: 	  root device (e.g. root=/dev/nfs).

 19: 

 20: config CMDLINE_FORCE

 21: 	bool "Always use the default kernel command string"

 22: 	help

 23: 	  Always use the default kernel command string, even if the boot

 24: 	  loader passes other arguments to the kernel.

 25: 	  This is useful if you cannot or don't want to change the

 26: 	  command-line options your boot loader passes to the kernel.

 27: 

 28: config EFI_STUB

 29: 	bool

 30: 

 31: config EFI

 32: 	bool "UEFI runtime support"

 33: 	depends on OF && !CPU_BIG_ENDIAN

 34: 	select LIBFDT

 35: 	select UCS2_STRING

 36: 	select EFI_PARAMS_FROM_FDT

 37: 	select EFI_RUNTIME_WRAPPERS

 38: 	select EFI_STUB

 39: 	select EFI_ARMSTUB

 40: 	default y

 41: 	help

 42: 	  This option provides support for runtime services provided

 43: 	  by UEFI firmware (such as non-volatile variables, realtime

 44:           clock, and platform reset). A UEFI stub is also provided to

 45: 	  allow the kernel to be booted as an EFI application. This

 46: 	  is only useful on systems that have UEFI firmware.

 47: 

 48: config DMI

 49: 	bool "Enable support for SMBIOS (DMI) tables"

 50: 	depends on EFI

 51: 	default y

 52: 	help

 53: 	  This enables SMBIOS/DMI feature for systems.

 54: 

 55: 	  This option is only useful on systems that have UEFI firmware.

 56: 	  However, even with this option, the resultant kernel should

 57: 	  continue to boot on existing non-UEFI platforms.

 58: 

 59: endmenu

#### 3. 配置项说明

注2：Linux kernel的配置项虽然众多，但大多使用默认值就可以。因此在kernel移植和开发的过程中，真正需要关心的并不是特别多。对于那些常用的、需要关心的配置项，我会在分析文章中用黄色背景标注。

##### 3.1 ACPI有关的配置项

|   |   |   |
|---|---|---|
|配置项|说明|默认值|
|CONFIG_ARM64_ACPI_  <br>PARKING_PROTOCOL|是否支持“ARM64 ACPI parking protocol”。关于ACPI和parking protocol，有机会的话我们会在其它文章中分析，这里不需要过多关注。|依赖于CONFIG_ACPI|

##### 3.2 Kernel命令行参数有关的配置项

|   |   |   |
|---|---|---|
|配置项|说明|默认值|
|CONFIG_CMDLINE|内核默认的命令行参数。设置该参数后，可以不需要bootloader传递（开始porting kernel的时候比较有用，因为不能保证bootloader可以正确传递^_^）|无|
|CONFIG_CMDLINE_FORCE|强制使用内核默认的命令行参数（可以忽略bootloader传递来的）；  <br>一般在kernel开发的过程中，用来测试某些新的命令行参数（先不修修改bootloader传递的内容）。|无|

注3：如果Kconfig没有通过“default”关键字为某个配置项指定默认值，那么生成的.config文件中就不会出现该配置项，也就是变相的“禁止”了。后同。

##### 3.3 UEFI有关的配置项

DMI

|   |   |   |
|---|---|---|
|配置项|说明|默认值|
|CONFIG_EFI_STUB|用于支持EFI启动；  <br>使能该配置项之后，会修改Kenrel bzImage header，把kernel Image变成一个可以被EFI运行的PE/COFF Image。  <br>具体可参考Documentation/efi-stub.txt中的介绍。|无|
|CONFIG_EFI|支持一些由UEFI Firmware提供的、运行时的服务，如RTC、reset等；  <br>该配置项依赖Open Firmware（device tree），并且有很多的关联项（可以参考Kconfig文件中的select关键字）；  <br>另外，有意思的是（参考第2章Kconfig文件中的“depends on OF && !CPU_BIG_ENDIAN”），该配置项只能在小端CPU中才能使用。有空可以研究一下为什么。|y|
|CONFIG_DMI|用于控制是否支持“SMBIOS/DMI feature”，依赖于CONFIG_EFI；  <br>需要注意的是，就算使能了该配置项，kernel也需要能够在其它非UEFI环境下正常启动。|y|

#### 4. 参考文档

[1] UEFI,[http://www.wowotech.net/armv8a_arch/UEFI.html](http://www.wowotech.net/armv8a_arch/UEFI.html "http://www.wowotech.net/armv8a_arch/UEFI.html")

[2] ACPI,[https://zh.wikipedia.org/zh-cn/%E9%AB%98%E7%BA%A7%E9%85%8D%E7%BD%AE%E4%B8%8E%E7%94%B5%E6%BA%90%E6%8E%A5%E5%8F%A3](https://zh.wikipedia.org/zh-cn/%E9%AB%98%E7%BA%A7%E9%85%8D%E7%BD%AE%E4%B8%8E%E7%94%B5%E6%BA%90%E6%8E%A5%E5%8F%A3 "https://zh.wikipedia.org/zh-cn/%E9%AB%98%E7%BA%A7%E9%85%8D%E7%BD%AE%E4%B8%8E%E7%94%B5%E6%BA%90%E6%8E%A5%E5%8F%A3")

[3] SMBIOS/DMI,[http://www.dmtf.org/cn/standards/smbios](http://www.dmtf.org/cn/standards/smbios "http://www.dmtf.org/cn/standards/smbios")

_原创文章，转发请注明出处。蜗窝科技_，[www.wowotech.net](http://www.wowotech.net/sort/linux_kenrel/kernel_config_boot_option.html)。

标签: [Linux](http://www.wowotech.net/tag/Linux) [内核](http://www.wowotech.net/tag/%E5%86%85%E6%A0%B8) [boot](http://www.wowotech.net/tag/boot) [配置项](http://www.wowotech.net/tag/%E9%85%8D%E7%BD%AE%E9%A1%B9) [menuconfig](http://www.wowotech.net/tag/menuconfig)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [X-009-KERNEL-Linux kernel的移植(Bubblegum-96平台)](http://www.wowotech.net/x_project/bubblegum_kernel_porting.html) | [ARM64架构下地址翻译相关的宏定义](http://www.wowotech.net/memory_management/arm64-memory-addressing.html)»

**评论：**

**amalgam-silver**  
2016-08-16 14:17

标题是不是手滑把2打成5了? :)

[回复](http://www.wowotech.net/linux_kenrel/kernel_config_boot_option.html#comment-4410)

**[wowo](http://www.wowotech.net/)**  
2016-08-16 18:18

@amalgam-silver：没有手滑，参考“http://www.wowotech.net/linux_kenrel/kernel_config_overview.html”中的计划，这个确实是第5篇，之所以先写，因为它比较简单:-)

[回复](http://www.wowotech.net/linux_kenrel/kernel_config_boot_option.html#comment-4411)

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
    
    - [Linux2.6.23 ：sleepable RCU的实现](http://www.wowotech.net/kernel_synchronization/linux2-6-23-RCU.html)
    - [中断上下文中调度会怎样？](http://www.wowotech.net/process_management/schedule-in-interrupt.html)
    - [X-000-PRE-开发环境搭建](http://www.wowotech.net/x_project/develop_env.html)
    - [Linux serial framework(1)_概述](http://www.wowotech.net/comm/serial_overview.html)
    - [Linux CPU core的电源管理(5)_cpu control及cpu hotplug](http://www.wowotech.net/pm_subsystem/cpu_hotplug.html)
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
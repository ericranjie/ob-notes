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

作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2017-7-23 22:21 分类：[X Project](http://www.wowotech.net/sort/x_project)

## 1. 前言

话说我们“[X Project](http://www.wowotech.net/sort/x_project)”的第一个任务就是通过USB将主机上的Image文件下载到开发板的Ram中执行（参考[1]中有关的内容），为此我们在host中porting了一个简单的应用程序（称作DFU[2]），负责和开发板ROM中的代码交流，下载并执行Image文件。为了方便，该应用程序使用libusb[3]进行USB有关的操作。

libusb不止使用起来简单，还有一个极大的优点，就是“跨平台”的特性。我们之前的例子[4]都是在Linux平台下操作的，最近由于win10内置了Ubuntu，Linux平台有关的开发工作，基本上都可以在这里完成了，因此就不需要费时、费神地切换到纯Linux环境下工作了。

不过呢，Win10的Ubuntu好是好，但没法像纯Linux系统那样支持USB设备，DFU有关的工作就无法在这里正常工作，因此就发挥libusb的特性，把“[X Project](http://www.wowotech.net/sort/x_project)” DFU[2]有关的代码在Windows下跑起来，也算感受一下“跨平台”的魅力。具体步骤如下。

## 2. 步骤

1）安装MinGW[6]

参考[5]中的介绍，libusb在Windows下可以在MinGW环境下编译、使用，因此我们可以按照[7]中的步骤，在Windows中下载并安装MinGW。

2） 编译libusb、dfu，并运行dfu（具体可参考[1]中有关的文章）

发现没有USB的驱动程序（见下面的log）：

> cd /d/work/xprj/build && make libusb && make dfu  
>   
> cd /d/work/xprj/tools/dfu && ./dfu.exe bubblegum 0  
> board_bubblegum_init  
> board bubblegum  
> address 0x0  
> filename (null)  
> need_run 0  
> bubblegum_init  
> b96_init_usb  
> b96_init_device  
> libusb: info [windows_get_device_list] The following device has no driver: '\\.\USB#VID_10D6&PID_10D6#5&A3E6D0F&0&3'  
> libusb: info [windows_get_device_list] libusb will not be able to access it.  
> Error: cannot open device 10d6:10d6  
> b96_init_device failed  
> board->init failed!

3）通过Zadig[8]安装USB的通用驱动

参考[5]中的说明，要在Windows访问USB设备，需要相应的驱动程序，例如WinUSB等，可以使用Zadig[8]工具辅助安装：

> ##### [https://github.com/libusb/libusb/wiki/Windows#How_to_use_libusb_on_Windows](https://github.com/libusb/libusb/wiki/Windows#How_to_use_libusb_on_Windows "https://github.com/libusb/libusb/wiki/Windows#How_to_use_libusb_on_Windows")  
>   
> Driver Installation
> 
> If your target device is not HID, you **must** install a driver before you can communicate with it using libusb. Currently, this means installing one of Microsoft's `WinUSB`, [libusb-win32](http://sourceforge.net/apps/trac/libusb-win32/wiki) or [libusbK](http://libusbk.sourceforge.net/UsbK3/index.html) drivers. Two options are available:
> 
> - **Recommended**: Use the most recent version of **[Zadig](http://zadig.akeo.ie/)**, an Automated Driver Installer GUI application for `WinUSB`, `libusb-win32` and `libusbK`...
> - Alternatively, if you are only interested in `WinUSB`, you can download the [WinUSB driver files](https://storage.googleapis.com/google-code-archive-downloads/v2/code.google.com/libusb-winusb-wip/winusb%20driver.zip) and customize the `inf` file for your device.

> [http://zadig.akeo.ie/](http://zadig.akeo.ie/ "http://zadig.akeo.ie/")  
> **_  
> Zadig_** is a Windows application that installs generic USB drivers, such as [WinUSB](http://msdn.microsoft.com/en-us/library/windows/hardware/ff540174.aspx), [libusb-win32/libusb0.sys](http://sourceforge.net/apps/trac/libusb-win32/wiki) or [libusbK](http://code.google.com/p/usb-travis/), to help you access USB devices.

Zadig的使用极其简单，通过USB ID确定需要安装驱动的USB设备，然后再右边的列表中选择安装的驱动类型（这里以WinUSB为例），点击“Install Driver”即可，如下图所示：

[![image](http://www.wowotech.net/content/uploadfile/201707/fc14b80aa29aafc174432f4167535e9e20170723142114.png "image")](http://www.wowotech.net/content/uploadfile/201707/b085b16f660d3bf0e9b640e276d01ac720170723142053.png)

4）再次运行dfu工具，已经和开发板对上暗号了，成功！！

> $ /d/work/xprj/build/../tools/dfu/dfu bubblegum 0xe406b200 /d/work/xprj/build/../tools/actions/splboot.bin 1  
> board_bubblegum_init  
> board bubblegum  
> address 0xe406b200  
> filename d:/work/xprj/build/../tools/actions/splboot.bin  
> need_run 1  
> bubblegum_init  
> b96_init_usb  
> b96_init_device  
> bDescriptorType: 1  
> bNumConfigurations: 1  
> iManufacturer: 0  
> bNumInterfaces: 1  
> Info: cannot detach kernel driver: LIBUSB_ERROR_NOT_SUPPORTED  
> Configuiration: 1  
> Handler: 009AB270  
> bubblegum_upload, filename d:/work/xprj/build/../tools/actions/splboot.bin, addr 0xe406b200  
> writeBinaryFile  
> writeBinaryFileSeek  
> CBW: 55 53 42 43 00 00 00 00 83 43 00 00 00 00 10 05 00 b2 06 e4 83 43 00 00 00 00 00 00 00 00 00  
> Bulk transferred 17283 bytes  
> readCSW  
> CSW:55 53 42 53 00 00 00 00 00 00 00 00 00  
> bubblegum_run, addr 0xe406b200  
> unknownCMD07  
> CBW: 55 53 42 43 00 00 00 00 00 00 00 00 00 00 00 10 00 b2 06 e4 00 00 00 00 00 00 00 00 00 00 00  
> Transffered: 31  
> readCSW  
> CSW:55 53 42 53 00 00 00 00 00 00 00 00 00  
> bubblegum_exit  
> b96_uninit_device  
> b96_uninit_usb

## 3. 参考文档

[1] [【任务1】启动过程-Boot from USB](http://www.wowotech.net/forum/viewtopic.php?id=15)

[2] xprj dfu, [https://github.com/wowotechX/tools/tree/master/dfu](https://github.com/wowotechX/tools/tree/master/dfu "https://github.com/wowotechX/tools/tree/master/dfu")

[3] libusb, [http://libusb.info/](http://libusb.info/ "http://libusb.info/")

[4] [X-010-UBOOT-使用booti命令启动kernel(Bubblegum-96平台)](http://www.wowotech.net/x_project/bubblegum_uboot_booti.html)

[5] [https://github.com/libusb/libusb/wiki/Windows#How_to_use_libusb_on_Windows](https://github.com/libusb/libusb/wiki/Windows#How_to_use_libusb_on_Windows "https://github.com/libusb/libusb/wiki/Windows#How_to_use_libusb_on_Windows")

[6] MinGW, [www.mingw.org/](http://www.mingw.org/ "http://www.mingw.org/")

[7] [Windows系统结合MinGW搭建软件开发环境](http://www.wowotech.net/soft/6.html)

[8] zadig, [http://zadig.akeo.ie/](http://zadig.akeo.ie/ "http://zadig.akeo.ie/")

[9] [X-002-HW-S900芯片boot from USB有关的硬件描述](http://www.wowotech.net/x_project/s900_hw_adfu.html)

原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/x_project/libusb_on_windows.html)。

标签: [MinGW](http://www.wowotech.net/tag/MinGW) [libusb](http://www.wowotech.net/tag/libusb) [windows](http://www.wowotech.net/tag/windows) [zadig](http://www.wowotech.net/tag/zadig) [dfu](http://www.wowotech.net/tag/dfu)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [《奔跑吧，Linux内核》已经上架预售了](http://www.wowotech.net/tech_discuss/running_kernel.html) | [Dynamic DMA mapping Guide](http://www.wowotech.net/memory_management/DMA-Mapping-api.html)»

**评论：**

**King Jin**  
2017-07-28 14:37

马克一下

[回复](http://www.wowotech.net/x_project/libusb_on_windows.html#comment-5856)

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
    
    - [linux内核中的GPIO系统之（5）：gpio subsysem和pinctrl subsystem之间的耦合](http://www.wowotech.net/gpio_subsystem/pinctrl-and-gpio.html)
    - [傅立叶级数（Fourier Series）和周期现象](http://www.wowotech.net/basic_subject/Fourier_Series.html)
    - [计算机科学基础知识（一）:The Memory Hierarchy](http://www.wowotech.net/basic_subject/memory-hierarchy.html)
    - [Linux的时钟](http://www.wowotech.net/timer_subsystem/clock-id-in-linux.html)
    - [Perf book 9.3章节翻译（下）](http://www.wowotech.net/kernel_synchronization/perfbook-9-3-rcu.html)
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
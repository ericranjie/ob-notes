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

作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2017-2-27 19:28 分类：[Linux应用技巧](http://www.wowotech.net/sort/linux_application)

一、前言

一直以来，我都是在使用一台ThinkPad T450 ＋ Debian 8的机器来研究内核，Debian 8上缺省的内核版本是3.16，为什么不把内核升级到4.4.6版本上呢？反正现在蜗窝主要分析的也是这个版本的内核？

本文主要记录了整个升级过程，方便后续重复使用，哈哈，也许哪天要升级到8.8版本的内核呢，到时候可以把这份文档调出来轻松升级。

二、搞清楚/boot目录下的东西

首先列车/boot目录下的文件如下：

> -rw-r--r-- 1 root root   157726 Feb 29  2016 config-3.16.0-4-amd64\
> drwxr-xr-x 5 root root     4096 Apr  1  2016 grub\
> -rw-r--r-- 1 root root 16873678 Sep 26 16:14 initrd.img-3.16.0-4-amd64\
> -rw-r--r-- 1 root root  2676357 Feb 29  2016 System.map-3.16.0-4-amd64\
> -rw-r--r-- 1 root root  3119232 Feb 29  2016 vmlinuz-3.16.0-4-amd64

vmlinuz-3.16.0-4-amd64是Debian 8使用的kernel image，这个image是如何配置的呢？config-3.16.0-4-amd64就是对应的内核配置文件，System.map-3.16.0-4-amd64是对应的内核符号表，调试内核的时候也许会用到它。initrd.img-3.16.0-4-amd64是内核使用的inital ramdisk image。

OK，收集了这些信息后，我们先理一理思路：要想升级到4.4.6内核，估计必须要提供上面四个文件，配置文件可以沿用，并且用这个配置文件编译出一个新版本的内核image，initrd image有点麻烦，可以考虑沿用，也可以看看是否可以生成一个。符号表文件最简单，编译好4.4.6内核自然就有了。搞定！

三、编译4.4内核

1、准备好4.4.6的源代码

去kernel.org上去下载一个linux-4.4.6.tar.xz，然后解压在自己的工作目录，例如：/home/linuxer/work/linux-4.4.6/。

> xd –d linux-4.4.6.tar.xz
>
> tar –xf linux-4.4.6.tar

2、准备好内核配置文件

自己生成一个配置文件比较费事，借用当前kernel的配置文件是一个不错的主意。

> cd /home/linuxer/work/linux-4.4.6
>
> cp /boot/config-3.16.0-4-amd64 ./.config

3、配置内核

> make olddefconfig

从3.16到4.4，内核的配置项一定会有变化，因此使用旧的内核配置文件存在这样的问题：4.4内核中新的配置项如何选择？如果你的内心足够强大，可以考虑使用make oldconfig，这样，那些新的配置项就可以逐一列出了并请你进行配置。当然，我是没有那么多耐心了，直接使用make olddefconfig，让那些新增加的配置项选择default值吧。

4、生成内核包

> make clean
>
> make deb-pkg

这条命令是用来编译kernel package的。输完该命令后，我建议你可以外出走走，上个厕所，喝杯茶，考虑一下诗和远方什么的，反正随后的一段时间，你的计算机屏幕只是有一行行的字符不断的翻滚而已。当一切归于平静的时候，在kernel source目录的上一级目录可以看到不少的\*.deb的包文件，当然，最重要的就是那个linux-image-4.4.6_4.4.6-2_amd64.deb，这个是内核安装包。

四、安装内核包

下面的命令用来安装内核包：

> dpkg -i linux-image-4.4.6_4.4.6-2_amd64.deb

作为一个有情怀的工程师，我们当然想知道到底安装了哪些文件，这个信息其实可以从/var/lib/dpkg/info/linux-image-4.4.6.list文件中得到，我们简单整理如下：

（1）/boot目录下的内核镜像（vmlinuz-4.4.6）、内核配置文件（config-4.4.6）和内核符号（System.map-4.4.6）

（2）内核的模块（/lib/modules/4.4.6/\*）

（3）一些内核安装相关的处理脚本（/etc/kernel/\*）。主要是和initrd image以及grub更新相关的工作。通过这些脚本生成了/boot目录下的initrd.img-4.4.6文件（哈哈，initrd image的问题解决了）并且修改了/boot/grub/menu.lst文件。

五、加载内核

很遗憾，安装内核package之后重启系统，一切都没有什么变化，仍然是旧的内核，稍微看了看网上的文章，似乎是和grub的配置相关。升级内核这事以前也做过，对于grub而言，修改menu.lst就OK了，现在的grub2似乎变化了，其配置文件是grub.cfg，而且建议不要手工修改，好吧，看来还是要更新grub的知识啊。

grub的配置文件包括：

（1）/boot/grub/grub.cfg

（2）/etc/grub.d/

（3）/etc/default/grub

有一个能够自动生成配置文件的命令grub-mkconfig，我们可以用这个命令来生成配置文件：

> grub-mkconfig –o /boot/grub/grub.cfg

执行完该命令后，系统会扫描boot目录的内容，以及etc目录下的配置文件，最终生成一个可以使用的grub.cfg文件。OK，现在可以重启享受4.4.6内核吧。

_原创文章，转发请注明出处。蜗窝科技_

标签: [内核升级](http://www.wowotech.net/tag/%E5%86%85%E6%A0%B8%E5%8D%87%E7%BA%A7)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [eMMC 原理 4 ：总线协议](http://www.wowotech.net/basic_tech/emmc_bus_protocol.html) | [Linux系统如何标识进程？](http://www.wowotech.net/process_management/pid.html)»

**评论：**

**orangeboyye**\
2022-04-27 03:09

用发行版的config有个问题，就是发行版为了在各种机器上运行配置了大量的ko，导致编译时间会非常长。其实内核已经给我们提供了一个方法 make localmodconfig，会根据系统现在的状态只配置目前加载的ko，这样编译时间就会非常快，在我的电脑上virtualbox虚拟机里4核编译不到30分钟就编译好了。

[回复](http://www.wowotech.net/linux_application/debian8-upgrade-kernel.html#comment-8597)

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

  - [Linux电源管理(6)\_Generic PM之Suspend功能](http://www.wowotech.net/pm_subsystem/suspend_and_resume.html)
  - [Linux时间子系统之（三）：用户空间接口函数](http://www.wowotech.net/timer_subsystem/timer_subsystem_userspace.html)
  - [Linux设备模型(6)\_Bus](http://www.wowotech.net/device_model/bus.html)
  - [从“码农”说起](http://www.wowotech.net/tech_discuss/111.html)
  - [Linux CPU core的电源管理(3)\_cpu ops](http://www.wowotech.net/pm_subsystem/cpu_ops.html)

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


作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2016-12-20 22:36 分类：[X Project](http://www.wowotech.net/sort/x_project)

# 1. 前言

话说在半年前，乐美客送给蜗窝几块Hikey(乐美客版)开发板\[1\]，不过由于太忙，就一直把它们放在角落里思考人生，因此甚是愧疚。这几天，闲来无事，翻了下Hikey的资料，觉得挺有意思，就想花点时间让“[X Project](http://www.wowotech.net/sort/x_project)”在这个板子上跑起来。当然，按照“规矩”，先从“[【任务1】启动过程-Boot from USB](http://www.wowotech.net/forum/viewtopic.php?id=15)”做起，记录如下。

# 2. Hikey介绍

说实话，作为一块开发板，Hikey有着国产平台的通病----资料匮乏，但它的source code却非常完整：

> 除ROM code之外的所有代码，都能在github上看到源码，这对ARM linux的学习（“[X Project](http://www.wowotech.net/sort/x_project)”的关注点）来说，是非常有用的。

另外，之所以觉得这块板子很有意思，是因为海思的这颗SOC----[Hi6220V100](https://github.com/96boards/documentation/blob/master/ConsumerEdition/HiKey/HardwareDocs/Hi6220V100_Multi-Mode_Application_Processor_Function_Description.pdf)\[2\]，它有如下的处理器架构：

![[Pasted image 20241021184109.png]]

图片1 Hi6220V100 SOC Architecture

> 该SOC包含两个处理器子系统：
>
> MCU subsystem，里面有一个Cortex-M3的核，用于充电管理、电源管理等，也包括Boot code的USB download协议（猜测）；
>
> ACPU(这个缩写也太逗了，哈哈) subsystem，采用CCI-400互联总线（CoreLink）协议，支持8个Cortex-A53核（2个cluster\[3\]，每个cluster包含4个完全一样 A53核）。

基于上述的架构，Hikey USB boot的流程如下：

![[Pasted image 20241021184130.png]]

图片2 Hi6220V100 USB BOOT

注1：网上没有官方的spec介绍USB boot流程，上面的流程是我根据github上的开源代码，推测出来的，具体可参考：

> 1）l-loader，[https://github.com/96boards/l-loader.gi](https://github.com/96boards/l-loader.git "https://github.com/96boards/l-loader.git")[t](https://github.com/96boards/l-loader.git "https://github.com/96boards/l-loader.git")，其中的[start.S](https://github.com/96boards/l-loader/blob/master/start.S)即为上面图片中的“l-loader”，它是一段运行于Cortex-M3的汇编代码。另外由链接脚本（[l-loader.lds](https://github.com/96boards/l-loader/blob/master/l-loader.lds)）可知，它的执行位置0xf9800800。
>
> 2）ARM Trusted Firmware BL1，[https://github.com/96boards/arm-trusted-firmware/tree/hikey/bl1](https://github.com/96boards/arm-trusted-firmware/tree/hikey/bl1 "https://github.com/96boards/arm-trusted-firmware/tree/hikey/bl1")，l-loader正确初始化Cortex-A53之后，会将A53的控制权交给ARM Trusted Firmware，首先运行的是BL1，对Hikey来说，BL1的运行地址为0xf9801000（参考BL1_RO_BASE的定义）。
>
> 3）[https://github.com/96boards/l-loader.gi](https://github.com/96boards/l-loader.git "https://github.com/96boards/l-loader.git")[t](https://github.com/96boards/l-loader.git "https://github.com/96boards/l-loader.git")中有一个脚本（[gen_loader.py](https://github.com/96boards/l-loader/blob/master/gen_loader.py)），可以将l-loader和BL1打包生成一个二进制文件，然后通过[hisi-idt.py](http://builds.96boards.org/releases/reference-platform/aosp/hikey/15.10/bootloader/hisi-idt.py)脚本，将这个二进制文件下载到0xf9800800处执行。

注2：Hikey的官方Image并没有使用常规的bootloader（u-boot等），而是用ARM Trusted Firmware代替了bootloader的功能，因此它的boot过程为：l-loader---->ARM Trusted Firmware(BL1, BL2, BL3等)---->Linux kernel。另外，ARM Trusted Firmware也集成了Android的fast boot功能，用起来还是很方便的。感兴趣的同学可以研究一下相关的source code。

# 3. 移植u-boot SPL到Hikey中

本章将根据第2章罗列的信息，基于“[X Project](http://www.wowotech.net/sort/x_project)”有关的文档，尝试将u-boot的SPL跑在Hikey开发板上，并点亮一盏灯，步骤如下。

## 3.1 修改“[X Project](http://www.wowotech.net/sort/x_project)”的编译脚本，加入对Hikey的支持

不再详细描述修改步骤，具体可参考下面的patch：

> [\[xprj, hikey\] support hikey.](https://github.com/wowotechX/build/commit/030be3ea25a4d103ca515d898e95d33b849b89db)

## 3.2 修改“[X Project](http://www.wowotech.net/sort/x_project)”的u-boot，增加对Hikey SPL的支持

mainline的u-boot已经支持了Hikey，但没有使能SPL功能，这刚好给我们大显身手的机会，步骤如下：

1）修改“arch/arm/Kconfig”，找到config TARGET_HIKEY配置项，增加select SUPPORT_SPL和select SPL，以支持SPL功能

```cpp
config TARGET_HIKEY  
bool "Support HiKey 96boards Consumer Edition Platform"  
select ARM64  
+ select SUPPORT_SPL  
+ select SPL  
select DM  
select DM_GPIO  
select DM_SERIAL
```

2）修改完成后，在build目录执行make uboot-config，打开配置界面后，直接保存退出，从新生成config文件，具体可参考：

> [https://github.com/wowotechX/u-boot/blob/1775dca1f14df5b3ba0884c54b665f91fff24933/configs/hikey_defconfig](https://github.com/wowotechX/u-boot/blob/1775dca1f14df5b3ba0884c54b665f91fff24933/configs/hikey_defconfig "https://github.com/wowotechX/u-boot/blob/1775dca1f14df5b3ba0884c54b665f91fff24933/configs/hikey_defconfig")

3）修改include/configs/hikey.h文件，增加SPL有关的TEXT配置，如下（黄色部分比较重要）：

```cpp
define CONFIG_SPL_TEXT_BASE 0xf9801000  
define CONFIG_SPL_MAX_SIZE (1024 * 20)  
define CONFIG_SPL_BSS_START_ADDR (CONFIG_SPL_TEXT_BASE + \  
CONFIG_SPL_MAX_SIZE)  
define CONFIG_SPL_BSS_MAX_SIZE (1024 * 12)  
define CONFIG_SPL_STACK (CONFIG_SPL_BSS_START_ADDR + \  
CONFIG_SPL_BSS_MAX_SIZE + \  
1024 * 12)
```

4）修改board/hisilicon/hikey/hikey.c，加入两个和SPL有关的函数实现，board_init_f和panic，并在board_init_f中点亮一盏LED灯，代码如下：

```cpp
+#ifdef CONFIG_SPL_BUILD  
+void board_init_f(ulong bootflag)  
+{  
+     /* GPIO4_2(User LED3), 0xF7020000, GPIODIR(0x400), GPIODAT2(0x10) */  
+     writel(readl(0xF7020400) | (1 << 2), 0xF7020400);  
+     writel(readl(0xF7020010) | 0xFF, 0xF7020010);  
+     while (1);  
+}  
+  
+void panic(const char *fmt, ...)  
+{  
+}  
+#endif
```

其中LED有关的配置可参考Hikey（乐美客版）的原理图\[7\]以及[Hi6220V100](https://github.com/96boards/documentation/blob/master/ConsumerEdition/HiKey/HardwareDocs/Hi6220V100_Multi-Mode_Application_Processor_Function_Description.pdf)的spec\[2\]。

修改完毕后，编译生成u-boot-spl.bin，留作后用。

## 3.3 单独编译出l-loader

从[https://github.com/96boards/l-loader.gi](https://github.com/96boards/l-loader.git "https://github.com/96boards/l-loader.git")[t](https://github.com/96boards/l-loader.git "https://github.com/96boards/l-loader.git")中将l-loader单独编译出，并保存在“[X Project](http://www.wowotech.net/sort/x_project)”的tools/hisilicon目录中，留作后用，如下：

> [https://github.com/wowotechX/tools/blob/hikey/hisilicon/img_loader.bin](https://github.com/wowotechX/tools/blob/hikey/hisilicon/img_loader.bin "https://github.com/wowotechX/tools/blob/hikey/hisilicon/img_loader.bin")

注3：l-loader的编译方法，这里不再详细介绍（无非就是下载一个gcc-arm-linux-gnueabihf交叉编译器，稍微修改一下[l-loader.git](https://github.com/96boards/l-loader.git "https://github.com/96boards/l-loader.git")的Makefile文件）。

## 3.4 将[gen_loader.py](https://github.com/96boards/l-loader/blob/master/gen_loader.py)和[hisi-idt.py](http://builds.96boards.org/releases/reference-platform/aosp/hikey/15.10/bootloader/hisi-idt.py)保存到“[X Project](http://www.wowotech.net/sort/x_project)”的tools/hisilicon目录中

如下：

> [https://github.com/wowotechX/tools/blob/hikey/hisilicon/gen_loader.py](https://github.com/wowotechX/tools/blob/hikey/hisilicon/gen_loader.py "https://github.com/wowotechX/tools/blob/hikey/hisilicon/gen_loader.py")
>
> [https://github.com/wowotechX/tools/blob/hikey/hisilicon/hisi-idt.py](https://github.com/wowotechX/tools/blob/hikey/hisilicon/hisi-idt.py "https://github.com/wowotechX/tools/blob/hikey/hisilicon/hisi-idt.py")

## 3.5 修改“[X Project](http://www.wowotech.net/sort/x_project)”的编译脚本，增加Hikey u-boot-spl的运行命令

如下：

```cpp
img-loader=(TOOLS_DIR)/(BOARD_VENDOR)/img_loader.bin  
uboot-spl-bin=$(UBOOT_OUT_DIR)/spl/u-boot-spl.bin  
gen-loader=(TOOLS_DIR)/(BOARD_VENDOR)/gen_loader.py  
hisi-idt=(TOOLS_DIR)/(BOARD_VENDOR)/hisi-idt.py  
```

spl-run:

#generate SPL image

sudo python (gen-loader) -o spl.img --img_loader=(img-loader) --img_bl1=$(uboot-spl-bin)\
sudo python $(hisi-idt) --img1=spl.img -d /dev/ttyUSB0\
rm -f spl.img

主要思路为：

1）使用[gen_loader.py](https://github.com/96boards/l-loader/blob/master/gen_loader.py)脚本将img_loader.bin和u-boot-spl.bin打包在一起，生成spl.img文件。

2）使用[hisi-idt.py](http://builds.96boards.org/releases/reference-platform/aosp/hikey/15.10/bootloader/hisi-idt.py)将spl.img下载到板子中运行。

## 3.6 运行并测试

按照\[1\]中的步骤，为Hikey供电，并让它进入USB boot模式，然后在“[X Project](http://www.wowotech.net/sort/x_project)”的build目录中执行make spl-run即可。观察Hikey的LED灯，应该有一个亮了，说明移植成功。

## 4. 参考文档

\[1\] [http://wiki.lemaker.org/HiKey(LeMaker_version):Quick_Start/zh-hans](<http://wiki.lemaker.org/HiKey(LeMaker_version):Quick_Start/zh-hans> "http://wiki.lemaker.org/HiKey(LeMaker_version):Quick_Start/zh-hans")

\[2\] [Hi6220V100](https://github.com/96boards/documentation/blob/master/ConsumerEdition/HiKey/HardwareDocs/Hi6220V100_Multi-Mode_Application_Processor_Function_Description.pdf)

\[3\] [Linux CPU core的电源管理(2)\_cpu topology](http://www.wowotech.net/pm_subsystem/cpu_topology.html)

\[4\] Hikey l-loader，[https://github.com/96boards/l-loader.gi](https://github.com/96boards/l-loader.git "https://github.com/96boards/l-loader.git")[t](https://github.com/96boards/l-loader.git "https://github.com/96boards/l-loader.git")

\[5\] Hikey USB download脚本，[http://builds.96boards.org/releases/reference-platform/aosp/hikey/15.10/bootloader/hisi-idt.py](http://builds.96boards.org/releases/reference-platform/aosp/hikey/15.10/bootloader/hisi-idt.py "http://builds.96boards.org/releases/reference-platform/aosp/hikey/15.10/bootloader/hisi-idt.py")

\[6\] ARM Trusted Firmware, [https://github.com/96boards/arm-trusted-firmware.git](https://github.com/96boards/arm-trusted-firmware.git "https://github.com/96boards/arm-trusted-firmware.git")

\[7\] Hikey（乐美客版）原理图，[http://mirror.lemaker.org/LeMaker%20Hikey%20Schematic.pdf](http://mirror.lemaker.org/LeMaker%20Hikey%20Schematic.pdf "http://mirror.lemaker.org/LeMaker%20Hikey%20Schematic.pdf")

原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/x_project/hikey_usb_boot.html)。

标签: [USB](http://www.wowotech.net/tag/USB) [boot](http://www.wowotech.net/tag/boot) [u-boot](http://www.wowotech.net/tag/u-boot) [spl](http://www.wowotech.net/tag/spl) [hikey](http://www.wowotech.net/tag/hikey) [Hi6220V100](http://www.wowotech.net/tag/Hi6220V100)

---

« [MMC/SD/SDIO介绍](http://www.wowotech.net/basic_tech/mmc_sd_sdio_intro.html) | [Linux serial framework(1)\_概述](http://www.wowotech.net/comm/serial_overview.html)»

**评论：**

**running**\
2022-06-26 09:38

start.s应该不是运行在cortex M3上的，很多ROM code会运行在AArch32模式，比如高通等，hikey应该也是一样，AP上电工作在AArch32，通过warm reset执行BL2时，切换为AArch64.

[回复](http://www.wowotech.net/x_project/hikey_usb_boot.html#comment-8633)

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

  - [Linux电源管理(10)\_autosleep](http://www.wowotech.net/pm_subsystem/autosleep.html)
  - [中断唤醒系统流程](http://www.wowotech.net/irq_subsystem/irq_handle_procedure.html)
  - [Linux内核同步机制之（五）：Read/Write spin lock](http://www.wowotech.net/kernel_synchronization/rw-spinlock.html)
  - [Linux内核同步机制之（三）：memory barrier](http://www.wowotech.net/kernel_synchronization/memory-barrier.html)
  - [ARMv8之memory model](http://www.wowotech.net/armv8a_arch/memory-model.html)

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

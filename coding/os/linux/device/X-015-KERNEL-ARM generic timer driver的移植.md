
作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2016-11-2 22:31 分类：[X Project](http://www.wowotech.net/sort/x_project)

## 1. 前言

本文将基于“[Linux时间子系统之（十七）：ARM generic timer驱动代码分析](http://www.wowotech.net/timer_subsystem/armgeneraltimer.html)\[1\]”，以bubblegum-96平台为例，介绍ARM generic timer的移植步骤。

另外，我们在\[2\]中完成了ARM GIC驱动的移植，但还没有测试是否可用。刚好借助timer驱动，测试GIC是否可以正常工作，顺便理解Interrupt的使用方法。

## 2. Generic timer硬件说明

有关ARM generic timer的技术细节，可参考\[1\]。本文所使用的bubblegum-96平台，其SOC包含了4个Cortex A53的core，支持CP15 type的Generic timer。为了驱动它，我们需要如下两个信息：

> 1）System counter的输入频率。
>
> 2）Per-CPU的timer和GIC之间的接口（即这些Timer的中断源以及中断触发方式）。参考\[2\]中的介绍，对于支持virtualization extension的SOC，每个cpu有四个timer：Non-secure PL1 physical timer，Secure PL1 physical timer，Non-secure PL2 physical timer和virtual timer ，因此将会有四个中断源。

不过，和我们移植GIC驱动\[2\]时遇到的问题一样，我们找不到bubblegum-96平台有关的信息（大家在开发自己的平台时，应该没有这些困扰），只能从开源出来的代码中\[5\]反推，结论如下：

> 1）System counter的输入频率为24MHz，这一点可以从\[4\]中推测出来，因为bubblegum-96开发板的晶振是24MHz，一般system counter直接使用晶振为输入时钟。
>
> 2）4个Per-CPU timer的中断源分别是：Non-secure PL1 physical timer----PPI 13，Secure PL1 physical timer----PPI 14，Non-secure PL2 physical timer----PPI 11，virtual timer----PPI 10。 它们都是低电平触发的方式。

## 3. Generic timer移植

#### 3.1 u-boot中的移植

记得我们在“[X-013-UBOOT-使能autoboot功能](http://www.wowotech.net/x_project/uboot_autoboot.html)”中调试u-boot的autoboot功能的时候，由于没有timer驱动，无法正确使用CONFIG_BOOTDELAY（因为无法计时）。既然本文要在kernel中移植generic timer，就顺便提一下，在u-boot中支持ARM generic timer的方法。

其实很简单，对于ARM64平台来说，支持generic timer只需要知道System counter的输入频率，并通过COUNTER_FREQUENCY宏定义告诉u-boot即可，如下（我们同时修改boot delay为5s，以验证timer功能）：

> /\* include/configs/bubblegum.h \*/
>
> @@ -74,7 +74,8 @@\
> #define CONFIG_CMD_BOOTI
>
> -#define CONFIG_BOOTDELAY -2\
> +#define CONFIG_BOOTDELAY 5\
> #define CONFIG_BOOTCOMMAND "bootm 0x6400000"
>
> +#define COUNTER_FREQUENCY (24000000) /\* 24MHz \*/\
> #endif

修改完成后，编译u-boot并启动kernel，就可以看到自动boot之前的倒计时了，说明timer添加成功：

> cd ~/work/xprj/build\
> make u-boot uImage\
> make spi-run
>
> make kernel-run

#### 3.2 kernel中的移植

在kernel中的移植也很简单，只需要修改dts文件，添加generic timer的节点，并提供第2章所描述的硬件信息即可：

> /\* ./arch/arm64/boot/dts/actions/s900-bubblegum.dts \*/
>
> +#include\
> +\
> / {\
> model = "Bubblegum-96 Development Board";\
> compatible = "actions,s900-bubblegum", "actions,s900";\
> +\
> +       timer {\
> +               compatible = "arm,armv8-timer";\
> +               interrupts = ,\
> +                            ,\
> +                            ,\
> +                            ;\
> +               clock-frequency = \<24000000>;\
> +       };\
> };

说明如下：

1）为了使用GIC有关的信息，我们需要包含“include/dt-bindings/interrupt-controller/arm-gic.h”头文件。

2）timer节点的compatible字段为"arm,armv8-timer"，表明我们是armv8 generic timer。

3）通过clock-frequency字段指定system counter的输入频率，这里为24000000（24MHz）。

4）interrupts字段指定了该设备所使用的中断源，可以是一个，也可以是多个，按照第2章的介绍，这里共有4个。参考\[2\]中的介绍，GIC的interrupt-cell为3，分别是：

> 第一个cell：PPI或SPI，此处均为PPI；
>
> 第二个cell，中断号，这里分别为13，14，11，10；
>
> 第三个cell，终端flag，这里需要包含一个GIC强制要求的----GIC_CPU_MASK_SIMPLE(4)，另外就是中断触发类型，此处全是低电平触发。

## 4. 测试和验证

## 4.1 编译及debug

修改完dts文件，编译并运行kernel，GIC移植\[4\]时最后的kernel panic（如下）不见了：

> clocksource_probe: no matching clocksources found\
> Kernel panic - not syncing: Unable to initialise architected timer.
>
> ---\[ end Kernel panic - not syncing: Unable to initialise architected timer.

取而代之的是：

> NR_IRQS:64 nr_irqs:64 0\
> arch_timer: No interrupt available, giving up\
> sched_clock: 64 bits at 250 Hz, resolution 4000000ns, wraps every 9007199254000000ns\
> Calibrating delay loop (skipped), value calculated using timer frequency.. 48.00 BogoMIPS (lpj=96000)\
> pid_max: default: 4096 minimum: 301\
> Mount-cache hash table entries: 4096 (order: 3, 32768 bytes)\
> Mountpoint-cache hash table entries: 4096 (order: 3, 32768 bytes)\
> No CPU information found in DT\
> ASID allocator initialised with 65536 entries\
> Brought up 1 CPUs\
> SMP: Total of 1 processors activated.\
> CPU: All CPU(s) started at EL2\
> clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 7645041785100000 ns\
> vdso: 2 pages (1 code @ ffffff8008185000, 1 data @ ffffff80081c4000)\
> DMA: preallocated 256 KiB pool for atomic allocations\
> workingset: timestamp_bits=60 max_order=19 bucket_order=0\
> Failed to find cpu0 device node\
> Unable to detect cache hierarchy from DT for CPU 0\
> Warning: unable to open an initial console.\
> Freeing unused kernel memory: 116K (ffffff80081a0000 - ffffff80081bd000)\
> This architecture does not have kernel memory protection.\
> Kernel panic - not syncing: No working init found.  Try passing init= option to kernel. See Linux Documentation/init.txt for guidance.\
> Kernel Offset: disabled\
> Memory Limit: none\
> ---\[ end Kernel panic - not syncing: No working init found.  Try passing init= option to kernel. See Linux Documentation/init.txt for guidance.

“arch_timer: No interrupt available, giving up”好像还有哪里不对劲，好像我们在第3章dts中指定的interrupts字段，没有被正常解析。没关系，根据\[6\]的介绍，打开所有的DEBUG输出，然后根据更为详细的日志，检查代码（过程略掉……），如下（从drivers/clocksource/arm_arch_timer.c开始）：

> CLOCKSOURCE_OF_DECLARE(armv8_arch_timer, "arm,armv8-timer", arch_timer_of_init);\
> irq_of_parse_and_map(drivers/of/irq.c)\
> of_irq_parse_one\
> of_irq_find_parent

好像是找不到interrupt parent？原来是我们的dts还没有添加interrupt-parent字段，按照下面的代码增加：

> --- a/arch/arm64/boot/dts/actions/s900-bubblegum.dts\
> +++ b/arch/arm64/boot/dts/actions/s900-bubblegum.dts
>
> @@ -12,6 +12,8 @@
>
> model = "Bubblegum-96 Development Board";\
> compatible = "actions,s900-bubblegum", "actions,s900";\
> +       interrupt-parent = \<&gic>;\
> +\
> #address-cells = \<2>;\
> #size-cells = \<2>;

再次编译执行，异常消失了，算是移植完成了吧。

#### 4.2 测试

测试的方法很简单，我们把printk的时间戳打开，如果可以正确显示时间戳，则说明移植成功。打开printk时间戳的方法如下：

> cd ~/work/xprj/build;\
> make kernel-config
>
> Kernel hacking  --->\
> printk and dmesg options  --->\
> \[\*\] Show timing information on printks

然后编译并运行kernel，输出的日志如下：

> …
>
> \[    0.093281\] Failed to find cpu0 device node
>
> \[    0.097437\] Unable to detect cache hierarchy from DT for CPU 0
>
> \[    0.103562\] Warning: unable to open an initial console.
>
> \[    0.108531\] Freeing unused kernel memory: 116K (ffffff80081a0000 - ffffff80081bd000)
>
> \[    0.116156\] This architecture does not have kernel memory protection.
>
> \[    0.122593\] Kernel panic - not syncing: No working init found.  Try passing init= option to kernel. See Linux Documentation/init.txt for guidance.
>
> \[    0.135656\] Kernel Offset: disabled
>
> \[    0.139124\] Memory Limit: none
>
> \[    0.142156\] ---\[ end Kernel panic - not syncing: No working init found.  Try passing init= option to kernel. See Linux Documentation/init.txt for guidance.

成功了。

## 5. 参考文档

\[1\] [Linux时间子系统之（十七）：ARM generic timer驱动代码分析](http://www.wowotech.net/timer_subsystem/armgeneraltimer.html)

\[2\] [X-014-KERNEL-ARM GIC driver的移植](http://www.wowotech.net/x_project/gic_driver_porting.html)

\[3\] [bubblegum-96/SoC_bubblegum96.pdf](https://github.com/96boards/documentation/blob/master/ConsumerEdition/Bubblegum-96/AdditionalDocs/SoC_bubblegum96.pdf)

\[4\] [bubblegum-96/HardwareManual_Bubblegum96.pdf](https://github.com/uCRDev/documentation/blob/master/ConsumerEdition/Bubblegum-96/AdditionalDocs/HardwareManual_Bubblegum96_S900_V1.1.pdf)

\[5\] [https://github.com/96boards-bubblegum/linux/blob/bubblegum96-3.10/arch/arm64/boot/dts/s900.dtsi](https://github.com/96boards-bubblegum/linux/blob/bubblegum96-3.10/arch/arm64/boot/dts/s900.dtsi)

\[6\] [Linux kernel debug技巧----开启DEBUG选项](http://www.wowotech.net/linux_application/kernel_debug_enable.html)

原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/x_project/generic_timer_porting.html)。

标签: [Linux](http://www.wowotech.net/tag/Linux) [Kernel](http://www.wowotech.net/tag/Kernel) [内核](http://www.wowotech.net/tag/%E5%86%85%E6%A0%B8) [timer](http://www.wowotech.net/tag/timer) [porting](http://www.wowotech.net/tag/porting) [generic](http://www.wowotech.net/tag/generic)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [蓝牙协议分析(8)\_BLE安全机制之白名单](http://www.wowotech.net/bluetooth/ble_white_list.html) | [Linux kernel debug技巧----开启DEBUG选项](http://www.wowotech.net/linux_application/kernel_debug_enable.html)»

**评论：**

**im驴肉火烧**\
2016-11-03 13:54

@wowo，我回到家里，就没法访问这里了，只能在公司访问，这是为什么呢。。

[回复](http://www.wowotech.net/x_project/generic_timer_porting.html#comment-4839)

**[wowo](http://www.wowotech.net/)**\
2016-11-03 21:59

@im驴肉火烧：一直这样吗？可能服务器在香港，国内访问不太稳定。

[回复](http://www.wowotech.net/x_project/generic_timer_porting.html#comment-4842)

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

  - [msm8994 热插拔sim卡导致modem重新启动的流程](http://www.wowotech.net/190.html)
  - [Process Creation（一）](http://www.wowotech.net/process_management/Process-Creation-1.html)
  - [X-008-UBOOT-支持命令行(Bubblegum-96平台)](http://www.wowotech.net/x_project/bubblegum_uboot_cmdline.html)
  - [linux内核中的GPIO系统之（5）：gpio subsysem和pinctrl subsystem之间的耦合](http://www.wowotech.net/gpio_subsystem/pinctrl-and-gpio.html)
  - [Binder从入门到放弃（下）](http://www.wowotech.net/binder2.html)

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

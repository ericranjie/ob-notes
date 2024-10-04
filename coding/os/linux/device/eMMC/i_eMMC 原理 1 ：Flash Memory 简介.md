作者：[codingbelief](http://www.wowotech.net/author/5) 发布于：2017-1-9 19:28 分类：[基础技术](http://www.wowotech.net/sort/basic_tech)

eMMC 是 Flash Memory 的一类，在详细介绍 eMMC 之前，先简单介绍一下 Flash Memory。

Flash Memory 是一种非易失性的存储器。在嵌入式系统中通常用于存放系统、应用和数据等。在 PC 系统中，则主要用在固态硬盘以及主板 BIOS 中。另外，绝大部分的 U 盘、SDCard 等移动存储设备也都是使用 Flash Memory 作为存储介质。

## 1. Flash Memory 的主要特性

与传统的硬盘存储器相比，Flash Memory 具有质量轻、能耗低、体积小、抗震能力强等的优点，但也有不少局限性，主要如下：

1. 需要先擦除再写入  
    Flash Memory 写入数据时有一定的限制。它只能将当前为 1 的比特改写为 0，而无法将已经为 0 的比特改写为 1，只有在擦除的操作中，才能把整块的比特改写为 1。
    
2. 块擦除次数有限  
    Flash Memory 的每个数据块都有擦除次数的限制（十万到百万次不等），擦写超过一定次数后，该数据块将无法可靠存储数据，成为坏块。  
    为了最大化的延长 Flash Memory 的寿命，在软件上需要做擦写均衡（Wear Leveling），通过分散写入、动态映射等手段均衡使用各个数据块。同时，软件还需要进行坏块管理（Bad Block Management，BBM），标识坏块，不让坏块参与数据存储。（注：除了擦写导致的坏块外，Flash Memory 在生产过程也会产生坏块，即固有坏块。）
    
3. 读写干扰  
    由于硬件实现上的物理特性，Flash Memory 在进行读写操作时，有可能会导致邻近的其他比特发生位翻转，导致数据异常。这种异常可以通过重新擦除来恢复。Flash Memory 应用中通常会使用 ECC 等算法进行错误检测和数据修正。
    
4. 电荷泄漏  
    存储在 Flash Memory 存储单元的电荷，如果长期没有使用，会发生电荷泄漏，导致数据错误。不过这个时间比较长，一般十年左右。此种异常是非永久性的，重新擦除可以恢复。
    
## 2. NOR Flash 和 NAND Flash

根据硬件上存储原理的不同，Flash Memory 主要可以分为 NOR Flash 和 NAND Flash 两类。 主要的差异如下所示：

- NAND Flash 读取速度与 NOR Flash 相近，根据接口的不同有所差异；
- NAND Flash 的写入速度比 NOR Flash 快很多；
- NAND Flash 的擦除速度比 NOR Flash 快很多；
- NAND Flash 最大擦次数比 NOR Flash 多；
- NOR Flash 支持片上执行，可以在上面直接运行代码；
- NOR Flash 软件驱动比 NAND Flash 简单；
- NOR Flash 可以随机按字节读取数据，NAND Flash 需要按块进行读取。
- 大容量下 NAND Flash 比 NOR Flash 成本要低很多，体积也更小；

（注：NOR Flash 和 NAND Flash 的擦除都是按块块进行的，执行一个擦除或者写入操作时，NOR Flash 大约需要 5s，而 NAND Flash 通常不超过 4ms。）

### 2.1 NOR Flash

NOR Flash 根据与 CPU 端接口的不同，可以分为 Parallel NOR Flash 和 Serial NOR Flash 两类。  
Parallel NOR Flash 可以接入到 Host 的 SRAM/DRAM Controller 上，所存储的内容可以直接映射到 CPU 地址空间，不需要拷贝到 RAM 中即可被 CPU 访问，因而支持片上执行。Serial NOR Flash 的成本比 Parallel NOR Flash 低，主要通过 SPI 接口与 Host 连接。
  

![](https://linux.codingbelief.com/zh/storage/flash_memory/nor_flash_interface.png)

图片： Parallel NOR Flash 与 Serial NOR Flash

  

鉴于 NOR Flash 擦写速度慢，成本高等特性，NOR Flash 主要应用于小容量、内容更新少的场景，例如 PC 主板 BIOS、路由器系统存储等。

更多 NOR Flash 的相关细节，请参考 [NOR Flash](https://linux.codingbelief.com/zh/storage/flash_memory/nor_flash/index.html) 章节。

### 2.2 NAND Flash

NAND Flash 需要通过专门的 NFI（NAND Flash Interface）与 Host 端进行通信，如下图所示：

  

![](https://linux.codingbelief.com/zh/storage/flash_memory/nand_flash_interface.png)

图片：NAND Flash Interface

  

NAND Flash 根据每个存储单元内存储比特个数的不同，可以分为 SLC（Single-Level Cell）、MLC（Multi-Level Cell） 和 TLC（Triple-Level Cell） 三类。其中，在一个存储单元中，SLC 可以存储 1 个比特，MLC 可以存储 2 个比特，TLC 则可以存储 3 个比特。

NAND Flash 的一个存储单元内部，是通过不同的电压等级，来表示其所存储的信息的。在 SLC 中，存储单元的电压被分为两个等级，分别表示 0 和 1 两个状态，即 1 个比特。在 MLC 中，存储单元的电压则被分为 4 个等级，分别表示 00 01 10 11 四个状态，即 2 个比特位。同理，在 TLC 中，存储单元的电压被分为 8 个等级，存储 3 个比特信息。

  

![](https://linux.codingbelief.com/zh/storage/flash_memory/slc_mlc_tlc.png)

图片： SLC、MLC 与 TLC

  

NAND Flash 的单个存储单元存储的比特位越多，读写性能会越差，寿命也越短，但是成本会更低。Table 1 中，给出了特定工艺和技术水平下的成本和寿命数据。

  

Table 1

||SLC|MLC|TLC|
|---|---|---|---|
|制造成本|30-35 美元 / 32GB|17 美元 / 32GB|9-12 美元 / 32GB|
|擦写次数|10万次或更高|1万次或更高|5000次甚至更高|
|存储单元|1 bit / cell|2 bits / cell|3 bits / cell|

（注：以上数据来源于互联网，仅供参考）

  

相比于 NOR Flash，NAND Flash 写入性能好，大容量下成本低。目前，绝大部分手机和平板等移动设备中所使用的 eMMC 内部的 Flash Memory 都属于 NAND Flash。PC 中的固态硬盘中也是使用 NAND Flash。

更多 NAND Flash 的相关细节，请参考 [NAND Flash](https://linux.codingbelief.com/zh/storage/flash_memory/nand_flash/index.html) 章节。

## 3. Raw Flash 和 Managed Flash

由于 Flash Memory 存在按块擦写、擦写次数的限制、读写干扰、电荷泄露等的局限，为了最大程度的发挥 Flash Memory 的价值，通常需要有一个特殊的软件层次，实现坏块管理、擦写均衡、ECC、垃圾回收等的功能，这一个软件层次称为 FTL（Flash Translation Layer）。

在具体实现中，根据 FTL 所在的位置的不同，可以把 Flash Memory 分为 Raw Flash 和 Managed Flash 两类。

  

![](https://linux.codingbelief.com/zh/storage/flash_memory/raw_vs_managed_flash.png)

图片： Raw Flash 和 Managed Flash

  

Raw Flash  
在此类应用中，在 Host 端通常有专门的 FTL 或者 Flash 文件系统来实现坏块管理、擦写均衡等的功能。Host 端的软件复杂度较高，但是整体方案的成本较低，常用于价格敏感的嵌入式产品中。  
通常我们所说的 NOR Flash 和 NAND Flash 都属于这类型。

Managed Flash  
Managed Flash 在其内部集成了 Flash Controller，用于完成擦写均衡、坏块管理、ECC校验等功能。相比于直接将 Flash 接入到 Host 端，Managed Flash 屏蔽了 Flash 的物理特性，对 Host 提供标准化的接口，可以减少 Host 端软件的复杂度，让 Host 端专注于上层业务，省去对 Flash 进行特殊的处理。  
[eMMC](https://linux.codingbelief.com/zh/storage/flash_memory/emmc/index.html)、[SD Card](https://linux.codingbelief.com/zh/storage/flash_memory/sd_card/index.html)、[UFS](https://linux.codingbelief.com/zh/storage/flash_memory/ufs/index.html)、U 盘等产品是属于 Managed Flash 这一类。

## 4. 参考资料

1. [NOR NAND Flash Guide: Selecting a Flash Storage Solution](https://www.micron.com/~/media/documents/products/product-flyer/flyer_nor_nand_flash_guide.pdf) [PDF]
2. [Wiki: Common Flash Memory Interface](https://en.wikipedia.org/wiki/Common_Flash_Memory_Interface) [Web]
3. [Quick Guide to Common Flash Interface](https://www.spansion.com/Support/Application%20Notes/Quick_Guide_to_CFI_AN.pdf) [PDF]
4. [MICRON NOR Flash Technology](https://www.micron.com/products/nor-flash) [Web]
5. [MICRON NAND Flash Technology](https://www.micron.com/products/nand-flash) [Web]
6. [Wiki：闪存](https://zh.wikipedia.org/wiki/%E9%97%AA%E5%AD%98) [Web]
7. [Wiki：Flash File System](https://en.wikipedia.org/wiki/Flash_file_system) [Web]
8. [Wear Leveling in Micron® NAND Flash Memory](https://www.micron.com/~/media/documents/products/technical-note/nand-flash/tn2961_wear_leveling_in_nand.pdf) [PDF]
9. [Understanding Flash: The Flash Translation Layer](https://flashdba.com/2014/09/17/understanding-flash-the-flash-translation-layer/) [Web]
10. [谈NAND Flash的底层结构和解析](http://blog.sina.com.cn/s/blog_4b4b54da01016rx3.html) [Web]
11. [闪存基础](http://www.ssdfans.com/?p=45) [Web]
12. [Open NAND Flash Interface (ONFI)](http://www.onfi.org/) [Web]

_原创文章，转发请注明出处。蜗窝科技_  

标签: [emmc](http://www.wowotech.net/tag/emmc)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Linux MMC framework(1)_软件架构](http://www.wowotech.net/comm/mmc_framework_arch.html) | [X-022-OTHERS-git操作记录之合并远端分支的更新](http://www.wowotech.net/x_project/u_boot_merge_denx.html)»

**评论：**

**六个九十度**  
2021-01-05 10:53

cell的含义和managed flash的解释很有收获，多谢分享！

[回复](http://www.wowotech.net/basic_tech/flash_memory_intro.html#comment-8178)

**[mimolock](http://www.wowotech.net/)**  
2017-03-11 21:21

不错，之前对eMMC使用的flash有疑惑，现在豁然开朗

[回复](http://www.wowotech.net/basic_tech/flash_memory_intro.html#comment-5307)

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
    
    - [基于Hikey的"Boot from USB"调试](http://www.wowotech.net/x_project/hikey_usb_boot.html)
    - [调试手段之sys节点](http://www.wowotech.net/linux_application/15.html)
    - [Linux PM QoS framework(1)_概述和软件架构](http://www.wowotech.net/pm_subsystem/pm_qos_overview.html)
    - [systemd：为何要创建一个新的init系统软件](http://www.wowotech.net/linux_application/why-systemd.html)
    - [Linux CPU core的电源管理(1)_概述](http://www.wowotech.net/pm_subsystem/cpu_core_pm_overview.html)
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
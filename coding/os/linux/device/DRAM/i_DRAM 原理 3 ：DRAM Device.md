作者：[codingbelief](http://www.wowotech.net/author/5) 发布于：2016-7-27 22:12 分类：[基础技术](http://www.wowotech.net/sort/basic_tech)

在前面的文章中，介绍了 DRAM Cell 和 Memory Array。 本文则以 SDR SDRAM 为例，描述 DRAM Device 与 Host 端的接口，以及其内部的其他模块，包括 Control Logic、IO、Row & Column Decoder 等。

## 1. SDRAM Interface

SDR SDRAM 是 DRAM 的一种，它与 Host 端的硬件接口如下图所示：

![](http://linux.codingbelief.com/zh/memory/dram/sdram_interface.png)

总线上各个信号的描述如下表所示：

|Symbol|Type|Description|
|---|---|---|
|CLK|Input|从 Host 端输出的同步时钟信号|
|CKE|Input|用于指示 CLK 信号是否有效，SDRAM 会根据此信号进入或者退出 Power down、Self-refresh 等模式|
|CS#|Input|Chip Select 信号|
|CAS#|Input|Column Address Strobe，列地址选通信号|
|RAS#|Input|Row Address Strobe， 行地址选通信号|
|WE#|Input|Write Enable，写使能信号|
|DQML|Input|当进行写数据时，如果该 DQML 为高，那么 DQ\[7:0\] 的数据会被忽略，不写入到 DRAM|
|DQMH|Input|当进行写数据时，如果该 DQMH 为高，那么 DQ\[15:8\] 的数据会被忽略，不写入到 DRAM|
|BA\[1:0\]|Input|Bank Address，用于选择操作的 Memory Bank|
|A\[12:0\]|Input|Address 总线，用于传输行列地址|
|DQ\[15:0\]|I/O|Data 总线，用于传输读写的数据内容|

### 1.1 SDRAM Operations

Host 与 SDRAM 之间的交互都是由 Host 以 Command 的形式发起的。一个 Command 由多个信号组合而成，下面表格中描述了主要的 Command。

|Command|CS#|RAS#|CAS#|WE#|DQM|BA\[1:0\] & A\[12:0\]|DQ\[15:0\]|
|---|---|---|---|---|---|---|---|
|Active|L|L|H|H|X|Bank & Row|X|
|Read|L|H|L|H|L/H|Bank & Col|X|
|Write|L|H|L|L|L/H|Bank & Col|Valid|
|Precharge|L|L|H|L|X|Code|X|
|Auto-refresh|L|L|L|H|X|X|X|
|Self-refresh|L|L|L|H|X|X|X|
|Load Mode Register|L|L|L|L|X|REG Value|X|

#### 1.1.1 Active

Active Command 会通过 BA\[1:0\] 和 A\[12:0\] 信号，选中指定 Bank 中的一个 Row，并打开该 Row 的 wordline。在进行 Read 或者 Write 前，都需要先执行 Active Command。

#### 1.1.2 Read

Read Command 将通过 A\[12:0\] 信号，发送需要读取的 Column 的地址给 SDRAM。然后 SDRAM 再将 Active Command 所选中的 Row 中，将对应 Column 的数据通过 DQ\[15:0\] 发送给 Host。

Host 端发送 Read Command，到 SDRAM 将数据发送到总线上的需要的时钟周期个数定义为 CL。

#### 1.1.3 Write

Write Command 将通过 A\[12:0\] 信号，发送需要写入的 Column 的地址给 SDRAM，同时通过 DQ\[15:0\] 将待写入的数据发送给 SDRAM。然后 SDRAM 将数据写入到 Actived Row 的指定 Column 中。

SDRAM 接收到最后一个数据到完成数据写入到 Memory 的时间定义为 tWR （Write Recovery）。

#### 1.1.4 Precharge

在进行下一次的 Read 或者 Write 操作前，必须要先执行 Precharge 操作。（具体的细节可以参考 [DRAM Storage Cell](http://www.wowotech.net/basic_tech/307.html) 章节）

Precharge 操作是以 Bank 为单位进行的，可以单独对某一个 Bank 进行，也可以一次对所有 Bank 进行。如果 A10 为高，那么 SDRAM 进行 All Bank Precharge 操作，如果 A10 为低，那么 SDRAM 根据 BA\[1:0\] 的值，对指定的 Bank 进行 Precharge 操作。

SDRAM 完成 Precharge 操作需要的时间定义为 tPR。

#### 1.1.5 Auto-Refresh

DRAM 的 Storage Cell 中的电荷会随着时间慢慢减少，为了保证其存储的信息不丢失，需要周期性的对其进行刷新操作。

SDRAM 的刷新是按 Row 进行，标准中定义了在一个刷新周期内（常温下 64ms，高温下 32ms）需要完成一次所有 Row 的刷新操作。

为了简化 SDRAM Controller 的设计，SDRAM 标准定义了 Auto-Refresh 机制，该机制要求 SDRAM Controller 在一个刷新周期内，发送 8192 个 Auto-Refresh Command，即 AR， 给 SDRAM。

SDRAM 每收到一个 AR，就进行 n 个 Row 的刷新操作，其中，n = 总的 Row 数量 / 8192 。\
此外，SDRAM 内部维护一个刷新计数器，每完成一次刷新操作，就将计数器更新为下一次需要进行刷新操作的 Row。

一般情况下，SDRAM Controller 会周期性的发送 AR，每两个 AR 直接的时间间隔定义为 tREFI = 64ms / 8192 = 7.8 us。

SDRAM 完成一次刷新操作所需要的时间定义为 tRFC, 这个时间会随着 SDRAM Row 的数量的增加而变大。

由于 AR 会占用总线，阻塞正常的数据请求，同时 SDRAM 在执行 refresh 操作是很费电，所以在 SDRAM 的标准中，还提供了一些优化的措施，例如 DRAM Controller 可以最多延时 8 个 tREFI 后，再一起把 8 个 AR 同时发出。

更多相关的优化可以参考《大容量 DRAM 的刷新开销问题及优化技术综述》文中的描述。

#### 1.1.6 Self-Refresh

Host 还可以让 SDRAM 进入 Self-Refresh 模式，降低功耗。在该模式下，Host 不能对 SDRAM 进行读写操作，SDRAM 内部自行进行刷新操作保证数据的完整。通常在设备进入待机状态时，Host 会让 SDRAM 进入 Self-Refresh 模式，以节省功耗。

更多各个 Command 相关的细节，可以参考后续的 DRAM Timing 章节。

### 1.2 Address Mapping

SDRAM Controller 的主要功能之一是将 CPU 对指定物理地址的内存访问操作，转换为 SDRAM 读写时序，完成数据的传输。\
在实际的产品中，通常需要考虑 CPU 中的物理地址到 SDRAM 的 Bank、Row 和 Column 地址映射。下图是一个 32 位物理地址映射的一个例子：

![](http://linux.codingbelief.com/zh/memory/dram/address_mapping.png)

## 2. SDRAM 内部结构

如图所示，DRAM Device 内部主要有 Control Logic、Memory Array、Decoders、Reflash Counter 等模块。在后续的小节中，将逐一介绍各个模块的主要功能。

![](http://linux.codingbelief.com/zh/memory/dram/sdr_block_diagram.png)

### 2.1 Control Logic

Control Logic 的主要功能是解析 SDRAM Controller 发出的 Command，然后根据具体的 Command 做具体内部模块的控制，例如：选中指定的 Bank、触发 refresh 等的操作。

Control Logic 包含了 1 个或者多个 Mode Register。该 Register 中包含了时序、数据模式等的配置，更多的细节会在 [DRAM Timing](http://linux.codingbelief.com/zh/memory/dram_timing.html) 章节进行描述。

### 2.2 Row & Column Decoder

Row Decoder 的主要功能是将 Active Command 所带的 Row Address 映射到具体的 wordline，最终打开指定的 Row。同样 Column Decoder 则是把 Column Address 映射到具体的 csl，最终选中特定的 Column。

### 2.3 Memory Array

Memory Array 是存储信息的主要模块，具体细节可以参考 [DRAM Memory Orgization](http://www.wowotech.net/basic_tech/309.html) 章节的描述。

### 2.4 IO

IO 电路主要是用于处理数据的缓存、输入和输出。其中 Data Latch 和 Data Register 用于缓存数据，DQM Mask Logic 和 IO Gating 等则用于输入输出的控制。

### 2.5 Refresh Counter

Refresh Counter 用于记录下次需要进行 refresh 操作的 Row。在接收到 AR 或者在 Self-Refresh 模式下，完成 一次 refresh 后，Refresh Counter 会进行更新。

## 3. 不同类型的 SDRAM

目前市面上在使用的 DRAM 主要有 SDR、DDR、LPDDR、GDDR 这几类，后续小节中，将对各种类型的 DRAM 进行简单的介绍。

### 3.1 SDR 和 DDR

SDR（Single Data Rate） SDRAM 是第一个引入 Clock 信号的 DRAM 产品，SDR 在 Clock 的上升沿进行总线信号的处理，一个时钟周期内可以传输一组数据。

DDR（Double Data Rate） SDRAM 是在 SDR 基础上的一个更新。DDR 内部采用 2n-Prefetch 架构，相对于 SDR，在一个读写周期内可以完成 2 倍宽度数据的预取，然后在 Clock 的上升沿和下降沿都进行数据传输，最终达到在相同时钟频率下 2 倍于 SDR 的数据传输速率。（更多 2n-Prefetch 相关的细节可以参考 《Micron Technical Note - General DDR SDRAM Functionality》文中的介绍）

Prefetch 的基本原理如下图所示。在示例 B 中，内部总线宽度是 A 的两倍，在一次操作周期内，可以将两倍于 A 的数据传输到 Output Register 中，接着外部 IO 电路再以两倍于 A 的频率将数据呈现到总线上，最终实现两倍 A 的传输速率。

![](http://linux.codingbelief.com/zh/memory/dram/2n-prefetch.png)

DDR 后续还有 DDR2、DDR3、DDR4 的更新，基本上每一代都通过更多的 Prefetch 和更高的时钟频率，达到 2 倍于上一代的数据传输速率。

|DDR SDRAM Standard|Bus clock (MHz)|Internal rate (MHz)|Prefetch (min burst)|Transfer Rate (MT/s)|Voltage|
|---|---|---|---|---|---|
|DDR|100–200|100–200|2n|200–400|2.5/2.6|
|DDR2|200–533.33|100–266.67|4n|400–1066.67|1.8|
|DDR3|400–1066.67|100–266.67|8n|800–2133.33|1.5|
|DDR4|1066.67–2133.33|133.33–266.67|8n|2133.33–4266.67|1.05/1.2|

> Transfer Rate (MT/s) 为每秒发生的 Transfer 的数量，一般为 Bus Clock 的 2 倍 （一个 Clock 周期内，上升沿和下降沿各有一个 Transfer）\
> Internal rate (MHz) 则是内部 Memory Array 读写的频率。由于 SDRAM 采用电容作为存储介质，由于工艺和物理特性的限制，电容充放电的时间难以进一步的缩短，所以内部 Memory Array 的读写频率也受到了限制，目前最高能到 266.67 MHz，这也是 SDR 到 DDR 采用 Prefetch 架构的主要原因。\
> Memory Array 读写频率受到限制，那就只能在读写宽度上做优化，通过增加单次读写周期内操作的数据宽度，结合总线和 IO 频率的增加来提高整体传输速率。

### 3.2 LPDDRx

LPDDR，即 Low Power DDR SDRAM，主要是用着移动设备上，例如手机、平板等。相对于 DDR，LPDDR 采用了更低的工作电压、Partial Array Self-Refresh 等机制，降低整体的功耗，以满足移动设备的低功耗需求。

### 3.3 GDDRx

GDDR，即 Graphic DDR，主要用在显卡设备上。相对于 DDR，GDDR 具有更高的性能、更低的功耗、更少的发热，以满足显卡设备的计算需求。

## 4. 参考资料

1. Memory Systems - Cache Dram and Disk
1. 大容量 DRAM 的刷新开销问题及优化技术综述 \[PDF\]
1. Micron Technical Note - General DDR SDRAM Functionality \[PDF\]
1. [Everything You Need To Know About DDR, DDR2 and DDR3 Memories \[WEB\]](http://www.hardwaresecrets.com/everything-you-need-to-know-about-ddr-ddr2-and-ddr3-memories/)
1. [記憶體10年技術演進史 \[WEB\]](http://www.techbang.com/posts/17190)

标签: [SDRAM](http://www.wowotech.net/tag/SDRAM) [dram](http://www.wowotech.net/tag/dram)

______________________________________________________________________

« [Linux kernel内核配置解析(1)\_概述(基于ARM64架构)](http://www.wowotech.net/linux_kenrel/kernel_config_overview.html) | [X-008-UBOOT-支持命令行(Bubblegum-96平台)](http://www.wowotech.net/x_project/bubblegum_uboot_cmdline.html)»

**评论：**

**11**\
2022-09-30 01:18

第一次进行读写的时候需要precharge吗？这个precharge为什么叫做关闭行？明明是读写前的准备工作呀。

[回复](http://www.wowotech.net/basic_tech/321.html#comment-8679)

**[bsp](http://www.wowotech.net/)**\
2020-06-12 09:51

lpddr5可以接受cpu的 类似memset 指令，对一片内存清零，而不是传统的只能4/8 bytes操作。\
请问这是怎么实现的？lpddr5的控制芯片升级了？

[回复](http://www.wowotech.net/basic_tech/321.html#comment-8022)

**wink**\
2020-06-10 17:21

不需要了。precharge的作用是关闭已打开的row，每次突发的时候是在同row中对BL个column操作。只有在切换row的时候才需要precharge关闭当前row，然后再发送act激活下一个操作行.

[回复](http://www.wowotech.net/basic_tech/321.html#comment-8021)

**wink**\
2020-06-10 17:20

不需要了。precharge的作用是关闭已打开的row，每次突发的时候是在同row中对BL个column操作。只有在切换row的时候才需要precharge关闭当前row，然后再发送act激活下一个操作行，

[回复](http://www.wowotech.net/basic_tech/321.html#comment-8020)

**CARL**\
2019-07-29 13:17

請教一下,DQ是什麼的縮寫?

[回复](http://www.wowotech.net/basic_tech/321.html#comment-7554)

**LeonZHAO**\
2019-07-16 19:28

关于Precharge部分，如果一开始对某一个Column进行读写操作，之后对对于同一Bank同一Row不同Column进行读写时，是否还需要进行Precharge？

[回复](http://www.wowotech.net/basic_tech/321.html#comment-7533)

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

  - [Linux内核的自旋锁](http://www.wowotech.net/kernel_synchronization/460.html)
  - [Linux TTY framework(4)\_TTY driver](http://www.wowotech.net/tty_framework/tty_driver.html)
  - [Perf book 9.3章节翻译（下）](http://www.wowotech.net/kernel_synchronization/perfbook-9-3-rcu.html)
  - [小printf大作用（用日志打印的方式调试程序）](http://www.wowotech.net/soft/7.html)
  - [linux cpufreq framework(5)\_ARM big Little driver](http://www.wowotech.net/pm_subsystem/arm_big_little_driver.html)

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

作者：[codingbelief](http://www.wowotech.net/author/5) 发布于：2016-6-8 16:41 分类：[基础技术](http://www.wowotech.net/sort/basic_tech)

## 1. Storage Capacitor

DRAM Storage Cell 使用 Storage Capacitor 来存储 Bit 信息。

从原理层面上看，一个最简单的，存储一个 Bit 信息的 DRAM Storage Cell 的结构如下图所示：

![](http://linux.codingbelief.com/zh/memory/dram/dram_storage_cell.png)

由以下 4 个部分组成：

- Storage Capacitor，即存储电容，它通过存储在其中的电荷的多和少，或者说电容两端电压差的高和低，来表示逻辑上的 1 和 0。
- Access Transistor，即访问晶体管，它的导通和截止，决定了允许或禁止对 Storage Capacitor 所存储的信息的读取和改写。
- Wordline，即字线，它决定了 Access Transistor 的导通或者截止。
- Bitline，即位线，它是外界访问 Storage Capacitor 的唯一通道，当 Access Transistor 导通后，外界可以通过 Bitline 对 Storage Capacitor 进行读取或者写入操作。

Storage Capacitor 的 Common 端接在 Vcc/2。\
当 Storage Capacitor 存储的信息为 1 时，另一端电压为 Vcc，此时其所存储的电荷

Q = +Vcc/2 / C

当 Storage Capacitor 存储的信息为 0 时，另一端电压为 0，此时其所存储的电荷

Q = -Vcc/2 / C

### 1.1 数据读写原理

从上面的结构图上分析，我们可以很容易的推测出 DRAM Storage Cell 的数据读写流程：

1. 读数据时，Wordline 设为逻辑高电平，打开 Access Transistor，然后读取 Bitline 上的状态
1. 写数据时，先把要写入的电平状态设定到 Bitline 上，然后打开 Access Transistor，通过 Bitline 改变 Storage Capacitor 内部的状态。

然而，在具体实现上，如果按照上面的流程对 DRAM Storage Cell 进行读写，会遇到以下的问题：

1. 外界的逻辑电平与 Storage Capacitor 的电平不匹配\
   由于 Bitline 的电容值比 Storage Capacitor 要大的多（通常为 10 倍以上），当 Access Transistor 导通后，如果 Storage Capacitor 存储的信息为 1 时，Bitline 电压变化非常小。外界电路无法直接通过 Bitline 来读取 Storage Capacitor 所存储的信息。

1. 进行一次读取操作后，Storage Capacitor 存储的电荷会变化\
   在进行一次读取操作的过程中，Access Transistor 导通后，由于 Bitline 和 Storage Capacitor 端的电压不一致，会导致 Storage Capacitor 中存储的电荷量被改变。最终可能会导致在下一次读取操作过程中，无法正确的判断 Storage Capacitor 内存储的信息。

1. 由于 Capacitor 的物理特性，即使不进行读写操作，其所存储的电荷都会慢慢变少\
   这个特性要求 DRAM 在没有读写操作时，也要主动对 Storage Capacitor 进行电荷恢复的操作。

为解决上述的问题，DRAM 在设计上，引入了 Differential Sense Amplifier。

## 2. Differential Sense Amplifier

Differential Sense Amplifier 包含 Sensing Circuit 和 Voltage Equalization Circuit 两个主要部分。它主要的功能就是将 Storage Capacitor 存储的信息转换为逻辑 1 或者 0 所对应的电压，并且呈现到 Bitline 上。同时，在完成一次读取操作后，通过 Bitline 将 Storage Capacitor 中的电荷恢复到读取之前的状态。

![](http://linux.codingbelief.com/zh/memory/dram/differential_sense_amplifier.png)

在后面的小节中，我们通过完整的数据读取和写入过程，来了解 Differential Sense Amplifier 工作原理。

## 2.1 Read Operation

一个完整的 Read Operation 包含了，Precharge、Access、Sense、Restore 四个阶段。后续的小节中，将描述从 Storage Capacitor 读取 Bit 1 的完整过程。

### 2.1.1 Precharge

在这个阶段，首先会通过控制 EQ 信号，让 Te1、Te2、Te3 晶体管处于导通状态，将 Bitline 和 /Bitline 线上的电压稳定在 Vref 上, Vref = Vcc/2。然后进入到下一个阶段。

![](http://linux.codingbelief.com/zh/memory/dram/precharge.png)

### 2.1.2 Access

经过 Precharge 阶段， Bitline 和 /Bitline 线上的电压已经稳定在 Vref 上了，此时，通过控制 Wordline 信号，将 Ta 晶体管导通。Storage Capacitor 中存储正电荷会流向 Bitline，继而将 Bitline 的电压拉升到 Vref+。然后进入到下一个阶段。

![](http://linux.codingbelief.com/zh/memory/dram/access.png)

### 2.1.3 Sense

由于在 Access 阶段，Bitline 的电压被拉升到 Vref+，Tn2 会比 Tn1 更具导通性，Tp1 则会比 Tp2 更具导通性。\
此时，SAN (Sense-Amplifier N-Fet Control) 会被设定为逻辑 0 的电压，SAP (Sense-Amplifier P-Fet Control) 则会被设定为逻辑 1 的电压，即 Vcc。由于 Tn2 会比 Tn1 更具导通性，/Bitline 上的电压会更快被 SAN 拉到逻辑 0 电压，同理，Bitline 上的电压也会更快被 SAP 拉到逻辑 1 电压。接着 Tp1 和 Tn2 进入导通状态，Tp2 和 Tn1 进入截止状态。\
最后，Bitline 和 /Bitline 的电压都进入稳定状态，正确的呈现了 Storage Capacitor 所存储的信息 Bit。

![](http://linux.codingbelief.com/zh/memory/dram/sense.png)

### 2.1.4 Restore

在完成 Sense 阶段的操作后，Bitline 线处于稳定的逻辑 1 电压 Vcc，此时 Bitline 会对 Storage Capacitor 进行充电。经过特定的时间后，Storage Capacitor 的电荷就可以恢复到读取操作前的状态。

![](http://linux.codingbelief.com/zh/memory/dram/restore.png)

最后，通过 CSL 信号，让 Tc1 和 Tc2 进入导通状态，外界就可以从 Bitline 上读取到具体的信息。

### 2.1.5 Timing

整个 Read Operation 的时序如下图所示，其中的 Vcc 即为逻辑 1 所对应的电压，Gnd 为逻辑 0。

![](http://linux.codingbelief.com/zh/memory/dram/read_operation_timing.png)

## 3. Write Operation

Write Operation 的前期流程和 Read Operation 是一样的，执行 Precharge、Access、Sense 和 Restore 操作。差异在于，在 Restore 阶段后，还会进行 Write Recovery 操作。

### 3.1 Write Recovery

在 Write Recovery 阶段时，通过控制 WE (Write Enable) 信号，让 Tw1 和 Tw2 进入导通状态。此时，Bitline 会被 input 拉到逻辑 0 电平，/Bitline 则会被 /input 拉到逻辑 1 电平。\
经过特定的时间后，当 Storage Capacitor 的电荷被 Discharge 到 0 状态时，就可以通过控制 Wordline，将 Storage Capacitor 的 Access Transistor 截止，写入 0 的操作就完成了。

## 4. 参考资料

1. Memory Systems - Cache Dram and Disk

_原创文章，转发请注明出处。蜗窝科技_

标签: [SDRAM](http://www.wowotech.net/tag/SDRAM) [dram](http://www.wowotech.net/tag/dram)

______________________________________________________________________

« [通过点亮LED的方法调试嵌入式代码](http://www.wowotech.net/soft/debug_using_led.html) | [u-boot启动流程分析(2)\_板级(board)部分](http://www.wowotech.net/u-boot/boot_flow_2.html)»

**评论：**

**新手**\
2024-07-05 16:38

圖面好像顯示不出來，請問有什麼方法可以看到圖示呢謝謝

[回复](http://www.wowotech.net/basic_tech/307.html#comment-8911)

**新手**\
2024-05-24 16:18

請問為什麼Storage Capacitor 中儲存正電荷會流向Bitline 呢兩邊電壓不是都是VCC/2嗎\
感謝。

[回复](http://www.wowotech.net/basic_tech/307.html#comment-8901)

**sxr**\
2022-11-29 09:52

图片没挂，但是加载不出来，f12查看源代码找到图片链接是能打开的，求修正

[回复](http://www.wowotech.net/basic_tech/307.html#comment-8705)

**sxr**\
2022-11-29 10:13

@sxr：哦哦，没事了，貌似是加载很慢，我过了五六分钟就出来了

[回复](http://www.wowotech.net/basic_tech/307.html#comment-8706)

**hw07**\
2022-05-19 10:12

output 和/output 那个是输出？

[回复](http://www.wowotech.net/basic_tech/307.html#comment-8617)

**CCC**\
2021-02-08 21:18

请问如果sense amplifier两根bit line上都有cell并且会同时被wordline信号打开，这时sense amplifier是不是就没法正确将cell里信号放大？6F2的架构的sense amplifier的电路是怎样的？

[回复](http://www.wowotech.net/basic_tech/307.html#comment-8194)

**Jasming**\
2021-01-18 20:09

Q = +Vcc/2 / C，这个换算错了，应该是Q = +Vcc/2 * C

[回复](http://www.wowotech.net/basic_tech/307.html#comment-8185)

**HH_Xz**\
2021-01-05 17:18

写的太好了，这才是从基础把ddr弄明白了，其它地方没有找到那个能把原理说清楚的！

[回复](http://www.wowotech.net/basic_tech/307.html#comment-8179)

**北海风云**\
2019-08-27 10:39

图又挂了，能麻烦补下马，谢谢

[回复](http://www.wowotech.net/basic_tech/307.html#comment-7617)

**yygcs**\
2018-10-10 20:26

为什么当 Access Transistor 导通后，如果 Storage Capacitor 存储的信息为 1 时，Bitline 电压变化非常小呢

[回复](http://www.wowotech.net/basic_tech/307.html#comment-6976)

**陈里昂**\
2017-11-06 18:27

我不太明白，为什么 “Write Operation 的前期流程和 Read Operation 是一样的，执行 Precharge、Access、Sense 和 Restore 操作。”？我觉得没必要呀？直接Write Recovery为什么不可以？

[回复](http://www.wowotech.net/basic_tech/307.html#comment-6177)

**[codingbelief](http://www.wowotech.net/)**\
2017-11-16 10:29

@陈里昂：可以参考 “数据读写原理”小节的描述，由于一些问题，DRAM 引入了 Differential Sense Amplifier 的设计，这一设计要求写操作也需要执 Precharge、Access 等操作。

[回复](http://www.wowotech.net/basic_tech/307.html#comment-6204)

1 [2](http://www.wowotech.net/basic_tech/307.html/comment-page-2#comments)

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

  - [PELT算法浅析](http://www.wowotech.net/process_management/pelt.html)
  - [u-boot启动流程分析(1)\_平台相关部分](http://www.wowotech.net/u-boot/boot_flow_1.html)
  - [TLB flush操作](http://www.wowotech.net/memory_management/tlb-flush.html)
  - [计算机科学基础知识（一）:The Memory Hierarchy](http://www.wowotech.net/basic_subject/memory-hierarchy.html)
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

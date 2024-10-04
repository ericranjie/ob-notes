作者：[codingbelief](http://www.wowotech.net/author/5) 发布于：2016-8-29 22:42 分类：[基础技术](http://www.wowotech.net/sort/basic_tech)

在 [DRAM Device](http://www.wowotech.net/basic_tech/321.html) 章节中，我们简单介绍了 SDRAM 的 Active、Read、Write 等的操作，在本中，我们将详细的介绍各个操作的时序。

## 1. Overview

![](http://linux.codingbelief.com/zh/memory/dram/dram-command-and-data-movement.png)

如上图所示，SDRAM 的相关操作在内部大概可以分为以下的几个阶段：

1. Command transport and decode
    
    在这个阶段，Host 端会通过 Command Bus 和 Address Bus 将具体的 Command 以及相应参数传递给 SDRAM。SDRAM 接收并解析 Command，接着驱动内部模块进行相应的操作。
    
2. In bank data movement
    
    在这个阶段，SDRAM 主要是将 Memory Array 中的数据从 DRAM Cells 中读出到 Sense Amplifiers，或者将数据从 Sense Amplifiers 写入到 DRAM Cells。
    
3. In device data movement
    
    这个阶段中，数据将通过 IO 电路缓存到 Read Latchs 或者通过 IO 电路和 Write Drivers 更新到 Sense Amplifiers。
    
4. System data transport
    
    在这个阶段，进行读数据操作时，SDRAM 会将数据输出到数据总线上，进行写数据操作时，则是 Host 端的 Controller 将数据输出到总线上。
    

在上述的四个阶段中，每个阶段都会有一定的耗时，例如数据从 DRAM Cells 搬运到 Read Latchs 的操作需要一定的时间，因此在一个具体的操作需要按照一定时序进行。

同时，由于内部的一些部件可能会被多个操作使用，例如读数据和写数据都需要用到部分 IO 电路，因此多个不同的操作通常不能同时进行，也需要遵守一定的时序。

此外，某些操作会消耗很大的电流，为了满足 SDRAM 设计上的功耗指标，可能会限制某一些操作的执行频率。

基于上面的几点限制，SDRAM Controller 在发出 Command 时，需要遵守一定的时序和规则，这些时序和规则由相应的 SDRAM 标准定义。在后续的小节中，我们将对各个 Command 的时序进行详细的介绍。

## 2. 时序图例

后续的小节中，我们将通过下图类似的时序图，来描述各个 Command 的详细时序。

![](http://linux.codingbelief.com/zh/memory/dram/basic-format-of-dram-commands.png)

上图中，Clock 信号是由 SDRAM Controller 发出的，用于和 DRAM 之间的同步。在 DDRx 中，Clock 信号是一组差分信号，在本文中为了简化描述，将只画出其中的 Positive Clock。

Controller 与 DRAM 之间的交互，都是以 Controller 发起一个 Command 开始的。从 Controller 发出一个 Command 到 DRAM 接收并解析该 Command 所需要的时间定义为 tCMD，不同类型的 Command 的 tCMD 都是相同的。

DRAM 在成功解析 Command 后，就会根据 Command 在内部进行相应的操作。从 Controller 发出 Command 到 DRAM 执行完 Command 所对应的操作所需要的时间定义为 tParam。不同类型的 Command 的 tParam 可能不一样，相同 Command 的 tParam 由于 Command 参数的不同也可能会不一样。

  

+

> NOTE:  
> 各种 Command 的定义和内部操作细节可以参考前面的几篇文章，本文将主要关注时序方面的细节。

  

## 3. Row Active Command

在进行数据的读写前，Controller 需要先发送 Row Active Command，打开 DRAM Memory Array 中的指定的 Row。Row Active Command 的时序如下图所示：

![](http://linux.codingbelief.com/zh/memory/dram/row-active-command-timing.png)

Row Active Command 可以分为两个阶段：

### 3.1 Row Sense

Row Active Command 通过地址总线指明需要打开某一个 Bank 的某一个 Row。

DRAM 在接收到该 Command 后，会打开该 Row 的 Wordline，将其存储的数据读取到 Sense Amplifiers 中，这一时间定义为 tRCD（RCD for Row Address to Column Address Delay）。

DRAM 在完成 Row Sense 阶段后，Controller 就可以发送 Read 或 Write Command 进行数据的读写了。这也意味着，Controller 在发送 Row Active Command 后，需要等待 tRCD 时间才能接着发送 Read 或者 Write Command 进行数据的读写。

### 3.2 Row Restore

由于 DRAM 的特性，Row 中的数据在被读取到 Sense Amplifiers 后，需要进行 Restore 的操作（细节请参考 [DRAM Storage Cell](http://linux.codingbelief.com/zh/memory/dram/dram_storage_cell.html) 文中的描述）。Restore 操作可以和数据的读取同时进行，即在这个阶段，Controller 可能发送了 Read Command 进行数据读取。

DRAM 接收到 Row Active Command 到完成 Row Restore 操作所需要的时间定义为 tRAS（RAS for Row Address Strobe）。  
Controller 在发出一个 Row Active Command 后，必须要等待 tRAS 时间后，才可以发起另一次的 Precharge 和 Row Access。

## 4. Column Read Command

Controller 发送 Row Active Command 并等待 tRCD 时间后，再发送 Column Read Command 进行数据读取。  
数据 Burst Length 为 8 时的 Column Read Command 时序如下图所示：

![](http://linux.codingbelief.com/zh/memory/dram/column-read-command-timing.png)

Column Read Command 通过地址总线 A[0:9] 指明需要读取的 Column 的起始地址。DRAM 在接收到该 Command 后，会将数据从 Sense Amplifiers 中通过 IO 电路搬运到数据总线上。

DRAM 从接收到 Command 到第一组数据从数据总线上输出的时间称为 tCAS（CAS for Column Address Strobe），也称为 tCL（CL for CAS Latency），这一时间可以通过 mode register 进行配置，通常为 3~5 个时钟周期。

DRAM 在接收到 Column Read Command 的 tCAS 时间后，会通过数据总线，将 n 个 Column 的数据逐个发送给 Controller，其中 n 由 mode register 中的 burst length 决定，通常可以将 burst length 设定为 2、4 或者 8。

开始发送第一个 Column 数据，到最后一个 Column 数据的时间定义为 tBurst。

## 5. Column Write Command

Controller 发送 Row Active Command 并等待 tRCD 时间后，再发送 Column Write Command 进行数据写入。数据 Burst Length 为 8 时的 Column Write Command 时序如下图所示：

![](http://linux.codingbelief.com/zh/memory/dram/column-write-command-timing.png)

Column Write Command 通过地址总线 A[0:9] 指明需要写入数据的 Column 的起始地址。Controller 在发送完 Write Command 后，需要等待 tCWD （CWD for Column Write Delay） 时间后，才可以发送待写入的数据。tCWD 在一些描述中也称为 tCWL（CWL for Column Write Latency）

tCWD 在不同类型的 SDRAM 标准有所不同：

|Memory Type|tCWD|
|---|---|
|SDRAM|0 cycles|
|DDR SDRAM|1 cycle|
|DDR2 SDRAM|tCAS - 1 cycle|
|DDR3 SDRAM|programmable|

DRAM 接收完数据后，需要一定的时间将数据写入到 DRAM Cells 中，这个时间定义为 tWR（WR for Write Recovery）。

## 6. Precharge Command

在 [DRAM Storage Cell](http://linux.codingbelief.com/zh/memory/dram/dram_storage_cell.html) 章节中，我们了解到，要访问 DRAM Cell 中的数据，需要先进行 Precharge 操作。相应地，在 Controller 发送 Row Active Command 访问一个具体的 Row 前， Controller 需要发送 Precharge Command 对该 Row 所在的 Bank 进行 Precharge 操作。

下面的时序图描述了 Controller 访问一个 Row 后，执行 Precharge，然后再访问另一个 Row 的流程。

![](http://linux.codingbelief.com/zh/memory/dram/precharge-command-timing.png)

DRAM 执行 Precharge Command 所需要的时间定义为 tRP（RP for Row Precharge）。Controller 在发送一个 Row Active Command 后，需要等待 tRC（RC for Row Cycle）时间后，才能发送第二个 Row Active Command 进行另一个 Row 的访问。

从时序图上我们可以看到，tRC = tRAS + tRP，tRC 时间决定了访问 DRAM 不同 Row 的性能。在实际的产品中，通常会通过降低 tRC 耗时或者在一个 Row Cycle 执行尽可能多数据读写等方式来优化性能。

> NOTE：  
> 在一个 Row Cycle 中，发送 Row Active Command 打开一个 Row 后，Controller 可以发起多个 Read 或者 Write Command 进行一个 Row 内的数据访问。这种情况下，由于不用进行 Row 切换，数据访问的性能会比需要切换 Row 的情况好。  
> 在一些产品上，DRAM Controller 会利用这一特性，对 CPU 发起的内存访问进行调度，在不影响数据有效性的情况下，将同一个 Row 上的数据访问汇聚到一直起执行，以提供整体访问性能。

## 7. Row Refresh Command

一般情况下，为了保证 DRAM 数据的有效性，Controller 每隔 tREFI（REFI for Refresh Interval） 时间就需要发送一个 Row Refresh Command 给 DRAM，进行 Row 刷新操作。DRAM 在接收到 Row Refresh Command 后，会根据内部 Refresh Counter 的值，对所有 Bank 的一个或者多个 Row 进行刷新操作。

DRAM 刷新的操作与 Active + Precharge Command 组合类似，差别在于 Refresh Command 是对 DRAM 所有 Bank 同时进行操作的。下图为 DRAM Row Refresh Command 的时序图：

![](http://linux.codingbelief.com/zh/memory/dram/row-refresh-command-timing.png)

DRAM 完成刷新操作所需的时间定义为 tRFC（RFC for Refresh Cycle）。

tRFC 包含两个部分的时间，一是完成刷新操作所需要的时间，由于 DRAM Refresh 是同时对所有 Bank 进行的，刷新操作会比单个 Row 的 Active + Precharge 操作需要更长的时间；tRFC 的另一部分时间则是为了降低平均功耗而引入的延时，DRAM Refresh 操作所消耗的电流会比单个 Row 的 Active + Precharge 操作要大的多，tRFC 中引入额外的时延可以限制 Refresh 操作的频率。

> NOTE:  
> 在 DDR3 SDRAM 上，tRFC 最小的值大概为 110ns，tRC 则为 52.5ns。

## 8. Read Cycle

一个完整的 Burst Length 为 4 的 Read Cycle 如下图所示：

![](http://linux.codingbelief.com/zh/memory/dram/read-cycle-timing.png)

## 9. Read Command With Auto Precharge

DRAM 还可以支持 Auto Precharge 机制。在 Read Command 中的地址线 A10 设为 1 时，就可以触发 Auto Precharge。此时 DRAM 会在完成 Read Command 后的合适的时机，在内部自动执行 Precharge 操作。

Read Command With Auto Precharge 的时序如下图所示：

![](http://linux.codingbelief.com/zh/memory/dram/auto-precharge-timing.png)

Auto Precharge 机制的引入，可以降低 Controller 实现的复杂度，进而在功耗和性能上带来改善。

> NOTE:  
> Write Command 也支持 Auto Precharge 机制，参考下一小节的时序图。

## 10. Additive Latency

在 DDR2 中，又引入了 Additive Latency 机制，即 AL。通过 AL 机制，Controller 可以在发送完 Active Command 后紧接着就发送 Read 或者 Write Command，而后 DRAM 会在合适的时机（延时 tAL 时间）执行 Read 或者 Write Command。时序如下图所示：

![](http://linux.codingbelief.com/zh/memory/dram/al-read-timing.png)

![](http://linux.codingbelief.com/zh/memory/dram/al-read-timing.png)

Additive Latency 机制同样是降低了 Controller 实现的复杂度，在功耗和性能上带来改善。

## 11. DRAM Timing 设定

上述的 DRAM Timing 中的一部分参数可以编程设定，例如 tCAS、tAL、Burst Length 等。这些参数通常是在 Host 初始化时，通过 Controller 发起 Load Mode Register Command 写入到 DRAM 的 Mode Register 中。DRAM 完成初始化后，就会按照设定的参数运行。

> NOTE:  
> 初始化和参数设定过程不在本文中详细描述，感兴趣的同学可以参考具体 CPU 和 DRAM 芯片的 Datasheet。

## 12. 参考资料

1. Memory Systems - Cache Dram and Disk
2. High Performance Dram System Design Constraints and Considerations

_原创文章，转发请注明出处。蜗窝科技_  

标签: [SDRAM](http://www.wowotech.net/tag/SDRAM) [dram](http://www.wowotech.net/tag/dram)

---

« [Linux内存模型](http://www.wowotech.net/memory_management/memory_model.html) | [eMMC框架及其初始化](http://www.wowotech.net/filesystem/329.html)»

**评论：**

**JW**  
2020-08-11 12:27

讲的很透彻

[回复](http://www.wowotech.net/basic_tech/330.html#comment-8088)

**zj**  
2017-09-24 21:01

我有个问题，请楼主解答一下。因为读是破坏性的，会破坏电容中的数据，因此要进行precharge，比如读之前电容存储的是1，读之后变成了0，进行precharge后又还原成1.那如果写之前电容存储的是0，写之后变成了1，之后要进行precharge不是把数据有还原成0了吗？

[回复](http://www.wowotech.net/basic_tech/330.html#comment-6067)

**zj**  
2017-09-25 09:13

@zj：我知道了，我把 precharge 和 restore 弄混了

[回复](http://www.wowotech.net/basic_tech/330.html#comment-6068)

**Miles**  
2016-12-18 16:25

解释很透彻.赞.

[回复](http://www.wowotech.net/basic_tech/330.html#comment-5039)

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
    
    - [ARM64的启动过程之（四）：打开MMU](http://www.wowotech.net/armv8a_arch/turn-on-mmu.html)
    - [我的bash shell学习笔记](http://www.wowotech.net/linux_application/134.html)
    - [tty驱动分析](http://www.wowotech.net/tty_framework/435.html)
    - [Device Tree（二）：基本概念](http://www.wowotech.net/device_model/dt_basic_concept.html)
    - [Linux CPU core的电源管理(2)_cpu topology](http://www.wowotech.net/pm_subsystem/cpu_topology.html)
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
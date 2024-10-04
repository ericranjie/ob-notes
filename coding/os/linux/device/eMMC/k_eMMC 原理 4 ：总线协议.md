 作者：[codingbelief](http://www.wowotech.net/author/5) 发布于：2017-3-1 22:48 分类：[基础技术](http://www.wowotech.net/sort/basic_tech)

## 1. eMMC 总线接口

eMMC 总线接口定义如下图所示：

![](https://linux.codingbelief.com/zh/storage/flash_memory/emmc/emmc_host_interfaces.png)

各个信号的描述如下：

CLK

CLK 信号用于从 Host 端输出时钟信号，进行数据传输的同步和设备运作的驱动。  
在一个时钟周期内，CMD 和 DAT0-7 信号上都可以支持传输 1 个比特，即 SDR (Single Data Rate) 模式。此外，DAT0-7 信号还支持配置为 DDR (Double Data Rate) 模式，在一个时钟周期内，可以传输 2 个比特。  
Host 可以在通讯过程中动态调整时钟信号的频率（注，频率范围需要满足 Spec 的定义）。通过调整时钟频率，可以实现省电或者数据流控（避免 Over-run 或者 Under-run）功能。 在一些场景中，Host 端还可以关闭时钟，例如 eMMC 处于 Busy 状态时，或者接收完数据，进入 Programming State 时。

CMD

CMD 信号主要用于 Host 向 eMMC 发送 Command 和 eMMC 向 Host 发送对于的 Response。Command 和 Response 的细节会在后续章节中介绍。

DAT0-7

DAT0-7 信号主要用于 Host 和 eMMC 之间的数据传输。在 eMMC 上电或者软复位后，只有 DAT0 可以进行数据传输，完成初始化后，可配置 DAT0-3 或者 DAT0-7 进行数据传输，即数据总线可以配置为 4 bits 或者 8 bits 模式。

Data Strobe

Data Strobe 时钟信号由 eMMC 发送给 Host，频率与 CLK 信号相同，用于 Host 端进行数据接收的同步。Data Strobe 信号只能在 HS400 模式下配置启用，启用后可以提高数据传输的稳定性，省去总线 tuning 过程。

> NOTE:  
> Extended CSD byte[183] BUS_WIDTH 寄存器用于配置总线宽度和 Data Strobe

## 2. eMMC 总线模型

eMMC 总线中，可以有一个 Host，多个 eMMC Devices。总线上的所有通讯都由 Host 端以一个 Command 开发发起，Host 一次只能与一个 eMMC Device 通讯。

系统在上电启动后，Host 会为所有 eMMC Device 逐个分配地址（RCA，Relative device Address）。当 Host 需要和某一个 eMMC Device 通讯时，会先根据 RCA 选中该 eMMC Device，只有被选中的 eMMC Device 才会响应 Host 的 Command。

> NOTE:  
> 更详细的工作原理请参考 [eMMC 工作模式](https://linux.codingbelief.com/zh/storage/flash_memory/emmc/emmc_modes.html) 章节。

### 2.1 速率模式

随着 eMMC 协议的版本迭代，eMMC 总线的速率越来越高。为了兼容旧版本的 eMMC Device，所有 Devices 在上电启动或者 Reset 后，都会先进入兼容速率模式（Backward Compatible Mode）。在完成 eMMC Devices 的初始化后，Host 可以通过特定的流程，让 Device 进入其他高速率模式，目前支持以下的几种速率模式。

|Mode|Data Rate|Bus Width|Frequency|Max Data Transfer (x8)|
|---|---|---|---|---|
|Backward Compatible|Single|x1, x4, x8|0-26 MHz|26 MB/s|
|High Speed SDR|Single|x1, x4, x8|0-52 MHz|52 MB/s|
|High Speed DDR|Dual|x4, x8|0-52 MHz|104 MB/s|
|HS200|Single|x4, x8|0-200 MHz|200 MB/s|
|HS400|Dual|x8|0-200 MHz|400 MB/s|

> NOTE:  
> Extended CSD byte[185] HS_TIMING 寄存器可以配置总线速率模式  
> Extended CSD byte[183] BUS_WIDTH 寄存器用于配置总线宽度和 Data Strobe

### 2.2 通信模型

Host 与 eMMC Device 之间的通信都是由 Host 以一个 Command 开始发起的，eMMC Device 在完成 Command 所指定的任务后，则返回一个 Response。

#### 2.2.1 Read Data

![](https://linux.codingbelief.com/zh/storage/flash_memory/emmc/multi_block_read.png)

Host 从 eMMC Device 读取数据的流程如上图所示。

如果 Host 发送的是 Single Block Read 的 Command，那么 eMMC Device 只会发送一个 Block 的数据（一个 Block 的数据的字节数由 Host 设定或者为 eMMC Device 的默认值，更多细节请参考 [eMMC 工作模式](https://linux.codingbelief.com/zh/storage/flash_memory/emmc/emmc_modes.html) 章节）。  
如果 Host 发送的是 Multiple Block Read 的 Command，那么 eMMC Device 会持续发送数据，直到 Host 主动发送 Stop Command。

> NOTE:  
> 从 eMMC Device 读数据都是按 Block 读取的。

#### 2.2.2 Write Data

![](https://linux.codingbelief.com/zh/storage/flash_memory/emmc/multi_block_write.png)

Host 向 eMMC Device 写入数据的流程如上图所示。

如果 Host 发送的是 Single Block Write Command，那么 eMMC Device 只会将后续第一个 Block 的数据写入的存储器中。  
如果 Host 发送的是 Multiple Block Write Command，那么 eMMC Device 会持续地将接收到的数据写入到存储器中，直到 Host 主动发送 Stop Command。

eMMC Device 在接收到一个 Block 的数据后，会进行 CRC 校验，然后将校验结果通过 CRC Token 发送给 Host。  
发送完 CRC Token 后，如果 CRC 校验成功，eMMC Device 会将数据写入到内部存储器时，此时 DAT0 信号会拉低，作为 Busy 信号。Host 会持续检测 DAT0 信号，直到为高电平时，才会接着发送下一个 Block 的数据。如果 CRC 校验失败，那么 eMMC Device 不会进行数据写入，此次传输后续的数据都会被忽略。

> NOTE:  
> 向 eMMC Device 写数据都是按 Block 写入的。

#### 2.2.3 No Data

在 Host 与 eMMC Device 的通信中，有部分交互是不需要进行数据传输的，还有部分交互甚至不需要 eMMC Device 的回复 Response。

![](https://linux.codingbelief.com/zh/storage/flash_memory/emmc/no_resp_or_data.png)

#### 2.2.4 Command

![](https://linux.codingbelief.com/zh/storage/flash_memory/emmc/command_token_format.png)

如上图所示，eMMC Command 由 48 Bits 组成，各个 Bits 的解析如下所示：

|Description|Start Bit|Transmission Bit|Command Index|Argument|CRC7|End Bit|
|---|---|---|---|---|---|---|
|Bit position|47|46|[45:40]|[39:8]|[7:1]|0|
|Width (bits)|1|1|6|32|7|1|
|Value|"0"|"1"|x|x|x|"1"|

Start Bit 固定为 "0"，在没有数据传输的情况下，CMD 信号保持高电平，当 Host 将 Start Bit 发送到总线上时，eMMC Device 可以很方便检测到该信号，并开始接收 Command。

Transmission Bit 固定为 "1"，指示了该数据包的传输方向为 Host 发送到 eMMC Device。

Command Index 和 Argument 为 Command 的具体内容，不同的 Command 有不同的 Index，不同的 Command 也有各自的 Argument。 更多的细节，请参考 [eMMC Commands](https://linux.codingbelief.com/zh/storage/flash_memory/emmc/emmc_commands.html) 章节。

CRC7 是包含 Start Bit、Transmission Bit、 Command Index 和 Argument 内容的 CRC 校验值。

End Bit 为结束标志位，固定为"1"。

> NOTE:  
> CRC 校验简单来说，是发送方将需要传输的数据“除于”（模2除）一个约定的数，并将得到的余数附在数据上一并发送出去。接收方收到数据后，再做同样的“除法”，然后校验得到余数是否与接收的余数相同。如果不相同，那么意味着数据在传输过程中发生了改变。更多的细节不在本文展开描述，感兴趣的读者可以参考 [CRC wiki](https://en.wikipedia.org/wiki/Cyclic_redundancy_check) 中的介绍。

#### 2.2.5 Response

![](https://linux.codingbelief.com/zh/storage/flash_memory/emmc/respone_token_format.png)

eMMC Response 有两种长度的数据包，分别为 48 Bits 和 136 Bits。

Start Bit 与 Command 一样，固定为 "0"，在没有数据传输的情况下，CMD 信号保持高电平，当 eMMC Device 将 Start Bit 发送到总线上时，Host 可以很方便检测到该信号，并开始接收 Response。

Transmission Bit 固定为 "0"，指示了该数据包的传输方向为 eMMC Device 发送到 Host。

Content 为 Response 的具体内容，不同的 Command 会有不同的 Content。 更多的细节，请参考 [eMMC Responses](https://linux.codingbelief.com/zh/storage/flash_memory/emmc/emmc_responses.html) 章节。

CRC7 是包含 Start Bit、Transmission Bit 和 Content 内容的 CRC 校验值。

End Bit 为结束标志位，固定为"1"。

#### 2.2.6 Data Block

Data Block 由 Start Bit、Data、CRC16 和 End Bit 组成。以下是不同总线宽度和 Data Rate 下，Data Block 详细格式。

1 Bit Bus SDR

![](https://linux.codingbelief.com/zh/storage/flash_memory/emmc/1_bit_bus_sdr.png)

CRC 为 Data 的 16 bit CRC 校验值，不包含 Start Bit。

4 Bits Bus SDR

![](https://linux.codingbelief.com/zh/storage/flash_memory/emmc/4_bits_bus_sdr.png)

各个 Data Line 上的 CRC 为对应 Data Line 的 Data 的 16 bit CRC 校验值。

8 Bits Bus SDR

![](https://linux.codingbelief.com/zh/storage/flash_memory/emmc/8_bits_bus_sdr.png)

各个 Data Line 上的 CRC 为对应 Data Line 的 Data 的16 bit CRC 校验值。

4 Bits Bus DDR

![](https://linux.codingbelief.com/zh/storage/flash_memory/emmc/4_bits_bus_ddr.png)

8 Bits Bus DDR

![](https://linux.codingbelief.com/zh/storage/flash_memory/emmc/8_bits_bus_ddr.png)

在 DDR 模式下，Data Line 在时钟的上升沿和下降沿都会传输数据，其中上升沿传输数据的奇数字节 （Byte 1,3,5 ...），下降沿则传输数据的偶数字节（Byte 2,4,6 ...）。  
此外，在 DDR 模式下，1 个 Data Line 上有两个相互交织的 CRC16，上升沿的 CRC 比特组成 odd CRC16，下降沿的 CRC 比特组成 even CRC16。odd CRC16 用于校验该 Data Line 上所有上升沿比特组成的数据，even CRC16 则用于校验该 Data Line 上所有下降沿比特组成的数据。

> NOTE:  
> DDR 模式下使用两个 CRC16 作为校验，可能是为了更可靠的校验，选用 CRC16 而非 CRC32 则可能是出于兼容性设计的考虑。

#### 2.2.7 CRC Status Token

在写数据传输中，eMMC Device 接收到 Host 发送的一个 Data Block 后，会进行 CRC 校验，如果校验成功，eMMC 会在对应的 Data Line 上向 Host 发回一个 Positive CRC status token (010)，如果校验失败，则会在对应的 Data Line 上发送一个 Negative CRC status token (101)。

> NOTE:  
> 读数据时，Host 接收到 eMMC Device 发送的 Data Block 后，也会进行 CRC 校验，但是不管校验成功或者失败，都不会向 eMMC Device 发送 CRC Status Token。

详细格式如下图所示：

Positive CRC status token

![](https://linux.codingbelief.com/zh/storage/flash_memory/emmc/positive_crc_status_token.png)

Negative CRC status token

![](https://linux.codingbelief.com/zh/storage/flash_memory/emmc/negative_crc_status_token.png)

## 3. eMMC 总线测试过程

当 eMMC Device 处于 SDR 模式时，Host 可以发送 CMD19 命令，触发总线测试过程（Bus testing procedure），测试总线硬件上的连通性。如果 eMMC Device 支持总线测试，那么 eMMC Device 在接收到 CMD19 后，会发回对应的 Response，接着 eMMC Device 会发送一组固定的测试数据给 Host。Host 接收到数据后，检查数据正确与否，即可得知总线是否正确连通。

> NOTE: 如果 eMMC Device 不支持总线测试，那么接收到 CMD19 时，不会发回 Response。  
> 总线测试不支持在 DDR 模式下进行。

测试数据如下所示：

![](https://linux.codingbelief.com/zh/storage/flash_memory/emmc/bus_testing_data.png)

> NOTE: 总线宽度为 1 时，只发送 DAT0 上的数据，总线宽度为 4 时，则只发送 DAT0-3 上的数据

## 4. eMMC 总线 Sampling Tuning

由于芯片制造工艺、PCB 走线、电压、温度等因素的影响，数据信号从 eMMC Device 到达 Host 端的时间是存在差异的，Host 接收数据时采样的时间点也需要相应的进行调整。而 Host 端最佳采样时间点，则是通过 Sampling Tuning 流程得到。

> NOTE:  
> 不同 eMMC Device 最佳的采样点可能不同，同一 eMMC Device 在不同的环境下运作时的最佳采样点也可能不同。  
> 在 eMMC 标准中，定义了在 HS200 模式下可以进行 Sampling Tuning。

### 4.1 Sampling Tuning 流程

Sampling Tuning 是用于计算 Host 最佳采样时间点的流程，大致的流程如下：

1. Host 将采样时间点重置为默认值
2. Host 向 eMMC Device 发送 Send Tuning Block 命令
3. eMMC Device 向 Host 发送固定的 Tuning Block 数据
4. Host 接收到 Tuning Block 并进行校验
5. Host 修改采样时点，重新从第 2 步开始执行，直到 Host 获取到一个有效采样时间点区间
6. Host 取有效采样时间点区间的中间值作为采样时间点，并推出 Tuning 流程

> NOTE:  
> 上述流程仅仅是一个示例。Tuning 流程执行的时机、频率和具体的步骤是由 Host 端的 eMMC Controller 具体实现而定的。

### 4.2 Tuning Block 数据

Tuning Block 是专门为了 Tuning 而设计的一组特殊数据。相对于普通的数据，这组特殊数据在传输过程中，会更高概率的出现 high SSO noise、deterministic jitter、ISI、timing errors 等问题。这组数据的具体内容如下所示：

![](https://linux.codingbelief.com/zh/storage/flash_memory/emmc/tuning_block_pattern.png)

![](https://linux.codingbelief.com/zh/storage/flash_memory/emmc/tuning_block_on_data_lines.png)

> NOTE: 总线宽度为 1 时，只发送 DAT0 上的数据，总线宽度为 4 时，则只发送 DAT0-3 上的数据

## 5. 参考资料

1. [Embedded Multi-Media Card (e•MMC) Electrical Standard (5.1)](http://www.jedec.org/sites/default/files/docs/JESD84-B51.pdf) [PDF]
2. [SD/MMC Controller, Hard Processor System (HPS) Technical Reference Manual (TRM)](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.296.5338&rep=rep1&type=pdf) [PDF]
3. [CRC wiki](https://en.wikipedia.org/wiki/Cyclic_redundancy_check) [WEB]

_原创文章，转发请注明出处。蜗窝科技_  

  

标签: [emmc](http://www.wowotech.net/tag/emmc)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Linux MMC framework(2)_host controller driver](http://www.wowotech.net/comm/mmc_host_driver.html) | [Debian8 内核升级实验](http://www.wowotech.net/linux_application/debian8-upgrade-kernel.html)»

**评论：**

**YoO-OoY**  
2021-12-13 16:43

你好~关于emmc协议5.1我有一个地方不太理解想请教一下，协议描述修改emmc csd寄存器可以使用cmd27，但是cmd27是如何修改csd的，应该带什么参数发送cmd27呢？关于这个部分的一个流程

[回复](http://www.wowotech.net/basic_tech/emmc_bus_protocol.html#comment-8411)

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
    
    - [Linux设备模型(9)_device resource management](http://www.wowotech.net/device_model/device_resource_management.html)
    - [DRAM 原理 1 ：DRAM Storage Cell](http://www.wowotech.net/basic_tech/307.html)
    - [Linux cpuidle framework(3)_ARM64 generic CPU idle driver](http://www.wowotech.net/pm_subsystem/cpuidle_arm64.html)
    - [Device Tree（二）：基本概念](http://www.wowotech.net/device_model/dt_basic_concept.html)
    - [Linux设备模型(4)_sysfs](http://www.wowotech.net/device_model/dm_sysfs.html)
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
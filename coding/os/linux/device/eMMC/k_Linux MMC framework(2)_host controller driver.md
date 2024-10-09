作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2017-3-8 21:33 分类：[通信类协议](http://www.wowotech.net/sort/comm)

## 1. 前言

本文是Linux MMC framework的第二篇，将从驱动工程师的角度，介绍MMC host controller driver有关的知识，学习并掌握如何在MMC framework的框架下，编写MMC控制器的驱动程序。同时，通过本篇文章，我们会进一步的理解MMC、SD、SDIO等有关的基础知识。

## 2. MMC host驱动介绍

MMC的host driver，是用于驱动MMC host控制器的程序，位于“drivers/mmc/host”目录。从大的流程上看，编写一个这样的驱动非常简单，只需要三步：

> 1）调用mmc_alloc_host，分配一个struct mmc_host类型的变量，用于描述某一个具体的MMC host控制器。
>
> 2）根据MMC host控制器的硬件特性，填充struct mmc_host变量的各个字段，例如MMC类型、电压范围、操作函数集等等。
>
> 3）调用mmc_add_host接口，将正确填充的MMC host注册到MMC core中。

当然，看着简单，一牵涉到实现细节，还是很麻烦的，后面我们会慢慢分析。

注1：分析MMC host driver的时候，Linux kernel中有大把大把的例子（例如drivers/mmc/host/pxamci.c），大家可尽情参考、学习，不必谦虚（这是学习Linux的最佳方法）。

注2：由于MMC host driver牵涉到具体的硬件controller，分析的过程中需要一些具体的硬件辅助理解，本文将以“[X Project](http://www.wowotech.net/sort/x_project)”所使用Bubblegum-96平台为例，具体的硬件spec可参考\[1\]。

## 3. 主要数据结构

#### 3.1 struct mmc_host

MMC core使用struct mmc_host结构抽象具体的MMC host controller，该结构的定义位于“include/linux/mmc/host.h”中，它既可以用来描述MMC控制器所具有的特性、能力（host driver关心的内容），也保存了host driver运行过程中的一些状态、参数（MMC core关心的内容）。需要host driver关心的部分的具体的介绍如下：

> parent，一个struct device类型的指针，指向该MMC host的父设备，一般是注册该host的那个platform设备；
>
> class_dev，一个struct device类型的变量，是该MMC host在设备模型中作为一个“设备”的体现。当然，人如其名，该设备从属于某一个class（mmc_host_class）；
>
> ops，一个struct mmc_host_ops类型的指针，保存了该MMC host有关的操作函数集，具体可参考3.2小节的介绍；
>
> pwrseq，一个struct mmc_pwrseq类型的指针，保存了该MMC host电源管理有关的操作函数集，具体可参考3.2小节的介绍；
>
> f_min、f_max、f_init，该MMC host支持的时钟频率范围，最小频率、最大频率以及初始频率；
>
> ocr_avail，该MMC host可支持的操作电压范围（具体可参考include/linux/mmc/host.h中MMC_VDD_开头的定义）；\
> 注3：OCR（Operating Conditions Register）是MMC/SD/SDIO卡的一个32-bit的寄存器，其中有些bit指明了该卡的操作电压。MMC host在驱动这些卡的时候，需要和Host自身所支持的电压范围匹配之后，才能正常操作，这就是ocr_avail的存在意义。
>
> ocr_avail_sdio、ocr_avail_sd、ocr_avail_mmc，如果MMC host针对SDIO、SD、MMC等不同类型的卡，所支持的电压范围不同的话，需要通过这几个字段特别指定。否则，不需要赋值（初始化为0）；
>
> pm_notify，一个struct notifier_block类型的变量，用于支持power management有关的notify实现；
>
> max_current_330、max_current_300、max_current_180，当工作电压分别是3.3v、3v以及1.8v的时候，所支持的最大操作电流（如果MMC host没有特别的限制，可以不赋值）；
>
> caps、caps2，指示该MMC host所支持的功能特性，具体可参考3.4小节的介绍；
>
> pm_caps，mmc_pm_flag_t类型的变量，指示该MMC host所支持的电源管理特性；
>
> max_seg_size、max_segs、max_req_size、max_blk_size、max_blk_count、max_busy_timeout，和块设备（如MMC、SD、eMMC等）有关的参数，在古老的磁盘时代，这些参数比较重要。对基于MMC技术的块设备来说，硬件的性能大大提升，这些参数就没有太大的意义了。具体可参考5.2章节有关MMC数据传输的介绍；
>
> lock，一个spin lock，是MMC host driver的私有变量，可用于保护host driver的临界资源；
>
> ios，一个struct mmc_ios类型的变量，用于保存MMC bus的当前配置，具体可参考3.5小节的介绍；
>
> supply，一个struct mmc_supply类型的变量，用于描述MMC系统中的供电信息，具体可参考3.6小节的介绍；
>
> ……
>
> private，一个0长度的数组，可以在mmc_alloc_host时指定长度，由host controller driver自行支配。

#### 3.2 struct mmc_host_ops

struct mmc_host_ops抽象并集合了MMC host controller所有的操作函数集，包括：

1）数据传输有关的函数

```cpp
/*  
* It is optional for the host to implement pre_req and post_req in  
* order to support double buffering of requests (prepare one  
* request while another request is active).  
* pre_req() must always be followed by a post_req().  
* To undo a call made to pre_req(), call post_req() with  
* a nonzero err condition.  
*/  
void    (*post_req)(struct mmc_host *host, struct mmc_request *req,  
int err);  
void    (*pre_req)(struct mmc_host *host, struct mmc_request *req,  
bool is_first_req);  
void    (*request)(struct mmc_host *host, struct mmc_request *req);
```

pre_req和post_req是非必需的，host driver可以利用它们实现诸如双buffer之类的高级功能。

数据传输的主题是struct mmc_request类型的指针，具体可参考3.7小节的介绍。

2）总线参数的配置以及卡状态的获取函数

```cpp
/*  
* Avoid calling these three functions too often or in a "fast path",  
* since underlaying controller might implement them in an expensive  
* and/or slow way.  
*  
* Also note that these functions might sleep, so don't call them  
* in the atomic contexts!  
*  
* Return values for the get_ro callback should be:  
*   0 for a read/write card  
*   1 for a read-only card  
*   -ENOSYS when not supported (equal to NULL callback)  
*   or a negative errno value when something bad happened  
*  
* Return values for the get_cd callback should be:  
*   0 for a absent card  
*   1 for a present card  
*   -ENOSYS when not supported (equal to NULL callback)  
*   or a negative errno value when something bad happened  
*/  
void    (*set_ios)(struct mmc_host *host, struct mmc_ios *ios);  
int     (*get_ro)(struct mmc_host *host);  
int     (*get_cd)(struct mmc_host *host);
```

set_ios用于设置bus的参数（ios，可参考3.5小节的介绍）；get_ro可获取card的读写状态（具体可参考上面的注释）；get_cd用于检测卡的存在状态。

注4：注释中特别说明了，这几个函数可以sleep，耗时较长，没事别乱用。

3）其它一些非主流函数，都是optional的，用到的时候再去细看即可。

#### 3.3 struct mmc_pwrseq

MMC framework的power sequence是一个比较有意思的功能，它提供一个名称为struct mmc_pwrseq_ops的操作函数集，集合了power on、power off等操作函数，用于控制MMC系统的供电，如下：

```cpp
struct mmc_pwrseq_ops {  
void (pre_power_on)(struct mmc_host *host);  
void (*post_power_on)(struct mmc_host *host);  
void (*power_off)(struct mmc_host *host);  
void (*free)(struct mmc_host *host);  
};

struct mmc_pwrseq {  
const struct mmc_pwrseq_ops *ops;  
};
```

与此同时，MMC core提供了一个通用的pwrseq的管理模块（drivers/mmc/core/pwrseq.c），以及一些简单的pwrseq策略（如drivers/mmc/core/pwrseq_simple.c、drivers/mmc/core/pwrseq_emmc.c），最终的目的是，通过一些简单的dts配置，即可正确配置MMC的供电，例如：

```cpp
/* arch/arm/boot/dts/omap3-igep0020.dts */  
mmc2_pwrseq: mmc2_pwrseq {  
compatible = "mmc-pwrseq-simple";  
reset-gpios = <&gpio5 11 GPIO_ACTIVE_LOW>,      / gpio_139 - RESET_N_W /  
<&gpio5 10 GPIO_ACTIVE_LOW>;      / gpio_138 - WIFI_PDN /  
};

/ arch/arm/boot/dts/rk3288-veyron.dtsi  /  
emmc_pwrseq: emmc-pwrseq {  
compatible = "mmc-pwrseq-emmc";  
pinctrl-0 = <&emmc_reset>;  
pinctrl-names = "default";  
reset-gpios = <&gpio2 9 GPIO_ACTIVE_HIGH>;  
};
```

具体的细节，在需要的时候，阅读代码即可，这里不再赘述。

#### 3.4 Host capabilities

通过的caps和caps2两个字段，MMC host driver可以告诉MMC core控制器的一些特性、能力，包括（具体可参考include/linux/mmc/host.h中相关的宏定义，更为细致的使用场景和指南，需要结合实际的硬件，具体分析）：

> MMC_CAP_4_BIT_DATA，支持4-bit的总线传输；\
> MMC_CAP_MMC_HIGHSPEED，支持“高速MMC时序”；\
> MMC_CAP_SD_HIGHSPEED，支持“高速SD时序”；\
> MMC_CAP_SDIO_IRQ，可以产生SDIO有关的中断；\
> MMC_CAP_SPI，仅仅支持SPI协议（可参考“drivers/mmc/host/mmc_spi.c”中有关的实现）；\
> MMC_CAP_NEEDS_POLL，表明需要不停的查询卡的插入状态（如果需要支持热拔插的卡，则需要设置该feature）；\
> MMC_CAP_8_BIT_DATA，支持8-bit的总线传输；\
> MMC_CAP_AGGRESSIVE_PM，支持比较积极的电源管理策略（kernel的注释为“Suspend (e)MMC/SD at idle”）；\
> MMC_CAP_NONREMOVABLE，表明该MMC控制器所连接的卡是不可拆卸的，例如eMMC；\
> MMC_CAP_WAIT_WHILE_BUSY，表面Host controller在向卡发送命令时，如果卡处于busy状态，是否等待。如果不等待，MMC core在一些流程中（如查询busy状态），需要额外做一些处理；\
> MMC_CAP_ERASE，表明该MMC控制器允许执行擦除命令；\
> MMC_CAP_1_8V_DDR，支持工作电压为1.8v的DDR（Double Data Rate）mode\[3\]；\
> MMC_CAP_1_2V_DDR，支持工作电压为1.2v的DDR（Double Data Rate）mode\[3\]；\
> MMC_CAP_POWER_OFF_CARD，可以在启动之后，关闭卡的供电（一般只有在SDIO的应用场景中才可能用到，因为SDIO所连接的设备可能是一个独立的设备）；\
> MMC_CAP_BUS_WIDTH_TEST，支持通过CMD14和CMD19进行总线宽度的测试，以便选择一个合适的总线宽度进行通信；\
> MMC_CAP_UHS_SDR12、MMC_CAP_UHS_SDR25、MMC_CAP_UHS_SDR50、MMC_CAP_UHS_SDR104，它们是SD3.0的速率模式，分别表示支持25MHz、50MHz、100MHz和208MHz的SDR（Single Data Rate，相对\[3\]中的DDR）模式；\
> MMC_CAP_UHS_DDR50，它也是SD3.0的速率模式，表示支持50MHz的DDR（Double Data Rate\[3\]）模式；\
> MMC_CAP_DRIVER_TYPE_A、MMC_CAP_DRIVER_TYPE_C、MMC_CAP_DRIVER_TYPE_D，分别表示支持A/C/D类型的driver strength（驱动能力，具体可参考相应的spec）；\
> MMC_CAP_CMD23，表示该controller支持multiblock transfers（通过CMD23）；\
> MMC_CAP_HW_RESET，支持硬件reset；
>
> MMC_CAP2_FULL_PWR_CYCLE，表示该controller支持从power off到power on的完整的power cycle；\
> MMC_CAP2_HS200_1_8V_SDR、MMC_CAP2_HS200_1_2V_SDR，HS200是eMMC5.0支持的一种速率模式，200是200MHz的缩写，分别表示支持1.8v和1.2v的SDR模式；\
> MMC_CAP2_HS200，表示同时支持MMC_CAP2_HS200_1_8V_SDR和MMC_CAP2_HS200_1_2V_SDR；\
> MMC_CAP2_HC_ERASE_SZ，支持High-capacity erase size；\
> MMC_CAP2_CD_ACTIVE_HIGH，CD（Card-detect）信号高有效；\
> MMC_CAP2_RO_ACTIVE_HIGH，RO（Write-protect）信号高有效；\
> MMC_CAP2_PACKED_RD、MMC_CAP2_PACKED_WR，允许packed read、packed write；\
> MMC_CAP2_PACKED_CMD，同时支持DMMC_CAP2_PACKED_RD和MMC_CAP2_PACKED_WR；\
> MMC_CAP2_NO_PRESCAN_POWERUP，在scan之前不要上电；\
> MMC_CAP2_HS400_1_8V、MMC_CAP2_HS400_1_2V，HS400是eMMC5.0支持的一种速率模式，400是400MHz的缩写，分别表示支持1.8v和1.2v的HS400模式；\
> MMC_CAP2_HS400，同时支持MMC_CAP2_HS400_1_8V和MMC_CAP2_HS400_1_2V；\
> MMC_CAP2_HSX00_1_2V，同时支持MMC_CAP2_HS200_1_2V_SDR和MMC_CAP2_HS400_1_2V；\
> MMC_CAP2_SDIO_IRQ_NOTHREAD，SDIO的IRQ的处理函数，不能在线程里面执行；\
> MMC_CAP2_NO_WRITE_PROTECT，没有物理的RO管脚，意味着任何时候都是可以读写的；\
> MMC_CAP2_NO_SDIO，在初始化的时候，不会发送SDIO相关的命令（也就是说不支持SDIO模式）。

#### 3.5 struct mmc_ios

struct mmc_ios中保存了MMC总线当前的配置情况，包括如下信息：

1）clock，时钟频率。
2）vdd，卡的供电电压，通过“1 \<\< vdd”可以得到MMC_VDD_x_x（具体可参考include/linux/mmc/host.h中MMC_VDD_开头的定义），进而得到电压信息。
3）bus_mode，两种信号模式，open-drain（MMC_BUSMODE_OPENDRAIN）和push-pull（MMC_BUSMODE_PUSHPULL），对应不同的高低电平（可参考相应的spec，例如\[2\]）。
4）chip_select，只针对SPI模式，指定片选信号的有效模式，包括没有片选信号（MMC_CS_DONTCARE）、高电平有效（MMC_CS_HIGH）、低电平有效（MMC_CS_LOW）。
5）power_mode，当前的电源状态，包括MMC_POWER_OFF、MMC_POWER_UP、MMC_POWER_ON和MMC_POWER_UNDEFINED。
6）bus_width，总线的宽度，包括1-bit（MMC_BUS_WIDTH_1）、4-bit（MMC_BUS_WIDTH_4）和8-bit（MMC_BUS_WIDTH_8）。
7）timing，符合哪一种总线时序（大多对应某一类MMC规范），包括：

> MMC_TIMING_LEGACY，旧的、不再使用的规范；\
> MMC_TIMING_MMC_HS，High speed MMC规范（具体可参考相应的spec，这里不再详细介绍，下同）；\
> MMC_TIMING_SD_HS，High speed SD；\
> MMC_TIMING_UHS_SDR12；\
> MMC_TIMING_UHS_SDR25\
> MMC_TIMING_UHS_SDR50\
> MMC_TIMING_UHS_SDR104\
> MMC_TIMING_UHS_DDR50\
> MMC_TIMING_MMC_DDR52\
> MMC_TIMING_MMC_HS200\
> MMC_TIMING_MMC_HS400

8）signal_voltage，总线信号使用哪一种电压，3.3v（MMC_SIGNAL_VOLTAGE_330）、1.8v（MMC_SIGNAL_VOLTAGE_180）或者1.2v（MMC_SIGNAL_VOLTAGE_120）。

9）drv_type，驱动能力，包括：

> MMC_SET_DRIVER_TYPE_B\
> MMC_SET_DRIVER_TYPE_A\
> MMC_SET_DRIVER_TYPE_C\
> MMC_SET_DRIVER_TYPE_D

#### 3.6 struct mmc_supply

struct mmc_supply中保存了两个struct regulator指针（如下），用于控制MMC子系统有关的供电（vmmc和vqmmc）。

```cpp
struct mmc_supply {  
struct regulator vmmc;         / Card power supply /  
struct regulator *vqmmc;        / Optional Vccq supply /  
};
```

关于vmmc和vqmmc，说明如下：

> vmmc是卡的供电电压，一般连接到卡的VDD管脚上。而vqmmc则用于上拉信号线（CMD、CLK和DATA\[6\]）。
>
> 通常情况下vqmmc使用和vmmc相同的regulator，同时供电即可。
>
> 后来，一些高速卡（例如UHS SD）要求在高速模式下，vmmc为3.3v，vqmmc为1.8v，这就需要两个不同的regulator独立控制。

#### 3.7 struct mmc_request

struct mmc_request封装了一次传输请求，定义如下：

```cpp
/* include/linux/mmc/core.h */

struct mmc_request {  
struct mmc_command      *sbc;           / SET_BLOCK_COUNT for multiblock /  
struct mmc_command      *cmd;  
struct mmc_data         *data;  
struct mmc_command      *stop;

struct completion       completion;  
void                    (*done)(struct mmc_request *);/ completion function /  
struct mmc_host         *host;  
};
```

要理解这个数据结构，需要先了解MMC的总线协议（bus protocol），这里以eMMC\[2\]为例进行简单的介绍（更为详细的解释，可参考相应的spec以及本站的文章--“[eMMC 原理 4 ：总线协议](http://www.wowotech.net/basic_tech/emmc_bus_protocol.html)\[7\]”）。

**3.7.1 MMC bus protocol**

在eMMC的spec中，称总线协议为“message-based MultiMediaCard bus protocol”，这里的message由三种信标（token）组成：

> Command，用于启动（或者停止）一次传输，由Host通过CMD line向Card发送；
>
> Response，用于应答上一次的Command，由Card通过CMD line想Host发送；
>
> Data，传输数据，由Host（或者Card）通过DATA lines向Card（或者Host发送）。

以上token除了Command之外，剩余的两个（Response和Data）都是非必需的，也就是说，一次传输可以是：不需要应答、不需要数据传输的Command；需要应答、不需要数据传输的Command；不需要应答、需要数据传输的Command；不需要应答、不需要数据传输的Command。

Command token的格式只有一种（具体可参考\[2\]中“Command token format”有关的表述），长度为48bits，包括Start bit(0)、Transmitter bit(1, host command)、Content(38bits)、CRC checksum(7bits)、Stop bit(1)。

根据内容的不同，Response token的格式有5中，分别简称为R1/R3/R4/R5/R2，其中R1/R3/R4/R5的长度为48bits，R2为136bits（具体可参考\[2\]中“Response token format”有关的表述）。

对于包含了Data token的Command，有两种类型：

> Sequential commands，发送Start command之后，数据以stream的形式传输，直到Stop command为止。这种方式只支持1-bit总线模式，主要为了兼容旧的技术，一般不使用；
>
> Block-oriented commands，发送Start command之后，数据以block的形式传输（每个block的大小是固定的，且都由CRC保护）。

最后，以block为单位的传输，大体上也分为两类：

> 在传输开始的时候（Start command），没有指定需要传输的block数目，直到发送Stop command为止。这种方法在spec中称作“Open-ended”；
>
> 在传输开始的时候（Start command），指定需要传输的block数据，当达到数据之后，Card会自动停止传输，这样可以省略Stop command。这种方法在spec中称作pre-defined block count。

**3.7.2 struct mmc_request**

了解MMC bus protocol之后，再来看一次MMC传输请求（struct mmc_request ）所包含的内容：

> cmd，Start command，为struct mmc_command类型（具体请参考3.7.3中的介绍）的指针，在一次传输的过程中是必须的；
>
> data，传输的数据，为struct mmc_data类型（具体请参考3.7.4中的介绍）的指针，不是必须要的；
>
> stop、sbc，如果需要进行数据传输，根据数据传输的方式（参考3.7.1中的介绍）：如果是“Open-ended”，则需要stop命令（stop指针，或者data->stop指针）；如果是pre-defined block count，则需要sbc指针（用于发送SET_BLOCK_COUNT--CMD23命令）；
>
> completion，一个struct completion变量，用于等待此次传输完成，host controller driver可以根据需要使用；
>
> done，传输完成时的回调，用于通知传输请求的发起者；
>
> host，对应的mmc host controller指针。

**3.7.3 struct mmc_command**

struct mmc_command结构抽象了一个MMC command，包括如下内容：

> /\* include/linux/mmc/core.h \*/
>
> opcode，Command的操作码，用于标识该命令是哪一个命令，具体可参考相应的spec（例如\[2\]）；
>
> arg，一个Command可能会携带参数，具体可参考相应的spec（例如\[2\]）；
>
> resp\[4\]，Command发出后，如果需要应答，结果保存在resp数组中，该数组是32-bit的，因此最多可以保存128bits的应答；
>
> flags，是一个bitmap，保存该命令所期望的应答类型，例如：\
> MMC_RSP_PRESENT（1 \<\< 0)，是否需要应答，如果该bit为0，则表示该命令不需要应答，否则，需要应答；\
> MMC_RSP_136（1 \<\< 1），如果为1，表示需要136bits的应答；\
> MMC_RSP_CRC（1 \<\< 2），如果为1，表示需要对该命令进行CRC校验；\
> 等等，具体可参考include/linux/mmc/core.h中“MMC_RSP\_”开头的定义；
>
> retries，如果命令发送出错，可以重新发送，该字段向host driver指明最多可重发的次数；
>
> error，如果最终还是出错，host driver需要通过该字段返回错误的原因，kernel定义了一些标准错误，例如ETIMEDOUT、EILSEQ、EINVAL、ENOMEDIUM等，具体含义可参考include/linux/mmc/core.h中注释；
>
> busy_timeout，如果card具有busy检测的功能，该字段指定等待card返回busy状态的超时时间，单位为ms；
>
> data，和该命令对应的struct mmc_data指针；
>
> mrq，和该命令对应的struct mmc_request指针。

**3.7.4 struct mmc_data**

struct mmc_data结构包含了数据传输有关的内容：

> /\* include/linux/mmc/core.h \*/
>
> timeout_ns、timeout_clks，这一笔数据传输的超时时间（单位分别为ns和clks），如果超过这个时间host driver还无法成功发送，则要将状态返回给mmc core；
>
> blksz、blocks，该笔数据包含多少block（blocks），每个block的size多大（blksz），这两个值不会大于struct mmc_host中上报的max_blk_size和max_blk_count；
>
> error，如果数据传输出错，错误值保存在该字段，具体意义和struct mmc_command中的一致；
>
> flags，一个bitmap，指明该笔传说的方向（MMC_DATA_WRITE或者MMC_DATA_READ）；
>
> sg，一个struct scatterlist类型的数组，保存了需要传输的数据（可以通过dma_相关的接口，获得相应的物理地址）；\
> sg_len，sg数组的size；\
> sg_count，通过sg map出来的实际的entry的个数（可能由于物理地址的连续、IOMMU的干涉等，map出来的entry的个数，可能会小于sg的size）；\
> 注5：有关scatterlist的介绍，可参考本站另外的文章（TODO）。有关struct mmc_data的使用场景，可参考5.2小节的介绍；
>
> host_cookie，host driver的私有数据，怎么用由host driver自行决定。

## 4. 主要API

第3章花了很大的篇幅介绍了用于抽象MMC host的数据结构----struct mmc_host，并详细说明了和mmc_host相关的mmc request、mmc command、mmc data等结构。基于这些知识，本章将介绍MMC core提供的和struct mmc_host有关的操作函数，主要包括如下几类。

#### 4.1 向MMC host controller driver提供的用于操作struct mmc_host的API

包括：

|   |
|---|
|struct mmc_host \*mmc_alloc_host(int extra, struct device \*);  <br>int mmc_add_host(struct mmc_host \*);  <br>void mmc_remove_host(struct mmc_host \*);  <br>void mmc_free_host(struct mmc_host \*);  <br>int mmc_of_parse(struct mmc_host \*host);  <br>static inline void \*mmc_priv(struct mmc_host \*host) {  <br>        return (void \*)host->private;  <br>}|

> mmc_alloc_host，动态分配一个struct mmc_host变量。extra是私有数据的大小，可通过host->private指针访问（也可通过mmc_priv接口直接获取）。mmc_free_host执行相反动作。
>
> mmc_add_host，将已初始化好的host变量注册到kernel中。mmc_remove_host执行相反动作。
>
> 为了方便，host controller driver可以在dts中定义host的各种特性，然后在代码中调用mmc_of_parse解析并填充到struct mmc_host变量中。dts属性关键字可参考mmc_of_parse的source code（drivers/mmc/core/host.c），并结合第三章的内容自行理解。

|   |
|---|
|int mmc_power_save_host(struct mmc_host \*host);   <br>int mmc_power_restore_host(struct mmc_host \*host);|

> 从mmc host的角度进行电源管理，进入/退出power save状态。

|   |
|---|
|void mmc_detect_change(struct mmc_host \*, unsigned long delay);|

> 当host driver检测到总线上的设备有变动的话（例如卡的插入和拔出等），需要调用这个接口，让MMC core帮忙做后续的工作，例如检测新插入的卡到底是个什么东东……
>
> 另外，可以通过delay参数告诉MMC core延时多久（单位为jiffies）开始处理，通常可以用来对卡的拔插进行去抖动。

|   |
|---|
|void mmc_request_done(struct mmc_host \*, struct mmc_request \*);|

> 当host driver处理完成一个mmc request之后,需要调用该函数通知MMC core，MMC core会进行一些善后的操作，例如校验结果、调用mmc request的.done回调等等。

|   |
|---|
|static inline void mmc_signal_sdio_irq(struct mmc_host \*host)  <br>void sdio_run_irqs(struct mmc_host \*host);|

> 对于SDIO类型的总线，这两个函数用于操作SDIO irqs，后面用到的时候再分析。

|   |
|---|
|int mmc_regulator_get_ocrmask(struct regulator \*supply);  <br>int mmc_regulator_set_ocr(struct mmc_host \*mmc,  <br>                        struct regulator \*supply,  <br>                        unsigned short vdd_bit);  <br>int mmc_regulator_set_vqmmc(struct mmc_host \*mmc, struct mmc_ios \*ios);  <br>int mmc_regulator_get_supply(struct mmc_host \*mmc);|

> regulator有关的辅助函数：
>
> mmc_regulator_get_ocrmask可根据传入的regulator指针，获取该regulator支持的所有电压值，并以此推导出对应的ocr mask（可参考3.1中的介绍）。
>
> mmc_regulator_set_ocr用于设置host controller为某一个操作电压（vdd_bit），该接口会调用regulator framework的API，进行具体的电压切换。
>
> mmc_regulator_set_vqmmc可根据struct mmc_ios信息，自行调用regulator framework的接口，设置vqmmc的电压。
>
> 最后，mmc_regulator_get_supply可以帮忙从dts的vmmc、vqmmc属性值中，解析出对应的regulator指针，以便后面使用。

#### 4.2 用于判断MMC host controller所具备的能力的API

比较简单，可结合第3章的介绍理解：

|   |
|---|
|#define mmc_host_is_spi(host)   ((host)->caps & MMC_CAP_SPI)  <br>static inline int mmc_card_is_removable(struct mmc_host \*host)  <br>static inline int mmc_card_keep_power(struct mmc_host \*host)  <br>static inline int mmc_card_wake_sdio_irq(struct mmc_host \*host)  <br>static inline int mmc_host_cmd23(struct mmc_host \*host)  <br>static inline int mmc_boot_partition_access(struct mmc_host \*host)  <br>static inline int mmc_host_uhs(struct mmc_host \*host)  <br>static inline int mmc_host_packed_wr(struct mmc_host \*host)  <br>static inline int mmc_card_hs(struct mmc_card \*card)  <br>static inline int mmc_card_uhs(struct mmc_card \*card)  <br>static inline bool mmc_card_hs200(struct mmc_card \*card)  <br>static inline bool mmc_card_ddr52(struct mmc_card \*card)  <br>static inline bool mmc_card_hs400(struct mmc_card \*card)  <br>static inline void mmc_retune_needed(struct mmc_host \*host)  <br>static inline void mmc_retune_recheck(struct mmc_host \*host)|

## 5. MMC host驱动的编写步骤

经过上面章节的描述，相信大家对MMC controller driver有了比较深的理解，接下来驱动的编写就是一件水到渠成的事情了。这里简要描述一下驱动编写步骤，也顺便为本文做一个总结。

#### 5.1 struct mmc_host的填充和注册

编写MMC host驱动的所有工作，都是围绕struct mmc_host结构展开的。在对应的platform driver的probe函数中，通过mmc_alloc_host分配一个mmc host后，我们需要根据controller的实际情况，填充对应的字段。

mmc host中大部分和controller能力/特性有关的字段，可以通过dts配置（然后在代码中调用mmc_of_parse自动解析并填充），举例如下（注意其中红色的部分，都是MMC framework的标准字段）：

```cpp
/* arch/arm/boot/dts/exynos5420-peach-pit.dts */

&mmc_1 {  
status = "okay";  
num-slots = <1>;  
non-removable;  
cap-sdio-irq;  
keep-power-in-suspend;  
clock-frequency = <400000000>;  
samsung,dw-mshc-ciu-div = <1>;  
samsung,dw-mshc-sdr-timing = <0 1>;  
samsung,dw-mshc-ddr-timing = <0 2>;  
pinctrl-names = "default";  
pinctrl-0 = <&sd1_clk>, <&sd1_cmd>, <&sd1_int>, <&sd1_bus1>,  
<&sd1_bus4>, <&sd1_bus8>, <&wifi_en>;  
bus-width = <4>;  
cap-sd-highspeed;  
mmc-pwrseq = <&mmc1_pwrseq>;  
vqmmc-supply = <&buck10_reg>;  
};
```

#### 5.2 数据传输的实现

填充struct mmc_host变量的过程中，工作量最大的，就是对struct mmc_host_ops的实现（毫无疑问！所有MMC host的操作逻辑都封在这里呢！！）。这里简单介绍一下相关的概念，具体的驱动编写步骤，后面文章会结合“[X Project](http://www.wowotech.net/sort/x_project)”详细描述。

**5.2.1 Sectors（扇区）、Blocks（块）以及Segments（段）的理解**

我们在3.1小节介绍struct mmc_host的时候，提到了max_seg_size、max_segs、max_req_size、max_blk_size、max_blk_count等参数。这和磁盘设备（块设备）中Sectors、Blocks、Segments等概念有关，下面简单介绍一下：

1）Sectors

> Sectors是存储设备访问的基本单位。
>
> 对磁盘、NAND等块设备来说，Sector的size是固定的，例如512、2048等。
>
> 对存储类的MMC设备来说，按理说也应有固定size的sector。但因为有MMC协议的封装，host驱动以及上面的块设备驱动，不需要关注物理的size。它们需要关注的就是bus上的数据传输单位（具体可参考MMC protocol的介绍\[7\]）。
>
> 最后，对那些非存储类的MMC设备来说，完全没有sector的概念了。

2） Blocks

> Blocks是数据传输的基本单位，是VFS（虚拟文件系统）抽象出来的概念，是纯软件的概念，和硬件无关。它必须是2的整数倍、不能大于Sectors的单位、不能大于page的长度，一般为512B、2048B或者4096B。
>
> 对MMC设备来说，由于协议的封装，淡化了Sector的概念，或者说，MMC设备可以支持一定范围内的任意的Block size。Block size的范围，由两个因素决定：\
> a）host controller的能力，这反映在struct mmc_host结构的max_blk_size字段上。\
> b）卡的能力，这可以通过MMC command从卡的CSD（Card-Specific Data）寄存器中读出。

3）Segments\[8\]

> 块设备的数据传输，本质上是设备上相邻扇区与内存之间的数据传输。通常情况下，为了提升性能，数据传输通过DMA方式。
>
> 在磁盘控制器的旧时代，DMA操作都比较简单，每次传输，数据在内存中必须是连续的。现在则不同，很多SOC都支持“分散/聚合”（scatter-gather）DMA操作，这种操作模式下，数据传输可以在多个非连续的内存区域中进行。
>
> 对于每个“分散/聚合”DMA操作，块设备驱动需要向控制器发送：\
> a）初始扇区号和传输的总共扇区数\
> b）内存区域的描述链表，每个描述都包含一个地址和长度。不同的描述之间，可以在物理上连续，也可以不连续。
>
> 控制器来管理整个数据传输，例如：在读操作中，控制器从块设备相邻的扇区上读取数据，然后将数据分散存储在内存的不同区域。
>
> 这里的每个内存区域描述（物理连续的一段内存，可以是一个page，也可以是page的一部分），就称作Segment。一个Segment包含多个相邻扇区。
>
> 最后，利用“分散/聚合”的DMA操作，一次数据传输可以会涉及多个segments。

理解了Segment的概念之后，max_seg_size和max_segs两个字段就好理解了：

> 虽然控制器支持“分散/聚合”的DMA操作，但物理硬件总有限制，例如最大的Segment size（也即一个内存描述的最大长度），最多支持的segment个数（max_segs）等。

**5.2.2 struct mmc_data中的sg**

我们在3.7.4小节介绍struct mmc_data时，提到了scatterlist的概念。结合上面Segment的解释，就很好理解了：

> MMC core提交给MMC host driver的数据传输请求，是一个struct scatterlist链表（也即内存区域的描述链表），也可以理解为是一个个的Segment（Segment的个数保存在sg_len变量中了）。
>
> 每个Segment是一段物理地址连续的内存区域，所有的Segments对应了MMC设备中连续的Sector（或者说Block，初始扇区号和传输的总共扇区数已经在之前的MMC command中指定了。
>
> host driver在接收到这样的数据传输请求之后，需要调用dma_map_sg将这些Segment映射出来（获得相应的物理地址），以便交给MMC controller传输。
>
> 当然，相邻两个Segment的物理地址可能是连续（或者其它原因），map的时候可能会将两个Segment合成一个。因此可供MMC controller传输的内存片可能少于sg_len（具体要看dma_map_sg的返回值，可将结果保存在sg_count中）。
>
> 最后，如何实施传输，则要看具体的MMC controller的硬件实现（可能涉及DMA操作），后面文章再详细介绍。

## 6. 参考文档

\[1\] [SoC_bubblegum96.pdf](https://github.com/96boards/documentation/blob/master/ConsumerEdition/Bubblegum-96/HardwareDocs/Schematics_Bubblegum96.pdf)

\[2\] [JESD84-A44.pdf](http://rere.qmqm.pl/~mirq/JESD84-A44.pdf)

\[3\] DDR mode, [https://en.wikipedia.org/wiki/Double_data_rate](https://en.wikipedia.org/wiki/Double_data_rate "https://en.wikipedia.org/wiki/Double_data_rate")

\[4\] [http://www.hjreggel.net/cardspeed/cs_sdxc.html](http://www.hjreggel.net/cardspeed/cs_sdxc.html "http://www.hjreggel.net/cardspeed/cs_sdxc.html")

\[5\] regulator framework，[http://www.wowotech.net/tag/regulator](http://www.wowotech.net/tag/regulator "http://www.wowotech.net/tag/regulator")

\[6\] [MMC/SD/SDIO介绍](http://www.wowotech.net/basic_tech/mmc_sd_sdio_intro.html)

\[7\] [eMMC 原理 4 ：总线协议](http://www.wowotech.net/basic_tech/emmc_bus_protocol.html)

\[8\] [http://www.ilinuxkernel.com/files/Linux.Generic.Block.Layer.pdf](http://www.ilinuxkernel.com/files/Linux.Generic.Block.Layer.pdf "http://www.ilinuxkernel.com/files/Linux.Generic.Block.Layer.pdf")

_原创文章，转发请注明出处。蜗窝科技_，[www.wowotech.net](http://www.wowotech.net/comm/mmc_host_driver.html)。

标签: [Linux](http://www.wowotech.net/tag/Linux) [Kernel](http://www.wowotech.net/tag/Kernel) [内核](http://www.wowotech.net/tag/%E5%86%85%E6%A0%B8) [driver](http://www.wowotech.net/tag/driver) [mmc](http://www.wowotech.net/tag/mmc) [host](http://www.wowotech.net/tag/host)

______________________________________________________________________

« [Linux调度器：用户空间接口](http://www.wowotech.net/process_management/scheduler-API.html) | [eMMC 原理 4 ：总线协议](http://www.wowotech.net/basic_tech/emmc_bus_protocol.html)»

**评论：**

**草京虫**\
2019-01-09 16:51

因该是在sdhci_s3c_probe->sdhci_add_host->mmc_start->\_mmc_detect_change中触发检测

[回复](http://www.wowotech.net/comm/mmc_host_driver.html#comment-7127)

**jasonactions**\
2018-11-14 16:36

这里的每个内存区域描述（物理连续的一段内存，可以是一个page，也可以是page的一部分），就称作Segment。一个Segment包含多个相邻扇区。\
这里 应该说一个Segment对应多个相邻扇区比较好吧？因为Segment是内存缓冲区，对应的是bio_vec

[回复](http://www.wowotech.net/comm/mmc_host_driver.html#comment-7036)

**rega**\
2017-11-20 14:09

hi wowo MMC部分还会继续写完吗？

[回复](http://www.wowotech.net/comm/mmc_host_driver.html#comment-6222)

**[wowo](http://www.wowotech.net/)**\
2017-11-20 17:13

@rega：有时间的话应该会，不过时间太少了，估计写的比较慢:-(

[回复](http://www.wowotech.net/comm/mmc_host_driver.html#comment-6226)

**rega**\
2017-11-23 15:12

@wowo：期待wowo能完成这部分。。。

[回复](http://www.wowotech.net/comm/mmc_host_driver.html#comment-6234)

**[江南书生](http://no/)**\
2017-05-23 11:33

老板，你好！\
SD卡上面的CD引脚，如果悬空（开机前已经把卡插上去了）的话，SD卡的驱动就无法识别到SD卡设备的存在，请问这是为什么？CD引脚必须要使用么？

[回复](http://www.wowotech.net/comm/mmc_host_driver.html#comment-5592)

**[wowo](http://www.wowotech.net/)**\
2017-05-23 16:57

@江南书生：你是指要设置为不可拔插的卡吗？是的话，看一下kernel拔插检测的流程吧，试着设置一下MMC_CAP_NONREMOVABLE属性。

[回复](http://www.wowotech.net/comm/mmc_host_driver.html#comment-5594)

**[江南书生](http://no/)**\
2017-05-23 16:59

@wowo：老板，你好！

热插拔会引起中断，但是我如果直接把卡插上去（开机前），为啥也能检测的卡，这是什么原理啊？

谢谢，老板

[回复](http://www.wowotech.net/comm/mmc_host_driver.html#comment-5596)

**[wowo](http://www.wowotech.net/)**\
2017-05-23 19:01

@江南书生：照你的描述，mmc子系统初始化的时候，会去查询一次（这就是为什么插着卡开机可以检测到），后面就不再查询了。

[回复](http://www.wowotech.net/comm/mmc_host_driver.html#comment-5597)

**秋暮离**\
2017-06-13 09:08

@江南书生：按wowo所说，s3c平台在MMC子系统初始化阶段会去查询一次卡的有无：sdhci_s3c_probe->sdhci_add_host->sdhci_enable_card_detection->sdhci_set_card_detection，以后的卡插拔事件由中断负责检测

[回复](http://www.wowotech.net/comm/mmc_host_driver.html#comment-5665)

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

  - [小printf大作用（用日志打印的方式调试程序）](http://www.wowotech.net/soft/7.html)
  - [Linux TTY framework(5)\_System console driver](http://www.wowotech.net/tty_framework/system_console_driver.html)
  - [process identification](http://www.wowotech.net/process_management/process_identification.html)
  - [futex基础问答](http://www.wowotech.net/kernel_synchronization/futex.html)
  - [进化论、人工智能和外星人](http://www.wowotech.net/tech_discuss/toe_ai_et.html)

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

人人极客社区
_2022年04月06日 08:30_

- eMMC 简介
- Host Interface
- Flash Controller
- Flash Memory
- eMMC 分区管理
- Boot Area Partitions
- eMMC 分区应用实例
- eMMC 总线协议
- eMMC 总线接口
- eMMC 总线模型

## eMMC 简介

eMMC 是 embedded MultiMediaCard 的简称。MultiMediaCard，即MMC， 是一种闪存卡（Flash Memory Card）标准，它定义了 MMC 的架构以及访问　Flash Memory 的接口和协议。而eMMC 则是对 MMC 的一个拓展，以满足更高标准的性能、成本、体积、稳定、易用等的需求。

eMMC 的整体架构如下图片所示：

![图片](https://mmbiz.qpic.cn/mmbiz_png/9sNwsXcN68plCVxp0t5k2HBoMWqtrSYqic03UMW8r40K42Nib6kD5icUAbIuwbZ9vj9WZDwazTBHdicXGu4fHYdoAA/640?wx_fmt=png&wxfrom=13&tp=wxpic)

eMMC 内部主要可以分为 Flash Memory、Flash Controller 以及Host Interface 三大部分。

### Host Interface

eMMC 与 Host 之间的连接如下图所示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

各个信号的用途如下所示：

- CLK: 用于同步的时钟信号

- Data Strobe：此信号是从 Device 端输出的时钟信号，频率和 CLK 信号相同，用于同步从 Device 端输出的数据。该信号在 eMMC 5.0 中引入。

- CMD：此信号用于发送 Host 的 command 和 Device 的 response。

- DAT0-7：用于传输数据的 8 bit 总线。

Host 与 eMMC 之间的通信都是 Host 以一个 Command 开始发起的。针对不同的 Command，Device 会做出不同的响应。详细的通信协议相关内容，可以参考eMMC总线协议章节。

### Flash Controller

NAND Flash 直接接入 Host 时，Host 端通常需要有 NAND Flash Translation Layer，即 NFTL 或者 NAND Flash 文件系统来做坏块管理、ECC等的功能。

eMMC 则在其内部集成了 Flash Controller，用于完成擦写均衡、坏块管理、ECC校验等功能。相比于直接将 NAND Flash 接入到 Host 端，eMMC 屏蔽了 NAND Flash 的物理特性，可以减少 Host 端软件的复杂度，让 Host 端专注于上层业务，省去对 NAND Flash 进行特殊的处理。同时，eMMC 通过使用 Cache、Memory Array 等技术，在读写性能上也比 NAND Flash 要好很多。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### Flash Memory

Flash Memory 是一种非易失性的存储器，通常在嵌入式系统中用于存放系统、应用和数据等，类似于 PC 系统中的硬盘。目前，绝大部分手机和平板等移动设备中所使用的 eMMC 内部的 Flash Memory 都属于 NAND Flash。

eMMC 在内部对 Flash Memory 划分了几个主要区域，如下图所示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

1. BOOT Area Partition 1 & 2

此分区主要是为了支持从 eMMC 启动系统而设计的。该分区的数据，在 eMMC 上电后，可以通过很简单的协议就可以读取出来。同时，大部分的 SOC 都可以通过 GPIO 或者 FUSE 的配置，让 ROM 代码在上电后，将 eMMC BOOT 分区的内容加载到 SOC 内部的 SRAM 中执行。

2. RPMB Partition

RPMB 是 Replay Protected Memory Block的简称，它通过 HMAC SHA-256 和 Write Counter 来保证保存在 RPMB 内部的数据不被非法篡改。在实际应用中，RPMB 分区通常用来保存安全相关的数据，例如指纹数据、安全支付相关的密钥等。

3. General Purpose Partition 1～4

此区域则主要用于存储系统或者用户数据。General Purpose Partition 在芯片出厂时，通常是不存在的，需要主动进行配置后，才会存在。

4. User Data Area

此区域则主要用于存储系统和用户数据。User Data Area 通常会进行再分区，例如 Android 系统中，通常在此区域分出 boot、system、userdata 等分区。

## eMMC 分区管理

### Boot Area Partitions

Boot Area 包含两个 Boot Area Partitions，主要用于存储 Bootloader，支持 SOC 从 eMMC 启动系统。

**1. 容量大小**

两个 Boot Area Partitions 的大小是完全一致的，由 Extended CSD register 的 BOOT_SIZE_MULT Field 决定，大小的计算公式如下：

> Size = 128Kbytes x BOOT_SIZE_MULT

一般情况下，Boot Area Partition 的大小都为 4 MB，即 BOOT_SIZE_MULT 为 32，部分芯片厂家会提供改写 BOOT_SIZE_MULT 的功能来改变 Boot Area Partition 的容量大小。BOOT_SIZE_MULT 最大可以为 255，即 Boot Area Partition 的最大容量大小可以为 255 x 128 KB = 32640 KB = 31.875 MB。

**2. 从 Boot Area 启动**

eMMC 中定义了 Boot State，在 Power-up、HW reset 或者 SW reset 后，如果满足一定的条件，eMMC 就会进入该 State。进入 Boot State 的条件如下：

- Original Boot Operation

CMD 信号保持低电平不少于 74 个时钟周期，会触发 Original Boot Operation，进入 Boot State。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- Alternative Boot Operation

在 74 个时钟周期后，在 CMD 信号首次拉低或者 Host 发送 CMD1 之前，Host 发送参数为 0xFFFFFFFA 的 COM0时，会触发 Alternative Boot Operation，进入 Boot State。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在 Boot State 下，如果有配置 BOOT_ACK，eMMC 会先发送 “010” 的 ACK 包，接着 eMMC 会将最大为 128Kbytes x BOOT_SIZE_MULT 的 Boot Data 发送给 Host。传输过程中，Host 可以通过拉高 CMD 信号 (Original Boot 中)，或者发送 Reset 命令 (Alternative Boot 中) 来中断 eMMC 的数据发送，完成 Boot Data 传输。

Boot Data 根据 Extended CSD register 的 PARTITION_CONFIG Field 的 Bit\[5:3\]:BOOT_PARTITION_ENABLE 的设定，可以从 Boot Area Partition 1、Boot Area Partition 2 或者 User Data Area 读出。

**3. RPMB Partition**

RPMB（Replay Protected Memory Block）Partition 是 eMMC 中的一个具有安全特性的分区。eMMC 在写入数据到 RPMB 时，会校验数据的合法性，只有指定的 Host 才能够写入，同时在读数据时，也提供了签名机制，保证 Host 读取到的数据是 RPMB 内部数据，而不是攻击者伪造的数据。

RPMB 在实际应用中，通常用于存储一些有防止非法篡改需求的数据，例如手机上指纹支付相关的公钥、序列号等。RPMB 可以对写入操作进行鉴权，但是读取并不需要鉴权，任何人都可以进行读取的操作，因此存储到 RPMB 的数据通常会进行加密后再存储。

- 3.1 容量大小

RPMB Partition 的大小是由 Extended CSD register 的 BOOT_SIZE_MULT Field 决定，大小的计算公式如下：

> Size = 128Kbytes x BOOT_SIZE_MULT

一般情况下，Boot Area Partition （笔误？RPMB Partition）的大小为 4 MB，即 RPMB_SIZE_MULT 为 32，部分芯片厂家会提供改写 RPMB_SIZE_MULT 的功能来改变 RPMB Partition 的容量大小。RPMB_SIZE_MULT 最大可以为 128，即 Boot Area Partition（笔误？RPMB Partition） 的最大容量大小可以为 128 x 128 KB = 16384 KB = 16 MB。

- 3.2 Replay Protect 原理

使用 eMMC 的产品，在产线生产时，会为每一个产品生产一个唯一的 256 bits 的 Secure Key，烧写到 eMMC 的 OTP 区域（只能烧写一次的区域），同时 Host 在安全区域中（例如：TEE）也会保留该 Secure Key。

在 eMMC 内部，还有一个RPMB Write Counter。RPMB 每进行一次合法的写入操作时，Write Counter 就会自动加一 。通过 Secure Key 和 Write Counter 的应用，RMPB 可以实现数据读取和写入的 Replay Protect。

- 3.3 RPMB 数据读取

RPMB 数据读取的流程如下：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

a. Host 向 eMMC 发起读 RPMB 的请求，同时生成一个 16 bytes 的随机数，发送给 eMMC。

b. eMMC 将请求的数据从 RPMB 中读出，并使用 Secure Key 通过 HMAC SHA-256 算法，计算读取到的数据和接收到的随机数拼接到一起后的签名。然后，eMMC 将读取到的数据、接收到的随机数、计算得到的签名一并发送给 Host。

c. Host 接收到 RPMB 的数据、随机数以及签名后，首先比较随机数是否与自己发送的一致，如果一致，再用同样的 Secure Key 通过 HMAC SHA-256 算法对数据和随机数组合到一起进行签名，如果签名与 eMMC 发送的签名是一致的，那么就可以确定该数据是从 RPMB 中读取到的正确数据，而不是攻击者伪造的数据。

通过上述的读取流程，可以保证 Host 正确的读取到 RPMB 的数据。

3.4 RPMB 数据写入

RPMB 数据写入的流程如下：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

a. Host 按照上面的读数据流程，读取 RPMB 的 Write Counter。

b. Host 将需要写入的数据和 Write Counter 拼接到一起并计算签名，然后将数据、Write Counter 以及签名一并发给 eMMC。

c. eMMC 接收到数据后，先对比 Write Counter 是否与当前的值相同，如果相同那么再对数据和 Write Counter 的组合进行签名，然后和 Host 发送过来的签名进行比较，如果签名相同则鉴权通过，将数据写入到 RPMB 中。

通过上述的写入流程，可以保证 RPMB 不会被非法篡改。

**4. User Data Area**

User Data Area (UDA) 通常是 eMMC 中最大的一个分区，是实际产品中，最主要的存储区域。

4.1 容量大小

UDA 的容量大小不需要设置，在配置完其他分区大小后，再扣除设置 Enhanced attribute 所损耗的容量，剩下的容量就是 UDA 的容量。

4.2 软件分区

为了更合理的管理数据，满足不同的应用需求，UDA 在实际产品中，会进行软件再分区。目前主流的软件分区技术有 MBR（Master Boot Record）和 GPT（GUID Partition Table）两种。这两种分区技术的基本原理类似，如下图所示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

软件分区技术一般是将存储介质划分为多个区域，既 SW Partitions，然后通过一个 Partition Table 来维护这些 SW Partitions。在 Partition Table 中，每一个条目都保存着一个 SW Partition 的起始地址、大小等的属性信息。软件系统在启动后，会去扫描 Partition Table，获取存储介质上的各个 SW Partitions 信息，然后根据这些信息，将各个 Partitions 加载到系统中，进行数据存取。

4.3 区域属性

eMMC 标准中，支持为 UDA 中一个特定大小的区域设定 Enhanced attribute。与 GPP 中的 Enhanced attribute 相同，eMMC 标准也没有定义该区域设定 Enhanced attribute 后对 eMMC 的影响。Enhanced attribute 的具体作用，由芯片制造商定义。

## eMMC 分区应用实例

在一个 Android 手机系统中，各个分区的呈现形式如下：

- mmcblk0 为 eMMC 的块设备;

- mmcblk0boot0 和 mmcblk0boot1 对应两个 Boot Area Partitions;

- mmcblk0rpmb 则为 RPMB Partition，

- mmcblk0px 为 UDA 划分出来的 SW Partitions;

- 如果存在 GPP，名称则为 mmcblk0gp1、mmcblk0gp2、mmcblk0gp3、mmcblk0gp4;

`root@xxx:/ # ls /dev/block/mmcblk0*   /dev/block/mmcblk0   /dev/block/mmcblk0boot0   /dev/block/mmcblk0boot1   /dev/block/mmcblk0rpmb   /dev/block/mmcblk0p1   /dev/block/mmcblk0p2   /dev/block/mmcblk0p3   /dev/block/mmcblk0p4   /dev/block/mmcblk0p5   /dev/block/mmcblk0p6   /dev/block/mmcblk0p7   /dev/block/mmcblk0p8   /dev/block/mmcblk0p9   /dev/block/mmcblk0p10   /dev/block/mmcblk0p11   /dev/block/mmcblk0p12   /dev/block/mmcblk0p13   /dev/block/mmcblk0p14   /dev/block/mmcblk0p15   /dev/block/mmcblk0p16   /dev/block/mmcblk0p17   /dev/block/mmcblk0p18   /dev/block/mmcblk0p19   /dev/block/mmcblk0p20   `

每一个分区会根据实际的功能来设定名称。

`root@xxx:/ # ls -l /dev/block/platform/mtk-msdc.0/11230000.msdc0/by-name/   lrwxrwxrwx root root 2015-01-03 04:03 boot -> /dev/block/mmcblk0p22   lrwxrwxrwx root root 2015-01-03 04:03 cache -> /dev/block/mmcblk0p30   lrwxrwxrwx root root 2015-01-03 04:03 custom -> /dev/block/mmcblk0p3   lrwxrwxrwx root root 2015-01-03 04:03 devinfo -> /dev/block/mmcblk0p28   lrwxrwxrwx root root 2015-01-03 04:03 expdb -> /dev/block/mmcblk0p4   lrwxrwxrwx root root 2015-01-03 04:03 flashinfo -> /dev/block/mmcblk0p32   lrwxrwxrwx root root 2015-01-03 04:03 frp -> /dev/block/mmcblk0p5   lrwxrwxrwx root root 2015-01-03 04:03 keystore -> /dev/block/mmcblk0p27   lrwxrwxrwx root root 2015-01-03 04:03 lk -> /dev/block/mmcblk0p20   lrwxrwxrwx root root 2015-01-03 04:03 lk2 -> /dev/block/mmcblk0p21   lrwxrwxrwx root root 2015-01-03 04:03 logo -> /dev/block/mmcblk0p23   lrwxrwxrwx root root 2015-01-03 04:03 md1arm7 -> /dev/block/mmcblk0p17   lrwxrwxrwx root root 2015-01-03 04:03 md1dsp -> /dev/block/mmcblk0p16   lrwxrwxrwx root root 2015-01-03 04:03 md1img -> /dev/block/mmcblk0p15   lrwxrwxrwx root root 2015-01-03 04:03 md3img -> /dev/block/mmcblk0p18   lrwxrwxrwx root root 2015-01-03 04:03 metadata -> /dev/block/mmcblk0p8   lrwxrwxrwx root root 2015-01-03 04:03 nvdata -> /dev/block/mmcblk0p7   lrwxrwxrwx root root 2015-01-03 04:03 nvram -> /dev/block/mmcblk0p19   lrwxrwxrwx root root 2015-01-03 04:03 oemkeystore -> /dev/block/mmcblk0p12   lrwxrwxrwx root root 2015-01-03 04:03 para -> /dev/block/mmcblk0p2   lrwxrwxrwx root root 2015-01-03 04:03 ppl -> /dev/block/mmcblk0p6   lrwxrwxrwx root root 2015-01-03 04:03 proinfo -> /dev/block/mmcblk0p13   lrwxrwxrwx root root 2015-01-03 04:03 protect1 -> /dev/block/mmcblk0p9   lrwxrwxrwx root root 2015-01-03 04:03 protect2 -> /dev/block/mmcblk0p10   lrwxrwxrwx root root 2015-01-03 04:03 recovery -> /dev/block/mmcblk0p1   `

## eMMC 总线协议

### eMMC 总线接口

eMMC 总线接口定义如下图所示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- CLK

CLK 信号用于从 Host 端输出时钟信号，进行数据传输的同步和设备运作的驱动。

在一个时钟周期内，CMD 和 DAT0-7 信号上都可以支持传输 1 个比特，即SDR (Single Data Rate) 模式。此外，DAT0-7 信号还支持配置为DDR (Double Data Rate) 模式，在一个时钟周期内，可以传输 2 个比特。

Host 可以在通讯过程中动态调整时钟信号的频率（注，频率范围需要满足 Spec 的定义）。通过调整时钟频率，可以实现省电或者数据流控（避免 Over-run 或者 Under-run）功能。在一些场景中，Host 端还可以关闭时钟，例如 eMMC 处于 Busy 状态时，或者接收完数据，进入 Programming State 时。

- CMD

CMD 信号主要用于 Host 向 eMMC 发送 Command 和 eMMC 向 Host 发送对于的 Response。

- DAT0-7

DAT0-7 信号主要用于 Host 和 eMMC 之间的数据传输。在 eMMC 上电或者软复位后，只有 DAT0 可以进行数据传输，完成初始化后，可配置 DAT0-3 或者 DAT0-7 进行数据传输，即数据总线可以配置为 4 bits 或者 8 bits 模式。

- Data Strobe

Data Strobe 时钟信号由 eMMC 发送给 Host，频率与 CLK 信号相同，用于 Host 端进行数据接收的同步。Data Strobe 信号只能在HS400 模式下配置启用，启用后可以提高数据传输的稳定性，省去总线 tuning 过程。

### eMMC 总线模型

eMMC 总线中一个 Host可以有多个 eMMC Devices。总线上的所有通讯都由 Host 端以一个 Command 开发发起，Host 一次只能与一个 eMMC Device 通讯。

系统在上电启动后，Host 会为所有 eMMC Device 逐个分配地址（RCA，Relative device Address）。当 Host 需要和某一个 eMMC Device 通讯时，会先根据 RCA 选中该 eMMC Device，只有被选中的 eMMC Device 才会响应 Host 的 Command。

**1. 速率模式**

随着 eMMC 协议的版本迭代，eMMC 总线的速率越来越高。为了兼容旧版本的 eMMC Device，所有 Devices 在上电启动或者 Reset 后，都会先进入兼容速率模式（Backward Compatible Mode）。在完成 eMMC Devices 的初始化后，Host 可以通过特定的流程，让 Device 进入其他高速率模式，目前支持以下的几种速率模式。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

> 注：
>
> Extended CSD byte\[185\] HS_TIMING 寄存器可以配置总线速率模式
>
> Extended CSD byte\[183\] BUS_WIDTH 寄存器用于配置总线宽度和 Data Strobe

**2. 通信模型**

Host 与 eMMC Device 之间的通信都是由 Host 以一个 Command 开始发起的，eMMC Device 在完成 Command 所指定的任务后，则返回一个 Response。

2.1 Read Data

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Host 从 eMMC Device 读取数据的流程如上图所示。

如果 Host 发送的是 Single Block Read 的 Command，那么 eMMC Device 只会发送一个 Block 的数据。

如果 Host 在发送 Multiple Block Read 的 Command 前，先发送一个设定需要读取的 Block Count 的 Command。eMMC Device 在完成指定 Block Count 的数据发送后，就自动结束数据传输，不需要 Host 主动发送 Stop Command。

如果 Host 没有发送设定需要读取的 Block Count 的 Command，发送 Multiple Block Read 的 Command 后，eMMC Device 会持续发送数据，直到 Host 发送 Stop Command 停止数据传输。

> 注：从 eMMC Device 读数据都是按 Block 读取的。Block 大小可以由 Host 设定，或者固定为 512 Bytes，不同的速率模式下有所不同。

2.2 Write Data

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Host 向 eMMC Device 写入数据的流程如上图所示。

如果 Host 发送的是 Single Block Write Command，那么 eMMC Device 只会将后续第一个 Block 的数据写入的存储器中。

如果 Host 在发送 Multiple Block Write 的 Command 前，先发送一个设定需要读取的 Block Count 的 Command。eMMC Device 在接收到指定 Block Count 的数据后，就自动结束数据接收，不需要 Host 主动发送 Stop Command。

如果 Host 没有发送设定需要读取的 Block Count 的 Command，发送 Multiple Block Write 的 Command 后，eMMC Device 会持续接收数据，直到 Host 发送 Stop Command 停止数据传输。eMMC Device 在接收到一个 Block 的数据后，会进行 CRC 校验，然后将校验结果通过 CRC Token 发送给 Host。

发送完 CRC Token 后，如果 CRC 校验成功，eMMC Device 会将数据写入到内部存储器时，此时 DAT0 信号会拉低，作为 Busy 信号。Host 会持续检测 DAT0 信号，直到为高电平时，才会接着发送下一个 Block 的数据。如果 CRC 校验失败，那么 eMMC Device 不会进行数据写入，此次传输后续的数据都会被忽略。

> 注：向 eMMC Device 写数据都是按 Block 写入的。Block 大小可以由 Host 设定，或者固定为 512 Bytes，不同的速率模式下有所不同。

2.3 No Data

在 Host 与 eMMC Device 的通信中，有部分交互是不需要进行数据传输的，还有部分交互甚至不需要 eMMC Device 的回复 Response。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

2.4 Command

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

如上图所示，eMMC Command 由 48 Bits 组成，各个 Bits 的解析如下所示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Start Bit 固定为 "0"，在没有数据传输的情况下，CMD 信号保持高电平，当 Host 将 Start Bit 发送到总线上时，eMMC Device 可以很方便检测到该信号，并开始接收 Command。

Transmission Bit 固定为 "1"，指示了该数据包的传输方向为 Host 发送到 eMMC Device。

Command Index 和 Argument 为 Command 的具体内容，不同的 Command 有不同的 Index，不同的 Command 也有各自的 Argument。更多的细节，请参考eMMC Commands 章节。

CRC7 是包含 Start Bit、Transmission Bit、 Command Index 和 Argument 内容的 CRC 校验值。End Bit 为结束标志位，固定为"1"。

> 注：CRC 校验简单来说，是发送方将需要传输的数据“除于”（模2除）一个约定的数，并将得到的余数附在数据上一并发送出去。接收方收到数据后，再做同样的“除法”，然后校验得到余数是否与接收的余数相同。如果不相同，那么意味着数据在传输过程中发生了改变。更多的细节不在本文展开描述，感兴趣的读者可以参考 CRC wiki 中的介绍。

2.5 Response

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

eMMC Response 有两种长度的数据包，分别为 48 Bits 和 136 Bits。

Start Bit 与 Command 一样，固定为 "0"，在没有数据传输的情况下，CMD 信号保持高电平，当 eMMC Device 将 Start Bit 发送到总线上时，Host 可以很方便检测到该信号，并开始接收 Response。

Transmission Bit 固定为 "0"，指示了该数据包的传输方向为 eMMC Device 发送到 Host。

Content 为 Response 的具体内容，不同的 Command 会有不同的 Content。更多的细节，请参考eMMC Responses 章节。CRC7 是包含 Start Bit、Transmission Bit 和 Content 内容的 CRC 校验值。End Bit 为结束标志位，固定为"1"。

【转自https://blog.csdn.net/u013686019/article/details/66472291】

eMMC2

驱动22

eMMC · 目录

下一篇学习下 eMMC 的工作模式

阅读 2653

​

写留言

**留言 3**

- 逸

  2022年4月6日

  赞

  想做点实验改一下有例子吗哥

  人人极客社区

  作者2022年4月6日

  赞1

  后面有实践

- 周立广

  北京2022年6月25日

  赞

  写的太好了，就是想问一下，写数据完成后CRC 校验正确，eMMC还会给主机发送什么信号吗，还是直接写入flash呢

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/9sNwsXcN68pL55XIyzTrCHZTbIUdTibQcuzuCaYeGTXNMyn6ACmicUrpoDC0xZSap46XJ59sKysPg9Rg379f32cA/300?wx_fmt=png&wxfrom=18)

人人极客社区

1836

3

写留言

**留言 3**

- 逸

  2022年4月6日

  赞

  想做点实验改一下有例子吗哥

  人人极客社区

  作者2022年4月6日

  赞1

  后面有实践

- 周立广

  北京2022年6月25日

  赞

  写的太好了，就是想问一下，写数据完成后CRC 校验正确，eMMC还会给主机发送什么信号吗，还是直接写入flash呢

已无更多数据

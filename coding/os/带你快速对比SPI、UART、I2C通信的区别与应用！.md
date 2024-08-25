# 

土豆居士 一口Linux

 _2021年10月01日 10:49_

击上方“**一口Linux**”，选择“**星标公众号**”

# 

干货福利，第一时间送达！

# ![图片](https://mmbiz.qpic.cn/mmbiz/cZV2hRpuAPiaJQXWGyC9wrUzIicibgXayrgibTYarT3A1yzttbtaO0JlV21wMqroGYT3QtPq2C7HMYsvicSB2p7dTBg/640?wx_fmt=gif&wxfrom=13&tp=wxpic "动态黑色音符")

电子设备之间的通信就像人类之间的交流，双方都需要说相同的语言。在电子产品中，这些语言称为通信协议。  

之前有单独地分享了SPI、UART、I2C通信的文章，这篇对它们做一些对比。  

## 串行 VS 并行

  

电子设备通过发送数据位从而实现相互交谈。位是二进制的，只能是1或0。通过电压的快速变化，位从一个设备传输到另一个设备。在以5V工作的系统中，“0”通过0V的短脉冲进行通信，而“1”通过5V的短脉冲进行通信。   

  

数据位可以通过并行或串行的形式进行传输。 在并行通信中，数据位在导线上同时传输。下图显示了二进制（01000011）中字母“C”的并行传输：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

在串行通信中，位通过单根线一一发送。下图显示了二进制（01000011）中字母“C”的串行传输：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

## SPI通信

  

SPI是一种常见的设备通用通信协议。它有一个独特优势就是可以无中断传输数据，可以连续地发送或接收任意数量的位。而在I2C和UART中，数据以数据包的形式发送，有着限定位数。

  

在SPI设备中，设备分为**主机与从机系统**。主机是控制设备（通常是微控制器），而从机（通常是传感器，显示器或存储芯片）从主机那获取指令。

  

一套SPI通讯共包含四种信号线：**MOSI** (Master Output/Slave Input) – 信号线，主机输出，从机输入。**MISO** (Master Input/Slave Output) – 信号线，主机输入，从机输出。**SCLK** (Clock) – 时钟信号。**SS/CS** (Slave Select/Chip Select) – 片选信号。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

  

**SPI协议特点**

实际上，从机的数量受系统负载电容的限制，它会降低主机在电压电平之间准确切换的能力。

  

**工作原理**

  

**时钟信号**

每个时钟周期传输一位数据，因此数据传输的速度取决于时钟信号的频率。 时钟信号由于是主机配置生成的，因此SPI通信始终由主机启动。 

**设备共享时钟信号的任何通信协议都称为同步。**SPI是一种同步通信协议，还有一些异步通信不使用时钟信号。 例如在UART通信中，双方都设置为预先配置的波特率，该波特率决定了数据传输的速度和时序。

  

**片选信号**

主机通过拉低从机的CS/SS来使能通信。 **在空闲/非传输状态下，片选线保持高电平**。在主机上可以存在多个CS/SS引脚，允许主机与多个不同的从机进行通讯。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

如果主机只有一个片选引脚可用，则可以通过以下方式连接这些从器件：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**MOSI和MISO**

主机通过MOSI以串行方式将数据发送给从机，从机也可以通过MISO将数据发送给主机，两者可以同时进行。所以理论上，**S****PI是一种全双工的通讯协议。**

  

**传输步骤**

  

  

**1. 主机输出时钟信号**

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**2. 主机拉低SS / CS引脚，激活从机**

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**3. 主机通过MOSI将数据发送给从机**

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**4. 如果需要响应，则从机通过MISO将数据返回给主机**

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

使用SPI有一些优点和缺点，如果在不同的通信协议之间进行选择，则应根据项目要求进行充分考量。

  

**优劣**

  

**优点**

SPI通讯无起始位和停止位，因此数据可以连续流传输而不会中断；没有像I2C这样的复杂的从站寻址系统，数据传输速率比I2C更高（几乎快两倍）。独立的MISO和MOSI线路，可以同时发送和接收数据。

  

**缺点**

SPI使用四根线（I2C和UART使用两根线），没有信号接收成功的确认（I2C拥有此功能），没有任何形式的错误检查（如UART中的奇偶校验位等）。  

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

UART代表通用异步接收器/发送器也称为串口通讯，它不像SPI和I2C这样的通信协议，而是微控制器中的物理电路或独立的IC。

  
UART的主要目的是发送和接收串行数据，其最好的优点是它仅使用两条线在设备之间传输数据。UART的原理很容易理解，但是如果您还没有阅读SPI 通讯协议，那可能是一个不错的起点。

## UART通信

  

在UART通信中，两个UART直接相互通信。 发送UART将控制设备（如CPU）的并行数据转换为串行形式，以串行方式将其发送到接收UART。只需要两条线即可在两个UART之间传输数据，数据从发送UART的Tx引脚流到接收UART的Rx引脚：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**UART属于异步通讯**，这意味着没有时钟信号，取而代之的是在数据包中添加开始和停止位。这些位定义了数据包的开始和结束，因此接收UART知道何时读取这些数据。 

  

当接收UART检测到起始位时，它将以特定波特率的频率读取。波特率是数据传输速度的度量，以每秒比特数（bps）表示。两个UART必须以大约相同的波特率工作，发送和接收UART之间的波特率只能相差约10％。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

  

**工作原理**

发送UART从数据总线获取并行数据后，它会添加一个起始位，一个奇偶校验位和一个停止位来组成数据包并从Tx引脚上逐位串行输出，接收UART在其Rx引脚上逐位读取数据包。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

UART数据包含有1个起始位，5至9个数据位（取决于UART），一个可选的奇偶校验位以及1个或2个停止位：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**起始位：**

  

UART数据传输线通常在不传输数据时保持在高电压电平。开始传输时发送UART在一个时钟周期内将传输线从高电平拉低到低电平，当接收UART检测到高电压到低电压转换时，它开始以波特率的频率读取数据帧中的位。

  

**数据帧：**

  

数据帧内包含正在传输的实际数据。如果使用奇偶校验位，则可以是5位，最多8位。如果不使用奇偶校验位，则数据帧的长度可以为9位。 

  

**校验位：**

  

奇偶校验位是接收UART判断传输期间是否有任何数据更改的方式。接收UART读取数据帧后，它将对值为1的位数进行计数，并检查总数是偶数还是奇数，是否与数据相匹配。

  

**停止位：**

  

为了向数据包的结尾发出信号，发送UART将数据传输线从低电压驱动到高电压至少持续两位时间。

  

  

  

  

**传输步骤**

  

  

1. 发送UART从数据总线并行接收数据： 
    

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

2.发送UART将起始位，奇偶校验位和停止位添加到数据帧：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

3.整个数据包从发送UART串行发送到接收UART。接收UART以预先配置的波特率对数据线进行采样：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

4.接收UART丢弃数据帧中的起始位，奇偶校验位和停止位：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

5.接收UART将串行数据转换回并行数据，并将其传输到接收端的数据总线：  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

  

**优劣**

  

没有任何通信协议是完美的，但是UART非常擅长于其工作。以下是一些利弊，可帮助您确定它们是否适合您的项目需求：

  

**优点**

- 仅使用两根电线
    
- 无需时钟信号
    
- 具有奇偶校验位以允许进行错误检查
    
- 只要双方都设置好数据包的结构
    
- 有据可查并得到广泛使用的方法
    

  

**缺点**

- 数据帧的大小最大为9位
    
- 不支持多个从属系统或多个主系统
    
- 每个UART的波特率必须在彼此的10％之内
    

  

## I2C通信

  

I2C总线是由Philips公司开发的一种简单、双向二线制同步串行总线。它只需要两根线即可传送信息。它结合了 SPI 和 UART 的优点，您可以将多个从机连接到单个主机（如SPI那样），也可以使用多个主机控制一个或多个从机。当您想让多个微控制器将数据记录到单个存储卡或将文本显示到单个LCD时，这将非常有用。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

SDA (Serial Data) – 数据线。

SCL (Serial Clock) – 时钟线。

I2C是串行通信协议，因此数据沿着SDA一点一点地传输。与SPI一样，I2C也需要时钟同步信号且时钟始终由主机控制。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**工作原理**

  

I2C的数据传输是以多个msg的形式进行，每个msg都包含从机的二进制地址帧，以及一个或多个数据帧，还包括开始条件和停止条件，读/写位和数据帧之间的ACK / NACK位：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

启动条件：当SCL是高电平时，SDA从高电平向低电平切换。

  

停止条件：当SCL是高电平时，SDA由低电平向高电平切换。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

地址帧：每个从属设备唯一的7位或10位序列，用于主从设备之间的地址识别。

  

读/写位：一位，如果主机是向从机发送数据则为低电平，请求数据则为高电平。

  

ACK/NACK：消息中的每个帧后均带有一个ACK/NACK位。如果成功接收到地址帧或数据帧，接收设备会返回一个ACK位用于表示确认。

  

**寻址**

由于I2C没有像SPI那样的片选线，因此它需要使用另一种方式来确认某一个从设备，而这个方式就是 —— 寻址 。

  

主机将要通信的从机地址发送给每个从机，然后每个从机将其与自己的地址进行比较。如果地址匹配，它将向主机发送一个低电平ACK位。如果不匹配，则不执行任何操作，SDA线保持高电平。

  

**读/写位** 

地址帧的末尾包含一个读/写位。如果主机要向从机发送数据，则为低电平。如果是主机向从机请求数据，则为高电平。

  

**数据帧**

当主机检测到从机的ACK位后，就可以发送第一个数据帧了。数据帧始终为8位，每个数据帧后紧跟一个ACK / NACK位，来验证接收状态。当发送完所有数据帧后，主机可以向从机发送停止条件来终止通信。

  

**传输步骤**

  

1. 在SCL线为高电平时，主机通过将SDA线从高电平切换到低电平来启动总线通信。

  

2. 主机向总线发送要与之通信的从机的7位或10位地址，以及读/写位：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

3. 每个从机将主机发送的地址与其自己的地址进行比较。如果地址匹配，则从机通过将SDA线拉低一位返回一个ACK位。如果主机的地址与从机的地址不匹配，则从机将SDA线拉高。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

4. 主机发送或接收数据帧：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

5. 传输完每个数据帧后，接收设备将另一个ACK位返回给发送方，以确认已成功接收到该帧：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

6. 随后主机将SCL切换为高电平，然后再将SDA切换为高电平，从而向从机发送停止条件。

  

**单个主机VS多个从机**

由于I2C使用寻址功能，可以通过一个主机控制多个从机。使用7位地址时，最多可以使用128(27)个唯一地址。使用10位地址并不常见，但可以提供1,024(210)个唯一地址。如果要将多个从机连接到单个主机时，请使用4.7K欧的上拉电阻将它们连接，例如将SDA和SCL线连接到Vcc：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**多个主机VS多个从机**

I2C支持多个主机同时与多个从机相连，当两个主机试图通过SDA线路同时发送或接收数据时，就会出现问题。因此每个主机都需要在发送消息之前检测SDA线是低电平还是高电平。如果SDA线为低电平，则意味着另一个主机正在控制总线。如果SDA线高，则可以安全地发送数据。如果要将多个主机连接到多个从机，请使用4.7K欧的上拉电阻将SDA和SCL线连接到Vcc：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**优劣**

  

与其他协议相比，I2C可能听起来很复杂。以下是一些利弊，可帮助您确定它们是否适合您的项目需求：

  

**优点**

- 仅使用两根电线
    
- 支持多个主机和多个从机
    
- 每个UART的波特率必须在彼此的10％之内
    
- 硬件比UART更简单
    
- 众所周知且被广泛使用的协议  
    
      
    

**缺点**

- 数据传输速率比SPI慢
    
- 数据帧的大小限制为8位
    

  

END

  

****来源：网络****

---

版权归原作者所有，如有侵权，请联系删除。

**关注，回复【****1024****】海量Linux资料赠送**

![](http://mmbiz.qpic.cn/mmbiz_png/icRxcMBeJfc8535w2vKlsLPf5hwdMjpYrzuVCHx0rcQmvv8rYqTFtIyic5qErtciaibqaIOWgeKkDsOMeae4HciaUaw/300?wx_fmt=png&wxfrom=19)

**一口Linux**

《从零开始学ARM》作者，一起学习嵌入式，Linux，网络，驱动，arm知识。

300篇原创内容

公众号

**精彩文章合集**

[linux](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1507350615537025026#wechat_redirect)入门

[C语言](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1531367002169212930#wechat_redirect)

[Linux驱动](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1502410824114569216#wechat_redirect)

[ARM](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1614665559315382276#wechat_redirect)

[计算机网络](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1598710257097179137#wechat_redirect)

[粉丝问答](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1629876820810465283#wechat_redirect)

[所有原创](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1479949091139813387#wechat_redirect)

点击“**阅读原文**”查看更多分享，欢迎**点分享、收藏、点赞、在看**

uart4

uart · 目录

下一篇Linux 终端(TTY)

阅读原文

阅读 1705

​

写留言

**留言 8**

- Allen
    
    江苏2022年11月16日
    
    赞1
    
    多主机情况下为什么SDA线高，就可以安全地发送数据，假如此时另一个从机刚好发送的数据位是1呢
    
    一口Linux
    
    作者2022年11月16日
    
    赞1
    
    只有主机才能发起通信，每次通信前，他都要先确认自己是否有权限，去内核代码里看下，一些控制器的那个发送函数，每次通信前都要判断的
    
    Allen
    
    江苏2022年11月16日
    
    赞1
    
    那这篇文章说的“如果SDA线高，则可安全地放松数据”是否欠妥，好像有个总线仲裁的过程吧
    
- 堇
    
    2021年10月1日
    
    赞1
    
    彭老师辛苦，国庆节还这么拼！节日快乐！![[爱心]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    一口Linux
    
    作者2021年10月2日
    
    赞
    
    学习永不停息![[偷笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 刘长春
    
    2021年10月2日
    
    赞
    
    对比一目了然，收藏了
    
    一口Linux
    
    作者2021年10月2日
    
    赞
    
    国庆不忘学习！卷起来！！
    
- qiuzy
    
    2021年10月2日
    
    赞
    
    写的很详细，一目了然
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/icRxcMBeJfc8535w2vKlsLPf5hwdMjpYrzuVCHx0rcQmvv8rYqTFtIyic5qErtciaibqaIOWgeKkDsOMeae4HciaUaw/300?wx_fmt=png&wxfrom=18)

一口Linux

1219

8

写留言

**留言 8**

- Allen
    
    江苏2022年11月16日
    
    赞1
    
    多主机情况下为什么SDA线高，就可以安全地发送数据，假如此时另一个从机刚好发送的数据位是1呢
    
    一口Linux
    
    作者2022年11月16日
    
    赞1
    
    只有主机才能发起通信，每次通信前，他都要先确认自己是否有权限，去内核代码里看下，一些控制器的那个发送函数，每次通信前都要判断的
    
    Allen
    
    江苏2022年11月16日
    
    赞1
    
    那这篇文章说的“如果SDA线高，则可安全地放松数据”是否欠妥，好像有个总线仲裁的过程吧
    
- 堇
    
    2021年10月1日
    
    赞1
    
    彭老师辛苦，国庆节还这么拼！节日快乐！![[爱心]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    一口Linux
    
    作者2021年10月2日
    
    赞
    
    学习永不停息![[偷笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 刘长春
    
    2021年10月2日
    
    赞
    
    对比一目了然，收藏了
    
    一口Linux
    
    作者2021年10月2日
    
    赞
    
    国庆不忘学习！卷起来！！
    
- qiuzy
    
    2021年10月2日
    
    赞
    
    写的很详细，一目了然
    

已无更多数据
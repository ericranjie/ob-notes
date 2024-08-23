# 
原创 土豆居士 一口Linux

 _2021年12月29日 11:50_

击上方“**一口Linux**”，选择“**星标公众号**”

# 

干货福利，第一时间送达！

# ![图片](https://mmbiz.qpic.cn/mmbiz/cZV2hRpuAPiaJQXWGyC9wrUzIicibgXayrgibTYarT3A1yzttbtaO0JlV21wMqroGYT3QtPq2C7HMYsvicSB2p7dTBg/640?wx_fmt=gif&wxfrom=13&tp=wxpic "动态黑色音符")

  

**代码中自由颜如玉！**

  
**代码中自有黄金屋！**

  

**那么Linux内核代码到底有多少行？**

  

**我们需要多久能读完呢？**

## 一、内核行数

Linux内核分为CPU调度、内存管理、网络和存储四大子系统，针对硬件的驱动成百上千。代码的数量更是大的惊人。

先说说最早的内核linux 0.11，下面这本书可以说很多驱动工程师都学习过，我花了大概1个半月，勉强看了一遍。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

再来看看内核代码量的统计。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

2020年1月1日，Linux内核Git源码树中的代码达到了2780万行。

phoronix网站统计了Linux内核在进入2020年时的一些源码数据并作了总结。

从统计数据来看，Linux内核源码树共有：

**27852148行(包括文档、Kconfig文件、树中的用户空间实用程序等)**、

**887925次commit**

**21074位不同的作者**

**2780万行代码分布在66492个文件中**。

Linux内核从最初的10000行代码到现在的2780万行代码就是全球精英共同贡献的结果。

按照一天一万行的速度，也需要2700天，也需要7年多。

这还是建立在所有单次都认识，

所有代码逻辑看了的都懂，

而且都不忘记的基础上。

实际上即使我们真的看完了，

几年后内核又会有非常大的变化，

可以说一辈子都看不完Linux内核的代码。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Linux内核Git源码树中的代码达到了2780万行，核心代码只有2%是由李纳斯•托瓦兹自己编写的，其他均是其他个人和组织贡献的，李纳斯•托瓦兹公开了Linux但保留了选择新代码和需要合并的新方法的最终裁定权。

除了Linus Torvalds，对内核贡献最多的是David S.Miller、 Mark Brown、Takashi Iwai、Arnd Bergmann、Al Viro和Mauro Carvalho Chehab。

而参与贡献的公司，从域名统计来看，谷歌、Intel与Red Hat排在了最前列。

## 二、内核目录文件大小

然而，现在的内核已经膨胀的不成样子了，以还不算最新的linux-4.1.15为例：

整个内核源码一共约 793M：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)驱动代码占了大概一半，大约380M:

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)体系相关的代码大约134M：![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)网路子系统相关的代码26M：![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)文件系统相关的代码37M:

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)linux内核核心代码大约6.8M:

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这些目录任意一个目录想完全看明白都非常不容易。

## 三、内核子系统

**什么是内核：**

在计算机科学中是一个用来管理软件发出的数据I/O（输入与输出）要求的计算机程序，将这些要求转译为数据处理的指令并交由中央处理器（CPU）及计算机中其他电子组件进行处理，是现代操作系统中最基本的部分。![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

它是为众多应用程序提供对计算机硬件的安全访问的一部分软件，这种访问是有限的，并由内核决定一个程序在什么时候对某部分硬件操作多长时间。

linux内核代码涉及知识点包括汇编指令、c语言、硬件组成原理、操作系统、数据结构和算法、各种外设总线、驱动、网络协议栈。

直接对硬件操作是非常复杂的。所以内核通常提供一种硬件抽象的方法，来完成这些操作。

通过进程间通信机制及系统调用，应用进程可间接控制所需的硬件资源（特别是处理器及IO设备）。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

最上面是用户（或应用程序）空间。这是用户应用程序执行的地方。用户空间之下是内核空间，Linux 内核正是位于这里。

GNU C Library （glibc）也在这里。它提供了连接内核的系统调用接口，还提供了在用户空间应用程序和内核之间进行转换的机制。

内核和用户空间的应用程序使用的是不同的保护地址空间。

每个用户空间的进程都使用自己的虚拟地址空间，而内核则占用单独的地址空间。

Linux 内核可以进一步划分成 3 层。最上面是系统调用接口，它实现了一些基本的功能，例如 read 和 write。

系统调用接口之下是内核代码，可以更精确地定义为独立于体系结构的内核代码。这些代码是 Linux 所支持的所有处理器体系结构所通用的。

在这些代码之下是依赖于体系结构的代码，构成了通常称为 BSP（Board Support Package）的部分。这些代码用作给定体系结构的处理器和特定于平台的代码。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

内核主要系统包括：SCI：系统调用接口 PM：进程管理 VFS：虚拟文件系统 MM：内存管理 Network Stack：内核协议栈 Arch：体系架构 DD：设备驱动

### 1 系统调用接口

SCI 层提供了某些机制执行从用户空间到内核的函数调用。这个接口依赖于体系结构，甚至在相同的处理器家族内也是如此。

SCI 实际上是一个非常有用的函数调用多路复用和多路分解服务。

在 ./linux/kernel 中您可以找到 SCI 的实现，并在 ./linux/arch 中找到依赖于体系结构的部分。

### 2 进程管理

进程管理的重点是进程的执行。

在内核中，这些进程称为线程，代表了单独的处理器虚拟化（线程代码、数据、堆栈和 CPU 寄存器）。

在用户空间，通常使用进程 这个术语，不过 Linux 实现并没有区分这两个概念（进程和线程）。

内核通过 SCI 提供了一个应用程序编程接口（API）来创建一个新进程（fork、exec 或 Portable Operating System Interface [POSIX] 函数），停止进程（kill、exit），并在它们之间进行通信和同步（signal 或者 POSIX 机制）。

### 3 内存管理

内核所管理的另外一个重要资源是内存。为了提高效率，如果由硬件管理虚拟内存，内存是按照所谓的内存页方式进行管理的（对于大部分体系结构来说都是 4KB）。

Linux 包括了管理可用内存的方式，以及物理和虚拟映射所使用的硬件机制。

### 4 虚拟文件系统

虚拟文件系统（VFS）是 Linux 内核中非常有用的一个方面，因为它为文件系统提供了一个通用的接口抽象。VFS 在 SCI 和内核所支持的文件系统之间提供了一个交换层。![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在 VFS 上面，是对诸如 open、close、read 和 write 之类的函数的一个通用 API 抽象。在 VFS 下面是文件系统抽象，它定义了上层函数的实现方式。

它们是给定文件系统（超过 50 个）的插件。文件系统的源代码可以在 ./linux/fs 中找到。

文件系统层之下是缓冲区缓存，它为文件系统层提供了一个通用函数集（与具体文件系统无关）。

这个缓存层通过将数据保留一段时间（或者随即预先读取数据以便在需要是就可用）优化了对物理设备的访问。缓冲区缓存之下是设备驱动程序，它实现了特定物理设备的接口。

### 5 网络堆栈

网络堆栈在设计上遵循模拟协议本身的分层体系结构。

回想一下，Internet Protocol (IP) 是传输协议（通常称为传输控制协议或 TCP）下面的核心网络层协议。TCP 上面是 socket 层，它是通过 SCI 进行调用的。

socket 层是网络子系统的标准 API，它为各种网络协议提供了一个用户接口。

从原始帧访问到 IP 协议数据单元（PDU），再到 TCP 和 User Datagram Protocol (UDP)，socket 层提供了一种标准化的方法来管理连接，并在各个终点之间移动数据。内核中网络源代码可以在 ./linux/net 中找到。

### 6 设备驱动程序

Linux 内核中有大量代码都在设备驱动程序中，它们能够运转特定的硬件设备。

Linux 源码树提供了一个驱动程序子目录，这个目录又进一步划分为各种支持设备，例如 Bluetooth、I2C、serial 等。设备驱动程序的代码可以在 ./linux/drivers 中找到。

下面这个图形象的讲解了Linux内核都有哪些东西！![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 四、如何学习内核？

### 1. 学习主线

linux内核源码大而全，一个人，即使再聪明、再有精力，也不可能完全看完、看懂所有的linux内核源码。

一口君建议按照以下主线进行深入研究：

- linux驱动架构
    
- linux网络子系统
    
-   
    

- linux内核启动过程
    

- linux内存管理机制
    
- linux调度器
    
- linux进程管理
    
- linux虚拟机制(kvm)
    
- linux内核实时化技术
    

沿着某一个主线，深入进去，在研究清楚这个主线的同时，向其他的主线扩展、渗透和学习。

> 此处之所以将驱动列为学习内核的入口，是因为内核为很多外设驱动实现了架构， 比如I2C、SPI、UART、PCIE、字符设备、网络设备、块设备， 我们可以从最基本的字符设备学起， 学习如何编写一个简单的模块 学习如何如何为一些简单的设备比如LED、KEY、ADC等编写驱动 可以说驱动是我们学习内核最简单的入口，

由点到线、由线到面、由面到体，层层深入、不断精进，是学习linux内核源码的一个有效的方法。

### 2. 代码阅读工具

对于代码阅读方法从两个角度来介绍，一个方面是需要选择一个比较有效阅读代码的工具。

一口君强烈推荐：source insight这款阅读代码神器！

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

也可以使用vscode或者vim+ctags的组合。

不过一口君十几年的从业经验，

99%以上的开发人员都选择SI阅读内核代码。

代码并不是写给人看的，而是交给机器运行的。

所以我们去理解别人的代码时，并不能像看小说一样去通篇的阅读代码，而应该是像研究化石一样去调查它，解密它。

有时我们往往也需要把对方的一段代码亲手的实现一遍，然后自己举一反三看自己会怎么去实现它，才能真正的理解。

### 3. 学习的内核版本

有些人推荐先阅读一些低版本的内核，比如0.01版的，总代码量才1万行左右。

阅读这个代码大概一个月应该能比较清晰了。

但是，改代码与现在的代码差异巨大，阅读后可以理解基本思想，但对理解现有代码的帮助不是特别明显。

3.10版本之后的内核都支持设备树！

所以一口君建议是尽量选择3.10版本之后的代码阅读学习。

最好选择一款开发板学习！

开发板的选择一定要选择资料比较全，

售后比较好的品牌！

**否则学习中遇到的一个小问题都可能被卡个一两周。**

无形中增加了学习的成本，

要知道时间就是金钱！

对于初学者来说，

强烈推荐正点原子的开发板！

### 4. 学习Linux最重要的是培养自己写代码的能力和对Linux框架结构的了解

Linux内核中绝大部分代码都是由这个地球上顶尖的技术大牛所编写，

这些代码的高内聚低耦合，

其精准度，简洁度、质量都相当的高，

每每看到一段高质量的代码，

一口君都会被那一行行枯燥的代码背后隐藏的设计思想所震撼，所折服！

**阅读内核的代码简直就是在欣赏艺术品！**

很多粉丝问我如何提高自己的C语言编程水平， 一口君不厌其烦的 重复着同样一句话：**看Linux内核！**

**代码中自由颜如玉！代码中自有黄金屋！**

我们一定要像泡妞一样来泡内核！

时刻保持激情，任性和耐性！

耐住寂寞，天天读它，泡她！

从量变到质变！

水滴石穿！

愿各位都能够熟练掌握Linux，

**实现从程序员涅槃成为真正的软件大师！**

end

  

  

**一口Linux** 

  

**关注，回复【****1024****】海量Linux资料赠送**

![](https://res.wx.qq.com/t/fed_upload/b39ef69e-c4d6-4169-8612-5f00a84860e7/wx-avatar-default.svg)

公众号

**精彩文章合集**  

文章推荐

☞【专辑】[ARM](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1614665559315382276#wechat_redirect)

☞【专辑】[粉丝问答](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1629876820810465283#wechat_redirect)

☞【专辑】[所有原创](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1479949091139813387#wechat_redirect)

☞【专辑】[linux](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1507350615537025026#wechat_redirect)入门

☞【专辑】[计算机网络](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1598710257097179137#wechat_redirect)

☞【专辑】[Linux驱动](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUxMjEyNDgyNw==&action=getalbum&album_id=1502410824114569216#wechat_redirect)

☞【干货】[嵌入式驱动工程师学习路线](http://mp.weixin.qq.com/s?__biz=MzUxMjEyNDgyNw==&mid=2247496985&idx=1&sn=c3d5e8406ff328be92d3ef4814108cd0&chksm=f96b87edce1c0efb6f60a6a0088c714087e4a908db1938c44251cdd5175462160e26d50baf24&scene=21#wechat_redirect)

☞【干货】[Linux嵌入式所有知识点-思维导图](http://mp.weixin.qq.com/s?__biz=MzUxMjEyNDgyNw==&mid=2247497822&idx=1&sn=1e2aed9294f95ae43b1ad057c2262980&chksm=f96b8aaace1c03bc2c9b0c3a94c023062f15e9ccdea20cd76fd38967b8f2eaad4dfd28e1ca3d&scene=21#wechat_redirect)

  

点击“**阅读原文**”查看更多分享，欢迎**点分享、收藏、点赞、在看**

  

![](https://mmbiz.qlogo.cn/mmbiz_jpg/h5qLUehhRhd5fEkYx3icmhEn7HzMF25aia45k0kTVelC8aG5ZNsXn5IXCjR3hkgg4fqM2zFRXDp3GqrbDhKoeEoQ/0?wx_fmt=jpeg)

土豆居士

 植发 

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=MzUxMjEyNDgyNw==&mid=2247501138&idx=1&sn=ca1a71b9e5ec2cac875ff7e1b46a8993&chksm=f96bb7a6ce1c3eb03a56c0b53f6a4011b1906d006fd05732e58f6ef7767f8bf50c98e367e1d3&mpshare=1&scene=24&srcid=1229LWTT9ufligs9us0IuSey&sharer_sharetime=1640753252175&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d06ca38213d92bf55dc50e30053abc95f9055ac04fc5f9d7ec42119a16eadf90be88dbe3a78004b86d55c53728ce3140d2561c0807755a3b05005413e0a00eeb5544d02cb86140e97e9611623ad12ef2f69e5dcb9b0a9d9482b7161b03771a8a32c47a277ab09fb7d1ec3de89070ccd04ca68c429fef519cf7&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQjWPmr1pdNaK%2Ben%2B2o5D%2B0RLmAQIE97dBBAEAAAAAAAiRObwGNYkAAAAOpnltbLcz9gKNyK89dVj0QCXczZ2j0KxeA2TmX7NCE2IeEBYiGiNHh7ZMTrJ4e6G9yg%2FswrGyFfxJ%2FMnHkkKGsXc9qTeP9B4WR8DInP8FjX0UWr2QxSFute1zr9VJDSxSsXB0vcubPGMdZRgeq6s2G38E%2FHHjqMfmTXLOCbLwfAWDLrbYLAuMZPvxfTZ17LHUlnT4bZaT0rHDCvsdpsxpGrqZlXogclyw0jgiTE0i9BfSP5cIvMnLDRgwlz7EXQckqBxlzOAiAZ%2FuXqC3gGms&acctmode=0&pass_ticket=a%2BU3G0IE73rKC2InBR96atzCvTpdeiWC0hQncjpJ9deN0nhyl8gHBjoiDyN9h9ab&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1)喜欢作者

linux内核13

所有原创240

linux内核 · 目录

上一篇内核该怎么学？Linux进程管理工作原理(代码演示)下一篇Linux驱动小技巧 | 利用DRIVER_ATTR实现调用内核函数

阅读原文

阅读 6510

​
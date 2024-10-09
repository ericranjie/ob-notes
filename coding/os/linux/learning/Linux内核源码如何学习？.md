极客重生

_2022年01月15日 18:22_

以下文章来源于一口Linux ，作者土豆居士

\[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM6Y7VnVXfice2VtFnTr3IEibCiboib78PWNs5M2GeOz265umA/0)

**一口Linux**.

《从零开始学ARM》作者，一起学习嵌入式，Linux，网络，驱动，arm知识。

\](https://mp.weixin.qq.com/s?\_\_biz=MzkyMTIzMTkzNA==&mid=2247567433&idx=1&sn=d74705f6125ed79a81ff4cd31e16d7a4&chksm=c1853f18f6f2b60ec6df47fc74267a752ee50d5dc4fe6433ff5e5811eb58824d8ce6cdf19d70&mpshare=1&scene=24&srcid=0115GDLl7shoD7FUCndTzo1N&sharer_sharetime=1642243204093&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d039cd4eb16a414dfb1716284070cc40dda6f5333e2477ed886c7083f2c29f7bacb2b2bb663a4e2c6c13cd9a2cdd690b042513890b99c273c7cb6e0dca217fba45cd83c73349e917f63f7fb0dc30d1ac71c13201f49bc68cc21148482c1987e8dc4e8e854e6b8f93a38ae4bb7fccebfa78505eacdf756bd346&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQlbIPrd66tXc%2B4y4go8%2FPbRLmAQIE97dBBAEAAAAAAA%2FQN5LzkXAAAAAOpnltbLcz9gKNyK89dVj0YwfFeZCk35ZrspQ%2FgDmqZZZrnUp2NqAqubkMXbu3iVk%2FjLXT%2Bg6TXiBUlc80v7Z1ngPe1rbeI%2BjCciPjLrrffpw9u69YRiX2f8XaV3FM4qsqvp8gdlWwaKUUznRf%2F1Oxd6RyuSJWMdYOV4A25ay%2Bq4d%2BCFpwGrBG9eVvzVzPFdZ4hZjh8MZ%2BptdPqpJIVGP1TIk%2BWRLVjo0V%2BrIsG%2BStMSoOH4i5FdUYQiEZtInM9hY2tt0ufPNukJs0Pb9CKx5a&acctmode=0&pass_ticket=2DLfQD8GeGxGHZdoNQcttshKm8F2%2BHo4br%2Bnko460ER45Gfa4PI1XmxSjeVMICmG&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1#)

**码中自有颜如玉！**

**码中自有黄金屋！**

**那么Linux内核代码到底有多少行？**

**我们怎么读完呢？**

**https://www.kernel.org**

## 一、内核行数

Linux内核分为CPU调度、内存管理、网络和存储四大子系统，针对硬件的驱动成百上千。代码的数量更是大的惊人。

先说说最早的内核linux 0.11，下面这本书可以说很多驱动工程师都学习过，我花了大概1个半月，勉强看了一遍。

![图片](https://mmbiz.qpic.cn/mmbiz_png/icRxcMBeJfcibGMuiamgfuIcyw97XQQWtEUM1u3g3uOr0ibYQo8TPIGSBSJy144zvJbTWcpNbauiczEL7eRlAE1UDIA/640?wx_fmt=png&wxfrom=13&tp=wxpic)

再来看看内核代码量的统计。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

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

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Linux内核Git源码树中的代码达到了2780万行，核心代码只有2%是由李纳斯•托瓦兹自己编写的，其他均是其他个人和组织贡献的，李纳斯•托瓦兹公开了Linux但保留了选择新代码和需要合并的新方法的最终裁定权。

除了Linus Torvalds，对内核贡献最多的是David S.Miller、 Mark Brown、Takashi Iwai、Arnd Bergmann、Al Viro和Mauro Carvalho Chehab。

而参与贡献的公司，从域名统计来看，谷歌、Intel与Red Hat排在了最前列。

## 二、内核目录文件大小

然而，现在的内核已经膨胀的不成样子了，以还不算最新的linux-4.1.15为例：

整个内核源码一共约 793M：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

驱动代码占了大概一半，大约380M:

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

体系相关的代码大约134M：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

网路子系统相关的代码26M：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

文件系统相关的代码37M:

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

linux内核核心代码大约6.8M:

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这些目录任意一个目录想完全看明白都非常不容易。

**内核源码获取**

最新源码离线下载：

**https://www.kernel.org**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

一个在线浏览的内核源码网站(可以浏览2.6.11~latest的源码)：\
https://elixir.bootlin.com/linux/v5.16/source

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

[技术分享|深入理解Linux操作系统](http://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247567263&idx=1&sn=87f4449bb7005cf77cbed9d3d55e6f17&chksm=c18530cef6f2b9d89096312c5b1686421a1f15ac084750d9ec05ce9f0b5959e83bfb6259f6e4&scene=21#wechat_redirect)

## 三、内核子系统

**什么是内核：**

在计算机科学中是一个用来管理软件发出的数据I/O（输入与输出）要求的计算机程序，将这些要求转译为数据处理的指令并交由中央处理器（CPU）及计算机中其他电子组件进行处理，是现代操作系统中最基本的部分。!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

它是为众多应用程序提供对计算机硬件的安全访问的一部分软件，这种访问是有限的，并由内核决定一个程序在什么时候对某部分硬件操作多长时间。

linux内核代码涉及知识点包括汇编指令、c语言、硬件组成原理、操作系统、数据结构和算法、各种外设总线、驱动、网络协议栈。

直接对硬件操作是非常复杂的。所以内核通常提供一种硬件抽象的方法，来完成这些操作。

通过进程间通信机制及系统调用，应用进程可间接控制所需的硬件资源（特别是处理器及IO设备）。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

最上面是用户（或应用程序）空间。这是用户应用程序执行的地方。用户空间之下是内核空间，Linux 内核正是位于这里。

GNU C Library （glibc）也在这里。它提供了连接内核的系统调用接口，还提供了在用户空间应用程序和内核之间进行转换的机制。

内核和用户空间的应用程序使用的是不同的保护地址空间。

每个用户空间的进程都使用自己的虚拟地址空间，而内核则占用单独的地址空间。

Linux 内核可以进一步划分成 3 层。最上面是系统调用接口，它实现了一些基本的功能，例如 read 和 write。

系统调用接口之下是内核代码，可以更精确地定义为独立于体系结构的内核代码。这些代码是 Linux 所支持的所有处理器体系结构所通用的。

在这些代码之下是依赖于体系结构的代码，构成了通常称为 BSP（Board Support Package）的部分。这些代码用作给定体系结构的处理器和特定于平台的代码。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

内核主要系统包括：SCI：系统调用接口 PM：进程管理 VFS：虚拟文件系统 MM：内存管理 Network Stack：内核协议栈 Arch：体系架构 DD：设备驱动

### 1 系统调用接口

SCI 层提供了某些机制执行从用户空间到内核的函数调用。这个接口依赖于体系结构，甚至在相同的处理器家族内也是如此。

SCI 实际上是一个非常有用的函数调用多路复用和多路分解服务。

在 ./linux/kernel 中您可以找到 SCI 的实现，并在 ./linux/arch 中找到依赖于体系结构的部分。

### 2 进程管理

进程管理的重点是进程的执行。

在内核中，这些进程称为线程，代表了单独的处理器虚拟化（线程代码、数据、堆栈和 CPU 寄存器）。

在用户空间，通常使用进程 这个术语，不过 Linux 实现并没有区分这两个概念（进程和线程）。

内核通过 SCI 提供了一个应用程序编程接口（API）来创建一个新进程（fork、exec 或 Portable Operating System Interface \[POSIX\] 函数），停止进程（kill、exit），并在它们之间进行通信和同步（signal 或者 POSIX 机制）。

### 3 内存管理

内核所管理的另外一个重要资源是内存。为了提高效率，如果由硬件管理虚拟内存，内存是按照所谓的内存页方式进行管理的（对于大部分体系结构来说都是 4KB）。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Linux 包括了管理可用内存的方式，以及物理和虚拟映射所使用的硬件机制。

### 4 虚拟文件系统

虚拟文件系统（VFS）是 Linux 内核中非常有用的一个方面，因为它为文件系统提供了一个通用的接口抽象。VFS 在 SCI 和内核所支持的文件系统之间提供了一个交换层。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在 VFS 上面，是对诸如 open、close、read 和 write 之类的函数的一个通用 API 抽象。在 VFS 下面是文件系统抽象，它定义了上层函数的实现方式。

它们是给定文件系统（超过 50 个）的插件。文件系统的源代码可以在 ./linux/fs 中找到。

文件系统层之下是缓冲区缓存，它为文件系统层提供了一个通用函数集（与具体文件系统无关）。

这个缓存层通过将数据保留一段时间（或者随即预先读取数据以便在需要是就可用）优化了对物理设备的访问。缓冲区缓存之下是设备驱动程序，它实现了特定物理设备的接口。

### 5 网络堆栈

网络堆栈在设计上遵循模拟协议本身的分层体系结构。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

回想一下，Internet Protocol (IP) 是传输协议（通常称为传输控制协议或 TCP）下面的核心网络层协议。TCP 上面是 socket 层，它是通过 SCI 进行调用的。

socket 层是网络子系统的标准 API，它为各种网络协议提供了一个用户接口。

从原始帧访问到 IP 协议数据单元（PDU），再到 TCP 和 User Datagram Protocol (UDP)，socket 层提供了一种标准化的方法来管理连接，并在各个终点之间移动数据。内核中网络源代码可以在 ./linux/net 中找到。

### 6 设备驱动程序

Linux 内核中有大量代码都在设备驱动程序中，它们能够运转特定的硬件设备。

Linux 源码树提供了一个驱动程序子目录，这个目录又进一步划分为各种支持设备，例如 Bluetooth、I2C、serial 等。设备驱动程序的代码可以在 ./linux/drivers 中找到。

下面这个图形象的讲解了Linux内核都有哪些东西！!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 四、如何学习内核？

### 1. 学习主线

linux内核源码大而全，一个人，即使再聪明、再有精力，也不可能完全看完、看懂所有的linux内核源码。

一口君建议按照以下主线进行深入研究：

- linux驱动架构

- linux网络子系统

- linux内核启动过程

- linux内存管理机制

- linux调度器

- linux进程管理

- linux虚拟机制(kvm)

- linux内核实时化技术

之前分享：[技术分享|深入理解Linux操作系统](http://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247567263&idx=1&sn=87f4449bb7005cf77cbed9d3d55e6f17&chksm=c18530cef6f2b9d89096312c5b1686421a1f15ac084750d9ec05ce9f0b5959e83bfb6259f6e4&scene=21#wechat_redirect)

沿着某一个主线，深入进去，在研究清楚这个主线的同时，向其他的主线扩展、渗透和学习。

> 此处之所以将驱动列为学习内核的入口，是因为内核为很多外设驱动实现了架构， 比如I2C、SPI、UART、PCIE、字符设备、网络设备、块设备， 我们可以从最基本的字符设备学起， 学习如何编写一个简单的模块 学习如何如何为一些简单的设备比如LED、KEY、ADC等编写驱动 可以说驱动是我们学习内核最简单的入口，

由点到线、由线到面、由面到体，层层深入、不断精进，是学习linux内核源码的一个有效的方法。

### 2. 代码阅读工具

对于代码阅读方法从两个角度来介绍，一个方面是需要选择一个比较有效阅读代码的工具。

一口君强烈推荐：source insight这款阅读代码神器！

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

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

### 4. 学习Linux最重要的是培养自己写代码的能力和对Linux框架结构的了解

Linux内核中绝大部分代码都是由这个地球上顶尖的技术大牛所编写，

这些代码的高内聚低耦合，

其精准度，简洁度、质量都相当的高，

每每看到一段高质量的代码，

一口君都会被那一行行枯燥的代码背后隐藏的设计思想所震撼，所折服！

**阅读内核的代码简直就是在欣赏艺术品！**

很多粉丝问我如何提高自己的C语言编程水平， 一口君不厌其烦的 重复着同样一句话：**看Linux内核！**

**代码中自由颜如玉！代码中自有黄金屋！**

时刻保持激情，任性和耐性！

从量变到质变！

水滴石穿！

愿各位都能够熟练掌握Linux，

![](http://mmbiz.qpic.cn/mmbiz_png/icRxcMBeJfc8535w2vKlsLPf5hwdMjpYrzuVCHx0rcQmvv8rYqTFtIyic5qErtciaibqaIOWgeKkDsOMeae4HciaUaw/300?wx_fmt=png&wxfrom=19)

**一口Linux**

《从零开始学ARM》作者，一起学习嵌入式，Linux，网络，驱动，arm知识。

300篇原创内容

公众号

- END -

______________________________________________________________________

**看完一键三连********在看******\*\*，**转发**\*\*，\*\*\*\*****点赞****

**是对文章最大的赞赏，极客重生感谢你**\*\*!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)\*\*

推荐阅读

[技术分享|深入理解Linux操作系统](http://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247567263&idx=1&sn=87f4449bb7005cf77cbed9d3d55e6f17&chksm=c18530cef6f2b9d89096312c5b1686421a1f15ac084750d9ec05ce9f0b5959e83bfb6259f6e4&scene=21#wechat_redirect)

\[

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

大厂后台开发基本功修炼路线和经典资料

\](https://mp.weixin.qq.com/s?\_\_biz=MzkyMTIzMTkzNA==&mid=2247566986&idx=1&sn=40c3846a5a328ba18b73a16d329b756f&chksm=c18531dbf6f2b8cd4a93bd9aa5cad4020daf8da7a54156f754f5c2b403c58b02b40a79ed79f4&scene=21#wechat_redirect)

你好，这里是极客重生，我是阿荣，大家都叫我荣哥，从华为->外企->到互联网大厂，目前是大厂资深工程师，多次获得五星员工，多年职场经验，技术扎实，专业后端开发和后台架构设计，热爱底层技术，丰富的实战经验，分享技术的本质原理，希望帮助更多人蜕变重生，拿BAT大厂offer，培养高级工程师能力，成为技术专家，实现高薪梦想，期待你的关注！[**点击蓝字查看我的成长之路**。](http://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247564006&idx=1&sn=8c88b0d7222ce0e310a012e90bff961f&chksm=c1850db7f6f284a133be27ad27ecd965c3c942583f69bd1a26c34ebf179b4cd6ceb3d7e787b4&scene=21#wechat_redirect)

校招/社招/简历/面试技巧/大厂技术栈分析/后端开发进阶/优秀开源项目/直播分享/技术视野/实战高手等, [极客星球](http://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247563979&idx=1&sn=b26f56f933d9c9e2fd1fe3260362d6e5&chksm=c1850d9af6f2848cf5e3282aa6464b79fc32d848bb8d8155f317c3be7a021a19ac937f8dd10b&scene=21#wechat_redirect)希望成为最有技术价值星球，尽最大努力为星球的同学提供技术和成长帮助！详情查看->[极客星球](http://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247563979&idx=1&sn=b26f56f933d9c9e2fd1fe3260362d6e5&chksm=c1850d9af6f2848cf5e3282aa6464b79fc32d848bb8d8155f317c3be7a021a19ac937f8dd10b&scene=21#wechat_redirect)

求点赞，在看，分享三连!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

阅读 2901

​

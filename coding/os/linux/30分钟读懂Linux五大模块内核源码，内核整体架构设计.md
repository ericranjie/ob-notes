

土豆居士 一口Linux

 _2021年12月03日 11:49_

击上方“**一口Linux**”，选择“**星标公众号**”

  

# 

干货福利，第一时间送达！

# ![Image](https://mmbiz.qpic.cn/mmbiz/cZV2hRpuAPiaJQXWGyC9wrUzIicibgXayrgibTYarT3A1yzttbtaO0JlV21wMqroGYT3QtPq2C7HMYsvicSB2p7dTBg/640?wx_fmt=gif&tp=wxpic&wxfrom=5&wx_lazy=1 "动态黑色音符")

一、前言

本文是“Linux内核源码分析”系列的专业，会以内核的核心功能为出发点，描述Linux内核的整体架构，以及架构之下主要的软件子系统。之后，会介绍Linux内核源文件的目录结构，并和各个软件子系统对应。

注：本文和其它的“Linux内核分析”文章都基于如下约定：  
a) 内核版本为Linux 5.6.18，可以从下面的链接获取：  
https://mirrors.edge.kernel.org/pub/linux/kernel/v5.x/linux-5.6.18.tar.xz  
b) 鉴于嵌入式系统大多使用ARM处理器，因此涉及到体系结构部分的内容，都以ARM为分析对象

二、 Linux内核的核心功能

如下图所示，Linux内核只是Linux操作系统一部分。对下，它管理系统的所有硬件设备；对上，它通过系统调用，向Library Routine（例如C库）或者其它应用程序提供接口。

![Image](https://mmbiz.qpic.cn/mmbiz_png/icRxcMBeJfc9Foufmia0cI7xX6AibBCQibzK6iaib1oqzbza2iccr6EsR6Sic0Aibzic5TT5XiaTheUyBb6knn4ey4yxwUxlw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

  

因此，其核心功能就是：管理硬件设备，供应用程序使用。而现代计算机（无论是PC还是嵌入式系统）的标准组成，就是CPU、Memory（内存和外存）、输入输出设备、网络设备和其它的外围设备。所以为了管理这些设备，Linux内核提出了如下的架构。

（代码免费获取后台私信【代码】）

三、Linux内核的整体架构

3.1 整体架构和子系统划分

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

上图说明了Linux内核的整体架构。根据内核的核心功能，Linux内核提出了5个子系统，分别负责如下的功能：

1. Process Scheduler，也称作进程管理、进程调度。负责管理CPU资源，以便让各个进程可以以尽量公平的方式访问CPU。

2. Memory Manager，内存管理。负责管理Memory（内存）资源，以便让各个进程可以安全地共享机器的内存资源。另外，内存管理会提供虚拟内存的机制，该机制可以让进程使用多于系统可用Memory的内存，不用的内存会通过文件系统保存在外部非易失存储器中，需要使用的时候，再取回到内存中。

3. VFS（Virtual File System），虚拟文件系统。Linux内核将不同功能的外部设备，例如Disk设备（硬盘、磁盘、NAND Flash、Nor Flash等）、输入输出设备、显示设备等等，抽象为可以通过统一的文件操作接口（open、close、read、write等）来访问。这就是Linux系统“一切皆是文件”的体现（其实Linux做的并不彻底，因为CPU、内存、网络等还不是文件，如果真的需要一切皆是文件，还得看贝尔实验室正在开发的"Plan 9”的）。

4. Network，网络子系统。负责管理系统的网络设备，并实现多种多样的网络标准。

5. IPC（Inter-Process Communication），进程间通信。IPC不管理任何的硬件，它主要负责Linux系统中进程之间的通信。

3.2 进程调度（Process Scheduler)

进程调度是Linux内核中最重要的子系统，它主要提供对CPU的访问控制。因为在计算机中，CPU资源是有限的，而众多的应用程序都要使用CPU资源，所以需要“进程调度子系统”对CPU进行调度管理。

进程调度子系统包括4个子模块（见下图），它们的功能如下：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

1. Scheduling Policy，实现进程调度的策略，它决定哪个（或哪几个）进程将拥有CPU。

2. Architecture-specific Schedulers，体系结构相关的部分，用于将对不同CPU的控制，抽象为统一的接口。这些控制主要在suspend和resume进程时使用，牵涉到CPU的寄存器访问、汇编指令操作等。

3. Architecture-independent Scheduler，体系结构无关的部分。它会和“Scheduling Policy模块”沟通，决定接下来要执行哪个进程，然后通过“Architecture-specific Schedulers模块”resume指定的进程。

4. System Call Interface，系统调用接口。进程调度子系统通过系统调用接口，将需要提供给用户空间的接口开放出去，同时屏蔽掉不需要用户空间程序关心的细节。

3.3 内存管理（Memory Manager, MM)

内存管理同样是Linux内核中最重要的子系统，它主要提供对内存资源的访问控制。Linux系统会在硬件物理内存和进程所使用的内存（称作虚拟内存）之间建立一种映射关系，这种映射是以进程为单位，因而不同的进程可以使用相同的虚拟内存，而这些相同的虚拟内存，可以映射到不同的物理内存上。

内存管理子系统包括3个子模块（见下图），它们的功能如下：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

1. Architecture Specific Managers，体系结构相关部分。提供用于访问硬件Memory的虚拟接口。

2. Architecture Independent Manager，体系结构无关部分。提供所有的内存管理机制，包括：以进程为单位的memory mapping；虚拟内存的Swapping。

3. System Call Interface，系统调用接口。通过该接口，向用户空间程序应用程序提供内存的分配、释放，文件的map等功能。

3.4 虚拟文件系统（Virtual Filesystem, VFS）

传统意义上的文件系统，是一种存储和组织计算机数据的方法。它用易懂、人性化的方法（文件和目录结构），抽象计算机磁盘、硬盘等设备上冰冷的数据块，从而使对它们的查找和访问变得容易。因而文件系统的实质，就是“存储和组织数据的方法”，文件系统的表现形式，就是“从某个设备中读取数据和向某个设备写入数据”。

随着计算机技术的进步，存储和组织数据的方法也是在不断进步的，从而导致有多种类型的文件系统，例如FAT、FAT32、NTFS、EXT2、EXT3等等。而为了兼容，操作系统或者内核，要以相同的表现形式，同时支持多种类型的文件系统，这就延伸出了虚拟文件系统（VFS）的概念。

VFS的功能就是管理各种各样的文件系统，屏蔽它们的差异，以统一的方式，为用户程序提供访问文件的接口。

我们可以从磁盘、硬盘、NAND Flash等设备中读取或写入数据，因而最初的文件系统都是构建在这些设备之上的。这个概念也可以推广到其它的硬件设备，例如内存、显示器（LCD）、键盘、串口等等。

我们对硬件设备的访问控制，也可以归纳为读取或者写入数据，因而可以用统一的文件操作接口访问。Linux内核就是这样做的，除了传统的磁盘文件系统之外，它还抽象出了设备文件系统、内存文件系统等等。这些逻辑，都是由VFS子系统实现。

VFS子系统包括6个子模块（见下图），它们的功能如下：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

1. Device Drivers，设备驱动，用于控制所有的外部设备及控制器。由于存在大量不能相互兼容的硬件设备（特别是嵌入式产品），所以也有非常多的设备驱动。因此，Linux内核中将近一半的Source Code都是设备驱动，大多数的Linux底层工程师（特别是国内的企业）都是在编写或者维护设备驱动，而无暇估计其它内容（它们恰恰是Linux内核的精髓所在）。

2. Device Independent Interface， 该模块定义了描述硬件设备的统一方式（统一设备模型），所有的设备驱动都遵守这个定义，可以降低开发的难度。同时可以用一致的形势向上提供接口。

3. Logical Systems，每一种文件系统，都会对应一个Logical System（逻辑文件系统），它会实现具体的文件系统逻辑。

4. System Independent Interface，该模块负责以统一的接口（快设备和字符设备）表示硬件设备和逻辑文件系统，这样上层软件就不再关心具体的硬件形态了。

5. System Call Interface，系统调用接口，向用户空间提供访问文件系统和硬件设备的统一的接口。

3.5 网络子系统（Net）

网络子系统在Linux内核中主要负责管理各种网络设备，并实现各种网络协议栈，最终实现通过网络连接其它系统的功能。在Linux内核中，网络子系统几乎是自成体系，它包括5个子模块（见下图），它们的功能如下：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

1. Network Device Drivers，网络设备的驱动，和VFS子系统中的设备驱动是一样的。

2. Device Independent Interface，和VFS子系统中的是一样的。

3. Network Protocols，实现各种网络传输协议，例如IP, TCP, UDP等等。

4. Protocol Independent Interface，屏蔽不同的硬件设备和网络协议，以相同的格式提供接口（socket)。

5. System Call interface，系统调用接口，向用户空间提供访问网络设备的统一的接口。

至于IPC子系统，由于功能比较单纯，这里就不再描述。

四、Linux内核源代码的目录结构

Linux内核源代码包括三个主要部分：

1. 内核核心代码，包括第3章所描述的各个子系统和子模块，以及其它的支撑子系统，例如电源管理、Linux初始化等

2. 其它非核心代码，例如库文件（因为Linux内核是一个自包含的内核，即内核不依赖其它的任何软件，自己就可以编译通过）、固件集合、KVM（虚拟机技术）等

3. 编译脚本、配置文件、帮助文档、版权说明等辅助性文件

下图r所示使用ls命令看到的内核源代码的顶层目录结构，具体描述如下。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

```
include/ ---- 内核头文件，需要提供给外部模块（例如用户空间代码）使用。
```

end

  

  

**一口Linux** 

  

**关注，回复【****1024****】海量Linux资料赠送**

![](http://mmbiz.qpic.cn/mmbiz_png/icRxcMBeJfc8535w2vKlsLPf5hwdMjpYrzuVCHx0rcQmvv8rYqTFtIyic5qErtciaibqaIOWgeKkDsOMeae4HciaUaw/300?wx_fmt=png&wxfrom=19)

**一口Linux**

《从零开始学ARM》作者，一起学习嵌入式，Linux，网络，驱动，arm知识。

300篇原创内容

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

Linux驱动71

Linux驱动 · 目录

上一篇Linux设备树语法规范下一篇Linux驱动 | 手写一个设备树使用的实例

Read more

Reads 3427

​

Comment

**留言 1**

- MTUS® 黄魏峰
    
    2021年12月3日
    
    Like2
    
    与时俱进![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/icRxcMBeJfc8535w2vKlsLPf5hwdMjpYrzuVCHx0rcQmvv8rYqTFtIyic5qErtciaibqaIOWgeKkDsOMeae4HciaUaw/300?wx_fmt=png&wxfrom=18)

一口Linux

2159

1

Comment

**留言 1**

- MTUS® 黄魏峰
    
    2021年12月3日
    
    Like2
    
    与时俱进![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    

已无更多数据
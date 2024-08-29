# [蜗窝科技](http://www.wowotech.net/)

### 慢下来，享受技术。

[![](http://www.wowotech.net/content/uploadfile/201401/top-1389777175.jpg)](http://www.wowotech.net/)

- [博客](http://www.wowotech.net/)
- [项目](http://www.wowotech.net/sort/project)
- [关于蜗窝](http://www.wowotech.net/about.html)
- [联系我们](http://www.wowotech.net/contact_us.html)
- [支持与合作](http://www.wowotech.net/support_us.html)
- [登录](http://www.wowotech.net/admin)

﻿

## 

作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2017-3-30 22:01 分类：[Linux内核分析](http://www.wowotech.net/sort/linux_kenrel)

## 1. 前言

前面文章介绍“[Linux MMC framework](http://www.wowotech.net/tag/mmc)”的时候，涉及到了MMC数据传输，进而不可避免地遭遇了DMA(Direct Memory Access)。因而，择日不如撞日，就开几篇文章介绍Linux的DMA Engine framework吧。

本文是DMA Engine framework分析文章的第一篇，主要介绍DMA controller的概念、术语（从硬件的角度，大部分翻译自kernel的document[1]）。之后，会分别从Provider（DMA controller驱动）和Consumer（其它驱动怎么使用DMA传输数据）两个角度，介绍Linux DMA engine有关的技术细节。

## 2. DMA Engine硬件介绍

DMA是Direct Memory Access的缩写，顾名思义，就是绕开CPU直接访问memory的意思。在计算机中，相比CPU，memory和外设的速度是非常慢的，因而在memory和memory（或者memory和设备）之间搬运数据，非常浪费CPU的时间，造成CPU无法及时处理一些实时事件。因此，工程师们就设计出来一种专门用来搬运数据的器件----DMA控制器，协助CPU进行数据搬运，如下图所示：

[![dma](http://www.wowotech.net/content/uploadfile/201703/761a9244e7916bc8000a8f79dab9669a20170330140311.gif "dma")](http://www.wowotech.net/content/uploadfile/201703/ed79a6952349c83c0a1c9993fdc0fa3b20170330140311.gif)

图片1 DMA示意图

思路很简单，因而大多数的DMA controller都有类似的设计原则，归纳如下[1]。

注1：得益于类似的设计原则，Linux kernel才有机会使用一套framework去抽象DMA engine有关的功能。

#### 2.1 DMA channels

一个DMA controller可以“同时”进行的DMA传输的个数是有限的，这称作DMA channels。当然，这里的channel，只是一个逻辑概念，因为：

> 鉴于总线访问的冲突，以及内存一致性的考量，从物理的角度看，不大可能会同时进行两个（及以上）的DMA传输。因而DMA channel不太可能是物理上独立的通道；
> 
> 很多时候，DMA channels是DMA controller为了方便，抽象出来的概念，让consumer以为独占了一个channel，实际上所有channel的DMA传输请求都会在DMA controller中进行仲裁，进而串行传输；
> 
> 因此，软件也可以基于controller提供的channel（我们称为“物理”channel），自行抽象更多的“逻辑”channel，软件会管理这些逻辑channel上的传输请求。实际上很多平台都这样做了，在DMA Engine framework中，不会区分这两种channel（本质上没区别）。

#### 2.2 DMA request lines

由图片1的介绍可知，DMA传输是由CPU发起的：CPU会告诉DMA控制器，帮忙将xxx地方的数据搬到xxx地方。CPU发完指令之后，就当甩手掌柜了。而DMA控制器，除了负责怎么搬之外，还要决定一件非常重要的事情（特别是有外部设备参与的数据传输）：

> 何时可以开始数据搬运？

因为，CPU发起DMA传输的时候，并不知道当前是否具备传输条件，例如source设备是否有数据、dest设备的FIFO是否空闲等等。那谁知道是否可以传输呢？设备！因此，需要DMA传输的设备和DMA控制器之间，会有几条物理的连接线（称作DMA request，DRQ），用于通知DMA控制器可以开始传输了。

这就是DMA request lines的由来，通常来说，每一个数据收发的节点（称作endpoint），和DMA controller之间，就有一条DMA request line（memory设备除外）。

最后总结：

> DMA channel是Provider（提供传输服务），DMA request line是Consumer（消费传输服务）。在一个系统中DMA request line的数量通常比DMA channel的数量多，因为并不是每个request line在每一时刻都需要传输。

#### 2.3 传输参数

在最简单的DMA传输中，只需为DMA controller提供一个参数----transfer size，它就可以欢快的工作了：

> 在每一个时钟周期，DMA controller将1byte的数据从一个buffer搬到另一个buffer，直到搬完“transfer size”个bytes即可停止。

不过这在现实世界中往往不能满足需求，因为有些设备可能需要在一个时钟周期中，传输指定bit的数据，例如：

> memory之间传输数据的时候，希望能以总线的最大宽度为单位（32-bit、64-bit等），以提升数据传输的效率；  
> 而在音频设备中，需要每次写入精确的16-bit或者24-bit的数据；  
> 等等。

因此，为了满足这些多样的需求，我们需要为DMA controller提供一个额外的参数----transfer width。

另外，当传输的源或者目的地是memory的时候，为了提高效率，DMA controller不愿意每一次传输都访问memory，而是在内部开一个buffer，将数据缓存在自己buffer中：

> memory是源的时候，一次从memory读出一批数据，保存在自己的buffer中，然后再一点点（以时钟为节拍），传输到目的地；  
> memory是目的地的时候，先将源的数据传输到自己的buffer中，当累计一定量的数据之后，再一次性的写入memory。

这种场景下，DMA控制器内部可缓存的数据量的大小，称作burst size----另一个参数。

#### 2.4 scatter-gather

我们在“[Linux MMC framework(2)_host controller driver](http://www.wowotech.net/comm/mmc_host_driver.html)”中已经提过scatter的概念，DMA传输时也有类似概念：

> 一般情况下，DMA传输一般只能处理在物理上连续的buffer。但在有些场景下，我们需要将一些非连续的buffer拷贝到一个连续buffer中（这样的操作称作scatter gather，挺形象的）。
> 
> 对于这种非连续的传输，大多时候都是通过软件，将传输分成多个连续的小块（chunk）。但为了提高传输效率（特别是在图像、视频等场景中），有些DMA controller从硬件上支持了这种操作。  
> 注2：具体怎么支持，和硬件实现有关，这里不再多说（只需要知道有这个事情即可，编写DMA controller驱动的时候，自然会知道怎么做）。

## 3. 总结

本文简单的介绍了DMA传输有关的概念、术语，接下来将会通过下面两篇文章，介绍Linux DMA engine有关的实现细节：

> Linux DMA Engine framework(2)_provider
> 
> Linux DMA Engine framework(3)_consumer

## 4. 参考文档

[1] Documentation/dmaengine/provider.txt

_原创文章，转发请注明出处。蜗窝科技_，[www.wowotech.net](http://www.wowotech.net/linux_kenrel/dma_engine_overview.html)。

标签: [Linux](http://www.wowotech.net/tag/Linux) [Kernel](http://www.wowotech.net/tag/Kernel) [内核](http://www.wowotech.net/tag/%E5%86%85%E6%A0%B8) [framework](http://www.wowotech.net/tag/framework) [dma](http://www.wowotech.net/tag/dma) [engine](http://www.wowotech.net/tag/engine)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [系统休眠（System Suspend）和设备中断处理](http://www.wowotech.net/pm_subsystem/suspend-irq.html) | [蓝牙协议分析(10)_BLE安全机制之LE Encryption](http://www.wowotech.net/bluetooth/le_encryption.html)»

**评论：**

**wangjb**  
2023-09-28 16:02

老板，你好！    
我这里遇到一个dma 直接映射物理内存的问题，是用cma的机制，我发的时间太长了，都还没有 解决，能否帮忙提供思路，非常感谢！！！  
驱动qca-7850调试的结果和我写的关于A函数的内核测试模块的结果一样，我就用我的测试程序描述一下问题。  
  问题1    
           linux 内核中 dma_addr是u64类型，size是size_t类型，  min_not_zero(dev->coherent_dma_mask, dev->bus_dma_limit)的结果是0xffffffff;  
                      1.1 当dma_addr=0x730bb000(alloc_pages_node分配), size=0x1000 ,关系表达式 dma_addr + size - 1 <= min_not_zero(dev->coherent_dma_mask, dev->bus_dma_limit),  
                                最后结果是1.  
                      1.2  当dma_addr=0x600000(dma_alloc_contiguous) size=0x6c0000,关系表达式 dma_addr + size - 1 <=  min_not_zero(dev->coherent_dma_mask, dev->bus_dma_limit),  
                                 最后结果为什么是0.          
  问题2  
                怎么用A函数通过cma机制申请直接映射物理内存？  
                  
   背景描述如下：  
    linux kernel 5.16.0-rc8 ------自己编译的  
    ubuntu 22.04    
      
     我用 dma_alloc_coherent函数分配DMA的相关联内存，结果分配失败，发现__dma_direct_alloc_pages函数调用dma_alloc_contiguous后能得到虚拟地址和DMA物理地址，  
用dma_coherent_ok检查后，发现关系表达式为假。  
  
                   return dma_addr + size - 1 <=  
                                    min_not_zero(dev->coherent_dma_mask, dev->bus_dma_limit);  
，最后释放dma_alloc_contiguous申请的cma的地址空间，最后调用普通的分配内存机制alloc_pages_node，由于此时申请的  
空间（0x6f0000)比较大，就申请失败了。如果申请空间比较小，就能通过alloc_pages_node分配成功。  
相应的内核编译配置如下：  
  
1.Build the Linux kernel  
make menuconfig  
CONFIG_CFG80211_INTERNAL_REGDB=y  
CONFIG_CFG80211=y  
CONFIG_NL80211_TESTMODE=y  
CONFIG_FRAME_WARN=2048  
CONFIG_CMA_SIZE_MBYTES=512  
CONFIG_IRQ_REMAP=y  
CONFIG_DMA_CMA=y  
CONFIG_CMA_SIZE_SEL_MBYTES=y  
CONFIG_CMA_ALIGNMENT=8

[回复](http://www.wowotech.net/linux_kenrel/dma_engine_overview.html#comment-8832)

**[江南书生](http://no/)**  
2017-09-14 09:49

老板，你好！  
    在DMA方面我一直有个疑问，DMA请求总线后，CPU把总线控制权交给了DMA，那么CPU不是就没法从内存读取指令和数据了吗？那么，CPU不就处于干瞪眼的状态吗？这样的话，也没法提高CPU的效率啊，还不然直接让CPU去干DMA的活呢。我的这个想法的bug，在哪里呢？

[回复](http://www.wowotech.net/linux_kenrel/dma_engine_overview.html#comment-6037)

**[wowo](http://www.wowotech.net/)**  
2017-09-14 10:45

@江南书生：关于这个问题，其实是cpu architecture设计里面最核心的问题，不是一两句话可以讨论的清楚的。不过可以从下面几点去思考。  
你的疑问是对的，资源是有限的，肯定有争抢和效率的问题（不仅仅是memory，在memory之前还存在对总线的争抢）。但也不用过于悲观，因为（我们从CPU的视角往下说）：  
1）CPU不是一直在取指和取数。指令执行的过程包取指、译码和执行，译码肯定不需要访问memory，执行访问memory的概率也不会超过50%。  
2）复杂SoC通常有很多级的指令cache和数据cache，在顺序执行的情况下，又可以大大减少CPU访问memory的可能性，降低和DMA冲突、争抢的概率。  
3）复杂SoC的总线通常具有仲裁能力，可以配置CPU、DMA等访问总线的优先级，这很大程度上会影响CPU和DMA对memory的访问，如果不想CPU被影响，可以调高它的优先级，反正DMA传输可以慢慢来（见缝插针，这也是DMA设计的初衷）。  
4）关于CPU和DMA对memory资源的争抢，也可以通过DMA burst size进行调整。  
5）很多时候，系统中的memory（例如DDR），是有并发访问的能力的，可以想象成有多个可并行工作的memory，因此竞争的概率又可以降低了。  
6）最后再强调：DMA的性能和CPU的性能是不可调和的，需要根据实际的应用场景小心的平衡。

[回复](http://www.wowotech.net/linux_kenrel/dma_engine_overview.html#comment-6038)

**[江南书生](http://no/)**  
2017-09-14 10:50

@wowo：好的，谢谢老板，大概了解了

[回复](http://www.wowotech.net/linux_kenrel/dma_engine_overview.html#comment-6039)

**max**  
2020-05-13 11:01

@江南书生：传输过程中cpu和dma都可以访问内存。这里会有个总线仲裁机制。

[回复](http://www.wowotech.net/linux_kenrel/dma_engine_overview.html#comment-7991)

**[bsp](http://www.wowotech.net/)**  
2022-05-09 10:58

@江南书生：内存之上还有cache，cache还有L1/L2/L3。很多时候cpu会从DDR 一次性load一个cacheline的内存（或数据或指令），然后CPU基于cache工作即可，此时就不需要DDR；

[回复](http://www.wowotech.net/linux_kenrel/dma_engine_overview.html#comment-8614)

**向往**  
2017-09-07 20:33

终于知道busrt size的含义了，蜗窝给力；另外，我想问一下dma framework提供的dma_mag_sg是不是就是kernel实现scatter-gather的接口呢？dma_sg_map会将可能会连续的内存块合并成一个，然后供dma传输

[回复](http://www.wowotech.net/linux_kenrel/dma_engine_overview.html#comment-6000)

**chenglei**  
2017-08-31 13:36

Hello,  
  
我的dma_burst_size可以支持1, 4, 8, 16, 32, 64, 128, 256，我应该如何选择呢？  
  
谢谢。

[回复](http://www.wowotech.net/linux_kenrel/dma_engine_overview.html#comment-5961)

**[wowo](http://www.wowotech.net/)**  
2017-08-31 14:14

@chenglei：这要看传输的双方是什么东西，你把它们摆在一张纸的两边，然后思考2分钟：怎么传可以获得最大性能？答案很快就出来了:-)

[回复](http://www.wowotech.net/linux_kenrel/dma_engine_overview.html#comment-5962)

**chenglei**  
2017-08-31 14:19

@wowo：感谢wowo。  
  
目前是UVC device功能，同步传输模式，不考虑性能，就想使数据尽可能快速的传递到host。  
  
默认是32.

[回复](http://www.wowotech.net/linux_kenrel/dma_engine_overview.html#comment-5963)

**[wowo](http://www.wowotech.net/)**  
2017-08-31 14:47

@chenglei：哈哈，快也是性能啊。其实用多大的size，不是很好决定。  
  
假设usb iso传输的数据包大小是256（很有可能比这还大）  
如果burst size是256，传输的行为是这样的：  
   1. dma从USB读取256B数据  
   2. dma将这256B的数据写入到ram  
   ...  
如果burst size是32，传输的行为是这样的：  
   1. dma从USB读取32B的数据  
   2. dma将32B的数据写入memory  
   3. dma再从USB读取32B的数据  
   4. dma再将32B的数据写入memory  
   ...  
   如此反复8次。  
    
那么问题来了，那种方法好呢？  
暂且不管dma和usb设备之间的事情，只看dma和memory之间的交互，是一次写入256B写1次好呢，还是一次写入32B写8次好呢？  
前者，dma抢到总线后，一直把数据写完才放。后者，分批次写入，意味着中间可以把总线放掉给其他人（或者其他dma）使用。  
是不是意味着：前者对自己好，后者对其他人好？那么你想要哪种效果呢？  
  
最后，虽然理论上是这样的，实际上怎么样呢？对系统性能影响大吗？不得而知，你可以试试。试验结果才是最有说服力的。

[回复](http://www.wowotech.net/linux_kenrel/dma_engine_overview.html#comment-5964)

**javier**  
2017-07-19 15:04

精彩文字抽象总结的非常到位，受益匪浅，感谢分享！还有一点疑惑请教， 通常DMA都是访问的连续物理地址，在CPU和DMA互操作的时候需要考虑到cache和虚实地址转换，有没有DMA有能力可以访问虚拟内存和cache的？ （ARM架构）我理解应该是可以的，但是不清楚现有的kernel有没有这样的支持。  
另外， DMA通常都是异步操作，所以每次传输完毕都会有相应的通知机制，如中断，所以会和中断子系统有交互。方便的话可以加入一些这方面的介绍就更完整了。

[回复](http://www.wowotech.net/linux_kenrel/dma_engine_overview.html#comment-5821)

**[wowo](http://www.wowotech.net/)**  
2017-07-19 22:08

@javier：多谢建议，您说的非常对。不过牵涉到memory、中断等事情，比较复杂，还没有功力把它们融会贯通啊。会朝着这个目标努力的~~

[回复](http://www.wowotech.net/linux_kenrel/dma_engine_overview.html#comment-5827)

**[bsp-jiang](http://www.wowotech.net/)**  
2019-12-24 11:24

@javier：cpu这边的虚拟内存dma无法访问吧，页表都不一样，各映射各的。  
cache的话，如果dma具有cache-coherent的能力，是可以访问cache的，因为它要维护cache一致性。比如cpu申请了一块cacheable的内存，然后创建map交于dma搬运，那么dma就要去维护cache的一致性。  
个人理解，错误请改正。

[回复](http://www.wowotech.net/linux_kenrel/dma_engine_overview.html#comment-7793)

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
    
    - [MMC/SD/SDIO介绍](http://www.wowotech.net/basic_tech/mmc_sd_sdio_intro.html)
    - [蓝牙协议分析(8)_BLE安全机制之白名单](http://www.wowotech.net/bluetooth/ble_white_list.html)
    - [DRAM 原理 3 ：DRAM Device](http://www.wowotech.net/basic_tech/321.html)
    - [CFS调度器（1）-基本原理](http://www.wowotech.net/process_management/447.html)
    - [Device Tree（三）：代码分析](http://www.wowotech.net/device_model/dt-code-analysis.html)
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
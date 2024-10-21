
作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2017-10-13 22:20 分类：[内存管理](http://www.wowotech.net/sort/memory_management)

# 1. 前言

我们在那些需要和用户空间交互大量数据的子系统（例如MMC\[1\]、Video、Audio等）中，经常看到scatterlist的影子。对我们这些“非英语母语”的人来说，初见这个词汇，脑袋瞬间就蒙圈了。scatter可翻译成“散开、分散”，list是“列表”的意思，因而scatterlist可翻译为“散列表”。“散列表”又是什么？太抽象了！

之所以抽象，是因为这个词省略了主语----物理内存（Physical memory），加上后，就好理解了多了，既：物理内存的散列表。再通俗一些，就是把一些分散的物理内存，以列表的形式组织起来。那么，也许你会问，有什么用处呢？

当然有用，具体可参考本文后续的介绍。

# 2. scatterlist产生的背景

我没有去考究scatterlist API是在哪个kernel版本中引入的（年代太久远了），凭猜测，我觉得应该和MMU有关。因为在引入MMU之后，linux系统中的软件将不得不面对一个困扰（下文将以图片1中所示的系统架构为例进行说明）：

> 假设在一个系统中（参考下面图片1）有三个模块可以访问memory：CPU、DMA控制器和某个外设。CPU通过MMU以虚拟地址（VA）的形式访问memory；DMA直接以物理地址（PA）的形式访问memory；Device通过自己的IOMMU以设备地址（DA）的形式访问memory。
>
> 然后，某个“软件实体”分配并使用了一片存储空间（参考下面图片2）。该存储空间在CPU视角上（虚拟空间）是连续的，起始地址是va1（实际上，它映射到了3块不连续的物理内存上，我们以pa1,pa2,pa3表示）。
>
> 那么，如果该软件单纯的以CPU视角访问这块空间（操作va1），则完全没有问题，因为MMU实现了连续VA到非连续PA的映射。
>
> 不过，如果软件经过一系列操作后，要把该存储空间交给DMA控制器，最终由DMA控制器将其中的数据搬移给某个外设的时候，由于DMA控制器只能访问物理地址，只能以“不连续的物理内存块”为单位递交（而不是我们所熟悉的虚拟地址）。
>
> 此时，scatterlist就诞生了：为了方便，我们需要使用一个数据结构来描述这一个个“不连续的物理内存块”（起始地址、长度等信息），这个数据结构就是scatterlist（具体可参考下面第3章的说明）。而多个scatterlist组合在一起形成一个表（可以是一个struct scatterlist类型的数组，也可以是kernel帮忙抽象出来的struct sg_table），就可以完整的描述这个虚拟地址了。
>
> 最后，从本质上说：scatterlist（数组）是各种不同地址映射空间（PA、VA、DA、等等）的媒介（因为物理地址是真实的、实在的存在，因而可以作为通用语言），借助它，这些映射空间才能相互转换（例如从VA转到DA）。

![[Pasted image 20241021205233.png]]

图片1 cpu_dma_device_memory

![[Pasted image 20241021205244.png]]

图片2 cpu_view_memory

# 3. scatterlist API介绍

## 3.1 struct scatterlist

struct scatterlist用于描述一个在物理地址上连续的内存块（以page为单位），它的定义位于“include/linux/scatterlist.h”中，如下：

|                                                                                                                                                                                                                                                                                                                                                                    |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| struct scatterlist {  <br>#ifdef CONFIG_DEBUG_SG  <br>        unsigned long   sg_magic;  <br>#endif  <br>        unsigned long   page_link;  <br>        unsigned int    offset;  <br>        unsigned int    length;  <br>        dma_addr_t      dma_address;  <br>#ifdef CONFIG_NEED_SG_DMA_LENGTH  <br>        unsigned int    dma_length;  <br>#endif  <br>}; |

> page_link，指示该内存块所在的页面。bit0和bit1有特殊用途（可参考后面的介绍），因此要求page最低4字节对齐。\
> offset，指示该内存块在页面中的偏移（起始位置）。\
> length，该内存块的长度。
>
> dma_address，该内存块实际的起始地址（PA，相比page更接近我们人类的语言）。\
> dma_length，相应的长度信息。

## 3.2 struct sg_table

在实际的应用场景中，单个的scatterlist是没有多少意义的，我们需要多个scatterlist组成一个数组，以表示在物理上不连续的虚拟地址空间。通常情况下，使用scatterlist功能的模块，会自行维护这个数组（指针和长度），例如\[2\]中所提到的struct mmc_data：

```cpp
struct mmc_data {  
    …  
    unsigned int sg_len;    /* size of scatter list */       
    struct scatterlist *sg; /* I/O scatter list */           
	s32 host_cookie;    /* host private data \*/          
};
```


不过呢，为了使用者可以偷懒，kernel抽象出来了一个简单的数据结构：struct sg_table，帮忙保存scatterlist的数组指针和长度：

|   |
|---|
|struct sg_table {  <br>        struct scatterlist *sgl;        /* the list */  <br>        unsigned int nents;             /* number of mapped entries */  <br>        unsigned int orig_nents;        /* original size of list \*/  <br>};|

其中sgl是内存块数组的首地址，orig_nents是内存块数组的size，nents是有效的内存块个数（可能会小于orig_nents）。

以上心思都比较直接，不过有一点，我们要仔细理解：

scatterlist数组中到底有多少有效内存块呢？这不是一个很直观的事情，主要有如下2个规则决定：

> 1）如果scatterlist数组中某个scatterlist的page_link的bit0为1，表示该scatterlist不是一个有效的内存块，而是一个chain（铰链），指向另一个scatterlist数组。通过这种机制，可以将不同的scatterlist数组链在一起，因为scatterlist也称作chain scatterlist。
>
> 2）如果scatterlist数组中某个scatterlist的page_link的bit1为1，表示该scatterlist是scatterlist数组中最后一个有效内存块（后面的就忽略不计了）。

## 3.3 API介绍

理解了scatterlist的含义之后，再去看“include/linux/scatterlist.h”中的API，就容易多了，例如（简单介绍一下，不再详细分析）：

|   |
|---|
|#define sg_dma_address(sg)      ((sg)->dma_address)<br><br>#ifdef CONFIG_NEED_SG_DMA_LENGTH  <br>#define sg_dma_len(sg)          ((sg)->dma_length)  <br>#else  <br>#define sg_dma_len(sg)          ((sg)->length)  <br>#endif|

> sg_dma_address、sg_dma_len，获取某一个scatterlist的物理地址和长度。

|   |
|---|
|#define sg_is_chain(sg)         ((sg)->page_link & 0x01)  <br>#define sg_is_last(sg)          ((sg)->page_link & 0x02)  <br>#define sg_chain_ptr(sg)        \\  <br>        ((struct scatterlist \*) ((sg)->page_link & ~0x03))|

> sg_is_chain可用来判断某个scatterlist是否为一个chain，sg_is_last可用来判断某个scatterlist是否是sg_table中最后一个scatterlist。
>
> sg_chain_ptr可获取chain scatterlist指向的那个scatterlist。

|   |
|---|
|static inline void sg_assign_page(struct scatterlist \*sg, struct page \*page)  <br>static inline void sg_set_page(struct scatterlist \*sg, struct page \*page,  <br>                               unsigned int len, unsigned int offset)  <br>static inline struct page \*sg_page(struct scatterlist \*sg)  <br>static inline void sg_set_buf(struct scatterlist \*sg, const void \*buf,  <br>                               unsigned int buflen)  <br>  <br>#define for_each_sg(sglist, sg, nr, \_\_i)        \\  <br>        for (\_\_i = 0, sg = (sglist); \_\_i \< (nr); \_\_i++, sg = sg_next(sg))  <br>  <br>static inline void sg_chain(struct scatterlist \*prv, unsigned int prv_nents,  <br>                             struct scatterlist \*sgl)  <br>  <br>static inline void sg_mark_end(struct scatterlist \*sg)  <br>static inline void sg_unmark_end(struct scatterlist \*sg)  <br>  <br>static inline dma_addr_t sg_phys(struct scatterlist \*sg)  <br>static inline void \*sg_virt(struct scatterlist \*sg)|

> sg_assign_page，将page赋给指定的scatterlist（设置page_link字段）。\
> sg_set_page，将page中指定offset、指定长度的内存赋给指定的scatterlist（设置page_link、offset、len字段）。\
> sg_page，获取scatterlist所对应的page指针。\
> sg_set_buf，将指定长度的buffer赋给scatterlist（从虚拟地址中获得page指针、在page中的offset之后，再调用sg_set_page）。
>
> for_each_sg，遍历一个scatterlist数组（sglist）中所有的有效scatterlist（考虑sg_is_chain和sg_is_last的情况）。
>
> sg_chain，将两个scatterlist 数组捆绑在一起。
>
> sg_mark_end、sg_unmark_end，将某个scatterlist 标记（或者不标记）为the last one。
>
> sg_phys、sg_virt，获取某个scatterlist的物理或者虚拟地址。

等等（不再罗列了，感兴趣的同学直接去看代码就行了）。

# 4. 参考文档

\[1\] [Linux MMC framework(2)\_host controller driver](http://www.wowotech.net/comm/mmc_host_driver.html)

\[2\] The chained scatterlist API，[https://lwn.net/Articles/256368/](https://lwn.net/Articles/256368/ "https://lwn.net/Articles/256368/")

原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/memory_management/scatterlist.html)。

标签: [Linux](http://www.wowotech.net/tag/Linux) [Kernel](http://www.wowotech.net/tag/Kernel) [内核](http://www.wowotech.net/tag/%E5%86%85%E6%A0%B8) [scatterlist](http://www.wowotech.net/tag/scatterlist) [sg_table](http://www.wowotech.net/tag/sg_table)

______________________________________________________________________

« [Linux kernel内存管理的基本概念](http://www.wowotech.net/memory_management/concept.html) | [X-026-KERNEL-Linux gpio driver的移植之gpio range](http://www.wowotech.net/x_project/kernel_gpio_driver_porting_2.html)»

**评论：**

**一个网友**\
2021-07-30 09:42

想请教个问题， struct scatterlist指向的单位可以大于1个Page么，比如大页，512个Page，2M内容。

[回复](http://www.wowotech.net/memory_management/scatterlist.html#comment-8262)

**code_搬运工**\
2019-09-27 14:50

dma_address，该内存块实际的起始地址（PA，相比page更接近我们人类的语言）。\
这个表述是否有误？\
我的理解是：\
sg_phys(sg)    ---> 物理地址\
sg_dma_address ---> 总线(DMA)地址  对应着dma_address

[回复](http://www.wowotech.net/memory_management/scatterlist.html#comment-7672)

**XuLiDown**\
2023-03-30 12:49

@code_搬运工：个人感觉这里的表述可能有点问题。sg_dma_address 代表的应该是内存块在总线地址空间中的基地址，dma_length 代表的则是内存块在总线地址空间中的长度

[回复](http://www.wowotech.net/memory_management/scatterlist.html#comment-8762)

**Peter**\
2019-02-21 20:46

是不是有了SG，DMA使用的前提不一定是连续的物理内存了。因为DMA可以通过scatterlist来访问不连续的物理内存

[回复](http://www.wowotech.net/memory_management/scatterlist.html#comment-7199)

**[wowo](http://www.wowotech.net/)**\
2019-02-22 21:32

@Peter：SG其实是一个软件概念，大多是给CPU（运行之上的软件）使用。\
DMA是否必须访问连续的物理内存，要看DMA的具体实现。

[回复](http://www.wowotech.net/memory_management/scatterlist.html#comment-7200)

**1688**\
2018-04-18 20:57

讲得很透彻啊

[回复](http://www.wowotech.net/memory_management/scatterlist.html#comment-6679)

**TGSP**\
2018-01-18 16:51

你好！咨询一下相关的问题，我现在在看USB storage驱动实现，里面用到了sg_miter_start()、sg_miter_skip()、sg_miter_next()、sg_miter_stop()代码实现，可否指点一下Mapping sg iterator是个什么样的结构逻辑么？

[回复](http://www.wowotech.net/memory_management/scatterlist.html#comment-6486)

**小亮**\
2017-10-31 16:02

page_link除了存放page地址+flag标记外，还可以存放sg地址+flag，参见接口sg_chain（）

[回复](http://www.wowotech.net/memory_management/scatterlist.html#comment-6154)

**小王**\
2017-11-28 16:13

@小亮：感谢作者的这篇文章，解答了我对scatterlist的疑惑!

[回复](http://www.wowotech.net/memory_management/scatterlist.html#comment-6252)

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

  - [进程切换分析（2）：TLB处理](http://www.wowotech.net/process_management/context-switch-tlb.html)
  - [关于java单线程经常占用cpu100%分析](http://www.wowotech.net/linux_kenrel/483.html)
  - [tty驱动分析](http://www.wowotech.net/tty_framework/435.html)
  - [linux kernel内存碎片防治技术](http://www.wowotech.net/memory_management/memory-fragment.html)
  - [Linux kernel debug技巧----开启DEBUG选项](http://www.wowotech.net/linux_application/kernel_debug_enable.html)

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

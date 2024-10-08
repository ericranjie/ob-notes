作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2017-5-18 21:56 分类：[Linux内核分析](http://www.wowotech.net/sort/linux_kenrel)
## 1. 前言

本文将从provider的角度，介绍怎样在linux kernel dmaengine的框架下，编写dma controller驱动。
## 2. dma controller驱动的软件框架

设备驱动的本质是描述并抽象硬件，然后为consumer提供操作硬件的友好接口。dma controller驱动也不例外，它要做的事情无外乎是：

> 1）抽象并控制DMA控制器。
> 2）管理DMA channel（可以是物理channel，也可以是虚拟channel，具体可参考[1]中的介绍），并向client driver提供友好、易用的接口。
> 3）以DMA channel为操作对象，响应client driver（consumer）的传输请求，并控制DMA controller，执行传输。

当然，按照惯例，为了统一提供给consumer的API（参考[2]），并减少DMA controller driver的开发难度（从论述题变为填空题），dmaengine framework提供了一套controller driver的开发框架，主要思路是（参考图片1）：

[![dma驱动框架](http://www.wowotech.net/content/uploadfile/201705/8439a73ea45ce217d00b88f1bc1fd39c20170518135633.gif "dma驱动框架")](http://www.wowotech.net/content/uploadfile/201705/aedb6cf06c35b4fcd5577e8f3dd14d0a20170518135633.gif)

图片1 DMA驱动框架

> 1）使用struct dma_device抽象DMA controller，controller driver只要填充该结构中必要的字段，就可以完成dma controller的驱动开发。
> 2）使用struct dma_chan（图片1中的DCn）抽象物理的DMA channel（图片1中的CHn），物理channel和controller所能提供的通道数一一对应。
> 3）基于物理的DMA channel，使用struct virt_dma_cha抽象出虚拟的dma channel（图片1中的VCx）。多个虚拟channel可以共享一个物理channel，并在这个物理channel上进行分时传输。
> 4）基于这些数据结构，提供一些便于controller driver开发的API，供driver使用。

上面三个数据结构的描述，可参考第3章的介绍。然后，我们会在第4章介绍相关的API、controller driver的开发思路和步骤以及dmaengine中和controller driver有关的重要流程。
## 3. 主要数据结构描述

#### 3.1 struct dma_device

用于抽象dma controller的struct dma_device是一个庞杂的数据结构（具体可参考include/linux/dmaengine.h中的代码），不过真正需要dma controller driver关心的内容却不是很多，主要包括：

注1：为了加快对dmaengine framework的理解和掌握，这里只描述一些简单的应用场景，更复杂的场景，只有等到有需求的时候，再更深入的理解。

> channels，一个链表头，用于保存该controller支持的所有dma channel（struct dma_chan，具体可参考3.2小节）。在初始化的时候，dma controller driver首先要调用INIT_LIST_HEAD初始化它，然后调用list_add_tail将所有的channel添加到该链表头中。
> 
> cap_mask，一个bitmap，用于指示该dma controller所具备的能力（可以进行什么样的DMA传输），例如（具体可参考enum dma_transaction_type的定义）：  
>     DMA_MEMCPY，可进行memory copy；  
>     DMA_MEMSET，可进行memory set；  
>     DMA_SG，可进行scatter list传输；  
>     DMA_CYCLIC，可进行cyclic类[2]的传输；  
>     DMA_INTERLEAVE，可进行交叉传输[2]；  
>     等等，等等（各种奇奇怪怪的传输类型，不看不知道，一看吓一跳！！）。  
> 另外，该bitmap的定义，需要和后面device_prep_dma_xxx形式的回调函数对应（bitmap中支持某个传输类型，就必须提供该类型对应的回调函数）。
> 
> src_addr_widths，一个bitmap，表示该controller支持哪些宽度的src类型，包括1、2、3、4、8、16、32、64（bytes）等（具体可参考enum dma_slave_buswidth 的定义）。  
> dst_addr_widths，一个bitmap，表示该controller支持哪些宽度的dst类型，包括1、2、3、4、8、16、32、64（bytes）等（具体可参考enum dma_slave_buswidth 的定义）。
> 
> directions，一个bitmap，表示该controller支持哪些传输方向，包括DMA_MEM_TO_MEM、DMA_MEM_TO_DEV、DMA_DEV_TO_MEM、DMA_DEV_TO_DEV，具体可参考enum dma_transfer_direction的定义和注释，以及[2]中相关的说明。
> 
> max_burst，支持的最大的burst传输的size。有关burst传输的概念可参考[1]。
> 
> descriptor_reuse，指示该controller的传输描述可否可重复使用（client driver可只获取一次传输描述，然后进行多次传输）。
> 
> device_alloc_chan_resources/device_free_chan_resources，client driver申请/释放[2] dma channel的时候，dmaengine会调用dma controller driver相应的alloc/free回调函数，以准备相应的资源。具体要准备哪些资源，则需要dma controller driver根据硬件的实际情况，自行决定（这就是dmaengine framework的流氓之处，呵呵~）。
> 
> device_prep_dma_xxx，同理，client driver通过dmaengine_prep_xxx API获取传输描述符的时候，damengine则会直接回调dma controller driver相应的device_prep_dma_xxx接口。至于要在这些回调函数中做什么事情，dma controller driver自己决定就是了（真懒啊！）。
> 
> device_config，client driver调用dmaengine_slave_config[2]配置dma channel的时候，dmaengine会调用该回调函数，交给dma controller driver处理。
> 
> device_pause/device_resume/device_terminate_all，同理，client driver调用dmaengine_pause、dmaengine_resume、dmaengine_terminate_xxx等API的时候，dmaengine会调用相应的回调函数。
> 
> device_issue_pending，client driver调用dma_async_issue_pending启动传输的时候，会调用调用该回调函数。

总结：dmaengine对dma controller的抽象和封装，只是薄薄的一层：仅封装出来一些回调函数，由dma controller driver实现，被client driver调用，dmaengine本身没有太多的操作逻辑。
#### 3.2 struct dma_chan

struct dma_chan用于抽象dma channel，其内容为：

|   |
|---|
|struct dma_chan {  <br>        struct dma_device *device;  <br>        dma_cookie_t cookie;  <br>        dma_cookie_t completed_cookie;<br><br>        /* sysfs */  <br>        int chan_id;  <br>        struct dma_chan_dev *dev;<br><br>        struct list_head device_node;  <br>        struct dma_chan_percpu __percpu *local;  <br>        int client_count;  <br>        int table_count;<br><br>        /* DMA router */  <br>        struct dma_router *router;  <br>        void *route_data;<br><br>        void *private;  <br>};|

需要dma controller driver关心的字段包括：

> device，指向该channel所在的dma controller。
> 
> cookie，client driver以该channel为操作对象获取传输描述符时，dma controller driver返回给client的最后一个cookie。
> 
> completed_cookie，在这个channel上最后一次完成的传输的cookie。dma controller driver可以在传输完成时调用辅助函数dma_cookie_complete设置它的value。
> 
> device_node，链表node，用于将该channel添加到dma_device的channel列表中。
> 
> router、route_data，TODO。
#### 3.3 struct virt_dma_cha

struct virt_dma_chan用于抽象一个虚拟的dma channel，多个虚拟channel可以共用一个物理channel，并由软件调度多个传输请求，将多个虚拟channel的传输串行地在物理channel上完成。该数据结构的定义如下：

|   |
|---|
|/* drivers/dma/virt-dma.h */  <br>  <br><br>struct virt_dma_desc {  <br>        struct dma_async_tx_descriptor tx;  <br>        /* protected by vc.lock */  <br>        struct list_head node;  <br>};<br><br>struct virt_dma_chan {  <br>        struct dma_chan chan;  <br>        struct tasklet_struct task;  <br>        void (*desc_free)(struct virt_dma_desc *);<br><br>        spinlock_t lock;<br><br>        /* protected by vc.lock */  <br>        struct list_head desc_allocated;  <br>        struct list_head desc_submitted;  <br>        struct list_head desc_issued;  <br>        struct list_head desc_completed;<br><br>        struct virt_dma_desc *cyclic;<br><br>};|

> chan，一个struct dma_chan类型的变量，用于和client driver打交道（屏蔽物理channel和虚拟channel的差异）。
> 
> task，一个tasklet，用于等待该虚拟channel上传输的完成（由于是虚拟channel，传输完成与否只能由软件判断）。
> 
> desc_allocated、desc_submitted、desc_issued、desc_completed，四个链表头，用于保存不同状态的虚拟channel描述符（struct virt_dma_desc，仅仅对struct dma_async_tx_descriptor[2]做了一个简单的封装）。
## 4. dmaengine向dma controller driver提供的API汇整

damengine直接向dma controller driver提供的API并不多（大部分的逻辑交互都位于struct dma_device结构的回调函数中），主要包括：

1）struct dma_device变量的注册和注销接口

|   |
|---|
|/* include/linux/dmaengine.h */  <br>int dma_async_device_register(struct dma_device *device);  <br>void dma_async_device_unregister(struct dma_device *device);|

> dma controller driver准备好struct dma_device变量后，可以调用dma_async_device_register将它（controller）注册到kernel中。该接口会对device指针进行一系列的检查，然后对其做进一步的初始化，最后会放在一个名称为dma_device_list的全局链表上，以便后面使用。
> 
> dma_async_device_unregister，注销接口。

2）cookie有关的辅助接口，位于“drivers/dma/dmaengine.h”中，包括

|   |
|---|
|static inline void dma_cookie_init(struct dma_chan *chan)  <br>static inline dma_cookie_t dma_cookie_assign(struct dma_async_tx_descriptor *tx)  <br>static inline void dma_cookie_complete(struct dma_async_tx_descriptor *tx)  <br>static inline enum dma_status dma_cookie_status(struct dma_chan *chan,  <br>        dma_cookie_t cookie, struct dma_tx_state *state)|

> 由于cookie有关的操作，有很多共性，dmaengine就提供了一些通用实现：
> 
> void dma_cookie_init，初始化dma channel中的cookie、completed_cookie字段。
> 
> dma_cookie_assign，为指针的传输描述（tx）分配一个cookie。
> 
> dma_cookie_complete，当某一个传输（tx）完成的时候，可以调用该接口，更新该传输所对应channel的completed_cookie字段。
> 
> dma_cookie_status，获取指定channel（chan）上指定cookie的传输状态。

3）依赖处理接口

|   |
|---|
|void dma_run_dependencies(struct dma_async_tx_descriptor *tx);|

> 由前面的描述可知，client可以同时提交多个具有依赖关系的dma传输。因此当某个传输结束的时候，dma controller driver需要检查是否有依赖该传输的传输，如果有，则传输之。这个检查并传输的过程，可以借助该接口进行（dma controller driver只需调用即可，省很多事）。

4）device tree有关的辅助接口

|   |
|---|
|extern struct dma_chan *of_dma_simple_xlate(struct of_phandle_args *dma_spec,  <br>                struct of_dma *ofdma);  <br>extern struct dma_chan *of_dma_xlate_by_chan_id(struct of_phandle_args *dma_spec,  <br>                struct of_dma *ofdma);|

> 上面两个接口可用于将client device node中有关dma的字段解析出来，并获取对应的dma channel。后面实际开发的时候会举例说明。

5）虚拟dma channel有关的API

后面会有专门的文章介绍虚拟dma，这里不再介绍。

## 5. 编写一个dma controller driver的方法和步骤

上面啰嗦了这么多，相信大家还是似懂非懂（很正常，我也是，dmaengine framework特点就是框架简单，细节复杂）。到底怎么在dmaengine的框架下编写dma controller驱动呢？现在看来，只靠这篇文章，可能达不到目的了，这里先罗列一下基本步骤，后续我们会结合实际的开发过程，进一步的理解和掌握。

编写一个dma controller driver的基本步骤包括（不考虑虚拟channel的情况）：

1）定义一个struct dma_device变量，并根据实际的硬件情况，填充其中的关键字段。

2）根据controller支持的channel个数，为每个channel定义一个struct dma_chan变量，进行必要的初始化后，将每个channel都添加到struct dma_device变量的channels链表中。

3）根据硬件特性，实现struct dma_device变量中必要的回调函数（device_alloc_chan_resources/device_free_chan_resources、device_prep_dma_xxx、device_config、device_issue_pending等等）。

4）调用dma_async_device_register将struct dma_device变量注册到kernel中。

5）当client driver申请dma channel时（例如通过device tree中的dma节点获取），dmaengine core会调用dma controller driver的device_alloc_chan_resources函数，controller driver需要在这个接口中奖该channel的资源准备好。

6）当client driver配置某个dma channel时，dmaengine core会调用dma controller driver的device_config函数，controller driver需要在这个函数中将client想配置的内容准备好，以便进行后续的传输。

7）client driver开始一个传输之前，会把传输的信息通过dmaengine_prep_slave_xxx接口交给controller driver，controller driver需要在对应的device_prep_dma_xxx回调中，将这些要传输的内容准备好，并返回给client driver一个传输描述符。

8）然后，client driver会调用dmaengine_submit将该传输提交给controller driver，此时dmaengine会调用controller driver为每个传输描述符所提供的tx_submit回调函数，controller driver需要在这个函数中将描述符挂到该channel对应的传输队列中。

9）client driver开始传输时，会调用dma_async_issue_pending，controller driver需要在对应的回调函数（device_issue_pending）中，依次将队列上所有的传输请求提交给硬件。

10）等等。

## 6. 参考文档

[1] [Linux DMA Engine framework(1)_概述](http://www.wowotech.net/linux_kenrel/dma_engine_overview.html)

[2] [Linux DMA Engine framework(2)_功能介绍及解接口分析](http://www.wowotech.net/linux_kenrel/dma_engine_api.html)

_原创文章，转发请注明出处。蜗窝科技_，[www.wowotech.net](http://www.wowotech.net/linux_kenrel/dma_controller_driver.html)。

标签: [driver](http://www.wowotech.net/tag/driver) [controller](http://www.wowotech.net/tag/controller) [framework](http://www.wowotech.net/tag/framework) [dma](http://www.wowotech.net/tag/dma) [engine](http://www.wowotech.net/tag/engine)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [为什么会有文件系统(二)](http://www.wowotech.net/filesystem/396.html) | [Linux的时钟](http://www.wowotech.net/timer_subsystem/clock-id-in-linux.html)»

**评论：**

**秋暮离**  
2017-06-22 11:33

wowo好，  
有些ISP、USB HS、Display等控制模块内嵌有自己的DMA controller，这是出于什么考虑？用系统中的DMA控制器不行吗？

[回复](http://www.wowotech.net/linux_kenrel/dma_controller_driver.html#comment-5718)

**Toni**  
2022-07-27 20:34

@秋暮离：同样的问题

[回复](http://www.wowotech.net/linux_kenrel/dma_controller_driver.html#comment-8657)

**llama**  
2022-11-02 17:01

@Toni：用内嵌DMA是标准做法。用系统的DMA理论上是可以的。但是这样就要每个系统都要做芯片设计上的集成。这样比起即插即用的内嵌DMA，成本上差太多了。

[回复](http://www.wowotech.net/linux_kenrel/dma_controller_driver.html#comment-8695)

**R**  
2017-06-14 18:11

平台不同,好像DMA的操作相差很大.

[回复](http://www.wowotech.net/linux_kenrel/dma_controller_driver.html#comment-5673)

**[wowo](http://www.wowotech.net/)**  
2017-06-14 21:30

@R：是的，所以kernel的dmaengine框架就薄薄的一层。

[回复](http://www.wowotech.net/linux_kenrel/dma_controller_driver.html#comment-5674)

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
    
    - [Concurrency Managed Workqueue之（四）：workqueue如何处理work](http://www.wowotech.net/irq_subsystem/queue_and_handle_work.html)
    - [建立讨论区的原因](http://www.wowotech.net/73.html)
    - [DRAM 原理 1 ：DRAM Storage Cell](http://www.wowotech.net/basic_tech/307.html)
    - [eMMC 原理 3 ：分区管理](http://www.wowotech.net/basic_tech/emmc_partitions.html)
    - [Linux系统如何标识进程？](http://www.wowotech.net/process_management/pid.html)
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
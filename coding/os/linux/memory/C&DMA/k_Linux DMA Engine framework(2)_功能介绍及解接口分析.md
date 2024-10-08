作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2017-5-2 22:47 分类：[Linux内核分析](http://www.wowotech.net/sort/linux_kenrel)
## 1. 前言

从我们的直观感受来说，DMA并不是一个复杂的东西，要做的事情也很单纯直白。因此Linux kernel对它的抽象和实现，也应该简洁、易懂才是。不过现实却不甚乐观（个人感觉），Linux kernel dmaengine framework的实现，真有点晦涩的感觉。为什么会这样呢？

如果一个软件模块比较复杂、晦涩，要么是设计者的功力不够，要么是需求使然。当然，我们不敢对Linux kernel的那些大神们有丝毫怀疑和不敬，只能从需求上下功夫了：难道Linux kernel中的driver对DMA的使用上，有一些超出了我们日常的认知范围？

要回答这些问题并不难，将dmaengine framework为consumers提供的功能和API梳理一遍就可以了，这就是本文的目的。当然，也可以借助这个过程，加深对DMA的理解，以便在编写那些需要DMA传输的driver的时候，可以更游刃有余。

## 2. Slave-DMA API和Async TX API

从方向上来说，DMA传输可以分为4类：memory到memory、memory到device、device到memory以及device到device。Linux kernel作为CPU的代理人，从它的视角看，外设都是slave，因此称这些有device参与的传输（MEM2DEV、DEV2MEM、DEV2DEV）为Slave-DMA传输。而另一种memory到memory的传输，被称为Async TX。

为什么强调这种差别呢？因为Linux为了方便基于DMA的memcpy、memset等操作，在dma engine之上，封装了一层更为简洁的API（如下面图片1所示），这种API就是Async TX API（以async_开头，例如async_memcpy、async_memset、async_xor等）。

[![dma_api](http://www.wowotech.net/content/uploadfile/201705/ed9181303abaaa537b1ac8416bd62ffa20170502144741.gif "dma_api")](http://www.wowotech.net/content/uploadfile/201705/82157c2e1311563d5ea661cec023c30a20170502144740.gif)

图片1 DMA Engine API示意图

最后，因为memory到memory的DMA传输有了比较简洁的API，没必要直接使用dma engine提供的API，最后就导致dma engine所提供的API就特指为Slave-DMA API（把mem2mem剔除了）。

本文主要介绍dma engine为consumers提供的功能和API，因此就不再涉及Async TX API了（具体可参考本站后续的文章。

注1：Slave-DMA中的“slave”，指的是参与DMA传输的设备。而对应的，“master”就是指DMA controller自身。一定要明白“slave”的概念，才能更好的理解kernel dma engine中有关的术语和逻辑。

## 3. dma engine的使用步骤

注2：本文大部分内容翻译自kernel document[1]，喜欢读英语的读者可以自行参考。

对设备驱动的编写者来说，要基于dma engine提供的Slave-DMA API进行DMA传输的话，需要如下的操作步骤：

> 1）申请一个DMA channel。
> 
> 2）根据设备（slave）的特性，配置DMA channel的参数。
> 
> 3）要进行DMA传输的时候，获取一个用于识别本次传输（transaction）的描述符（descriptor）。
> 
> 4）将本次传输（transaction）提交给dma engine并启动传输。
> 
> 5）等待传输（transaction）结束。
> 
> 然后，重复3~5即可。

上面5个步骤，除了3有点不好理解外，其它的都比较直观易懂，具体可参考后面的介绍。

#### 3.1 申请DMA channel

任何consumer（文档[1]中称作client，也可称作slave driver，意思都差不多，不再特意区分）在开始DMA传输之前，都要申请一个DMA channel（有关DMA channel的概念，请参考[2]中的介绍）。

DMA channel（在kernel中由“struct dma_chan”数据结构表示）由provider（或者是DMA controller）提供，被consumer（或者client）使用。对consumer来说，不需要关心该数据结构的具体内容（我们会在dmaengine provider的介绍中在详细介绍）。

consumer可以通过如下的API申请DMA channel：

|   |
|---|
|struct dma_chan *dma_request_chan(struct device *dev, const char *name);|

> 该接口会返回绑定在指定设备（dev）上名称为name的dma channel。dma engine的provider和consumer可以使用device tree、ACPI或者struct dma_slave_map类型的match table提供这种绑定关系，具体可参考**XXXX章节**的介绍。

最后，申请得到的dma channel可以在不需要使用的时候通过下面的API释放掉：

|   |
|---|
|void dma_release_channel(struct dma_chan *chan);|

#### 3.2 配置DMA channel的参数

driver申请到一个为自己使用的DMA channel之后，需要根据自身的实际情况，以及DMA controller的能力，对该channel进行一些配置。可配置的内容由struct dma_slave_config数据结构表示（具体可参考4.1小节的介绍）。driver将它们填充到一个struct dma_slave_config变量中后，可以调用如下API将这些信息告诉给DMA controller：

|   |
|---|
|int dmaengine_slave_config(struct dma_chan *chan, struct dma_slave_config *config)|

#### 3.3 获取传输描述（tx descriptor）

DMA传输属于异步传输，在启动传输之前，slave driver需要将此次传输的一些信息（例如src/dst的buffer、传输的方向等）提交给dma engine（本质上是dma controller driver），dma engine确认okay后，返回一个描述符（由struct dma_async_tx_descriptor抽象）。此后，slave driver就可以以该描述符为单位，控制并跟踪此次传输。

struct dma_async_tx_descriptor数据结构可参考4.2小节的介绍。根据传输模式的不同，slave driver可以使用下面三个API获取传输描述符（具体可参考Documentation/dmaengine/client.txt[1]中的说明）：

|   |
|---|
|struct dma_async_tx_descriptor *dmaengine_prep_slave_sg(  <br>        struct dma_chan *chan, struct scatterlist *sgl,  <br>        unsigned int sg_len, enum dma_data_direction direction,  <br>        unsigned long flags);<br><br>struct dma_async_tx_descriptor *dmaengine_prep_dma_cyclic(  <br>        struct dma_chan *chan, dma_addr_t buf_addr, size_t buf_len,  <br>        size_t period_len, enum dma_data_direction direction);<br><br>struct dma_async_tx_descriptor *dmaengine_prep_interleaved_dma(  <br>        struct dma_chan *chan, struct dma_interleaved_template *xt,  <br>        unsigned long flags);|

dmaengine_prep_slave_sg用于在“scatter gather buffers”列表和总线设备之间进行DMA传输，参数如下：  
注3：有关scatterlist 我们在[3][2]中有提及，后续会有专门的文章介绍它，这里暂且按下不表。

> chan，本次传输所使用的dma channel。
> 
> sgl，要传输的“scatter gather buffers”数组的地址；  
> sg_len，“scatter gather buffers”数组的长度。
> 
> direction，数据传输的方向，具体可参考enum dma_data_direction （include/linux/dma-direction.h）的定义。
> 
> flags，可用于向dma controller driver传递一些额外的信息，包括（具体可参考enum dma_ctrl_flags中以DMA_PREP_开头的定义）：  
> DMA_PREP_INTERRUPT，告诉DMA controller driver，本次传输完成后，产生一个中断，并调用client提供的回调函数（可在该函数返回后，通过设置struct dma_async_tx_descriptor指针中的相关字段，提供回调函数，具体可参考4.2小节的介绍）；  
> DMA_PREP_FENCE，告诉DMA controller driver，后续的传输，依赖本次传输的结果（这样controller driver就会小心的组织多个dma传输之间的顺序）；  
> DMA_PREP_PQ_DISABLE_P、DMA_PREP_PQ_DISABLE_Q、DMA_PREP_CONTINUE，PQ有关的操作，TODO。

dmaengine_prep_dma_cyclic常用于音频等场景中，在进行一定长度的dma传输（buf_addr&buf_len）的过程中，每传输一定的byte（period_len），就会调用一次传输完成的回调函数，参数包括：

> chan，本次传输所使用的dma channel。
> 
> buf_addr、buf_len，传输的buffer地址和长度。
> 
> period_len，每隔多久（单位为byte）调用一次回调函数。需要注意的是，buf_len应该是period_len的整数倍。
> 
> direction，数据传输的方向。

dmaengine_prep_interleaved_dma可进行不连续的、交叉的DMA传输，通常用在图像处理、显示等场景中，具体可参考struct dma_interleaved_template结构的定义和解释（这里不再详细介绍，需要用到的时候，再去学习也okay）。

#### 3.4 启动传输

通过3.3章节介绍的API获取传输描述符之后，client driver可以通过dmaengine_submit接口将该描述符放到传输队列上，然后调用dma_async_issue_pending接口，启动传输。

dmaengine_submit的原型如下：

|   |
|---|
|dma_cookie_t dmaengine_submit(struct dma_async_tx_descriptor *desc)|

> 参数为传输描述符指针，返回一个唯一识别该描述符的cookie，用于后续的跟踪、监控。

dma_async_issue_pending的原型如下：

|   |
|---|
|void dma_async_issue_pending(struct dma_chan *chan);|

> 参数为dma channel,无返回值。

注4：由上面两个API的特征可知，kernel dma engine鼓励client driver一次提交多个传输，然后由kernel（或者dma controller driver）统一完成这些传输。

#### 3.5 等待传输结束

传输请求被提交之后，client driver可以通过回调函数获取传输完成的消息，当然，也可以通过dma_async_is_tx_complete等API，测试传输是否完成。不再详细说明了。

最后，如果等不及了，也可以使用dmaengine_pause、dmaengine_resume、dmaengine_terminate_xxx等API，暂停、终止传输，具体请参考kernel document[1]以及source code。

## 4. 重要数据结构说明

#### 4.1 struct dma_slave_config

中包含了完成一次DMA传输所需要的所有可能的参数，其定义如下：

|   |
|---|
|/* include/linux/dmaengine.h */  <br>  <br>struct dma_slave_config {  <br>        enum dma_transfer_direction direction;  <br>        phys_addr_t src_addr;  <br>        phys_addr_t dst_addr;  <br>        enum dma_slave_buswidth src_addr_width;  <br>        enum dma_slave_buswidth dst_addr_width;  <br>        u32 src_maxburst;  <br>        u32 dst_maxburst;  <br>        bool device_fc;  <br>        unsigned int slave_id;  <br>};|

> direction，指明传输的方向，包括（具体可参考enum dma_transfer_direction的定义和注释）：  
>     DMA_MEM_TO_MEM，memory到memory的传输；  
>     DMA_MEM_TO_DEV，memory到设备的传输；  
>     DMA_DEV_TO_MEM，设备到memory的传输；  
>     DMA_DEV_TO_DEV，设备到设备的传输。  
>     注5：controller不一定支持所有的DMA传输方向，具体要看provider的实现。  
>     注6：参考第2章的介绍，MEM to MEM的传输，一般不会直接使用dma engine提供的API。
> 
> src_addr，传输方向是dev2mem或者dev2dev时，读取数据的位置（通常是固定的FIFO地址）。对mem2dev类型的channel，不需配置该参数（每次传输的时候会指定）；  
> dst_addr，传输方向是mem2dev或者dev2dev时，写入数据的位置（通常是固定的FIFO地址）。对dev2mem类型的channel，不需配置该参数（每次传输的时候会指定）；  
> src_addr_width、dst_addr_width，src/dst地址的宽度，包括1、2、3、4、8、16、32、64（bytes）等（具体可参考enum dma_slave_buswidth 的定义）。
> 
> src_maxburst、dst_maxburst，src/dst最大可传输的burst size（可参考[2]中有关burst size的介绍），单位是src_addr_width/dst_addr_width（注意，不是byte）。
> 
> device_fc，当外设是Flow Controller（流控制器）的时候，需要将该字段设置为true。CPU中有关DMA和外部设备之间连接方式的设计中，决定DMA传输是否结束的模块，称作flow controller，DMA controller或者外部设备，都可以作为flow controller，具体要看外设和DMA controller的设计原理、信号连接方式等，不在详细说明(感兴趣的同学可参考[4]中的介绍)。
> 
> slave_id，外部设备通过slave_id告诉dma controller自己是谁（一般和某个request line对应）。很多dma controller并不区分slave，只要给它src、dst、len等信息，它就可以进行传输，因此slave_id可以忽略。而有些controller，必须清晰地知道此次传输的对象是哪个外设，就必须要提供slave_id了（至于怎么提供，可dma controller的硬件以及驱动有关，要具体场景具体对待）。

#### 4.2 struct dma_async_tx_descriptor

传输描述符用于描述一次DMA传输（类似于一个文件句柄）。client driver将自己的传输请求通过3.3中介绍的API提交给dma controller driver后，controller driver会返回给client driver一个描述符。

client driver获取描述符后，可以以它为单位，进行后续的操作（启动传输、等待传输完成、等等）。也可以将自己的回调函数通过描述符提供给controller driver。

传输描述符的定义如下：

|   |
|---|
|struct dma_async_tx_descriptor {  <br>         dma_cookie_t cookie;  <br>         enum dma_ctrl_flags flags; /* not a 'long' to pack with cookie */  <br>         dma_addr_t phys;  <br>         struct dma_chan *chan;  <br>         dma_cookie_t (*tx_submit)(struct dma_async_tx_descriptor *tx);  <br>         int (*desc_free)(struct dma_async_tx_descriptor *tx);  <br>         dma_async_tx_callback callback;  <br>         void *callback_param;  <br>         struct dmaengine_unmap_data *unmap;  <br>#ifdef CONFIG_ASYNC_TX_ENABLE_CHANNEL_SWITCH  <br>         struct dma_async_tx_descriptor *next;  <br>         struct dma_async_tx_descriptor *parent;  <br>         spinlock_t lock;  <br>#endif  <br>};|

> cookie，一个整型数，用于追踪本次传输。一般情况下，dma controller driver会在内部维护一个递增的number，每当client获取传输描述的时候（参考3.3中的介绍），都会将该number赋予cookie，然后加一。  
> 注7：有关cookie的使用场景，我们会在后续的文章中再详细介绍。
> 
> flags， DMA_CTRL_开头的标记，包括：  
> DMA_CTRL_REUSE，表明这个描述符可以被重复使用，直到它被清除或者释放；  
> DMA_CTRL_ACK，如果该flag为0，表明暂时不能被重复使用。
> 
> phys，该描述符的物理地址？？不太懂！
> 
> chan，对应的dma channel。
> 
> tx_submit，controller driver提供的回调函数，用于把改描述符提交到待传输列表。通常由dma engine调用，client driver不会直接和该接口打交道。
> 
> desc_free，用于释放该描述符的回调函数，由controller driver提供，dma engine调用，client driver不会直接和该接口打交道。
> 
> callback、callback_param，传输完成的回调函数（及其参数），由client driver提供。
> 
> 后面其它参数，client driver不需要关心，暂不描述了。

## 5. 参考文档

[1] Documentation/dmaengine/client.txt

[2] [Linux DMA Engine framework(1)_概述](http://www.wowotech.net/linux_kenrel/dma_engine_overview.html)

[3] [Linux MMC framework(2)_host controller driver](http://www.wowotech.net/comm/mmc_host_driver.html)

[4] [https://forums.xilinx.com/xlnx/attachments/xlnx/ELINUX/10658/1/drivers-session4-dma-4public.pdf](https://forums.xilinx.com/xlnx/attachments/xlnx/ELINUX/10658/1/drivers-session4-dma-4public.pdf)

_原创文章，转发请注明出处。蜗窝科技_，[www.wowotech.net](http://www.wowotech.net/linux_kenrel/dma_engine_api.html)。

标签: [Linux](http://www.wowotech.net/tag/Linux) [Kernel](http://www.wowotech.net/tag/Kernel) [内核](http://www.wowotech.net/tag/%E5%86%85%E6%A0%B8) [API](http://www.wowotech.net/tag/API) [dma](http://www.wowotech.net/tag/dma) [engine](http://www.wowotech.net/tag/engine)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Linux的时钟](http://www.wowotech.net/timer_subsystem/clock-id-in-linux.html) | [系统休眠（System Suspend）和设备中断处理](http://www.wowotech.net/pm_subsystem/suspend-irq.html)»

**评论：**

**菜鸟**  
2019-02-11 11:19

请问内核中还有一组  
dma_async_memcpy_buf_to_buf API 这个和async_memcpy API有什么关系吗？

[回复](http://www.wowotech.net/linux_kenrel/dma_engine_api.html#comment-7179)

**benjoying**  
2017-08-03 17:27

hi wowo,  
问个问题，  
在使用dma传输的时候  
1）申请一个DMA channel。  
2）根据设备（slave）的特性，配置DMA channel的参数。  
3）要进行DMA传输的时候，获取一个用于识别本次传输（transaction）的描述符（descriptor）。  
4）将本次传输（transaction）提交给dma engine并启动传输。  
5）等待传输（transaction）结束。  
正常的逻辑思路都是这样来走的，但是我看到有些项目上是直接配置寄存器操作，没有这些过程，  
这个实在没理解，不知道这是DMA物理硬件上就已经做好了，直接配寄存器就好了？

[回复](http://www.wowotech.net/linux_kenrel/dma_engine_api.html#comment-5871)

**[wowo](http://www.wowotech.net/)**  
2017-08-07 10:08

@benjoying：这也可以理解，框架的好处是接口统一、使用规范，但也有弊端，就是复杂性稍微会增加，理解起来也不是那么直接。  
因此有些开发者（或开发团队）喜欢绕过框架，直接折腾硬件了。这样做可以在最短的时间内将功能做出来，但后期的维护成本可能会高一些，因为通用的框架类似统一的语言，不同公司、不同团队的开发者都懂，自己折腾出来的，就不一样了。

[回复](http://www.wowotech.net/linux_kenrel/dma_engine_api.html#comment-5879)

**benjoying**  
2017-08-07 11:14

@wowo：感谢，如您所说，框架的比较好理解，但是直接折腾硬件的如果没有相关的文档，确实看起来很晦涩

[回复](http://www.wowotech.net/linux_kenrel/dma_engine_api.html#comment-5882)

**irreallich**  
2022-06-05 10:05

@benjoying：其实对于目前的很多开发者来说,反倒是直接折腾硬件更加简便  
做控制器开发很多资深工程师大多经历过单片机,rots,到linux的过程,对linux的dmaengine系统并不熟, 但是因为以前的经验, 对dma控制器的逻辑很熟, 直接从芯片手册操作寄存器的来实现dma功能的速度更快

[回复](http://www.wowotech.net/linux_kenrel/dma_engine_api.html#comment-8625)

**mars**  
2022-01-07 16:19

@benjoying：1.如果是单片机的话,直接使用phy 地址安全系数高  
2.如果是有mmu 的mcu 或者 soc 用linux系统就得做到 kernel 中去,这样就内存管理上来讲,就会安全的多  
3.如果ddr 空间大的情况下,我可以在用户空间指定一块内存空间,供我使用,但是mmap只提供映射关系,不提供空间访问限制,所以不太安全,如果内存紧张,就非常容易出错传输的数据  
4.我现在想用axi sg dma kenrel 驱动 操作，但是fpga 那边修改了参考设计，就很麻烦  
5.我就只能在kernel 中 写一个驱动 使用寄存器的方式, 如题主所说,维护起来很麻烦

[回复](http://www.wowotech.net/linux_kenrel/dma_engine_api.html#comment-8477)

**秋暮离**  
2017-06-19 11:37

wowo好，  
最近在学习V4L2时产生一些疑问，麻烦wowo帮忙点拨以下理解是否正确？  
1、在驱动实现的mmap接口函数中，可以使用dma_mmap_attrs把外部sensor申请的DMA缓冲区mmap到用户空间，此时之前申请DMA缓冲区的函数必须是dma_alloc_attrs，mmap之后相当于有两块虚拟地址对应了同一块物理地址。  
2、dma_alloc_attrs和dma_alloc_coherent在用法上有什么区分吗？适用场景？  
3、我看到像一些fb驱动实现的mmap调用的都是remap_pfn_range，网上说可以使用它来映射内存中的保留页、设备I/O、framebuffer、camera等内存。那是不是可以这样理解：remap_pfn_range作用是用来将物理地址映射到用户空间，ioremap作用是用来将物理地址映射到内核空间。  
4、在videobuf2-dma-config.c中有dma-buf的实现，大概是一个供用户空间或其他设备驱动共享DMA buffer的技术，虽然使用步骤很简单（调用几个API）但我这里还是不太理解：假设1中的想法正确的话，那它和dma_mmap_attrs映射DMA缓冲区到用户空间的手段有什么区别吗，感觉dma_mmap_attrs也有“共享”的意思，ISP负责向DMA buffer放数据，用户空间负责从DMA buffer取数据。  
5、ISP各个子模块间流通数据需要内存额外开辟buffer中转吗（比如preview、resizer、H3A之间），还是它们有相应的硬件缓冲区，只需要将最终处理完的数据放到内存buffer就OK了  
6、像camera、framebuffer用的大内存都来自保留内存吗（mem=size），感觉这种方式比较粗放、浪费，有其它更好的技术吗？  
以上问题麻烦wowo前辈解惑，等您不帮时回复即可，不胜感激！

[回复](http://www.wowotech.net/linux_kenrel/dma_engine_api.html#comment-5696)

**[wowo](http://www.wowotech.net/)**  
2017-06-19 17:18

@秋暮离：这个问题留在这篇文章下面有点不符啊，因为此dma非彼dma（dmaengine）啊。memory的事情，让@linuxer大神回答吧，不敢班门弄斧啊～～

[回复](http://www.wowotech.net/linux_kenrel/dma_engine_api.html#comment-5701)

**秋暮离**  
2017-06-19 18:55

@wowo：抱歉，这些问题是不太符合本文，那我贴到讨论区好了，看看@linuxer大神有时间解答。

[回复](http://www.wowotech.net/linux_kenrel/dma_engine_api.html#comment-5703)

**秋暮离**  
2017-05-28 21:51

wowo好，  
麻烦请教一个问题，spi-pl022.c有如下代码：  
pl022_transfer_one_message  
    do_interrupt_dma_transfer(pl022);  
        /* If we're using DMA, set up DMA here */  
    if (pl022->cur_chip->enable_dma) {  
        /* Configure DMA transfer */  
        if (configure_dma(pl022)) {  
            dev_dbg(&pl022->adev->dev,  
            "configuration of DMA failed, fall back to interrupt mode\n");  
        goto err_config_dma;  
        }  
        /* Disable interrupts in DMA mode, IRQ from DMA controller */  
        irqflags = DISABLE_ALL_INTERRUPTS; //禁掉SSP的所有中断  
            /* Enable SSP, turn on interrupts */  
        writew((readw(SSP_CR1(pl022->virtbase)) | SSP_CR1_MASK_SSE),  
               SSP_CR1(pl022->virtbase));  
        writew(irqflags, SSP_IMSC(pl022->virtbase));  
  
它这里判断如果是采用DMA传输message，就禁掉SPI控制器的所有中断，意思是数据的后续推进全部交由DMA控制器负责，但这里有一点我不是很明确，麻烦请教：DMA传输也经由设备FIFO中转，这里把FIFO相关中断都禁掉了，那DMA传输期间设备FIFO是否可用，是不是就应该由SPI控制器通过DMA请求线告知DMA控制器？但他这里似乎没有写寄存器的动作来设置自己使用DMA传输？是禁掉所有中断默认就是DMA传输吗？  
另外，想知道是不是所有平台控制器采用DMA传输时，都像这样禁掉自身所有中断，然后一切交给DMA负责的路子？

[回复](http://www.wowotech.net/linux_kenrel/dma_engine_api.html#comment-5611)

**[wowo](http://www.wowotech.net/)**  
2017-05-31 17:25

@秋暮离：FIFO是否可用，和“FIFO是否产生中断”，是两个独立的功能，没有依赖。  
如果用DMA传输，SPI控制器不需要向CPU产生中断，但它会向DMA控制器产生一些信号，而DMA控制器在传输的过程中会产生中断。  
  
至于怎么启动dma传输，和这个平台的实现有关，看看spec就行了。

[回复](http://www.wowotech.net/linux_kenrel/dma_engine_api.html#comment-5616)

**秋暮离**  
2017-05-24 20:46

wowo好，  
__dma_request_channel和dma_request_slave_channel应用场景有什么不同吗？我看到一些驱动里面用dma_request_slave_channel的情形好像更多一些。

[回复](http://www.wowotech.net/linux_kenrel/dma_engine_api.html#comment-5599)

**[wowo](http://www.wowotech.net/)**  
2017-05-25 10:14

@秋暮离：dma_request_channel----直接找一个符合规则的dma channel。  
dma_request_slave_channel---配合device tree等，获取和对应设备绑定的dma channel（一般是在dts中指定了）。

[回复](http://www.wowotech.net/linux_kenrel/dma_engine_api.html#comment-5601)

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
    
    - [Linux电源管理(4)_Power Management Interface](http://www.wowotech.net/pm_subsystem/pm_interface.html)
    - [Concurrency Managed Workqueue之（三）：创建workqueue代码分析](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html)
    - [玩转BLE(3)_使用微信蓝牙精简协议伪造记步数据](http://www.wowotech.net/bluetooth/weixin_ble_1.html)
    - [ARMv8之Observability](http://www.wowotech.net/armv8a_arch/Observability.html)
    - [串口通信技术浅析](http://www.wowotech.net/basic_tech/serial_intro.html)
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
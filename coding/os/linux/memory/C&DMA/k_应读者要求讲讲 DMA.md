Linux内核之旅 _2021年12月16日 11:23_

以下文章来源于人人都是极客 ，作者布道师Peter

## DMA 概念介绍

DMA 传输是由 CPU 发起的：CPU 会告诉 DMA 控制器，帮忙将 source 地方的数据搬到 dest 地方。CPU 发完指令之后，就不管了。具体怎么搬，何时搬，完全由 DMA 控制器决定。DMA 控制器搬运数据的方向有如下几种：
![[Pasted image 20241008162332.png]]

### 何时传输（DMA request lines）

因为 CPU 发起 DMA 传输的时候，并不知道当前是否具备传输条件，例如 source 设备是否有数据、dest 设备的 FIFO 是否空闲等等。那谁知道是否可以传输呢？设备！因此，需要设备和 DMA 控制器之间，有几条物理的连接线（称作DMA request，DRQ），用于通知 DMA 控制器可以开始传输了。

### 传输通道（DMA channels）

DMA 控制器可以同时进行的传输个数是有限的，每一个传输都需要使用到 DMA 物理通道。DMA 物理通道的数量决定了 DMA 控制器能够同时传输的任务量。

在软件上，DMA 控制器会为外设分配一个 DMA 虚拟通道，这个虚拟通道是根据 DMA request 信号来区分的。

### 传输参数

- transfer size
- transfer width
- burst size

### scatter gather

我们知道，一般情况下 DMA 传输只能处理在物理上连续的 buffer。但在有些场景下，我们需要将一些非连续的 buffer 拷贝到一个连续 buffer 中，这样的操作称作 scatter gather。

对于这种非连续的传输，大多时候都是通过软件，将传输分成多个连续的小块。但为了提高传输效率，有些 DMA controller 从硬件上支持了这种操作。

## DMA 客户端设备驱动

我们先看下一个 DMA 客户端设备驱动涉及到的数据结构有哪些，然后再看下使用 dmaengine 内核框架的步骤。

### 数据结构

- **dma_slave_config**

完成一次 DMA 传输所需要的参数：

```cpp
struct dma_slave_config {    /*    指明传输的方向    DMA_MEM_TO_MEM，memory到memory的传输；       DMA_MEM_TO_DEV，memory到设备的传输；       DMA_DEV_TO_MEM，设备到memory的传输；       DMA_DEV_TO_DEV，设备到设备的传输。    */    enum dma_transfer_direction direction;    /*    传输方向是dev2mem或者dev2dev时，读取数据的位置（通常是固定的FIFO地址）。    对mem2dev类型的channel，不需配置该参数（每次传输的时候会指定）；    */    phys_addr_t src_addr;    /*    传输方向是mem2dev或者dev2dev时，写入数据的位置（通常是固定的FIFO地址）。    对dev2mem类型的channel，不需配置该参数（每次传输的时候会指定）；    */    phys_addr_t dst_addr;    //src地址的宽度    enum dma_slave_buswidth src_addr_width;    //dst地址的宽度    enum dma_slave_buswidth dst_addr_width;    //src最大可传输的burst size    u32 src_maxburst;    //dst最大可传输的burst size    u32 dst_maxburst;    u32 src_port_window_size;    u32 dst_port_window_size;    u32 src_fifo_num;    u32 dst_fifo_num;    bool device_fc;    /*    外部设备通过slave_id告诉dma controller自己是谁（一般和某个request line对应）。    很多dma controller并不区分slave，只要给它src、dst、len等信息，它就可以进行传输，因此slave_id可以忽略。    而有些controller，必须清晰地知道此次传输的对象是哪个外设，就必须要提供slave_id了。    */    unsigned int slave_id;   };   
```

- **dma_async_tx_descriptor**

用于描述一次 DMA 传输:

```cpp
struct dma_async_tx_descriptor {    dma_cookie_t cookie;    /*    DMA_CTRL_开头的标记，包括：    DMA_CTRL_REUSE，表明这个描述符可以被重复使用，直到它被清除或者释放；    DMA_CTRL_ACK，如果该flag为0，表明暂时不能被重复使用。    */    enum dma_ctrl_flags flags; /* not a 'long' to pack with cookie */    //该描述符的物理地址    dma_addr_t phys;    //对应的dma channel    struct dma_chan *chan;    /*    controller driver提供的回调函数，用于把改描述符提交到待传输列表。    通常由dma engine调用，client driver不会直接和该接口打交道。    */    dma_cookie_t (*tx_submit)(struct dma_async_tx_descriptor *tx);    /*    用于释放该描述符的回调函数。    通常由dma engine调用，client driver不会直接和该接口打交道。    */    int (*desc_free)(struct dma_async_tx_descriptor *tx);    //传输完成的回调函数（及其参数），由client driver提供。    dma_async_tx_callback callback;    dma_async_tx_callback_result callback_result;    //传输完成的回调函数（及其参数），由client driver提供。    void *callback_param;    struct dmaengine_unmap_data *unmap;   #ifdef CONFIG_ASYNC_TX_ENABLE_CHANNEL_SWITCH    struct dma_async_tx_descriptor *next;    struct dma_async_tx_descriptor *parent;    spinlock_t lock;   #endif   }; 
```

### 设备驱动使用 dmaengine 的步骤

一个设备驱动程序使用 dmaengine 的话一般步骤如下：

1. 申请一个 DMA channel。
1. 根据设备（slave）的特性，配置 DMA channel 的参数。
1. 要进行 DMA 传输的时候，获取一个用于识别本次传输（transaction）的描述符（descriptor）。
1. 将本次传输（transaction）提交给 DMA Engine 并启动传输。
1. 等待传输（transaction）结束。

然后，重复3~5步。下面具体看下每一步的实现和相关接口：

- **申请 DMA channel**

任何设备驱动在开始 DMA 传输之前，都要申请一个 DMA channel(由“struct dma_chan”数据结构表示)，可以通过如下的 API 申请 DMA channel：

> struct dma_chan \*dma_request_chan(struct device \*dev, const char \*name);

可以用如下接口释放申请得到的 DMA channel：

> void dma_release_channel(struct dma_chan \*chan);

- **配置 DMA channel 的参数**

设备驱动申请到一个为自己使用的 DMA channel 之后，需要根据自身的实际情况，以及 DMA controller 的能力，对该 channel 进行一些配置。设备驱动将它们填充到一个 struct dma_slave_config 变量中后，可以调用如下 API 将这些信息告诉给 DMA 控制器：

> int dmaengine_slave_config(struct dma_chan \*chan, struct dma_slave_config \*config)

- **获取传输描述**

DMA 传输属于异步传输，在启动传输之前，设备驱动需要将此次传输的一些信息（例如src/dst的buffer、传输的方向等）提交给 DMA 控制器，DMA 控制器确认好后，返回一个描述符。此后，设备驱动就可以以该描述符为单位，控制并跟踪此次传输。设备驱动可以使用下面三个 API 获取传输描述符:

> **用于在“scatter gather buffers”列表和总线设备之间进行 DMA 传输**:
>
> struct dma_async_tx_descriptor \*dmaengine_prep_slave_sg( struct dma_chan \*chan, struct scatterlist \*sgl, unsigned int sg_len, enum dma_data_direction direction, unsigned long flags);

> **用于音频等场景中，在进行一定长度的dma传输（buf_addr&buf_len）的过程中，每传输一定的byte（period_len），就会调用一次传输完成的回调函数:**
>
> struct dma_async_tx_descriptor \*dmaengine_prep_dma_cyclic( struct dma_chan \*chan, dma_addr_t buf_addr, size_t buf_len, size_t period_len, enum dma_data_direction direction);

> **可进行不连续的、交叉的DMA传输，通常用在图像处理、显示等场景中:**
>
> struct dma_async_tx_descriptor \*dmaengine_prep_interleaved_dma( struct dma_chan \*chan, struct dma_interleaved_template \*xt, unsigned long flags);

- **启动传输**

获取传输描述符之后，设备驱动可以通过 dmaengine_submit 接口将该描述符放到传输队列上，然后调用 dma_async_issue_pending 接口，启动传输。

> dma_cookie_t dmaengine_submit(struct dma_async_tx_descriptor \*desc);

> void dma_async_issue_pending(struct dma_chan \*chan);

由这两个 API 的特征可知，内核 DMA Engine 鼓励设备驱动一次提交多个传输，然后由 DMA 控制器统一完成这些传输。

- **等待传输结束**

传输请求被提交之后，设备驱动可以通过回调函数获取传输完成的消息，当然，也可以通过 dma_async_is_tx_complete 等 API，测试传输是否完成。如果等不及了，也可以使用 dmaengine_pause、dmaengine_resume、dmaengine_terminate_xxx 等 API，暂停、终止传输。

## eDMA（DMA 控制器） 驱动

DMA 控制器驱动主要负责抽象 eDMA 硬件，管理 DMA channel，以 channel 为操作对象，响应 设备驱动的传输请求，并控制 eDMA 驱动，执行传输。

内核的 dmaengine framework 提供了一套 DMA 控制器的开发框架，如下图所示：
!\[\[Pasted image 20241008162409.png\]\]

1. 使用 struct dma_device 抽象 eDMA，eDMA 驱动只要填充该结构中必要的字段。
1. 使用struct dma_chan（即图中的 DCn）抽象物理的 DMA channel（即图中的 CHn），DCn 和 CHn 一一对应。
1. 基于物理的 DMA channel，使用 struct virt_dma_chan 抽象出虚拟的 dma channel（即图中的 VCx）。多个虚拟 channel 可以共享一个物理 channel，并在这个物理 channel 上进行分时传输。

下面我们看下 dma_device，dma_chan，virt_dma_chan 的数据结构。

### 数据结构

- **dma_device**

主要用于抽象 eDMA:

```cpp
struct dma_device {    //一个链表头，用于保存该controller支持的所有dma channel    //在初始化的时候，dma controller driver首先要调用 INIT_LIST_HEAD 初始化它，然后调用 list_add_tail 将所有的 channel 添加到该链表头中。    
unsigned int chancnt;    unsigned int privatecnt;    struct list_head channels;     ......    /*    一个bitmap，用于指示该dma controller所具备的能力       DMA_MEMCPY，可进行memory copy；       DMA_MEMSET，可进行memory set；       DMA_SG，可进行 scatter list 传输；       DMA_CYCLIC，可进行cyclic类的传输；       DMA_INTERLEAVE，可进行交叉传输；    */    dma_cap_mask_t  cap_mask;     ......    int dev_id;    struct device *dev;       //表示该controller支持哪些宽度的src类型。具体可参考 enum dma_slave_buswidth 的定义    u32 src_addr_widths;    //表示该controller支持哪些宽度的dst类型。具体可参考 enum dma_slave_buswidth 的定义    u32 dst_addr_widths;    /*    表示该controller支持哪些传输方向    包括DMA_MEM_TO_MEM、DMA_MEM_TO_DEV、DMA_DEV_TO_MEM、DMA_DEV_TO_DEV，具体可参考enum dma_transfer_direction的定义    */    u32 directions;    //支持的最大的burst传输的size    u32 max_burst;    //指示该controller的传输描述可否可重复使用    bool descriptor_reuse;    enum dma_residue_granularity residue_granularity;       //申请/释放 dma channel 的时候，dmaengine会调用dma controller driver相应的alloc/free回调函数，以准备相应的资源。    int (*device_alloc_chan_resources)(struct dma_chan *chan);    void (*device_free_chan_resources)(struct dma_chan *chan);       //设备驱动通过 dmaengine_prep_xxx API 获取传输描述符的时候，damengine则会直接回调 eDMA 驱动相应的 device_prep_dma_xxx 接口。    struct dma_async_tx_descriptor *(*device_prep_dma_memcpy)(     struct dma_chan *chan, dma_addr_t dst, dma_addr_t src,     size_t len, unsigned long flags);     ......    struct dma_async_tx_descriptor *(*device_prep_dma_imm_data)(     struct dma_chan *chan, dma_addr_t dst, u64 data,     unsigned long flags);       //设备驱动调用 dmaengine_slave_config 配置 dma channel 的时候，dmaengine 会调用该回调函数，交给 eDMA 驱动处理。    int (*device_config)(struct dma_chan *chan,           struct dma_slave_config *config);       //设备驱动调用 dmaengine_pause、dmaengine_resume、dmaengine_terminate_xxx 等API的时候，dmaengine 会调用相应的回调函数，交给 eDMA 驱动处理。    int (*device_pause)(struct dma_chan *chan);     ......    void (*device_synchronize)(struct dma_chan *chan);       enum dma_status (*device_tx_status)(struct dma_chan *chan,            dma_cookie_t cookie,            struct dma_tx_state *txstate);    //设备驱动调用 dma_async_issue_pending 启动传输的时候，会调用调用该回调函数。    void (*device_issue_pending)(struct dma_chan *chan);   };   
```

- **dma_chan**

主要用于抽象物理的 DMA channel：

`struct dma_chan {    //指向该channel所在的dma controller    struct dma_device *device;    //设备驱动以该 channel 为操作对象获取传输描述符时，eDMA 驱动返回给设备的最后一个cookie。    dma_cookie_t cookie;    //在这个 channel 上最后一次完成的传输的 cookie。eDMA 驱动可以在传输完成时调用辅助函数 dma_cookie_complete 设置它的 value。    dma_cookie_t completed_cookie;       /* sysfs */    int chan_id;    struct dma_chan_dev *dev;       //链表 node，用于将该 channel 添加到 dma_device 的 channel 列表中    struct list_head device_node;    struct dma_chan_percpu __percpu *local;    int client_count;    int table_count;       /* DMA router */    struct dma_router *router;    void *route_data;       void *private;   };   `

- **virt_dma_chan**

主要用于抽象一个虚拟的 DMA channel：

`struct virt_dma_chan {    //一个 struct dma_chan 类型的变量，用于和设备驱动打交道（屏蔽物理channel和虚拟channel的差异）。    struct dma_chan chan;    //一个 tasklet，用于等待该虚拟 channel 上传输的完成（由于是虚拟channel，传输完成与否只能由软件判断）。    struct tasklet_struct task;    void (*desc_free)(struct virt_dma_desc *);       spinlock_t lock;       /* protected by vc.lock */    //用于保存不同状态的虚拟 channel 描述符（struct virt_dma_desc，仅仅对 struct dma_async_tx_descriptor 做了一个简单的封装）。    struct list_head desc_allocated;    struct list_head desc_submitted;    struct list_head desc_issued;    struct list_head desc_completed;       struct virt_dma_desc *cyclic;   };   `

### eDMA 使用 dmaengine 的接口

damengine 直接向 DMA 控制器驱动提供的API并不多，主要包括：

- struct dma_device 变量的注册和注销接口：

> int dma_async_device_register(struct dma_device \*device);

> void dma_async_device_unregister(struct dma_device \*device);

- cookie 有关的辅助接口:

> 初始化dma channel中的cookie、completed_cookie字段:
>
> static inline void dma_cookie_init(struct dma_chan \*chan)

> 为指针的传输描述（tx）分配一个cookie:
>
> static inline dma_cookie_t dma_cookie_assign(struct dma_async_tx_descriptor \*tx)

> 当某一个传输（tx）完成的时候，可以调用该接口，更新该传输所对应channel的completed_cookie字段:
>
> static inline void dma_cookie_complete(struct dma_async_tx_descriptor \*tx)

> 获取指定channel（chan）上指定cookie的传输状态:
>
> static inline enum dma_status dma_cookie_status(struct dma_chan \*chan, dma_cookie_t cookie, struct dma_tx_state \*state)

### eDMA 使用 dmaengine 的步骤

1. 定义一个 struct dma_device 变量，并根据实际的硬件情况，填充其中的关键字段。
1. 根据 eDMA 支持的 channel 个数，为每个 channel 定义一个 struct dma_chan 变量，进行必要的初始化后，将每个 channel 都添加到 struct dma_device 变量的 channels 链表中。
1. 根据硬件特性，实现 struct dma_device 变量中必要的回调函数（device_alloc_chan_resources/device_free_chan_resources、device_prep_dma_xxx、device_config、device_issue_pending等等）。
1. 调用 dma_async_device_register 将 struct dma_device 变量注册到 kernel 中。
1. 当 DMA 客户端设备驱动申请 dma channel 时（例如通过 device tree 中的 dma 节点获取），dmaengine core 会调用 eDMA 驱动的 device_alloc_chan_resources 函数，controller driver需要在这个接口中奖该channel的资源准备好。
1. 当 DMA 客户端设备驱动配置某个 dma channel 时，dmaengine core 会调用 eDMA 驱动的 device_config 函数，eDMA 驱动需要在这个函数中将 DMA 客户端设备驱动想配置的内容准备好，以便进行后续的传输。
1. DMA 客户端设备驱动开始一个传输之前，会把传输的信息通过 dmaengine_prep_slave_xxx 接口交给eDMA 驱动，eDMA 驱动需要在对应的 device_prep_dma_xxx 回调中，将这些要传输的内容准备好，并返回给DMA 客户端设备驱动一个传输描述符。
1. 然后，DMA 客户端设备驱动会调用 dmaengine_submit 将该传输提交给 eDMA 驱动，此时 dmaengine 会调用 eDMA 驱动为每个传输描述符所提供的 tx_submit 回调函数，eDMA 驱动需要在这个函数中将描述符挂到该 channel 对应的传输队列中。
1. DMA 客户端设备驱动开始传输时，会调用 dma_async_issue_pending，eDMA 驱动需要在对应的回调函数（device_issue_pending）中，依次将队列上所有的传输请求提交给硬件。

## Dynamic DMA mapping

待续...

阅读 4768

​

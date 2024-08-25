# 

Original qinhua OPPO内核工匠

 _2024年04月26日 17:30_

# **1.V4L2简介**   

  

### **1.1 什么是V4L2**  

V4L，其全称是Video4Linux（即Video for Linux），是Linux内核中关于视频设备的驱动框架，涉及开关视频设备，以及从该类设备采集并处理相关的音、视频信息。V4L从Linux2.1版本的内核中开始出现。现在Linux内核中用的是V4L2，即Video4Linux2（即Video for Linux Two），其是修改V4L相关Bug后的一个升级版，始于Linux 2.5内核。

  

由于硬件的复杂性，V4L2 驱动程序往往非常复杂：大多数设备都有多个 IC，在 /dev 中导出多个设备节点，并创建非 V4L2 设备，例如 DVB、ALSA、FB、I2C 和输入（IR ） 设备。尤其是 V4L2 驱动程序必须设置支持 IC 来进行音频/视频复用/编码/解码，这一事实使其比大多数驱动程序更加复杂。通常这些IC通过一条或多条I2C总线连接到主桥驱动器，但也可以使用其他总线。此类设备称为“子设备”。长期以来，该框架仅限于用于创建 V4L 设备节点的 video_device 结构和用于处理视频缓冲区的 video_buf。这意味着所有驱动程序都必须自行设置设备实例并连接到子设备。其中一些操作要正确执行起来相当复杂，许多驱动程序从未正确执行过。还有很多通用代码由于缺乏框架而永远无法重构。

  

因此，这个V4L2框架设置了所有驱动程序所需的基本构建块，并且这个相同的框架应该使将公共代码重构为所有驱动程序共享的实用程序函数变得更加容易。

### **1.2 Video设备的V4L2框架**  

图1-1是基于Video设备的V4L2框架

![Image](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjMoCHeaK1vDMpxYAUylEppKSptjXicggVRQeVNib7xRSiamAy3ylFH1m6JF8hC1rbUcnprooENCXADww/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

             图1-1 V4L2框架

  

Linux系统中视频设备主要包括以下四个部分：    

1.字符设备驱动程序核心：V4L2本身就是一个字符设备，具有字符设备所有的特性，暴露接口给用户空间；用户空间可以通过Ioctl系统调用控制设备；在应用层，我们可以在 /dev 目录发现 videoxx 类似的设备节点。

2.V4L2驱动核心：主要是构建一个内核中标准视频设备驱动的框架，为视频操作提供统一的接口函数；

3.平台V4L2设备驱动：在V4L2框架下，根据平台自身的特性实现与平台相关V4L2驱动部分，包括注册video_device和v4l2_dev。

4.具体的sensor驱动：主要上电、提供工作时钟、视频图像裁剪、流IO开启等，实现各种设备控制方法供上层调用并注册v4l2_subdev。

### **1.3 V4L2支持的设备类型**  

V4L2支持的设备十分广泛，但是其中只有很少一部分在本质上是真正的视频设备：

- video capture interface：视频采集接口，从camera上获取视频数据，视频捕获是V4L2的基本应用；
    
- video output interface：视频输出接口，允许应用程序驱动外设提供视频图像
    
- video overlay interface：视频覆盖接口，是捕获接口的一个变体，其工作是便于捕获设备直接显示到显示器，无需经过CPU；Android拍照应用在进行预览时，可能就是这种模式。
    
- VBI interfaces：基于电视场消隐实现远程传送文字的技术与设备；    
    
- radio interface：无线电接口，从AM和FM调谐器设备访问音频流。
    
- Codec Interface：编解码接口，对视频数据流执行转换。
    

通常情况下V4L2的主设备号是81，次设备号为0～255；这些次设备号里又分为多类设备：视频设备、Radio（收音机）设备、Teletext on VBI等。因此V4L2设备对应的文件节点有：/dev/videoX、/dev/vbiX、/dev/radioX。对于Radio设备，即用于收发声音。但要提醒注意的是，对于声音的采集与处理，在我们的Android手持设备中常会有个Mic设备，它则是属于ALSA子系统的。我们主要讨论的是Video设备。

## **2.编解码设备**  

  

传统的Capture设备从Camera硬件读取数据到DDR，Output设备从DDR读取数据输出到Display；Codec Interface 不同于传统的V4L2 Capture / Output 设备；通常是基于M2M(memory to memory)，即从DDR读取数据，进行一些处理后再写入DDR;

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图2-1. V4L2 M2M Device

  

V4L2 M2M设备可以在内存中压缩、解压缩、转换或以其他方式将视频数据从一种格式转换为另一种格式。M2M设备的示例包括编解码器、缩放器、去隔行器或格式转换器（如从 YUV 转换为 RGB）。    

M2M视频节点的行为就像普通视频节点一样，但它支持输出（将帧从内存发送到硬件设备）和捕获（将处理后的帧从硬件接收到内存）I/O Steaming。应用程序必须为双方设置流 I/O，并最终调用 VIDIOC_STREAMON 进行捕获和输出以启动硬件。

内存到内存设备充当共享资源：您可以多次打开视频节点，每个应用程序设置自己的文件句柄本地属性，并且每个应用程序都可以独立于其他应用程序使用它。驱动程序将仲裁对硬件的访问，并在另一个文件处理程序获得访问权限时对其重新编程。这与通常的视频节点行为不同，在通常的视频节点行为中，视频属性对于设备来说是全局的（即通过一个文件句柄更改某些内容可以通过另一个文件句柄看到）。

最常见的内存到内存设备之一是编解码器。编解码器比大多数编解码器更复杂，并且需要对其编解码器参数进行额外设置。这是通过编解码器控件完成的。以下各节给出了有关如何使用编解码器内存到内存设备的更多详细信息。

### **2.1.Stateless Video Codecs**  

  

虽然对于上层应用来说视频播放有暂停/播放状态，其实是上层应用通过是否给底层编解码器送数据来模拟的状态；如果上层应用没有给底层编解码器喂数据，则可以认为是pause状态，反之则是playing状态；    

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图2-2  Android Mediaplayer状态图

对于底层soc硬件编/解码器都是基于无状态的；

无状态解码器是一种在处理的帧之间不保留任何状态的解码器。这意味着每一帧的编解码独立于任何先前和未来的帧，并且客户端负责维护解码状态并将其通过每个解码请求提供给解码器。这与有状态视频解码器接口形成对比，在有状态视频解码器接口中，硬件和驱动程序维护解码状态，并且客户端所要做的就是提供原始编码流并按显示顺序使解码帧出队。    

### **2.2 用户空间如何与无状态解码器通信**  

  

本节描述用户空间（“客户端”）如何与无状态解码器通信，以便成功解码压缩流。与有状态编解码器相比，解码器/客户端序列更简单，但这种简单性的代价是客户端中负责维护一致解码状态的额外复杂性。

先用应用角度看用户空间如何调用V4L2接口：    

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图2-3 用户空间解码流程

  

基本流程如下：

1. 打开设备：open  设备节点 /dev/videoX

2. 查询设备能力，获取设备driver_name等；

3. 枚举格式/设置格式：查询设备能够支持处理的color format等；

4. “请求”缓存：目前Android设备采用DMA-Buf，kernel不涉及内存申请，只是创建videobuf2队列；    

5. 启动流转：

6.循环处理每一帧上层送入的数据，将处理好的数据返回用户空间；

V4L2  相关 API如下：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

             用户空间API                        kernel space对应API

### **2.3.Buffer 管理**  

  

Buffer管理主要涉及如何在用户空间和内核空间传递数据，如视频H.264压缩数据和解码之后的YUV数据；如下图，对于解码case，output buffers是视频压缩数据，capture buffers是解码后的YUV数据；    

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图2-4：V4L2 M2M Buffer管理

#### **关键数据结构：**  

vb2_queue：

提供内核与用户空间的 buffer 流转接口，

Vb2_queue代表一个videobuffer队列，vb2_buffer是这个队列中的成员，vb2_mem_ops是缓冲内存的操作函数集，vb2_ops用来管理队列。

|   |
|---|
|struct vb2_queue {<br><br>         enum v4l2_buf_type                  type;  //buffer类型<br><br>         unsigned int                        io_modes;  //访问IO的方式:mmap、userptr etc<br><br>         const struct vb2_ops                 *ops;   //buffer队列操作函数集合<br><br>         const struct vb2_mem_ops     *mem_ops;  //buffer memory操作集合<br><br>         struct vb2_buffer              *bufs[VIDEO_MAX_FRAME];  //代表每个buffer        <br><br>         unsigned int                        num_buffers;    //分配的buffer个数<br><br>……<br><br>};|

vb2_buffer：

vb2_buffer是缓存队列的基本单位，内嵌在其中v4l2_buffer是核心成员。

当开始IO Streaming时，帧以v4l2_buffer的格式在应用和驱动之间传输。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

当用户空间拿到v4l2_buffer，可以获取到缓冲区的相关信息。Byteused是图像数据所占的字节数，如果是V4L2_MEMORY_MMAP方式，m.offset是内核空间图像数据存放的开始地址，会传递给mmap函数作为一个偏移，通过mmap映射返回一个缓冲区指针p，p+byteused是图像数据在进程的虚拟地址空间所占区域；

如果是VB2_MEMORY_USERPTR的方式，可以获取的图像数据开始地址的指针m.userptr，userptr是一个用户空间的指针，userptr+byteused便是所占的虚拟地址空间，应用可以直接访问。

一个缓冲区可以有三种状态：

在驱动的传入队列中，驱动程序将会对此队列中的缓冲区进行处理，用户空间通过IOCTL:VIDIOC_QBUF把缓冲区放入到队列。

在驱动的传出队列中，这些缓冲区已由驱动处理过，正等用户空间取走；对于一个视频捕获设备，缓存区已经填充了从camera采集的YUV数据；对于视频解码设备，填充了已解码后的YUV数据；

用户空间状态的队列，已经通过IOCTL:VIDIOC_DQBUF传出到用户空间的缓冲区，此时缓冲区由用户空间拥有，驱动无法访问。

#### **Buffer类型：**  

VB2_MEMORY_MMAP：

缓冲区由内核分配，并通过 mmap() ioctl 进行内存映射。当用户通过 read() 或 write() 系统调用使用缓冲区时，也会使用此模型。对于mmap方式，由于buffer是在内核空间中分配的，这种情况下这些buffer不能被交换到SWAP中。虽然这种方法不怎么影响读写效率，但是它一直占用着内核空间中的内存，当系统的内存有限的时候，如果同时运行大量的进程，则对系统的整体性能有一定的影响。    

VB2_MEMORY_USERPTR：

内存由用户空间的应用程序分配，并把地址传递到内核中的驱动程序，然后由V4L2驱动程序直接将数据填充到用户空间的内存中。这点需要在v4l2_requestbuffers里将memory字段设置成V4L2_MEMORY_USERPTR。

VB2_MEMORY_DMABUF：

缓冲区通过 DMA 缓冲区传递到用户空间。Android系统视频编解码采用此类型；input/output buffer都是用户空间通过gralloc模块申请GraphicBuffer，通过dma-buf fd在用户空间和内核空间之间传递；通过dmabuf fd，可以做到零拷贝(零拷贝是指CPU不参与拷贝)

Android系统视频编解码时采用DMABUF；通过GraphicBuffer 对应文件句柄fd 传递，减少内存拷贝；    

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图2-5：DMABUF角色

  

DMABUF 是一个buffer sharing framework,本身并没有分配内存的能力。Dma-buf框架实现了进程与进程之间、进程与内核之间的内存共享方案。应该把ION/ DMA-BUF Heaps与dma-buf当做是一个整体，看成是共享内存机制。需要说明的是从Android12开始，GKI 2.0 将 ION 分配器替换为了 DMA-BUF 堆。

## **3.总结**  

   本文简要介绍了V4L2框架及其在视频编解码中的应用，包括编解码器的工作原理、用户空间与编解码器的交互流程、以及V4L2的Buffer管理机制。特别提到了Android系统中视频编解码的实现方式，强调了DMABUF在进程间内存共享中的重要性。

## **4.参考文档：**  

1.https://www.cnblogs.com/LoyenWang/p/15456230.html

2.https://lwn.net/Articles/203924/

3.https://www.kernel.org/doc/html/v4.9/media/kapi/v4l2-core.html  
4. Android驱动开发权威指南 杨柳编著

5.https://elinux.org/images/e/ec/V4L2-M2M-as-the-driver-framework-for-Video-Processing-IP.pdf 

  

往

期

推

荐

[Android中基于DWARF的stack unwind实现原理](http://mp.weixin.qq.com/s?__biz=MzAxMDM0NjExNA==&mid=2247493516&idx=1&sn=c8ff0c1adaecb008a60d8b3a472a8b50&chksm=9b536c61ac24e5772abbf6fc4b9f2d48cf714321dac2e7b9bb5b26fb375dc507d43520b4ac95&scene=21#wechat_redirect)  

[基于devfreq framework的GPU调频](http://mp.weixin.qq.com/s?__biz=MzAxMDM0NjExNA==&mid=2247492136&idx=1&sn=6121fbbdfc56282bfe892712f84eb330&chksm=9b5369c5ac24e0d37c69fd336f85dfb5707e937ead74269c29d153db4ce90e6f63986a4079b5&scene=21#wechat_redirect)  

[Linux Large Folios大页在社区和产品的现状和未来](http://mp.weixin.qq.com/s?__biz=MzAxMDM0NjExNA==&mid=2247493526&idx=1&sn=fe1f9f7d13d3dee696359aa175a77540&chksm=9b536c7bac24e56d77f4e9c66193b7231b41aa3a917c7f7c5a26eb05605d17317367aba06bb6&scene=21#wechat_redirect)  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

长按关注内核工匠微信

Linux内核黑科技| 技术文章| 精选教程

Reads 3872

​

Comment

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjNfxSer7sH7b1yJBvlpWyx1AdCugCLScXKu60Ezh9oSCrZw36x9GTL1qvIzptqlefgS1vkQwBE7OA/300?wx_fmt=png&wxfrom=18)

OPPO内核工匠

4632820

Comment

Comment

**Comment**

暂无留言
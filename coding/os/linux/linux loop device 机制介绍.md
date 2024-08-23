
原创 Link OPPO内核工匠

 _2021年10月22日 17:00_

  

**一、简介**

在类Unix操作系统中，存在一种特殊的块设备loopdevice，它是基于现有文件虚拟出来的一种设备文件。如果文件中包含可用的文件系统，那么虚拟出来的块设备可以像正常的磁盘一样进行挂载，因此loop device常用于挂载磁盘镜像文件。

  

挂载包含文件系统的磁盘镜像后，可以通过操作系统中通用的文件系统接口对该镜像中的文件进行访问，而无需使用特定的接口或软件。这是一种便捷的镜像管理方法，具有多种应用场景。例如，可以在保持磁盘分区的情况下，将操作系统的安装到镜像文件中；也可以实现数据的隔离，在虚拟的块设备上，封装一层加密文件系统。

  

**二、使用方法**

通过losetup命令，可以关联和分离loopdevice与文件，或查询loop device的状态。可通过以下步骤进行loop device的初始化。

  

创建包含文件系统的磁盘镜像：

  

![图片](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjNMRVVAc0SuvId7t5TRcdoZrLpsUlM0neGibSNJQibwOfunQfwCAgA3thhN8J3vib2oCpJn5CFI1iaOMA/640?wx_fmt=png&wxfrom=13&tp=wxpic)

  

获取可用的loop device并关联文件：

  

![图片](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjNMRVVAc0SuvId7t5TRcdoZ2HKRibQibmchFOxQUksjEGekb3OkCgXsUH57gAicOR9PJNP22EwEa2gpw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

  

除了使用losetup命令，也可通过ioctl的方式配置，具体的flag可以参考<linux/loop.h>中的定义。

  

**三、实现细节**  

配置loopdevice时，通过loop_add接口初始化出一个虚拟的块设备，对应的blk_mq_ops如下：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

当block层提交一个request到块设备时，会调用对应的queue_rq方法，根据输入的request对loop_cmd进行初始化，用于后续的io转发，最后会提交到对应loop device的workqueue进行处理。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

经过一些前置条件的检查后，会调用do_req_filebacked对关联的文件进行操作：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

随后跟据request的flag对文件进行不同的操作,  
对于flush操作，需要将对应的数据落盘，则调用fsync对整个文件进行flush：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

对于discard操作，由于是虚拟的块设备，只支持了REQ_OP_WRITE_ZEROES和REQ_OP_DISCARD两种命令：

当执行REQ_OP_WRITE_ZEROES时，会要求对指定的范围置0，因此调用fallocate对关联文件的对应范围置0；  
当执行REQ_OP_DISCARD时，则只会将执行的范围通过fallocate舍弃，后续从中读取的内容，取决于不同文件系统对fallocate的实现。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

对于读写操作，如果是配置了加密或direct I/O的loop device，会有不同的处理：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

如果在初始化时配置了加密算法，则会调用对应的实现对文件进行加解密:

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

  

如果配置了direct I/O的方式来对关联文件进行操作，则会通过非阻塞I/O来进行io的转发。根据patch作者的说明，使用非阻塞I/O主要是为了避免上下文切换，以获得良好的吞吐量。

  

在缓冲文件的读取中，随机I/O通常只有在大量任务并发提交时才能获得最高吞吐量；但是对于顺序 I/O，大多数时候可以从页面缓存中命中，因此并发提交通常会引入不必要的上下文切换，并且不能太大提高吞吐量。通过非阻塞I/O，可以避免并发提交，同时不影响随机读取吞吐量。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

##   

## **四、结语**

本文主要对linux loop device的机制进行了简要介绍，并对其IO转发的细节进行了分析。整体来看，loop device转发IO的软件栈很短，保证了性能，同时对块设备的接口具有较好的兼容性和可扩展性。

##   

##   

## 参考文献

https://en.wikipedia.org/wiki/Loop_device  
https://man7.org/linux/man-pages/man8/losetup.8.html  
https://man7.org/linux/man-pages/man4/loop.4.html  
https://lwn.net/Articles/612483/

  

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**长按关注**

**内核工匠微信**

  

Linux 内核黑科技 | 技术文章 | 精选教程

  

阅读 1684

​
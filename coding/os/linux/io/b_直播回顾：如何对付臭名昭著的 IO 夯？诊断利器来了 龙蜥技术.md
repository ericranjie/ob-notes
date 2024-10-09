# 

云巅论剑

_2021年12月02日 20:00_

以下文章来源于OpenAnolis龙蜥 ，作者李光水、毛文安

\[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM4bZCboj0qz7G2rqBCKMh4iblVGkMN9XKUxiaECBVF6nj1g/0)

**OpenAnolis龙蜥**.

龙蜥社区（OpenAnolis）是立足中国面向国际的 Linux 服务器操作系统开源根社区及创新平台。

\](https://mp.weixin.qq.com/s?\_\_biz=MzUxNjE3MTcwMg==&mid=2247486692&idx=1&sn=c82230b7611e9e9b33a1d012bd2f2e1c&chksm=f9aa3e3dceddb72b4d15345c6d4f6139fed7735f522e5bd4d71d7f37d1f1dba03789a80addbc&mpshare=1&scene=24&srcid=1202q8pDUW6wZWzoio9d6Rdv&sharer_sharetime=1638446727189&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d04789e970ff79682ec977325c54793dfacbe24ae3b4fb8959bb1b315e0fa33f40d63a4ce737cc5b2ab3871f5adddec18333e67f078d563be124afcd9e204dce7dcc53f220ced61acdea106ab005ba7e963ce4bd2575567e5073c980c43ffe25cc94d698232824e647c62304c0a62272daaf147cd7a76cdc75&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQ4rEFUnxkzAJuUlWE0Grn7hLbAQIE97dBBAEAAAAAAAyFMlvbMI8AAAAOpnltbLcz9gKNyK89dVj0bCF5ouRek50Xxj2HtuU9Nf0a4%2FytFsCNgoVDk%2BkfuKoX4io8zlyqXAm%2F5%2FioyxnISFd%2FqYkrAHLr%2FxFP3v%2FFfXzAfFCsOKfyGUp6i1L5jjDgcS9Y4nllN58uV87Hl9DRoJUNzkZnRC6gBz9nAdFZ%2FqIAF73zveDCzalVuFZVeTwtLnmI0cENDkb80pfNg77gQvkA7B8Lmv%2BQQ3moMfe9SbDMY%2F8qSXfln2qcIeGdUxF73fwYpw%3D%3D&acctmode=0&pass_ticket=uhDtJfYrSkyY%2FeQQwkqcGt3fQmILzsYnwmLyvq5xM2mXVOu5oeT4biFv38In5BNO&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1#)

\*\*编者按：**sysAK（system analyse kit），**是\*\*\*\*龙蜥社区（OpenAnolis）系统运维 SIG 下面的一个开源项目**，聚集阿里百万服务器的多年运维经验，针对不同的运维需求提供了一系列工具，形成统一的产品进行服务。作者总结了实际工作中处理的 IO 夯问题的经验，将它梳理成一套理论分析方法并形成 iosdiag 工具，集成到了sysAK 工具集里。本文将由作者带大家一道领略一下 iosdiag 在 IO 夯领域叱咤风云的魅力。本文整理自**[龙蜥大讲堂第三期技术解读](http://mp.weixin.qq.com/s?__biz=Mzg4MTMyMTUwMQ==&mid=2247488486&idx=1&sn=47c066b12244b610fb6136323419af6d&chksm=cf66e094f811698276bf6cc5220559efcdcce9e034b79a90d6e1bab5251b60296ba839065c9d&scene=21#wechat_redirect)，\*\*直播回顾可在龙蜥社区官网查看。

作者：李光水（君然）系统运维SIG核心成员、 毛文安（品文）系统运维SIG负责人。

# **一、引言**

这是作者第二次备战双十一，怀着激动的心情迎接双十一的到来，不曾想迎来的是一枚深水炸弹：赶紧处理一下业务那边出现的 Load 高问题。

“为什么 Load 高呢？”

“因为有 500 多个进程变成了 D 状态。”

“那为什么会有这个多进程 D 状态呢？”

“因为出现了 IO 夯问题...”

曾几何时，听到 IO 夯，作者会有点头皮发麻，为啥呢？因为没有有效手段去定位这个问题，或者就算是有手段，也得经历山路十八弯，成不成还的看运气，若是幸运，还能分析点啥出来，若是不幸运，把机器整挂掉，得不偿失。时至今日，遇到 IO 夯问题再也不虚了，因为作者现在手里有可以**分析 IO 夯问题的利器\*\*\*\*——sysak iosdiag**。

先来看看这个栈，500 多个进程是因为在内核下等待某个磁盘的块设备互斥锁而进入 D 状态，如图 1-1 所示：

![图片](https://mmbiz.qpic.cn/mmbiz_png/xHicTWic3y1DsXI7duicccO0DzCmwIueQFVKSUt2UNxNUnX0ict4vUpYoJiaU2Sv38LupTEVQ3FbE2Kktiaiccn2Dsz0A/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

图 1-1

互斥锁正被执行读 IO 请求的内核进程 kworker 持有，如图 1-2 所示，只有读 IO 流程完成之后才能释放锁。但是因为 IO 夯住了，读 IO 流程无法顺利完成，所以就没法正常释放锁了。所以接下来就需要找到是访问哪块磁盘出现了 IO 夯？IO 究竟夯在哪里？

![图片](https://mmbiz.qpic.cn/mmbiz_png/xHicTWic3y1DsXI7duicccO0DzCmwIueQFVwK0Tjplaib1GUFYdeiczJcRfCZ1Tfib2KyggUnPzRiaic7QKibAfcWaI9hfA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

图1-2

之后作者使用 sysak iosdiag 工具找到了出现 IO 夯问题的磁盘，同时也定位出来这个 IO 是夯在了磁盘侧，如图 1-3 所示：

![图片](https://mmbiz.qpic.cn/mmbiz_png/xHicTWic3y1DsXI7duicccO0DzCmwIueQFVYaCsQ25gxo3OpW9Tp1Z7YfDZ5PRt2UxGic6MyWDtV4SDTfjuh3eeIpQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

图1-3

作者通过查询 virtio 的后端磁盘的硬件队列的 IO 信息，发现 IO 实际已经处理完了，进一步查询 vring 信息，发现后端没有更新 used ring，最终将问题原因锁定到了 virtio 后端。最后此问题的具体修复方法我们在此不再一一表述，总之，该工具可以方便的定界到问题出自前端驱动还是后端设备侧，节省了不少人力。

经过此问题，作者也简单做了下总结，聊一下 IO 夯的那些事。

**二、史诗级的IO架构**

在聊 IO 夯之前，了解一个 IO 会经过哪些路径还是很有必要的。网上有各式各样的 IO 架构图，足以让人看到眼花撩乱；作者从一个 IO 的生命周期的角度画了一幅图，然后描述一个 IO 在不同阶段的那些事儿（下图中去掉了部分软件层次，如 dm、lvm 等），流程有点长，请耐心看完～

![图片](https://mmbiz.qpic.cn/mmbiz_png/xHicTWic3y1DsXI7duicccO0DzCmwIueQFVNGGCLbuIrL56wNpDZ8VicNlBMZrFr1J59tmUG0V5mOe9HLznc6dIiawQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

图 2-1

1. **假设我们以一次用户态程序的写 IO 为例**，那么在调用 write 时候会传入一个数据 buf，这个 buf 在内核层面也是有对应page的。在默认情况下，IO 会以 buffer io 方式往下走；

1. **假设以 buffer io 方式往下走**，走到文件系统层，会将 1 中 buf 里面的数据拷贝到内核 page cache 中，然后把这些 page 置脏，如果此时系统的脏页水位还没有达到系统所设置的阈值，这里就返回了，对用户而言这次 IO 结束了；如果此时脏页水位达到系统所设置的阈值，那么就会启动刷脏流程，这里根据脏页水位具体达到的不同阈值，对用户进程会有不同的处理策略，如短暂休眠或不休眠，在此之后依旧会返回，用户一次 IO 结束；

1. **刷脏是通过 writeback 机制进行**，这一流程的触发，可能是通过定期触发，或者如 2 所说水位已经超过系统所设定的阈值了，又或者是用户执行了同步命令或者调用了 sync/fsync 之类的 api 等等。writeback 机制将脏页回写包装成一个个 work，然后这些 work 由系统的 worker 进程，也就是内核的 kworker 进程来执行。刷脏的过程以一个个 inode 为单位，然后将其中的脏页进一步包装成 bio 结构（bio 结构中，会有一个 bio_vec 来描述被包装的脏页 page，简单理解就是会有一个 page 指针指向这个脏页），之后将 bio 提交到 block 层；当然如果此次 IO 涉及到文件系统元数据的变更，中途内核进程 jbd 也会往 block 层提交 bio；

1. **bio 进入 block 层之后会经历一系列的限制性的处理**，如这个 bio 所包装的数据长度是否过大，有或者是否已经触发到 IO 限流，因为这些机制有点小复杂，就不在图上展示了。在此之后，会尝试与已有的 IO 请求进行合并，具体合并则根据 bio 所描述磁盘上起始扇区和长度与 IO 请求中 bio 描述的磁盘地址是否连续来进行合并。如能合并就不再往下，直接返回了；

1. **如不能合并，则申请一个新的 IO 请求**，在多队列架构中，根据磁盘的队列个数和队列深度，在磁盘初始化阶段，在内存上对每条硬件队列已经预先分配了一个 request 集合，同时会有对应的 bitmap 来表示每个请求是否已经被申请。当没有请求可以被申请到时，说明此队列上的 IO 请求已经满了，申请的进程将进入深度休眠等待队列上的 IO 请求已经成功刷到磁盘并释放 IO 请求，才能申请到；

1. **申请到之后，将 bio 包装成 IO 请求——request**，将 IO 请求添加到进程的 plug 队列中，当积蓄到一定量的 IO 请求之后，一把“泄洪”到派发队列上；（补充说明：plug队列“蓄流”机制属内核行为，用户无法控制，plug 队列能积累的 IO 请求数也是有限制的；触发“泄洪”，有可能是 plug 队列满了自己触发的，也有可能是提交完一段数据之后，主动调用的内核接口触发）；

1. **request 进入派发队列之后**，会被派发到驱动层，上图以 virtio blk 为例，request 会被封装成一个个 sg，这些 sg 里面有描述数据 page 的物理地址、本次 IO 访问的磁盘起始扇区、数据长度等信息。之后驱动将 sg 推入到 vring 缓存中，vring缓存在主机上，是主机上 virtio blk 前端和后端磁盘共享的一块可 dma 访问的内存。sg 推入 vring 之后，会 kick 磁盘提取 IO 并通过 dma 完成数据传输，IO 完成之后，磁盘给主机一个中断，virtio blk 驱动从 vring 中取出完成的 request，进入 io cpmplete 路径；

1. **request 进入 io complete 路径之后**，首先执行在 bio 结构体中设置的回调函数，这些回调函数一般在创建 bio 的的流程中指定，其一般负责唤醒等待进程、释放锁资源的操作；之后会记录 io stat 信息，这些信息也是系统 iostat 工具的指标来源；最后释放掉 request，这个释放并不是将 request 内存清空，而是清除该 reqeust 对应的在硬件队列 request 集合上的 bitmap 位，以便于后来的 IO 可以再去申请使用；

1. **至此，一个 IO 的生命周期结束**，而对于 direct io 的方式，整个过程会缺少数据buf复制到内核 page cache、脏页回写这个步骤。上面提到的流程可能不是一个完成 IO 生命周期的全部，由于 IO 链路的复杂性，中间也省掉了部分流程，有兴趣的读者可以再去摸索摸索，或者加入我们系统运维 sig 交流群，欢迎一起探讨。

# **三、臭名昭著的IO夯**

## **3.1 何为IO夯**

IO 夯，可简单理解为 IO 路径在一定程度上堵住了，轻则经过特定路径的 IO 不可访问，重则整条 IO 路径堵住不可用，任你多少 IO 丢下来，我就是没反应。为什么 IO 路径会堵住呢，无外乎是在等待资源。

等待资源一般涉及的是 IO 路径上不可重入的临界区，要求进程持有资源进入、释放资源退出，又或者是事物处理型，允许接受有限个进程的事物，但要等待这些进程的事物全部被处理完之后，才能接收新的进程事物，开始处理新一轮的流程。而当处于临界区内的进程，由于内核 bug 或存储介质原因，导致无法顺利完成 IO 后正常退出，最终造成临界区外的进程因为拿不到资源而处于阻塞状态，导致无 IO 可用。由此可见，只要临界区内的进程不退出这种尴尬的状态，整条 IO 路径就不可用。
!\[\[Pasted image 20240923191615.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图 3-1

内核下有条 io complete 的关键路径，这条路径属于可重入路径，其主要职责为对 IO 结束后的收尾工作，一般地，会先执行一个 IO 回调流程，而后更新一些 IO 的stat信息，最后结束生命周期。内核中也有一些特殊类的 IO，在 IO 子系统中的处理方式与一般的 IO 有所差异，如 flush/fua io，而作者曾经碰到过一起 flush/fua io 夯的问题。flush/fua io 在使用日志型的文件系统场景下可能会比较常见，如 ext4 文件系统，为了保证文件系统元数据能够真正的持久化存储到磁盘的日志区域，jbd 线程在提交 commit record 的时候会发起这种 IO，而flush/fua io的处理流程也是比较繁琐的：
!\[\[Pasted image 20240923191620.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图 3-2

如上图所示，IO 子系统在处理这个 IO 时，会先发起一个 flush io 到磁盘（在内核中唤作pre-flush），等这个 IO 结束之后，再发起真正的数据 IO，当磁盘执行完数据 IO 之后，再此发起一个flush io（在内核中也唤作 post-flush），终于 flush io 结束之后，数据 IO 进入complete 路径。在这个数据 IO 里面，一般会存在比较关键的 IO 回调，其涉及到释放 buffer head 的 lock bit 或者 page cache 的 lock bit，而且往往等待这个 lock bit 的是 jbd 线程本身；作者曾经就遇到 pre-flush io 因为 race 问题被直接释放掉了，导致后续的 IO 流程没有走到，data io 没有得到处理，buffer lock 得不到释放，造成 jbd 夯住，最终引起这个分区的 IO 都得不到响应。

经过本小节的介绍，结合章节 2 的 IO 结构图（图 2-1），可以发现 **IO 夯是可以发生 IO 路径上的任何地方**。

**3.2 IO夯的危害**

从 3.1 可知，IO 夯会造成有 IO 需求的进程无 IO 可用；从业务稳定性角度来看，对于那些有 IO 访问需求的业务进程，IO 夯可能会引起进程长期阻塞，且在 IO 路径恢复之前，都无法对外提供服务。从系统稳定性角度来看，IO 夯可能会引起大量的进程进入 D 状态，导致系统高负载，甚至系统夯住，shell 命令无法执行，机器无法登陆，最终不得不重启系统去解决。

**3.3 IO夯问题分析方法现状**

当遇到 IO 夯问题时，我们通常会分析 dmesg 中 hungtask 调用栈、或者是 iosta 信息、sysfs/debugfs 中的统计信息，再结合以往经验去推测问题可能出在哪。当我们碰到如下图 3-3 所示的 iostat 信息时，根据经验，会怀疑是磁盘侧有 IO 没回，因此怀疑io夯在磁盘上，让存储的同学去排查磁盘侧。但这种经验却不一定靠谱，如果是在磁盘返回到 io complete 之间有内核 bug，iostat 也会出现下图中的信息。
!\[\[Pasted image 20240923191626.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图 3-3

\*\*那如何分析 IO 夯问题是最有效的呢？\*\*答案肯定是要找出来夯住的 IO 请求，然后根据请求里面的信息去分析当前这个请求是处于什么状态、已经走到哪个路径了。但遗憾的是，目前没有实现这个功能的通用工具，唯一能快速实现这一需求的，就只有对问题现场做 crash 分析了，找到夯住的 IO 请求，根据 IO 请求中的信息，再结合代码流程，一步步深入最终找到问题原因，但前提是业务可以容忍这么操作。

**3.4 利器简介——sysak iosdiag**

sysAK iosdiag，是 sysAK 工具平台中的 IO 诊断工具，已具备 IO 时延探测、IO 夯诊断两大功能，其中 IO 夯诊断可用于检测当前系统中 IO 夯事件并确定问题边界。工具的大体架构图 3-4 所示：
!\[\[Pasted image 20240923191634.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图 3-4

首先通过 sysAK 的 iosdiag 功能去使能 IO 夯诊断，这里诊断到 IO 夯之后，会对 IO 进行数据分析，然后形成诊断结论，诊断结论是以 json 的数据格式保存在一个日志文件里面，同时也支持将数据上传到指定的地方，**目前支持 oss 的上传方式**，不上传的话，数据也会存在机器本地，供调用者去查看。
!\[\[Pasted image 20240923191641.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图 3-5

工具的性能开销情况：单核 cpu 低于 1%、内存消耗低于 10MB、消耗少许磁盘空间保存诊断结果。

工具支持的内核版本种类：3.10/4.9/4.19多种版本。

iosdiag 诊断结果输出，力求信息准确、结果直观，期望即便不具备内核IO子系统知识的同学也能快速上手。工具会输出一些结论性的信息，如什么时间点，检测到了 IO 事件，这是一个什么样的 IO，从哪个 cpu 发出来的，从哪个磁盘的哪个位置访问多大的数据量，然后这个 IO 夯在哪个路径上，夯住了多久。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图 3-6

# **四、TODO**

工具目前只能覆盖到进入内核 block 层的 IO，在 block 之上的覆盖不到，因此作者目前也在研究如何扩大工具的覆盖面；其次，工具在结果输出上还不够直观，使用者还无法简单明了地从输出信息上看到问题，针对这一点，作者也一直在优化。

**往期精彩推荐**

[1.sysAK（青囊）系统运维工具集：如何实现高效自动化运维？| 龙蜥技术](http://mp.weixin.qq.com/s?__biz=MzUxNjE3MTcwMg==&mid=2247486688&idx=1&sn=7333426e446892d013b49593014b086e&chksm=f9aa3e39ceddb72fdf980a8159787d264b922ef28d24ce4ebf18f0d26e45085caf0c338a1002&scene=21#wechat_redirect)

[2.直播回顾：如何基于Linux内核构建起商用密码基础设施？| 龙蜥技术](http://mp.weixin.qq.com/s?__biz=MzUxNjE3MTcwMg==&mid=2247486676&idx=1&sn=bc4027417ab1f1831b6aec9124eddec5&chksm=f9aa3e0dceddb71be9feded1388e2721a896586be080e4a8bb9874522f06f703aecf95a840d9&scene=21#wechat_redirect)

[3.关于龙蜥，你关心的问题都在这里](http://mp.weixin.qq.com/s?__biz=MzUxNjE3MTcwMg==&mid=2247486673&idx=1&sn=a559f89f88d040347b14f7088460b0c6&chksm=f9aa3e08ceddb71ed9a7e791077f071acef5f3cebf5b0bb3b58b49a5fb8df9a3ca1e8eb2e6b2&scene=21#wechat_redirect)

[4.Arm加入龙蜥社区并成为理事单位，国内开源再添国际新力量](http://mp.weixin.qq.com/s?__biz=MzUxNjE3MTcwMg==&mid=2247486657&idx=1&sn=240aa906a7cfcfeaa4e5e506a611ec10&chksm=f9aa3e18ceddb70e72524cd4c54a5ff3cb0150cb528f4347b9d2fdccf9fa685390cffe64618d&scene=21#wechat_redirect)

[5.祝贺龙蜥社区理事单位成员入选C++ 标准委员会，首个国内企业代表加入](http://mp.weixin.qq.com/s?__biz=MzUxNjE3MTcwMg==&mid=2247486637&idx=1&sn=6382b573e3cb7a57bd35b8929bfaad1f&chksm=f9aa3e74ceddb7622b0cf29c97841ab5e985f5ab4c113f0348bfd536411fc9ebcb7deb6665f0&scene=21#wechat_redirect)

[!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MzUxNjE3MTcwMg==&mid=2247486688&idx=1&sn=7333426e446892d013b49593014b086e&chksm=f9aa3e39ceddb72fdf980a8159787d264b922ef28d24ce4ebf18f0d26e45085caf0c338a1002&scene=21#wechat_redirect)

阅读 630

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/bDZYPQUwvDnKe3dtY0vLjWoesXQTtm9b7FibLgECY6rl5Hz4C7NYVQf26icepTa1yUOdbxTNWFxYNm9g5Fv0iaMRA/300?wx_fmt=png&wxfrom=18)

云巅论剑

31在看

写留言

写留言

**留言**

暂无留言

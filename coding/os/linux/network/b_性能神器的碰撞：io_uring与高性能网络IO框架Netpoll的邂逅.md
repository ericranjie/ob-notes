
Linux云计算网络

 _2024年06月10日 09:36_ _广东_

以下文章来源于字节跳动SYS Tech ，作者字节跳动STE团队


**字节跳动SYS Tech**.

聚焦系统技术领域，分享前沿技术动态、技术创新与实践、行业技术热点分析。

](https://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247508030&idx=1&sn=cebb9dd8c7efd2314e2000f78a342541&chksm=ea779486dd001d90daa2425c29e46581aeb5ae603e871c6e465c23eae1789eacab5640693ae7&mpshare=1&scene=24&srcid=06101wnuvxzFZfvAekSZqri1&sharer_shareinfo=5c42252de21551c3020cadbcedc5fe44&sharer_shareinfo_first=5c42252de21551c3020cadbcedc5fe44&key=daf9bdc5abc4e8d09a6c7bd3fccc966f77215f3393615502b8b84899304357dd2420a95707b199f102c16875635304e05ea19b8a9bd15d51c28116383e5ea49bda42406541cf4f16e41f7814f28e4b9b214a6f74a4d6b270fbb0a807345e11259728cf10c53cae91d0619944737981c647aba45ee43482c9474ef90dfb0c26f1&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090621&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQQKi%2B1eabcBzGw3uKGFZHBhLmAQIE97dBBAEAAAAAAFidMk4ZNTgAAAAOpnltbLcz9gKNyK89dVj0mnd64s%2FRYeOM42vYfjBk7X1u%2BFbi60K05Aky0zBqkrpqElPOpg2CuMFsL05mKzMNQM54B67riR2pAXa%2FhothkfyWDb8YYCrPOSbTC2L%2FUvqFFzSk0jfD4JEgLGzWU5s7x92%2FIXslTMZSOh0%2BsUrgJ8%2FkG31XdfO5vdFNQlS05r9vWzMMBN8W4a6rGQhm3Yk4j%2FCehAgO5gjXS3yVB6g9AxXqQgGeZXFMCc8H1rlc%2Fi8tR7WHWB06CGfsYX8D56%2Fr&acctmode=0&pass_ticket=hHg0fMjwM%2FA1xn0m4JIZiuqy%2FchXLkNm9585SJ%2FYTETmJSXzDrQn55YdataNu2XD&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1#)

![](https://mmbiz.qpic.cn/mmbiz_png/1TDxR6xkRSEplO6BjCUUzQ0dGo5KhZ6d3HYZCTGyWqauM0BLWKKtSOicg4czIvKTLbXB6frzdf5x9PY8jzbicZkg/640?wx_fmt=png&wxfrom=13&tp=wxpic)

**简介**

Netpoll 是字节跳动基于 epoll 自研的一款专注于 RPC 场景的高性能 NIO 网络库。相对于 Go 语言原生的 net 网络库，netpoll 能够实现对网络层的控制权，可以先于业务做一些探索并最终赋能业务。Netpoll 目前是基于事件循环和同步数据流 API 设计的 NIO 网络库，io_uring 是 linux 5.15 版本推出的全新异步 IO 接口，相对于传统异步 IO 方案，io_uring 具备以下优势：

（1）**减少系统调用**：通过 batch 的模式，可以将多次请求合并成一次，减少了系统调用次数。对于一些高性能场景，可以选择polling 模式完全消除系统调用。

（2）**可扩展性好**：通过提供一个统一的异步 IO 框架，可以支持各种类型的IO 操作（如文件，套接字等），从而让软件架构更加简洁优雅。

所以本文希望探索将 io_uring 集成到 Netpoll 的最佳实践，详细分析 Netpoll 与 io_uring 之间架构之间的差异并提出集成设计方案，最后通过实验证明，通过集成 io_uring 给 Netpoll 带来的价值：本文所提出的方案通过 echo server 进行基准测试，结果表明该设计可以将**吞吐量提高 10% 以上，同时将延迟降低 20-40%**。该设计背后的关键思想是通过使用 io_uring 接口消除系统调用所引入的开销。通过该方案可以实现**零系统调用**。

  

**Netpoll 设计**

Netpoll 目前的架构设计如下。

  

**Poller contexts**

在初始化阶段，Netpoll 会创建一个主服务器 poller 上下文和一组其余的 poller 上下文，它们都是基于 goroutines 来实现事件处理循环。
![[Pasted image 20240920121642.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

main server poller 监听 server fd 用于接受新的连接。main server poller 事件循环将新连接的 fd 以负载均衡的方式分配到其他 poller。Netpoll的默认设置是为每 20 个 cpu 创建一个 poller。在满负载的服务器中，所有连接都均衡地分布到这些poller中，每个 poller 负责处理连接上的各种事件。

  

**收方向**

正如上文解释的，每个连接都被分配到其中一个 poller 中，Kernel 通过触发 EPOLLIN 事件的方式通知 poller 连接有数据可被读取。
![[Pasted image 20240920121648.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

事件循环分配好输入缓冲区，执行recv系统调用，然后从池中分配一个 goroutine 来执行用户注册的 task handler 程序（应用程序逻辑）。从这里开始，task 上下文与 poller 上下文并行执行。用户处理程序负责释放输入缓冲区。poller在处理完所有事件后进入事件等待状态。

  

**发方向**

Netpoll 的 connection 类为用户提供了接口，用来分配 output buffer，并通过网络发送数据。应用通过调用 connection.Flush API  来发起网络传输。
![[Pasted image 20240920121654.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

sendmsg 系统调用在用户 task 上下文中被执行。如果内核目前无法完成数据传输，poller 配置会从 read 模式变为 read-write 模式，这意味着 poller 同时开始监听 EPOLLIN 和 EPOLLOUT事件。当 socket 准备好发送更多数据时，将触发 EPOLLOUT 事件，并从轮询器上下文触发进一步的 sendmsg 系统调用。

  

**io_uring 模型**

本章节总结了 io_uring 接口的 IO 模型，本文只关注网络 IO 所需的特性。单个io_uring由提交队列（SQ）、完成队列（CQ）和缓冲区队列（PBQ）组成。
![[Pasted image 20240920121707.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

io_uring 提供了一种对文件描述符执行异步 IO 操作的方法：https://github.com/axboe/liburing/blob/master/src/include/liburing/io_uring.h#L188。

就其本质而言，文件描述符与 io_uring 实际没有任何关系。io_uring 只是一种请求内核对文件描述符执行操作的机制。可以选择使用不同的 io_uring 对同一个文件描述符执行 IO 操作。反之亦然，单个 io_uring 也可以用于对多个文件描述符执行 IO 操作。一个 SQ entry 包含文件描述符和操作码。由于它是异步 IO，CQ 用于通知用户空间已完成 IO 操作。

  

对于 SQ，有两种类型的 entry 分别称为 singlesshot 和 multishot。singleshot, 如名称所指，内核每次处理一个 SQ entry，就为其生成一个 CQ entry。如果应用在同一 fd 上有更多相同类型的请求，则必须再次相应的插入同样的 SQ entry。Multishot 是一种优化，为避免在SQ中一次又一次地插入相似的entry，应用程序只需要插入一次 multitshot SQ entry，完成操作后，内核则会在相应的 fd 上生成多个 CQ entry。Multishot 对于接收（RECV）等操作类型帮助较大，用户通过提交 multishot SQ entry 来通知内核希望读取给定fd上的数据，之后，每当收到数据时，内核都会持续生成 CQ entry。为了让 multishot 工作，用户空间分配的缓冲区需要提前被分配并通知给内核，这就是所谓的 PBQ，用来将用户空间缓冲区共享给内核的地方。

  

数据从用户空间流入（recv）和流出（send）有两种方法。第一种方法是，用户显式地陷入内核态来消费 SQ entry 或生成 CQ entry。另一种是通过将 io_uring 设置IORING_SETUP_SQPOLL标志的方式，这个标志设置后，会在内核中创建一个 SQ 轮询线程，该线程消费 SQ entry 并为 recv 生成 CQ entry。两种方法之间的基本区别在于：一种是对 SQ 和 CQ 的操作发生在用户程序上下文中，另一种是发生在单独的内核线程中。SQ 轮询线程有一个空闲时间段，在空闲时间超过设置的值后，它通过设置一个位来通知用户自己将进入睡眠状态。一旦线程进入睡眠状态，用户需要负责在合适的时间唤醒线程。
![[Pasted image 20240920121716.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

让我们绘制一个IORING_OP_SEND和IORING_OP_RECV的端到端工作流程，以进一步增强我们对 io_uring 的理解。

  

**IORING_OP_SEND**
![[Pasted image 20240920121725.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

发送工作流程很简单，但 io_uring 的两种不同配置模式之间存在一些差异。在左边，用户向 SQ 中插入多个 entry，并决定何时准备好进入内核处理请求。对于批处理的责任在于用户，能减少多少系统调用的程度取决于批处理的数量。将数据从用户空间复制到内核的sock_send_msg在用户空间线程上下文中调用，CQ entry 在返回用户空间之前生成。在右边，内核线程负责执行sock_send_msg，用户不需要进入内核。用户可以在这段时间做一些其他工作，例如处理 CQ entry 或进入内核等待 操作完成。如果用户可以在内核处理操作和生成 CQ entry 时执行其他活动，可以实现零系统调用。

  

**IORING_OP_RECV**

接收过程比较复杂，因为它依赖于外部读取的触发。用户在提交一个 multishot entry 后，内核通过 VFS 层在 fd 上创建一个轮询事件。这个过程只需要发生一次，这就是 multishot 的意思。
![[Pasted image 20240920121732.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

io_uring 设计利用 Linux 中的task_work机制将数据读取到用户空间缓冲区。顾名思义，task_work代表每个任务（线程）并以链表形式组织，在每次上下文返回用户空间时，例如，在系统调用结束返回用户空间之前，内核都会遍历该链表。因此，当数据到达轮询 fd 时，会为处理 multishot 的 SQ 条目的线程创建一个 task_work 条目。每当用户进行系统调用或有意进入内核等待完成时，任务就会被执行，或者在 SQ_POLL线程的情况下，它会在列表上循环以执行 task_work。
![[Pasted image 20240920121737.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**Batching in io_uring**

从理论上讲，epoll和io_uring都依赖于 Linux 的vfs_poll 系统调用，但是将数据流式传输给用户态的方式有本质的不同。io_uring 在内核内部以task_work链表的形式进行批量处理。为了理解区别，让我们以单线程echo服务器为例，基于epoll的单线程 echo 服务器将等待内核中的事件，然后通过recv和 send系统调用来处理请求。而如果使用 io_uring，read 操作则会由内核执行。
![[Pasted image 20240920121742.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

batching 操作可以显著提高吞吐，但如果处理不当，会导致更多的时延，在 echo ping pong 测试中，下面的数据可以证明。
![[Pasted image 20240920121751.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
![[Pasted image 20240920121755.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在 1KB 的 echo server 基准测试中，io_uring 提供了一致的良好吞吐量，但延迟也增加了 10% 以上。

  

因此，在使用 io_uring 时，应始终注意隐藏的读取批处理将带来的影响。

  

**Netpoll integration to io_uring**

Netpoll 的主要设计特征是有固定数量的 goroutine 运行事件处理循环并映射到它们负责的连接。用户处理程序则从池中选择 goroutin 进行执行，执行结束后，goroutin 会放回池中等待下次冲用。这使得 go 运行时更加轻量级，并为 IO 操作提供良好的尾部延迟。我认为这是 Netpoll 设计的关键原则，基于 io_uring 的方案也应该遵循这一原则。

  
![[Pasted image 20240920121805.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**方案架构**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**初始化**

- 默认配置中，每个 Poller context 会负责管理 20 个 CPU
    
- 每个 poller 创建 21 个 uring，1 个负责接收，剩下的 20 个负责发送
    
- uring 以 SQ Poll 的模式创建：
    
    （1）接收 uring 会在 kernel 创建一个专用的 SQ poll 线程，接受 uring 在创建时还会准备好缓冲区 ring（PBQ）。
    
    （2）所有的发送 uring 只会创建两个 SQ poll 线程。io_uring 允许通过使用 ATTACH_WA 的方式，让多个 uring 共享 SQ poll 线程。发送 uring 平均的分不到创建的 SQ poll 线程上。
    

  

**Accept工作流**

- 新连接在 recv uring 中被设置成 multishot 模式
    
- 除了将新连接映射到 poller，还需要将连接随机映射到其中一个 send uring 中
    

  

**Receive 工作流**

- poller 处理recv uring 上的 CQ entry。poller 会从池中选择 goroutine传递 buffer 和执行用户注册的 handler。
    
- poller 还需要处理所有连接已经释放的buffer，并将其重新插入 PBQ用来接受后续的数据。
    

  

**Send 工作流**

- 用户处理程序通过映射到连接的 uring 发送数据
    
- 每次通过 uring 的发送都需要使用锁进行保护。这把锁的竞争不会过于激烈，因为 uring 的数量设置为和物理执行单元数量一样。然后，如果发现比较激烈的竞争，uring 数量可以进一步增加。
    
- poller 还需要负责处理 CQ entry。在处理过程中，buffer 将会被释放。
    

基于 io_uring 的方案和 Netpoll 原有方案的对比：

- goroutine 的数量保持不变
    
- 用户态 send 和 recevie 并发执行单元数量保持不变。每 20 个 CPU 使用一个接收执行单元，发送单元数量和 CPU 数量一致。
    
- 内核态负责接收（copy_to_user）的执行单元数量和 poller 数量一样。不同点在于，现有设计 copy_to_user 是在 poller 中被执行，基于 io_uring 的方案中，是在 SQ poller 内核线程中被执行。
    
- 内核态负责发送（copy_from_user）的数量，在 io_uring 方案中，被设置为和 SQ poller 内核线程数量一致，也就是 2 个。在现有设计中，所有的核都需要参与发送。
    
- 最后一点也是最重要的一点，基于 io_uring 的方案实现了零系统调用，poller 线程运行在两边（用户态和内核态），使用 io_uring 作为通信接口。
    

关于 polling 的说明：

- SQ poller 内核线程是动态的，在一段时间没有时间后，将会陷入睡眠。一旦有用户态或内核态的 IO 活动后，马上会被唤醒。
    
- 用户空线程也可以采用相同策略，能够动态进行 polling 和陷入睡眠。
    
      
    

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**io_uring 方案基准测试**

我们编写了一个 go 语言的 echo（ping-pong）server 来对 io_uring  方案进行基准测试。服务器创建以 PBQ multishot 模式创建一个 receive uring 和与 CPU 数量一样多的 sender urings。至于事件循环，它使用busy polling模式来处理所有 uring 的 CQ entry。对于接收到的数据（ping），它会创建一个 goroutine，执行 pong 并以随机的方式选择一个发送  uring。pong goroutine 在分配的 uring 上发出发送请求。每个发送者 uring 都有一个锁来保护任何并发访问的情况。  

  

测试程序在物理机器上运行。整个工作负载使用 taskset 限制在 32 个 CPU。此外，通过设置 softirq 以将内核与用户应用程序运行的 core 分开。在 32 个 CPU 中，11 个 CPU用于 softirq 网卡工作负载，其余 21 个 CPU 用于应用程序。

  

**分析**

所有的实验都是用 1KB 进行测试，因为它是一个标准的测量单位用来测量 Netpoll RPC 工作负载。本文测量了三种不同配置模式的 uring。uring-2 表示使用两个内核 SQ poll 线程负责 sender uring，uring-4 表示使用 4个内核 SQ poll 线程。uring-hyb 是一种混合方法，其中事件循环基于 io_uring（不使用内核 SQ poll 模式），发送是通过 sendmsg 系统调用进行的。这些数字是通过在另一台服务器上运行客户端 60 秒来进行测量的。

  

**（1）吞吐量**
![[Pasted image 20240920121925.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
![[Pasted image 20240920121930.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
![[Pasted image 20240920121935.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

最后的图表显示了使用 1% 的 CPU 利用率可以实现的吞吐量。对于连接数较低的场景（100），系统没有满载运行，并且由于其轮询模式导致较高的 CPU 利用率，使得每 1% 的 CPU 利用率的吞吐量低于 epoll。在 RPC 模式下，较高的连接意味着会有更多的网络数据进入网络协议栈导致整个系统更加强繁忙，从而有效地利用了轮询模式。**对于 200 以上的连接，uring-2 配置的吞吐量增加了 10% ~ 15%，uring-4 与 epoll 的吞吐量差距不大**。

  

**（2）时延**
![[Pasted image 20240920121941.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
![[Pasted image 20240920121946.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

用户空间和内核中的轮询模式有两个主要优点。首先，它解决了内核内部读取的 io_uring 批处理问题，因为读取的数据对用户空间可见的速度要快得多，从而优化了延迟增加的问题。其次，系统调用的计数变为零。这两个因素都有助于在两种 uring 配置中降低延迟。表中的负数表示相比于 epoll 延迟百分比的减少。**uring-2 配置显示 10% ~ 20%，uring-4 显示 30% ~ 40% 的延迟减少。uring-4 对于延迟的优化更加明显是由于执行发送的并行线程更多。**

  

对于 uring-2 1000 连接场景的测试将会在后续的工作中进行。uring-hyb 的 latency 测试并没有在上图中展示，实际测试中发现测试结果和 epoll 类似。

  

**Netpoll集成io_uring性能测试**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**环境**

1. 内核版本：6.4

2. server 和 client 分别运行在两个使用 25G网卡直连的机器上

3. 机器信息：

  （1）使用的 CPU 总数：30

    - 通过将网卡队列设置成 11 来限制内核用于收包和网络处理的 CPU 数量

    - 19 个 CPU 用于 server app

  （2）单个 poller 设置

    GOMAXPROC 的值为 20，所以 Netpoll 只会为 19 个 CPU 创建一个事件循环

  

运行如下图所示：
![[Pasted image 20240920121959.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

对于基于 io_uring 的 server app，需要保留额外的 CPU 用于 sqpoll 线程。例如，如果配置 server app 使用 3 个 sqpoll 线程，19 个用于 server app 的CPU 将会预留 3 个 CPU 用于 sqpoll。在这个 case 中，server app 会用"numactl -C 14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29 -m 0"命令启动，CPU 11,12,13 被 pin 住用于 sqpoll。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**测试结果**

一次执行的配置如下：

- 用于测试的 value 大小为 1KB
    
- 测试次数为 50 million 次，这大概需要花费 50～90 秒，具体时间取决于 server app 的吞吐
    
- 并发链接数量：10, 50,100, 200, 300, 400, 500, 1000
    

io_uring 配置：

- 一共17 个 rings：一个用于收方向，16 个用于发方向
    
- 使用两个 sqpoll线程处理 16 个发方向 ring。发方向 ring 都被配置成 SQPOLL。
    
- 配置 1: 和发方向一样，收方向 ring 也使用 SQPOLL，这样在这种配置下，一共使用三个sqpoll线程，
    
- 一个用于收方向队列，两个用于发方向队列。
    
- 配置 2：收方向 ring 不使用 SQPOLL，只有两个 sqpoll 线程用于发方向队列。
    

  

**（1）吞吐量**
![[Pasted image 20240920122006.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
![[Pasted image 20240920122010.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
![[Pasted image 20240920122015.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

基于 io_uring 的 Netpoll server，在相同 CPU资源的情况下， **提升了 10%～15%的吞吐量**。

  

**（2）时延**
![[Pasted image 20240920122020.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
![[Pasted image 20240920122027.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

基于 io_uring 的 Netpoll server，**在 P99 时延测试中，所有场景下都有 20%～40% 的提升。对于 P9999 延迟测试，对于一些场景有 10%～20% 的提升**，剩下场景基本和原生方案持平。

  

**（3）系统调用次数**
![[Pasted image 20240920122245.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

基于 io_uring 的 Netpoll server，在 polling 模式下，可以将系统调用次数降至 0，在非 poling 模式下，系统调用次数也最多降了 15 倍。

  

**总结**

从上述实验数据可以看出，Netpoll 在集成了 io_uring 后，吞吐和时延都带来了较为显著的性能提升：**吞吐量提高 10% 以上，同时延迟降低 20-40%**。该方案中 io_uring 接口消除系统调用所引入的开销，实现了**零系统调用**，可以帮助开发者提升网络性能。但是目前方案代码还未合并至 Netpoll，欢迎感兴趣的开发者可以根据实际业务需求，自行进行测试和试用。

  

![](http://mmbiz.qpic.cn/mmbiz_png/1TDxR6xkRSFD7ibPKdP9JOqxFxp199B6LLCw80xbTAicxIQPgpsq5ib8kvx7eKfq6erLb1CAT4ZKa3RSnl9IWfELw/300?wx_fmt=png&wxfrom=19)

**Linux云计算网络**

专注于 「Linux」 「云计算」「网络」技术栈，分享的干货涉及到 Linux、网络、虚拟化、Docker、Kubernetes、SDN、Python、Go、编程等，后台回复「1024」，送你一套 10T 资源学习大礼包，期待与你相遇。

93篇原创内容

公众号

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

阅读 708

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/1TDxR6xkRSFD7ibPKdP9JOqxFxp199B6LLCw80xbTAicxIQPgpsq5ib8kvx7eKfq6erLb1CAT4ZKa3RSnl9IWfELw/300?wx_fmt=png&wxfrom=18)

Linux云计算网络

3581

写留言

写留言

**留言**

暂无留言
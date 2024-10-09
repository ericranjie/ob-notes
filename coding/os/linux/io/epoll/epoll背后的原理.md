极客重生

_2021年12月20日 12:30_

以下文章来源于Qunar技术沙龙 ，作者冯志明

\[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM4zfH9WsPjiaCjqflC0o4ZzKiac2TSlUGZKk0vz0COz2Jsg/0)

**Qunar技术沙龙**.

Qunar技术沙龙是去哪儿网工程师小伙伴以及业界小伙伴们的学习交流平台。我们会分享Qunar和业界最前沿的热门技术趋势和话题,为中高端技术同学提供一个自由的技术交流和学习分享平台。

\](https://mp.weixin.qq.com/s?\_\_biz=MzkyMTIzMTkzNA==&mid=2247563937&idx=2&sn=ae50d7fe2997b4609cb749d3fa1c3d6f&chksm=c1850df0f6f284e64767e787734bc53ef3234c68749566f2fcb04478a3778ecfd890cca752ef&mpshare=1&scene=24&srcid=1220UNb3LtoN6Sb5hQ0rAUZ5&sharer_sharetime=1639979706270&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0f4af897b334a1f99373e485b6d34f73547a54cc6c07471ca8b7ab3f218c12e3aa3948366ebd9bc918aa015953e8af9de18cf0ef07f28b5cf3e7fce5117eb50ef24b7bbbadb15ec60534f548e855a0b148910c5dfcfbea2fad3bce6504263d7efc36a31bb5125053313fe35eb530f166f91efd3cf74b32cd0&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQTTrNVsKeTe%2B3UAgfvpkPAxLmAQIE97dBBAEAAAAAAAyuL%2FrzeJ0AAAAOpnltbLcz9gKNyK89dVj0I50h4Fx9iyhOYLnhyVyb6rJ9szn0dkMI%2B4YqCQqJkVTWCcI9JKSxx07WRtYERNWf49JOB3mDvjrylwErfxgQtV%2FMnpIiMaAGZlWTuIxGa2a0R42XJVR9%2FQHqewSQl8hwfpjWV%2FqboXF0ErnaFCwaKUuDEUDPOw1s5iHjPgqANvTZnV4TRRT9hvBWpB%2Fl2aHf%2BQBaVwiPEB6AmBfA8tHEQ17saIYkUHxB%2Fto83z7AVGkvsYmcCNYiMZbnIuzyBuyV&acctmode=0&pass_ticket=VjZoPP4h4kGl%2Bm8sw33zja1%2FH9yY2YT5zr0bCqCsB5WLmUViXGnPp4m7lLv8HptZ&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7351805-zh_CN-zip&fasttmpl_flag=1#)

## **1 简介**

Epoll 是个很老的知识点，是后端工程师的经典必修课。这种知识具备的特点就是研究的人多，所以研究的趋势就会越来越深。当然分享的人也多，由于分享者水平参差不齐，也产生的大量错误理解。

今天我再次分享 epoll，肯定不会列个表格，对比一下差异，那就太无聊了。我将从线程阻塞的原理，中断优化，网卡处理数据过程出发，深入的介绍 epoll 背后的原理，最后还会 diss 一些流行的观点。相信无论你是否已经熟悉 epoll，本文都会对你有价值。

### **2 引言**

正文开始前，先问大家几个问题。

1、epoll 性能到底有多高。很多文章介绍 epoll 可以轻松处理几十万个连接。而传统 IO 只能处理几百个连接 是不是说 epoll 的性能就是传统 IO 的千倍呢？

2、很多文章把网络 IO 划分为阻塞，非阻塞，同步，异步。并表示：非阻塞的性能比阻塞性能好，异步的性能比同步性能好。

- 如果说阻塞导致性能低，那传统 IO 为什么要阻塞呢？

- epoll 是否需要阻塞呢？

- Java 的 NIO 和 AIO 底层都是 epoll 实现的，这又怎么理解同步和异步的区别？

3、都是 IO 多路复用。

- 既生瑜何生亮，为什么会有 select，poll 和 epoll 呢？

- 为什么 epoll 比 select 性能高？

PS：

本文共包含三大部分：**初识 epoll、epoll 背后的原理 、Diss 环节**。

本文的重点是介绍原理，建议读者的关注点尽量放在：“为什么”。

Linux 下进程和线程的区别其实并不大，尤其是在讨论原理和性能问题时，因此本文中“进程”和“线程”两个词是混用的。

### **3 初识 epoll**

epoll 是 Linux 内核的可扩展 I/O 事件通知机制，其最大的特点就是性能优异。下图是 **libevent**(一个知名的异步事件处理软件库)对 select，poll，epoll ，kqueue 这几个 I/O 多路复用技术做的性能测试。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

很多文章在描述 epoll 性能时都引用了这个基准测试，但少有文章能够清晰的解释这个测试结果。

这是一个限制了100个活跃连接的基准测试，每个连接发生1000次读写操作为止。纵轴是请求的响应时间，横轴是持有的 socket 句柄数量。随着句柄数量的增加，epoll 和 kqueue 响应时间几乎无变化，而  poll 和 select 的响应时间却增长了非常多。

可以看出来，epoll 性能是很高的，并且随着监听的文件描述符的增加，epoll 的优势更加明显。

不过，这里限制的100个连接很重要。epoll 在应对大量网络连接时，只有活跃连接很少的情况下才能表现的性能优异。换句话说，epoll 在处理大量非活跃的连接时性能才会表现的优异。如果15000个 socket 都是活跃的，epoll 和 select 其实差不了太多。

**为什么 epoll 的高性能有这样的局限性？**

问题好像越来越多了，看来我们需要更深入的研究了。

### **4 epoll背后的原理**

#### **4.1 阻塞**

**4.1.1 为什么阻塞**

我们以网卡接收数据举例，回顾一下之前我分享过的网卡接收数据的过程。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

为了方便理解，我尽量简化技术细节，可以把接收数据的过程分为4步：

1. NIC（网卡） 接收到数据，通过 DMA 方式写入内存(Ring Buffer 和 sk_buff)。

1. NIC 发出中断请求（IRQ），告诉内核有新的数据过来了。

1. Linux 内核响应中断，系统切换为内核态，处理 Interrupt Handler，从RingBuffer 拿出一个 Packet， 并处理协议栈，填充 Socket 并交给用户进程。

1. 系统切换为用户态，用户进程处理数据内容。

网卡何时接收到数据是依赖发送方和传输路径的，这个延迟通常都很高，是毫秒(ms)级别的。而应用程序处理数据是纳秒(ns)级别的。也就是说整个过程中，内核态等待数据，处理协议栈是个相对很慢的过程。这么长的时间里，用户态的进程是无事可做的，因此用到了“阻塞（挂起）”。

**4.1.2 阻塞不占用 cpu**

阻塞是进程调度的关键一环，指的是进程在等待某事件发生之前的等待状态。请看下表，在 Linux 中，进程状态大致有7种（在 include/linux/sched.h 中有更多状态）：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

从说明中其实就可以发现，“可运行状态”会占用 CPU 资源，另外创建和销毁进程也需要占用 CPU 资源（内核）。重点是，当进程被"阻塞/挂起"时，是不会占用 CPU 资源的。

换个角度来讲。为了支持多任务，Linux 实现了进程调度的功能（CPU 时间片的调度）。而这个时间片的切换，只会在“可运行状态”的进程间进行。因此“阻塞/挂起”的进程是不占用 CPU 资源的。

另外讲个知识点，为了方便时间片的调度，所有“可运行状态”状态的进程，会组成一个队列，就叫\*\*“工作队列”\*\*。

**4.1.3 阻塞的恢复**

内核当然可以很容易的修改一个进程的状态，问题是网络 IO 中，内核该修改那个进程的状态。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

socket 结构体，包含了两个重要数据：进程 ID 和端口号。进程 ID 存放的就是执行 connect，send，read 函数，被挂起的进程。在 socket 创建之初，端口号就被确定了下来，操作系统会维护一个端口号到 socket 的数据结构。

当网卡接收到数据时，数据中一定会带着端口号，内核就可以找到对应的 socket，并从中取得“挂起”进程的 ID。将进程的状态修改为“可运行状态”（加入到工作队列）。此时内核代码执行完毕，将控制权交还给用户态。通过正常的“CPU 时间片的调度”，用户进程得以处理数据。

**4.1.4 进程模型**

上面介绍的整个过程，基本就是 BIO（阻塞 IO）的基本原理了。用户进程都是独立的处理自己的业务，这其实是一种符合进程模型的处理方式。

#### **4.2 上下文切换的优化**

上面介绍的过程中，有两个地方会造成频繁的上下文切换，效率可能会很低。

1. 如果频繁的收到数据包，NIC 可能频繁发出中断请求（IRQ）。CPU 也许在用户态，也许在内核态，也许还在处理上一条数据的协议栈。但无论如何，CPU 都要尽快的响应中断。这么做实际上非常低效，造成了大量的上下文切换，也可能导致用户进程长时间无法获得数据。（即使是多核，每次协议栈都没有处理完，自然无法交给用户进程）

1. 每个 Packet 对应一个 socket，每个 socket 对应一个用户态的进程。这些用户态进程转为“可运行状态”，必然要引起进程间的上下文切换。

**4.2.1 网卡驱动的 NAPI 机制**

在 NIC 上，解决频繁 IRQ 的技术叫做 New API(NAPI) 。原理其实特别简单，把 Interrupt Handler 分为两部分。

1. 函数名为 napi_schedule，专门快速响应 IRQ，只记录必要信息，并在合适的时机发出软中断 softirq。

1. 函数名为 netrxaction，在另一个进程中执行，专门响应 napi_schedule 发出的软中断，批量的处理 RingBuffer 中的数据。

所以使用了 NAPI 的驱动，接收数据过程可以简化描述为：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

1. NIC 接收到数据，通过 DMA 方式写入内存(Ring Buffer 和 sk_buff)。

1. NIC 发出中断请求（IRQ），告诉内核有新的数据过来了。

1. driver 的 napi_schedule 函数响应 IRQ，并在合适的时机发出软中断（NET_RX_SOFTIRQ）

1. driver 的 net_rx_action 函数响应软中断，从 Ring Buffer 中批量拉取收到的数据。并处理协议栈，填充 Socket 并交给用户进程。

1. 系统切换为用户态，多个用户进程切换为“可运行状态”，按 CPU 时间片调度，处理数据内容。

一句话概括就是：等着收到一批数据，再一次批量的处理数据。

**4.2.2 单线程的 IO 多路复用**

内核优化“进程间上下文切换”的技术叫的“IO 多路复用”，思路和 NAPI 是很接近的。

每个 socket 不再阻塞读写它的进程，而是用一个专门的线程，批量的处理用户态数据，这样就减少了线程间的上下文切换。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

作为 IO 多路复用的一个实现，select 的原理也很简单。所有的 socket 统一保存执行 select 函数的（监视进程）进程 ID。任何一个 socket 接收了数据，都会唤醒“监视进程”。内核只要告诉“监视进程”，那些 socket 已经就绪，监视进程就可以批量处理了。

#### **4.3 IO 多路复用的进化**

**4.3.1 对比 epoll 与 select**

select，poll 和 epoll 都是“IO 多路复用”，那为什么还会有性能差距呢？篇幅限制，这里我们只简单对比 select 和 epoll 的基本原理差异。

对于内核，同时处理的 socket 可能有很多，监视进程也可能有多个。所以监视进程每次“批量处理数据”，都需要告诉内核它“关心的 socket”。内核在唤醒监视进程时，就可以把“关心的 socket”中，就绪的 socket 传给监视进程。

换句话说，在执行系统调用 select 或 epoll_create 时，入参是“关心的 socket”，出参是“就绪的 socket”。

而 select 与 epoll 的区别在于：

- **select （一次O(n)查找）**

1. 每次传给内核一个用户空间分配的 fd_set 用于表示“关心的 socket”。其结构（相当于 bitset）限制了只能保存1024个 socket。

1. 每次 socket 状态变化，内核利用 fd_set 查询O(1)，就能知道监视进程是否关心这个 socket。

1. 内核是复用了 fd_set 作为出参，返还给监视进程（所以每次 select 入参需要重置）。

   然而监视进程必须遍历一遍 socket 数组O(n)，才知道哪些 socket 就绪了。

- **epoll （全是O(1)查找）**

1. 每次传给内核一个实例句柄。这个句柄是在内核分配的红黑树 rbr+双向链表 rdllist。只要句柄不变，内核就能复用上次计算的结果。

1. 每次 socket 状态变化，内核就可以快速从 rbr 查询O(1)，监视进程是否关心这个 socket。同时修改 rdllist，所以 rdllist 实际上是“就绪的 socket”的一个缓存。

1. 内核复制 rdllist 的一部分或者全部（LT 和 ET），到专门的 epoll_event 作为出参。

   所以监视进程，可以直接一个个处理数据，无需再遍历确认。

**Select 示例代码**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**Epoll 示例代码**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

另外，epoll_create 底层实现，到底是不是红黑树，其实也不太重要（完全可以换成 hashtable）。重要的是 efd 是个指针，其数据结构完全可以对外透明的修改成任意其他数据结构。

**4.3.2 API 发布的时间线**

另外，我们再来看看网络 IO 中，各个 api 的发布时间线。就可以得到两个有意思的结论。

> 1983，socket 发布在 Unix(4.2 BSD)
>
> 1983，select 发布在 Unix(4.2 BSD)
>
> 1994，Linux的1.0，已经支持socket和select
>
> 1997，poll 发布在 Linux 2.1.23
>
> 2002，epoll发布在 Linux 2.5.44

1、socket 和 select 是同时发布的。这说明了，select 不是用来代替传统 IO 的。这是两种不同的用法(或模型)，适用于不同的场景。

2、select、poll 和 epoll，这三个“IO 多路复用 API”是相继发布的。这说明了，它们是 IO 多路复用的3个进化版本。因为 API 设计缺陷，无法在不改变 API 的前提下优化内部逻辑。所以用 poll 替代 select，再用 epoll 替代 poll。

#### **4.4 总结**

我们花了三个章节，阐述 Epoll 背后的原理，现在用三句话总结一下。

1. 基于数据收发的基本原理，系统利用阻塞提高了 CPU 利用率。

1. 为了优化上线文切换，设计了“IO 多路复用”（和 NAPI）。

1. 为了优化“内核与监视进程的交互”，设计了三个版本的 API(select,poll,epoll)。

### **5 Diss 环节**

讲完“Epoll 背后的原理”，已经可以回答最初的几个问题。这已经是一个完整的文章，很多人劝我删掉下面的 diss 环节。

我的观点是：学习就是个研究+理解的过程。上面是研究，下面再讲一下我的个人“理解”，欢迎指正。

#### **5.1 关于 IO 模型的分类**

关于阻塞，非阻塞，同步，异步的分类，这么分自然有其道理。但是在操作系统的角度来看\*\*“这样分类，容易产生误解，并不好”\*\*。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**5.1.1 阻塞和非阻塞**

Linux 下所有的 IO 模型都是阻塞的，这是收发数据的基本原理导致的。阻塞用户线程是一种高效的方式。

你当然可以写一个程序，socket 设置成非阻塞模式，在不使用监视器的情况下，依靠死循环完成一次 IO 操作。但是这样做的效率实在是太低了，完全没有实际意义。

换句话说，阻塞不是问题，运行才是问题，运行才会消耗 CPU。IO 多路复用不是减少了阻塞，是减少了运行。上下文切换才是问题，IO 多路复用，通过减少运行的进程，有效的减少了上下文切换。

**5.1.2 同步和异步**

Linux 下所有的 IO 模型都是同步的。BIO 是同步的，select 同步的，poll 同步的，epoll 还是同步的。

Java 提供的 AIO，也许可以称作“异步”的。但是 JVM 是运行在用户态的，Linux 没有提供任何的异步支持。因此 JVM 提供的异步支持，和你自己封装成“异步”的框架是没有本质区别的（你完全可以使用 BIO 封装成异步框架）。

所谓的“同步“和”异步”只是两种事件分发器（event dispatcher）或者说是两个设计模式（Reactor 和 Proactor）。都是运行在用户态的，两个设计模式能有多少性能差异呢？

- Reactor 对应 java 的 NIO，也就是 Channel，Buffer 和 Selector 构成的核心的 API。

- Proactor对应 java 的 AIO，也就是 Async 组件和 Future 或 Callback 构成的核心的 API。

**5.1.3 我的分类**

我认为 IO 模型只分两类：

1. 更加符合程序员理解和使用的，进程模型；

1. 更加符合操作系统处理逻辑的，IO 多路复用模型。

对于“IO多路复用”的事件分发，又分为两类：Reactor 和 Proactor。

#### **5.2 关于 mmap**

epoll 到底用没用到 mmap？

**答案：没有！**

这是个以讹传讹的谣言。其实很容易证明的，用 epoll 写个 demo。strace 一下就清楚了。

- END -

______________________________________________________________________

**看完一键三连****在看****，**转发****，点赞\*\*\*\*

**是对文章最大的赞赏，极客重生感谢你**\*\*!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)\*\*

推荐阅读

\[

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

深入理解Linux异步I/O框架 io_uring

\](http://mp.weixin.qq.com/s?\_\_biz=MzkyMTIzMTkzNA==&mid=2247562787&idx=1&sn=471a0956249ca789afad774978522717&chksm=c1850172f6f28864474f9832bfc61f723b5f54e174417d570a6b1e3f9f04bda7b539662c0bed&scene=21#wechat_redirect)

\[

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

五个半小时

\](http://mp.weixin.qq.com/s?\_\_biz=MzkyMTIzMTkzNA==&mid=2247563566&idx=1&sn=26156d79dffb3f0f10b6a26931f993cc&chksm=c1850e7ff6f28769b6ff3358366e917d3d54fc0f0563131422da4bed201768c958262b5d5a99&scene=21#wechat_redirect)

\[

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

服务器性能优化之网络性能优化

\](http://mp.weixin.qq.com/s?\_\_biz=MzkyMTIzMTkzNA==&mid=2247561718&idx=1&sn=f93ad69bff3ab80665e4b9d67265e6bd&chksm=c18506a7f6f28fb12341c3e439f998d09c4b1d93f8bf59af6b1c6f4427cea0c48b51244a3e53&scene=21#wechat_redirect)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

求点赞，在看，分享三连!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

阅读 1896

​

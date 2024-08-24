# 

崔皓 高可用架构

 _2021年11月23日 09:03_

“

Tomcat 作为应用最广泛的 Web 容器被各大厂商所使用，从体系结构上看 Tomcat 分为连接器和容器两个部分。其中连接器负责 IO 请求转换、网络请求解析以及请求适配等工作。

  

![图片](https://mmbiz.qpic.cn/mmbiz_png/MOwlO0INfQpbVjOicSFnAjrEdBrbLnhUFFk6Lb6tNervPicGROJmia5sxEkichjHwrI3ejSEZgKNXkPevmadCEA19g/640?wx_fmt=png&wxfrom=13&tp=wxpic)

图片来自 Pexels  

  

为了深入了解其工作原理，今天让我们走进 Tomcat 连接器原理与实现。

  

Tomcat 连接器结构与原理

  

在开始介绍 Tomcat 连接器之前，先来回顾一下连接器的结构和工作原理。

![图片](https://mmbiz.qpic.cn/mmbiz_png/MOwlO0INfQoxZRiaFxexZmia9eDQALvbJNj9fpT4KoT78cic05uWV6yNg1ux12zuPGCg4tF3CvUnEWypo6sUdqPJQ/640?wx_fmt=png&wxfrom=13&tp=wxpic)

图 1：连接器结构图

  

如图 1 所示，Tomcat 连接器接收来自浏览器的请求，通过 ProtocolHandler 中的 EndPoint 和 Processor 组件完成对 IO 模型额解析和处理，并且通过 Adapter 适配器将请求转化为 ServletRequest 交给容器处理。

  

连接器对于 Servlet 容器屏蔽了协议及 IO 模型，让容器专注于 ServletRequest 的处理工作。无论是 HTTP 还是 AJP 请求，在容器最终都会获取 ServletRequest 对象。

  

因此 Tomcat 连接器的主要功能是：

- 监听网络端口。
    
- 接收网络请求的字节流信息。
    
- 根据协议（HTTP/AJP）解析字节流，生成 Tomcat Request 对象。
    
- 将 Tomcat Request 对象转成 ServletRequest 对象发送给容器。
    
- 获取容器返回的 ServletResponse 对象，并且将其转化为 Tomcat Response 对象。
    
- 将 Tomcat Response 转成网络字节流，返回给网络请求方。
    

  

为了实现上述功能，Tomcat 连接器需要下面几个组件的支持：

- **Endpoint：**监听通信接口，用来接收和发送网络请求，它对传输层进行了抽象。
    
- **Acceptor：**当 Endpoint 接收到 Socket 请求以后，由 Acceptor 对其进行监听。其中 SocketProcessor 用于处理接收到的 Socket 请求，它实现 Runnable 接口，在 run 方法里调用 Processor 进行处理。
    
- **Processor：**接收来自 Endpoint 的 Socket 请求，并将其解析成 Tomcat Request 对象，并交给 Adapter 进行后续的转换处理。
    
- **Adapter：**针对不同容器的 Servlet 进行请求/响应适配。将客户端发过来的 Tomcat Request 对象通过 ProtocolHandler 接口解析生成 ServletRequest，并将其发送给容器中的 Servlet。
    

  

这里我们将 Tomcat 连接器的基本功能、构成和组件给大家做了简单介绍，其具体架构的介绍在另外一篇的 [Tomcat 架构](http://mp.weixin.qq.com/s?__biz=MjM5ODI5Njc2MA==&mid=2655854098&idx=1&sn=1afa2aa7a372b379d45290e5b17c3081&chksm=bd7553c58a02dad34936ed4dc3070bc66cb3817b62f19acf50f727bd5f24e51f6e540d65b3cb&scene=21#wechat_redirect)文章中有详细描述。

  

回到本篇的主题，针对 Tomcat 连接器处理 IO 请求并且进行转换传递的能力。

  

会陆续给大家介绍几个 IO 处理组件：NioEndpiont、Nio2Endpoint 以及 Tomcat 的连接池是如何配合这几个组件完成 IO 处理操作的。

  

IO 模型与多路复用

  

Tomcat 连接器 IO 处理的组件包括三个，NioEndpiont、Nio2Endpoint 和 AprEndpoint。

  

我们会从 NioEndpoint 开始介绍，NioEndpoint 组件实际上是实现了 IO 多路复用模型。

  

所以在介绍 NioEndpoint 之前需要对 IO 模型与多路复用模型进行讲解，从而让大家对 NioEndpoint 的工作原理能够有深刻的认识。 

  

就操作系统而言它的核心是内核，是独立于普通的应用程序，内核空间可以访问受保护的内存空间，也有访问底层硬件设备。

  

为了保证用户进程不能直接操作内核），保证内核的安全，操作系统将空间划分为两部分，一部分为内核空间，一部分为用户空间。同时用户空间需要通过内核访问硬件设备。

  

当网络数据请求达到时，用户进程会先等待内核将数据从网卡拷贝到内核空间，然后再将数据从内核空间拷贝到用户空间，IO 模型就是对这一过程的描述。

  

如果有多个网络请求进来，那么就对应多个用户进行运行，并且处理这些请求。

  

为了控制进程的执行，内核必须有能力挂起正在 CPU 上运行的进程，并恢复以前挂起的某个进程的执行。

  

换句话说内核会让哪些具备运行条件的进程运行，让哪些不满足条件的进程挂起，当条件满足的时候再恢复运行，内核的这种行为被称为进程切换。

  

但是恰恰是这种进程切换是非常耗费资源的，因为内核需要保存进程的状态和运行上下文的信息。

  

对于正在执行的进程而言，也会因为某些事件的发生导致工作暂停，例如请求系统资源失败、等待某种操作的完成、新数据尚未到达或无新工作做等，则由系统自动执行阻塞原语（Block），使自己由运行状态变为阻塞状态。

  

阻塞状态是因为进程自身行为决定，当进程在运行状态时会占用 CPU 资源，但是进入阻赛状态后就不会消耗 CPU 资源了。

  

有了上面知识铺垫以后，我们来看看 IO 模型的几种实现方式：

  

**①同步阻塞 IO**

  

用户进程发起 read 调用后就进程阻塞了，因为此时需要等待网卡的数据才能 read。

  

于是用户进程让出 CPU，当网卡数据到来了并且数据从网卡拷贝到内核空间，接着将数据拷贝到用户空间，此时进行进程切换将用户进程唤醒，继续 read 操作。

  

**②同步非阻塞 IO**

  

用户进程会不断的发起 read 调用，如果数据没有到达内核空间，那么每次 read 调用都会返回失败，不过此时的用户进程并没有阻赛，仅仅需要不断询问内核空间数据是否达到。

  

当数据到达内核空间之后，但是在数据从内核空间拷贝到用户空间这段时间里进程会进入阻塞状态，等数据到了用户空间才会把进程叫醒。

  

**③IO 多路复用**

  

有 select、poll、epoll 三种实现方式，这里以 select 为例给大家讲解。

  

这里将用户进程的 read 操作分成两步了，用户进程先向内核发起 select 调用，询问数据数据是否准备好了。

  

当内核把数据准备好了，用户进程再发起 read 调用。但是在等待数据从内核空间拷贝到用户空间这段时间里，进程还是阻塞的。

  

换句话说在发起 select 调用和 read 调用这段时间内进程并没有阻赛，还可以执行其他操作，与同步非阻赛 IO 相比就省去了不断发起 read 调用的过程，提升了效率。

  

这里的多路复用是指一次 select 调用可以向内核查多个数据通道（Channel）的状态。

  

由于 NioEndpoint 组件使用的多路复用模型，这里对该模型进行详细的介绍，争取让大家从原理上能够理解这个模型的工作原理。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图 2：多路复用的 read 和 select 过程

  

如图 2 所示，用户空间的用户进程为了访问对应的文件会建立一个 fd 的列表，这里的 fd 是文件描述符（File descriptor）的缩写，其用来表述文件引用。

  

当使用或者打开一个文件时，会返回一个文件描述符，说白了就是操作文件的许可证。

  

从图中可以看到用户进程和内核进程中都维护了相同的 fd 列表，这就是需要进行 read 调用的文件列表。

  

从图中可以看出每个 fd 在用户空间和内核空间都是一一对应的，这里我们用虚线将文件描述符 fd 建立了一个通道（channel）。

  

接着来看看 select 和 read 的过程：

- 用户进程根据需要操作文件的 fd 列表向内核发起 select 调用。
    
- 内核空间接到 select 请求以后，会针对这个 fd 列表进行多路的监听，看看哪些文件的从网卡拷贝到内核空间了。
    
    假设这里标有绿色的 fd 文件从网卡拷贝到了内核空间，此时内核会通过 select 返回的方式通知用户进程有文件准备好了。
    
- 用户进程接收到内核的 select 返回以后，会开始执行 read 调用读取对应的文件内容。
    

  

NioEndpoint 组件

  

有了上面对 IO 多路复用原理的介绍，这里对 NioEndpoint 组件接收网络请求并处理的过程进行介绍。

  

过程中会涉及到：LimitLatch、Acceptor、Poller、SocketProcessor 和 Executor 等 5 个组件。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图 3：NioEndpoint 运行原理

  

如图 3 所示，NioEndpoint 经过 8 个步骤来出来网络请求：

  

**①**当服务器接收到网络请求，最先会到达网卡，此时网卡上的数据会被拷贝到内存空间上。

  

**②**内存空间会创建接收队列，用来存放数据包，将数据包传递给 NioEndpoint 组件处理。

  

LimitLatch 是 NioEndpoint 的连接控制器，用来控制最大连接数，NIO 模式下默认是 10000，达到这个阈值后，连接请求会被拒绝。

  

**③**请求通过 LimitLatch 到达 Acceptor，Acceptor 跑在一个单独的线程里，通过在死循环里调用 accept 方法来接收新连接。

  

**④**一旦有新的连接请求到来，accept 方法返回一个 Channel 对象，接着把 Channel 对象交给 Poller 去处理。这里的 Channel 对象就是用来监听数据包是否可以读取的。

  

**⑤**Poller 在内部维护 Channel 数组，并且会不断检测 Channel 的数据就绪状态。

  

一旦就绪就说明 Channel中的数据包可读，于是生成一个 SocketProcessor 任务对象交给 Executor 去处理。

  

**⑥**Executor 线程池，负责运行 SocketProcessor 任务类，SocketProcessor 的 run 方法会调用 Http11Processor 来读取和解析请求数据。

  

**⑦**Http11Processor 是应用层协议的封装，它会调用对应的容器并且获得响应，再把响应通过 Channel 返回给发送队列。这里专注于连接器的工作，就没有花容器的部分。

  

**⑧**发送队列在接收到响应的数据包以后会将响应数据通过网卡返回给网络请求方。

  

说完了 NioEndppoint 的执行流程，这里需要顺便提一下 AprEndpoint 组件。

  

APR（Apache Portable Runtime Libraries）是 Apache 可移植运行时库，它是用 C 语言实现的，它的工作也是处理包括文件和网络 IO请求。

  

为什么要在这里提起是因为 AprEndpoint 与 NioEndpoint 一样居于非阻塞  IO 的多路复用机制实现的。

  

所不同的是，AprEndpoint 是通过 JNI 调用 APR 本地库而实现非阻塞 I/O 的。APR 的实现是为了处理一些特殊场景，比如一些需要频繁与操作系统交互的场景。

  

例如 Web 应用使用了 TLS 进行加密传输，由于在传输过程中存在多次网络交互，这种情况 C 语言程序与操作系统交互操作会提高执行效率，这也是 APR 的强项。

  

另外补充一点，上面提到的 JNI（Java Native Interface） 是 JDK 提供的一个编程接口，它允许 Java 程序调用其他语言编写的程序或者代码库，其实 JDK 本身的实现也大量用到 JNI 技术来调用本地 C 程序库。

  

说白了就是通过 JNI 去调用 C，通过 C 与操作系统进行交互操作，目的是提高交互执行的效率，用在与操作系统交互频繁的场景。

  

由于 AprEndpoint 组件的原理和 NioEndpiont 相似，我们这里就将其不同点和特点做扩展说明，不去展开描述了。

  

Nio2Endpoint 组件

  

上面介绍了 NioEndpoint 的实现原理和处理流程，Nio2Endpoint 与 NioEndpoint 最大的区别是前者是一步执行请求处理而后者是同步执行请求处理。

  

异步的特点就是用户空间的应用程序不需要自己去触发数据从内核空间到用户空间的拷贝。

  

这是因为应用程序是不能直接访问内核空间的，因此数据拷贝工作是由内核来完成。

  

NioEndpoint 是用户空间的进程通过 select 方式监听 channel 中的数据/文件是否就绪，如果就绪通过 read 调用把内核的数据拷贝到用户空间。

  

而 Nio2Endpoint 是内核当数据/文件就绪的时候将数据/文件主动拷贝到用户空间。

  

NioEndpoint 组件的工作模式下，数据从内核空间拷贝到用户空间这段时间，进程仍旧还是阻塞的，必须等到数据拷贝完毕才能继续执行。而 Nio2Endpoint 组件的工作模式下，数据拷贝的过程中进程是不会被阻塞。

  

以网络数据读取为例，使用 Nio2Endpoint 的异步模式的时候，用户进程在通过 read 读取网络数据时会告诉内核两件事情：

- 第一，数据就绪以后将其拷贝到哪个 Buffer。
    
- 第二，调用用户进程中的哪个回调函数去处理数据。
    

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图 4：Nio2Endpiont 异步读取数据

  

如图 4 所示，当内核接到 read 指令后，依旧会等待网卡数据到达，数据到了后，产生硬件中断，内核在中断程序里把数据从网卡拷贝到内核空间。

  

接着做 TCP/IP 协议层面的数据解包和重组，再把数据拷贝到应用程序指定的 Buffer，最后调用用户进程指定的回调函数。

  

在了解了 Nio2Endpoint 读取数据的过程以后，让我们进一步分解起内部的执行流程，和分析 NioEndpoint 的流程一样，借助图 5 来对其进行分析。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图 5：Nio2Endpint 工作流程

  

如图 5 所示，把网卡、内核空间以及 Nio2Endpoint 组件放到一起讨论，看看进程调用和数据流转是如何进行的。

  

**①**当服务器接收到网络请求，最先会到达网卡，此时网卡上的数据会被拷贝到内存空间上。

  

**②**内存空间会创建接收队列，用来存放数据包，将数据包传递给 Nio2Endpoint 组件处理。

  

LimitLatch 是 Nio2Endpoint 的连接控制器，用来控制最大连接数，NIO 模式下默认是 10000，达到这个阈值后，连接请求会被拒绝。

  

**③**请求通过 LimitLatch 到达 Nio2Acceptor，Nio2Acceptor 是一个进程组并且扩展了 Acceptor，用异步 I/O 的方式来接收连接。

  

**④**Nio2Acceptor 接收新的连接后，得到一个 AsynchronousSocketChannel，并且将其封装成 Nio2SocketWrapper，同时对应创建一个 SocketProcessor 任务类交给线程池处理。

  

**⑤**Executor 线程池，负责运行 SocketProcessor 任务类，SocketProcessor 的 run 方法会调用 Http11Processor 来读取和解析请求数据。

  

**⑥**Http11Processor 会通过 Nio2SocketWrapper 读取和解析请求数据，请求经过容器处理后，再把响应通过 Nio2SocketWrapper 写出。

  

**⑦**Http11Processor 通过调用 Nio2SocketWrapper 的 read 方法发出第一次读请求，同时告知数据拷贝的buffer和回调的类 readCompletionHandler。

  

由于数据没有就绪，因此 Http11Processor 把 Nio2SocketWrapper 标记为数据不完整。

  

与之对应的 SocketProcessor 线程被回收，Http11Processor 并没有阻塞等待数据。

  

同时 Http11Processor 会维护 Nio2SocketWrapper 的列表，其目的是维护了连接的状态，方便数据就绪以后执行后续回调操作。

  

**⑧**当数据就绪以后，内核已经把数据拷贝到 Http11Processor 指定的 Buffer 里，同时调用 readCompletionHandler。

  

在回调处理中创建一个新的 SocketProcessor 来处理连接，由于Http11Processor 维护了 Nio2SocketWraper 的列表。

  

此时即便是新的 SocketProcessor 任务类也可以持发起read命令的Nio2SocketWrapper。

  

此时 Http11Processor 可以通过这个 Nio2SocketWrapper 从缓冲区中读取数据并且进行后续处理。

  

**⑨**Nio2SocketWrapper 将处理完毕的数据传给发送队列。

  

**⑩**发送队列在接收到响应的数据包以后会将响应数据通过网卡返回给网络请求方。

  

总结

  

本文从 Tomcat 连接器的结构和原理切入，让大家知道 Tomcat 连接器通过 Endpoint 接收 IO 请求，通过 Processor 处理请求内容，使用 Adapter 将 Tomcat Reqeust 转换成 Servlet 容器能够接受的 ServletRequest 请求，并且交给容器处理。

  

然后将重点转移到连接器的核心 Endpoint 组件，看看它们是如何处理来自网络的 IO 请求。

  

在介绍具体实现 IO 组件之前，先通过 IO 模型的演进和多路复用模型的铺垫，让大家了解底层的原理。

  

然后再分别介绍 NioEndpoint 组件和 Nio2Endpoint 组件实现网络请求监听和处理的原理和流程。

  

**参考阅读：**

  

- [爱奇艺基于SpringCloud的韧性能力建设](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653557768&idx=1&sn=c19df84860adb962015d1f74e44ad798&chksm=81398590b64e0c86d809423fa290fd22a7f61949d59bda7b4eecd3cb3badf188ad3e80b8057c&scene=21#wechat_redirect)
    
- [vivo统一告警平台建设与实践](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653557750&idx=1&sn=d27b9b702a3bbdfa58d686ea30fbe14f&chksm=8139826eb64e0b7816434d4f04a6e5756cb53a64edb030bc31284e42d7d6f04d17ec37d3486f&scene=21#wechat_redirect)
    
- [美图Android编译速度优化实践指南](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653557685&idx=1&sn=f0be7227e57943ee3b4bffd5388396f5&chksm=8139822db64e0b3b19b1bfee97d9ca5c756e2bf9c416fa93c228727eb4987229af72955436af&scene=21#wechat_redirect)
    
- [基于etcd实现大规模服务治理应用实战](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653557639&idx=1&sn=7655a93ad702670b5ff685e6aa7b7468&chksm=8139821fb64e0b09d47272952230a73325af8a9a5dbb14f94cd8c4d2c2ba6e2e5cc411555885&scene=21#wechat_redirect)
    
- [深入剖析Redis客户端Jedis的特性和原理](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653557624&idx=1&sn=11274fdc8454ab6ebc4ba1d562148a6c&chksm=813982e0b64e0bf671d7b3a32219e5f2a905372018d1add48f78ac0ff63b9e033a7c58b1782c&scene=21#wechat_redirect)
    

  

技术原创及架构实践文章，欢迎通过公众号菜单「联系我们」进行投稿。  

  

**高可用架构**

**改变互联网的构建方式**

  

![](http://mmbiz.qpic.cn/mmbiz_png/8XkvNnTiapONl06YmHad4csRU93kcbJ76JIWzEAmOSVooibFHHkzfWzzkc7dpU4H06Wp9F6Z687vIghdawxvl47A/300?wx_fmt=png&wxfrom=19)

**高可用架构**

高可用架构公众号。

439篇原创内容

公众号

阅读 2294

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/8XkvNnTiapONl06YmHad4csRU93kcbJ76JIWzEAmOSVooibFHHkzfWzzkc7dpU4H06Wp9F6Z687vIghdawxvl47A/300?wx_fmt=png&wxfrom=18)

高可用架构

923

写留言

写留言

**留言**

暂无留言
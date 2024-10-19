
原创 Chaobowx 软硬件融合 _2024年06月04日 21:00_

编者按

随着大模型的广泛流行，GPU集群计算的规模越来越大（单芯片算力提升有限，只能通过扩规模的方式来提升整体算力），千卡、万卡已经成为主流，十万卡、百万卡也都在未来3-5年的规划中。

集群计算的网络可以分为两类：南北向流量，也就是俗称的外网流量；东西向流量，也就是俗称的内网流量。集群计算的网络连接数 S = N x (N-1)（N为节点数）。这也是为什么现在数据中心中东西向流量占比超过90%的原因所在。再加上系统规模扩大导致的南北向网络流量的增加，这使得数据中心所需的网络带宽呈指数级增长趋势。

2019年，NVIDIA收购Mellanox，凭借着在InfiniBand和ROCEv2领域的领先优势，NVIDIA成为了高性能网络的霸主。各大竞争对手不甘示弱，特别是AWS、阿里云等云计算厂家，都陆续推出了自己的高性能网络协议和对应的产品，行业呈现百家争鸣之象。高性能网络，兵家必争之地；未来谁主沉浮，我们拭目以待。

本篇文章，从软硬件融合视角，对高性能网络相关技术进行介绍并进行分析，让大家一文看透高性能网络。

# **1 高性能网络综述**

## **1.1 网络性能参数**

![[Pasted image 20241019191628.png]]

网络性能主要关心三个参数，带宽、吞吐量和延迟：

- 带宽：指特定时间段内可以传输的数据量。高带宽并不一定保证最佳的网络性能。例如，网络吞吐量受到数据包丢失、抖动或延迟的影响，可能会遇到延迟问题。

- 吞吐量：指在特定时间段内能够发送和接收的数据量。网络上数据的平均吞吐量使用户能够深入了解成功到达正确目的地的数据包数量。为了高性能，数据包必须能够到达正确目的地。如果在传输过程中丢失了太多数据包，则网络性能可能不足。

- 延迟：数据包在发送后到达目的地所用的时间。我们将网络延迟测量为往返行程。延迟的结果通常是不稳定和滞后的服务。例如视频会议等，对延迟非常敏感。

除此之外，还有一个非常重要的指标，PPS（Packet Per Second，每秒传输数据包数）。许多网络设备，在大包的时候，可以做到线速，但在小包（64字节）的情况下，其性能降低的非常严重，主要就是PPS能力不足引起的。所以，理想的情况是，在最小包的情况下，仍然能够达到线速。

高性能网络，就是要在低延迟（越低越好）、低抖动的情况下，（不管大包小包，任意网络节点，都要）实现最高的吞吐量（无限接近于网络带宽）。当然了，这些参数是相互影响的，实际的系统是在这些参数之间取得的平衡。

## **1.2 复杂的网络分层**

!\[\[Pasted image 20240920122820.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

OSI定义了七层网络协议，实际工程应用的TCP/IP网络协议一般为五层：

- 从下到上依次为：物理层、数据链路层、网络层、传输层和应用层。

- 通常情况下，物理层、数据链层在硬件实现，而网络层、传输层和应用层在软件实现。

- TCP/IP协议族是计算机网络中使用的一组通信协议。包括的基础协议有传输控制协议(TCP) 、互联网协议(IP)，以及用户数据报协议(UDP)。

- 以太网（Ethernet）是为了实现局域网通信而设计的一种技术，它规定了包括物理层的连线、电子信号和介质访问层协议的内容。以太网是目前应用最普遍的网络技术。

!\[\[Pasted image 20240920122826.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

数据中心网络要更加复杂，会分为Overlay网络和Underlay网络。如果按照功能逻辑把网络分层，云计算数据中心网络可以分成三层：

- 第一层，物理的基础网络连接，也就是我们通常所理解的Underlay底层承载网；

- 第二层，基于基础的物理网络构建的虚拟网络，也就是我们通常理解的基于隧道的Overlay网络；

- 第三层，各种用户可见的应用级的网络服务，比如接入网关、负载均衡等。也可以是其他需要用到网络的普通应用。

物理的数据中心是一个局域网，通过Overlay网络分割成数以万计的虚拟私有网；域间隔离后，再需要一定的网络安全机制实现高性能的跨域访问。
!\[\[Pasted image 20240920122837.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

依据网络处理逻辑，网络可以分为三部分：

- 第一阶段，业务数据的封包/解包。Tx向的时候，把业务的数据按照既定的网络格式进行封包；Rx向，则是把收到的数据包进行解包。

- 第二阶段，网络包的处理。比如Overlay网络，需要将数据包再次进行封装；比如防火墙，要对网络包进行鉴别，是否允许传输（输入输出双向）；比如网络包加解密。

- 第三阶段，网络包的传输（Tx/Rx）。也即是高性能网络关注的部分。

**1.3 为什么需要高性能网络**

为什么需要高性能网络？

- 原因一，网络的极端重要性。

1. 网络连接的重要性：网络连接所有节点，各类服务都通过网络链接，用户通过网络远程操作。没有网络，一切都是空的。

1. 网络的复杂性：业务系统，要么是单服务器级别的或者集群级别；网络系统基本上都是数据中心级别，在整个数据中心的规模上，构建各种复杂的网络业务逻辑，整个系统复杂度非常高，影响面也大。

1. 网络故障的严重性：计算服务器故障、存储服务器故障都是局部的故障，而网络故障则牵一发而动全身。任何一个微小的网络故障，都可能会引起整个数据中心不可用。网络故障一旦发生，必然是重大故障。

- 原因二，超大规模集群计算，东西向流量激增。数据中心复杂计算场景，系统持续解构，东西向流量激增，网络带宽要求迅速的从10G、25G升级到100G，甚至200G。

- 原因三，短距离传输，系统堆栈延迟凸显。相比城域网或者全球互联网络，数据中心网络传输距离很短，服务器系统堆栈的延迟凸显。

- 原因四，CPU性能瓶颈，网络处理延迟进一步加大。网络带宽快速增加，而CPU的性能已经瓶颈。网络堆栈处理的CPU资源消耗快速增加，并且延迟还进一步增大。

- 原因五，跨服务器调用延迟敏感。而软件服务之间的调用，要求跨服务器的远程调用能够低延迟，接近于本地调用的延迟。

## **1.4 网络拥塞控制**

!\[\[Pasted image 20240920122844.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

网络中如果存在太多的数据包，会导致包延迟，并且会因为超时而丢弃，从而降低传输性能，这称为拥塞。高性能网络，就是要充分利用网络容量，提供低延迟网络传输的同时，尽可能的避免网络拥塞。

当主机发送的数据包数量在网络的承载范围之内时，送达与发送的数据包成正比例增长。但随着负载接近网络承载极限，偶尔突发的网络流量会导致拥塞崩溃；当加载的数据包接近承载上限的时候，延迟会急剧上升，这就是网络拥塞。

拥塞控制（Congestion Control，CC）和流量控制有什么区别？拥塞控制，确保网络能够承载所有到达的流量，是一个全局问题；流量控制，则是确保快速的发送方不会持续地以超过接收方接收能力的速率传输，是端到端传输的问题。

根据效果从慢到快，处理拥塞的办法可以分为：

- 更高带宽的网络；

- 根据流量模式定制流量感知路由；

- 降低负载，如准入控制；

- 给源端传递反馈信息，要求源端抑制流量；

- 一切努力失败，网络丢弃无法传递的数据包。

拥塞控制算法的基本原则如下：

- 优化的带宽分配方法，既能充分利用所有的可用带宽却能避免拥塞；

- 带宽分配算法对所有传输是公平的，既保证大象流，也保证老鼠流；

- 拥塞控制算法能够快速收敛。

## **1.5 等价多路径路由ECMP**

CLOS架构是一种用于构建高性能、高可靠性数据中心网络的拓扑结构。受东西向流量激增等原因影响，数据中心为满足大通量网络流量的需求，网络拓扑通常采用CLOS结构。CLOS架构会使得主机和主机之间就存在多条网络路径，而“多路径”是实现高性能和高可靠网络的基础。

如何利用数据中心网络拓扑、路径资源、带宽资源等，更好的实现网络流量的负载均衡，将数据流分布到不同路径上进行数据传输，避免拥塞，提高数据中心内的资源利用率，就变得越来越重要。

ECMP（Equal-Cost Multi-Path routing，等价多路径路由）是一个逐跳的基于流的负载均衡策略，当路由器发现同一目的地址出现多个最优路径时，会更新路由表，为此目的地址添加多条规则，对应于多个下一跳。可同时利用这些路径转发数据，增加带宽。

ECMP常见的路径选择策略有多种方法：

- 哈希，例如根据源IP地址的哈希为流选择路径。

- 轮询，各个流在多条路径之间轮询传输。

- 基于路径权重，根据路径的权重分配流，权重大的路径分配的流数量更多。

需要注意的是，在数据中心这种突发性流量多，大象流与老鼠流并存的环境中，需要慎重考虑选择的负载均衡策略，ECMP虽然简单易部署，但也存在较多问题需要注意。

# **2 TCP/IP高性能网络**

TCP/IP协议栈主要指Ethernet+TCP/IP+Socket的整个系统栈。

## **2.1 TCP/IP协议栈硬件卸载**

**第一类，TCP的部分特征卸载**

- TSO，TCP Segmentation Offload，利用网卡硬件能力对TCP数据包分段。

- UFO，UDP Fragmentation Offload ，支持UDP发大包，硬件会进行分片。

- LRO，Large Receive Offload，网卡硬件将接收到的多个TCP数据聚合成一个大的数据包。

- RSS，Receive Side Scaling，将网络流分成不同的队列，再分别将这些队列分配到多个CPU核处理。

- CRC 卸载，由硬件完成CRC计算和封包。

**第二类，TCP/IP协议栈卸载（TCP/IP Offload Engine，TOE）**
!\[\[Pasted image 20240920122854.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

传统软件网络栈的处理开销很大，把整个TCP/IP协议栈卸载到硬件，可以提升网络处理的性能，显著降低CPU代价。

Linux 标准内核不支持 TOE网卡，原因主要是：

- 安全性。TOE硬件实现，与OS中的经过良好测试的软件TCP/IP栈相比，硬件的安全风险很大。

- 硬件约束。因为网络连接在TOE芯片上进行缓冲和处理，与软件可用的大量CPU和内存相比，资源匮乏更容易发生。

- 复杂性。TOE打破了kernel始终可以访问所有资源的假设，还需要对网络堆栈进行非常大的更改，即便如此，QoS和包过滤等功能也可能无法工作。

- 定制性。每个Vendor都是定制的TOE，这意味着必须重写更多代码来处理各种TOE实现，代价是方案的复杂性和可能的安全风险。此外，TOE固件闭源，软件人员无法修改。

- 过时。TOE NIC使用寿命有限：CPU性能会迅速赶上并超过TOE性能；软件系统栈的迭代会显著快于硬件协议栈。

因此，受上述原因影响，TOE并没有大规模使用起来。

## **2.2 应用层TCP/IP协议栈Onload**

!\[\[Pasted image 20240920122901.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Onload是Solarflare（Solarflare已被Xilinx收购，Xilinx又被AMD收购）加速网络中间件。主要特征包括：

- 基于IP的TCP/UDP实现，动态链接到用户模式应用程序的地址空间，应用程序可以直接从网络收发数据，而无需OS参与。

- Onload实现的内核绕过（Kernel Bypass）可避免系统调用、上下文切换和中断等破坏性事件，从而提高处理器执行应用程序代码的效率。

- Onload库在运行时使用标准Socket API与应用程序动态链接，这意味着不需要对应用程序进行任何修改。

- Onload通过减少CPU开销、改善延迟和带宽，以及提高应用的可扩展性，来显著降低网络成本。

Onload核心技术可以总结为：硬件虚拟化SR-IOV，内核bypass，以及应用SocketAPI。

## **2.3 传输协议优化：从TCP到QUIC**

!\[\[Pasted image 20240920122907.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

QUIC（Quick UDP Internet Connections，快速UDP互联网连接）是一种新的互联网传输协议，对L4、L5（安全）、L7进行部分或全部的优化和替代。

QUIC是一个应用自包含的协议，允许创新。这在现有协议中不可能实现，受遗留的客户端和系统中间件的阻碍。

与TCP+TLS+HTTP2相比，QUIC的主要优点包括:

- 连接建立的延迟。简单地说，QUIC握手在发送有效载荷之前需要0次往返，而TCP+TLS则需要1-3次往返。

- 改进的拥塞控制。QUIC主要实现并优化了TCP的慢启动、拥塞避免、快重传、快恢复，可以根据情况选择不同的拥塞控制算法；避免TCP重传歧义问题；允许精确的往返时间计算；等等。

- 无前端阻塞的多路复用。TCP存在行首阻塞问题，QUIC为多路复用操作设计，没有损失的流可以继续推进。

- 前向纠错。为了在不等待重传的情况下恢复丢失的数据包，QUIC可以用一个FEC数据包补充一组数据包。

- 连接迁移。TCP连接由四元组标识，QUIC连接由64位连接ID标识。如果客户端更改了IP地址或端口，则TCP连接不再有效；而QUIC可以使用旧的连接ID，而不会中断任何正在进行的请求。

# **3 高性能网络协议栈综述，以IB为例**

## **3.1 IB网络协议简介**

!\[\[Pasted image 20240920122913.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

InfiniBand (IB)是一种用于高性能计算的网络通信标准，具有极高的吞吐量和极低的延迟。

IB用于计算机之间和计算机内部的数据互连，还可以用作服务器和存储系统之间的直接或交换互连，以及存储系统之间的互连。

IB的设计目标：综合考虑计算、网络和存储技术，设计一个可扩展的、高性能的通信和I/O体系结构。

IB的主要优点：

- 高性能，超算TOP500中一半左右采用IB；

- 低延迟，IB端到端测量延迟为1µs；

- 高效率，IB原生支持RDMA等协议，帮助客户提高工作负载处理效率。

!\[\[Pasted image 20240920122920.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

IB也是五层网络分层：物理层、链路层、网络层、传输层和应用层。传统网络的L1&L2硬件实现，而L3/L4软件实现；而IB则是L1-L4均在硬件实现。IB传输层API即HCA网卡和CPU之间的软硬件接口。Socket API是传统TCP/IP网络的应用网络接口，而Verbs API是IB的应用网络接口。MPI是一种通过提供并行库来实现并行化的方法库，MPI可以基于OFA Verbs API，也可以基于TCP/IP Socket。

## **3.2 IB为什么能够高性能？**

!\[\[Pasted image 20240920122927.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
!\[\[Pasted image 20240920123030.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

传统网络协议栈的问题：

- 长路径，应用需要通过系统层连接到网卡；

- 多拷贝，很多拷贝是没有必要的；

- 弹性能力不足，协议栈经过内核，这样用户就很难去优化协议功能。

通过网络协议栈性能优化的四个方面，来总结IB全栈所采用的主要优化技术：

- 优化协议栈：简化、轻量、解耦的系统堆栈。

- 加速协议处理：协议硬件加速处理， L2-L4多个协议层卸载；

- 减少中间环节：如RDMA kernel Bypass，IB网络原生支持RDMA；

- 优化接口交互：如软硬件交互机制，Send/Rcv Queue Pair、Work/Completion Queues。

## **3.3 InfiniBand vs Ethernet**

!\[\[Pasted image 20240920123036.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Infiniband和Ethernet从多种方面的比较：

- 设计目标。Eth考虑多个系统如何流畅的信息交换，优先考虑兼容性与分布式；IB则是为了解决HPC场景数据传输瓶颈。

- 带宽。IB的网络速率通常快于Eth，主要原因是IB应用于HPC场景的服务器互联，而Eth更多的是面向终端设备互联。

- 延迟。Eth交换机处理的任务比较复杂，导致其延迟较大（>200ns）。而IB交换机处理非常简单，远快于Eth交换机（\<100ns）。采用RDMA+IB的网络收发延迟在600ns，而Eth+TCP/IP的收发延迟高达10us，相差10倍以上。

- 可靠性。丢包重传对性能的影响非常大。IB基于端到端的流控，保证报文全程不拥塞，时延抖动控制到最小。Eth没有基于调度的流控，极端情况会出现拥塞而导致丢包，使得数据转发性能大幅波动。

- 组网方式。Eth网络内节点的增删都要通知到每一个节点，当节点数量增加到一定数量时，会产生广播风暴。IB二层网络内有子网管理器来配置LocalID，然后统一计算转发路径信息，可以轻松部署一个规模几万台服务器的超大二层网络。

# **4 RDMA高性能网络**

## **4.1 RDMA综述**

!\[\[Pasted image 20240920123044.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

RDMA，是一种高带宽、低延迟、低CPU消耗的网络互联技术，克服了传统TCP/IP网络的许多问题。RDMA技术特点：

- 远程，数据在两个节点之间传输；

- 直接，不需要内核参与，传输的处理都卸载到NIC硬件完成；

- 内存，数据在两个节点的应用虚拟内存间传输，不需要额外的复制；

- 访问，访问操作有send/receive、read/write等。

RDMA的其他一些特征：

- RDMA和IB是原生一体；

- RoCEv1是基于标准Eth的RDMA；

- RoCEv2基于标准Eth、IP和UDP，支持标准L3路由器。

## 4.2 RDMA/RoCEv2系统栈分层

!\[\[Pasted image 20240920123049.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

RoCEv2是当前数据中心比较流行的RDMA技术，我们以RoCEv2为例，介绍RoCE的系统分层：

- 以太网层：标准以太网层，即网络五层协议物理层和数据链路层。

- 网络层IP：网络五层协议中的网络层。

- 传输层UDP：网络五层协议中的传输层（UDP而不是TCP）。

- IB传输层：负责数据包的分发、分割、通道复用和传输服务。

- RDMA数据引擎层（Data Engine Layer）：负责内存队列和RDMA硬件之间工作/完成请求的数据传输等。

- RDMA接口驱动层：负责RDMA硬件的配置管理、队列和内存管理，负责工作请求添加到工作队列中，负责完成请求的处理等。接口驱动层和数据引擎层共同组成RDMA软硬件接口。

- Verbs API层：接口驱动Verbs封装，管理连接状态、管理内存和队列访问、提交工作给RDMA硬件、从RDMA硬件获取工作和事件。

- ULP层：OFED ULP（上层协议）库，提供了各种软件协议的RDMA verbs支持，让上层应用可以无缝移植到RDMA平台。

- 应用层：RDMA原生的应用，基于RDMA verbs API开发；OFA还提供OFED协议栈，让已有应用可以无缝使用RDMA功能。

## **4.3 RDMA工作队列**

!\[\[Pasted image 20240920123054.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

RDMA并没有约束严格的软硬件接口，各家的实现可以有所不同，只需要支持RDMA的队列机制即可。

软件驱动和硬件设备的交互通常基于生产者消费者模型，这样能够实现软件和硬件交互的异步解耦。RDMA软硬件共享的队列数据结构称为工作队列（Work Queue）。

驱动负责把工作请求（Work Request）添加到工作队列，成为工作队列中的一项，称为工作队列项WQE。RDMA硬件设备会负责WQE在内存和硬件之间的传输，并且通过RDMA网络最终把WQE送到接收方的工作队列中去。

接收方RDMA硬件会反馈确认信息给到发送方RDMA硬件，发送方RDMA硬件会根据确认信息生成完成队列项CQE发送到内存的完成队列。

RDMA Queue类型有：发送队列、接收队列、完成队列以及队列对。发送队列和接收队列组成一组队列对。SRQ，共享接收队列。把一个RQ共享给所有关联的QP使用，这个公用的RQ就称为SRQ。

## **4.4 RDMA Verbs API**

!\[\[Pasted image 20240920123100.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

RDMA Verbs是提供给应用使用的RDMA功能和动作抽象。Verbs API则是Verbs具体实现，提供标准接口供应用调用。

RoCEv2中的Verbs操作主要有两类：

- Send/Recv。类似于Client/Server结构，发送操作和接收操作协作完成，在发送方连接之前，接收方必须处于侦听状态；发送方不知道接收方的虚拟内存位置，接收方也不知道发送方的虚拟内存地址。RDMA Send/Recv因为是直接对内存操作，需要提前注册用于传输的内存区域。

- Write/Read。与Client/Server架构不同，Write/Read是请求方处于主动，响应方处于被动。请求方执行Write/Read操作，响应方不需要做任何操作。为了能够操作响应方的内存，请求方需要提前获得响应方的地址和键值。

# **5 AWS SRD高性能网络**

## **5.1 AWS SRD和EFA综述**

!\[\[Pasted image 20240920123105.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

SRD（Scalable Reliable Datagram，可扩展可靠数据报），是AWS设计的可靠的、高性能的、低延迟的网络传输协议；EFA（Elastic Fabric Adapter，弹性互联适配器），是AWS EC2实例提供的一种高性能网络接口。

AWS的SRD和EFA技术主要有如下特征：

- SRD基于标准的以太网，用于优化传输层协议。

- SRD受IB可靠数据报的启发，与此同时，考虑到大规模的云计算场景下的工作负载，利用了云计算的资源和特点（例如AWS的复杂多路径主干网络）来支持新的传输策略，为其在紧耦合的工作负载中发挥价值。

- 借助EFA，使用消息传递接口MPI的HPC应用，以及使用NVIDIA集体通信库（NCCL）的ML应用，可以轻松扩展到数千个CPU或GPU。

- EFA用户空间软件有两个：基本驱动，暴露EFA硬件的可靠乱序交付能力；而在其之上的libfabric实现了包的重排序。

- EFA类似于IB Verbs。所有EFA数据通信都通过发送/接收队列对完成。依靠共享队列机制，可显著降低所需的QPs数量。

- 由消息传递层完成的缓冲区管理和流控制，与应用程序是紧密耦合的。

## **5.2 AWS为什么不选择TCP或RoCE**

AWS认为：

- TCP是可靠数据传输的主要手段，但不适合对延迟敏感的处理。TCP的最优往返延迟大约25us，因拥塞等引起的等待时间可以到50ms，甚至数秒。主要原因是丢包重传。

- IB是用于HPC的高吞吐量、低延迟网络，但IB不适合可扩展性要求。原因之一是RoCEv2的优先级流量控制(PFC)，在大型网络上不可行。会造成队头阻塞、拥塞扩散和偶尔的死锁。即使PFC可用，RoCE在拥塞和次优拥塞控制下仍会遭受ECMP（Equal-Cost Multi-Path routing，等价多路径路由）冲突。

- SRD针对超大规模数据中心进行了优化：跨多个路径的负载平衡、快速恢复、发送方控制ECMP、专门的拥塞控制算法、可靠但乱序的交付等。

- SRD是在AWS Nitro卡中实现。让SRD尽可能靠近物理网络层，避免OS和Hypervisor性能噪音注入。可快速重传并迅速减速以响应队列建立。SRD作为EFA设备公开给主机，EFA是Amazon EC2实例的网络接口。

## **5.3 SRD特征之一：多路负载均衡**

!\[\[Pasted image 20240920123111.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

为了更好的多路负载均衡，AWS的SRD遵循如下原则：

- 为了减少丢包的几率，流量应该在可用路径上均匀分布。

- SRD发送方需要在多个路径上Spray（喷洒）数据包（即使是单个应用程序流），特别是大象流。以最小化热点出现的可能性，并检测次优路径。

- 为了与未启用多路径的传统流量共享网络，因此随机Spray流量是不合适的，SRD避免使用过载路径。

- 因为规模的原因，偶尔的硬件故障不可避免。为了让网络从链路故障中快速恢复，如果原始传输路径不可用，SRD能够重新路由重传的数据包，而无需等待路由更新收敛。

## **5.4 SRD特征之二：乱序交付**

在网络传输中，平衡多条可用路径上的流量有助于减少排队延迟并防止数据包丢失；但不可避免地会导致数据包的无序到达。

与此同时，众所周知，恢复网卡中的数据包排序代价昂贵，网卡通常具有有限的资源（内存带宽、重排序缓冲容量或开放排序上下文的数量）。

如果按顺序发送接收消息，将限制可伸缩性或在出现丢包时增加平均延迟。如果推迟向主机软件发送乱序数据包，将需要一个非常大的缓冲区，并且将大大增加平均延迟。并且许多数据包会因为延迟可能被丢弃，重传增加了无谓的网络带宽消耗。

SRD设计为：将可能乱序的数据包，不做处理，直接传送到主机。

应用程序处理乱序数据包，在字节流协议，如TCP，是不可行的，但在基于消息的语义时很容易。每个流的排序或其他类型的依赖追踪由SRD之上的消息传递层完成；消息层排序信息与数据包一起传输到另一端，SRD是不可见的。

## **5.5 SRD特征之三：拥塞控制**

!\[\[Pasted image 20240920123130.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Incast，是一种流量模式，许多流量汇聚到交换机的同一接口，耗尽该接口的缓冲空间，导致数据包丢失。

多路径Spray减少了中间交换机的负载，但其本身并不能缓解Incast拥塞问题。Spray可能会使Incast问题变得更糟：即使发送方受自身链路带宽限制，来自发送方不同时刻的流量，也可能通过不同的路径同时到达。对于多路径传输来说，拥塞控制将所有路径上的聚合队列保持在最小值至关重要。

因此，SRD CC的目标是：用最少的飞行中的数据量，获得最公平的带宽共享，防止队列堆积和数据包丢失。发送方需要根据速率估计调整其每个连接的传输速率，速率估计依据确认包的时间提示，同时需要考虑最近的传输速率和RTT的变化。

# **6 其他高性能网络技术**

## **6.1 微软Fungible的Truefabric**

!\[\[Pasted image 20240920123136.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

TrueFabric是开放标准的网络技术，提高数据中心的性能、经济性、可靠性和安全性，单集群规模从几个到数千机架。

- 横向扩展数据中心，所有服务器节点都通过可靠的高性能局域网连接。目前，大多数数据中心仍构建在Eth+TCP/IP之上。

- 局域网比广域网速度快三个数量级或更多，TCP/IP成为性能的瓶颈。TCP卸载并不成功，很难在CPU和卸载引擎之间清晰拆分TCP。

- 网络接口和SSD设备的性能比通用CPU提高得更快；系统持续解构，东西向流量激增。I/O技术和业务应用的发展，给网络栈带来了巨大的压力。

- TrueFabric的所有特性，都是通过基于标准UDP/ IP以太网的新型Fabric控制协议(FCP)实现。

- TrueFabric支持：中小规模，服务器节点直连Spine交换机；大规模，两层拓扑，Spine和Leaf交换机；FCP还可以支持三层或更多层交换机的更大规模网络。

- 右图为TrueFabric网络部署的抽象视图，有四种服务器类型：CPU服务器、AI/数据分析服务器、SSD服务器和HDD服务器。每个服务器实例包含一个Fungible DPU。

TrueFabric/FCP的特性

- 可扩展性：可以从使用100GE接口的小规模部署，扩展到使用200/400GE接口的数十万台服务器的大规模署。

- 全截面带宽：支持端到端的全截面带宽，适用于标准IP的以太网数据包大小。TrueFabric支持短的、低延迟消息的高效交换。

- 低延迟和低抖动：提供节点之间的最小延迟，以及非常严格的长尾延迟控制。

- 公平性：在竞争节点之间以微秒粒度公平分配网络带宽。

- 拥塞避免：内置的主动拥塞避免，这意味着数据包基本上不会因为拥塞而丢失。并且，拥塞避免技术并不依赖于核心网络交换机的特性。

- 容错：内置的数据包丢失检测和恢复，错误恢复比依赖于路由协议的传统恢复技术快五个数量级。

- 软件定义的安全和策略：支持基于AES的端到端加密。可以通过配置将特定的部署划分为单独加密域。

- 开放标准：FCP建立在基于以太网的标准IP之上，可以与以太网上的标准TCP/IP完全互操作。

!\[\[Pasted image 20240920123143.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

上图是TrueFabric和RoCEv2在10:1 Incast下的性能比较。右上图可以看到，Truefabric在P99长尾延迟方面，相比ROCEv2有非常显著的提升。从下面两张图的RoCEv2和Truefabric性能抖动对比来看，Truefabric的性能稳定性要显著优于RoCEv2。

## **6.2 阿里云HPCC和eRDMA**

!\[\[Pasted image 20240920123149.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

阿里云的HPCC（High Precision Congestion Control，高精度拥塞控制），其关键方法是利用INT（In-Network Telemetry，网络内遥测）提供的精确的链路负载信息来计算准确的流量更新。

- 大规模RDMA网络在平衡低延迟、高带宽利用率和高稳定性方面面临挑战。经典的RDMA拥塞机制，例如DCQCN和TIMELY算法，具有一些局限性：收敛缓慢、不可避免的数据包排队、复杂的调参参数。

- HPCC发送方可以快速提高流量以实现高利用率，或者快速降低流量以避免拥塞；HPCC发送者可以快速调整流量，以使每个链接的输入速率略低于链接的容量，保持高链接利用率；由于发送速率是根据交换机直接测量的结果精确计算得出的，HPCC仅需要3个独立参数即可调整公平性和效率。

- 与DCQCN、TIMELY等方案相比，HPCC对可用带宽和拥塞的反应更快，并保持接近零的队列。

弹性RDMA（Elastic Remote Direct Memory Access，简称eRDMA）是阿里云自研的云上弹性RDMA网络，底层链路复用VPC网络，采用全栈自研的拥塞控制CC（Congestion Control，HPCC）算法，享有传统RDMA网络高吞吐、低延迟特性的同时，可支持秒级的大规模RDMA组网。可兼容传统HPC应用，以及传统TCP/IP应用。

eRDMA能力主要具有以下产品优势：

- 高性能。RDMA绕过内核协议栈，将数据直接从用户态程序转移到HCA中进行网络传输，极大地降低了CPU负载和延迟。eRDMA具有传统RDMA网卡的优点，同时将传统的RDMA技术应用到VPC网络下。

- 规模部署。传统的RDMA依赖于网络的无损特性，规模部署成本高、规模部署困难。而eRDMA在实现中采用了自研的拥塞控制CC算法，容忍VPC网络中的传输质量变化（延迟、丢包等），在有损的网络环境中依然拥有良好的性能表现。

- 弹性扩展。不同于传统的RDMA网卡需要单独一个硬件网卡，eRDMA可以在使用ECS的过程中动态添加设备，支持热迁移，部署灵活。

- 共享VPC网络。eRDMA依附于弹性网卡（ENI），网络完全复用，可以在不改变业务组网的情况下，即可在原来的网络下激活RDMA功能，体验到RDMA。

## **6.3 新兴的Ultra-Ethernet**

2023年7 月，超以太网联盟 (Ultra Ethernet Consortium，UEC) 正式成立，它是一个由 Linux 基金会及其联合开发基金会倡议主办的新组织。UEC 的目标是超越现有的以太网功能，例如远程直接内存访问 ( RDMA ) 和融合以太网RDMA (RoCE)，提供针对高性能计算和人工智能进行优化的高性能、分布式和无损传输层，直接将矛头对准竞争对手的传输协议 InfiniBand。

人工智能和高性能计算给网络带来了新的挑战，比如需要更大规模、更高带宽密度、多路径、对拥塞的快速反应以及对单个数据流执行度的相互依赖（其中尾延迟是关键考量点）。UEC 规范的设计将弥补这些差距，并为这些工作任务提供所需的更大规模组网。UEC 的目标是一个完整的通信栈，解决跨越多个协议层的技术问题，并提供易于配置和管理的功能。

UEC将提供基于以太网的开放、可互操作、高性能的全通信堆栈架构，以满足大规模人工智能和高性能计算不断增长的网络需求。从物理层到软件层，UEC计划对以太网堆栈的多个层进行更改。

UEC的技术目标是开发规范、API 和源代码，UEC定义：

- 以太网通信的协议、电信号和光信号特征、应用程序接口/数据结构。

- 链路级和端到端网络传输协议，可扩展或替换现有链路和传输协议。

- 链路级和端到端拥塞、遥测和信令机制，均适用于人工智能、机器学习和高性能计算环境。

- 支持各种工作负载和操作环境的软件、存储、管理和安全结构。

UEC传输协议:

- 从一开始就设计为在IP和以太网上运行的开放协议规范。

- 多路径、包喷洒传输，充分利用AI网络，不会造成拥塞或队头阻塞，无需集中式负载均衡算法和路由控制器。

- Incast管理机制，以最小的丢包控制到目标主机的最终链接上的扇入。

- 高效的速率控制算法，允许传输快速提升至线速，同时不会导致竞争流的性能损失。

- 用于无序数据包传送的 API，可选择按顺序完成消息，最大限度地提高网络和应用程序的并发性，并最大限度地减少消息延迟。

- 可扩展未来网络，支持1,000,000个端点。

- 性能和最佳网络利用率，无需针对网络和工作负载进行特定的拥塞算法参数调优。

- 旨在在商用硬件上实现 800G、1.6T 和未来更快以太网的线速性能。

# **7 全球互联网络的高性能**

!\[\[Pasted image 20240920123157.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

随着5G通信基础设施的大规模覆盖，短视频、视频会议等多媒体应用的大量使用，自动驾驶和智能网联汽车的广泛落地，以及VR/AR元宇宙的快速发展，对整个全球互联网络的吞吐量、实时性、稳定性等各个方面提出了更高的要求。

上图为声网通过基于SDN架构的SD-RTN（软件定义全球实时网络）技术，构建全球实时通信网络，利用廉价的边缘节点和独特的智能路由加速技术，让全球端到端延迟控制在400ms以内。

# **8 高性能网络总结**

## **8.1 高性能网络优化总结**

## 高性能网络优化的主要手段：

- 网络容量升级：例如网络带宽升级，把整个网络从25Gbps升级到100Gbps；例如更多的网络端口和连线。

- 轻量协议栈：数据中心网络跟互联网不在一个层级，物理距离短，堆栈延迟敏感，可以不需要复杂的用于全球物理互联的TCP/IP协议栈；

- 网络协议硬件加速处理；

- 高性能软硬件交互：高效交互协议 + PMD + PF/VF/MQ等，如AWS EFA；

- 高性能网络优化：通过①多路负载均衡、②乱序交付、③拥塞控制、④多路处理并行等技术（例如AWS SRD技术），实现①低延迟、②高可靠性（低性能抖动）、③高网络利用率。

## **8.2 从高性能网络协议栈看系统分层**

从高性能网络协议栈看系统分层：

- 系统是由多个分层（layer）组成。

- 每个分层可以当作独立的组件模块，根据需要，选择不同分层，并对业务敏感的特定层进行定制设计和优化，组合出符合业务需求的个性化的系统解决方案。

- 即使分层间有很好的解耦，但每个分层仍不是孤立的；每个分层不宜是一个黑盒，这样无法从系统层次对其进行优化；要想实现极致的性能，需要系统栈不同分层间的通力协作。

- 全栈优化。系统需要不断优化，每个分层需要不断优化，上下分层间的交互接口也需要不断优化，分层的运行平台也需要不断优化。

- 每个分层、交互接口以及整个系统，都需要持续不断的变化，来适应业务需求的变化。

（正文完）

______________________________________________________________________

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)扫描二维码，加入软硬件融合技术社区

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

______________________________________________________________________

更多阅读：

- [算力基础设施的风险与挑战](http://mp.weixin.qq.com/s?__biz=MzkxMDE3MDQxMA==&mid=2247489340&idx=1&sn=22df5fcfc2d38acd98499398551b79f2&chksm=c12ed779f6595e6fa28d956b80c7f9caf699d2e7d0b06ad855cf29e72f0fe95ab18be9223cf4&scene=21#wechat_redirect)

- [融合的系统，融合的计算](http://mp.weixin.qq.com/s?__biz=MzkxMDE3MDQxMA==&mid=2247489287&idx=1&sn=d27332f13a62b05322296b0b122f23c6&chksm=c12ed742f6595e542a692f572ccd397083335097c18db0ebffa7294e5abc6afacd2a59018b7d&scene=21#wechat_redirect)

- [算力网络系列文章（二）：从云计算到算力网络](http://mp.weixin.qq.com/s?__biz=MzkxMDE3MDQxMA==&mid=2247489225&idx=1&sn=cf7a2101401803ed24d02468738944c4&chksm=c12ed68cf6595f9a43a5de5340f29e24cfdfcca0d2bda75307d9c5dfdfae4c2d511def369e9d&scene=21#wechat_redirect)

- [算力网络系列文章（一）：算力提升综述](http://mp.weixin.qq.com/s?__biz=MzkxMDE3MDQxMA==&mid=2247489158&idx=1&sn=4e1b86f2b9076b41061d0c8c93d0f0ce&chksm=c12ed6c3f6595fd5cb5511df5d0a901ed4a84c797b34d5639fd2d0e5d413b1ecd2520a2b06a3&scene=21#wechat_redirect)

- [算力芯片，终局之战？](http://mp.weixin.qq.com/s?__biz=MzkxMDE3MDQxMA==&mid=2247489013&idx=1&sn=a6f10a6bce5b84af891720deb1d7fa0c&chksm=c12ed5b0f6595ca6eeb6fa7c039230a83c3886cd32d4981189ddc8b137a3c07c640c38261990&scene=21#wechat_redirect)

- [什么是第三代通用计算？](http://mp.weixin.qq.com/s?__biz=MzkxMDE3MDQxMA==&mid=2247488968&idx=1&sn=723d99a6146b2ee05fc2c3008c73ee92&chksm=c12ed58df6595c9b88e35c5291cda3e3c774d5a1449938ffde9b6fe1a2142a55badbbfbcf720&scene=21#wechat_redirect)

- [给Kubernetes装上腾飞的翅膀](http://mp.weixin.qq.com/s?__biz=MzkxMDE3MDQxMA==&mid=2247488890&idx=1&sn=dd4f393ea4da577276307dde6ea1b4b1&chksm=c12ed53ff6595c29ba7abb1a9b2e94e24e07d1b0e16ffe39090d1339f9a87d3e2688c31abae0&scene=21#wechat_redirect)

- [连微软都自研芯片了，独立芯片公司的未来在哪里？](http://mp.weixin.qq.com/s?__biz=MzkxMDE3MDQxMA==&mid=2247488858&idx=1&sn=743249e534764c43e33601a943a8aeb4&chksm=c12ed51ff6595c0970746bef3e6a3e9eb3f1d8652b77a46814cf5138cfd9515ea9752089e2b3&scene=21#wechat_redirect)

- [算力芯片，如何突围？](http://mp.weixin.qq.com/s?__biz=MzkxMDE3MDQxMA==&mid=2247488831&idx=1&sn=685df3e6d4c2a5b6aadefb704208a85f&chksm=c12ed57af6595c6cce0edf5979bc67c6b7c47e1141f64c0636c5e43da3c2a4e5e171c6c5a276&scene=21#wechat_redirect)

- [能不能面向通用人工智能AGI，定义一款新的AI处理器？](http://mp.weixin.qq.com/s?__biz=MzkxMDE3MDQxMA==&mid=2247488795&idx=1&sn=851961bec54071a48539da8d9e2ed660&chksm=c12ed55ef6595c48e9f0d4098cbeff33b900e64dfffca007b78f00a358c44b667a35484c4f48&scene=21#wechat_redirect)

- [什么是软硬件融合？](http://mp.weixin.qq.com/s?__biz=MzkxMDE3MDQxMA==&mid=2247488674&idx=1&sn=6d467342338c2f0bfc3e886ed8b4e932&chksm=c12ed4e7f6595df145b7bd6e94c36ad7e8fdb21cfbef508dfab79dc3024626f55bad8d62970a&scene=21#wechat_redirect)

- [这可能是计算机体系结构领域一个重要的里程碑事件](http://mp.weixin.qq.com/s?__biz=MzkxMDE3MDQxMA==&mid=2247488645&idx=1&sn=74d1ef8c9ebb9271b6fc8d4c345249dd&chksm=c12ed4c0f6595dd6e9a7961cec2e0359976cda0562e305762460cbb0e9fe2b87b347a5011819&scene=21#wechat_redirect)

- [第三代通用计算，大算力芯片”弯道超车“的历史时机](http://mp.weixin.qq.com/s?__biz=MzkxMDE3MDQxMA==&mid=2247488453&idx=1&sn=75c523109e058073b521be8e442b99df&chksm=c12ed380f6595a9621c5ed6227ede9bf0b71d94146869f73dac2ac141f2e2e48ccaed195cfa7&scene=21#wechat_redirect)

- [从ChatGPT等大模型的兴起，看未来计算芯片的发展趋势](http://mp.weixin.qq.com/s?__biz=MzkxMDE3MDQxMA==&mid=2247488290&idx=1&sn=ead71d194e0c8ac82a1fc8520f6c5730&chksm=c12ed367f6595a7128353a97f3b7beb2d30a4238f35f726780371f60f80ed673a8b7746da971&scene=21#wechat_redirect)

- [从算力网络发展，看未来十年的宏观算力体系](http://mp.weixin.qq.com/s?__biz=MzkxMDE3MDQxMA==&mid=2247488237&idx=1&sn=c157d057bd7ec61161d3e7469517dec3&chksm=c12ed2a8f6595bbef20cfe716e88f2f3d045597a23d3b01713e608e7f711779bbdcd9ddbe654&scene=21#wechat_redirect)

- [超异构处理器HPU和系统级芯片SOC的区别在哪里？](http://mp.weixin.qq.com/s?__biz=MzkxMDE3MDQxMA==&mid=2247488203&idx=1&sn=5828f791ecef28a95449568093d62168&chksm=c12ed28ef6595b982633232a03e4eb434123dd6183ca526fe13fe6a5deeb609b35edb68c4bd9&scene=21#wechat_redirect)

- [超异构计算时代的操作系统架构初探](http://mp.weixin.qq.com/s?__biz=MzkxMDE3MDQxMA==&mid=2247487967&idx=1&sn=708114e3ce500acf64ffaf2668ed8263&chksm=c12ed19af659588c6b6bf1fe86eea98ad744f60563d6b72918efc5ef70dad2a3f10490716064&scene=21#wechat_redirect)

- [异构计算面临的挑战和未来发展趋势](http://mp.weixin.qq.com/s?__biz=MzkxMDE3MDQxMA==&mid=2247487918&idx=1&sn=6d879ad760e25ae06b19feae8691447b&chksm=c12ed1ebf65958fdcffcb97ca386ae9f97030c1f126ae2f4525713831b2829cf17e2b809e01b&scene=21#wechat_redirect)

- [为什么要从“软硬件协同”走向“软硬件融合”？](http://mp.weixin.qq.com/s?__biz=MzkxMDE3MDQxMA==&mid=2247487597&idx=1&sn=88a3333b5dc22c1c63238fd722f0c2cd&chksm=c12ed028f659593e32debcb8d6fd92847d11cda76d057d6d85e8b737ab2f0f63e738f1d1893c&scene=21#wechat_redirect)

- [未来，所有芯片公司都需要进化成互联网公司](http://mp.weixin.qq.com/s?__biz=MzkxMDE3MDQxMA==&mid=2247487437&idx=1&sn=5a5ac38b22a3dfd600f47f1d6c41602d&chksm=c12ecf88f659469ea10a35f4da7e4b312cc19116241c96da8449249cbe9f7d3a99e7a9e44e2e&scene=21#wechat_redirect)

- [从NVIDIA自动驾驶芯片Thor，看大芯片的发展趋势](http://mp.weixin.qq.com/s?__biz=MzkxMDE3MDQxMA==&mid=2247487393&idx=1&sn=5545d1c5dea7d7afd66558e9708fc3dc&chksm=c12ecfe4f65946f237c6b673e81f54e78b9f2ef001a3b8b8a83be4d64d9e90b549add21ae480&scene=21#wechat_redirect)

- [“黄金年代”之后，计算机体系结构将何去何从？](http://mp.weixin.qq.com/s?__biz=MzkxMDE3MDQxMA==&mid=2247487273&idx=1&sn=ca48d4ca83e7ac4924e2357b181465b9&chksm=c12ecf6cf659467a9f4901d8b45f7f4312d5462f85dbe9058a7377a16ce2719f13ccc0bf6673&scene=21#wechat_redirect)

- [从DPU看大芯片的发展趋势](http://mp.weixin.qq.com/s?__biz=MzkxMDE3MDQxMA==&mid=2247487054&idx=1&sn=6a6a9016077d897b4998e273acf13007&chksm=c12ece0bf659471d524c8aaf39caa8d3c34a4b37516207ebfc3ef7a4589e4d39edbc3285844c&scene=21#wechat_redirect)

- [大芯片面临的共同挑战](http://mp.weixin.qq.com/s?__biz=MzkxMDE3MDQxMA==&mid=2247487007&idx=1&sn=03e4a3bea7490f60b74bed4edb0df9db&chksm=c12ece5af659474c23ff8a733411ea164b55e83b8d16325e7dc48f6242252dd4e85d9135b429&scene=21#wechat_redirect)

- [大算力芯片，向左（定制）还是向右（通用）？](http://mp.weixin.qq.com/s?__biz=MzkxMDE3MDQxMA==&mid=2247486727&idx=1&sn=aa30cf0bc28e8a9dceca4fbb1c720856&chksm=c12ecd42f65944549caba4353da71f41212e9776261438833668a28b579fd8d2d2b774e21b26&scene=21#wechat_redirect)

- [DPU发展面临的困境和机遇](http://mp.weixin.qq.com/s?__biz=MzkxMDE3MDQxMA==&mid=2247486416&idx=1&sn=fddfc02b776ea1e6704800cdeda07350&chksm=c12ecb95f6594283ce1442ade60e55f43d2df0a20c7fa8d5dbb8c0245a76131f09e7035da597&scene=21#wechat_redirect)

- [Chiplet UCIe协议已定，CPU、GPU、DPU混战开启，未来路在何方？](http://mp.weixin.qq.com/s?__biz=MzkxMDE3MDQxMA==&mid=2247485870&idx=1&sn=b92e1df5594c7a4399e382cc5982ed40&chksm=c12ec9ebf65940fd35fa4aba47ac65947d463f0b0682c10945440dcc1bc21b819cb85bf13e75&scene=21#wechat_redirect)

- [“DPU”非DPU](http://mp.weixin.qq.com/s?__biz=MzkxMDE3MDQxMA==&mid=2247485656&idx=1&sn=345571fb94cba06798efb690afa42568&chksm=c12ec89df659418bb3a19246666576d4b4b9a5a19854467153ce74802ead2ff7db20a782f255&scene=21#wechat_redirect)

![](https://mmbiz.qlogo.cn/mmbiz_jpg/ZREJwn5W6toMMZnEZu8TqGtS6W1IzCAPDoSJ8abD1icxiaQz0dibeVbZQiaiaviciaDyGnm2Gl4vJvyeNsVYmKdicDPhdQ/0?wx_fmt=jpeg)

Chaobowx

写作辛苦，赞助作者一点咖啡钱☕

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=MzkxMDE3MDQxMA==&mid=2247489402&idx=1&sn=ff76a5458dea17652c405dd17a2ae13a&chksm=c12ed73ff6595e29a45a5d309c18d52ac0b8d68f96fa91028b9a904a16e316121c950bab1478&mpshare=1&scene=24&srcid=0605oHyTZ5kv7ufTc7EbUmfH&sharer_shareinfo=a2ff3014339f58e02bc8139f69cc3444&sharer_shareinfo_first=a2ff3014339f58e02bc8139f69cc3444&key=daf9bdc5abc4e8d08e4c92eb3bf4b63d960720e35d22c29d562e69275d791f5591d2377bd4dd711f10c00071223b6d09e8261673d2396260a155a1faf18db8f3ad5c3c5341da0469816420f6c39bd9ec73144a4e01b1cf0dd61b948915226d5ebdad369eb891949458a5f96252474542d73e59d930e5bc3c32c860f03aaad3cb&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090621&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQAZPSolbbWRvQBreFC6xAnRLmAQIE97dBBAEAAAAAACWiBTGRiv4AAAAOpnltbLcz9gKNyK89dVj0CFosSTiO7LVK%2F%2Fr8G%2FRk2eLhUc2EYBOHie%2Bu7R8CYaApENy8kGCxSqb4hw7KLXKuW3q8AUbv2ju2rjIBykhd5wvFo%2BC587a7GYFoP8PZQrzIKOV%2BNMkp55XXzE1qhMaSwNOnqK0BjC3vt3%2B3jQOHM8mYS3LM0x72uDrmGHwrLso268anW6mhXc1GVV4M4WPKixe7vTeJM3bzxuOuIYflo5jOsxlvHdAqxT77XkqZHB0ho5IeRANLomai5arKA45Y&acctmode=0&pass_ticket=pqHtB70YZBFUBJZ0wM0QYXkbBY2GIFNMJv414YcTy6GFRT1XE%2BOFPtrNvI%2FU3qno&wx_header=1)钟意作者

高性能网络12

硬件加速2

高性能网络 · 目录

上一篇亚马逊AWS高性能网络技术SRD：用于弹性可扩展的HPC云优化传输协议

阅读原文

阅读 3877

​

写留言

**留言 4**

- Tim

  北京6月5日

  赞

  1.4、1.5真是点到即止啊，PFC/ECN以及AR这些技术的优化才是以太网承载RDMA的核心。

  软硬件融合

  作者6月5日

  赞2

  你说的对，欢迎就这方面内容深入探讨。![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 子誉

  北京6月28日

  赞

  开元云科技（Kaiyuan_Cloud）申请开白，会严格按照要求转载，感谢

  软硬件融合

  作者6月28日

  赞

  已全局开白![[抱拳]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/ezFXiawTIBGd6JdszIEQ1mkM8aHd9liauicLQ1uHAhHyDebll80WKmjE9KnJPoWpO3oMmvgW6Vjj1pU6UY2ZhpjSg/300?wx_fmt=png&wxfrom=18)

软硬件融合

3752120

4

写留言

**留言 4**

- Tim

  北京6月5日

  赞

  1.4、1.5真是点到即止啊，PFC/ECN以及AR这些技术的优化才是以太网承载RDMA的核心。

  软硬件融合

  作者6月5日

  赞2

  你说的对，欢迎就这方面内容深入探讨。![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 子誉

  北京6月28日

  赞

  开元云科技（Kaiyuan_Cloud）申请开白，会严格按照要求转载，感谢

  软硬件融合

  作者6月28日

  赞

  已全局开白![[抱拳]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

已无更多数据

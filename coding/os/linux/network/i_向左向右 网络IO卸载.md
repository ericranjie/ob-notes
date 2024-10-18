
Linux阅码场 _2022年01月27日 08:02_

以下文章来源于大魏分享 ，作者魏新宇

https://github.com/davidsajare/david-share.git


\*\*Statement:

**This views and opinions expressed in this account are those of my own and do not represent those of my employer, NVIDIA.**

**The information covered in this article is public information on the Internet and does not contain any commercial or technical secrets. I will give reference links after the article.**

**本文仅代表作者个人观点，本文中的内容不能作为生产上的指导！**

**本文在书写过程中，得到了我同事韩潮的指点，在此表示感谢！**

**RDMA与DPDK**

现在被大量应用的网络I/O加速主要是RDMA和DPDK两种。

- RDMA/RoCE适合大量内存拷贝类的应用，如分布式存储等。降低CPU使用率，降低延迟。

- OVS-DPDK会增加流表的插入速度、适合大量包转发。大概能做到 210M PPS左右（具体内容见下图和下面链接）。

http://fast.dpdk.org/doc/perf/DPDK_21_08_Mellanox_NIC_performance_report.pdf

![[Pasted image 20241018125607.png]]

Mellanox是RDMA的发起者之一，不再赘述。在DPDK方面，每个网卡厂商自己的PMD驱动。NVIDIA Mellanox的PMD驱动有两个： mlx4 和mlx5。

The two NVIDIA PMDs are mlx4 for NVIDIA® ConnectX®-3 Pro Ethernet adapters and mlx5 for ConnectX-4 Lx, ConnectX-5, ConnectX-5 Ex, ConnectX-6, ConnectX-6 Lx, ConnectX-6 Dx, and NVIDIA BlueField®-2 Ethernet adapters SmartNICs and data processing units (DPUs). NVIDIA PMDs support bare-metal, Kernel-Based Virtual Machine (KVM) and VMware SR-IOV on x86_64, Arm, and Power architectures.\
NVIDIA PMDs are part of the dpdk.org starting with the DPDK 2.0 release (mlx4) and DPDK 2.2 release (mlx5).

详细内容见如下链接：

https://developer.nvidia.com/networking/dpdk

**e-switch的工作原理**

NVIDIA Mellanox网卡能做网络I/O卸载，本质上是因为它内嵌了一个硬件的e-switch。

![[Pasted image 20241018125622.png]]

我们先介绍一下e-switch的工作原理：

网络首报文过来抵达网卡后，网卡首先识别报文，提取报文头的field，基于多元组对包进行识别和分类（如3、5、7、11元组）、基于配置的规则进行match报文。match完毕后，会去查流表，看是一条新流还是旧流。新的流走CPU，由kernel ovs数据路径处理。查到规则后，把流表配置到硬件，这样硬件就知道后续的报文怎么进行处理。

后续的报文的流表在eswitch有记录的话，提取多元组进行匹配，查到一个表象，表象里有action，然后就会执行对应的action，如forward、drop、header rewrite。然后报文再出去。这是ASAP简单的处理流程。

除此之外，eswitch可以做tunnal头的encap和decap。识别隧道报文，然后进行封包解包。VxLAN、GRE等。

那么，eswitch向上如何呈现？或者说如何使用？

有两种方式。

1.SRIOV，以VF的形式呈现给host。每个VF直接通VM。

![[Pasted image 20241018125729.png]]

2.通过VirtIO的形式呈现。

![[Pasted image 20241018125737.png]]

上面这两种使用方式，对于报文的处理没有任何区别。但对于Guest的使用有区别。

最明显的是：VF方式的性能高。

如果driver访问网卡，如果按照virtIO方式，实际上需要在ASIC里转成普通硬件识别的格式，然后再到ASIC进一步处理，反之亦然。这一次转换决定了性能的上限是40Mpps左右，而VF是直通的，可以近似达到PF的性能。

![[Pasted image 20241018125747.png]]

除了性能角度，SR-IOV和VirtIO最大的功能差异点是：是否需要做热迁移。需要做热迁移，目前virtI/O会比较成熟。VirtI/O不需要对GuestOS进行更改，比较适合云环境。SRIOV需要Mellanox的Driver。

接下来，我们进一步对比RDMA、DPDK的使用场景。

**RDMA**

![[Pasted image 20241018125757.png]]

应用场景

- Messages send/receive, remote DMA and remote atomics Hardware transport

- Kernel and hypervisor bypass

RDMA/RoCE优势：

- Lower latency – 10us → 700ns  低延迟

- Higher message rate – 215M messages/s   高包转发率

- Lower CPU utilization – 0%    低CPU

我们在使用RDMA的时候，需要调用ibvirbs，而不是像普通的应用调用socket。

写应用大多数调用的都是VPI Verbs API。早期Mellanox对于API的划分方案是按照下图四种划分，新的划分方式是按照libibverbs和librdmacm两种划分。

![[Pasted image 20241018125806.png]]

![[Pasted image 20241018125817.png]]

![[Pasted image 20241018125823.png]]

![[Pasted image 20241018125836.png]]

![[Pasted image 20241018125840.png]]

具体VPI Verbs API的使用，详见：

[几个常用VPI Verbs API：RDMA编程系列1](http://mp.weixin.qq.com/s?__biz=MzAwMDc2NjQ4Nw==&mid=2663510413&idx=1&sn=b26e055e9a2150ca80225efca0651db1&chksm=81d6b6f5b6a13fe3027abad2314e330505cb75ba7bb639716c5b7a72992a0f20bd940703cd3d&scene=21#wechat_redirect)

[从一个C代码开始：RDMA编程系列1](http://mp.weixin.qq.com/s?__biz=MzAwMDc2NjQ4Nw==&mid=2663510393&idx=1&sn=cb76b9e543ce30351eceb19e27e0cd67&chksm=81d6b601b6a13f1715f53681fabf4c45f7f8ab646ba8bf762d4101564a37dd847010ca61023a&scene=21#wechat_redirect)

[RDMA性能调优与故障诊断思路](http://mp.weixin.qq.com/s?__biz=MzAwMDc2NjQ4Nw==&mid=2663510300&idx=1&sn=34df88dc6e55c32d0e2db4fcb23eac0a&chksm=81d6b664b6a13f72f9fae392e10d9e41e2d043cf1362723fe6a49a5d7d510f51f14ca7bb4dfb&scene=21#wechat_redirect)

[六板斧-有损RDMA网络](http://mp.weixin.qq.com/s?__biz=MzAwMDc2NjQ4Nw==&mid=2663510335&idx=2&sn=6e576f5616db5b19e73efc8da9144a3f&chksm=81d6b647b6a13f519b5c0829f8709b128c34fa6b6f24fcb5ba7061088e4e2037f9465094d7bb&scene=21#wechat_redirect)

需要指出的是，如果进行RDMA开发，建议使用UCX，更为智能和方便。

https://openucx.org/

![[Pasted image 20241018125858.png]]

由于篇幅有限，本文对UCX不展开介绍，后续会详细说明。

**DPDK**

\*\*PMD运行在用户态\
\*\*

Poll Mode Driver is running in the user space

Access the Rx and Tx descriptors directly without any interrupts

Control Path – through kernel modules

- Use Verbs API (standard openfabrtics.org API)

- Hardware Agnostic

- Objects – create, initialize, modify, query, free

Data Path

- HW Provider specific interface

- Send, Poll

- Highly efficient and optimal code

针对下图，我们把它想象成一个Host（还没有VM啥事）。

针对控制面的流表下发：

OVS-kernel是走的tc下发；

OVS-DPDK 通过rteflow下发；

针对数据面：

OVS-kernel不做卸载走linux kernel；

OVS-kernel卸载走网卡eswitch；

OVS-DPDK通过PMD直接轮询网卡硬件收发包；

RoCE/RDMA通过rdma verbs bypass kernel到网卡硬件；

数据平面如果卸载到网卡的e-switch，尤其是长链接，DPDK没有明显的加速作用。对于首包，DPDK可以发挥作用。此外，控制面也可以通过DPDK发挥作用。在OFED的安装的时候指定DPDK，但这时候并没有给DPDK指定Core。这时候请求还是需要CPU中断响应。所以最好是配置1个Core。但如果只是PoC的话，不配置也可以。

![[Pasted image 20241018125910.png]]

DPDK应用场景：

▪ Network Acceleration

▪ Proprietary API

▪ Handles raw-Ethernet only  这点很关键，就是DPDK处理的就是以太网的包。

▪ Requires integration with application

▪ Keen on throughput performance

▪ Provides abstraction layer to support multiple

architectures (x86, ARM, POWER etc) and HW NICs

(Intel, Mellanox, Cavium etc)

关于DPDK相关的配置，此前已有文章涉及，详细参考：

[VF LAG结合OVS-DPDK的配置](http://mp.weixin.qq.com/s?__biz=MzAwMDc2NjQ4Nw==&mid=2663510423&idx=1&sn=c6ef8bf8c1b8a522db9b90c47ab6e45a&chksm=81d6b6efb6a13ff9a073e920dcca154c5f524cc7e8fb59dec674fef1b1a9b26b637ea879abba&scene=21#wechat_redirect)

[OVS-DPDK硬件卸载](http://mp.weixin.qq.com/s?__biz=MzAwMDc2NjQ4Nw==&mid=2663510079&idx=1&sn=caaef71300d25336408cc95b7c7664cc&chksm=81d6b747b6a13e518a6e72f6d044dfd9b35f646c83a77dddc14a76334418f461e826e2fc909a&scene=21#wechat_redirect)

[DPDK性能评估](http://mp.weixin.qq.com/s?__biz=MzAwMDc2NjQ4Nw==&mid=2663507148&idx=1&sn=7515acc28850cdde2fc36153a1a8dbd0&chksm=81d6a3b4b6a12aa25884615259384088ea30e9ffad6a0d39184590c1116271cd8c62b2ffcdd3&scene=21#wechat_redirect)

[DPDK的编译与测试](http://mp.weixin.qq.com/s?__biz=MzAwMDc2NjQ4Nw==&mid=2663507113&idx=1&sn=178f184c095347efcdc000fd7c5e32c3&chksm=81d6a3d1b6a12ac7364aa0710fae4f90bb6836bcd888b6c254f724bdd66b8721383eb85e42da&scene=21#wechat_redirect)

最后，我们想一个问题：对于CX网卡，既然硬件能够做流卸载，那么OVS-kernel和OVS-DPDK的区别是什么？

1. 数据面首包的加快处理

1. 控制面的加快处理

本质上，DPDK是加快了流表的插入速度，但如果网络都是长链接，没有新的流表插入，那么是否配置DPDK区别并不是很大，但在生产环境，这种情况是不存在的。

此外，上图展现的是Host，如果有VM，那么VM也可以起DPDK或者不起DPDK，这主要看客户的业务需求。

**结论：**

1. NVIDIA Mellanox网卡支持内嵌一个eswitch，它应用OVS卸载、overlay网络封包和解封包。

1. eswitch向上提供SR-IOV或VirtI/O。而无论SR-IOV或VirtI/O的呈现，都支持OVS-Kernel和OVS-DPDK。

1. 针对CX网卡网络流量

针对控制面的流表下发：

OVS-kernel是走的tc下发；

OVS-DPDK 通过rteflow下发；

针对数据面：

OVS-kernel不做卸载走linux kernel；

OVS-kernel卸载走网卡eswitch；

OVS-DPDK通过PMD直接轮询网卡硬件收发包；

RoCE/RDMA通过rdma verbs bypass kernel到网卡硬件；

https://community.mellanox.com/s/article/mellanox-dpdk

https://developer.nvidia.com/networking/dpdk

https://www.openvswitch.org/support/ovscon2020/slides/ovs-offload-design-challenges.pdf


---


写留言

**留言 1**

- Cloud Huang

  2022年1月27日

  赞

  减少中断，卸载CPU处理负载，虚拟机通过函数直接轮询物理网卡通讯，犀利

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/YHBSoNHqDiaH5V6ycE2TlM9McXhAbaV9aUBaTAibVhLibXH1rL4JTMb2eoQfdCicTm4wycOsopO82jG3szvYF3mTJw/300?wx_fmt=png&wxfrom=18)

Linux阅码场

755

1

写留言

**留言 1**

- Cloud Huang

  2022年1月27日

  赞

  减少中断，卸载CPU处理负载，虚拟机通过函数直接轮询物理网卡通讯，犀利

已无更多数据

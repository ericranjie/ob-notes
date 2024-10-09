# 

Linux阅码场

_2022年01月18日 08:02_

以下文章来源于软硬件融合 ，作者Chaobowx

\[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM7vVtWeSIlqsLH8N7fKMff8nUX9DiaBC2wuKSbUPdMmCQA/0)

**软硬件融合**.

软件迭代越来越快，硬件迭代却越来越慢，软硬件之间的鸿沟越来越大。软硬件融合，既是理论和理念，也是方法和解决方案。让硬件灵活可扩展，弥补软硬件之间的鸿沟。 本公众号致力于传播和推广“软硬件融合”相关技术和理念。

\](https://mp.weixin.qq.com/s?\_\_biz=Mzg2OTc0ODAzMw==&mid=2247502837&idx=1&sn=a0a7c6e0f95c4cf185d7e59c07edffd8&source=41&key=daf9bdc5abc4e8d040bffbfff003d852841edd8336f6f2064189bade13ce61c8402ec042759507ba053f2ac44bd427d7bb64c14dc7fd95b6695d6d320f394a40b6fb95dfe42e878237cd12a37e60a880935291a16dbab23a296553e1ae66dc5af824e5662ccfe02c41247896f7d53f1d4aee2550f486ebd5ff950205a791d30d&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQjDGpqXa5UAzKZSq7Ut5B6hLmAQIE97dBBAEAAAAAAK8TKpapSVoAAAAOpnltbLcz9gKNyK89dVj0h%2FM9iuCufE8qD%2Fi%2BKhwjxbJ5ugyzJ%2Fysviu2Aguqeee70TNkGVTK5LTFb99hub3KPRNW%2FayPdtZYY%2BMGRTRkSYiIPWKgzSgsbyWva9vZpl4sxq2QJO3DISkUa8jlWLoTO%2BeMV7TFnQ85MDg12XvMGJtJ9Mz8pCTRHRr66e1lviNgdGY8tyDq8wrdlbH5WJkY5NsE5arLCvMq6rnE4DWoNKdYQosKGTzC7WQuGG1Cz8iQNcvszmG%2BH%2Bo%2FlhUVO%2Bwy&acctmode=0&pass_ticket=BOsindNPt2c%2BgB6ln8Osnn4GXFfo8R42uD7p4zBUvdERtstDcUVe8daqduu7Hy1m&wx_header=1#)

______________________________________________________________________

编者按

很多公司都号称自己做DPU，例如：

- 有把基于FPGA的加速卡方案称之为DPU的；

- 也有把增加了网络加速的智能网卡称之为DPU的；

- 甚至有把增加了加解密、压缩等功能的增强型存储控制器称之为DPU的。

DPU是一个筐，什么都往里装。到这里，事情已经让大家眼花缭乱了。那么，到底什么是DPU？这里我们给出三个层次的标准：

- 第一层，帮助CPU减负。CPU的协处理器，把一些通用的任务卸载并加速。

- 第二层，支撑CPU工作的基础设施处理器。是否能够作为云基础设施处理器，融入IaaS等相关软件服务，隔离基础设施层和业务应用层，把DPU变成一个宏服务支撑平台，支持CPU和GPU侧的上层客户的云应用；

- 第三层，跳脱束缚，独立的算力解决方案。是否真的做到算力（数量级）的显著提升，形成数据中心甚至云网边端融合的整体算力解决方案，与此同时，算力足够灵活可编程可驾驭。

接下来，我们深入探讨。

______________________________________________________________________

# **1 不同视角的计算机体系结构演进**

## **1.1 硬件加速的视角**

CPU性能不够，因此需要硬件加速。这句话就像1+1一样简单明了，但当要真正做硬件加速的时候，会发现问题其实比想象的要复杂的多。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/ezFXiawTIBGeib64vAYEiao6Xrmpt9OMtJxV4bk2tDicnWF9aBibZThDR9REJl9SrntxWzjPNqE9EeDvQwpHxzMPhRQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

数据中心的算力需求一直在增长，但CPU的性能瓶颈了，因此我们需要有个平台，来帮助CPU承担绝大部分算力的压力，而让CPU专注于应用层算力需求虽然不大但非常高净值的工作。承载这个艰巨任务的平台，就是我们的DPU（姑且称为DPU吧，虽然这个名称不够准确）。计算任务从CPU卸载到DPU，不是一蹴而就的事情，而是个不断发展的过程（如下阶段划分，只是说明趋势，不代表一定是如下所描述的每个阶段的严格清晰划分）：

- 第0阶段，卸载0个任务。起始阶段，所有的事情都是在CPU侧完成，外部只是一个网络接口卡。

- 第1阶段，卸载1个任务。网络遇到问题，网络卸载，成为智能网卡。分布式远程存储遇到问题，卸载存储，成为智能存储卡。同样的，要是本地存储也需要做一些额外的处理，可以增加一张本地存储卡（注意，不是存储控制器）。例如，AWS Nitro的VPC卡、EBS卡、本地存储卡就分别承担上述三个功能的卸载。

- 第2阶段，卸载2-4个任务（非严格定义）。CPU的功能任务不但卸载，还需要卸载的功能集成到一个平台。比如把最底层的网络、存储、虚拟化和安全四大类功能从CPU侧卸载到DPU中。

- 第3阶段，卸载5项以上。更准确的说，是把整个系统栈里能够卸载的任务都尽可能的卸载到DPU中。这里可以给出卸载的一个更加通用的标准：①性能敏感，占据较多CPU资源；②广泛部署，运行于众多服务器。当整个系统栈都尽可能进行卸载加速之后，IPU的名称要更准确一些（IPU，基础设施处理器）。

- 第4阶段，不但全量卸载，还需要均衡和弹性。卸载下来的任务需要更多的灵活性，形成弹性的基础设施支撑平台。或者说，需要把IaaS的服务融入到DPU中，并且这些服务的业务逻辑需要仍由云服务提供商CSP的软件工程师负责定义，并且能够很好的支持多租户、微服务、一致性等云的高级特征。

## **1.2 I/O优化的视角**

### **传统异构计算的问题**

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/ezFXiawTIBGeib64vAYEiao6Xrmpt9OMtJxibDXz9vD2lGG0Y8XBejIGyg1LCK0KzulY4dOQwz68UWBk5mYme6Nlicw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

异构加速的实现架构通常是CPU+GPU/FPGA/DSA，主要由CPU完成不可加速部分的计算以及整个系统的控制调度，由GPU/FPGA完成特定任务的加速。这种架构面临一些挑战：

- 可加速部分占整个系统的比例有限，例如加速占比为80%，则加速最高不超过5倍；

- 数据在CPU和加速器之间来回搬运的影响，加速比率打了折扣，有些场景综合加速效果不明显；

- 异构加速显式的引入新的实体，计算变成两个或多个实体显式的协作完成，增加了整个系统的复杂度；

- 虽然GPU相比CPU性能提升不少，但是相比DSA/ASIC的性能，还是有显著的差距；而DSA/ASIC的问题则在于，无法适应复杂场景对业务灵活性的要求，导致大规模应用成为巨大的门槛；

- CPU+xPU架构，是以CPU为中心，整个IO路径很长，IO成为性能的瓶颈。

### **基于I/O的优化**

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/ezFXiawTIBGeib64vAYEiao6Xrmpt9OMtJxME59LTNnY8Yaiay47vaqqCrtLpGEtlhlUicKHwFuicz6jCsSSjVqiavh2w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这里我们给出来两个概念，基于传统“协处理器”概念，我们扩展出“基处理器”的概念：

- 协处理器：挂在CPU旁边的协助CPU工作的专有处理器，例如GPU、TPU等算是协处理器。

- 基处理器：在CPU之下，支撑CPU和协处理器的工作的基础设施处理器。如智能网卡、DPU/IPU。

如图所示，I/O的视角看CPU的加速，大概有四个类型：

- 第一类，协处理器。传统的异构计算架构都是作为CPU的协处理器的方式，随着I/O的数据量增速超过CPU计算的增速，CPU的计算能力和I/O处理能力都成为所有性能瓶颈的本质原因，基于CPU为中心的架构需要变革。有一些PCIe P2P的技术，例如NVIDIA的GPUDirect，可以缓解一部分性能问题，但没有改变问题的本质。数据交互依然要跨两条PCIe串行总线；并且输入输出两次跨越，对PCIe的带宽要求也是翻倍的。

- 第二类，用基处理器给协器做旁路。通过DPU卸载一部分CPU的工作之后，数据可以有部分Bypass，能够减轻CPU一半的压力，并且提升I/O带宽的同时降低I/O延迟。

- 第三类，只有基处理器。这样，整个I/O路径就要简单直接许多，整个计算可以看做是基于数据流Datapath的不同Stage的处理。从网络到应用，再从应用到网络。

- 第四类，所有处理在基处理器完全。通过运行于CPU的控制面和慢路径，定义好之后，绝大部分数据流量的处理都在基处理器中就可以完成，不需要进CPU。

## **1.3 虚拟化Host和Guest分离的视角**

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/ezFXiawTIBGeib64vAYEiao6Xrmpt9OMtJxPjXlicnRaj5h977ZQxkjytJcyLCOe0ia6wphWDLbyT37BvibWSGMia8amA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

上图概要的介绍的计算机虚拟化的三种资源类型和三种虚拟化类型。CPU和内存的虚拟化通过CPU的“加速机制”支持，需要额外独立芯片平台支持的加速主要是I/O设备的加速。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/ezFXiawTIBGeib64vAYEiao6Xrmpt9OMtJx8T7tLhgf7oUSrPsI9NLb7SdsmbMib3BOW8nNq8ujrny54hvjAXdjFfQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

上图可以简单的说，虚拟化卸载加速，就是把整个I/O栈，整体下沉到硬件的过程。

从虚拟化的角度，最开始我们是要把I/O设备加速，这样，VM要直接访问I/O设备，就需要VT-x相关的技术和设备端的PCIe SR-IOV的支持。然后呢，所谓卸载的设备，其实只是个设备呈现给上层软件的接口而已。在设备之下，有后台的I/O类的处理任务，比如网络VPC、存储EBS客户端这些任务。也就不得不跟着虚拟设备接口一起下沉到硬件。同时，我们也实现了这些网络、存储类任务的卸载加速。

可以说I/O模拟设备、I/O任务、虚拟化的下沉是“三位一体”的。

## **1.4 总结**

总结一下，不管是从硬件加速的视角，I/O路径优化的视角，还是虚拟化的视角，计算机体系结构的演进，最终都殊途同归，指向了一件事情：

- 需要有一个基处理器，来承担整个卸载加速的平台化解决方案。

- 最重要的，是要能够支撑IaaS层的各类云服务，（用AWS举例）如各类EC2、VPC、网关、防火墙服务、网络LB、本地存储，分布式存储、可信计算、“零信任”等等。

- 能够做到基础设施层和业务应用层隔离。

# **2 狭义DPU**

## **2.1 DPU的定义**

### **DPU要实现业务和基础设施分离**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

DPU的创新最开始是AWS做的，最原始驱动力就是以整个虚拟化架构的一些挑战开始的：

- 一方面是虚拟化的开销越来越大；

- 另一方面，是因为基础设施和业务处于同一个计算平台，基础设施的性能突发会干扰业务的性能稳定度；

- 还有一个问题就是安全访问，宿主机侧管理具有业务虚机等的所有权限，运维管理的一些误操作，以及宿主机OS的一些漏洞被攻破后，黑客不但可以破坏宿主机，还可能导致用户的数据安全风险。

业务和基础设施分离，可以做到：

- CPU资源完全交付，整体成本降低，CPU资源可以卖更多的钱；

- 此架构虚拟化开销非常小，可以支持虚拟化“嵌套”（比喻，严格来说不是嵌套），在业务CPU部署企业级虚拟化系统，方便传统客户轻松上云；

- 基础设施和主机侧系统完全物理隔离，主机侧独立安全域，安全访问；

- 能够兼顾物理机和虚拟机两种技术架构的优势；

- 此架构可以支持把服务器部署在私有云场景，然后由公有云运营商统一运维，统一了公有云和私有云运维；

- 等等。

### **DPU需要支持基础设施层的性能敏感任务的卸载加速**

基础设施层的任务主要有四类：

- 第一类，虚拟化。虚拟化需要在Host CPU侧存在一个轻量的Hypervisor Agent，然后在DPU支持Hypervisor、呈现给Host的设备管理、设备迁移等。

- 第二类，网络类任务加速。网络类任务在云场景，主要是用来做租户隔离的虚拟网络，如NVGRE和VxLAN协议处理等，以及支持网络vSwitch相关的软件应用。网络类的任务处理非常消耗CPU资源，因此必须要通过硬件加速。为了支持一个更加可编程的网络平台，还需要引入硬件级别可编程的技术。例如Intel在IPU中集成了Barefoot PISA架构网络处理引擎，能够实现ASIC级别性能的同时，能够支持P4的编程。

- 第三类，存储类任务加速。需要支持本地存储和远程分布式存储，集成存储客户端相关处理逻辑。存储对性能和延迟敏感，需要高性能网络和存储处理加速。

- 第四类，安全类任务加速。安全是多方面的，网络安全类场景和存储数据安全也是安全的范畴，此外还有业务虚机权限保护、可信根和隐私计算等。

### **DPU要融入IaaS等上层服务**

最重要的，DPU的加速方案，要能够把IaaS层的各类云服务融入DPU中。IaaS的主要服务有（用AWS举例）：

- 要融入各种类型的EC2云主机的支持，EC2的类型有：通用型、计算优化型、内存优化型、异构加速（GPU/FPGA/DSA）型、存储优化型。

- 要支持IaaS层网络类服务，如VPC、网关、防火墙服务、网络LB。

- 要支持本地存储，分布式存储EBS等。

- 等等。

## **2.2 几个常见的误解**

### **加速卡不是DPU**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

通常的加速卡是在协处理器的位置，一般只完成单个功能的加速。只做到了帮助CPU减负，但这远远不够。如果加速卡可以算DPU，那么GPU、FPGA和AI类的处理器，其功能远强大于普通加速卡，都算DPU了。

这样的定义肯定是不合适的。

### **SmartNIC不是DPU**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

SmartNIC也和DPU一样：在基处理器的位置，连接CPU和GPU的接口是PCIe，然后再通过网络接口连入数据中心网络。从物理外观上，两者是有一定相似性。可以说，DPU是从SmartNIC演进过来的，但演进之后，两者已经有了本质的不同：

- 智能网卡首先是聚焦网络加速，如果加入了存储相关的处理，为什么不能叫智能存储卡？当然还有加入了虚拟化、安全等其他功能的时候，又该怎么称呼呢？

- 另外，从名称可以看出，智能网卡依然是网卡，依然是一个I/O Device的定位。把自身当做基于CPU的整个计算机系统（计算机系统由CPU core、 内存和IO设备三部分组成）的一部分存在。而DPU的定位，则是把CPU和DPU当做是两个独立系统之间的交互协作。

这样大相径庭的定位，使得产品和架构定义会有非常大的差别。给用户的体验也会完全不同：

- 智能网卡只能做到帮助CPU减轻压力，但CPU侧软件仍要直面整个系统的复杂度；

- DPU要做到业务应用和基础设施分离，CPU侧的应用完全对底层系统无感。

### **PCIe Switch + NIC不是DPU**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

上图说的是，由于数据量大于计算量之后，整个计算的模式就从计算驱动变成了数据驱动。这样，DPU成为整个服务器架构的核心器件。

但是，如果把DPU简单理解成一个数据流的路由或交换则是不对的：

- 首先，这样的系统非常复杂，涉及到CPU、DSA、DPU和DPU多个独立芯片之间的完全显式的交互，谁来控制？就一个CPU+xPU的异构计算已经足够复杂了，还这么多处理芯片放一起，把复杂度更抬高了至少一个数量级。并且把复杂度度都交给软件，系统难以驾驭，也抬高了系统落地的门槛。

- 第二，DPU不仅仅是交换，不仅仅是设备接口，DPU本身是要承担很重要的任务加速。DPU作为CPU的集成加速平台，完成众多任务的卸载和加速。

- 第三，还需要考虑如何融入IaaS服务的问题，IaaS服务最终放在哪里？

- 第四，业务和基础设施隔离如何做？

### **存储控制器不是DPU**

DPU是数据中心IaaS层服务的承载平台，即使只关注存储类服务，存储控制器仍然无法承担DPU的作用。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

传统的观点认为，模块或层次之间的调用，是在模块内部封装复杂的功能，然后给外部提供简单的访问接口。但是，站在整个全局的角度，这就产生了如上图的问题。上图是以RocksDB为例的存储的整个系统栈，在这个系统栈里有三层虚拟化：SSD内部的FTL地址映射、系统层的文件系统、应用层软件的地址管理。三层虚拟化有点冗余，还会影响到存储的延迟。所以，行业开始流行ZNS存储，就是存储控制器只完成简单的控制，然后把SSD块的管理交给软件，实现软件定义存储。

如果只考虑个体，不考虑整体，会存在非常严重的问题：

- 会产生很多冗余和浪费；

- 如果个体的功能存在问题，会拖累整个系统。云计算需要高可用，一旦出问题可能只能“带病”运行，给运维和用户体验带来非常严重的影响。

存储控制器位置不对（DPU在服务器的入口处），定位也不对（云计算系统的发展，系统存储控制器更加简单稳定，而不是更加复杂）。

DPU要实现：

- （数据中心范围的）集中决策；

- （分散到各个服务器的）分布执行；

- （完全用户掌控的）软件定义；

- （接近ASIC极致性能的）硬件加速。

# **3 广义DPU：整体算力解决方案**

## **3.1 跳脱CPU的束缚**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

CPU、GPU和DPU，既相互协作，又相互竞争。按照互联网的法则：得入口者得天下。传统的观点认为，DPU是CPU的任务卸载加速。按照软硬件融合演进的观点：DPU/IPU成为数据中心算力和服务的核心，而独立CPU/GPU则是DPU的扩展。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

我们详细讲解一下：

- 小系统。DPU自身是包含CPU、GPU、FPGA、DSA、ASIC等各种处理引擎的一个超大的SOC。本身就能处理所有的任务。在一些业务应用层算力要求不高的情况下，最小计算系统的独立的DPU就能满足计算的要求。

- 中系统。DPU+CPU。在一些场景，业务应用层有更高的算力要求，或者必须业务和基础设施分离。这样，DPU+CPU的中等计算系统能够满足此类场景需求。

- 大系统。DPU+CPU+GPU。例如AI训练类的场景，例如一些应用需要加速的场景，并且需要业务和基础设施分离。这样的时候，DPU+CPU+DPU的最大就成为必须的选择。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

最后，上图是Intel认为的数据中心未来架构图，在这张图中，IPU成为数据中心中最关键的处理器：

- 后台的存储服务器和加速器池化服务器采用的是DPU小系统；

- 通用计算服务器采用的是DPU+CPU的中系统；

- AI服务器采用的是DPU+CPU+GPU的大系统。

## **3.2 DPU是更大号的SOC？**

当前，很多公司都号称做DPU，那是因为大家把DPU就简单的理解成完成主CPU一些加速任务的一个SOC而已。其实，这种认识还非常的初步。在这个概念下，DPU其实有点等同于ASIC或SOC的概念了。DPU可以作为一类产品，SOC肯定不能作为一类产品，SOC代表的是很多类产品的统称，甚至SOC可以等同于芯片的概念。

DPU的确算是SOC的一种，但DPU又跟传统SOC有很大的不同，如果把DPU认为是SOC，可能无法理解到DPU的本质。

站在系统的角度，传统SOC是单系统，而DPU是一个超异构宏系统，即多个系统整合到一起的大系统。传统SOC和超异构DPU的区别和联系：

- 单系统还是多系统。传统的SOC，有一个基于CPU的核心控制程序，来驱动CPU、GPU、外围其他模块以及接口数据IO等的工作，整个系统的运行是集中式管理和控制的。而超异构DPU由于其规模和复杂度，每个子系统其实就是一个传统SOC级别的系统，整个系统呈现出分布式的特点。

- 以计算为中心还是以数据为中心。传统SOC是计算为中心，CPU是由指令流（程序）来驱动运行的，然后CPU作为一切的“主管”再驱动外围的GPU、其他加速模块、IO模块运行。而在超异构的DPU系统中，由于数据处理带宽性能的影响，必须是以数据为中心，靠数据驱动计算。

## **3.3 软硬件融合视角的DPU四个层次**

DPU从给CPU减负而来，开始支撑CPU的工作，逐渐形成一个独立的计算平台，负担起绝大部分数据中心的计算算力（CPU负责更高价值的计算）。这样，DPU不是一个孤立的器件，需要和CPU、GPU联动，形成数据中心整体的算力解决方案。

DPU承担的事情越多，其功能也就需要越强大，其定位也就越来越不一样。可以把DPU的定位分为四个层次：

- 层次一，DPU是CPU的任务卸载/加速。CPU性能瓶颈。把网络、存储、虚拟化及安全等任务从CPU卸载到DPU加速，减轻CPU的压力。

- 层次二，DPU是基础设施，支撑上层应用。DPU成为集成加速平台，既完成基础设施层工作任务处理，也完成部分业务应用的加速，支撑CPU和GPU的应用层工作。

- 层次三，DPU/IPU是计算的核心。IaaS甚至PaaS、SaaS等云计算核心服务，融入到DPU软硬件。DPU图灵完备，并且是数据的入口。这使得DPU成为核心，而CPU和GPU成为扩展。

- 层次四。DPU/IPU的本质是超异构计算。算力持续提升，数据中心的超异构计算，DPU是核心承载。基于超异构的复杂计算，需要在极致灵活性的基础上，提供极致的性能。

## **3.4 DPU的本质：算力，并且必须是可驾驭的算力**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这里，我们给出“总算力”的概念：总算力 = 单位处理器算力 x 处理器数量。这个公式非常好理解，一方面是芯片本身的算力要高，另一方面，芯片要大规模采用。没有大规模落地的芯片，性能再高，都是浮云。既无法使整体算力提升，也由于单芯片的一次性成本太高使得其无法商业化落地。

数据中心当前的主力计算平台是CPU，这是因为“越是复杂的场景，对软件灵活性的要求越高”，而只有CPU能够提供云场景所需的灵活性。但很不幸的是，CPU已经达到了性能瓶颈。当前，云计算面临的基本矛盾是：CPU的性能，越来越无法满足上层软件的需要。

AI-DSA架构的处理器，目前还较少形成大规模的落地，最大的原因就在于其编程能力的欠缺。算法更新很快，业务逻辑也更新很快，而DSA架构的灵活性还是不能满足业务场景所需的快速演进。如果不能形成大规模落地，那么AI-DSA的价值就难以变现。

（独立ASIC更不用讨论，在复杂场景完全没法用。）

AI-DSA的难以落地，使得行业不得不进行回调，GPGPU越来越多的受到重视。GPGPU的性能，比CPU好，比DSA差；其灵活可编程能力，比DSA好，比CPU差。GPGPU能平衡好性能和灵活性，是一个相对均衡的处理器平台。但是，选择GPGPU只是逃避的问题，并没有本质的解决问题。GPGPU虽然相比CPU性能要好，但受限于架构的原因，也即将在未来3-5年达到性能瓶颈。而上层软件对算力的需求永无止境，这个问题如何本质解决？

业界需要全新的架构和解决方案，而DPU就成为新架构和解决方案的关键承载。这些方案至少要做到：

- 算力能相比DSA再持续增加，让整体算力的摩尔定律能够延续；

- 算力必须让用户可驾驭，需要足够灵活可编程能力给到用户。

至少要做到上述两点，才能提供高算力的同时，提供的是可驾驭的算力，才能真正实现总算力的持续显著提升。

阅读原文

阅读 3951

​

写留言

**留言 4**

- 杰克朱

  2022年1月18日

  赞1

  所以node产生了，目的是形成神经网络所需要的交换过程。

- 子虛

  2022年3月30日

  赞

  dpu承担这么多任务，这和之前的cpu为核心的架构有啥区别呢？感觉只是主角从cpu换成了dpu.

- gaoyf

  2022年1月19日

  赞

  控制面与业务面，基础设施与用户层面，算力的可分配，可重构，能否形成一个业务模型，根据模型划分算力资源。但是，DPU的可编程度也得发展，操作系统是否应该加强拓展异构，分布式等领域。

- Cloud Huang

  2022年1月18日

  赞

  拆分请求，旁路请求，分割请求/合并请求。好犀利

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/YHBSoNHqDiaH5V6ycE2TlM9McXhAbaV9aUBaTAibVhLibXH1rL4JTMb2eoQfdCicTm4wycOsopO82jG3szvYF3mTJw/300?wx_fmt=png&wxfrom=18)

Linux阅码场

19214

4

写留言

**留言 4**

- 杰克朱

  2022年1月18日

  赞1

  所以node产生了，目的是形成神经网络所需要的交换过程。

- 子虛

  2022年3月30日

  赞

  dpu承担这么多任务，这和之前的cpu为核心的架构有啥区别呢？感觉只是主角从cpu换成了dpu.

- gaoyf

  2022年1月19日

  赞

  控制面与业务面，基础设施与用户层面，算力的可分配，可重构，能否形成一个业务模型，根据模型划分算力资源。但是，DPU的可编程度也得发展，操作系统是否应该加强拓展异构，分布式等领域。

- Cloud Huang

  2022年1月18日

  赞

  拆分请求，旁路请求，分割请求/合并请求。好犀利

已无更多数据

# 

原创 逸帆 家恒 峥少等 美团技术团队

_2021年12月09日 20:01_

![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsUrXicw2VXTQTVVN5yxXWEacdY1ZdxTH195Pgibtib8EENJRMia3tzEnyVfgyfAgRibMssKqwlE186TLSw/640?wx_fmt=png&wxfrom=13&tp=wxpic)

**总第481\*\*\*\*篇**

**2021年 第051篇**

![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsVLR21NicmyQxcmiaqQ2KOJJj2JLwgJL4KSbo7CcuMF1hLf4xFjGQiaDRhSPyERxWGChWYP47Oc4sKGA/640?wx_fmt=png&wxfrom=13&tp=wxpic "undefined")

美团内部深度定制的TensorFlow版本，基于原生TensorFlow 1.x架构与接口，从大规模稀疏参数的支持、训练模式、分布式通信优化、流水线优化、算子优化融合等多维度进行了深度优化。在推荐系统场景中，分布式扩展性提升10倍以上，单位算力性能也有显著提升，并在美团内部业务中大量使用，本文介绍了相关的优化与实践工作。

- 1 背景

- 2 大规模训练优化挑战

- 2.1 业务迭代带来的挑战

- 2.2 系统负载分析

- 3 优化实践

- 3.1 大规模稀疏参数介绍

- 3.2 分布式负载均衡优化

- 3.3 通信优化

- 3.4 延迟优化

- 3.5 单实例PS并发优化

- 3.6 单位算力吞吐优化

- 4 大规模稀疏算法建模

- 5 总结与展望

## 1 背景

[TensorFlow](https://www.tensorflow.org/)（下文简称TF）是谷歌推出的一个开源深度学习框架，在美团推荐系统场景中得到了广泛的使用。但TensorFlow官方版本对工业级场景的支持，目前做得并不是特别的完善。美团在大规模生产落地的过程中，遇到了以下几方面的挑战：

- 所有参数都是用Variable表达， 对于百亿以上的稀疏参数开辟了大量的内存，造成了资源的浪费；

- 只支持百级别Worker的分布式扩展，对上千Worker的扩展性较差；

- 由于不支持大规模稀疏参数动态添加、删除，增量导出，导致无法支持Online Learning；

- 大规模集群运行时，会遇到慢机和宕机；由于框架层不能处理，导会致任务运行异常。

以上这些问题，并不是TensorFlow设计的问题，更多是底层实现的问题。考虑到美团大量业务的使用习惯以及社区的兼容性，我们基于原生TensorFlow 1.x架构与接口，从大规模稀疏参数的支持、训练模式、分布式通信优化、流水线优化、算子优化融合等多维度进行了深度定制，从而解决了该场景的核心痛点问题。

首先新系统在支持能力层面，目前可以做到千亿参数模型，上千Worker分布式训练的近线性加速，全年样本数据能够1天内完成训练，并支持Online Learning的能力。同时，新系统的各种架构和接口更加友好，美团内部包括美团外卖、美团优选、美团搜索、广告平台、大众点评Feeds等业务部门都在使用。**本文将重点介绍大规模分布式训练优化的工作**，希望对大家能够有所帮助或启发。

## 2 大规模训练优化挑战

### 2.1 业务迭代带来的挑战

随着美团业务的发展，推荐系统模型的规模和复杂度也在快速增长，具体表现如下：

- **训练数据**：训练样本从到百亿增长到千亿，增长了近10倍。

- **稀疏参数**：个数从几百到几千，也增长了近10倍；总参数量从几亿增长到百亿，增长了10~20倍。

- **模型复杂度**：越来越复杂，模型单步计算时间增长10倍以上。

对于大流量业务，一次训练实验，从几个小时增长到了几天，而此场景一次实验保持在1天之内是基本的需求。

### 2.2 系统负载分析

#### 2.2.1 问题分析工具链

TensorFlow是一个非常庞大的开源项目，代码有几百万行之多，原生系统的监控指标太粗，且不支持全局的监控，如果要定位一些复杂的性能瓶颈点，就比较困难。我们基于美团已经开源的监控系统CAT\[2\]，构建了TensorFlow的细粒度监控链路（如下图1所示），可以精准定位到性能的瓶颈问题。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图1 TensorFlow PS架构全链路监控

同时，在性能优化的过程中，会涉及到大量的性能测试和结果分析，这也是一个非常耗费人力的工作。我们抽象了一套自动化的实验框架（如下图2所示），可以自动化、多轮次地进行实验，并自动采集各类监控指标，然后生成报告。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图2 自动化实验框架

#### 2.2.2 业务视角的负载分析

在推荐系统场景中，我们使用了TensorFlow Parameter Server\[3\]（简称PS）异步训练模式来支持业务分布式训练需求。对于这套架构，上述的业务变化会带来什么样的负载变化？如下图3所示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图3 TensorFlow PS架构大规模训练负载分析

总结来看，主要包括通信压力、PS并发压力、Worker计算压力。对于分布式系统来说，通常是通过横向扩展来解决负载问题。虽然看来起可以解决问题，但从实验结果来看，当PS扩展到一定数量后，单步训练时间反而会增加，如下图4所示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图4 扩展PS提升训练性能实验

导致这种结果的核心原因是：Worker单步训练需要和所有的PS通信同步完成，每增加1个PS要增加N条通信链路，这大大增加了链路延迟（如下图5所示）。而一次训练要执行上百万、上千万步训练。最终导致**链路延迟超过了加PS算力并发的收益**。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图5 增加PS带来的链路开销

而对于这个系统，优化的核心难点在于：**如何在有限的PS实例下，进行分布式计算的优化**。

## 3 优化实践

### 3.1 大规模稀疏参数介绍

对于推荐系统模型，绝大多数参数都是稀疏参数，而对稀疏参数来说有一个非常重要的操作是Embedding，这个操作通常也是负载最重的，也是后续优化的重点。由于我们对稀疏参数进行了重新定义，后续的优化也基于此之上，所以我们先介绍一下这部分的工作。

在原生的TensorFlow中构建Embedding模块，用户需要首先创建一个足够装得下所有稀疏参数的Variable，然后在这个Variable上进行Embedding的学习。然而，使用Variable来进行Embedding训练存在很多弊端：

- Variable的大小必须提前设定好，对于百亿千亿的场景，该设定会带来巨大的空间浪费；

- 训练速度慢，无法针对稀疏模型进行定制优化。

我们首先解决了有无的问题，使用HashTable来替代Variable，将稀疏特征ID作为Key，Embedding向量作为Value。相比原生使用Variable进行Embedding的方式，具备以下的优势：

1. HashTable的大小可以在训练过程中自动伸缩，避免了开辟冗余的存储空间，同时用户无需关注申请大小，从而降低了使用成本。

1. 针对HashTable方案实施了一系列定制优化，训练速度相比Variable有了很大的提高，可以进行千亿规模模型的训练，扩展性较好。

1. 得益于稀疏参数的动态伸缩，我们在此基础上支持了Online Learning。

1. API设计上保持与社区版本兼容，在使用上几乎与原生Variable一致，对接成本极低。

简化版的基于PS架构的实现示意如下图6所示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图6 支撑大规模稀疏参数的HashTable方案

核心流程大致可以分为以下几步：

1. 稀疏特征ID（通常我们会提前完成统一编码的工作）进入Embedding模块，借助TensorFlow搭建的Send-Recv机制，这些稀疏特征ID被拉取到PS端，PS端上的Lookup等算子会实际从底层HashTable中查询并组装Embedding向量。

1. 上述Embedding向量被Worker拉回进行后续训练，并通过反向传播计算出这部分参数的梯度，这些梯度进一步被位于PS端的优化器拉回。

1. PS端的优化器首先调用Find算子，从HashTable获取到梯度对应的原始稀疏参数向量和相应的优化器参数，最终通过优化算法，完成对Embedding向量和优化器参数的更新计算，再通过Insert算子插入HashTable中。

### 3.2 分布式负载均衡优化

这部分优化，是分布式计算的经典优化方向。PS架构是一个典型的“水桶模型”，为了完成一步训练，Worker端需要和所有PS完成交互，因此PS之间的平衡就显得非常重要。但是在实践中，我们发现多个PS的耗时并不均衡，其中的原因，既包括TensorFlow PS架构简单的切图逻辑（Round-Robin）带来的负载不均衡，也有异构机器导致的不均衡。

对于推荐模型来说，我们的主要优化策略是，把所有稀疏参数和大的稠密参数自动、均匀的切分到每个PS上，可以解决大多数这类问题。而在实践过程中，我们也发现一个比较难排查的问题：原生Adam优化器，实现导致PS负载不均衡。下面会详细介绍一下。

在Adam优化器中，它的参数优化过程需要两个β参与计算，在原生TensorFlow的实现中，这两个β是所有需要此优化器进行优化的Variabl（或HashTable）所共享的，并且会与第一个Variable（名字字典序）落在同一个PS上面，这会带来一个问题：每个优化器只拥有一个β和一个β，且仅位于某个PS上。因此，在参数优化的过程中，该PS会承受远高于其他PS的请求，从而导致该PS成为性能瓶颈。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图7 Adam优化算法

但是通过观察Adam的优化算法，我们可以看到β和β都是常量，且蓝色高亮的部分都是相对独立的计算过程，各个PS之间可以独立完成。基于这样的发现，优化的方法也就非常直观了，我们为每一个PS上的Adam优化器冗余创建了β参数，并在本地计算t和alpha值，去除了因此负载不均导致的PS热点问题。

该优化所带来的提升具备普适性且效果明显，在美团内部某业务模型上，通过β热点去除可以带来9%左右的性能提升。此外，由于摆脱了对β的全局依赖，该优化还能提高PS架构的可扩展性，在扩增Worker数量的时候相比之前会带来更好的加速比。

### 3.3 通信优化

通过2.2章节的分析可知，系统的通信压力也非常大，我们主要基于RDMA做了通信优化的工作。首先简单介绍一下RDMA，相比较于传统基于套接字TCP/IP协议栈的通信过程，RDMA具有零拷贝、内核旁路的优势，不仅降低了网络的延迟，同时也降低了CPU的占用率，RDMA更适合深度学习模型的相关通信过程。

RDMA主要包括三种协议Infiniband、RoCE(V1, V2)、iWARP。在美团内部的深度学习场景中，RDMA通信协议使用的是RoCE V2协议。目前在深度学习训练领域，尤其是在稠密模型训练场景（NLP、CV等），RDMA已经是大规模分布式训练的标配。然而，在大规模稀疏模型的训练中，开源系统对于RDMA的支持非常有限，TensorFlow Verbs\[4\]通信模块已经很长时间没有更新了，通信效果也并不理想，我们基于此之上进行了很多的改进工作。

经过优化后的版本，在1TB Click Logs\[5\]公开数据集、DLRM\[6\]模型、100个Worker以上的训练，性能提升了20%~40%。在美团的多个业务模型上，对比TensorFlow Seastar\[7\]改造的通信层实现也有10%~60%的速度提升。同时也把我们的工作回馈给了[社区](https://github.com/tensorflow/networking/pull/38)。

#### 3.3.1 Memory Registration优化

RDMA有三种数据传输的方式SEND/RECV、WRITE、READ，其中WRITE、READ类似于数据发送方直接在远程Memory进行读写，Receiver无法感知，WRITE和READ适用于批量数据传输。在TensorFlow内部，基于RDMA的数据传输方式使用的是WRITE单边通信模式。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图8 RDMA传输方式

在RDMA传输数据时，需要提前开辟内存空间并将其注册到网卡设备上（Memory Registration过程，下称MR），使得这片空间可以被网卡直接操作。开辟新的内存并注册到设备上，整个过程是比较耗时的。下图9展示了不同大小的内存绑定到网卡设备上的耗时，可以看到随着注册内存的增大，绑定MR的耗时迅速增加。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图9 MR过程开销

社区版Tensorflow RDMA实现，Tensor创建依旧沿用了统一的BFC Allocator，并将所有创建的Tensor都注册到MR上。正如上面所提到的，MR的注册绑定具有性能开销，高频、大空间的MR注册会带来显著的性能下降。而训练过程中的Tensor，只有那些涉及到跨节点通信的Tensor有必要进行MR，其余Tensor并不需要注册到MR。因此，优化的方法也就比较直接了，我们识别并管理那些通信Tensor，仅对这些跨节点通信的Tensor进行MR注册就好了。

#### 3.3.2 RDMA静态分配器

RDMA静态分配器是上一个MR注册优化的延伸。通过Memory Registration优化，去除非传输Tensor的MR注册，我们降低了MR注册数量。但是在稀疏场景大规模的训练下，并行训练的Worker常有几百上千个，这会带来新的问题：

- PS架构中的PS和Worker互为Client-Server，这里以PS端为例，当Worker数目增加到上千个时，Worker数目的增多，造成PS端MR注册频次还是非常高，增加了内存分配注册的耗时。

- 由于稀疏场景不同Step之间同一个算子输出Tensor的形状可能发生变化，导致了创建的MR可复用性较差，带来了较高的内存碎片和重复注册MR开销。

针对上面的问题，我们引入了MR静态分配器的策略。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图10 MR静态分配器

这里核心的设计思路为：

1. 虽然稀疏场景同一个算子输出Tensor的Shape存在变化的可能，但是整体变化幅度可控，通过监控与分析，是可以找到一个较为稳定的内存大小，满足多Step间Tensor的存储需求。

1. 基于上面的信息，我们修改了原有逐Tensor(Request)的MR申请策略，通过一次性预申请一块较大的空间并注册到网卡端，后续通过自己维护的分配策略进行空间的分配，大大降低了MR申请的频率，绝大多数情况下，训练全过程中只需要一次MR注册申请即可。

1. 我们引入了一种简单的交换协议，将传输Tensor的Shape，Data打包到一起，写到Client端。Client端根据协议，解析出Tensor大小，并最终读取Data，避免了原生实现中因Tensor的Shape变化而产生的多次协商过程。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图11 MR静态分配器构造流程

具体到实现中，我们引入了Allocation Analysis模块，在训练开始的一段时间，我们会对分配的历史数据进行分析，以得到一个实际预开辟MR大小以及各个Tensor的预留空间大小。然后我们会暂停训练的进程，启动Allocator的构造过程，包括MR的创建以及通信双端的信息同步。利用相关信息构造MR Info Map，这个Map的Key是传输Tensor的唯一标记（ParsedKey，计算图切图时确定），Info结构体中包含了本地地址指针、offset大小、ibv_send_wr相关信息等。然后恢复训练，后续Tensor的传输就可以使用静态开辟好的MR进行收发，也免去了因Shape变化而产生的多次协商过程。

#### 3.3.3 Multi RequestBuffer与CQ负载均衡

TensorFlow社区版的RDMA通信过程，不仅仅包含上面Tensor数据的发送和接收过程，还包括传输相关的控制消息的发送和接收过程，控制消息的发送和接收过程同样是使用了ibv_post_send和ibv_post_recv原语。原生的控制流实现存在一些瓶颈，在大规模训练时会限制控制流的吞吐，进而影响数据收发的效率。具体体现在：

- 请求的发送通过同一片RequestBuffer内存进行写出，多个Client的请求均依赖这一片Buffer，也就导致到控制流信息实际是串行发送的，只有等到对端的Ack信息后，才可以下一个Request的写出，限制了请求的发送吞吐。

- 在Client端需要轮询RDMA Completion Queue来获得请求的到达，以及相关状态的变更。原生实现仅有一个Completion Queue，单线程进行轮询处理，在大规模分布式训练中，限制了应答的效率。

针对上面的问题，我们采用了Multi RequestBuffer与CQ负载均衡优化，破除了在请求发送和请求应答环节可能存在的吞吐瓶颈。

#### 3.3.4 Send-Driven & Rendezvous-Bypass

对于Tensorflow PS架构熟悉的同学会了解，一整张计算图被切割为Worker端和PS端后，为了使两张计算图能够彼此交换数据，建立了基于Rendezvous（汇合点）机制的异步数据交换模式。如下图12所示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图12 TensoFlow切图之Send-Recv对添加

基于上图的切图逻辑，Recv算子代表着这一侧计算图有Tensor的需求，而Tensor的生产者则位于与之配对的另一设备上的Send算子背后。

在具体实现上，Tensorflow实现了Recv-Driven的数据交换模式，如上图所示，位于DeviceA和DeviceB的两张计算图会异步并发的执行，位于DeviceB的Recv执行时会发起一条RPC请求发往DeviceA，DeviceA收到请求后，会将请求路由到Rendezvous中，如果在当中发现所需要的数据已经生产好，并被Send算子注册了进来，那么就地获取数据，返回给DeviceB；如果此时数据还没有生产好，则将来自于DeviceB的Recv请求注册在Rendezvous中，等待后续DeviceA生产好后，由Send算子发送过来，找到注册的Recv，触发回调，返回数据给DeviceB。

我们看到，汇合点机制优雅地解决了生产者消费者节奏不同情况下数据交换的问题。不过Recv-Driven的模式也引入了两个潜在的问题：

- 据我们的观察，在实际业务模型中，在Rendezvous中Recv算子等待Send算子的比例和Send算子等待Recv算子的比例相当，也就是说对于Send等到Recv的数据，在Send准备好的那一刹那就可以发给对端，但是由于机制实现问题，还是等待Recv算子过来，才将数据拉取回去，通信过程耗时较长。

- Rendezvous作为一个数据交换的热点，它内部的逻辑开销并不低。

针对上面提到的问题，我们在RDMA上实现了另外一种数据交换的模式，叫做Send-Driven模式。与Recv-Driven模式相对，顾名思义就是有Send算子直接将数据写到Recv端，Recv端接收数据并注册到本地Rendezvous中，Recv算子直接从本地的Rendezvous中获取数据。具体流程如下图13所示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图13 原生的Recv-Driven与补充的Send-Driven机制

从图中可以看到，相较于Recv-Driven模式，Send-Driven模式的通信流程得到了比较大的简化，另外在数据ready后立即发送的特性，跳过了一侧的Rendezvous，并且对于生产者先于消费者的情况，可以加快消费端数据获取的速度。

### 3.4 延迟优化

这部分优化，也是分布式计算的经典优化方向。整个流程链路上那些可以精简、合并、重叠需要不断去挖掘。对于机器学习系统来说，相比其它的系统，还可以用一些近似的算法来做这部分工作，从而获得较大的性能提升。下面介绍我们在两个这方面做的一些优化实践。

#### 3.4.1 稀疏域参数聚合

在启用HashTable存储稀疏参数后，对应的，一些配套参数也需要替换为HashTable实现，这样整个计算图中会出现多张HashTable以及大量的相关算子。在实践中，我们发现需要尽量降低Lookup/Insert等算子的个数，一方面降低PS的负载，一方面降低RPC QPS。因此，针对稀疏模型的常见用法，我们进行了相关的聚合工作。

以Adam优化器为例，需要创建两个slot，以保存优化中的动量信息，它的Shape与Embedding相同。在原生优化器中，这两个Variable是单独创建的，并在反向梯度更新的时候会去读写。同理，使用HashTable方案时，我们需要同时创建两张单独的HashTable用来训练m、v参数。那么在前向，反向中需要分别对Embedding、 m、v进行一次Lookup和一次Insert，总共需要三次Lookup和三次Insert。

这里一个优化点就是将Embedding、 m、v，以及低频过滤的计数器（见下图14的Counting HashTable）聚合到一起，作为HashTable的Value，这样对稀疏参数的相关操作就可以聚合执行，大大减少了稀疏参数操作频次，降低了PS的压力。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图14 基于HashTable的参数融合策略

该特性属于一个普适型优化，开启聚合功能后，训练速度有了显著的提高，性能提升幅度随着模型和Worker规模的变化，效果总是正向的。在美团内部真实业务模型上，聚合之后性能相比非聚合方式能提升了45%左右。

#### 3.4.2 Embedding流水线优化

流水线，在工业生产中，指每一个生产单位只专注处理某个片段的工作，以提高工作效率及产量的一种生产方式。在计算机领域内，更为大家熟知的是，流水线代表一种多任务之间Overlap执行的并行化技术。例如在典型的RISC处理器中，用户的程序由大量指令构成，而一条指令的执行又可以大致分为：取指、译码、执行、访存、写回等环节。这些环节会利用到指令Cache、数据Cache、寄存器、ALU等多种不同的硬件单元，在每一个指令周期内，这5个环节的硬件单元会并行执行，得以更加充分的利用硬件能力，以此提高整个处理器的指令吞吐性能。处理器的指令流水线是一套复杂而系统的底层技术，但其中的思想在分布式深度学习框架中也被大量的使用，例如：

- 如果将分布式训练简单的抽象为计算和通信两个过程，绝大多数主流的深度学习框架都支持在执行计算图DAG时，通信和计算的Overlap。

- 如果将深度模型训练简单的分为前向和反向，在单步内，由于两者的强依赖性，无法做到有效并行，字节BytePS\[8\]中引入的通信调度打破了step iteration间的屏障，上一轮的部分参数更新完毕后，即可提前开始下轮的前向计算，增强了整体视角下前反向的Overlap。

- 百度AIBox\[9\]为了解决CTR场景GPU训练时，参数位于主存，但计算位于GPU的问题，巧妙调度不同硬件设备，搭建起了主要利用CPU/主存/网卡的参数预准备阶段和主要利用GPU/NVLink的网络计算阶段，通过两个阶段的Overlap达到更高的训练吞吐。

我们看到，在深度学习框架设计上，通过分析场景，可以从不同的视角发掘可并行的阶段，来提高整体的训练吞吐。

对于大规模稀疏模型训练时，核心模型流程是：先执行稀疏参数的Embedding，然后执行稠密部分子网络。其中稀疏参数Embedding在远端PS上执行，主要耗费网络资源，而稠密部分子网络在本地Worker执行，主要耗费计算资源。这两部分占了整个流程的大部分时间，在美团某实际业务模型上分别耗时占比：40%+、50%+。

那我们是否可以提前执行稀疏参数的Embedding，来做到通信和计算的Overlap，隐藏掉这部分时间呢？从系统实现上肯定是可行的，但从算法上讲，这样做会引入参数Staleness的问题，可能会导致模型精度受到影响。但在实际的生产场景中，大规模异步训练时本身就会带来几十到几百个步的滞后性问题。经过我们测试，提前获取一两步的稀疏参数，模型精度并未受到影响。

在具体实现上，我们把整个计算图拆分为Embedding Graph（EG）和Main Graph（MG）两张子图，两者异步独立执行，做到拆分流程的Overlap（整个拆分过程，可以做到对用户透明）。EG主要覆盖从样本中抽取Embedding Key，查询组装Embedding向量，Embedding向量更新等环节。MG主要包含稠密部分子网络计算、梯度计算、稠密参数部分更新等环节。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图15 Embedding流水线模块交互关系

两张子图的交互关系为：EG向MG传递Embeding向量（从MG的视角看，是从一个稠密Variable读取数值）；MG向EG传递Embedding参数对应的梯度。上述两个过程的表达都是TensorFlow的计算图，我们利用两个线程，两个Session并发的执行两张计算图，使得两个阶段Overlap起来，以此到达了更大的训练吞吐。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图16 Embedding流水线架构流程图

上图是Embedding流水线的架构流程图。直观来看分为左侧的样本分发模块，顶部的跨Session数据交换模块，以及自动图切分得到的Embedding Graph和Main Graph，蓝色的圆圈代表新增算子，橙色箭头代表EG重点流程，蓝色箭头代表MG重点流程，红色箭头代表样本数据重点流程。

1. 以对用户透明的形式引入了一层名为Pipeline Dataset的抽象层，这一层的产生是为了满足EG/MG两张计算图以不同节奏运行的需求，支持自定义配置。另外，为了使得整个流水线中的数据做到彼此的配套，这里还会负责进行一个全局Batch ID的生成及注册工作。Pipeline Dataset对外暴露两种Iterator，一个供EG使用，一个供MG使用。Pipeline Dataset底部共享TensorFlow原生的各层Dataset。

1. 顶部的ExchangeManager是一个静态的，跨Session的数据交换媒介，对外暴露数据注册和数据拉取的能力。抽象这个模块的原因是，EG和MG原本归属于一张计算图，因为流水线的原因拆解为拆为两张图，这样我们需要建立一种跨Session的数据交换机制，并准确进行配套。它内部以全局Batch ID做Key，后面管理了样本数据、Embeding向量、Embedding梯度、Unique后的Index等数据，并负责这些数据的生命周期管理。

1. 中间的Embedding Graph由独立的TF Session运行于一个独立的线程中，通过a算子获得样本数据后，进行特征ID的抽取等动作，并进行基于HashTable方法的稀疏参数查询，查询结果通过c算子放置到ExchangeManager中。EG中还包含用于反向更新的f算子，它会从ExchangeManager中获取Embedding梯度和与其配套的前向参数，然后执行梯度更新参数逻辑。

1. 下面的Main Graph负责实际稠密子网络的计算，我们继承并实现一种可训练的EmbeddingVariable，它的构建过程（d算子）会从ExchangeManager查找与自己配套的Embedding向量封装成EmbeddingVariable，给稠密子网络。此外，在EmbeddingVariable注册的反向方法中，我们添加了e算子使得Embedding梯度得以添加到ExchangeManager中，供EG中的f算子消费。

通过上面的设计，我们就搭建起了一套可控的EG/MG并发流水线训练模式。总体来看，Embedding流水线训练模式的收益来源有：

- 经过我们对多个业务模型的Profiling分析发现，EG和MG在时间的比例上在3:7或4:6的左右，通过将这两个阶段并行起来，可以有效的隐藏Embedding阶段，使得MG网络计算部分几乎总是可以立即开始，大大加速了整体模型的训练吞吐。

- TensorFlow引擎中当使用多个优化器（稀疏与非稀疏）的时候，会出现重复构建反向计算图的问题，一定程度增加了额外计算，通过两张子图的拆分，恰好避免了这个问题。

- 在实施过程中的ExchangeManager不仅负责了Embedding参数和梯度的交换，还承担了元数据复用管理的职责。例如Unique等算子的结果保存，进一步降低了重复计算。

另外，在API设计上，我们做到了对用户透明，仅需一行代码即可开启Embedding流水线功能，对用户隐藏了EG/MG的切割过程。目前，在美团某业务训练中，Embedding流水线功能在CPU PS架构下可以带来20%~60%的性能提升（而且Worker并发规模越大，性能越好）。

### 3.5 单实例PS并发优化

经过2.2章节的分析可知，我们不能通过持续扩PS来提升分布式任务的吞吐，单实例PS的并发优化，也是非常重要的优化方向。我们主要的优化工作如下。

#### 3.5.1 高性能的HashTable

PS架构下，大规模稀疏模型训练对于HashTable的并发读写要求很高，因为每个PS都要承担成百乃至上千个Worker的Embedding压力，这里我们综合速度和稳定性考虑，选用了tbb::concurrent_hash_map\[10\]作为底层HashTable表实现，并将其包装成一个新的TBBConcurrentHashTable算子。经过测试，在千亿规模下TBBConcurrentHashTable比原生MutableDenseHashTable训练速度上快了3倍。

#### 3.5.2 HashTable BucketPool

对于大规模稀疏模型训练来说，Embedding HashTable会面对大量的并发操作，通过Profiling我们发现，频繁动态的内存申请会带来了较大性能开销（即使TensorFlow的Tensor有专门的内存分配器）。我们基于内存池化的思路优化了HashTable的内存管理。

我们在HashTable初始化时，会先为Key和Value分别创造两个BucketPool，每个池子都会先Malloc较大一块内存备用，考虑到可能会有对HashTable进行中的Key和Value进行Remove的场景（如Online Learning训练时），需要对从HashTable中删除的Key和Value所使用的内存进行回收，因此每个BucketPool还有一个ReuseQueue来负责维护回收的内存。每次向内部的哈希表数据结构中Insert Key和Value的时候，Key和Value内存和释放分配都进行池化管理。用这种方式降低了大规模稀疏训练中遇到稀疏内存分配开销，整体端到端训练性能提升了5%左右。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图17 HashTable内存优化

### 3.6 单位算力吞吐优化

经过2.2章节的分析，Worker的计算压力也非常大，如果不优化Worker，同时要保持吞吐，需要横向扩展更多的Worker，给PS带来更大的压力。而对于用户来说，如果能在有限的计算资源下带来性能提升，对业务价值更高。我们通过CAT统计出了一些高频算子，并进行了专项优化。这里选取Unique&DynamicPartition算子融合案例进行分享。

在TensorFlow PS架构中，包括Embedding向量在内的共享参数都存储在PS上，并通过网络与Worker交互，在进行Embedding查询过程中，往往会涉及如下两个环节：

- 由于稀疏参数的性质，从样本中抽取得到的待查询Embedding ID，它的重复率往往高达70%~90%，如果不进行去重查询，不论是对HashTable的查询还是网络的传输，都会带来不小的压力。因此，通常会在查询前进行Unique操作。

- 在大规模稀疏场景中，为了存储千亿规模的参数，会有多个PS机器共同承载。而Worker端会负责对查询请求按照设定的路由规则进行切分，这里通常会在查询前进行DynamicPartition动作。

通常这两个过程会利用TensorFlow既有的算子进行搭建，但在实际使用中，我们发现它并不是很高效，主要问题在于：

- Unique算子原生实现，它内部使用的内存分配策略较为低效。使用了两倍输入参数（Embedding ID）的大小进行内存分配，但由于输入参数较大，而且重复率高，导致HashTable创建过大且非常稀疏。几乎每次插入都会产生一次minor_page_fault，导致HashTable性能下降。我们使用Intel Vtune验证了这一点（参见图18）。

- Unique和Dynamic Partition算子存在冗余数据遍历，这些操作其实可以在一次数据遍历中全部做完，节省掉算子切换、冗余数据遍历的耗时。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图18 Unique算子内部出现DRAM Bound问题

总结来说，HashTable开辟过大会导致大量的minor_page_fault，导致访存的时间增加，HashTable过小又可能会导致扩容。我们采用了基于启发式算法的内存自适应Unique算子实现，通过对训练历史重复率的统计，我们可以得到一个相对合理的HashTable大小，来提高访存的性能；另外Unique算子内HashTable的具体选择上，经过我们的多种测试，选择了Robin HashTable替换了原生TF中的实现。

进一步，我们对围绕Embedding ID的Unique和Partition环节进行了算子合并，简化了逻辑实现。经过上述的优化，Unique单算子可以取得51%的加速，在真实模型端到端上可以获得10%左右的性能提升，算子总数量降低了4%。

**在整个关键算子优化的过程中，Intel公司的林立凡、张向泽、高明进行大量的技术支持，我们也复用了他们的部分优化工作，在此深表感谢！**

## 4 大规模稀疏算法建模

大规模稀疏能力在业务落地的过程中，算法层面还需要从特征和模型结构上进行对应升级，才能拿到非常好的效果。其中外卖广告从业务特点出发，引入大规模稀疏特征完成外卖场景下特征体系的升级，提供了更高维的特征空间和参数空间，增强了模型的拟合能力。重新设计了面向高维稀疏场景的特征编码方案，解决了特征编码过程中的特征冲突问题，同时编码过程去掉了部分冗余的特征哈希操作，一定程度上简化了特征处理逻辑，并降低了特征计算的耗时。

在系统层面，面对百亿参数、百亿样本以上量级的大规模稀疏模型的训练，会带来训练迭代效率的大大降低，单次实验从一天以内，增长到一周左右。美团机器学习平台训练引擎团队，除了上述TensorFlow框架层面的优化、还针对业务模型进行了专项优化，整体吞吐优化了8到10倍（如果投入更多计算资源，可以进一步加速），大大提升业务的迭代效率，助力外卖广告业务取得了较为明显的提升。

## 5 总结与展望

TensorFlow在大规模推荐系统中被广泛使用，但由于缺乏大规模稀疏的大规模分布式训练能力，阻碍了业务的发展。美团基于TensorFlow原生架构，支持了大规模稀疏能力，并从多个角度进行了深度优化，做到千亿参数、千亿样本高效的分布式训练，并在美团内部进行了大规模的使用。对于这类关键能力的缺失，TensorFlow社区也引起了共鸣，社区官方在2020年创建了SIG Recommenders\[11\]，通过社区共建的方式来解决此类问题，美团后续也会积极的参与到社区的贡献当中去。

美团推荐系统场景的模型训练，目前主要运行在CPU上，但随着业务的发展，有些模型变得越来越复杂，CPU上已经很难有优化空间（优化后的Worker CPU使用率在90%以上）。而近几年，GPU的计算能力突飞猛进，新一代的NVIDIA A100 GPU，算力达到了156TFLOPS（TF32 Tensor Cores）、80G显存、卡间带宽600GB/s。对于这类复杂模型的Workload，我们基于A100 GPU架构，设计了下一代的分布式训练架构，经过初步优化，在美团某大流量业务推荐模型上也拿到了较好的效果，目前还在进一步优化当中，后续我们会进行分享，敬请期待。

## 6 作者简介

逸帆、家恒、峥少、鹏鹏、永宇、正阳、黄军等，来自美团基础研发平台，机器学习平台训练引擎组，主要负责美团分布式机器学习训练系统的性能优化与能力建设。

海涛，来自美团外卖广告策略团队，主要负责美团外卖广告业务的算法探索和策略落地工作。

## 7 参考文献

\[1\] [https://www.usenix.org/system/files/conference/osdi16/osdi16-abadi.pdf](https://www.usenix.org/system/files/conference/osdi16/osdi16-abadi.pdf)

\[2\] [https://github.com/dianping/cat](https://github.com/dianping/cat)

\[3\] [https://www.usenix.org/system/files/conference/osdi14/osdi14-paper-li_mu.pdf](https://www.usenix.org/system/files/conference/osdi14/osdi14-paper-li_mu.pdf)

\[4\] [https://github.com/tensorflow/networking/tree/master/tensorflow_networking/verbs](https://github.com/tensorflow/networking/tree/master/tensorflow_networking/verbs)

\[5\] [https://labs.criteo.com/2013/12/download-terabyte-click-logs/](https://labs.criteo.com/2013/12/download-terabyte-click-logs/)

\[6\] [https://arxiv.org/abs/1906.00091](https://arxiv.org/abs/1906.00091)

\[7\] [https://github.com/tensorflow/networking/tree/master/tensorflow_networking/seastar](https://github.com/tensorflow/networking/tree/master/tensorflow_networking/seastar)

\[8\] [https://github.com/bytedance/byteps](https://github.com/bytedance/byteps)

\[9\] [http://research.baidu.com/Public/uploads/5e18a1017a7a0.pdf](http://research.baidu.com/Public/uploads/5e18a1017a7a0.pdf)

\[10\] [https://github.com/oneapi-src/oneTBB](https://github.com/oneapi-src/oneTBB)

\[11\] [https://github.com/tensorflow/recommenders-addons](https://github.com/tensorflow/recommenders-addons)

----------  END  ----------

**招聘信息**

美团机器学习平台大量岗位持续招聘中，社招/校招均可（欢迎投递我们的校招北斗岗位：美团机器学习平台基础架构），坐标北京/上海，构建多领域的公司级机器学习平台，帮大家吃得更好，生活更好。简历可投递至：huangjun03@meituan.com。

**也许你还想看**

**| [](http://mp.weixin.qq.com/s?__biz=MjM5NjQ5MTI5OA==&mid=2651751331&idx=1&sn=d917732980c4b7e91eb79e050edf5bd7&chksm=bd125aee8a65d3f84ed5400749ac0fd296dad70eaba658f81b8bcfda3702bc05dce1c61c3a7f&scene=21#wechat_redirect)** [一站式机器学习平台建设实践](http://mp.weixin.qq.com/s?__biz=MjM5NjQ5MTI5OA==&mid=2651751331&idx=1&sn=d917732980c4b7e91eb79e050edf5bd7&chksm=bd125aee8a65d3f84ed5400749ac0fd296dad70eaba658f81b8bcfda3702bc05dce1c61c3a7f&scene=21#wechat_redirect)**[](http://mp.weixin.qq.com/s?__biz=MjM5NjQ5MTI5OA==&mid=2651751331&idx=1&sn=d917732980c4b7e91eb79e050edf5bd7&chksm=bd125aee8a65d3f84ed5400749ac0fd296dad70eaba658f81b8bcfda3702bc05dce1c61c3a7f&scene=21#wechat_redirect)**

**|** [美团深度学习系统的工程实践](http://mp.weixin.qq.com/s?__biz=MjM5NjQ5MTI5OA==&mid=2651749161&idx=1&sn=2a34bc1e36fbf259ac63ffe7c94f528b&chksm=bd12a2648a652b72a9027a572a0038d3b8a6448949c53e7aefab617a12811da84efdc88bcd29&scene=21#wechat_redirect)

**|** [基于TensorFlow Serving的深度学习在线预估](http://mp.weixin.qq.com/s?__biz=MjM5NjQ5MTI5OA==&mid=2651748960&idx=2&sn=4c637290b0bd35dc5b541d01d76ce574&chksm=bd12a32d8a652a3b24e159f352c835fd7ff4eedead31dd5b0d0ae2c21da4a0fc587f69a99aef&scene=21#wechat_redirect)

**阅读更多**

______________________________________________________________________

[前端](https://t.1yb.co/jo7r) **|**  [](https://t.1yb.co/jo7v)[算法](https://t.1yb.co/jsdG) **|** [后端](https://t.1yb.co/jsWK) **|** [数据](https://t.1yb.co/jqRZ)

[安全](https://t.1yb.co/jo7v) **|** [Android](https://t.1yb.co/jui4) **|** [iOS](https://t.1yb.co/jtXE)  **|** [运维](https://t.1yb.co/jo7K) **|** [测试](https://t.1yb.co/jtsX)

![](https://res.wx.qq.com/t/fed_upload/b39ef69e-c4d6-4169-8612-5f00a84860e7/wx-avatar-default.svg)

公众号

机器学习14

TensorFlow2

后台38

基础研发平台31

架构5

机器学习 · 目录

上一篇细粒度情感分析在到餐场景中的应用下一篇GPU在外卖场景精排模型预估中的应用实践

阅读 1.8万

​

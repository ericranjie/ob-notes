
Original 沈典、杨彬 酷玩BPF

 _2024年08月26日 08:30_ _广西_

# 1、基于eBPF实现网络功能的优势

目前，基于eBPF实现的网络功能已经被很多公司应用于生产环境中，成为云环境下基础设施的重要组成部分。例如Meta的负载均衡器Katran, Google Cloud目前使用基于eBPF的网络数据平面等。在学术研究和开源社区中，eBPF也被广泛地用来实现网络功能。典型的例子有，学术研究: BMC (NSDI 2021), SPRIGHT (SIGCOMM 2023), Morpheus (ASPLOS 2022), Electrode (NSDI 2023), DINT (NSDI 2024)  等；开源项目: CIlium, PolyCube, Katran等。因此基于eBPF实现网络功能逐渐成为一种趋势。

这是因为eBPF拥有以下优势：

1. eBPF作为一种起源于内核的技术，能够很好地集成到依赖于内核的云生态中。例如，根据OvS团队的论文，在主机内部容器通信中，内核XDP数据路径的性能优于OvS中的DPDK数据路径。
    
2. 相比于DPDK等方案，eBPF实现了更好的性能和CPU利用率、安全、隔离性、运维成本之间的平衡。例如，eBPF支持高性能的数据包处理而不会使CPU饱和，使得网络功能和非网络功能应用能够在同一设备上运行。
    
3. eBPF允许动态加载用户代码然后安全地在内核中执行，无需修改内核源代码，从而提高了可维护性和灵活性，并加快了网络功能的开发和部署。
    

# 2、eBPF实现网络功能面临的技术挑战

## 2.1 用eBPF无法实现特定的网络功能

因为eBPF对非连续内存的使用施加了严格限制，阻碍了部分网络功能核心组件的实现。例如基于跳表的key-value store 和基于红黑树的优先级队列等。使用非连续内存意味着eBPF需要支持将可变数量的动态内存持久化。尽管最近的Linux内核（版本6.1及以上）支持分配动态内存并将其持久化到BPF MAP中，但验证器强制规定了BPF MAP只能持久化固定数量的动态内存。因此，由于缺乏对可变动态内存的支持，现有的eBPF无法使用非连续内存。

例如，以下代码展示了eBPF目前支持动态内存，但无法支持可变数量的动态内存。
![[Pasted image 20240920141231.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 2.2 用eBPF实现网络功能性能次优

首先，eBPF的RISC指令集缺乏对特定指令的支持，包括SIMD指令和bitscan指令 (FFS等)，导致性能下降。例如，不支持SIMD导致了eBPF在实现网络功能时无法采用并行计算、并行查找等在网络功能中被使用的加速方式。在sketch等网络功能中，这会导致49.2%的性能下降。其次，eBPF帮助函数 bpf_get_prandom_u32 对于网络功能来说性能开销太大。如果每一个包都调用一次bpf_get_prandom_u32 导致NitroSketch 46.6%的性能下降。

## 2.3 现有的解决方案存在的缺陷

为了解决这两个技术挑战，可以考虑两种解决方案。

第一种解决方案是增强eBPF的整体架构，例如扩展eBPF的指令集、增强验证器、引入新的运行时和语言级别的安全机制，以及将验证过程从内核解耦到用户空间。

然而，由于对内核的修改过于激进，实用价值较低，不易于部署和推广。例如，扩展eBPF指令集需要对内核代码库中架构特定的JIT编译器进行修改，目前涵盖多达14种硬件架构。此外，扩展指令集要求对验证器的代码进行修改，因为验证器针对eBPF指令进行验证。

但是修改验证器可能会引入新的bug和安全问题。尽管重新设计eBPF的安全和编程架构在理论上是可行的，这种方案目前难以被直接部署，并且可能对以后的eBPF网络功能产生负面影响。

第二种解决方案是将所有功能无法实现的和性能下降的网络功能实现为内核模块（通过kptr和kfunc技术）或者集成到内核中 （实现为新的帮助函数和BPF MAP)。

然而，将所有网络功能集成到内核中将对内核造成巨大的改动，难以被内核社区接受。而根据需求集成单个网络功能，可能会由于需求变化而导致频繁的内核模块更换，进而导致内核不稳定。

鉴于网络社区的快速发展，这种 "一个内核模块实现一种网络功能" 的方法可能会使内核变得相当不稳定。

# 3、基于标准库的优化eBPF网络功能技术方案

## 3.1 基于网络功能中的通用设计模式

网络功能中存在一些通用的设计模式，总结如下：

1. 使用bit scan指令，例如FFS (find the first bit), popcnt指令等，实现快速检索。这种设计会被用在高性能的优先级队列的实现上，例如通过FFS快速定位第一个存放元素的bucket 。
    
2. 同时计算多个hash函数。很多实现网络测量的网络功能，会使用一些基于概率和统计的数据结构，例如sketch和bloom filter。同时计算多个hash函数来降低冲突概率。
    
3. 使用基础的数据结构。例如，top-k heap, 桶链表等。
    
4. 使用随机数。为了提升性能，部分网络功能会根据概率执行特定的操作，例如一些Heavy Hitter。
    
5. 使用非连续内存。例如使用跳表和红黑树等。
    
6. 将数据保存在连续内存中。例如，网络功能中的一些高性能的hash表，例如DPDK中的cuckoo hash，将多个key保存在一块连续bucket中来降低hash冲突。
    

## 3.2 eBPF网络标准库的设计和实现

为了在不修改内核的前提下，解决上述的技术挑战，我们设计并实现一个可供eBPF调用的网络功能标准库eNetSTL。

eNetSTL将上述的通用的模式抽象并实现为一系列高性能低开销的API。在解决问题的同时，避免代码过度膨胀。eNetSTL基于eBPF的 kernel function (kfunc) 和 kernel pointer (kptr) 技术实现，并将API实现在内核模块中，从而避免了内核的修改。

目前eNetSTL的设计除了使用kfunc和kptr接口外，其他部分是self-contain的。因此能保持较好的内核版本的兼容性。eNetSTL包含的内容如下图所示：
![[Pasted image 20240920141240.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

具体来说，eNetSTL包含以下内容：

1. Memory wrapper:  支持在eBPF中使用非连续内存的同时，不破坏eBPF提供的安全保证。
    
2. 算法：包括位运算、基于SIMD的并行hash计算和并行比较算法。
    
3. 数据结构：list bucket 数据结构，支持GEO (几何随机数) 分布的随机数池。
    

其中Memory wrapper的实现充分利用了kfunc和kptr技术。其主要设计包括：

1. 通过用一个proxy kptr来管理所有新分配的 node kptr，避免BPF MAP中只能保存静态数量的kptr。
    
2. 由eNetSTL管理所有的底层指针，通过kfunc实现节点到节点的指针路由，通过给kfunc增加KF_ACQUIRE tag 来安全获取下一个节点的指针，并在eBPF中直接访问该指针，例如 a->b。
    

下面是Memory wrapper的部分API：
![[Pasted image 20240920141250.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 4、eNetSTL使用技术实践

### 4.1 基于eNetSTL实现跳表

通过Memory wrapper API，直接在eBPF里使用非连续内存。我们用简化版本的单链表来展示使用非连续内存 （跳表的实现类似）：
![[Pasted image 20240920141303.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

性能测试结果 （40G网卡 单核性能） 如下图所示 （红色折线代表用内核模块实现，黄色折线代表用eNetSTL实现)：
![[Pasted image 20240920141310.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

我们验证了跳表的查找性能和插入性能，可以看到使用eNetSTL在使能了原本无法直接实现的跳表的同时，其性能损耗在10%以下。

### 4.2 基于eNetSTL实现sketch

sketch是一种在网络测量领域常用的网络功能，其核心设计是使用多个hash函数将同一条流的数据包映射到多个counter上。我们使用eNetSTL的API来加速多个函数的计算，典型的Count-min sketch用eNetSTL实现代码如下：
![[Pasted image 20240920141317.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

性能测试结果 （40G网卡 单核性能） 如下图所示 （红色代表用内核模块实现，黄色代表用eNetSTL实现，蓝色表示用纯eBPF实现）：
![[Pasted image 20240920141329.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

实验结果显示，与eBPF相比，基于eNetSTL的实现平均性能提升了47.9%。特别是，随着哈希函数数量的增加，这种提升变得更加显著，使用8个哈希函数时达到了70.9%的峰值。这是由于随着哈希函数数量的增加，SIMD指令能带来更多的优化效果。并且调用eNetSTL几乎不会带来性能损失。

### 4.3 基于eNetSTL优化Cuckoo Switch中的hash性能

Cuckoo Switch中使用了Blocked Cuckoo hash这一核心数据结构。相比于原始的Cuckoo hash， Blocked Cuckoo hash为了降低hash的冲突率，在一个bucket中同时保存16个hash指纹。我们参考DPDK的实现，使用eNetSTL提供的 `hw_hash_crc`（用硬件指令生成crc来代替hash计算）和 基于SIMD的并行比较算`bpf__find_mask_u16`分别优化hash的计算、hash指纹的比较、和full-key的比较。

下面是一个简化后的例子：
![[Pasted image 20240920141336.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
![[Pasted image 20240920141342.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

性能测试结果 （40G网卡 单核性能） 如下图所示 （红色的折线代表用内核模块实现，黄色的折线代表用eNetSTL实现，蓝色的折线表示用纯eBPF实现）：
![[Pasted image 20240920141349.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

使用了eNetSTL的方案与纯eBPF 相比，平均性能提升 27.4%，并且随着负载的增加，性能提升更加明显，在满负载时达到 33.08%。这是因为，随着负载增加，单个条目中的平均比较次数也增加。基于 SIMD 的并行比较优化效果变得更好。在低负载场景下，优化主要体现在使用 hw_hash_crc 替代基于软件的哈希计算和 SIMD 优化的full key比较。与内核相比，采用eNetSTL的方案平均性能损失约为4.30%。

  

如果你有eBPF及linux相关问题，请联系小编微信号wenamao，邀请加入 酷玩BPF学习交流群。

另外，打个小广告，东南大学计算机学院沈典副教授课题组招收对网络系统相关方向感兴趣的硕士/博士生，欢迎报考！请扫描下面二维码了解详细信息。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Reads 1139

​

Comment

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/RWnbgykjTNLYUsm55aeV0aP5Qet4yIGmBEsbwoZdgdUdYTGJJrYknQLvVdzjs9wIdkHrNoKfEGcvCyy7HFjTvA/300?wx_fmt=png&wxfrom=18)

酷玩BPF

111219

Comment

Comment

**Comment**

暂无留言
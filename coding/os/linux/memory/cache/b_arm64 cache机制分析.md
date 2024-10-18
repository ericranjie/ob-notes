
原创 LoyenWang LoyenWang _2022年04月09日 23:29_

# 背景

- `Read the fucking source code!`  --By 鲁迅
- `A picture is worth a thousand words.` --By 高尔基

说明：

1. Kernel版本：4.14
1. ARM64处理器，Contex-A53，双核
1. 使用工具：Source Insight 3.5， Visio

# 1. 概述

先来看一下经典的存储器层次结构图：

![图片](https://mmbiz.qpic.cn/mmbiz_png/OtawKxIaP3ibe3fE48wFnb4sG5icQcYYTJkWh7ec6uzFF46sMYUaV3NmicMzVYnTV8t8OWHlgvulXOI0Cy5GOOFjQ/640?wx_fmt=png&wxfrom=13&tp=wxpic)

- 不同存储器技术的访问时间差异很大，CPU和主存的速度差距在增大，如果直接从主存进行数据的load/store，CPU的大部分时间将会处在等待的状态；
- cache的作用就是来解决CPU与主存的速度不匹配问题；

以ARMv8的CPU架构为例：

![图片](https://mmbiz.qpic.cn/mmbiz_png/OtawKxIaP3ibe3fE48wFnb4sG5icQcYYTJ1hdfRtHnL2fWYZvdpjIYQ3Xme31Zncso3YDI8zDJNeuZiaWANcGWOdA/640?wx_fmt=png&wxfrom=13&tp=wxpic)

- ARMv架构的CPU通常包含两级或多级cache；
- L1 cache为Core所独享的，通常包含Instruction cache和Data cache；
- L2 cache为同一个Cluster中的多个Core进行共享；
- L3 cache通常实现为外部的硬件模块，可以在Cluster之间进行共享，或者与其他IP进行共享；

接下来让我们对cache一探究竟。

# 2. cache

## 2.1 cache结构

先看一下cache的内部结构图：
![[Pasted image 20241008183012.png]]

- `cache line`：cache按行来组织，它是访问cache的最小单位，通常为16、32、64或128字节，`cache line`的大小通常在架构设计阶段确定；
- `64-bit address`：CPU访问cache的地址编码，分成三部分：`Tag`、`Index`、`Offset`；
- `Index`：地址编码中的index域，用于索引`cache line`；
- `Offset`：地址编码中的offset域，`cache line`中的偏移量，可按`word`和`byte`来寻址；
- `Tag`：`Tag`在cache中占用实际的物理空间，用于存储缓存地址的高位部分，通过与地址编码中的`Tag`域进行比较来确定是否`cache hit`；
- `Way`：cache分成大小相同的子块，每个子块以相同的方式进行索引；
- `Set`：所有`Way`中相同的索引对应的`cache line`组成的集合；

## 2.2 cache映射

### 2.2.1 direct mapped

直接映射的方式如下：
![[Pasted image 20241008183028.png]]

- 图中的示例cache只有四个缓存行，内存中`index域`相同的地址，都会映射到cache中的同一行上；
- 图中所示，`0x00,0x40,0x80`三个地址会映射到cache的第一行，而同一时刻只能允许1行；
- 如果有程序访问区域覆盖了这三个地址，可能造成cache line的频繁换入换出，从而导致`cache thrashing`（颠簸）问题；
- 优点是硬件设计简单，成本低，缺点是cache颠簸造成性能问题；

### 2.2.2 set associative

组相连的映射方式如下：
![[Pasted image 20241008183043.png]]

- 组相联方式在现代处理器中得到广泛的使用；
- 图中示例有两路cache，因此每组有两个缓存行；
- 每个内存地址在映射时，有两个缓存行可以选择，替换出去的概率降低了，这也就有效的降低了cache颠簸；
- 优点是减少了cache颠簸，缺点是成本和复杂度增大；

### 2.2.3 fully associative

全相连的映射方式如下：
![[Pasted image 20241008183052.png]]

- 内存地址不需要`index域`，全相连缓存相当是N路集中的所有缓存行，因此需要大量的比较器；
- 优点是最大程度降低cache颠簸，缺点依然是复杂度和成本问题；

## 2.3 cache策略

- Read allocation(RA) 当读操作`cache miss`时默认进行分配`cache line`；
- Write allocation(WA) 当写操作`cache miss`时，会触发一个burst读，通过读的方式来分配`cache line`，然后再将数据写入cache；
- Write-back(WB) WB方式下，数据只写入cache，并标记为dirty，当`cache line`被换出或者显式的clean操作才会更新到外部内存中，如下图：

![[Pasted image 20241008183105.png]]

- Write-through(WT) WT方式下，数据同时写入cache和外部内存，不会将`cache line`标记为dirty，如下图：
  ![[Pasted image 20241008183127.png]]

## 2.4 cache分类

先来看看cache中的重名(`aliasing`)问题和同名(`homonyms`)问题：

![[Pasted image 20241008183135.png]]

- `aliasing`：多个不同的虚拟地址可能映射到相同的物理地址；
- 引入的问题包括：1）浪费cache的空间，造成性能下降；2）写操作时可能造成cache数据更新不一致，造成物理地址在cache中维护多个不同的数据；
- `homonyms`：相同的虚拟地址映射到不同的物理地址；
- 引入的问题：比如进程切换时，上一个进程的相同虚拟地址在cache中的数据，对于本进程无用，需要额外的`invalidate`操作；

### 2.4.1 `VIVT`(`Virtually-Indexed Virtually-Tagged`)

![[Pasted image 20241008183158.png]]

- VIVT：处理器使用虚拟地址来进行cache的寻址操作，使用虚拟地址的`Tag域`和`Index域`进行判断是否hit；
- 导致cache重名问题（Index域导致）与同名问题（Tag域导致），当改变虚拟地址到物理地址映射时，需要`flush`和`invalidate`操作，导致性能下降；

### 2.4.2 `PIPT`(`Physically-Indexed Physically-Tagged`)

![[Pasted image 20241008183212.png]]

- PIPT：处理器使用物理地址进行cache的寻址操作，使用物理地址的`Tag域`和`Index域`进行判断是否hit；
- 处理器在查询cache时，需要先查询MMU/TLB后才能访问，增加了pipeline的时间；
- 能有效避免重名和同名问题，但是硬件的设计复杂度和成本更高；

### 2.4.3 `VIPT`(`Virtually-Indexed Physically-Tagged`)

![[Pasted image 20241008183225.png]]

- VIPT：使用虚拟地址的`Index域`和物理地址的`Tag域`进行判断是否cache hit；
- 使用物理地址的`Tag域`(物理Tag唯一)，能有效的避免同名问题；
- VIPT也可能存在重名问题，以Linux为例，Linux内核以4KB的页面大小进行管理，虚拟地址和物理地址的\[11:0\]是相同的，重名问题下多个虚拟地址的\[11:0\]是一样的。当index索引域在\[11:0\]之内时，不会发生重名问题，因为该范围属于一个页面内；

# 3. mesi

先来看问题的引入：
![[Pasted image 20241008183238.png]]

- 图中的cluster，不同CPU core的cache和DDR中可能维护了同一个数据的多个副本；
- 维护cache的一致性，需要跟踪cache行的状态，ARM采用MESI协议（`snooping protocol`）来维护cache的一致性；

> MESI协议的名字来源于cache line的四个状态：

- `Modified（M）`：cache line数据有效，cache line数据被修改，与内存中的数据不一致，修改的数据只存在本cache中；
- `Exclusive（E）`：cache line数据有效，cache line数据和内存中一致，数据只存在本cache中；
- `Shared（S）`：cache line数据有效，cache line数据和内存中一致，数据存在于多个cache中；
- `Invalid（I）`：cache line数据无效；

状态说明如下：

1. M和E状态，数据都是本地独有的，不同点在于M状态的数据是脏的，而E状态的数据是干净的，M状态的cache line写回内存后，状态变成了S；
1. S状态的cache line，数据和其他cache共享，只有干净的数据才能被多个cache共享；

![[Pasted image 20241008183259.png]]

MESI协议在总线上的操作分为两大类：CPU请求和总线请求，如下图：

![[Pasted image 20241008183310.png]]

MESI协议中涉及到各个状态的转换：

1. 本地CPU操作，状态转换如下图：

![[Pasted image 20241008183317.png]]

- 操作为本地CPU读写

2. 总线监听到其他CPU的操作请求，状态转换如下图：

![[Pasted image 20241008183324.png]]

- 操作类型为总线读写，来自其他CPU的请求，或者DMA的操作等；

当多个cpu访问同一个cache line中的不同数据时，根据MESI协议，容易造成cache的伪共享问题，解决方式是让多线程操作的数据处在不同的cache line中。

收工了。

# 参考

`《 ARM Cortex-A Series  Programmer's Guide for ARMv8-A》`
`《ARMv8-A CPU Architecture Overview》`
`《奔跑吧Linux内核》`
Lecture 8. Memory Hierarchy Design II
TEACHING THE CACHE MEMORY COHERENCE WITH THE MESI PROTOCOL SIMULATOR
Cache组织方式

______________________________________________________________________

​

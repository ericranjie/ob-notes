原创 我不想种地 码砖杂役

_2024年03月27日 07:32_ _北京_

## **术语解释**

- 存储和内存被交替使用，对应单词Memory

- 本地和局部被交替使用，对应单词Local

- 主机（Host）表示CPU，主机存储指内存

- 设备（Device）表示GPU，也叫卡，设备存储指显存

- CPU内存和GPU线程都是DRAM

- 共享存储和L1缓存都是SRAM

- 共享存储指CUDA中的Shared Memory，不是linux IPC shm

- 全局内存指GPU显存

- 核，指cuda kernel function

## **CUDA存储模型介绍**

内存访问和管理是所有编程语言的重要部分，内存管理对现代加速器的高性能计算有非常大的影响，因为许多工作负载都受限于它们能多快的加载和存储数据，有大容量、低延迟、高带宽的存储对性能很关键。然而追求大容量高性能存储并不可行或不经济。

因此，你必须依赖存储模型去获得给定硬件存储子系统的最优延迟和带宽。CUDA内存模型统一了单独的主机和设备存储系统，且暴露了全存储层次，以便于显式控制内存放置去获得最佳性能。

### **存储层次结构的好处**

- 时间局部性

- 空间局部性

现代计算机使用存储层次结构（Memory Hierarchy）去优化性能，存储层次结构之所以有用归功于**局部性原理**，一个存储层次由多级存储构成，离处理器越远延迟越高、容量越大、每位存储的成本越低，被访问的频次越低。

![](https://mmbiz.qpic.cn/mmbiz_png/pg57MH7pFC4nXoLXNgsM0SKMUsbBXK4xFeOjSKZgffXbhftvco0RYlC9Duz0cNFL0YgEDlERWeKpZyibc6n4naQ/640?wx_fmt=png&from=appmsg&wxfrom=13&tp=wxpic)

CPU和GPU的主存都使用DRAM（动态随机访问存储），低延迟存储（例如CPU L1 Cache）则使用SRAM（静态随机访问存储），存储层级中最大最慢的一级存储一般使用磁盘或闪存。

在存储层级结构中，当数据正被处理器活跃使用的时候，它被保存在低延迟低容量存储中，比如寄存器、SRAM等；当数据被存储为供以后使用的时候，它被存储在高延迟大容量的存储中，比如磁盘和SSD，存储层级结构提供一个存储假象（既容量大又延迟低）。

GPU和CPU在存储层次结构设计中使用类似的原则和模型，最大的不同是：GPU暴露存储层级的更多信息，程序员对它的行为有更多的显式控制。

### **CUDA内存模型**

对程序员而言，根据可编程性，分为2类存储：

- 可编程存储，你可以显式的控制什么数据放置到可编程存储。

- 不可编程存储，你无法控制数据布置，而是借助于自动技术去获得好的性能。

在CPU存储层级中，L1和L2级缓存是不可编程存储的例子，另一方面，CUDA存储模型向程序员暴露多种类型的可编程存储：

- 寄存器（Register）

- 共享存储（Shared Memory）

- 本地存储（Local Memory）

- 常量存储（Constant Memory）

- 纹理存储（Texture Memory）

- 全局存储（Global Memory）

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图4-2展示CUDA的存储层级，每一类有不同的范围、生命周期以及缓存行为。

- 本地存储：每个内核线程有它自己的私有本地存储

- 共享存储：每个线程块（Block）有它自己的共享存储，共享存储对线程块的所有线程可见，共享存储里的数据在线程块的生命周期内被保持

- 全局存储：所有线程都可以访问全局存储

- 常量存储和纹理存储：这是两类所有线程都可访问的只读存储。全局内存、常量内存、纹理内存这三类存储是为了不同用途做了优化。纹理存储提供不同寻址模式和为各种数据布局提供过滤。全局存储、常量存储和纹理存储拥有跟应用相同的生命周期

#### **寄存器**

寄存器是GPU上最快的存储空间，**在内核函数里声明的，没有其他任何类型修饰的自动变量通常保存在寄存器里**。声明在内核函数里的数组也可能被保存在寄存器，但需满足引用数组的索引是常量且能在编译期确定。

**寄存器变量是线程私有的**，内核通常使用寄存器去保存本地频繁访问的线程私有变量，寄存器变量与内核线程共享生命周期，一旦核函数执行完成，寄存器变量就不能再被访问了。

**寄存器是稀缺资源，它被流式多处理器上的活跃线程束瓜分**。不同GPU架构，对于每个线程使用的寄存器数量有硬件限制，比如费米GPU一个线程最多63个寄存器，开普勒扩大这个限制到255。在你的核函数里使用更少的寄存器将允许一个流式多处理器上驻留更多的线程块，每流式多处理器更多并发线程块可提升卡的占用和性能。

如果一个核函数使用了超过硬件限制的寄存器，**超出的寄存器将溢出到本地存储**，寄存器溢出损伤性能，nvcc编译器使用启发性的策略去最小化寄存器使用和避免寄存器溢出。你可以提供可选的辅助启发信息。

#### **本地存储**

内核函数里定义的变量，如果适合放到寄存器，但受限于寄存器的数量而不能给它分配寄存器，就会溢出到本地存储，可能被编译器放到本地存储的变量可能是：

- 索引值在编译期不能被确定的局部变量数组

- 会消耗太多寄存器的大的局部结构和数组

- 不能适应内核寄存器限制的任何变量，即因为内核寄存器限制导致分配不到寄存器的变量

本地存储的名字是有误导性的，**溢出到本地存储的值驻留在与全局内存同样的物理位置**，因此，本地存储访问也有高延迟低带宽的特征，也受制约于“内存访问模式”一节中描述高效存储访问的要求。计算能力2.0及以上的GPU，**本地存储数据也被缓存在每流式多处理器的L1缓存和每卡的L2缓存。**

#### **共享存储**

内核函数里，**受\_\_shared\_\_关键字修饰的变量保存在共享存储里**，因为**共享存储是片上存储**，因此，共享存储有比本地或全局存储高得多的带宽和低得多的延迟。它类似CPU的L1缓存，但是共享存储是可编程的，而CPU L1缓存不可编程。

每个流式多处理器都配置有限数量的共享存储，共享存储被线程块瓜分，因此，你必须小心的不过度的使用共享存储，否则将限制活跃线程束的数量。

共享存储在核函数里声明，但是生命周期跟线程块一致，当一个线程块执行完成，分配给它的共享存储就会被释放，然后被分配给其他线程块。

共享存储是线程间通信的基本方式，一个线程块内的所有线程都可以通过保存在共享存储的共享数据协作，访问共享存储需要借助CUDA运行时的同步接口：void \_\_syncthreads()。

这个函数相当于为线程块内的所有线程创建了一个屏障，每个线程都必须到达这个函数，才可以继续往下执行。通过为线程块内的所有线程创建屏障，你可以阻止潜在的数据冒险问题，这类数据冒险问题发生在多个线程以未定义的顺序访问相同内存位置的数据，其中至少有一个写访问。\_\_syncthreads对性能有负面影响，因为它会迫使流式多处理器频繁的空闲。

**流式多处理器的L1缓存和共享存储使用相同的64KB片上存储**，各自大小是被静态划分的，但可以通过运行时接口cudaFuncSetCacheConfig动态配置。

#### **常量存储**

**常量内存位于设备存储（DRAM）中**\*\*，且被缓存在每个流式多处理器的专门只读缓存，一个常量变量用\_\_constant\_\_修饰。\*\*

常量变量必须被声明在任何核函数外的全局空间里。常量内存容量有限，所有计算能力的GPU都只有64KB常量内存。常量内存被静态声明，且对相同编译单元内的所有核函数都可见。核函数只能从常量内存读，常量内存只能在主机端通过cudaMemcpyToSymbol初始化。

```
cudaError_t cudaMemcpyToSymbol(const void* symbol, const void* src, size_t count);
```

该函数从src指向的地址拷贝count字节到symbol，这个src是主机内存地址，而symbol是全局或者常量内存地址，该函数大多数情况下是同步的。

**当线程束里的线程从相同内存地址读数据的时候，常量内存性能表现最好**，例如用来存放数学公式的系数。如果线程束里的每个线程从不同地址读，且只读一次，那么常量内存不是最好的选择，因为从常量内存的一次读会广播给线程束内的所有线程。

#### 

#### **纹理存储**

\*\*纹理内存位于设备存储中，且被缓存在每个流式多处理器的专门只读缓存。\*\*纹理内存是一类典型的通过专门缓存访问的全局内存。这个只读缓存包括硬件过滤，它可以在读处理过程中做浮点插值。纹理内存针对2D空间局部性做了优化，因此，线程束里的线程使用纹理存储去访问2D数据将获得最好的性能。对于有些应用而言，它是理想的，它因缓存和硬件过滤而提供性能优势，然后，对于另一些应用，使用纹理内存比全局内存更慢。

#### **全局存储**

全局内存是GPU里容量最大、延迟最高、最广泛使用的存储。全局意味着全局范围和全生命周期。它的状态，在应用的生命周期内，可以被任何流式多处理器访问。

全局内存在本文中，也叫设备存储，DRAM。

全局内存既可定义成静态的，也可以被定义为动态的，在设备端代码里定义变量的时候要用\_\_device\_\_修饰。

全局内存在主机侧代码通过cudaMalloc分配、通过cudaFree释放，指向设备内存的指针随后被作为参数传给核函数，全局内存存续在所有核函数的所有线程的应用的整个生命周期，核函数的多线程对全局内存的访问要注意并发访问问题，不同线程块里的线程并发修改相同位置的全局内存会导致未定义的程序行为。

位于设备内存的全局内存通过32B、64B或者128B的内存事务访问，内存事务必须是按对齐要求对齐的，即首地址必须是32B、64B或128B的整数倍，优化内存事务对性能很重要。当一个线程束执行load、store，满足请求的事务的数量依赖于2个因素：

- 线程束中所有线程的内存地址的分布

- 每个事务的内存地址对齐

通常，为了满足内存请求的内存事务数越多，潜在的被传输的未使用字节数就越多，这会导致吞吐效率的降低。

对一个给定的线程束内存请求，事务的数量和吞吐效率由设备的计算能力确定，对于计算能力1.0和1.1的设备，全局内存访问的要求严格，1.1以上的设备，因为事务被缓存，所以要求被放松了，缓存内存事务利用数据局部性去提升吞吐效率。

后面的部分会检查如何优化全局内存访问和如何检查全局内存吞吐效率。

#### **GPU缓存**

像CPU缓存一下，GPU缓存也是不可编程存储，GPU设备有4类缓存。

- L1

- L2

- 只读常量

- 只读纹理

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

每个流式多处理器都有独立的L1缓存，所有流式多处理器共享同一个L2缓存。L1、L2缓存都用来保存局部和全局存储数据，包括寄存器溢出。后来的GPU允许你配置读操作是否被L1和L2缓存，亦或是只被L2缓存。

在CPU中，内存加载和存储都过缓存，然而GPU中，只有读过缓存，写不过缓存。

每个流式多处理器也有一个只读常量缓存和一个只读纹理缓存，它们分别被用于提升从设备存储空间读的性能。

#### 

#### **CUDA变量声明总结**

下表总结CUDA变量声明和它们对应的内存位置、范围、声明周期和修饰符。

|修饰符|变量名|内存|范围|生命周期|
|---|---|---|---|---|
||float var|Register|Thread|Thread|
||float var\[100\]|Local|Thread|Thread|
|__shared__|float var †|Shared|Block|Block|
|__device__|float var †|Global|Global|Application|
|__constant__|float var †|Constant|Global|Application|

† 即可是标量也可以是矢量

**各种存储类型的主要特征总结**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

#### **静态全局内存**

```
__device__ float devData;
```

devData是静态全局变量，核函数checkGlobalVariable打印它的值并修改它的值，main函数里通过cudaMemcpyToSymbol对它赋值，待kernel函数执行完后，main函数把devData的值拷贝到内存，并打印。

## **内存管理**

CUDA编程中的内存管理与C编程类似，但CUDA程序员需要显式负责数据在主机和设备间的移动。NVIDIA趋向于在每个release版本中统一主机和设备存储空间，对大多数应用手动数据移动仍然是需要的，该领域的最新进展会在“统一存储”一节阐述。现在，聚焦在如果通过CUDA函数去管理内存和数据移动。

- 分配和回收设备内存

- 在主机和设备之间传输数据

为了获得最大性能，CUDA提供接口：在主机准备设备存储和显式在主机和设备之间传输数据。

### **内存分配和回收**

cuda编程模型设定异构系统由主机和设备组成，主机和设备分别有自己独立的内存空间，核函数操作设备内存空间。cuda运行时提供分配、初始化、回收设备存储空间的接口。

cudaMalloc从设备分配count字节的全局内存，通过devPtr指针返回存储地址，如果失败该函数返回cudaErrorMemoryAllocation，分配的内存未被清理，你需要用主机内存数据去填充新分配的全局内存，或者通过cudaMemset去初始化。

一旦应用不再使用通过上述cudaMalloc分配的全局内存，那么应该通过cudaFree释放，devPtr需要传递之前通过cudaMalloc返回的设备内存，否则会报错cudaErrorInvalidDevicePointer，二次释放也会返回错误。

设备内存的分配和回收是重操作，因此，只要可能，设备内存就应该被应用复用，从而去减少对整体性能的影响。

```
// 从设备存储空间分配count字节，devPtr返回设备空间的内存地址
```

### 

### **内存传输**

一旦全局内存被分配，就可以从主机内存传输数据到设备内存，通过：

```
/*
```

kind参数指定拷贝的方向，可以是4个枚举值：

- cudaMemcpyHostToHost，从主机到主机

- cudaMemcpyHostToDevice，从主机到设备

- cudaMemcpyDeviceToHost，从设备到主机

- cudaMemcpyDeviceToDevice，从设备到设备

下图显示CPU内存和GPU内存的连接，图片显式GPU芯片和GPU内存之间的理论峰值带宽很高，而CPU和GPU之间通过PICe总线连接的理论峰值带宽低很多。这表示CPU和GPU内存之间的数据传输（如果没有被恰当的管理）可能是整个应用的瓶颈。因此，CUDA编程的一个基本原则是：你应该总是思考最小化主机和设备之间的数据传输。

### **固定内存**

主机分配的内存默认是pageable的，这导致GPU不能安全的操作主机内存，因为没法确保在从设备内存往主机内存拷贝数据的时候，主机内存不被系统回收。

```
// 分配固定内存
```

图4-4左边表示主机内存往设备内存拷贝数据的时候，会先从pageable内存拷贝到固定内存，然后再从主机固定内存拷贝到设备内存。如果使用cudaMallocHost分配固定内存，就能像图右侧那般，直接从主机固定内存拷贝到设备内存，减少一次中间拷贝。

固定内存的分配和释放比pageable内存成本更高，但是传输效率更高，特别是针对大数据量传输。

### 

### **零拷贝内存**

一般的，主机代码无法访问设备内存，而运行在设备上的核函数无法操作主机内存，但有一个例外：零拷贝内存，主机和设备都可以访问零拷贝内存。

GPU线程可以直接访问0拷贝内存，cuda核函数中使用0拷贝内存有几个好处：

- 可以利用主机内存，当设备内存不够的时候

- 避免主机和设备之间显式的数据传输

- 提升PICe传输效率

当使用0拷贝内存在主机和设备之间共享数据的时候，你必须跨主机和设备同步内存访问，从主机和设备同时修改0拷贝内存会导致未定义行为。

0拷贝内存是被映射进设备内存空间的固定内存，可以通过下面接口创建被映射进设备内存空间的固定内存：

```
cudaError_t cudaHostAlloc(void **pHost, size_t count, unsigned int flags);
```

cudaHostAlloc用于分配页锁定并可被设备空间访问的主机内存，该内存必须用cudaFreeHost释放。

cudaHostGetDevicePointer用于获取被映射被锁定的主机内存在设备存储空间的指针。

### 

### **统一虚拟寻址**

计算能力 >= 2.0+的GPU支持特殊的寻找模式：统一虚拟寻址（UVA）。通过UVA，主机内存和设备共享单一的虚拟地址空间，如图所示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 

### **统一内存**

CUDA 6.0引入了统一内存去简化cuda编程模型的内存管理。统一内存创建一个被管理的内存池，从这个池中分配的内存，可以通过同一指针，从CPU和GPU访问。底层系统在主机和设备之间自动的迁移数据，数据传输对应用透明，极大的简化了应用代码。

统一内存依赖于UVA的支持，但是它们是不同的技术，UVA为系统中的所有处理器提供一个虚拟内存寻址空间，然后，UVA不会自动从一个物理位置到另一个物理位置迁移数据，这个能力跟统一内存不同。

**内存访问模式**

大多数设备数据访问从全局内存开始，大多数GPU应用受限于存储带宽。因此，最大化你的应用的全局带宽使用是内核性能调优的基本步骤。如果你没有合适的调优你的全局内存使用，其他优化可能起负面效果。

为了在读写数据的时候获得最佳性能，内存访问操作必须满足一些条件。cuda执行模型的一个显著特征是指令按线程束发射和执行。大多数内存操作也按线程束发射。当执行一个存储指令，线程束里的每个线程提供一个加载或存储的内存地址。线程束里的32个线程提出一个单一内存访问请求，该请求由请求地址组成，它由一或多个设备存储事务去服务。依赖线程束里的内存地址的分布情况，内存访问可被分为几种模式。这一节，你将检查这些不同内存访问模式，并学会如何获得最佳全局内存访问性能。

### **对齐和合并访问**

全局内存加载/存储通过缓存暂存，如图4-6，全局内存是你能从内核函数里访问的逻辑存储空间。所有应用数据初始驻留在DRAM（物理设备存储）。内核函数的内存请求通过设备DRAM和流式多处理器片上存储之间128B或32B内存事务来满足。

所有到全局内存的访问都通过L2缓存，许多访问也通过L1缓存（依赖于访问类型和GPU架构）。如果L1和L2缓存都被使用，内存访问会被按128B内存事务服务，如果只有L2缓存使用，内存访问会被按32B内存事务服务。在允许L1缓存被用于全局内存缓存的架构，L1缓存可以在编译期显式的打开和禁用。

一个L1缓存行是128B，它被映射到一个128B对齐的设备内存段，如果一个线程束里的每个线程请求一个4B值，这将导致每个请求包括128B的数据，它正好匹配缓存行大小和设备存储段大小。

当优化应用的时候，你应该向两个设备存储访问特征的目标努力：

- 对齐内存访问

- 合并内存访问

- !\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**对齐内存访问**发生在当设备内存事务的首地址是用于服务事务的缓存粒度的整数倍时（L2缓存是32B，L1缓存是128B），不对齐的加载，会导致带宽浪费。

**合并内存访问**发生在线程束的所有32个线程访问一个连续的内存块。

**对齐且合并的内存访问**是理想的：线程束访问一块连续的内存，这块内存的首地址号是对齐的地址。为了最大化全局内存吞吐，组织内存操作让它既对齐又连续式非常重要的。

通常，你应该有话内存事务效率：使用最小数量的事务，去服务最多数量的内存请求。需要多少事务，有多大吞吐，是随设备计算能力而变化的。

内存访问模式：对齐和合并访问，减少内存事务次数，避免内存带宽浪费。

### **全局内存读**

在流式多处理器中，根据引用的设备存储的类型不同，数据会流水线式地通过三种cache/buffer路径之一：

- L1/L2缓存

- 常量缓存

- 只读缓存

L1/L2缓存是默认路径，为了让数据通过其他两条路径传递数据，需要应用做显式的管理，但是可获得性能提升（取决于所用访问模式）。全局内存加载操作是否穿过L1缓存依赖两个因素：

- 设备计算能力

- 编译选项

计算能力2.0的费米GPU、开普勒K40及后续GPU，全局内存数据是否过L1缓存可通过编译选项配置，编译选项（-Xptxas -dlcm=cg）用于通知编译期禁用L1缓存。

如果禁止L1缓存，那么加载全局数据直接过L2缓存，如L2缓存命失，请求会由DRAM服务。每个内存事务可能被分成1、2、4段，一段32B。

L1缓存也可以通过编译选项（-Xptxas -dlcm=ca）显式启用。借助该选项，全局内存加载先过L1，L1命失，则请求L2，L2命失，再由DRAM去服务。这个模式，一个加载内存请求被一个128B的设备内存事务去服务。

**内存加载模式**

有两类内存加载类型：

- 缓存加载（L1缓存启用）

- 非缓存加载（L1缓存禁用）

内存加载的访问模式可被区分为下列组合：

- 缓存 vs 非缓存：如果L1缓存启用，则内存加载被缓存

- 对齐 vs 非对齐：如果内存访问的首地址是32B的整数倍，则内存加载是对齐的

- 合并 vs 非合并：如果线程束访问一块连续数据块，则加载是合并的

#### 

#### **缓存加载**

缓存内存加载穿过L1缓存，被设备内存事务以L1缓存行（128B）的粒度服务，缓存加载可被分为对齐/非对齐和合并/非合并。

图4-9是理想状况，对齐且合并的内存访问。线程束里的所有线程请求的地址都落到一个128B的缓存行，只需要一个128B事务就能完成这个内存加载操作，总线利用率是100%，事务中不存在未使用数据。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图4-10显示另一种情况，访问是对齐的，但引用的地址不是依线程ID连续的，而是在128B内随机分布，因为线程束内的所有线程请求的地址还是落在一个缓存行，也只需要一个128B事务就能满足这个内存加载操作，总线利用率也是100%，因为每个线程也是请求128B内的独立4字节，所以事务中不存在未使用的数据。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图4-11显示另一种情况：线程束请求32个连续的4字节元素，它不是对齐的，请求的地址跨了2个全局内存中的128B段。因为被流式多处理器执行的物理加载操作必须在128B对齐的边界进行（当L1缓存开启的时候），需要2个128B事务才能实现内存加载操作，总线利用率50%，事务中的一半数据未被使用。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图4-12显示线程束里的所有线程请求同一个内存地址，因此只需要一个内存事务，但是总线利用率很低，4/128=3.125%。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图4-13演示了更糟糕的情况，线程束里的线程请求分散在全局内存的32个4字节内存，虽然总的请求内存是128B，但因为地址跨越了N个缓存行（1 \<= N \<= 32），需要N个内存事务才能满足一条内存加载请求。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- GPU L1缓存为空间局部性而设计，而不考虑时间局部性，这个CPU L1缓存为时空局部性而设计不同。

- GPU L1缓存行size是128字节，它是流式多处理器私有的。L1缓存可以enable也可以disable。

- GPU L2缓存行size是32字节，它是跨流式多处理器的。

#### **非缓存加载**

非缓存加载不通过L1缓存，以内存段（32B）粒度而非缓存行（128B）执行。这些是更细粒度的加载，并且对于未对齐或未合并的内存访问，可以导致更好的总线利用率。

图4-14显示的是理想状态例子，对齐且合并的内存访问，请求的128B地址落入4个内存段（32B一段），总线利用率100%。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图4-15演示的例子，内存访问是对齐的，但线程访问不是顺序、而是128B范围内随机。因为每个线程请求一个独立的地址，地址落入4个段，没有加载被浪费，这类随机访问不会影响内核性能。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图4-16演示的例子，线程束请求32个连续的4B元素，但是加载未对齐到128B边界。请求的128B地址落入最多5个内存段，总线利用率最少80%。跟缓存加载相比，非缓存加载性能更高，因为加载的不被请求的字节数更少。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图4-17显示，线程束里的所有线程请求同一个4字节的数据，地址落入一个段，总线利用率是4B/32B = 12.5%，这也比缓存加载性能更高。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图4-18显示了最糟糕的场景，线程束请求32个4B的数，这些数散落在全局内存，因为请求的128B将落入最多N个32B字段，而不是N个128B缓存行，最坏情况的例子，未缓存加载相比缓存加载也有提升。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

#### **只\*\*\*\*读缓存**

只读缓存原为纹理内存加载预留的。现在，对于计算能力3.5+的GPU，只读缓存也可以作为L1缓存的一个替代用来支持全局内存加载。

通过只读缓存加载的粒度是32B，一般地，这些更好的加载粒度更有利于分散的读取。

有2个方法可以通过只读缓存去直接读内存

- 使用函数\_\_ldg

- 在指针解引用的时候使用声明修饰符（__restrict__）

### **全局内存写**

内存存储操作相对简单，费米和开普勒GPU上存操作都不使用L1缓存，存储操作在被发送到设备存储去之前，只被L2 cache缓存，存储操作的粒度粒度是32字节段。一次内存事务可以是1、2或者4内存段，也就是一次Memory transaction可以是1段，也可以2 段，也可以是4段。

例如，如果2个地址落入相同的128B区域，但不在一个对齐的64B区域，会发射1个4段事务（发射1个4段事务比发射2个1段事务执行性能好）。

### **性能调优**

优化设备内存带宽利用率，有2个努力追求的目标：

- 对齐和合并的内存访问，它可以降低带宽浪费

- 足够的并发存储访问去遮掩内存延迟

在前面一节，你学会了如何组织对齐且合并的内存访问模式，这样做将确保设备DRAM和流式多处理器片上存储/寄存器之间传输的数据的高效利用。

第三章讨论了优化核函数的指令吞吐，回顾一下，最大化内存访问是通过：

- 增加每个核函数线程里的独立访存执操作的数量

- 实验不同启动内核函数的执行配置去向流式多处理器暴露足够并行度

## **核函数能达到怎样的带宽**

分析内存性能，聚焦于内存延迟和内存带宽非常重要，内存延迟是满足独立内存请求的耗时，内存带宽指流式多处理器访问设备内存的速率（用字节/时间单元表示）。

在最后一节，你实验了2个提升内核性能的方法：

- 通过最大化并发执行的线程束的数量去遮掩内存延迟，它能通过保持更多访存在途而获得更好的总线饱和度

- 最大化内存带宽效率，通过恰当的对齐和合并内存访问

然后，有时候，手上的问题本性会有坏的访问模式，对于内核函数而言，访问模式好到什么程度才是足够好呢？在次优情况下能获得的最佳性能是什么？本节，你讲使用矩阵转置的例子去学习用各种调优技术去挑战内存带宽。

### **内存带宽**

大多数内核函数都对内存带宽敏感，也就是说，它们内存带宽bound，或者叫内存带宽密集。内核调优的时候，聚焦在内存带宽度量很重要，数据在全局内存里如何组织，线程束如何访问数据，都会影响内存带宽，有2类带宽：

- 理论带宽

- 有效带宽

理论带宽是在该硬件上能获得的最大绝对带宽，例如对于Fermi M2090（关闭ECC），理论设备内存带宽峰值是177.6GB/s，有效带宽是测量内核实际达到带宽，用下列方程：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 

### **矩阵转置问题**

通过最大化并发执行的warp数量去隐藏内存延迟。

通过恰当的对齐和合并内存访问去最大化内存带宽效率。

## **总结**

CUDA编程模型的一个显著特征是它直接向程序员暴露GPU的存储层次。向程序员提供对数据移动和放置的更多控制能力，使得你能采取更激进的优化进而获得更高的峰值性能。

全局内存的容量最大、延迟最高且被最广泛使用，对全局内存的请求，由32B或者128B的事务去满足。把这些特征和粒度记在脑海里，对于应用的全局内存调优很重要。

通过本章中的示例，您学到了以下两条提高带宽利用率的指导原则：

- 最大化在途并发内存访问的数量

- 最大化通过总线在全局内存和流式多处理片上存储之间传输的数据的利用率

为了保持足够的在途访存操作，你可以使用循环展开技术在每个线程里创建更多的独立的内存请求，或者调整网格和线程块执行配置去向流式多处理器暴露足够的并行度。

为了避免在设备内存和片上存储之间的未使用的数据移动，你的目标应该是追求理想的访问模式：对齐且合并的内存访问。

对齐内存访问相对容易，但合并内存访问有时候充满挑战，有些算法天生无法合并访问，或者不可思议的困难。

当改善合并访问时，你应该聚焦在线程束的内存访问模式。另一方面，当移除分区霸占时，你的关注点在于所有活动线程束的访问模式。对角块坐标映射是一种调整块执行顺序以避免分区霸占的方法。

统一内存通过排除重复指针和排除主机设备间显式的数据传输，极大的简化了CUDA编程，CUDA 6.0的统一内存实现，透明的维护了一致性（优先于性能）。未来的软硬件提升将改善统一内存性能。

![](https://res.wx.qq.com/mmbizappmsg/zh_CN/htmledition/js/images/icon/common/icon_avatar_default.svg)

我不想种地

别打call，打钱

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=MzU4NDY5ODU3NQ==&mid=2247486069&idx=1&sn=bd3a980986d8d2cce63a72ae249ab619&chksm=fd9495d5cae31cc32d4621fc74bb2f5e6ca18b1c95d17b8fc3065ea8377689fba94cf533dc27&mpshare=1&scene=24&srcid=0327ZsoCaw9WBolNVkWkoRy5&sharer_shareinfo=09372f8ba15e03f43cde636ea5502f0a&sharer_shareinfo_first=09372f8ba15e03f43cde636ea5502f0a&key=daf9bdc5abc4e8d0df75c609275c20f1edb3d7daec1ced4aac14208398675b60e256772c3c78fddda10137a304c60700f4d137e753f22dc5c13ac966027091b04d13c79636fbe031487630caa409cf8ac125c83c82361d03da0b80b71539014f458d6a93042985fca9bdb5481a0d6743101cb5d0466228afd60efbe87991956b&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQoskCLWWT8wKuGem6aoEsTRLTAQIE97dBBAEAAAAAAPFPLKYsM5QAAAAOpnltbLcz9gKNyK89dVj02zBEzExlnusOMACD0jYBatf5AnVhaFroL8J482L%2B7y%2BhRNIIsM4QoyZZV2ONiRyCYoftmiho3qc78Yueb5lh%2BTN2Bl%2BSjrYogW0iASFYz3t49SCIM%2FbRTq8UxyyOOKgSf3fYsgXbTfxnFF0TP%2Fx6kXiHEbsV6tMzwzn45xPKeORjsceVSKgj7kcLopnw8Awk5i5ahFoe15mxC9AcpLKwLgSpEezjw5dKGg%2Fxd3E%3D&acctmode=0&pass_ticket=nN67s6LSCf6rPSkxTx3ijwR96T6eV%2B7DQI7f%2BtOUkaqRJZactdFcAxQkeKqaht2l&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1)钟意作者

阅读 598

​

发消息

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/pg57MH7pFC5SmXunUYSRovoeKZOqGl5TlL0pcGLNCtf9dxQX6SLaUIH2aJj76yEf9IhQAzDe0jCncyyaicT94bg/300?wx_fmt=png&wxfrom=18)

码砖杂役

2681

发消息

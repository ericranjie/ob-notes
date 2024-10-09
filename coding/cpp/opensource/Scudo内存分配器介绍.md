# 

原创 Little fish OPPO内核工匠

_2021年12月31日 17:00_

## 

## **一、背景**

早期Android中使用jemalloc作为默认的Native内存分配器，但是从R开始，Scudo替代jemalloc成为了non-svelte configuration模式下默认的内存分配器（svelte模式下默认的内存分配器依然是jemalloc）。

随着64位机器和大RAM的普及，虚拟内存和物理内存的瓶颈都在不断放宽，因此给了系统更多的选择，可以在性能合理的范围内兼顾其他特性。在所有安全性问题中，内存漏洞发生的入侵占到了半数以上，因此如果能在Allocator中抵御入侵，那将极大地降低安全问题的数量，Scudo也由此而引入。

## 

## 

## **二、设计实现**

Scudo的设计考虑到了安全性，但目的是在安全性和性能之间取得良好的平衡。单从性能角度分析Scudo未必超过jemalloc，虽然它的分配策略更加简化，但为了安全性所实施的一些策略会使其丧失一些性能。

### 

### 

### **1. Scudo组件**

Scudo分配器主要由Primary、Secondary、TSD、Quarantine四个组件构成。

\*\*Primary Allocator:\*\*它通过将预留内存区域分成相同大小的块来更快速高效得分配较小的内存块。目前实现了两个Primary分配器，分别针对32位和64位体系结构。它可以通过编译时选项进行配置。

对于64位andorid R/S，Primary Allocator如图1所示。在初始化时会mmap出256M\*33大小空间，共分为33段regions，分别通过classid 0~32标记。每段region大小为256M，且出于安全考虑其头部会随机空缺处1~16页。此外，每段region中再分为特定大小的内存块，如class 1 region分为32Bytes，class 2 region分为48Bytes，class 32 region为64K等（class 0 region用于存放内存管理元数据）。如此当分配小内存时，首先会检查合适大小的region中是否有空闲内存块，如果没有则去更高一级region中分配，当最高一级中也没有合适的，则会从Secondary Allocator中分配。

![图片](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjMwpX8vYFol0NQFTk5KChBh2Eddy3ibxibr34Lzqw5IRxicCEOFAdhdPpndwTD9bHcfDZo5e2R7xshQw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

此外，Primary Allocator提供了cache机制来加速内存分配。当线程分配内存时，会通过线程TSD中SizeClassAllocatorLocalCache对象的chunks数组来寻找合适的空闲内存。但chunks数组的大小是有限的(默认28个)，当它们用完时就需要补充空闲内存块了。补充是从region的freelist（freelist的详细数据存放于regioninfo）中直接获取。当freelist中的空闲对象不够时，会扩张region的空闲区域。

\*\*Secondary Allocator:\*\*相对于Primary更慢，它通过底层操作系统的内存映射来分配更大的内存。并且通过Secondary分配的内存块两端被保护页包围。

对于64位andorid R/S，Secondary Allocator主要用于分配大于64K内存，其直接使用mmap分配出一块新的VMA。为了加快分配速度，同时设计了相应Cache（MapAllocatorCache），内部最多可以缓存32个不超过2M的VMA。

![图片](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjMwpX8vYFol0NQFTk5KChBhw8C8LbAQCaeIyHlEBFSWeU4dEsLETz39gJXYSQDEABDdBmNRWxEibiaA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

**TSD:** 定义了每个线程的本地缓存如何操作。目前有两种模型实现:独占模型，其中每个线程拥有自己的缓存;或者共享模型，其中线程共享一个固定大小的缓存池。

对于64位andoridR/S，使用共享模型，且TSD pool只有两个TSD对象，每个TSD对象分别含有一个SizeClassAllocatorLocalCache和QuarantineCache对象。

![图片](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjMwpX8vYFol0NQFTk5KChBhmwD9HBeYWribkmQId2icqibJSjUsicVKz3NVQ39Mc36Shst622oGRZmOiaA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

**Qurantine**: 提供延迟释放内存的方法，防止内存块立即再分配。一旦达到一定的大小标准，内存块将被回收。这本质上是一个延迟的空闲链表，它可以帮助缓解一些释放后使用的情况。这个特性在性能和内存占用方面是相当昂贵的，主要由运行时选项控制，默认情况下是禁用的。

如果配置了Quarantine，那么内存释放的时候符合大小限制的block会被暂时隔离，状态设置为Quarantined，而不是Available，可以检测UAF。首先尝试放到线程TSD对应的QuarantineCache中，如果local quarantine cache size超标了，则把localquarantine cache中的内存块全部放到前端中的global quarantine cache中。如果global quarantine cache也超标了，则recycle释放到TSD->SizeClassAllocatorLocalCache。

![图片](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjMwpX8vYFol0NQFTk5KChBhHGrfibZMEydl9swH3go1hYQT4nv2xyPFmOkBseIibRMEJKvt3jcLiaXIQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

### **2. Chunk Header**

下面是chunkheader的详细信息，后面的数字代表每个字段所占用的bit，加起来为64bits，也即8字节。

![图片](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjMwpX8vYFol0NQFTk5KChBhRhh86y5VY2d2qeAPBlWlKzjwSfWiaakA2xEytj1R6NtzC347GgiaJ8ibg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

ClassId：表示内存块分配自Primary中的region id，ClassId为0表示从Secondary中分配。

State：表示内存块当前状态，0 Available，1 Allocated，2 Quarantined。

OriginOrWasZeroed：当State为allocated时，表示通过哪种方式发生的分配，譬如是new或malloc。

SizeOrUnusedBytes：当ClassId为正数指分配的size大小，当ClassId为0时表示未使用的字节大小。

Offset：内存块中chunk header的偏移。

Checksum：校验和，用于检测chunk header是否被破坏。

在一个内存地址通过free/delete释放时，该地址需要经过重重检测，以保证它在使用过程中是未经破坏的。下面按时间顺序列举出一个chunk需要经过的检测。

1）alignment检测：地址必须16字节对齐，如果是一个未经对齐的long型数字被当成了指针，这里就可以检测出来misaligned pointer错误。

![图片](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjMwpX8vYFol0NQFTk5KChBhofWqibagQmwbjRso16gdAUOnupJPibXR2Unqbn8P17TPNf9UmAicL2icGA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

2）checksum检测：checksum数字在deallocate时会再计算一遍，和chunk header中保存的checksum进行比较。如果二者不相等，则会报corrupted chunkheader错误。

![图片](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjMwpX8vYFol0NQFTk5KChBhGUWYIfCibB1af6IBNCTKbjkxRzhWtGyTn0XzgPcQc3icPeWqgTVGGEpQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

3）state检测：如果chunk header的state不为Allocated，表明此时不应该释放这块内存，这很有可能是一个double-free，会报invalid chunk state错误。

![图片](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjMwpX8vYFol0NQFTk5KChBhPIvtPLvPNRwF2icmKf7tkdD4MDK1tVsCGEvwpgicjxBpmSERPAttzaTw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

4）type检测：如果分配时的方法和释放的方法不匹配，会报allocationtype mismatch错误 (前提是打开DeallocTypeMismatch选项)。

![图片](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjMwpX8vYFol0NQFTk5KChBh1EYe9bdq9pMCTO2c2MYuPpdI9ngHaK1AMcKDhS1oZ4IyV0t2k5QwWg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

5）size检测：如果释放时的size和chunk header中的size不相等，会报invalid sized delete错误（前提是打开DeleteSizeMismatch选项）。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

以上内存释放时的检测环节都可以和AndroidScudo官方文档中对于典型错误对应起来，以下是Android Scudo官方文档中对于典型错误信息的分析：

1）corrupted chunk header：区块头的校验和验证失败。可能原因有二：区块头被部分或全部覆盖，也可能是传递给函数的指针不是区块。

2）race on chunk header：两个不同的线程会同时尝试操控同一区块头。这种症状通常是在对该区块执行操作时出现争用情况或通常未进行锁定造成的。

3）invalid chunk state：对于指定操作，区块未处于预期状态，例如，在尝试释放区块时其处于未分配状态，或者在尝试回收区块时其未处于隔离状态。双重释放是造成此错误的典型原因。

4）misaligned pointer：强制执行基本对齐要求：32 位平台上为 8 个字节，64 位平台上为 16 个字节。如果传递给函数的指针不适合这些函数，传递给其中一个函数的指针就不会对齐。

5）allocation type mismatch：启用此选项后，在区块上调用的取消分配函数必须与用于分配区块而调用的函数类型一致。类型不一致会引发安全问题。

6）invalid sized delete：如果使用的是符合 C++14 标准的删除运算符，在启用可选检查之后，取消分配区块时传递的大小与分配区块时请求的大小会出现不一致的情况。这通常是由于编译器出现问题或是对要取消分配的对象产生了类型混淆。

7）RSS limit exhausted：已超出选择性指定的 RSS 大小上限。

### **3. 内存分配流程**

Scudo中内存分配流程如下图所示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

1、  根据入参size计算真正需要从Allocator分配的NeededSize（入参size对齐到alignment再加上alignment和chunk headersize中较大的），如果NeededSize超过Primary Allocator最大的size class就从Secondary Allocator分配，走步骤2，否则走步骤4。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

2、  从Secondary Allocator分配：对NeededSize+LargeBlock Header Size按照PageSize大小对齐得到RoundedSize。如果RoundedSize能从MapAllocatorCache中获取成功，则直接获取MapAllocatorCache中的内存块，走步骤6。否则走步骤3

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

3、  需要mmap大小为RoundedSize + 2Page的VM，前后各一个Page，作用类似RedZone，注意这次mmap的权限是PROT_NONE。之后会skip一个Page大小，再次以RW方式重新map一次，大小为RoundedSize。最后跳过LargeBlock Header，获取内存块地址，继续走步骤6。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

4、  从Primary Allocator分配：Primary管理着每个size class的Region，分配时先获取线程对应的TSD中的SizeClassAllocatorLocalCache对象，尝试从SizeClassAllocatorLocalCache其中获取对应size class的空闲内存块，若SizeClassAllocatorLocalCache中存在指定size class的空闲内存块，则获取内存块地址，走步骤6。

5、  如果TSD中的SizeClassAllocatorLocalCache没有指定size class的空闲内存块，则再回到Primary中去从Region中以RW（Region初始化过程虽然已经map，但是为PROT_NONE）再次map适当大小的内存，填充到对应region的freelist中（以TranserBatch为node），并从free list中取一个TransferBatch填充给SizeClassAllocatorLocalCache，SizeClassAllocatorLocalCache被填充后就可以取一个空闲内存块，继续下一步。如果当前region中空闲内存块已全部使用完，没有则去更高一级region中分配，当最高一级中也没有合适的，则会从Secondary Allocator中分配。

6、获取到空闲内存块后，继续填充chunk header，并跳过chunk header返回内存地址给用户。

### 

### 

### **4. 内存释放流程**

Scudo中内存释放流程如下图所示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

1、获取内存块的chunk header，进行相关检查，如：会做checksum检查，以及分配类型是否匹配：malloc/free，delete/new， 再根据入参的ptr获得用户分配的size大小做delete size mismatch检查等。

2、释放内存有两种去向，即是否需要进行Quarantine，如果需要则进行步骤3进行Quarantine，否则释放回Primary或Secondary, 走步骤4。

3、如果配置了Quarantine，并且释放的时候符合大小限制的内存块会被暂时隔离，状态设置为Quarantined，而不是Available，可以检测UAF。首先尝试放到线程TSD对应的QuarantineCache对象（local quarantine cache size）中，如果localquarantine cache size超标了，则把local quarantine cache中的内存块全部放到全局global quarantine cache中。如果global quarantinecache也超标了，则回收释放到Primary中。

4、通过chunk header的classid决定释放到Primary还是Secondary中。class id = 0 代表是从Secondary Allocator分配。

5、对于Secondary Allocator，如果释放的内存块小于2M，则先尝试放到MapAllocatorCache的CachedBlock数组中，如果成功放入数组，会将当前时间作为这个内存块释放的时间，还会根据配置的gc时间间隔将数组中老化时间超过gc间隔的内存块释放掉（通过madvise(MADV_DONTNEED)而不是unmap）。另外一种情况如果在放回CachedBlock数组的时候发现数组满了，则会unmap数组中的所有内存块（累计发生4次数组满的情况才会清空），同时当前被释放的block也会被unmap。

6、对于Primary Allocator，首先还是拿到当前线程对应的TSD并获取TSD中的SizeClassAllocatorLocalCache对象，如果SizeClassAllocatorLocalCache不满则直接放到cache中，free到此结束。如果SizeClassAllocatorLocalCache满了，则将cache中一半数量的缓存内存块以TransferBatch为载体返回给Primary对应Region的freelist中，之后会判断是否需要做madvise释放freelist中的空闲内存块占用的pss，判断依据主要有当前时间与上次释放pss的间隔时间是否足够，以及freelist中的内存块大小是否足够大，至少要达到一个page等。最后再将当前被释放的内存块放到SizeClassAllocatorLocalCache中。

## 

## 

## **三、Scudo常用配置**

Scudo被设计为高度可调和可配置的，虽然提供了一些默认配置，但鼓励用户提出最适合他们用例的参数。

如Android Scudo官方文档所描述，可以通过以下几种方式针对各进程定义分配器的一些参数：

a、在编译时，通过将SCUDO_DEFAULT_OPTIONS定义为默认的选项字符串。

b、静态：在程序中定义 \_\_scudo_default_options函数（返回要解析的选项字符串）。该函数必须具有以下原型：extern "C" constchar \*\_\_scudo_default_options()。以这种方式定义的选项会替换编译时定义的选项。

例：

extern"C" const char \*\_\_scudo_default_options()

{return "delete_size_mismatch=false:release_to_os_interval_ms=-1"; }

c、动态：使用环境变量 SCUDO_OPTIONS（包含要解析的选项字符串）。以这种方式定义的选项会替换通过 \_\_scudo_default_options 定义的选项。

例：

SCUDO_OPTIONS="delete_size_mismatch=false:release_to_os_interval_ms=-1"./a.out

d、通过标准的mallopt API，使用Scudo特有的参数。

主要可以使用以下选项：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

下面是可用的" mallopt "选项:

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 

## 

## 参考文档

https://llvm.org/docs/ScudoHardenedAllocator.htmlhttps://source.android.com/devices/

tech/debug/scudohttps://zhuanlan.zhihu.com/p/235620563?utm_source=ZHShareTargetIDMore

https://zhuanlan.zhihu.com/p/353784014

https://juejin.cn/post/6914550038140026887

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**长按关注**

**内核工匠微信**

Linux 内核黑科技 | 技术文章 | 精选教程

阅读 1416

​

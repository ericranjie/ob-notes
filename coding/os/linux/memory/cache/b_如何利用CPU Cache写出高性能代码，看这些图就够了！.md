人人极客社区
_2021年12月30日 08:20_
以下文章来源于程序喵大人 ，作者程序喵大人

\](https://mp.weixin.qq.com/s?\_\_biz=MzIxMjE1MzU4OA==&mid=2648928335&idx=1&sn=07439a08e7fbbfe389fecf71bfd9ecde&chksm=8f5db5d4b82a3cc2b16e8c6c5f591c0d08ff3e82c2fc78e3f128da91e36cf9bc62f8aa4d3cea&mpshare=1&scene=24&srcid=1231PBwwIZlWUVAzkDm4KAl0&sharer_sharetime=1640948016855&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d02c3a61f6eb02bc648ed0fe9ad58e35c8f01d5ca228fa9c1190bef6f948ced075043c756ac6cce813ba525d5c7f9d4da23ee39fc7055fec302412f8ee7d9ad8e8c1461e74bdef41cf91925fdcadaff4943532f737a1fc595193d62a15f0ab5565ea33833030d8ced37b4ab1a112b4459ad4eb24c048cf6676&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQs2f2qDtIcA633M9WzsKsGhLmAQIE97dBBAEAAAAAAKFJL3DQOxoAAAAOpnltbLcz9gKNyK89dVj0lvAanBCgPH4O7wjDults0LvCSjv29CoFa8XYEDz8ZF2kDdCTHXnmB5VHldJZYX8DgoLujn2p6HDIfKXk4Za8gnpgtCSLYkJvLDplwVVRVl5fZnpJ1D3ZNLeKHLXjk0f%2B1A2B1LlUr%2FBXb0BM4Vje33Ytw2X65PQMxMfgFMIpusAuFq7rpg71Awo2w2Ywjg%2F9CYx65kGtWR0iJepk9T0OEUi7CUqe7JeNY3N3WCTQPMMgc9%2FsRNxx03B%2BxpY5vzor&acctmode=0&pass_ticket=cfcBFferA%2FTPEqB3aNP2zras5ceUHS%2F%2Fe25GFytkKaWaVEmP4%2BJbCmGiN90OzIUf&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1#)

我们平时编写的代码最后都会交给CPU来执行，如何能巧妙利用CPU写出性能比较高的代码呢？看完这篇文章您可能会有所收获。

先提出几个问题，之后我们逐个击破：

![图片](https://mmbiz.qpic.cn/mmbiz_gif/9lFFFiaKpEriceh1CAmQFpQSKm0aSAgt5QthMQibTKnEzmgxPmc2wnNlbD1zbYhP2XJnCTxgQfic0ibVbFicMpJze18g/640?wx_fmt=gif&wxfrom=13&tp=wxpic)

初入宝地：什么是CPU Cache？

连类比事：为什么要有Cache？为什么要有多级Cache？

挑灯细览：Cache的大小和速度如何？

追本溯源：什么是Cache Line？

旁枝末叶：什么是Cache一致性？

触类旁通：Cache与主存的映射关系如何？

更上一层：如何巧妙利用CPU Cache编程？

**1. 什么是CPU Cache？**

如图所示：

![图片](https://mmbiz.qpic.cn/mmbiz_png/9lFFFiaKpEribOicpWLdXXwnN18ia5Ew0ee0lAjfibwKV4tufhhUX185KpTqZKlWS5YbjM83OYnTibJHF4XJc7InPLicw/640?wx_fmt=png&wxfrom=13&tp=wxpic)

CPU Cache可以理解为CPU内部的高速缓存，当CPU从内存中读取数据时，并不是只读自己想要的那一部分，而是读取更多的字节到CPU高速缓存中。当CPU继续访问相邻的数据时，就不必每次都从内存中读取，可以直接从高速缓存行读取数据，而访问高速缓存比访问内存速度要快的多，所以速度会得到极大提升。

**2. 为什么要有Cache？为什么要有多级Cache？**\
!\[\[Pasted image 20240913160716.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

为什么要有Cache这个问题想必大家心里都已经有了答案了吧，CPU直接访问距离较远，容量较大，性能较差的主存速度很慢，所以在CPU和内存之间插入了Cache，CPU访问Cache的速度远高于访问主存的速度。

CPU Cache是位于CPU和内存之间的临时存储器，它的容量比内存小很多但速度极快，可以将内存中的一小部分加载到Cache中，当CPU需要访问这一小部分数据时可以直接从Cache中读取，加快了访问速度。

想必大家都听说过程序局部性原理，这也是CPU引入Cache的理论基础，程序局部性分为时间局部性和空间局部性。时间局部性是指被CPU访问的数据，短期内还要被继续访问，比如循环、递归、方法的反复调用等。空间局部性是指被CPU访问的数据相邻的数据，CPU短期内还要被继续访问，比如顺序执行的代码、连续创建的两个对象、数组等。因为如果将刚刚访问的数据和相邻的数据都缓存到Cache时，那下次CPU访问时，可以直接从Cache中读取，提高CPU访问数据的速度。
!\[\[Pasted image 20240913160723.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

一个存储器层次大体结构如图所示，速度越快的存储设备自然价格也就越高，随着数据访问量的增大，单纯的增加一级缓存的成本太高，性价比太低，所以才有了二级缓存和三级缓存，他们的容量越来越大，速度越来越慢（但还是比内存的速度快），成本越来越低。

**3. Cache的大小和速度如何？**

!\[\[Pasted image 20240913160730.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

通常越接近CPU的缓存级别越低，容量越小，速度越快。不同的处理器Cache大小不同，通常现在的处理器的L1 Cache大小都是64KB。

那CPU访问各个Cache的速度如何呢？
!\[\[Pasted image 20240913160738.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

如图所示，级别越低的高速缓存，CPU访问的速度越快。

CPU多级缓存架构大体如下：
!\[\[Pasted image 20240913160743.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

L1 Cache是最离CPU最近的，它容量最小，速度最快，每个CPU都有L1 Cache，见上图，其实每个CPU都有两个L1 Cache，一个是L1D Cache，用于存取数据，另一个是L1I Cache，用于存取指令。

L2 Cache容量较L1大，速度较L1较慢，每个CPU也都有一个L2 Cache。L2 Cache制造成本比L1 Cache更低，它的作用就是存储那些CPU需要用到的且L1 Cache miss的数据。

L3 Cache容量较L2大，速度较L2慢，L3 Cache不同于L1 Cache和L2 Cache，它是所有CPU共享的，可以把它理解为速度更快，容量更小的内存。

当CPU需要数据时，整体流程如下：
!\[\[Pasted image 20240913160750.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

会最先去CPU的L1 Cache中寻找相关的数据，找到了就返回，找不到就去L2 Cache，再找不到就去L3 Cache，再找不到就从内存中读取数据，寻找的距离越长，自然速度也就越慢。

**4. Cache Line？**

Cache Line可以理解为CPU Cache中的最小缓存单位。Main Memory-Cache或Cache-Cache之间的数据传输不是以字节为最小单位，而是以Cache Line为最小单位，称为缓存行。

目前主流的Cache Line大小都是64字节，假设有一个64K字节的Cache，那这个Cache所能存放的Cache Line的个数就是1K个。

**5. 写入策略**

Cache的写入策略有两种，分别是**WriteThrough（直写模式）**和**WriteBack（回写模式）**。

\*\*直写模式：\*\*在数据更新时，将数据同时写入内存和Cache，该策略操作简单，但是因为每次都要写入内存，速度较慢。

\*\*回写模式：\*\*在数据更新时，只将数据写入到Cache中，只有在数据被替换出Cache时，被修改的数据才会被写入到内存中，该策略因为不需要写入到内存中，所以速度较快。但数据仅写在了Cache中，Cache数据和内存数据不一致，此时如果有其它CPU访问数据，就会读到脏数据，出现bug，所以这里需要用到Cache的一致性协议来保证CPU读到的是最新的数据。

**6. 什么是Cache一致性呢？**

多个CPU对某块内存同时读写，就会引起冲突的问题，被称为Cache一致性问题。

有这样一种情况：

**a.**   CPU1读取了一个字节offset，该字节和相邻的数据就都会被写入到CPU1的Cache.

**b.**   此时CPU2也读取相同的字节offset，这样CPU1和CPU2的Cache就都拥有同样的数据。

**c.**   CPU1修改了offset这个字节，被修改后，这个字节被写入到CPU1的Cache中，但是没有被同步到内存中。

**d.**   CPU2 需要访问offset这个字节数据，但是由于最新的数据并没有被同步到内存中，所以CPU2 访问的数据不是最新的数据。

这种问题就被称为Cache一致性问题，为了解决这个问题大佬们设计了MESI协议，当一个CPU1修改了Cache中的某字节数据时，那么其它的所有CPU都会收到通知，它们的相应Cache就会被置为无效状态，当其他的CPU需要访问此字节的数据时，发现自己的Cache相关数据已失效，这时CPU1会立刻把数据写到内存中，其它的CPU就会立刻从内存中读取该数据。

MESI协议是通过四种状态的控制来解决Cache一致性的问题：

■ \*\*M：\*\*代表已修改（Modified） 缓存行是脏的（dirty），与主存的值不同。如果别的CPU内核要读主存这块数据，该缓存行必须回写到主存，状态变为共享（S）.

■ \*\*E：\*\*代表独占（Exclusive） 缓存行只在当前缓存中，但是干净的（clean）--缓存数据同于主存数据。当别的缓存读取它时，状态变为共享（S）；当前写数据时，变为已修改（M）状态。

■ \*\*S：\*\*代表共享（Shared） 缓存行也存在于其它缓存中且是干净（clean）的。缓存行可以在任意时刻抛弃。

■ \*\*I：\*\*代表已失效（Invalidated） 缓存行是脏的（dirty），无效的。

四种状态的相容关系如下：
!\[\[Pasted image 20240913160805.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这里我们只需要知道它是通过这四种状态的切换解决的Cache一致性问题就好，具体状态机的控制实现太繁琐，就不多介绍了，这是状态机转换图，是不是有点懵。
!\[\[Pasted image 20240913160812.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**7. Cache与主存的映射关系？**

**直接映射**
!\[\[Pasted image 20240913160818.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

直接映射如图所示，每个主存块只能映射Cache的一个特定块。直接映射是最简单的地址映射方式，它的硬件简单，成本低，地址转换速度快，但是这种方式不太灵活，Cache的存储空间得不到充分利用，每个主存块在Cache中只有一个固定位置可存放，容易产生冲突，使Cache效率下降，因此只适合大容量Cache采用。
!\[\[Pasted image 20240913160853.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

例如，如果一个程序需要重复引用主存中第0块与第16块，最好将主存第0块与第16块同时复制到Cache中，但由于它们都只能复制到Cache的第0块中去，即使Cache中别的存储空间空着也不能占用，因此这两个块会不断地交替装入Cache中，导致命中率降低。
!\[\[Pasted image 20240913160859.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

直接映射方式下主存地址格式如图，主存地址为s+w位，Cache空间有2的r次方行，每行大小有2的w次方字节，则Cache地址有w+r位。通过Line确定该内存块应该在Cache中的位置，确定位置后比较标记是否相同，如果相同则表示Cache命中，从Cache中读取。

**全相连映射**
!\[\[Pasted image 20240913160904.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

全相连映射如图所示，主存中任何一块都可以映射到Cache中的任何一块位置上。

全相联映射方式比较灵活，主存的各块可以映射到Cache的任一块中，Cache的利用率高，块冲突概率低，只要淘汰Cache中的某一块，即可调入主存的任一块。但是，由于Cache比较电路的设计和实现比较困难，这种方式只适合于小容量Cache采用。
!\[\[Pasted image 20240913160909.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

全相连映射的主存结构就很简单啦，将CPU发出的内存地址的块号部分与Cache所有行的标记进行比较，如果有相同的，则Cache命中，从Cache中读取，如果找不到，则没有命中，从主存中读取。

**组相连映射**
!\[\[Pasted image 20240913160913.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

组相联映射实际上是直接映射和全相联映射的折中方案，其组织结构如图3-16所示。主存和Cache都分组，主存中一个组内的块数与Cache中的分组数相同，组间采用直接映射，组内采用全相联映射。也就是说，将Cache分成u组，每组v块，主存块存放到哪个组是固定的，至于存到该组哪一块则是灵活的。例如，主存分为256组，每组8块，Cache分为8组，每组2块。

主存中的各块与Cache的组号之间有固定的映射关系，但可自由映射到对应Cache组中的任何一块。例如，主存中的第0块、第8块……均映射于Cache的第0组，但可映射到Cache第0组中的第0块或第1块；主存的第1块、第9块……均映射于Cache的第1组，但可映射到Cache第1组中的第2块或第3块。
!\[\[Pasted image 20240913160918.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

常采用的组相联结构Cache，每组内有2、4、8、16块，称为2路、4路、8路、16路组相联Cache。组相联结构Cache是前两种方法的折中方案，适度兼顾二者的优点，尽量避免二者的缺点，因而得到普遍采用。
!\[\[Pasted image 20240913160926.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

组相连映射方式下的主存地址格式如图，先确定主存应该在Cache中的哪一个组，之后组内是全相联映射，依次比较组内的标记，如果有标记相同的Cache，则命中，否则不命中。

在网上找到了三种映射方式下的主存格式对比图，大家也可以看下：
!\[\[Pasted image 20240913160931.png\]\]
!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**8. Cache的替换策略？**

Cache的替换策略想必大家都知道，就是LRU策略，即最近最少使用算法，选择未使用时间最长的Cache替换。

**9. 如何巧妙利用CPU Cache编程？**

```c
const int row = 1024; const int col = 1024; int matrix[row][col];  //按行遍历
int sum_row = 0; for (int r = 0; r < row; r++) {     for (int c = 0; c < col; c++) {         sum_row += matrix[r][c];     } }  //按列遍历
int sum_col = 0; for (int c = 0; c < col; c++) {     for (int r = 0; r < row; r++) {         sum_col += matrix[r][c];     } }
```

上面是两段二维数组的遍历方式，一种按行遍历，另一种是按列遍历，乍一看您可能认为计算量没有任何区别，但其实按行遍历比按列遍历速度快的多，这就是CPU Cache起到了作用，根据程序局部性原理，访问主存时会把相邻的部分数据也加载到Cache中，下次访问相邻数据时Cache的命中率极高，速度自然也会提升不少。

平时编程过程中也可以多利用好程序的时间局部性和空间局部性原理，就可以提高CPU Cache的命中率，提高程序运行的效率。

**参考链接**

> https://cseweb.ucsd.edu/classes/fa14/cse240A-a/pdf/08/CSE240A-MBT-L15-Cache.ppt.pdf

> http://www.ecs.csun.edu/~cputnam/Comp546/Putnam/Cache%20Memory.pdf

> https://ece752.ece.wisc.edu/lect11-cache-replacement.pdf

> http://www.ipdps.org/ipdps2010/ipdps2010-slides/session-22/2010IPDPS.pdf

> https://faculty.tarleton.edu/agapie/documents/cs_343_arch/04_CacheMemory.pdf

> https://www.cnblogs.com/bjlhx/p/10658938.html

> https://zh.wikipedia.org/wiki/MESI%E5%8D%8F%E8%AE%AE

> https://www.cnblogs.com/yanlong300/p/8986041.html

> https://coolshell.cn/articles/20793.html

> https://zhuanlan.zhihu.com/p/31875174

> https://www.cnblogs.com/jokerjason/p/10711022.html

> http://cenalulu.github.io/linux/all-about-cpu-cache/

> https://my.oschina.net/fileoptions/blog/1630855

> https://www.pianshen.com/article/9557956769/

> https://blog.csdn.net/weixin_42649617/article/details/105092395

文中图片来自网络

往期推荐

[!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MzI3NjA1OTEzMg==&mid=2247484234&idx=1&sn=40df2edd1dd42d95cb1e56752d471424&chksm=eb7a05d9dc0d8ccf6343ea977bd0fecf539a1b0885eda2545bd1939e07e40453f021161da03c&scene=21#wechat_redirect)

[!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MzI3NjA1OTEzMg==&mid=2247484131&idx=1&sn=c49df1d90d5b4daf7059f443038d3b7b&chksm=eb7a0470dc0d8d6672caaf461dd3310c921f0dabd08186ea6940c435be0602ba52210070e192&scene=21#wechat_redirect)

[!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MzI3NjA1OTEzMg==&mid=2247483885&idx=1&sn=ba3342f7807d347747946df4b9bb4b4c&chksm=eb7a077edc0d8e68f9bde2e39e64e18f5c2e2d85bd9abeb44986fe64a8c7dcbceb5333b78298&scene=21#wechat_redirect)

[!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MzI3NjA1OTEzMg==&mid=2247483996&idx=1&sn=83a15a7e7ce6a3999920e5ea8bb454b6&chksm=eb7a04cfdc0d8dd9d7e0400fd251e486ead533617e90d1c88fc747c0ba0c5a55e2e754f9ccd0&scene=21#wechat_redirect)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

阅读 2670

​

写留言

**留言 4**

- colapapa

  2021年12月30日

  赞2

  寄存器不在cache中，cache不能被寻址，cache对程序员是透明的。cache只是memory的一小块映射，cpu访问的地址首先会进入cache判断是否hit，如果hit,cache直接返回数据，否则需要总线上去访问memory。

- 辰风送暖

  2022年1月12日

  赞

  CPU读了一段数据后会将它放在哪级cache里？不同级cache之间是包含关系吗？

  人人极客社区

  作者2022年1月12日

  赞

  l2和l3，一层一层来

- 1男

  2021年12月30日

  赞

  学习汇编中，几个问题，寄存器不是在cache里面的吧，所以cache有地址吗？汇编指令去读取某块内存地址，cpu又如何知道cache和内存是如何map的。纯小白的弱智问题，请大佬解答，谢谢。

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/9sNwsXcN68pL55XIyzTrCHZTbIUdTibQcuzuCaYeGTXNMyn6ACmicUrpoDC0xZSap46XJ59sKysPg9Rg379f32cA/300?wx_fmt=png&wxfrom=18)

人人极客社区

14317

4

写留言

**留言 4**

- colapapa

  2021年12月30日

  赞2

  寄存器不在cache中，cache不能被寻址，cache对程序员是透明的。cache只是memory的一小块映射，cpu访问的地址首先会进入cache判断是否hit，如果hit,cache直接返回数据，否则需要总线上去访问memory。

- 辰风送暖

  2022年1月12日

  赞

  CPU读了一段数据后会将它放在哪级cache里？不同级cache之间是包含关系吗？

  人人极客社区

  作者2022年1月12日

  赞

  l2和l3，一层一层来

- 1男

  2021年12月30日

  赞

  学习汇编中，几个问题，寄存器不是在cache里面的吧，所以cache有地址吗？汇编指令去读取某块内存地址，cpu又如何知道cache和内存是如何map的。纯小白的弱智问题，请大佬解答，谢谢。

已无更多数据

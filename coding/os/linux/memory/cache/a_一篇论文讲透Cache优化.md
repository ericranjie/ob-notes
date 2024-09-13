![](https://zhuanlan.zhihu.com/p/608663298)

[](https://www.zhihu.com/)

首发于[分布式和存储的那些事](https://www.zhihu.com/column/distributed-storage)

写文章

![点击打开planet-frontier的主页](https://picx.zhimg.com/v2-abed1a8c04700ba7d72b45195223e0ff_l.jpg?source=32738c0c&needBackground=1)

![一篇论文讲透Cache优化](https://picx.zhimg.com/70/v2-8a4e110041f8c1667958caa2abb119bd_1440w.awebp?source=172ae18b&biz_tag=Post)

# 一篇论文讲透Cache优化

[![红星闪闪](https://picx.zhimg.com/v2-7e1fc938ae12f5dce67d46b570d99ded_l.jpg?source=172ae18b)](https://www.zhihu.com/people/hong-xing-shan-shan-17-75)

[红星闪闪](https://www.zhihu.com/people/hong-xing-shan-shan-17-75)

数据库资深工程师。在后台开发，高性能计算，分布式领域深耕。

已关注

韦易笑 等 1353 人赞同了该文章

《What Every Programmer Should Know About Memory》是Ulrich Drepper大佬的一篇神作，洋洋洒洒100多页，基本上涵盖了当时（2007年）关于访存原理和优化的所有问题。即使今天的CPU又有了进一步的发展，但是依然没有跳出这篇文章的探讨范围。只要是讨论访存优化的文章，基本上都会引用这篇论文。

这篇文章也是常读常新的，不同出发点和不同阶段的同学看这篇文章都会有自己的收获。刚入职的时候读了一遍，初窥门径，最近团队的论文分享又读了一遍，有了进一步的理解，这里简单梳理了一下这篇文章自己感兴趣的知识点，初步解答了访存的基本原理，访存面临的主要问题[和解法](https://zhida.zhihu.com/search?q=%E5%92%8C%E8%A7%A3%E6%B3%95&zhida_source=entity&is_preview=1)，希望对大家有所帮助。更重要的是，希望能激起大家阅读这篇论文的兴趣。

## 一 原理

## 1.1 Cache架构

  

![](https://pica.zhimg.com/80/v2-db76f92d9a8a56c74d1f64ef1f90e764_1440w.jpg)

  

  

![](https://pic2.zhimg.com/80/v2-9771fed8a54f17697f91b4dc9f7ad713_1440w.webp)

  

- 浅灰色代表一个Processor，代表一个CPU的物理封装，通过总线和内存相连。
- 深灰色代表一个物理Core：

- 每个Core中有两个Hyper-Thread，常见于Intel的CPU，每个H-Thread对上抽象出一个Core，但并没有物理上的拓展，硬件级别进行[时间片](https://zhida.zhihu.com/search?q=%E6%97%B6%E9%97%B4%E7%89%87&zhida_source=entity&is_preview=1)并发。优势在于，当一个H-Thread执行内存Stall的指令时，可以切换成另一个H-Thread，并不阻塞。
- 每个Core有一个L1d和一个L1i。两个H-Thread共享通过一个L1指令和数据Cache。

- L2和L3被多个Core共享

  

真实场景：

  

![](https://pic4.zhimg.com/80/v2-a5ea71f84ae7486c2e1e3a20059b023b_1440w.webp)

  

1个物理CPU，里面有4个core，每个core有两个H-Thread。对上层一共暴露1*4*2=8个CPU

  

![](https://pic1.zhimg.com/80/v2-49ec26a955bc8fa702d834ff54e50dfc_1440w.webp)

  

  

![](https://picx.zhimg.com/80/v2-4777e75437e0c30cbe35c547477039f5_1440w.webp)

  

L1的Data和Instruction Cache被同一个Core的两个H-Thread共用

  

![](https://pic4.zhimg.com/80/v2-28d10e4967238c2756c1e8523ddc9cf9_1440w.webp)

  

  

![](https://picx.zhimg.com/80/v2-00f16c9fa7cbf99c262b2fd490459867_1440w.webp)

  

L2和L3都是Unified类型，可以同时存指令和数据。L3被同一个CPU封装里的所有Core共享，和论文所述一致。但是L2有点不一样，只被同一个Core的两个H-Thread共享，不跨Core共享。

## 1.2 Cache速度差距

  

![](https://pic3.zhimg.com/80/v2-9bcc7186a94080ca9d1c5a0048896128_1440w.webp)

  

  

![](https://picx.zhimg.com/80/v2-f7ea4e112e0d45b5ca88ac11d41ff1eb_1440w.webp)

  

L1d：2^13 L2：2^20

## 1.3 Cache实现细节

### 1.3.1 Cache的Key

  

![](https://pic3.zhimg.com/80/v2-f5873c14a04bd7bb4cfc372884970b8c_1440w.webp)

  

- T和S一起，唯一标识一个CacheLine，将Cache的组织想象成一个二维数组，通过两个角标T和S定位
- Offset标识CacheLine里的具体数据位置。一个CacheLine通常是64Byte，所以需要6bit的Offset来定位里面的每个Byte

### 1.3.2 Associative Cache

- 全路组相连：最朴素的思想，没有Set位，Cache用一维数组的方式组织。查询时直接通过Tag拿到数据。但是当Cache较大时，需要大量的Comp原件，成本高。

![](https://pic4.zhimg.com/80/v2-367c5e3775d78370e31c1646c468dba9_1440w.webp)

  

- 直接组相连。一个Set里面只有一个CacheLine，给定Set值可以唯一选择（MUX）出一个Tag

  

![](https://pic2.zhimg.com/80/v2-0bf15e9972db4a7ecfe10196b00b568d_1440w.webp)

  

- 多路组相连。一个Set里面有多个CacheLine，给定Set值可以选择出多个Tag，再通过Tag，拿到数据。
![[Pasted image 20240913120135.png]]
  

![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='934' height='666'></svg>)

  

下图是各种组相连的性能数据：

- Cache越大CacheMiss越小
- 组数的提升有利于CacheMiss的减少

  
![[Pasted image 20240913120141.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='874' height='876'></svg>)

  

## 1.4 Cache实验

### 1.4.1 实验设计

  
![[Pasted image 20240913120148.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='870' height='266'></svg>)

  

在内存中开辟一块连续的内存初始化成结构体数组，每个结构体如上图所示。有两个变化点：

- 指针n（如下图所示）：

- 依次指向下一个item，顺序访问
- 随机指向一个item，随机访问

- 填充数据pad：用于增加减少两个item的内存距离

  
![[Pasted image 20240913120156.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='888' height='572'></svg>)

  

### 1.4.2 实验结果与分析

### NPAD=0顺序读

  
![[Pasted image 20240913120203.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='910' height='874'></svg>)

  

上图是NPAD=0的顺序访问，实际上就是顺序遍历一个数组：

- 总体耗时曲线都比较低。最大也就9个Cycle，这是因为顺序访问会触发CPU硬件预取，减少了CacheMiss
- 有3个较为明显的台阶。分别对应L1，L2和Memory。不是有预取么？为啥还有有差异？这是因为访问数据时还没有完全预取上来，会有一定的Memory Stall，但这个Stall时间远远小于Cache Miss的时间

### 不同NPAD的顺序读

  
![[Pasted image 20240913120211.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='844' height='918'></svg>)

  

NPAD的增加，意味着顺序访问的步长增加：

- NPAD=0就是前面的图3.10，在这张图里基本上是一条直线
- 在L1 Cache内，NPAD的差异基本对性能没有影响，不涉及任何的预取。
- 在L2Cache内，NPAD=0除外的3条线基本在一起，并高于NPAD=0的用时。这和CacheLine相关，CacheLine的Cache的基本单位，通常是64Byte：

- 当NPAD=0时，sizeof(struct_l)=8Byte，一个CacheLine可以装8个Struct元素。一次预取可以支持8个元素的访问。
- 当NPAD=7时，sizeof(struct_l)=64Byte，一个CacheLine只能装1个Struct元素。一次预取只能支持1个元素的访问。
- 当NPAD>7时类似

一方面，因为每次预取通常只是减少MemoryStall，无法完全消除，平均每次访问所需要的预取次数越多，耗时越多。另一方面，更多的预取也可能占用更多的[内存带宽](https://zhida.zhihu.com/search?q=%E5%86%85%E5%AD%98%E5%B8%A6%E5%AE%BD&zhida_source=entity&is_preview=1)，降低速率。

- 超出L2后需要访问内存，4条线的差异很大，主要有三个原因：

- Prefetch无法完全消除Memory Stall。被访问的数据虽然被预取，但是在访问时还没有完全加载到Cache中。这个原因可以解释NPAD >= 7的3条线高于NPAD=0，但是无法解释NPAD7，15，31的差异。
- PageBoundaries的影响。一个物理页通常是4KB，在物理页边界时无法预取，因为PageFault需要操作系统来调度，CPU做不了。当NPAD越大，达到PageBoundaries所需要的元素个数越少，每次PageBoundaries开销均摊下来也就越大。
- TLB表Miss。假设有N个TLB表entry，平均访问了N*4K大小后，会触发一次TLB表Miss。如果NPAD越大，达到TLBMiss所需要的元素个数越少，每次TLB表Miss开销均摊下来也就越大。

### 触发TLB表Miss的场景

  
![[Pasted image 20240913120221.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='910' height='948'></svg>)

  

- On Cache Line表示一个数组元素的大小为64Byte，也就是一个CacheLine大小，也是图3.11中的NPAD=7
- On Page表示一个数组元素大小为4K，也就是一个物理页大小，每次指针跳跃会跨越一个Page
- 计算Working Set Size的逻辑不太一样：

- OnCacheLine的逻辑是计算真实消耗的内存，一个数组元素消耗的真实内存是64B，WorkingSetSize也是64B
- OnPage的逻辑是指计算一个Page个中的CacheLine大小。一个元素占用了一个Page的大小（4K），但是只算一个CacheLine的大小（64B）。
- 也就是说，同样是64B的WorkingSize，OnCacheLine消耗真实内存64B，OnPage消耗真实内存是4K
- 为啥做这么奇怪的规定？因为数组元素的大小只是决定顺序访问的Stride，每次访问的只有第一个指针，只访问一个CacheLine。

- 可以看到从2^13开始OnPage飙高，这意味着2^12刚好可以装载TLB中，也就是2^12/2^6=64个Page（如前面所述，一个Page在WorkingSize中只算64Byte）可以推算出这台机器的TLB表有64个entry
- 作者特别强调了这个实验进行了内存锁定，不会有Page Fault的影响。
- 虽然Stride是一个Page，但是每次只是访问一个CacheLine，为什么Cache没有生效？L2的key是物理地址，需要先经过TLB的转换，所以失效。L1通常是依照逻辑地址，但是载入L1之前需要先载入到L2，所以也受TLB限制。
- 超过64*4K=2^18Byte就会触发TLB Miss，TLB Miss的开销是很大的。这也解释了图3.11的NPAD=31为啥在顺序访问主存用时为300多Cycle，这台机器的理论主存访问为240个Cycle。

### 随机读

  
![[Pasted image 20240913120230.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='936' height='906'></svg>)

  

- 在L1内，随机读和顺序读没有本质差别
- 超出L1后，顺序读和随机读逐渐拉开差距，顺序读基本上是一条水平线，开销很小。

## 二 实践

## 2.1 绕开Cache

Cache被无关数据占据称为Cache污染，我们需要尽可能减少Cache污染，提高Cache利用率。

一种经典的Cache污染场景就是写数据。对于写数据，CPU的通常行为是只修改Cache中的数据，标为[脏数据](https://zhida.zhihu.com/search?q=%E8%84%8F%E6%95%B0%E6%8D%AE&zhida_source=entity&is_preview=1)，并不急于写入内存。当被写脏的CacheLine被淘汰时（或定时刷内存时）进行真正的落内存。这通常是有效的，比如对临时变量的反复读写，可以一直将这个临时变量装载Cache中，最后只落一次内存。

但也存在另外一种场景，就是写完数据后很长一段时间并不会用到，此时采用“写回”的Cache模式就很浪费，Cache住的脏数据占据了Cache，但实际上并不会被用到。此时可以考虑使用“写穿”的模式，“写穿”不是默认行为，需要调用特定的API，称为Non-temporal接口。

“写穿”需要进行内存访问，所以很慢，但是现代CPU有“write-combining”机制可以解决这个问题。设计实验如下：

  
![[Pasted image 20240913120240.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='956' height='616'></svg>)

  

分配一个二维数组，每一行的数据在物理内存上连续，每列数据在物理内存上不连续（也就是按行存）。左边是行优先遍历，右边是列优先遍历。

  
![[Pasted image 20240913120248.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='746' height='346'></svg>)

  

遍历耗时如上图所示，在行优先的场景下，Non-temporal的性能和Normal的性能一样。Non-temporal需要直接写到内存，为什么速度和Normal这种写Cache的一样？

这获益于CPU的“write-combining”机制。CPU有特殊的硬件，会积攒属于同一个CacheLine的写内存数据，直到收到的写入数据不属于这个CacheLine，将之前积攒的数据“批”写入到内存。然后积攒新来的这个CacheLine。

行优先时，写入的数据物理上连续，所以触发“write-combining”，有提速。列优先时，写入的数据物理上不连续，无法“write-combining”，所以写穿会更慢。

有的CPU还提供类“读穿”的功能，配合combining机制，也可以做到既不退性能，也不污染Cache。

此外，这个实验还告诉我们一个道理，尽可能优化算法变成顺序访问，现代CPU有很多优化顺序访问的机制。

## 2.2 优化L1 Data访问

### 2.2.1 算法优化

主要是调整算法，提升代码的时间和空间局部性。以[矩阵乘法](https://zhida.zhihu.com/search?q=%E7%9F%A9%E9%98%B5%E4%B9%98%E6%B3%95&zhida_source=entity&is_preview=1)为例：

  
![[Pasted image 20240913120256.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='902' height='110'></svg>)

  

  
![[Pasted image 20240913120306.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='868' height='222'></svg>)

  

根据前面的论述，最里面循环的mul1是行优先遍历，访存友好，mul2是列优先遍历，访存不友好，有优化空间

根据数学变化，调整代码如下：

  
![[Pasted image 20240913120313.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='892' height='130'></svg>)

  

  
![[Pasted image 20240913120321.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='856' height='328'></svg>)

  

首先对mul2进行了[矩阵转置](https://zhida.zhihu.com/search?q=%E7%9F%A9%E9%98%B5%E8%BD%AC%E7%BD%AE&zhida_source=entity&is_preview=1)运算，用一块临时内存tmp存下来。然后使用tmp矩阵和mul1进行运算。mul1和tmp都是行优先遍历。前面做转置的时候，mul2也是行优先遍历。整个算法都是行优先，访存友好。效果很好，提速了76%

  
![[Pasted image 20240913120328.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='768' height='196'></svg>)

  

转置算法消耗了额外的内存，我们希望可以不消耗额外内存进行优化，如下：

  
![[Pasted image 20240913120335.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='830' height='448'></svg>)

  

SM通过[宏定义](https://zhida.zhihu.com/search?q=%E5%AE%8F%E5%AE%9A%E4%B9%89&zhida_source=entity&is_preview=1)，代表一个CacheLine可以装下的double个数。将一个大矩阵拆分成了长宽为SM的小矩阵进行分治计算。小矩阵中非连续访问的内存距离不超过一个CacheLine的大小。效果略好过转置方案。

  
![[Pasted image 20240913120341.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='1314' height='312'></svg>)

  

### 2.2.2 数据结构优化

### 2.2.2.1 结构体内部对齐

  
![[Pasted image 20240913120346.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='858' height='278'></svg>)

  

对如上结构体，使用pahole分析出具体的内存布局：

  
![[Pasted image 20240913120351.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='1576' height='640'></svg>)

  

- 结构体内部按照操作系统位数对齐，32bit系统按照4Byte对齐，64bit系统按照8Byte对齐
- 最右边的注释表示数据的起始位置，和占用的数据大小，比如第一个成员int a，在结构体偏移为0的地方开始，占用了4Byte
- 按照8Byte对齐，所以在int a之后需要填充4个Byte进行对齐，这里产生了1和hole，大小为4Byte
- 一个CacheLine是64Byte，在long int数组结束后使用完了一个CacheLine的大小，下面的成员只能分配到下一个CacheLine中

  

一个显然的优化是调整成员b的位置，将其放在成员a后面，这样有两个好处：

- 消除了因为对齐产生的空洞
- 整个结构体的大小刚好可以装进一个CacheLine

除此之外，为了尽可能避免跨CacheLine访问，还需要：

- 尽可能把经常用到的结构体元素放在最前面
- 访问结构体时，如果没有特别的业务逻辑需要，尽可能顺序访问结构体里的元素

### 2.2.2.2 结构体外部对齐

在满足上述结构体优化后依然是不够的，一个结构体大小即使在一个CacheLine大小内，但如果起始位没有CacheLine对齐，依然会跨CacheLine访问。所以，还需尽可能做到CacheLine对齐。

Malloc分配出来的结构体是8（32bit系统）或者16Byte（64bit系统）对齐的。也就是说Malloc出来的对象在64bit系统中的起始地址一定是16的倍数，用[十六进制](https://zhida.zhihu.com/search?q=%E5%8D%81%E5%85%AD%E8%BF%9B%E5%88%B6&zhida_source=entity&is_preview=1)表示，结尾一定是0，简单实验一下：

```c
printf("address t: %x\n",malloc(sizeof(st)));
printf("address c: %x\n",malloc(sizeof(char)));
printf("address x: %x\n",malloc(sizeof(int)));
printf("address y: %x\n",malloc(sizeof(long)));
```

输出：

  
![[Pasted image 20240913120407.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='322' height='186'></svg>)

  

按照16Byte外部对齐可能跨CacheLine，进而导致性能问题。如下，一个结构体的size为32Byte，Malloc可能把它放在地址48上（16Byte对齐）。但这跨CacheLine了，需要加载两个CacheLine才能读取这个数据。

  
![[Pasted image 20240913120414.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='786' height='760'></svg>)

  

### 直接使用接口指定对齐字节

  
![[Pasted image 20240913120540.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='800' height='212'></svg>)

  

```c
void * x2;
posix_memalign(&x2,64,sizeof(int));
printf("address x2: %x\n",x2);
void * x3;
posix_memalign(&x3,64,sizeof(long));
printf("address x3: %x\n",x3);
return 0;
```

结果如下，地址一定是64的倍数，也就是6个0结尾

  
![[Pasted image 20240913120549.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='348' height='70'></svg>)

  

### 在声明时指定对齐字节

如果是零时变量怎么对齐呢？

  
![[Pasted image 20240913120555.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='572' height='132'></svg>)

  

```c
struct _st st3 __attribute((aligned(64)));
char c3 __attribute((aligned(64)));
printf("address st3: %x\n",&st3); 
printf("address c3: %x\n",&c3); 
```

  

![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='386' height='68'></svg>)

  

### 在定义处指定对齐字节

上述两种方法都有一个问题，如果是数组的话，只能对齐第一个数据：

```text
struct _st st3[4] __attribute((aligned(64)));
printf("address st3: %x\n",&st3[0]); 
printf("address st3: %x\n",&st3[1]); 
```

第二个元素没有64byte对齐

  
![[Pasted image 20240913120611.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='374' height='64'></svg>)

  

解法是在定义出指定对齐字节

  
![[Pasted image 20240913120617.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='594' height='182'></svg>)

  

```c
typedef struct _st
{
    int a;
    long fill[7];
    int b;
} __attribute((aligned(64))) st;

struct _st st3[4];
printf("address st3: %x\n",&st3[0]); 
printf("address st3: %x\n",&st3[1]);
```

输出如下，每个元素都是对齐的

  
![[Pasted image 20240913120625.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='374' height='74'></svg>)

  

  

CacheLine对齐不是没有代价的，会产生大量空洞，消耗更多的内存。

  

未对齐较之于对齐后的性能回退如下图，未对齐的耗时都会增加，在Cache大小内的耗时增加更大。Random在L2中没有对齐的时间增加较少，这是因为随机访问本来的耗时基数较大，对齐与否的影响层度减弱。

  
![[Pasted image 20240913120630.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='902' height='904'></svg>)

  

### 2.2.2.3 结构体的拆分

如下图，通常我们会按照业务逻辑封装对象。

  
![[Pasted image 20240913120636.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='558' height='332'></svg>)

  

根据使用成员的频率进行拆分，有利于提升性能。比如，上面的结构体中的price和paid经常用到，buyer和buyer_id不常用到，拆分成两个独立的结构体，对于price和paid组成的新结构体而言，有效数据密度更大，Cache的效果也越好。但这样的拆分会引入代码的负载度，要根据实际情况进行判断。

### 2.2.2.4 避免触发conflict misses

现代CPU都采用多路组相连，在没有占满整个Cache的情况下，由于占满了一个set，导致的Cache的淘汰。

  
![[Pasted image 20240913120642.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='1872' height='988'></svg>)

  

- 实验的目标是验证conflict miss。L1的Key是逻辑地址，L2是物理地址，我们无法控制物理地址的访问，所以只能验证L1的Conflict miss。构造一个前文多次提到的链表，每个节点的NPAD=15，节点总大小是64Byte，一个CacheLine大小。按照一定的Stride进行多次遍历。
- X坐标表示访问的Stride距离，也是链表节点的距离，比如说距离为2，就是相隔两个链表节点，64*2Byte。Y坐标表示链表的总长度。Z表示耗时，单位是CPU Cycles
- 水平面是3个Cycles，刚好是L1的访存速度
- 此CPU是8路组相连，地址按照4096Byte（64距离*64B的CacheLine）取模选择Cache Set。所以，距离是64的倍数，每次访问链表节点都会落入同一个Cache Set，当List的长度超过8，CacheSet一定会满，进而把之前的CacheLine挤掉。再一次遍历时，无法命中，还需要再一次加载Cache

## 2.3 其他Cache优化

### 2.3.1 优化L1 Instruction访问

主要有3个优化方向：

- 减少代码体积。但这需要和unloop和inline优化做好权衡，因为unloop和inline会增加代码体积。
- 应该减少代码空洞。主要是减少代码中的jump指令，顺序执行的预取友好的。
- 如有可能，对代码本身进行对齐，避免跨CacheLine。

具体做法有：

- 使用-Os编译选项优化CodeSize。会关闭掉影响CodeSize的优化项，比如inline和unloop，具体谁更快，需要测试。
- inline可能加剧代码空洞，如下图，如果代码块B不是经常走到，inline后会增加A和C之间的代码距离

  
![[Pasted image 20240913120648.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='764' height='268'></svg>)

  

- 分支判断失败的开销很大，一方面是L1i的CacheMiss，另一方面还需要重新decode指令。消除的方式有两种：

- 两次编译，第一次收集分支统计信息，第二次正式编译利用统计信息选择概率更大的分支。
- 手动指定预测分支：

```c
//接口
long __builtin_expect(long EXP, long C);

//宏定义
#define unlikely(expr) __builtin_expect(!!(expr), 0)
#define likely(expr) __builtin_expect(!!(expr), 1)

//使用
if (likely(a > 1)) 
```

- 代码对齐。通常有3中对齐场景：

- 函数对齐
- 分支对齐
- 循环对齐

函数和分支对齐直接jump到对齐代码块即可，没有什么开销。但循环的代码对齐不一样，loop代码块通常是紧接着其他的顺序代码后面的，要对齐只有两种办法：

- 插入无效指令占位
- 显示jump一次

这两种方式都是有开销的，所以，只有当循环次数很大时才有必要循环对齐。

  
![[Pasted image 20240913120702.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='1776' height='530'></svg>)

  

### 2.3.2 L2 优化

L2Cache（在文中是最后一个Level的Cache）的优化和L1基本一致，但是有两个点比较特殊：

- 如果Miss，开销更大。L1Cache不住，还可以L2Cache，性能损失有限。L2Miss就要访问内存了，开销很大。
- L2会被多个Core共享。可以利用的Cache不及预期。在涉及L2敏感算法时，需要用L2Size处于共享的核数。如果是Linux系统，在/sys/devices/system/cpu/cpu*/cache中的shared_cpu_map查看

### 2.3.3 TLB优化

避免跨页访问，尽可能将Code和Data聚集在一个Page范围内集中访问。但ASLR（AddressSpaceLayoutRandomization）机制无法避免代码的跨页访问。如果代码加载到固定地址，将很容易受到攻击，所以操作系统加载代码时会进行随机处理。

## 2.4 预取优化

### 2.4.1 硬件预取

一个core通常会有8到16个硬件预取原件，会对CacheMiss进行监控，如果miss的CacheLine的Stride是定长，且累积miss次数超过阈值，触发硬件预取。通常预取原件都是部署在L2和更高层级的Cache上，由于L2会被共享，所以预取原件很快就会被消耗完。

前文已经提到过，硬件预取不能跨页。通常硬件预取的Stride不会超过512B，一个物理页是4K，如果Stride超过512B，预取8次就跨页了，不合理。

如果内存带宽被打满。可以考虑关闭掉硬件预取，但是这是CPU级别生效。可以将低优先级线程绑定在的定的CPU上，并关闭其硬件预取功能，减少内存带宽压力。

### 2.4.2 软件预取

  
![[Pasted image 20240913120708.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='778' height='464'></svg>)

  

- T0，T1，T2分别将CacheLine加载到L1，L2，L3中，如果加载到L1中，L2和L3也会被加载。使用[预取指令](https://zhida.zhihu.com/search?q=%E9%A2%84%E5%8F%96%E6%8C%87%E4%BB%A4&zhida_source=entity&is_preview=1)时需要考虑WorkSet大小，如果太大，不应该直接使用T0，而是考虑更合适的T1和T2。
- NTA，尽可能避免Cache污染进行预取。也就是前文提到的non-temporal操作

随机访问的软件预取效果如下，有一定的提升。

  
![[Pasted image 20240913120717.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='900' height='856'></svg>)

  

### 2.4.3 Helper Thread预取

H-Thread会共用L1 Cache，容易造成Cache污染等问题，论文提到了一种解决办法，使用Helper-Thread专门进行prefetch。属于同一个Core的两个H-Thread作为一组，一个用于用于处理业务逻辑，另一个专门用于辅助预取数据。可以使用NUMA库绑定线程到特定的H-Thread上实现。

Helper-Thread方法的效果如下，收益明显，但是也有自己的问题：

- Helper线程和主线程的同步需要额外逻辑和开销
- Helper线程本来可以用来做其他的计算任务，两种方式需要权衡

  
![[Pasted image 20240913120724.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='918' height='884'></svg>)

编辑于 2023-11-07 20:05・IP 属地浙江

[

计算机

](https://www.zhihu.com/topic/19555547)

[

性能

](https://www.zhihu.com/topic/19576984)

[

内存（RAM）

](https://www.zhihu.com/topic/19570383)

​已赞同 1354​​40 条评论

​分享

​喜欢​收藏​申请转载

​

已赞同 1354

​

分享

![](https://picx.zhimg.com/v2-abed1a8c04700ba7d72b45195223e0ff_l.jpg?source=32738c0c&needBackground=1)

理性发言，友善互动

  

40 条评论

默认

最新

[![ass assin](https://picx.zhimg.com/v2-7e050ab35793f784f55251baddd37b94_l.jpg?source=06d4cd63)](https://www.zhihu.com/people/bce4d5d35aaee77677d81f4750d15043)

[ass assin](https://www.zhihu.com/people/bce4d5d35aaee77677d81f4750d15043)

​

我只看懂了擦车（Cache）和公交汽车（Bus）还有地址（Address），推测这是一篇教大家怎么给公交车擦车的文章（大雾）![[爱]](https://pic1.zhimg.com/v2-0942128ebfe78f000e84339fbb745611.png)

2023-09-08 · 山东

​回复​11

[![Lavender](https://picx.zhimg.com/v2-8c2302ccef280f79a12d7f78ad674907_l.jpg?source=06d4cd63)](https://www.zhihu.com/people/deca4e702ef579b42ba80abcb2a5f595)

[Lavender](https://www.zhihu.com/people/deca4e702ef579b42ba80abcb2a5f595)

这文章不错，CSAPP一些内容应该就是来源于这篇文章的

2023-02-23 · 美国

​回复​8

[![Micro](https://picx.zhimg.com/v2-4806447ffad7c76e9f785b994e8a2151_l.jpg?source=06d4cd63)](https://www.zhihu.com/people/c751a0fa58a074fc0fddb9329d3d3141)

[Micro](https://www.zhihu.com/people/c751a0fa58a074fc0fddb9329d3d3141)

牛蛙！绝赞干货

2023-02-23 · 北京

​回复​1

[![狗娃娃](https://pic1.zhimg.com/v2-cb4af5022474b9c03aec7503933c0133_l.jpg?source=06d4cd63)](https://www.zhihu.com/people/cb0576e9bd245fa8aa282059de737e03)

[狗娃娃](https://www.zhihu.com/people/cb0576e9bd245fa8aa282059de737e03)

讲的好棒啊，我想请教一下，里面的实验中的cpu的cycle数是怎么得到的呢？是有什么工具接口专门统计某个过程的cycle数吗？

2023-02-23 · 上海

​回复​2

[![倒斗仙人孙殿英](https://pic1.zhimg.com/v2-c0f128106052a1608a4d22cb6510005e_l.jpg?source=06d4cd63)](https://www.zhihu.com/people/9bdebeb7644b48ec9a7c701b516be723)

[倒斗仙人孙殿英](https://www.zhihu.com/people/9bdebeb7644b48ec9a7c701b516be723)

rdtscp

2023-02-24 · 广东

​回复​2

[![红星闪闪](https://picx.zhimg.com/v2-7e1fc938ae12f5dce67d46b570d99ded_l.jpg?source=06d4cd63)](https://www.zhihu.com/people/75932681e68db058a2820edc51ca7507)

[红星闪闪](https://www.zhihu.com/people/75932681e68db058a2820edc51ca7507)

作者

perf就可以吧

2023-03-15 · 浙江

​回复​1

展开其他 1 条回复​

[![Mbappe](https://pica.zhimg.com/v2-6de8d97bb6f413b7d909797b2e29ee90_l.jpg?source=06d4cd63)](https://www.zhihu.com/people/1911caacb0127efee7486c34eb0b5110)

[Mbappe](https://www.zhihu.com/people/1911caacb0127efee7486c34eb0b5110)

![](https://pica.zhimg.com/v2-4812630bc27d642f7cafcd6cdeca3d7a.jpg?source=88ceefae)

好文！

2023-12-11 · 广东

​回复​喜欢

[![曼达洛人](https://picx.zhimg.com/v2-9a1d049b27317037342511b00462793b_l.jpg?source=06d4cd63)](https://www.zhihu.com/people/6b4eccfe6767706e1fcd4d004c71dbb0)

[曼达洛人](https://www.zhihu.com/people/6b4eccfe6767706e1fcd4d004c71dbb0)

L2Miss就要访问内存了，开销很大。   
这句话是不是应该说的L3![[思考]](https://pic4.zhimg.com/v2-bffb2bf11422c5ef7d8949788114c2ab.png)

2023-09-26 · 上海

​回复​喜欢

[![seven](https://picx.zhimg.com/v2-abed1a8c04700ba7d72b45195223e0ff_l.jpg?source=06d4cd63)](https://www.zhihu.com/people/cf1628f1cbf52eb7b261d06b32439a16)

[seven](https://www.zhihu.com/people/cf1628f1cbf52eb7b261d06b32439a16)

这个不算重点，看具体的架构，估计原文是有基于特定cpu进行描述的。不过稍微严谨应该应该说最后一级cache。

07-09 · 上海

​回复​喜欢

[![Isaac 丶zhang](https://pic1.zhimg.com/v2-abed1a8c04700ba7d72b45195223e0ff_l.jpg?source=06d4cd63)](https://www.zhihu.com/people/8461ec5fec83a9dd86bf00067aaa2c6f)

[Isaac 丶zhang](https://www.zhihu.com/people/8461ec5fec83a9dd86bf00067aaa2c6f)

Cache里面还有bank能不能介绍一下

2023-06-18 · 江苏

​回复​喜欢

[![黑夜弥漫思绪](https://pic1.zhimg.com/07f63825e800b73022ab65f35a85cea1_l.jpg?source=06d4cd63)](https://www.zhihu.com/people/1e7e95334620104ef6bfd991d7030d8a)

[黑夜弥漫思绪](https://www.zhihu.com/people/1e7e95334620104ef6bfd991d7030d8a)

为什么说：达到PageBoundaries所需要的元素个数越少，PageBoundaries开销均摊下来也就越大？

2023-04-14 · 四川

​回复​喜欢

[![卷心菜](https://picx.zhimg.com/989395927_l.jpg?source=06d4cd63)](https://www.zhihu.com/people/f71989b96523639878c8f1feae626e75)

[卷心菜](https://www.zhihu.com/people/f71989b96523639878c8f1feae626e75)

从内存里读取的数据会同时放到L1L2L3吗？

2023-03-05 · 广东

​回复​喜欢

[![unlsycn](https://picx.zhimg.com/v2-7db5401757c880616068b020c34cdec5_l.jpg?source=06d4cd63)](https://www.zhihu.com/people/b2330a94d31fff9f3ebcd3784f713b37)

[unlsycn](https://www.zhihu.com/people/b2330a94d31fff9f3ebcd3784f713b37)

![](https://picx.zhimg.com/v2-4812630bc27d642f7cafcd6cdeca3d7a.jpg?source=88ceefae)

取决于cache是exclusive还是inclusive的，有些处理器甚至L2是exclusive的但是LLC是inclusive的

05-17 · 江西

​回复​喜欢

[![知乎用户T048im](https://pic1.zhimg.com/v2-9401bda51aa0c419a020dad56c608f54_l.jpg?source=06d4cd63)](https://www.zhihu.com/people/be6636e16183262ebdebeb6087a9c4d6)

[知乎用户T048im](https://www.zhihu.com/people/be6636e16183262ebdebeb6087a9c4d6)

看架构，存储体系很多种

2023-11-07 · 广东

​回复​喜欢

展开其他 1 条回复​

[![蜘蛛侠](https://picx.zhimg.com/v2-e82441821eba0dd3a5662092091fab17_l.jpg?source=06d4cd63)](https://www.zhihu.com/people/f9dc0e44dcf6b97a02891db7fddfd39a)

[蜘蛛侠](https://www.zhihu.com/people/f9dc0e44dcf6b97a02891db7fddfd39a)

![](https://picx.zhimg.com/v2-4812630bc27d642f7cafcd6cdeca3d7a.jpg?source=88ceefae)

考大家一个问题，看似简单，但是大多数人都答不上来。从内存读取数据的时间远远大于从cache的，所以才会有cache提速。那么问题来了，cache中的数据源头还是内存，而且我们已经知道从内存读取数到cache的时间很长，那么这么长的时间从哪里来的？ 作者能回答下么？

2023-03-03 · 北京

​回复​喜欢

[![顾仁之](https://pic1.zhimg.com/v2-c3154bc309e7424adde3370f3cd0dbe8_l.jpg?source=06d4cd63)](https://www.zhihu.com/people/b43690edd381f131a81036549a099f9d)

[顾仁之](https://www.zhihu.com/people/b43690edd381f131a81036549a099f9d)

尝试回答一下：

cache取数据就是很耗时，你避免不了，所以有时候就得等那么长的时间，只是下次再访问，就不用等这么久了cache也有prefetch，其中又分为hw prefetch和sw prefetchhw prefetch是硬件自动做的，具体原理我不知道，但反正不会拖累当前的load行为sw prefetch是软件做的，该指令可以异步地提前将ddr数据取到cache中来

2023-10-27 · 广东

​回复​4

[![蜘蛛侠](https://picx.zhimg.com/v2-e82441821eba0dd3a5662092091fab17_l.jpg?source=06d4cd63)](https://www.zhihu.com/people/f9dc0e44dcf6b97a02891db7fddfd39a)

[蜘蛛侠](https://www.zhihu.com/people/f9dc0e44dcf6b97a02891db7fddfd39a)

![](https://picx.zhimg.com/v2-4812630bc27d642f7cafcd6cdeca3d7a.jpg?source=88ceefae)

[Sailing](https://www.zhihu.com/people/6182bba4f0886d37ead128ed9361e9c9)

你这个是一方面，其实主要还是分支预测，就是cpu预测到未来要执行什么程序

2023-04-25 · 北京

​回复​1

查看全部 14 条回复​

点击查看全部评论

![](https://picx.zhimg.com/v2-abed1a8c04700ba7d72b45195223e0ff_l.jpg?source=32738c0c&needBackground=1)

理性发言，友善互动

  

### 文章被以下专栏收录

[

![分布式和存储的那些事](https://pica.zhimg.com/v2-eb580efc3af40d32b23d103642427650_l.jpg?source=172ae18b)

](https://www.zhihu.com/column/distributed-storage)

## [

分布式和存储的那些事

](https://www.zhihu.com/column/distributed-storage)

关于分布式和存储

[

![计算机体系结构](https://picx.zhimg.com/4b70deef7_l.jpg?source=172ae18b)

](https://www.zhihu.com/column/c_1362743082979672064)

## [

计算机体系结构

](https://www.zhihu.com/column/c_1362743082979672064)

[

![IT](https://pica.zhimg.com/4b70deef7_l.jpg?source=172ae18b)

](https://www.zhihu.com/column/c_1705318239930867712)

## [

IT

](https://www.zhihu.com/column/c_1705318239930867712)

计算机基础知识、性能优化、数据库优化、OLAP

### 推荐阅读

[

![论文解读：pointer-gen for summarization](https://picx.zhimg.com/v2-754ad2041d593ac19cfc71abf4c959e6_250x0.jpg?source=172ae18b)

# 论文解读：pointer-gen for summarization

陈怡然



](https://zhuanlan.zhihu.com/p/109122901)[

# 28 篇最新论文解读 LLMs-based Agents

引言1. 博主在阅读Agent方向的论文时，做了些笔记，加工后形成了这篇文章。 2. 学术论文最核心的idea通常用一两段话就可以概括。 3. 这也是论文简读的意义所在。 4. 对每篇论文，本文从moti…

芒格的信号



](https://zhuanlan.zhihu.com/p/662506575)[

![如何快速理解一篇 ML 论文的要点？谷歌 Robotics 研究科学家：只要记住5个问题](https://pic1.zhimg.com/v2-ff854191ed5f2b655cd21495933c9ca1_250x0.jpg?source=172ae18b)

# 如何快速理解一篇 ML 论文的要点？谷歌 Robotics 研究科学家：只要记住5个问题

忆臻发表于机器学习算...



](https://zhuanlan.zhihu.com/p/350986421)[

# 学术英文论文结构 Dissertation structure

Dissertation structure Structure and organizationTitle pageAcknowledgementsAbstractTable of ContentsTable of ContentsFigure and table listsList of abbreviationsGlossaryIntroduc…

Alex发表于程序开发



](https://zhuanlan.zhihu.com/p/342063088)
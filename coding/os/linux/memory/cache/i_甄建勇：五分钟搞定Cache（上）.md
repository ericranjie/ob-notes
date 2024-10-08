原创 甄建勇 Linux阅码场
 _2021年11月19日 07:03_
作者简介
甄建勇，高级架构师（某国际大厂），十年以上半导体从业经验。主要研究领域:CPU/GPU/NPU架构与微架构设计。感兴趣领域:经济学、心理学、哲学。
本系列为甄建勇“五分钟系列”——用五分钟把计算机系统中一个关键的概念讲清楚，如果你也对计算机系统某些模块有独到的理解，欢迎赐稿“Linux阅码场”，投稿将获惊喜礼物一份，投稿请扫码加微信。

关于cache，大概可以从三个方面进行阐述：内存到cache的映射方式，cache的写策略，cache的替换策略。 
# **映射方式**

内存到cache的映射方式，大致可以分为三种，分别是：直接映射（directmapped），全相连（fullyassociative），组相连（setassociative）。

为了便于理解，现在假设一个例子，比如咱们的内存只有16bytes，而cache只有4bytes（cacheline是1byte），那么对于分别采用三种不同的映射方式，会是什么情况呢？如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Ass1lsY6byvxseviccopU8U1ecCX3SZtgqm93g5adnXgoibKnsfeNlXrxDS0JNbZgXblUgDib6tJablz4tic54Gfsw/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

 _图1      Cache的映射_

(direct mapped：直接映射 ; fully associative：全相连 ;set associative：组相连)
## **（1）directmapped**

对于directmapped（直接映射），为了便于数据查找，一般规定内存数据只能置于缓存的特定区域。对于直接匹配缓存，每一个内存块地址都可通过模运算对应到一个唯一缓存块上。注意这是一个多对一匹配：多个内存块地址须共享一个缓存区域。

对于咱们这个例子来说，内存的0地址只能映射到cache的第0个（0%4=0）cacheline，内存的1地址只能映射到cache的第1个（1%4=1）cacheline，内存的2地址只能映射到cache的第2个（2%4=2）cacheline，内存的3地址只能映射到cache的第3个（3%4=3）cacheline，内存的4地址只能映射到cache的第0个（4%4=0）cacheline，。。。。。。如此循环下去。

所以如果采用directmapped的话，core在访问cache时，根据TLB处理之后的物理地址，进行取模（%）运算，就可以直接确定其cache的位置，由于一个cacheline可能对应不同的内存地址（具有相同模运算结果的内存），然后将物理地址的tag部分与cache的tag部分进行一次比较，就可以确定是cache hit，还是cachemiss。

directmapped的特点是，逻辑简单，延迟短（只进行一次比较），但命中率低。
## **（2）fullyassociative**

对于fullyassociative（全相连），这种方式，内存中的数据块可以被放置到cache的任意区域。这种相联完全免去了索引的使用，而直接通过在整个缓存空间上匹配标签进行查找。 

对于咱们的这个例子来说，内存的某个地址，可以映射到cache的任意个cacheline。内存的0地址能映射到cache的第0个cacheline，也可以映射到第1个cacheline，也可以映射到第2个cache line，也可以映射到第3个cacheline。

所以如果采用fullyassociative的话，core在访问cache时，根据TLB处理之后的物理地址，要依次和所有的cacheline的tag进行比较。

fullyassociative的特点是：控制复杂，查找造成的电路延迟最长，因此仅在特殊场合，如缓存极小时，才会使用，命中率较高。
## **（3）setassociative**

set associative（组相连）是directmapped 和fully associative两种方式的一个折中。

对于咱们这个例子来说，我们将4个cacheline分成了两组，内存的0地址只能映射到cache的第0个组（0%2=0），但是在组内是任意的，既可以映射到组内的第0个cacheline，也可以映射到第1个cacheline。内存的1地址只能映射到cache的第1个组(1%2=1)，但是在组内也是任意的，既可以映射到组内的第0个cacheline，也可以映射到第1个cacheline。内存的2地址只能映射到cache的第0个组(2%2=0)，但是在组内也是任意的，既可以映射到组内的第0个cacheline，也可以映射到第1个cacheline，。。。。。。。依次类推。

所以，如果采用setassociative的话，core在访问cache时，根据TLB处理之后的物理地址，先将物理地址取模，得到其可能的cache的组，然后再依次与组内的所有cacheline的tag进行比较，确定是cache hit还是cachemiss。

setassociative是折中方案，所以其特点就是集directmapped 和fully associative之所长。是一个平衡方案。

咱们这个例子是2 way setassociative，即两路组相连，所谓的两路，是指每个cache组内的cacheline的数目，不是分组的数目。比如是4路组相连，指的是每个cache组内有4个cacheline。

对于直接映射，由于缓存字节数和缓存块数均为2的幂，上述运算可以由硬件通过移位极快地完成。直接匹配缓存尽管在电路逻辑上十分简单，但是存在显著的冲突问题。由于多个不同的内存块仅共享一个缓存块，一旦发生缓存失效就必须将缓存块的当前内容清除出去。这种做法不但因为频繁的更换缓存内容造成了大量延迟，而且未能有效利用程序运行期所具有的时间局部性。

组相联（SetAssociativity）是解决这一问题的主要办法。使用组相联的缓存把存储空间组织成多个组，每个组有若干数据块。通过建立内存数据和组索引的对应关系，一个内存块可以被载入到对应组内的任一数据块上。

直接映射可以认为是单路组相联。经验规则表明，在缓存小于128KB时，欲达到相同失效率，一个双路组相联缓存仅需相当于直接匹配缓存一半的存储空间。

为了和下级存储（如内存）保持数据一致性，就必须把数据更新适时传播下去。这种传播通过回写来完成。

# **写策略**

一般有两种回写策略：写回（Writeback）和写通（Writethrough）。

写回是指，仅当一个缓存块需要被替换回内存时，才将其内容写入内存。如果缓存命中，则总是不用更新内存。为了减少内存写操作，缓存块通常还设有一个脏位（dirtybit），用以标识该块在被载入之后是否发生过更新。如果一个缓存块在被置换回内存之前从未被写入过，则可以免去回写操作。

写回的优点是节省了大量的写操作。这主要是因为，对一个数据块内不同单元的更新仅需一次写操作即可完成。这种内存带宽上的节省进一步降低了能耗，因此颇适用于嵌入式系统。

写通是指，每当缓存接收到写数据指令，都直接将数据写回到内存。如果此数据地址也在缓存中，则必须同时更新缓存。由于这种设计会引发造成大量写内存操作，有必要设置一个缓冲来减少硬件冲突。这个缓冲称作写缓冲器（Writebuffer），通常不超过4个缓存块大小。不过，出于同样的目的，写缓冲器也可以用于写回型缓存。

写通较写回易于实现，并且能更简单地维持数据一致性。

当发生写失效时，缓存可有两种处理策略，分别称为分配写（Writeallocate）和非分配写（No-writeallocate）。

分配写是指，先如处理读失效一样，将所需数据读入缓存，然后再将数据写到被读入的单元。非分配写则总是直接将数据写回内存。

设计缓存时可以使用回写策略和分配策略的任意组合。对于不同组合，发生数据写操作时的行为也有所不同。

对于组相联缓存，当一个组的全部缓存块都被占满后，如果再次发生缓存失效，就必须选择一个缓存块来替换掉。存在多种策略决定哪个块被替换。

# *替换策略*


显然，最理想的替换块应当是距下一次被访问最晚的那个。这种理想策略无法真正实现，但它为设计其他策略提供了方向。

先进先出算法（FIFO）替换掉进入组内时间最长的缓存块。最久未使用算法（LRU）则跟踪各个缓存块的使用状况，并根据统计比较出哪个块已经最长时间未被访问。对于2路以上相联，这个算法的时间代价会非常高。

对最久未使用算法的一个近似是非最近使用（NMRU）。这个算法仅记录哪一个缓存块是最近被使用的。在替换时，会随机替换掉任何一个其他的块。故称非最近使用。相比于LRU，这种算法仅需硬件为每一个缓存块增加一个使用位（usebit）即可。

此外，也可使用纯粹的随机替换法。测试表明完全随机替换的性能近似于LRU。

未完待续

---

精彩回顾

  

[甄建勇：芯片架构方法学](http://mp.weixin.qq.com/s?__biz=MzAwMDUwNDgxOA==&mid=2652671184&idx=1&sn=16bfce9af8d5650a71ab961ea463a2db&chksm=810fca4db678435b634cb431f9dfc1fe805a57409e9b2d2c068ef16b696b978265c46a0c23ea&scene=21#wechat_redirect)

[甄建勇：五分钟搞定MMU](http://mp.weixin.qq.com/s?__biz=MzAwMDUwNDgxOA==&mid=2652680376&idx=1&sn=75fb4280dd0b141dd385b4421b958afd&chksm=810ff625b6787f33c75bcd900f0158d927c81fed0267c1b1aa0f8d8ec173e0fa9aef0bfc58cc&scene=21#wechat_redirect)  

  
  

更多精彩尽在"Linux阅码场"，扫描下方二维码关注

写留言

**留言 11**

- 谢英豪
    
    2021年11月19日
    
    赞1
    
    测试表明完全随机替换的性能近似于LRU 这句有引用出处嘛
    
    Linux阅码场
    
    作者2021年11月21日
    
    赞
    
    自己测试的结果，不同的场景可能结论不同。
    
- 杰克朱
    
    2021年11月19日
    
    赞1
    
    组相连的Cache策略需要cache控制器嘛？还是仅仅通过逻辑电路呢？
    
    Linux阅码场
    
    作者2021年11月21日
    
    赞
    
    Design time parameter
    
- Hugo
    
    2021年11月19日
    
    赞1
    
    大佬可以分享一下MESI协议
    
    Linux阅码场
    
    作者2021年11月21日
    
    赞
    
    一句话就是，保持最新数据只有一个。
    
- 简单
    
    2021年11月19日
    
    赞1
    
    现在MCU也有cache了，讲得很透彻。
    
- 宗哥
    
    2021年11月19日
    
    赞1
    
    看了后，突然有点迷糊的组相连豁然开朗了![[呲牙]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 大见
    
    2021年11月22日
    
    赞
    
    应该叫 5分钟了解更合适。cache给软件人员带来了太多的麻烦。
    
- 仁义
    
    2021年11月20日
    
    赞
    
    cache没什么用的，还是想办法加速ddr的速度吧，这样就可以淘汰cache
    
- 何旻隆
    
    2021年11月19日
    
    赞
    
    以前好像发过类似的，更详细一些的。
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/YHBSoNHqDiaH5V6ycE2TlM9McXhAbaV9aUBaTAibVhLibXH1rL4JTMb2eoQfdCicTm4wycOsopO82jG3szvYF3mTJw/300?wx_fmt=png&wxfrom=18)

Linux阅码场

13110

11

写留言

**留言 11**

- 谢英豪
    
    2021年11月19日
    
    赞1
    
    测试表明完全随机替换的性能近似于LRU 这句有引用出处嘛
    
    Linux阅码场
    
    作者2021年11月21日
    
    赞
    
    自己测试的结果，不同的场景可能结论不同。
    
- 杰克朱
    
    2021年11月19日
    
    赞1
    
    组相连的Cache策略需要cache控制器嘛？还是仅仅通过逻辑电路呢？
    
    Linux阅码场
    
    作者2021年11月21日
    
    赞
    
    Design time parameter
    
- Hugo
    
    2021年11月19日
    
    赞1
    
    大佬可以分享一下MESI协议
    
    Linux阅码场
    
    作者2021年11月21日
    
    赞
    
    一句话就是，保持最新数据只有一个。
    
- 简单
    
    2021年11月19日
    
    赞1
    
    现在MCU也有cache了，讲得很透彻。
    
- 宗哥
    
    2021年11月19日
    
    赞1
    
    看了后，突然有点迷糊的组相连豁然开朗了![[呲牙]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)![[强]](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=)
    
- 大见
    
    2021年11月22日
    
    赞
    
    应该叫 5分钟了解更合适。cache给软件人员带来了太多的麻烦。
    
- 仁义
    
    2021年11月20日
    
    赞
    
    cache没什么用的，还是想办法加速ddr的速度吧，这样就可以淘汰cache
    
- 何旻隆
    
    2021年11月19日
    
    赞
    
    以前好像发过类似的，更详细一些的。
    

已无更多数据
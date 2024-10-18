
大家好，我是飞哥！

在[深入了解服务器 CPU 的型号、代际、片内与片间互联架构](https://mp.weixin.qq.com/s/7nwiX3JaGwL-EOZz77GohA)一文中我们了解了服务器 CPU 的内部架构。在其中我们看到有一个内存控制器。

关于CPU内存控制器中会有很多专技术细节。例如下面是Skylake 代际 CPU 对内存的支持情况。参见[https://en.wikichip.org/wiki/intel/microarchitectures/skylake\_](https://en.wikichip.org/wiki/intel/microarchitectures/skylake_)(server)

```c
DRAM
  6 channels of DDR4, up to 2666 MT/s
  RDIMM and LRDIMM
  bandwidth of 21.33 GB/s
  aggregated bandwidth of 128 GB/s
```

对于这个输出我们给大家抛出几个问题

- 其中的 6 channle 是什么意思？
- RDIMM、LRDIMM 又分别代表的是什么内存？
- 为什么内存带宽中 bandwidth 是 21.33 GB/s，aggregated bandwidth 128 GB/s？

所以今天我们就详细展开对 CPU 内存控制器相关参数展开介绍。

## 内存通道数与带宽

下图是 Skylake 的 CPU 的总体结构图。

![图00.webp](https://kfngxl.cn/usr/uploads/2024/04/3147808058.webp "图00.webp")

该 CPU 有两个内存控制器（IMC，Integrate Memory Controller）。每个内存控制器上都有一个 DDR PHY。DDR PHY 是连接 DDR 内存条和 内存控制器的桥梁。它负责把内存控制器发过来的数据转化成符合 DDR 协议的信号，并传给内存颗粒。也负责把内存返回给 CPU 的数据转化成内存控制器认识的信号，最终交给 CPU 核来处理。

每个 DDR PHY 有 3 个 DDR4 channels（通道）。每个 channel 有两个内存插槽，也就是说可接 2 条 DIMM（关于 DIMM 后面会介绍，这里把它理解为一个内存条就可以了）。

所以该代际 CPU 总共有 6 channels，最多可支持插入 12 条内存条。

每个内存通道都是可以独立工作的。该 CPU 支持的内存数据频率是 2666MT/s，理论上每秒钟可以传输 2666M 次数据。由于现在都是 64 位的计算机。所以，可以算得

- 单通道内存带宽 = 2666M _64 比特 = 2666M_ 8 字节 = 21.33 GB/s
- 6个通道的总带宽 = 21.33 GB/s * 6 = 128 GB/s

## 内存条模块规格

我们再来看 Skylake 内存控制器中支持的 RDIMM and LRDIMM。 那么 RDIMM 和 LRDIMM 类型的内存是什么呢？ 我们先来看 DIMM。

DIMM 是双列直插内存模块，是现代最常用的内存条模块的规格，英文全名 Dual In-Line Memory Module。表示的是信号接触在金手指两侧，并且在 DIMM 条的边沿作为信号接触面。

根据它的名称可以看出历史上曾出现过 SIMM（Single-Inline Memory Module）。SIMM 的位宽是 32 位，这是 32 位机时代的产物。到了 64 位机时代后，就统一都用 DIMM 了。Dual 的意思是 32 位的双倍，64 位。这种规格一致延续至今。

针对不同的应用场景，内存条的标准也是不太一样的。大致可以分为如下几种。

**UDIMM：无缓冲双列直插内存模块**，是 Unbuffered DIMM 的缩写。

指地址和控制信号不经缓冲器，无需做任何时序调整，直接到达 DIMM 上的各个 DRAM 芯片。这种内存要求 CPU 到每个内存颗粒之间的传输距离相等，这样并行传输才有效。而保证 CPU 到每个颗粒之间传输距离需要较高的制造工艺，这样就对内存的容量和频率都产生了限制。这种内存由于容量小，所以在个人台式机上用的比较多。

下图是一个台式机的 UDIMM 16GB 内存条。该内存条背面是空的，总共有八个黑色的内存颗粒。

![图3.png](https://kfngxl.cn/usr/uploads/2024/04/3333515706.png "图3.png")

另外上面这个内存条还标识了 16 GB 2R\*8 PC4-3200AA-U82-11。

**SO-DIMM：小外形模块**，是 Small Outline DIMM 的缩写。

在笔记本电脑出现后，对内存的体积和功耗都要求更小一些。SO-DIMM 就是针对笔记本电脑定义的标准。其宽度标准是 67.6 mm。如下图是两个笔记本内存条，可见体积要比台式机小不少。

![SODIMM_1R*8.png](https://kfngxl.cn/usr/uploads/2024/04/2513320475.png "SODIMM_1R*8.png")\
![SODIMM_1R*16.png](https://kfngxl.cn/usr/uploads/2024/04/1732418588.png "SODIMM_1R*16.png")

**RDIMM：带寄存器双列直插模块**，是 Registered DIMM 的缩写。\
RDIMM 在内存条上加了一个寄存缓存器（RCD，Register Clock Driver）进行传输。控制器输出的地址和控制信号经过Register芯片寄存后输出到DRAM芯片。

CPU 访问数据时都先经过寄存器再到内存颗粒。减少了 CPU 到内存颗粒的距离，使得频率可以提高。而且不再像之前一样要求每个内存颗粒传输距离相等，工艺复杂度因寄存缓存器的引入而下降，使得容量也可以提高到 32 GB。主要用在服务器上。

下图是一个服务器RDIMM 32 GB 内存条。这个服务器内存条不光正面有很多内存颗粒，连背面也有。可见服务器内存的颗粒数量比普通笔记本电脑、个人台式机的颗粒都要多很多。最关键的是内存条正中央位置的较大颗粒的寄存缓存器，表明了这是一条RDIMM内存。

![图4_1.png](https://kfngxl.cn/usr/uploads/2024/04/3434529732.png "图4_1.png")\
![图4_2.png](https://kfngxl.cn/usr/uploads/2024/04/3468177107.png "图4_2.png")

从图中可见内存的参数标识是 32 GB 2R\*4 PC4-2666V-RB2-12-DB1。

**LRDIMM：低负载双列直插内存模块**，是 Load Reduced DIMM 的缩写。

LRDIMM 相比 RDIMM 在引入寄存缓存器 RCD 的基础上，又进一步引入了数据缓冲器 DB（Data Buffer）。引入数据缓冲器作用是缓冲来自内存控制器或内存颗粒的数据信号。实现了对地址、控制信号、数据的全缓冲。成本更高，但可以支持更大容量。

![LRDIMM_2.png](https://kfngxl.cn/usr/uploads/2024/04/3382489289.png "LRDIMM_2.png")

## ECC 内存

DRAM 内存是一种易失性的存储，它是根据电容的电位高低来判断存储的是 0 或 1 的。但是电容电位虽然有定时刷新来作为保障，却仍然不能保证其读取出来的数据和当时存进去的一致。如果出现了不一致，那这种现象就叫做比特翻转。一根 8 GB 的内存条平均大约每小时会出现 1 - 5 个这样的错误。

我们个人在办公的时候，由于内存主要都用来处理图片、视频等数据。即使内存出现了比特翻转，可能影响的只是一个像素值，没有太大的影响，也很难感觉出来。

在服务器应用中，处理的一般都是非常重要的计算，可能是一些推荐计算，也可能是一笔订单交易，对出错的容忍度是很低的。另外一台服务器经常是连续要运行几个月甚至是几年。因此总的来说，服务器对稳定性的要求极高，不允许比特翻转错误发生。

ECC 是一种内存专用的技术。它的英文全称是 “Error Checking and Correcting”，对应的中文名称就叫做“错误检查和纠正”。从它的名称中我们可以看出，ECC 不但能发现内存中的错误，而且还可以进行纠正。

在实现上，ECC 内存会板上额外再添加一个内存颗粒来专门负责检查错误并纠正错误。

![图7_1.png](https://kfngxl.cn/usr/uploads/2024/04/2772844672.png "图7_1.png")

而不带 ECC 的功能是没有多出来的这个颗粒的。

![图7_2.png](https://kfngxl.cn/usr/uploads/2024/04/874422032.png "图7_2.png")

CPU 每个 channel 支持同时支持 72 位的读写，其中 64 位是数据，另外 8 位用于 ECC 校验。

由于有额外的硬件引入。所以 ECC 内存的价格会比普通内存要贵一些，速度也会慢 2% 左右。

## 总结

服务器 CPU 比普通家用 CPU 贵的原因之一就是它对内存的支持和普通家用 CPU 不一样。

首先就是服务器的 CPU 对内存通道数的支持。普通家用 CPU 一般只有双通道，最多也是四通道。而本文中提到的 Skylake 是 2015 年的服务器 CPU，就已经支持了多达 6 个内存通道，最多可以支持 12 个内存条。2023 年 1 月发布的第四代英特尔至强（Intel Xeon）更是支持了 8 内存通道。可以插更多的内存条。

另外就是服务器模块。服务器 CPU 支持 RDIMM（带寄存器双列直插模块）和 LRDIMM（低负载双列直插内存模块）内存。这两种内存单条都有更大的容量。

![compare.png](https://kfngxl.cn/usr/uploads/2024/04/1824851540.png "compare.png")

另外就是服务器几乎全系都支持 ECC 内存。而家用 CPU 只有最近几年才开始支持 ECC。

我们再回到开篇提到的三个问题。

**问题1：其中的 6 channle 是什么意思？**\
这值得是 CPU 支持的内存通道数量为 6 ，不同的通道可以并行工作，通道数量越高，访问内存性能越好。

**问题2：RDIMM、LRDIMM 又分别代表的是什么内存？**\
RDIMM、LRDIMM 是在控制信号上、或者数据信号上加了一些硬件缓存，有了这些缓存可以支持单条做到更大的容量。制作工艺比较复杂，价格也会偏贵一些。

**问题3：为什么内存带宽中 bandwidth 是 21.33 GB/s，aggregated bandwidth 128 GB/s？**\
单通道内存的带宽是根据内存的数据频率计算出来的，由于数据频率是 2666M，所以算得单通道带宽为 21.33 GB/s。由于总共有 6 个通道，所以总的带宽可以达到 128 GB/s。

不过要注意的是，厂商的参数中都指的之理论最大带宽。而实际运行的过程中，内存硬件中会有各种延迟，实际带宽到不了这么高。

更多干货内容，详见：\
Github：[https://github.com/yanfeizhang/coder-kung-fu](https://github.com/yanfeizhang/coder-kung-fu)\
关注公众号：微信扫描下方二维码\
![qrcode2_640.png](https://kfngxl.cn/usr/uploads/2024/05/4275823318.png "qrcode2_640.png")

本原创文章未经允许不得转载 | 当前页面：[开发内功修炼@张彦飞](https://kfngxl.cn/) » [看懂服务器 CPU 内存支持，学会计算内存带宽](https://kfngxl.cn/index.php/archives/787/)

标签：没有标签

![张彦飞（@开发内功修炼）](https://secure.gravatar.com/avatar/23c60606a05a1e9b9fac9cadbd055ad7?s=50&r=g)

#### 张彦飞（@开发内功修炼）

腾讯、搜狗、字节等公司13年工作经验，著有《深入理解Linux网络》一书，个人技术公众号「开发内功修炼」

上一篇：[磁盘开篇：扒开机械硬盘坚硬的外衣！](https://kfngxl.cn/index.php/archives/774/ "磁盘开篇：扒开机械硬盘坚硬的外衣！")下一篇：[理解内存的Rank、位宽以及内存颗粒内部结构](https://kfngxl.cn/index.php/archives/798/ "理解内存的Rank、位宽以及内存颗粒内部结构")

### 相关推荐

- [C语言竟可以调用Go语言函数，这是如何实现的？](https://kfngxl.cn/index.php/archives/810/ "C语言竟可以调用Go语言函数，这是如何实现的？")
- [理解内存的Rank、位宽以及内存颗粒内部结构](https://kfngxl.cn/index.php/archives/798/ "理解内存的Rank、位宽以及内存颗粒内部结构")
- [磁盘开篇：扒开机械硬盘坚硬的外衣！](https://kfngxl.cn/index.php/archives/774/ "磁盘开篇：扒开机械硬盘坚硬的外衣！")
- [经典，Linux文件系统十问](https://kfngxl.cn/index.php/archives/769/ "经典，Linux文件系统十问")
- [Linux进程是如何创建出来的？](https://kfngxl.cn/index.php/archives/687/ "Linux进程是如何创建出来的？")
- [内核是如何给容器中的进程分配CPU资源的？](https://kfngxl.cn/index.php/archives/752/ "内核是如何给容器中的进程分配CPU资源的？")
- [Docker容器里进程的 pid 是如何申请出来的？](https://kfngxl.cn/index.php/archives/745/ "Docker容器里进程的 pid 是如何申请出来的？")
- [为什么新版内核将进程pid管理从bitmap替换成了radix-tree？](https://kfngxl.cn/index.php/archives/738/ "为什么新版内核将进程pid管理从bitmap替换成了radix-tree？")

### 标签云

[内存硬件 （1）](https://kfngxl.cn/index.php/tag/%E5%86%85%E5%AD%98%E7%A1%AC%E4%BB%B6/)[服务器 （1）](https://kfngxl.cn/index.php/tag/%E6%9C%8D%E5%8A%A1%E5%99%A8/)[技术面试 （1）](https://kfngxl.cn/index.php/tag/%E6%8A%80%E6%9C%AF%E9%9D%A2%E8%AF%95/)[同步阻塞 （1）](https://kfngxl.cn/index.php/tag/%E5%90%8C%E6%AD%A5%E9%98%BB%E5%A1%9E/)[进程 （1）](https://kfngxl.cn/index.php/tag/%E8%BF%9B%E7%A8%8B/)

- 最新文章

- - 06-13[C语言竟可以调用Go语言函数，这是如何实现的？](https://kfngxl.cn/index.php/archives/810/ "C语言竟可以调用Go语言函数，这是如何实现的？")
    - 05-13[理解内存的Rank、位宽以及内存颗粒内部结构](https://kfngxl.cn/index.php/archives/798/ "理解内存的Rank、位宽以及内存颗粒内部结构")
    - 05-13[看懂服务器 CPU 内存支持，学会计算内存带宽](https://kfngxl.cn/index.php/archives/787/ "看懂服务器 CPU 内存支持，学会计算内存带宽")
    - 04-09[磁盘开篇：扒开机械硬盘坚硬的外衣！](https://kfngxl.cn/index.php/archives/774/ "磁盘开篇：扒开机械硬盘坚硬的外衣！")
    - 04-08[经典，Linux文件系统十问](https://kfngxl.cn/index.php/archives/769/ "经典，Linux文件系统十问")

- 站点统计

- - 文章总数：87篇
    - 分类总数：3个
    - 总访问量：36919次
    - 本站运营：0年168天17小时

© 2010 - 2024 [开发内功修炼@张彦飞](https://kfngxl.cn/) | [京ICP备2024054136号](http://beian.miit.gov.cn/)\
本站部分图片、文章来源于网络，版权归原作者所有，如有侵权，请联系我们删除。

- ###### 去顶部

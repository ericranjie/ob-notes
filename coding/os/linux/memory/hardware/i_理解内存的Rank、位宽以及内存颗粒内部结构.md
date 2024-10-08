2024-05-13 [内存篇](https://kfngxl.cn/index.php/category/memory/) 阅读(437) 评论(0)

大家好，我是飞哥！

在冯诺依曼体系结构里，内存是除了CPU之外第二重要的设备。如果没有内存，服务器将完全无法运行。在这一节中，我们来了解下内存的物理结构。如下图的是一个 16 GB 的笔记本内存条实物的正面和反面图。其中的每个黑色颗粒也叫一个 Chip。

![图2_1.png](https://kfngxl.cn/usr/uploads/2024/05/1152699679.png "图2_1.png")  
![图2_2.png](https://kfngxl.cn/usr/uploads/2024/05/854680675.png "图2_2.png")

注意下，在正面有着一串字符串标识`16 GB 2R\*8 PC4-3200AA-SE1-11`。在这段标识中，16 GB 很好理解，是内存的容量大小。那么后面的 2R*8 是什么意思呢？

实际上，内存标识第二段中的 2R*8 非常重要，它直接简单清晰地把内存的物理结构给表示出来了。

- 2R：表示该内存有 2 个 Rank
- *8：表示每个内存颗粒的位宽是 8 bit，

接下来我们分两个小节，深入地看看 Rank、位宽与内存颗粒。

## 内存的 Rank 与位宽

在内存中，其中每一个黑色的内存颗粒叫一个 Chip。所谓 Rank 指的是属于同一个组的 Chip 的总和。这些 Chip 并行工作，共同组成组成一个 64 bit 的数据，供 CPU 来同时读取。

CPU 的内存控制器能够对同一个 rank 的 chip 进行读写操作。通常一个通道（channel）能够同时读写 64bit 的数据（ECC 功能的是 72 bit）。

**内存字符串标识中的 2 R 表示该内存有 2 个 Rank**。

2 R 后面的 * 4 表示每个内存颗粒的位宽是 4 bit。因为 CPU 要同时读写 64 bit 的数据。所以

- 对于位宽为 4 的颗粒，需要 16 个 Chip 来组成一个 Rank
- 对于位宽为 8 的颗粒，需要 6 个 Chip 来组成一个 Rank
- 对于位宽为 16 的颗粒，需要 4 个 Chip 来组成一个 Rank

例如，下面的笔记本内存条，是 1 R * 16。表示的是该内存条只有 1 个 Rank。每个 Chip 内存颗粒的位宽是 16 bit。

而一个 Rank 需要提供 64 位的数据，则需要 64 / 16 = 4 个 Chip 来组成一个 Rank 来同步地工作。从实物图中也确实可以看到，该内存条正反面加起来只有 4 个 Chip，

![图3_1.png](https://kfngxl.cn/usr/uploads/2024/05/1657551640.png "图3_1.png")  
![图3_2.png](https://kfngxl.cn/usr/uploads/2024/05/580694467.png "图3_2.png")

再比如，下面的笔记本内存条，是 2 R * 8。表示的是该内存条有 2 个 Rank，每个 Chip 内存颗粒的位宽是 8 bit。

一个 Rank 需要 64 / 8 = 8 个 Chip 来组成一个 Rank。则两个 Rank 总共需要 16 个 Chip。从内存条的实物图中看到，该内存条的正反面确实总共有 16 个 Chip。  
![图4_1.png](https://kfngxl.cn/usr/uploads/2024/05/2379430074.png "图4_1.png")  
![图4_2.png](https://kfngxl.cn/usr/uploads/2024/05/1536767034.png "图4_2.png")

## 内存颗粒 Chip 内部结构

一个内存是由若干个黑色的内存颗粒构成的。每一个内存颗粒叫做一个 chip。在每个 chip 内部，又是由一层层的 bank 组成的。

![图5.png](https://kfngxl.cn/usr/uploads/2024/05/2145136395.png "图5.png")

在每个 bank 内部，就是电容的行列矩阵结构了。   
![图6.png](https://kfngxl.cn/usr/uploads/2024/05/3003292243.png "图6.png")

这个矩阵由多个方块状的元素构成，这个方块元素是内存管理的最小单位，也叫内存颗粒位宽。在一个位宽中。有若干小电容。

- 对于 1 R * 16 的内存条，一个位宽有 16 个 bit 位
- 对于 2 R * 8 的内存条，一个位宽有 8 个 bit 位

值得注意的是，由于内存访问太慢了。所以 CPU 每次向内存请求数据的时候，并不只是请求一个 64 bit 的数据就完事了，而是会请求更多的数据然后用自己的 L1、L2、L3等模块缓存起来。下次如果访问的数据位于缓存中的话，就可以不用再发起内存 IO 了。一次请求的数据大小是 64 * 8 bit = 64 字节，这也是一个 Cache Line 的大小。

对于内存来说，一次 Cache Line 64 字节的访问属于是一次 Burst IO，需要内存连续工作多次，输出多个 64 字节。所以，内存在排列和组织二维矩阵结构的时候，会按方便 Burst IO 的方式来组织，实际二维矩阵单元中存储的字节数会比位宽要大。

例如下面是一个美光（Megon）内存 Chip 的内部结构。

![图7.png](https://kfngxl.cn/usr/uploads/2024/05/3816561433.png "图7.png")

在该 Chip 中，总共有 8 个 bank，每个 bank 是一个 32768 行 * 128 列的二维矩阵，每个二维矩阵单元存储的数据大小是 64 比特。

则该 Chip 总共可存储的数据大小是 8 _32768_ 128 * 64 = 2147483648 比特。   
换算成 MiB 2147483648 字节/(1024*1024*8) = 256 MiB

## 总结

内存标识字符串中的第二段是非常重要的表示内存物理结构的标识。它清楚地写明了当前内存条总共有几个 Rank，每个 Chip 中的位宽是多少。进而也能推算出 1 个 Rank 中有多少个 Chip 组成。例如

- 2R*4 表示的是内存条有 2 个 Rank，每个 Chip 的位宽大小是 4。可以推算出每个 Rank 需要 64/4 = 16 个 Chip 颗粒。这种内存常见于服务器内存。内存颗粒越多，就可以组成更大容量的内存条。
- 2R*8 表示的是内存条有 2 个 Rank，每个 Chip 的位宽大小是 8。可以推算出每个 Rank 需要 64/8 = 8 个 Chip 颗粒。这种规格常见于台式机。
- 1R*16 表示的是内存条有 1 个 Rank，每个 Chip 的位宽大小是 16。可以推算出每个 Rank 需要 64/16 = 4 个 Chip 颗粒。这种内存常见于笔记本内存条。因为内存颗粒越少，则体积越小。

至于每个 Chip 内存颗粒中有多少个二维矩阵元素，为了支持 Burst IO，也为了节约地址线数量。一般每个二维矩阵元素中存储的数据要比位宽更大一些。

---



更多干货内容，详见：  
Github：[https://github.com/yanfeizhang/coder-kung-fu](https://github.com/yanfeizhang/coder-kung-fu)  
关注公众号：微信扫描下方二维码  
![qrcode2_640.png](https://kfngxl.cn/usr/uploads/2024/05/4275823318.png "qrcode2_640.png")

本原创文章未经允许不得转载 | 当前页面：[开发内功修炼@张彦飞](https://kfngxl.cn/) » [理解内存的Rank、位宽以及内存颗粒内部结构](https://kfngxl.cn/index.php/archives/798/)

标签：没有标签

![张彦飞（@开发内功修炼）](https://secure.gravatar.com/avatar/23c60606a05a1e9b9fac9cadbd055ad7?s=50&r=g)

#### 张彦飞（@开发内功修炼）

腾讯、搜狗、字节等公司13年工作经验，著有《深入理解Linux网络》一书，个人技术公众号「开发内功修炼」

上一篇：[看懂服务器 CPU 内存支持，学会计算内存带宽](https://kfngxl.cn/index.php/archives/787/ "看懂服务器 CPU 内存支持，学会计算内存带宽")下一篇：[C语言竟可以调用Go语言函数，这是如何实现的？](https://kfngxl.cn/index.php/archives/810/ "C语言竟可以调用Go语言函数，这是如何实现的？")

### 相关推荐

- [C语言竟可以调用Go语言函数，这是如何实现的？](https://kfngxl.cn/index.php/archives/810/ "C语言竟可以调用Go语言函数，这是如何实现的？")
- [看懂服务器 CPU 内存支持，学会计算内存带宽](https://kfngxl.cn/index.php/archives/787/ "看懂服务器 CPU 内存支持，学会计算内存带宽")
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
    - 总访问量：36138次
    - 本站运营：0年163天12小时

© 2010 - 2024 [开发内功修炼@张彦飞](https://kfngxl.cn/) | [京ICP备2024054136号](http://beian.miit.gov.cn/)  
本站部分图片、文章来源于网络，版权归原作者所有，如有侵权，请联系我们删除。
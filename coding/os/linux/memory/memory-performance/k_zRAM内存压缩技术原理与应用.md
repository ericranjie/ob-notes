
作者：[OPPO内核团队](http://www.wowotech.net/author/538) 发布于：2020-3-8 8:38 分类：[内存管理](http://www.wowotech.net/sort/memory_management)

# 1. **技术背景**

说到压缩这个词，我们并不陌生，应该都能想到是降低占用空间，使同样的空间可以存放更多的东西，类似于我们平时常用的文件压缩,内存压缩同样也是为了节省内存。

尽管当前android手机6GB，8GB甚至12GB的机器都较为常见了，但内存无论多大，总是会有不够用的时候。当系统内存紧张的时候，会将文件页丢弃或回写回磁盘（如果是脏页），还可能会触发LMK杀进程进行内存回收。这些被回收的内存如果再次使用都需要重新从磁盘读取，而这个过程涉及到较多的IO操作。就目前的技术而言，IO的速度远远慢于这RAM操作速度。因此，如果频繁地做IO操作，不仅影响flash使用寿命，还严重影响系统性能。内存压缩是一种让IO过程平滑过渡的做法, 即尽量减少由于内存紧张导致的IO，提升性能。

# 2. **主流内存压缩技术**

目前linux内核主流的内存压缩技术主要有3种：zSwap, zRAM, zCache。

## 2.1 **zSwap**

zSwap是在memory与flash之间的一层“cache”,当内存需要swap出去磁盘的时候，先通过压缩放到zSwap中去，zSwap空间按需增长。达到一定程度后则会按照LRU的顺序(前提是使用的内存分配方法需要支持LRU)将就最旧的page解压写入磁盘swap device，之后将当前的page压缩写入zSwap。

![[Pasted image 20241021202149.png]]

zswap本身存在一些缺陷或问题:

1. 如果开启当zswap满交换出backing store的功能, 由于需要将zswap里的内存按LRU顺序解压再swap out, 这就要求内存分配器支持LRU功能。
1. 如果不开启当zswap满交换出backing store的功能, 和zRam是类似的。

## 2.2 **zRram**

zRram即压缩的内存， 使用内存模拟block device的做法。实际不会写到块设备中去，只会压缩后写到模拟的块设备中，其实也就是还是在RAM中，只是通过压缩了。由于压缩和解压缩的速度远比读写IO好，因此在移动终端设备广泛被应用。zRam是基于RAM的block device, 一般swap priority会比较高。只有当其满，系统才会考虑其他的swap devices。当然这个优先级用户可以配置。

zRram本身存在一些缺陷或问题:

1. zRam大小是可灵活配置的, 那是不是配置越大越好呢? 如果不是,配置多大是最合适的呢?
1. 使用zRam可能会在低内存场景由于频繁的内存压缩导致kswapd进程占CPU高, 怎样改善?
1. 增大了zRam配置,对系统内存碎片是否有影响?

要利用好zRam功能, 并不是简单地配置了就OK了, 还需要对各种场景和问题都做好处理, 才能发挥最优的效果。

## 2.3 **zCache**

zCache是oracle提出的一种实现文件页压缩技术，也是memory与block dev之间的一层“cache”,与zswap比较接近，但zcache目前压缩的是文件页，而zSwap和zRAM压缩是匿名页。

zcache本身存在一些缺陷或问题:

1. 有些文件页可能本身是压缩的内容, 这时可能无法再进行压缩了
1. zCache目前无法使用zsmalloc, 如果使用zbud,压缩率较低
1. 使用的zbud/z3fold分配的内存是不可移动的, 需要关注内存碎片问题

# 3. **内存压缩主流的内存分配器

### **3.2.1** **Zsmalloc**

zsmalloc是为ZRAM设计的一种内存分配器。内核已经有slub了， 为什么还需要zsmalloc内存分配器？这是由内存压缩的场景和特点决定的。zsmalloc内存分配器期望在低内存的场景也能很好地工作，事实上，当需要压缩内存进行zsmalloc内存分配时，内存一般都比较紧张且内存碎片都比较严重了。如果使用slub分配， 很可能由于高阶内存分配不到而失败。另外，slub也可能导致内存碎片浪费比较严重，最坏情况下，当对象大小略大于PAGE_SIZE/2时，每个内存页接近一般的内存将被浪费。

Android手机实测发现，anon pages的平均压缩比大约在1:3左右，所以compressed anon page size很多在1.2K左右。如果是Slub，为了分配大量1.2K的内存，可能内存浪费严重。zsmalloc分配器尝试将多个相同大小的对象存放在组合页（称为zspage）中，这个组合页不要求物理连续，从而提高内存的使用率。

![[Pasted image 20241021202248.png]]

需要注意的是, 当前zsmalloc不支持LRU功能, 旧版本内核分配的不可移动的页, 对内存碎片影响严重, 但最新版本内核已经是支持分配可移动类型内存了。

### **3.2.2 Zbud**

zbud是一个专门为存储压缩page而设计的内存分配器。用于将2个objects存到1个单独的page中。zbud是可以支持LRU的, 但分配的内存是不可移动的。

### **3.2.3 Z3fold**

z3fold是一个较新的内存分配器, 与zbud不同的是, 将3个objects存到1个单独的page中,也就是zbud内存利用率极限是1:2, z3fold极限是1:3。同样z3fold是可以支持LRU的, 但分配的内存是不可移动的。

# 4. **内存压缩技术与内存分配器组合对比分析**

结合上面zSwap / zRam /zCache的介绍, 与zsmalloc/zbud/z3fold分别怎样组合最合适呢?

下面总结了一下, 具体原因可以看上面介绍的时候各类型的特点。

|   |   |   |   |
|---|---|---|---|
||zsmalloc|zbud|z3fold|
|zSwap<br><br>（有实际swap device）|×(不可用)|√(可用)|√(最佳)|
|zSwap<br><br>（无实际swap device）|√(最佳)|√(可用)|√(可用)|
|zRam|√(最佳)|√(可用)|√(可用)|
|zCache|×(不可用)|√(可用)|√(最佳)|

# 5. **zRAM技术原理**

本文重点介绍zRam内存压缩技术，它是目前移动终端广泛使用的内存压缩技术。

## 5.1 **软件框架**

下图展示了内存管理大体的框架， 内存压缩技术处于内存回收memory reclaim部分中。

![[Pasted image 20241021202328.png]]

再具体到zRam, 它的软件架构可以分为3部分， 分别是数据流操作，内存压缩算法 ，zram驱动。

![[Pasted image 20241021202342.png]]

数据流操作:提供串行或者并行的压缩和解压操作。

内存压缩算法：每种压缩算法提供压缩和解压缩的具体实现回调接口供数据操作调用。

Zram驱动：创建一个基于ram的块设备， 并提供IO请求处理接口。

## 5.2 **实现原理**

Zram内存压缩技术本质上就是以时间换空间。通过CPU压缩、解压缩的开销换取更大的可用内存空间。

我们主要描述清楚下面这2个问题：

1） 什么时候会进行内存压缩？

2） 进行内存压缩/解压缩的流程是怎样的？

进行内存压缩的时机：

1） Kswapd场景：kswapd是内核内存回收线程， 当内存watermark低于low水线时会被唤醒工作， 其到内存watermark不小于high水线。

2） Direct reclaim场景：内存分配过程进入slowpath, 进行直接行内存回收。

![[Pasted image 20241021202409.png]]

下面是基于4.4内核理出的内存压缩、解压缩流程。

内存回收过程路径进行内存压缩。会将非活跃链表的页进行shrink, 如果是匿名页会进行pageout, 由此进行内存压缩存放到ZRAM中， 调用路径如下：

![[Pasted image 20241021202427.png]]

在匿名页换出到swap设备后， 访问页时， 产生页访问错误, 当发现“页表项不为空， 但页不在内存中”， 该页就是已换到swap区中，由此会开始将该页从swap区中重新读取， 如果是ZRAM， 则是解压缩的过程。调用路径如下：

![[Pasted image 20241021202551.png]]

## 5.3 **内存压缩算法**

目前比较主流的内存算法主要为LZ0, LZ4, ZSTD等。下面截取了几种算法在x86机器上的表现。各算法有各自特点， 有以压缩率高的， 有压缩/解压快的等， 具体要结合需求场景选择使用。

![[Pasted image 20241021202611.png]]

# 6. **zRAM技术应用**

本节描述一下在使用ZRAM常遇到的一些使用或配置，调试的方法。

## 6.1 **如何配置开启zRAM**

1） **配置内存压缩算法**

下面例子配置压缩算法为lz4

echo lz4 > /sys/block/zram0/comp_algorithm

2） **配置ZRAM大小**

下面例子配置zram大小为2GB

echo 2147483648 > /sys/block/zram0/disksize

3） **使能zram**

mkswap /dev/zram0

swapon /dev/zram0

## 6.2 **swappiness含义简述**

swappiness参数是内核倾向于回收匿名页到swap（使用的ZRAM就是swap设备）的积极程度， 原生内核范围是0~100， 参数值越大， 表示回收匿名页到swap的比例就越大。如果配置为0， 表示仅回收文件页，不回收匿名页。默认值为60。可以通过节点“/proc/sys/vm/swappiness”配置。

## 6.3 **zRam相关的技术指标**

### 1） **ZRAM大小及剩余空间**

Proc/meminfo中可以查看相关信息

SwapTotal：swap总大小, 如果配置为ZRAM, 这里就是ZRAM总大小

SwapFree：swap剩余大小, 如果配置为ZRAM, 这里就是ZRAM剩余大小

当然， 节点 /sys/block/zram0/disksize是最直接的。

### 2） **ZRAM压缩率**

/sys/block/zram/mm_stat中有压缩前后的大小数据， 由此可以计算出实际的压缩率

orig_data_size：压缩前数据大小， 单位为bytes

compr_data_size ：压缩后数据大小， 单位为bytes

### 3） **换出/换入swap区的总量, proc/vmstat中中有相关信息**

pswpin:换入总量， 单位为page

pswout:换出总量， 单位为page

## 6.4 **zRam相关优化**

上面提到zRam的一些缺陷, 怎么去改善呢?

1. zRam大小是可灵活配置的, 那是不是配置越大越好呢? 如果不是配置多大是最合适的呢?

zRam大小的配置比较灵活, 如果zRam配置过大, 后台缓存了应用过多, 这也是有可能会影响前台应用使用的流畅度。另外, zRam配置越大, 也需要关注系统的内存碎片化情。因此zRam并不是配置越大越好,具体的大小需要根据内存总大小及系统负载情况考虑及实测而定。

2. 使用zRam,可能会存在低内存场景由于频繁的内存压缩导致kswapd进程占CPU高, 怎样改善?

zRam本质就是以时间换空间, 在低内存的情况下, 肯定会比较频繁地回收内存, 这时kswapd进程是比较活跃的, 再加上通过压缩内存, 会更加消耗CPU资源。 改善这种情况方法也比较多, 比如, 可以使用更优的压缩算法, 区别使用场景, 后台不影响用户使用的场景异步进行深度内存压缩, 与用户体验相关的场景同步适当减少内存压缩, 通过增加文件页的回收比例加快内存回收等等。

3. 增大了zRam配置,对系统内存碎片是否有影响?

使用zRam是有可能导致系统内存碎片变得更严重的, 特别是zsmalloc分配不支持可移动内存类型的时候。新版的内核zsmalloc已经支持可移动类型分配的， 但由于增大了zRam,结合android手机的使用特点, 仍然会有可能导致系统内存碎片较严重的情况,因些内存碎片问题也是需要重点关注的。解决系统内存碎片的方法也比较多, 可以结合具体的原因及场景进行优化。

# 7. **参考资料**

1) [https://github.com/lz4/lz4](https://github.com/lz4/lz4)

2. kernel\\Documentation\\blockdev\\zram.txt

1. kernel\\Documentation\\vm\\zswap.txt

1. kernel\\Documentation\\sysctl\\vm.txt

标签: [zRAM](http://www.wowotech.net/tag/zRAM)

---

« [一例centos7.6内核hardlock的解析](http://www.wowotech.net/linux_kenrel/479.html) | [Binder从入门到放弃（下）](http://www.wowotech.net/binder2.html)»

**评论：**

**[bsp](http://www.wowotech.net/)**\
2023-01-04 17:19

内存压缩的调用路径中，add_to_swap并不会直接call pageout，add_to_swap只是将page变成SwapPage并和swap_entry建立对应关系；\
并且，add_to_swap时，page对应的mapping还未unmap；\
流程大致是：先add_to_swap，后try_to_unmap，再去call pageout；\
以上基于kernel-5.4;

[回复](http://www.wowotech.net/memory_management/zram.html#comment-8728)

**[bsp](http://www.wowotech.net/)**\
2023-02-17 15:20

@bsp：swap & page fault流程\
1:将待swap的page 加入到swapcache中；\
2:解除task和该page的映射关系；\
3:通过pageout 进行zram压缩，并将此page压到某个buffer中；\
4:压缩完成后，选择性从swapcache中free该page（比如swapcache满了）；

第2步之后，task如果访问该page，则会触发pagefault；\
如果该page还在swapcache中，则将task和该page重新建立映射关系即可，即minor fault；\
如果该page不在swapcache中，则会重申请一个新的page，并通过zram解压缩、将之前压缩后的buffer解压到此page中，即major fault；

[回复](http://www.wowotech.net/memory_management/zram.html#comment-8744)

**ziliang**\
2023-09-13 22:18

@bsp：大佬讲的好详细呀

[回复](http://www.wowotech.net/memory_management/zram.html#comment-8820)

**jzp0409**\
2021-08-03 16:50

write-back，能否介绍一下？

[回复](http://www.wowotech.net/memory_management/zram.html#comment-8264)

**遥望西伯利亚**\
2020-11-19 11:43

我想修改ZRAM的大小和算法，但是怎么测试该更后的性能呢？\
希望能答疑解惑，感谢！

[回复](http://www.wowotech.net/memory_management/zram.html#comment-8140)

**BoomMINote**\
2020-04-17 15:08

感谢

[回复](http://www.wowotech.net/memory_management/zram.html#comment-7961)

**hyde**\
2020-03-19 17:17

请问下微信是多少，我想加你了解下

[回复](http://www.wowotech.net/memory_management/zram.html#comment-7923)

**发表评论：**

昵称

邮件地址 (选填)

个人主页 (选填)

![](http://www.wowotech.net/include/lib/checkcode.php)

- ### 站内搜索

  蜗窝站内  互联网

- ### 功能

  [留言板\
  ](http://www.wowotech.net/message_board.html)[评论列表\
  ](http://www.wowotech.net/?plugin=commentlist)[支持者列表\
  ](http://www.wowotech.net/support_list)

- ### 最新评论

  - ja\
    [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
  - 元神高手\
    [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
  - 十七\
    [内核空间的映射在系统启动时就已经设定好，并且在所有进程的页表...](http://www.wowotech.net/process_management/context-switch-arch.html#8920)
  - lw\
    [sparse模型和disconti模型没看出来有什么本质区别...](http://www.wowotech.net/memory_management/memory_model.html#8919)
  - 肥饶\
    [一个没设置好就出错](http://www.wowotech.net/linux_kenrel/516.html#8918)
  - orange\
    [点赞点赞，对linuxer的文章总结到位](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html#8917)

- ### 文章分类

  - [Linux内核分析(25)](http://www.wowotech.net/sort/linux_kenrel) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=4)
    - [统一设备模型(15)](http://www.wowotech.net/sort/device_model) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=12)
    - [电源管理子系统(43)](http://www.wowotech.net/sort/pm_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=13)
    - [中断子系统(15)](http://www.wowotech.net/sort/irq_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=14)
    - [进程管理(31)](http://www.wowotech.net/sort/process_management) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=15)
    - [内核同步机制(26)](http://www.wowotech.net/sort/kernel_synchronization) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=16)
    - [GPIO子系统(5)](http://www.wowotech.net/sort/gpio_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=17)
    - [时间子系统(14)](http://www.wowotech.net/sort/timer_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=18)
    - [通信类协议(7)](http://www.wowotech.net/sort/comm) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=20)
    - [内存管理(31)](http://www.wowotech.net/sort/memory_management) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=21)
    - [图形子系统(2)](http://www.wowotech.net/sort/graphic_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=23)
    - [文件系统(5)](http://www.wowotech.net/sort/filesystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=26)
    - [TTY子系统(6)](http://www.wowotech.net/sort/tty_framework) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=27)
  - [u-boot分析(3)](http://www.wowotech.net/sort/u-boot) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=25)
  - [Linux应用技巧(13)](http://www.wowotech.net/sort/linux_application) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=3)
  - [软件开发(6)](http://www.wowotech.net/sort/soft) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=1)
  - [基础技术(13)](http://www.wowotech.net/sort/basic_tech) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=6)
    - [蓝牙(16)](http://www.wowotech.net/sort/bluetooth) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=10)
    - [ARMv8A Arch(15)](http://www.wowotech.net/sort/armv8a_arch) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=19)
    - [显示(3)](http://www.wowotech.net/sort/display) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=22)
    - [USB(1)](http://www.wowotech.net/sort/usb) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=28)
  - [基础学科(10)](http://www.wowotech.net/sort/basic_subject) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=7)
  - [技术漫谈(12)](http://www.wowotech.net/sort/tech_discuss) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=8)
  - [项目专区(0)](http://www.wowotech.net/sort/project) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=9)
    - [X Project(28)](http://www.wowotech.net/sort/x_project) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=24)

- ### 随机文章

  - [Common Clock Framework系统结构](http://www.wowotech.net/pm_subsystem/ccf-arch.html)
  - [浅谈Cache Memory](http://www.wowotech.net/memory_management/458.html)
  - [ACCESS_ONCE宏定义的解释](http://www.wowotech.net/process_management/access-once.html)
  - [perfbook memory barrier（14.2章节）中文翻译（下）](http://www.wowotech.net/kernel_synchronization/perfbook-memory-barrier-2.html)
  - [/proc/meminfo分析（一）](http://www.wowotech.net/memory_management/meminfo_1.html)

- ### 文章存档

  - [2024年2月(1)](http://www.wowotech.net/record/202402)
  - [2023年5月(1)](http://www.wowotech.net/record/202305)
  - [2022年10月(1)](http://www.wowotech.net/record/202210)
  - [2022年8月(1)](http://www.wowotech.net/record/202208)
  - [2022年6月(1)](http://www.wowotech.net/record/202206)
  - [2022年5月(1)](http://www.wowotech.net/record/202205)
  - [2022年4月(2)](http://www.wowotech.net/record/202204)
  - [2022年2月(2)](http://www.wowotech.net/record/202202)
  - [2021年12月(1)](http://www.wowotech.net/record/202112)
  - [2021年11月(5)](http://www.wowotech.net/record/202111)
  - [2021年7月(1)](http://www.wowotech.net/record/202107)
  - [2021年6月(1)](http://www.wowotech.net/record/202106)
  - [2021年5月(3)](http://www.wowotech.net/record/202105)
  - [2020年3月(3)](http://www.wowotech.net/record/202003)
  - [2020年2月(2)](http://www.wowotech.net/record/202002)
  - [2020年1月(3)](http://www.wowotech.net/record/202001)
  - [2019年12月(3)](http://www.wowotech.net/record/201912)
  - [2019年5月(4)](http://www.wowotech.net/record/201905)
  - [2019年3月(1)](http://www.wowotech.net/record/201903)
  - [2019年1月(3)](http://www.wowotech.net/record/201901)
  - [2018年12月(2)](http://www.wowotech.net/record/201812)
  - [2018年11月(1)](http://www.wowotech.net/record/201811)
  - [2018年10月(2)](http://www.wowotech.net/record/201810)
  - [2018年8月(1)](http://www.wowotech.net/record/201808)
  - [2018年6月(1)](http://www.wowotech.net/record/201806)
  - [2018年5月(1)](http://www.wowotech.net/record/201805)
  - [2018年4月(7)](http://www.wowotech.net/record/201804)
  - [2018年2月(4)](http://www.wowotech.net/record/201802)
  - [2018年1月(5)](http://www.wowotech.net/record/201801)
  - [2017年12月(2)](http://www.wowotech.net/record/201712)
  - [2017年11月(2)](http://www.wowotech.net/record/201711)
  - [2017年10月(1)](http://www.wowotech.net/record/201710)
  - [2017年9月(5)](http://www.wowotech.net/record/201709)
  - [2017年8月(4)](http://www.wowotech.net/record/201708)
  - [2017年7月(4)](http://www.wowotech.net/record/201707)
  - [2017年6月(3)](http://www.wowotech.net/record/201706)
  - [2017年5月(3)](http://www.wowotech.net/record/201705)
  - [2017年4月(1)](http://www.wowotech.net/record/201704)
  - [2017年3月(8)](http://www.wowotech.net/record/201703)
  - [2017年2月(6)](http://www.wowotech.net/record/201702)
  - [2017年1月(5)](http://www.wowotech.net/record/201701)
  - [2016年12月(6)](http://www.wowotech.net/record/201612)
  - [2016年11月(11)](http://www.wowotech.net/record/201611)
  - [2016年10月(9)](http://www.wowotech.net/record/201610)
  - [2016年9月(6)](http://www.wowotech.net/record/201609)
  - [2016年8月(9)](http://www.wowotech.net/record/201608)
  - [2016年7月(5)](http://www.wowotech.net/record/201607)
  - [2016年6月(8)](http://www.wowotech.net/record/201606)
  - [2016年5月(8)](http://www.wowotech.net/record/201605)
  - [2016年4月(7)](http://www.wowotech.net/record/201604)
  - [2016年3月(5)](http://www.wowotech.net/record/201603)
  - [2016年2月(5)](http://www.wowotech.net/record/201602)
  - [2016年1月(6)](http://www.wowotech.net/record/201601)
  - [2015年12月(6)](http://www.wowotech.net/record/201512)
  - [2015年11月(9)](http://www.wowotech.net/record/201511)
  - [2015年10月(9)](http://www.wowotech.net/record/201510)
  - [2015年9月(4)](http://www.wowotech.net/record/201509)
  - [2015年8月(3)](http://www.wowotech.net/record/201508)
  - [2015年7月(7)](http://www.wowotech.net/record/201507)
  - [2015年6月(3)](http://www.wowotech.net/record/201506)
  - [2015年5月(6)](http://www.wowotech.net/record/201505)
  - [2015年4月(9)](http://www.wowotech.net/record/201504)
  - [2015年3月(9)](http://www.wowotech.net/record/201503)
  - [2015年2月(6)](http://www.wowotech.net/record/201502)
  - [2015年1月(6)](http://www.wowotech.net/record/201501)
  - [2014年12月(17)](http://www.wowotech.net/record/201412)
  - [2014年11月(8)](http://www.wowotech.net/record/201411)
  - [2014年10月(9)](http://www.wowotech.net/record/201410)
  - [2014年9月(7)](http://www.wowotech.net/record/201409)
  - [2014年8月(12)](http://www.wowotech.net/record/201408)
  - [2014年7月(6)](http://www.wowotech.net/record/201407)
  - [2014年6月(6)](http://www.wowotech.net/record/201406)
  - [2014年5月(9)](http://www.wowotech.net/record/201405)
  - [2014年4月(9)](http://www.wowotech.net/record/201404)
  - [2014年3月(7)](http://www.wowotech.net/record/201403)
  - [2014年2月(3)](http://www.wowotech.net/record/201402)
  - [2014年1月(4)](http://www.wowotech.net/record/201401)

[![订阅Rss](http://www.wowotech.net/content/templates/default/images/rss.gif)](http://www.wowotech.net/rss.php "RSS订阅")

Copyright @ 2013-2015 [蜗窝科技](http://www.wowotech.net/ "wowotech") All rights reserved. Powered by [emlog](http://www.emlog.net/ "采用emlog系统")

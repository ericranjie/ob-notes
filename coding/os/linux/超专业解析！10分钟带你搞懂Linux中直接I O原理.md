# 

架构师社区

 _2021年11月17日 11:33_

The following article is from 云加社区 Author 王睿

[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM4VrfJGRic9cMlydQkzsTsDFptqtib3k4CxI3TOVia4Nmicpw/0)

**云加社区**.

腾讯云官方社区公众号，汇聚技术开发者群体，分享技术干货，打造技术影响力交流社区。

](https://mp.weixin.qq.com/s?__biz=MzU0OTE4MzYzMw==&mid=2247525908&idx=2&sn=5d89a9ddd4f6872f08fadac0af8fb723&chksm=fbb1efeaccc666fcf2064344830a1f5acdbaee653f20ada464e88521db50fa7a3de5f08eccfb&mpshare=1&scene=24&srcid=1117hOfV95AYL0XVRebW2phR&sharer_sharetime=1637132642030&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d03cfa3257f5353547c7d487bfb251d7da97e707ad7730e3e3e620baacb3b3942312069cc29cad538512a36f09473ca359813ffca2b6bdadf4e24566ddecc3f258758fdd61161d820947262245173ac5f9141079de8191ef036812892c45fe5babceab1d71bf9ddfe90f2ac2bf285bf2524e2f22be2891db2b&ascene=14&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=iMac+MacBookAir10%2C1+OSX+OSX+14.6.1+build(23G93)&version=13080710&nettype=WIFI&lang=en&session_us=gh_e985e0758284&countrycode=CN&fontScale=100&exportkey=n_ChQIAhIQZytwGuB1OykXH7dMdG49%2FxKUAgIE97dBBAEAAAAAANb%2BIRqCLjMAAAAOpnltbLcz9gKNyK89dVj0xU7nvov8VeXYNddW%2FDnpTBQoPvjllIpglcaZsSCaIqJ6fmwvmGoBKJ%2FLzeJ6mtpdPDnKfSvxEuUSiuLfFEGqhWjVqmc6R4VgOi%2Fya7a4vD7vgJZHCnoibpGl9CporQ88SF2DpKNjXHcsScMTDjtG5mqBXLEAM0eiczB53PKSAD1%2BP%2FPFQVIHG9A4%2BaGsATrKH37uc%2BDFh%2FAu%2FQfgwMZvhW3bpBVlb%2BQElCJcTeuhK4B6jCYMWucMrkBAiBs8zyKIKSpUXdj99Q%2Fphg5RVQVw%2FjCewTptfT48S7%2F9sQ2to3mK%2FymQGd1l8JQZJY2rfg%3D%3D&acctmode=0&pass_ticket=7fNS2EoV2Z1Tu%2FmIkKiejqQrKG62YiG8G32T33YZciJiC4ytnoVwJdA207ECSilc&wx_header=0#)

导语 | 本文主要以一张图为基础，向大家介绍Linux在I/O上做了哪些事情，即Linux中直接I/O原理，希望本文的经验和思路能为读者提供一些帮助和思考。

  

**引言**

  

我们先看一张图：

  

![Image](https://mmbiz.qpic.cn/mmbiz_png/VY8SELNGe95NTba044oj8aMrjicdTf0Iso4IicwzezOTuicTtJWNTARCPe9VqqaKlDqsbtx30UicRLYy8tpibomBXPA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

  

这张图大体上描述了Linux系统上，应用程序对磁盘上的文件进行读写时，从上到下经历了哪些事情。这篇文章就以这张图为基础，介绍Linux在I/O上做了哪些事情。

  

  

**一、文件系统**

  

### **（一）什么是文件系统**

  

文件系统，本身是对存储设备上的文件，进行组织管理的机制。组织方式不同，就会形成不同的文件系统。比如常见的Ext4、XFS、ZFS以及网络文件系统NFS等等。

  

但是**不同类型的文件系统标准和接口可能各有差异，我们在做应用开发的时候却很少关心系统调用以下的具体实现**，大部分时候都是直接系统调用open, read, write, close来实现应用程序的功能，不会再去关注我们具体用了什么文件系统（UFS、XFS、Ext4、ZFS），磁盘是什么接口（IDE、SCSI，SAS，SATA等），磁盘是什么存储介质（HDD、SSD）应用开发者之所以这么爽，各种复杂细节都不用管直接调接口，是因为内核为我们做了大量的有技术含量的脏活累活。

  

开始的那张图看到Linux在各种不同的文件系统之上，虚拟了一个VFS，目的就是统一各种不同文件系统的标准和接口，让开发者可以使用相同的系统调用来使用不同的文件系统。

  

  

### **（二）文件系统如何工作（VFS）**

####   

- #### **Linux系统下的文件**
    

  

在Linux中一切皆文件。不仅普通的文件和目录，就连块设备、套接字、管道等，也都要通过统一的文件系统来管理。

  

```
用 ls -l 命令看最前面的字符可以看到这个文件是什么类型
```

  

Linux文件系统设计了**两个**数据结构来管理这些不同种类的文件：

  

- inode(index node)：索引节点
    

  

- dentry(directory entry)：目录项
    
      
    

- **inode**
    

  

inode是用来记录文件的metadata，所谓metadata在Wikipedia上的描述是data of data，其实指的就是文件的各种属性，比如inode编号、文件大小、访问权限、修改日期、数据的位置等。

  

```
wrn3552@novadev:~/playground$ stat file
```

  

inode和文件一一对应，它跟文件内容一样，都会被持久化存储到磁盘中。所以，inode同样占用磁盘空间，只不过相对于文件来说它大小固定且大小不算大。

  

- **dentry**
    

  

dentry用来记录文件的名字、inode指针以及与其他dentry的关联关系。

  

```
wrn3552@novadev:~/playground$ tree
```

  

- 文件的名字：像dir、file、hardlink、file_in_dir这些名字是记录在dentry里的。
    

  

- inode指针：就是指向这个文件的inode。
    

  

- 与其他dentry的关联关系：其实就是每个文件的层级关系，哪个文件在哪个文件下面，构成了文件系统的目录结构。
    

  

不同于inode，dentry是由内核维护的一个内存数据结构，所以通常也被叫做dentry cache。

  

  

#### **（三）文****件是如何存储在磁盘上的**

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

这里有张图解释了文件是如何存储在磁盘上的：

  

首先，磁盘再进行文件系统格式化的时候，会分出来**3个区**：Superblock、inode blocks、data blocks。（其实还有boot block，可能会包含一些bootstrap代码，在机器启动的时候被读到，这里忽略）

  

其中inode blocks放的都是每个文件的inode，data blocks里放的是每个文件的内容数据。

  

这里关注一下**Superblock，它包含了整个文件系统的metadata**，具体有：

  

- inode/data block 总量、使用量、剩余量。
    

  

- 文件系统的格式，属主等等各种属性。
    

  

Superblock对于文件系统来说非常重要，如果Superblock损坏了，文件系统就挂载不了了，相应的文件也没办法读写。

  

既然Superblock这么重要，那肯定不能只有一份，坏了就没了，它在系统中是有很多副本的，在Superblock损坏的时候，可以使用fsck（File System Check and repair）来恢复。

  

回到上面的那张图，可以很清晰地看到文件的各种属性和文件的数据是如何存储在磁盘上的：

  

- dentry里包含了文件的名字、目录结构、inode指针。
    

  

- inode指针指向文件特定的inode（存在inode blocks里）
    

  

- 每个inode又指向data blocks里具体的logical block，这里的logical block存的就是文件具体的数据。
    

  

这里解释一下什么是logical block：

  

- 对于不同存储介质的磁盘，都有最小的读写单元 /sys/block/sda/queue/physical_block_size。
    

  

- HDD叫做sector（扇区），SSD叫做page（页面）
    

  

- 对于hdd来说，每个sector大小512Bytes。
    

  

- 对于SSD来说每个page大小不等（和cell类型有关），经典的大小是4KB。
    

  

- 但是Linux觉得按照存储介质的最小读写单元来进行读写可能会有效率问题，所以支持在文件系统格式化的时候指定block size的大小，一般是把几个physical_block拼起来就成了一个logical block /sys/block/sda/queue/logical_block_size。
    

  

- 理论上应该是logical_block_size>=physical_block_size，但是有时候我们会看到physical_block_size=4K，logical_block_size=512B情况，其实这是因为磁盘上做了一层512B的仿真（emulation）（详情可参考512e和4Kn）
    
      
    
      
    

**二、ZFS**

  

这里简单介绍一个广泛应用的文件系统ZFS，一些数据库应用也会用到 ZFS。

  

先看一张ZFS的层级结构图：

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

这是一张从底向上的图：

  

- 将若干物理设备disk组成一个虚拟设备vdev（同时，disk 也是一种vdev）
    

  

- 再将若干个虚拟设备vdev加到一个zpool里。
    

  

- 在zpool的基础上创建zfs并挂载（zvol可以先不看，我们没有用到）
    
      
    

#### **（一）ZFS的一些操作**

  

- **创建zpool**
    

  

```
root@:~ # zpool create tank raidz /dev/ada1 /dev/ada2 /dev/ada3 raidz /dev/ada4 /dev/ada5 /dev/ada6
```

  

- 创建了一个名为tank的zpool
    

  

- 这里的raidz同RAID5
    

  

除了raidz还支持其他方案：

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

- **创建ZFS**
    

  

```
root@:~ # zfs create -o mountpoint=/mnt/srev tank/srev
```

  

- 创建了一个ZFS，挂载到了/mnt/srev。
    

  

- 这里没有指定ZFS的quota，创建的ZFS大小即zpool大小。
    

  

- **对ZFS设置quota**
    

  

```
root@:~ # zfs set quota=1G tank/srev
```

  

  

#### **（二）ZFS特性**

  

- ##### **Pool存储**
    

  

上面的层级图和操作步骤可以看到ZFS是基于zpool创建的，zpool可以动态扩容意味着存储空间也可以动态扩容。而且可以创建多个文件系统，文件系统共享完整的zpool空间无需预分配。

  

- ##### **事务文件系统**
    

  

#### ZFS的写操作是事务的，意味着要么就没写，要么就写成功了，不会像其他文件系统那样，应用打开了文件，写入还没保存的时候断电，导致文件为空。

  

#### ZFS保证写操作事务采用的是copy on write的方式：

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

当block B有修改变成B1的时候，普通的文件系统会直接在block B原地进行修改变成B1。

  

ZFS则会再另一个地方写B1，然后再在后面安全的时候对原来的B进行回收。这样结果就不会出现B被打开而写失败的情况，大不了就是B1没写成功。

  

这个特性让ZFS在断电后不需要执行fsck来检查磁盘中是否存在写操作失败需要恢复的情况，大大提升了应用的可用性。

  

- ##### **ARC缓存**
    

  

ZFS中的ARC(Adjustable Replacement Cache) 读缓存淘汰算法，是基于IBM的ARP(Adaptive Replacement Cache) 演化而来。

  

在一些文件系统中实现的标准LRU算法其实是有缺陷的：比如复制大文件之类的线性大量I/O操作，导致缓存失效率猛增（大量文件只读一次，放到内存不会被再读，坐等淘汰）

  

另外，缓存可以根据时间来进行优化（LRU，最近最多使用），也可以根据频率进行优化（LFU，最近最常使用），这两种方法各有优劣，但是没办法适应所有场景。

  

**ARC的设计就是尝试在LRU和LFU之间找到一个平衡，根据当前的I/O workload来调整用LRU多一点还是LFU多一点**。

  

ARC定义了**4**个链表：

  

- LRU list：最近最多使用的页面，存具体数据。
    

  

- LFU list：最近最常使用的页面，存具体数据。
    

  

- Ghost list for LRU：最近从LRU表淘汰下来的页面信息，不存具体数据，只存页面信息。
    

  

- Ghost list for LFU：最近从LFU表淘汰下来的页面信息，不存具体数据，只存页面信息。
    

  

**ARC工作流程**大致如下：

  

- LRU list和LFU list填充和淘汰过程和标准算法一样。
    

  

- 当一个页面从LRU list淘汰下来时，这个页面的信息会放到LRU ghost表中。
    

  

- 如果这个页面一直没被再次引用到，那么这个页面的信息最终也会在LRU ghost表中被淘汰掉。
    

  

- 如果这个页面在LRU ghost表中未被淘汰的时候，被再一次访问了，这时候会引起一次幽灵（phantom）命中。
    

  

- phantom命中的时候，事实上还是要把数据从磁盘第一次放缓存。
    

  

- 但是这时候系统知道刚刚被LRU表淘汰的页面又被访问到了，说明LRU list太小了，这时它会把LRU list长度加一，LFU长度减一。
    

  

- 对于LFU的过程也与上述过程类似。
    

  

  

**三、磁盘类型**

  

磁盘根据不同的分类方式，有各种不一样的类型。

###   

### **（一）磁盘的存储介质**

  

根据磁盘的存储介质可以分两类（大家都很熟悉）：HDD（机械硬盘）和SSD（固态硬盘）

  

  

### **（二）磁盘的接口**

  

根据磁盘接口分类：

  

- IDE (Integrated Drive Electronics)
    

  

- SCSI (Small Computer System Interface)
    

  

- SAS (Serial Attached SCSI)
    

  

- SATA (Serial ATA)
    
      
    
- ...
    

  

不同的接口，往往分配不同的设备名称。比如，IDE设备会分配一个hd前缀的设备名，SCSI和SATA设备会分配一个sd前缀的设备名。如果是多块同类型的磁盘，就会按照a、b、c等的字母顺序来编号。

  

  

### **（三）Linux对磁盘的管理**

  

其实在Linux中，磁盘实际上是作为一个块设备来管理的，也就是以块为单位读写数据，并且支持随机读写。每个块设备都会被赋予两个设备号，分别是主、次设备号。主设备号用在驱动程序中，用来区分设备类型；而次设备号则是用来给多个同类设备编号。

  

```
g18-"299" on ~# ls -l /dev/sda*
```

  

- 这些sda磁盘主设备号都是8，表示它是一个sd类型的块设备。
    

  

- 次设备号0-10表示这些不同sd块设备的编号。
    
      
    
      
    

**四、Generic Block Layer**

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

可以看到中间的Block Layer其实就是Generic Block Layer。

  

在图中可以看到Block Layer的I/O调度分为两类，分别表示**单队列和多队列**的调度：

  

- I/O scheduler
    

  

- blkmq
    

  

  

### **（一）I/O调度**

  

老版本的内核里只支持单队列的I/O scheduler，在3.16版本的内核开始支持多队列blkmq。

  

这里介绍几种经典的I/O调度策略：

  

- **单队列I/O scheduler**
    

  

- NOOP：事实上是个FIFO的队列，只做基本的请求合并。
    

  

- CFQ：Completely Fair Queueing，完全公平调度器，给每个进程维护一个I/O调度队列，按照时间片来均匀分布每个进程I/O请求。
    
      
    
- DeadLine：为读和写请求创建不同的I/O队列，确保达到deadline的请求被优先处理。
    

  

  

- **多队列blkmq**
    

  

- bfq：Budget Fair Queueing，也是公平调度器，不过不是按时间片来分配，而是按请求的扇区数量（带宽）
    

  

- kyber：维护两个队列（同步/读、异步/写），同时严格限制发到这两个队列的请求数以保证相应时间。
    

  

- mq-deadline：多队列版本的deadline。
    

  

具体各种I/O调度策略可以参考IOSchedulers

（https://wiki.ubuntu.com/Kernel/Reference/IOSchedulers）

  

关于blkmq可以参考Linux Multi-Queue Block IO Queueing Mechanism (blk-mq) Details_Details)

  

多队列调度可以参考Block layer introduction part 2:the request layer

  

  

**五、性能指标**

  

一般来说I/O性能指标有这**5**个：

  

- 使用率：ioutil，指的是磁盘处理I/O的时间百分比，ioutil只看有没有I/O请求，不看I/O请求的大小。ioutil越高表示一直都有I/O请求，不代表磁盘无法响应新的I/O请求。
    

  

- IOPS：每秒的I/O请求数。
    

  

- 吞吐量/带宽：每秒的I/O请求大小，通常是MB/s或者GB/s为单位。
    

  

- 响应时间：I/O请求发出到收到响应的时间。
    

  

- 饱和度：指的是磁盘处理I/O的繁忙程度。这个指标比较玄学，没有直接的数据可以表示，一般是根据平均队列请求长度或者响应时间跟基准测试的结果进行对比来估算（在做基准测试时，还会分顺序/随机、读/写进行排列组合分别去测IOPS和带宽）
    

  

上面的指标除了饱和度外，其他都可以在监控系统中看到。**Linux也提供了一些命令来输出不同维度的I/O状态**：

  

- iostat-d-x：看各个设备的I/O状态，数据来源/proc/diskstats。
    

  

- pidstat-d：看近处的I/O。
    

  

- iotop：类似top，按I/O大小对进程排序。
    

  

  

 **作者简介**

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**王睿**

腾讯云游戏解决方案架构师

腾讯云游戏解决方案架构师，毕业于中山大学。目前负责腾讯云游戏行业解决方案设计等工作，有丰富的游戏运维及开发经验。

  

![](http://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk55KKLFaGCDRURMvFtPXf9fZXJOHOFsA3Ye8Qbibf3qHLkBQNpdjicAVpPf2T03EcakjAFbwqicjXSibXA/300?wx_fmt=png&wxfrom=19)

**架构师社区**

架构师社区，专注分享架构师技术干货，架构师行业秘闻，汇集各类奇妙好玩的架构师话题和流行的架构师动向！

641篇原创内容

公众号

Reads 1235

​

Comment

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk55KKLFaGCDRURMvFtPXf9fZXJOHOFsA3Ye8Qbibf3qHLkBQNpdjicAVpPf2T03EcakjAFbwqicjXSibXA/300?wx_fmt=png&wxfrom=18)

架构师社区

3Share3

Comment

Comment

**Comment**

暂无留言
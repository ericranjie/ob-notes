# [蜗窝科技](http://www.wowotech.net/)

### 慢下来，享受技术。

[![](http://www.wowotech.net/content/uploadfile/201401/top-1389777175.jpg)](http://www.wowotech.net/)

- [博客](http://www.wowotech.net/)
- [项目](http://www.wowotech.net/sort/project)
- [关于蜗窝](http://www.wowotech.net/about.html)
- [联系我们](http://www.wowotech.net/contact_us.html)
- [支持与合作](http://www.wowotech.net/support_us.html)
- [登录](http://www.wowotech.net/admin)

﻿

## 

作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2015-5-15 11:21 分类：[软件开发](http://www.wowotech.net/sort/soft)

底层开发过程中，经常需要在终端查看或者修改设备寄存器的值，busybox有一个工具----devmem，可用于读取或者修改物理寄存器的值，非常方便：

> # busybox devmem                                             
> 
> BusyBox v1.14.3 (2014-11-05 16:48:48 CST) multi-call binary
> 
>   
> 
> Usage: devmem ADDRESS [WIDTH [VALUE]]
> 
>   
> 
> Read/write from physical address
> 
>   
> 
>         ADDRESS Address to act upon
> 
>         WIDTH   Width (8/16/...)
> 
>         VALUE   Data to be written

但它有一个不足的地方：不能连续操作物理内存（虽然这很危险，但在显示相关的调试中，如果能向指定的物理内存加载一个图片，或者dump指定物理内存的内容，还是很方便的），因此我重新写了一些代码，实现如下功能：  

1. 利用/dev/mem，编写一个library，将物理地址访问的接口抽象出来。

2. 基于该library，实现devmd工具，可以显示指定的某个或者某些物理地址的内容。  

3. 基于该library，实现devmcpy工具，可以将指定地址的内容，copy到另一个地址。类似memcpy。

4. 基于该library，实现devmset工具，可以将指定地址设置为某一个value。类似memset。

5. 基于该library，实现devmload和devmsave工具，将指定文件的内容加载到指定物理地址，以及从指定物理地址dump数据到指定的文件。

实现的过程比较简单，包括基本的文件open、read、write、mmap等操作，这里就不详细分析代码了，贴一下README文件，具体可参考github中的source code：[https://github.com/wowotech/wowolib/tree/master/devmem。](https://github.com/wowotech/wowolib/tree/master/devmem)

> Purpose:
> 
> A simple library to operate physical address in usersapce,
> 
> which is based on linux "/dev/mem".
> 
> Also, some useful tools is provided, including:
> 
> --devmd, display PA's value.
> 
> --devmcpy, copy from src PA to dst PA.
> 
> --devmset, set PA to special value.
> 
> --devmload, load file to PA.
> 
> --devmsave, save PA to file.
> 
>   
> 
> Uasge:
> 
> --devmem library
> 
> 	Include devmem.h, and put devmem.c together with your souece code,
> 
> 	call the APIs you need, and build them.
> 
>   
> 
> --devmd 
> 
> 	./devmd { address } [type] [count]
> 
> 	address: Address to act upon
> 
> 	width:	Width (8/16/...), default is 32
> 
> 	count:	Data count to be read, default is 1
> 
>   
> 
> --devmcpy 
> 
> 	./devmcpy {src_addr} {dst_addr} {count} [width]
> 
> 	src_addr:	source address
> 
> 	dst_addr:	destination address
> 
> 	count:		copy count, default is 1
> 
> 	width:		width, 8/16/32/64..., default is 32
> 
>   
> 
> --devmset 
> 
> 	./devmset {address} {value} {count} [width]
> 
> 	address:	Address to act upon
> 
> 	value:		Value to write
> 
> 	count:		Set counts
> 
> 	width:		width, 8/16/32/64..., default is 32
> 
>   
> 
> --devmload 
> 
> 	./devmload {addr} {filename}
> 
> 	addr:		destination physical address
> 
> 	filename:	file name will load from
> 
>   
> 
> --devmsave 
> 
> 	./devmsave {addr} {len} {filename}
> 
> 	addr:	physical address need to save
> 
> 	len:	length in byte
> 
> 	filename: file save to
> 
>   
> 
> Author:
> 
> wowo<[www.wowotech.net](http://www.wowotech.net/)>
> 
>   
> 
> Date:
> 
> 2015-05-15

标签: [devmem](http://www.wowotech.net/tag/devmem)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Linux时间子系统之（十四）：tick broadcast framework](http://www.wowotech.net/timer_subsystem/tick-broadcast-framework.html) | [关于内核中的乘法和除法。](http://www.wowotech.net/185.html)»

**评论：**

**[descent](http://www.wowotech.net/)**  
2015-12-04 10:02

請問 open("/dev/mem", O_RDWR | O_SYNC)) 需要使用 O_SYNC, 是為了不要有 cache 的干擾嗎？

[回复](http://www.wowotech.net/soft/186.html#comment-3192)

**[linuxer](http://www.wowotech.net/)**  
2015-12-04 19:35

@descent：首先声明：我的回答是基于linux 4.1.10下的ARM64代码的，其他的情况我没有去看代码，估计类似。  
  
对"/dev/mem"这个设备的使用一般包括两步：  
（1）open并获取一个fd  
（2）通过mmap将该fd mapping到该进程的一段虚拟地址空间上去，当程序实际访问这段虚拟地址空间的时候，就象访问了fd指向的那个对象。对于"/dev/mem"的fd而言，其指向了系统中的物理地址空间  
  
如何建立进程虚拟地址空间（vma）和"/dev/mem"设备fd之间的mapping是通过其驱动file operations数据结构中的mmap成员函数来完成的，在该函数中会建立页表，从而控制cache write buffer等的设定。该函数的代码如下：  
static int mmap_mem(struct file *file, struct vm_area_struct *vma)  
{  
    size_t size = vma->vm_end - vma->vm_start;  
  
    if (!valid_mmap_phys_addr_range(vma->vm_pgoff, size))  
        return -EINVAL;  
  
    if (!private_mapping_ok(vma))  
        return -ENOSYS;  
  
    if (!range_is_allowed(vma->vm_pgoff, size))  
        return -EPERM;  
  
    if (!phys_mem_access_prot_allowed(file, vma->vm_pgoff, size,  
                        &vma->vm_page_prot))  
        return -EINVAL;  
  
    vma->vm_page_prot = phys_mem_access_prot(file, vma->vm_pgoff,  
                         size,  
                         vma->vm_page_prot);  
  
    vma->vm_ops = &mmap_mem_ops;  
  
    /* Remap-pfn-range will mark the range VM_IO */  
    if (remap_pfn_range(vma,  
                vma->vm_start,  
                vma->vm_pgoff,  
                size,  
                vma->vm_page_prot)) {  
        return -EAGAIN;  
    }  
    return 0;  
}  
  
其中，phys_mem_access_prot函数定义了页表中的memory attribute以及memory type的信息，代码如下：  
pgprot_t phys_mem_access_prot(struct file *file, unsigned long pfn,  
                  unsigned long size, pgprot_t vma_prot)  
{  
    if (!pfn_valid(pfn))  
        return pgprot_noncached(vma_prot);  
    else if (file->f_flags & O_SYNC)  
        return pgprot_writecombine(vma_prot);  
    return vma_prot;  
}  
  
对于ARM64的处理器而言，其物理地址空间有两种：  
（1）memory  
（2）device  
从上面的代码可以看到，实际上O_SYNC这个flag只会在你要mapping的物理地址是memroy的时候才是有效的，如果你要mapping的物理地址本身就是IO（这也是/dev/mem的大多数的应用场合），那么页表项中的内容是pgprot_noncached，即：  
#define pgprot_noncached(prot) \  
    __pgprot_modify(prot, PTE_ATTRINDX_MASK, PTE_ATTRINDX(MT_DEVICE_nGnRnE) | PTE_PXN | PTE_UXN)  
这时候，无论是否有O_SYNC，页表的属性都是device类型的，而对于memory type是device类型的地址影响，uncache是必须的。

[回复](http://www.wowotech.net/soft/186.html#comment-3193)

**anchen**  
2016-09-22 16:09

@linuxer：您好！我想请教一个和O_SYNC有关的具体问题。  
我的系统是Altera的SoC，我要在用户空间为DMAC（pl330）更新其微码，结果发现DMA第一次运行正常，第二次运行依然执行前一次操作（虽然微码中的源和目的我都更新过了）。现在的结论是，因为这个SoC的架构，DMAC总是通过ACP访问L2 cache，意外着，DMA第一次会将微码锁存在L2 cache中，而用户空间在更新微码的过程中，没有处理L2 cache和memory的一致性，造成DMA第二次访问时，L2 cache依然命中，所以再次运行第一次的操作。我考虑过如下O_SYNC的使用，但也没有成功，不知道这样使用对不对？  
  
FdforOcm = open("/dev/mem", O_RDWR | O_SYNC);  
OcmAddr = mmap(NULL, 0x40000,  PROT_READ|PROT_WRITE, MAP_SHARED, FdforOcm, 0xFFE00000);  
//这个返回的地址用来给程序更新DMA微码，使用O_SYNC，意味着一定会写入cache，然后更新memory吗？  
FdforDMA = open("/dev/mem", O_RDWR);  
DMAAddr = mmap(NULL, 0x1000,  PROT_READ|PROT_WRITE, MAP_SHARED, FdforDMA, 0xFFDA0000);    
//这个返回的地址用来给程序访问DMA的寄存器

[回复](http://www.wowotech.net/soft/186.html#comment-4597)

**anchen**  
2016-09-22 16:10

@linuxer：您好！我想请教一个和O_SYNC有关的具体问题。  
我的系统是Altera的SoC，我要在用户空间为DMAC（pl330）更新其微码，结果发现DMA第一次运行正常，第二次运行依然执行前一次操作（虽然微码中的源和目的我都更新过了）。现在的结论是，因为这个SoC的架构，DMAC总是通过ACP访问L2 cache，意外着，DMA第一次会将微码锁存在L2 cache中，而用户空间在更新微码的过程中，没有处理L2 cache和memory的一致性，造成DMA第二次访问时，L2 cache依然命中，所以再次运行第一次的操作。我考虑过如下O_SYNC的使用，但也没有成功，不知道这样使用对不对？  
  
FdforOcm = open("/dev/mem", O_RDWR | O_SYNC);  
OcmAddr = mmap(NULL, 0x40000,  PROT_READ|PROT_WRITE, MAP_SHARED, FdforOcm, 0xFFE00000);  
//这个返回的地址用来给程序更新DMA微码，使用O_SYNC，意味着一定会写入cache，然后更新memory吗？  
FdforDMA = open("/dev/mem", O_RDWR);  
DMAAddr = mmap(NULL, 0x1000,  PROT_READ|PROT_WRITE, MAP_SHARED, FdforDMA, 0xFFDA0000);    
//这个返回的地址用来给程序访问DMA的寄存器

[回复](http://www.wowotech.net/soft/186.html#comment-4598)

**[linuxer](http://www.wowotech.net/)**  
2016-09-24 23:41

@anchen：如果在open /dev/mem这个device node的时候有O_SYNC，那么，随后的mmap创建的映射将是non cache的（对于ARM64的平台，其他的平台我没有看，我估计是类似的），也就是说，你通过OcmAddr地址更新DMA微码的时候，应该不会通过cache，而直接进入最终的设备地址。

[回复](http://www.wowotech.net/soft/186.html#comment-4603)

**anchen**  
2016-09-26 14:51

@linuxer：感谢您的回复！看来我理解错了，但其实我两个都试了，无论是加还是不加 O_SYNC，结果似乎都是直接写入最终的设备地址，而没有同时更新到cache。有没有什么办法可以满足我的要求，写入最终设备的同时，更新到cache？或者说，如何在更新物理设备的同时，能够在用户空间，失效相对应的那部分cache？对现在的情况来说，我只要求物理memory和L2 cache能保持一致就可以了。

[回复](http://www.wowotech.net/soft/186.html#comment-4607)

**[wowo](http://www.wowotech.net/)**  
2016-09-26 18:47

@anchen：我觉得你的需求应该就是invalidate l2 cache？  
以arm平台为例，其实就是一条写p15协处理器的指令（当然，需要一些内存屏障操作）。  
arm平台为这个需求抽象出来了一系列的接口，头文件在：arch/arm/include/asm/outercache.h  
可以参考kernel中某些CPU的实现，例如：arch/arm/mm/cache-feroceon-l2.c  
当然，如果粗暴一些，你也可以自己写一个，开放给用户空间程序使用。  
不知道还有没有其它好的方法？知道的同学也吱一声~~~

[回复](http://www.wowotech.net/soft/186.html#comment-4608)

**[linuxer](http://www.wowotech.net/)**  
2016-09-26 22:24

@anchen：你遇到的这个问题其实就是cache coherence的问题。根据你的描述，DMAC总是通过ACP访问L2 cache，而这就导致了你无论使用O_SYNC或者不使用O_SYNC都是错误的：  
1、使用了O_SYNC，那么你的驱动在访问mmap返回的那些地址的时候是non cache的，也就是说，数据直接去到main memory，而不会操作L2 cache。对于DMAC，它的访问需要通过L2 cache，由于第一次访问的时候，L2是all invalidate的，因此数据访问都是cache miss的，需要从main memory加载数据并更新到L2 cache。后续，程序再次更新通过mmap返回的那些地址进行数据更新的时候，直达main memory，但是DMAC在访问的时候，L2 cache hit，从而不会访问main memory，从而导致运行失败。  
2、不使用O_SYNC的时候，你的驱动在访问mmap返回的那些地址的时候是cache的，但是首先进入的是L1 cache（我猜测cache 策略是write back的），而不会操作L2 cache，因此，DMAC也不能从L2中获取正确的数据。  
  
怎么破呢？我觉得首先是应该mapping成cache的，但是要辅以一些同步的手段和cache维护的指令。也就是说，当你的程序通过mmap返回的那些地址完成数据更新之后，要clean到POC，让系统中的所有的observer（CPU core和DMAC等等）看到一致性的数据。  
  
BTW， invalidate操作不适合，我们这个场合需要的是将L1的数据write back到L2，invalidate是将cache中的数据设置为无效。另外，cache操作其实无法操作到指定的某个level的cache，只能是poc或者是pou。不过其实你没有详细描述你系统的memory hierarchy，因此上面我建议使用poc。

[回复](http://www.wowotech.net/soft/186.html#comment-4609)

**[wowo](http://www.wowotech.net/)**  
2016-09-26 22:45

@linuxer：为什么invalidate无效呢？  
应用程序已经将微码更新到指定的memory中了，DMAC期望访问这个新的数据。  
如果cache有效，则直接从cache中取出了旧的数据。  
如果cache无效，DMAC将不得不从memory中从新读取数据（新的）。

**[linuxer](http://www.wowotech.net/)**  
2016-09-26 23:10

@linuxer：wowo，我是这么想的：  
1、程序的数据访问路径是CPU--->L1 cache--->L2 cache--->main meory  
2、DMAC的数据访问路径是DMAC--->L2 cache--->main meory  
  
程序写入的时候，首先进入L1 cache，硬件决定什么时候从L1进入L2以及main memory（除非程序手动去clean cache），因此DMAC无法从L2中得到正确的数据。当然，上面都是针对write back 类型的cache说的。

**anchen**  
2016-09-27 10:31

@linuxer：真的很幸运！在这个平台上，获得这么清晰的建议（实在是楼主太强大了！）。  
系统架构确实如你所述，CPU--->L1 cache--->L2 cache--->main memory。  
我的经验不足，没能想到L1和L2的关系。  
和你想法一样，我不打算invalidate L2 cache，这个不是最佳选择，也会引起效率的降低。关于poc或pou，我网上科普了下，能理解其概念。但具体怎么做？  
如何在用户空间用代码去实现？我的应用都需要在用户空间去完成。  
再次感谢，感谢楼主分享宝贵经验和知识！

**anchen**  
2016-09-27 10:37

@linuxer：我的平台是dual core （Cortex A9）

**anchen**  
2016-09-27 12:46

@linuxer：poc还是有点难懂。  
https://community.arm.com/thread/8571  
这里有些讨论，但是很难消化。  
我其实有个疑问，L1 cache既然已经写入，那么它一定应该在某个时刻去同步L2和main memory，对吗？那这是一个什么样的时刻，或者有什么操作可以主动发起这个同步吗？

**[linuxer](http://www.wowotech.net/)**  
2016-09-27 15:16

@linuxer：L1 cache既然已经写入，那么它一定应该在某个时刻去同步L2和main memory，对吗？那这是一个什么样的时刻，或者有什么操作可以主动发起这个同步吗？  
---------------------------  
基本上那个时刻是HW决定的。当L1 cache已经全部是valid的entry，当再次数据访问发生cache miss的时候，就需要从这些valid的entry中找出一个最近最少使用的那个entry把它踢出L1（具体算法是由硬件cache设计人员决定的），如果准备踢出的L1 entry是dirty的，那么就需要write back到L2 cache中。  
ARM处理器提供了cache维护指令来进行clean cache的操作，具体可以参考ARM ARM

**[linuxer](http://www.wowotech.net/)**  
2015-12-05 00:01

@descent：上面的回答稍显敷衍，主要是快下班了，着急着回家，现在老婆孩子睡了，可以安安静静的针对这个问题胡思乱考了，呵呵~~~  
/dev/mem这个设备节点可以让用户空间建立页表并mapping到任意的系统中的物理地址上去，如果这个设备节点提供O_SYNC这种接口来控制映射的cache属性，那么这个接口是否是一个好的接口呢？  
  
对于普通的memory而言，可以映射为non cached（例如DMA buffer，framebuffer等），也可以映射为cached（大部分的普通的data和heap上的memory都是cached），因此，通过O_SYNC来控制映射的cache属性是OK的，但是，对于IO memory而言，设置为cached真的适合吗？这可能会引入很多不必要的麻烦，因此，这种情况下，通过O_SYNC来控制映射的cache属性是徒增user的烦恼。而且，一旦提供了这样的接口，实际上用户空间的进程可以把多个虚拟地址映射到一个物理地址上去（建立多个页表，把不同的虚拟地址映射到同一个物理地址），并且指定不同的属性。软件这么任性，有没有和硬件商量呢？  
  
这个问题转换成了cpu设计的问题了，即硬件如何处理同一个物理地址不同属性的访问，对于ARM64处理器而言，一个物理地址有memory type的设定（体现在页表中的描述符中的某些bit），要么是device，要么是memory。如果virtual_A地址被mapping到physical_A并通过页表设定为device，同时virtual_B地址也被mapping到physical_A并通过页表设定为memory，那么估计ARM处理的内心是崩溃的。ARM64处理器不愿意浪费多余的晶体管来处理这些破事，因此规定软件符合一定的规则，如果打破这个规则，那么cpu的执行逻辑是未知的。不同的cpu有不同的设计理念，因此phys_mem_access_prot是architecture specific的。ARM64的phys_mem_access_prot函数会根据硬件的要求，对页表进行的设定，确保符合处理器的要求。

[回复](http://www.wowotech.net/soft/186.html#comment-3194)

**[descent](http://www.wowotech.net/)**  
2015-12-16 14:35

@linuxer：謝謝你的回覆, 真的要看第0手資料才能知道這些事情, 我那時候 google 了老半天只得到個不太清楚的回答, 所以才會問問看, 沒想過你還真的為了回答我的問題, 親身去看程式碼, 真的專業。

[回复](http://www.wowotech.net/soft/186.html#comment-3251)

**[RobinHsiang](http://www.wowotech.net/)**  
2015-05-19 09:57

#define ASSERT()                                        \  
do {                                                    \  
    PRINT("ASSERT: %s %s %d",                           \  
           __FILE__, __FUNCTION__, __LINE__);           \  
    while (1);                                          \  
} while (0)  
  
有时候不能用goto可以这样使用do{} while(0)，但内核中定义宏很多时候也用do{} while(0)，这样主要是为了编译器对宏的展开么？

[回复](http://www.wowotech.net/soft/186.html#comment-1833)

**[RobinHsiang](http://www.wowotech.net/)**  
2015-05-19 10:01

@RobinHsiang：呵呵，刚刚搜索到答案了，以前没多想这问题。  
Wowo真是深受linux内核的熏陶啊~~~

[回复](http://www.wowotech.net/soft/186.html#comment-1834)

**[wowo](http://www.wowotech.net/)**  
2015-05-19 10:46

@RobinHsiang：主要是让宏定义规范一些，我也没想那么多，看别人用自己就用了。呵呵~~

[回复](http://www.wowotech.net/soft/186.html#comment-1835)

**[printk](http://www.wowotech.net/)**  
2015-05-18 15:12

很好很强大！方便我调试bug了～

[回复](http://www.wowotech.net/soft/186.html#comment-1831)

**发表评论：**

 昵称

 邮件地址 (选填)

 个人主页 (选填)

![](http://www.wowotech.net/include/lib/checkcode.php) 

- ### 站内搜索
    
       
     蜗窝站内  互联网
    
- ### 功能
    
    [留言板  
    ](http://www.wowotech.net/message_board.html)[评论列表  
    ](http://www.wowotech.net/?plugin=commentlist)[支持者列表  
    ](http://www.wowotech.net/support_list)
- ### 最新评论
    
    - Shiina  
        [一个电路（circuit）中，由于是回路，所以用电势差的概念...](http://www.wowotech.net/basic_subject/voltage.html#8926)
    - Shiina  
        [其中比较关键的点是相对位置概念和点电荷的静电势能计算。](http://www.wowotech.net/basic_subject/voltage.html#8925)
    - leelockhey  
        [你这是哪个内核版本](http://www.wowotech.net/pm_subsystem/generic_pm_architecture.html#8924)
    - ja  
        [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
    - 元神高手  
        [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
    - 十七  
        [内核空间的映射在系统启动时就已经设定好，并且在所有进程的页表...](http://www.wowotech.net/process_management/context-switch-arch.html#8920)
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
    
    - [u-boot启动流程分析(1)_平台相关部分](http://www.wowotech.net/u-boot/boot_flow_1.html)
    - [Linux进程冻结技术](http://www.wowotech.net/pm_subsystem/237.html)
    - [内存初始化代码分析（一）：identity mapping和kernel image mapping](http://www.wowotech.net/memory_management/__create_page_tables_code_analysis.html)
    - [MinGW下安装man工具包](http://www.wowotech.net/linux_application/8.html)
    - [Debian8 内核升级实验](http://www.wowotech.net/linux_application/debian8-upgrade-kernel.html)
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
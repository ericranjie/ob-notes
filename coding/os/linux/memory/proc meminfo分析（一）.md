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

作者：[linuxer](http://www.wowotech.net/author/3 "linuxer") 发布于：2017-8-17 19:27 分类：[内存管理](http://www.wowotech.net/sort/memory_management)

本文主要分析/proc/meminfo文件的各种输出信息的具体含义。

一、MemTotal

MemTotal对应当前系统中可以使用的物理内存。

这个域实际是对应内核中的totalram_pages这个全局变量的，定义如下：

> unsigned long totalram_pages __read_mostly;

该变量表示当前系统中Linux内核可以管理的所有的page frame的数量。注意：这个值并不是系统配置的内存总数，而是指操作系统可以管理的内存总数。

内核是如何得到totalram_pages这个值的呢？是在初始化的过程中得到的，具体如下：我们知道，在内核初始化的时候，我们可以通过memblock模块来管理内存布局，从而得到了memory type和reserved type的内存列表信息。在初始化阶段如果有内存分配的需求，那么也可以通过memblock来完成，直到伙伴系统初始化完成，而在这个过程中，memory type的那些page frame被一个个的注入到各个zone的free list中，同时用totalram_pages 来记录那个时间点中空闲的page frame数量。这个点也就是伙伴系统的一个初始内存数量的起点。

举一个实际的例子：我的T450的计算机，配置的内存是8G，也就是8388608KB。但是MemTotal没有那么多，毕竟OS本身（正文段、数据段等等）就会占用不少内存，此外还有系统各种保留的内存，把这些都去掉，Linux能管理的Total memory是7873756KB。看来OS内存管理的开销也不小啊。

二、MemFree

启动的时候，系统确定了MemTotal的数目，但是随着系统启动过程，内核会动态申请内存，此外，用户空间也会不断创建进程，也会不断的消耗内存，因此MemTotal可以简单分成两个部分：正在使用的和空闲的。MemFree表示的就是当前空闲的内存数目，这些空闲的page应该是挂在各个node的各个zone的buddy系统中。

具体free memory的计算如下：

> freeram = global_page_state(NR_FREE_PAGES);

也就是把整个系统中在当前时间点上空闲的内存数目统计出来。

三、MemAvailable

所谓memory available，其实就是不引起swapping操作的情况下，我们能使用多少的内存。即便是free的，也不是每一个page都可以用于内存分配。例如buddy system会设定一个水位，一旦free memory低于这个水位，系统就会启动page reclaim，从而可能引起swapping操作。因此，我们需要从MemFree这个数值中去掉各个节点、各个zone的预留的内存（WMARK_LOW）数目。当然，也是不是说那些不是空闲的页面我们就不能使用，例如page cache，虽然不是空闲的，但是它也不过是为了加快性能而使用，其实也可以回收使用。当然，也不是所有的page cache都可以计算进入MemAvailable，有些page cache在回收的时候会引起swapping，这些page cache就不能算数了。此外，reclaimable slab中也有一些可以使用的内存，MemAvailable也会考虑这部分内存的情况。

总而言之，MemAvailable就是不需要额外磁盘操作（开销较大）就可以使用的空闲内存的数量。

四、Buffers

其实新的内核已经没有buffer cache了，一切都统一到了page cache的框架下了。因此，所谓的buffer cache就是块设备的page cache。具体的统计过程是通过nr_blockdev_pages函数完成，代码如下：

> long nr_blockdev_pages(void)  
> {  
>     struct block_device *bdev;  
>     long ret = 0;  
>     spin_lock(&bdev_lock);  
>     list_for_each_entry(bdev, &all_bdevs, bd_list) {  
>         ret += bdev->bd_inode->i_mapping->nrpages;  
>     }  
>     spin_unlock(&bdev_lock);  
>     return ret;  
> }

我们知道，内核是通过address_space来管理page cache的，那么块设备的address_space在哪里呢？这个稍微复杂一点，涉及多个inode，假设/dev/aaa和/dev/bbb都是指向同一个物理块设备，那么open/dev/aaa和/dev/bbb会分别产生两个inode，我们称之inode_aaa和inode_bbb，但是最后一个块设备的page cache还是需要统一管理起来，不可能分别在inode_aaa和inode_bbb中管理。因此，Linux构造了一个bdev文件系统，保存了系统所有块设备的inode，我们假设该物理块设备在bdev文件系统中的inode是inode_bdev。上面讲了这么多的inode，其实块设备的page cache就是在inode_bdev中管理的。

一般来说，buffers的数量不多，因为产生buffer的操作包括：

1、打开该block device的设备节点，直接进行读写操作（例如dd一个块设备）

2、mount文件系统的时候，需要从相应的block device上直接把块设备上的特定文件系统的super block相关信息读取出来，这些super block的raw data会保存在该block device的page cache中

3、文件操作的时候，和文件元数据相关的操作（例如读取磁盘上的inode相关的信息）也是通过buffer cache进行访问。

Linux中最多处理的是2和3的操作，1的操作很少有。

五、Cached

读写普通文件的时候，我们并不会直接操作磁盘，而是通过page cache来加速文件IO的性能。Cached域描述的就是用于普通文件IO的page cache的数量，具体计算过程如下：

> cached = global_page_state(NR_FILE_PAGES) -  
>             total_swapcache_pages() - i.bufferram;

系统中所有的page cache都会被记录在一个全局的状态中，通过global_page_state(NR_FILE_PAGES)可以知道这个数据，这个数据包括：

1、普通文件的page cache

2、block device 的page cache。参考上一节的描述。

3、swap cache。下一节会进一步描述。

对于Cached这个域，我们只关心普通文件的page cache，因此要从page cache的total number中减去buffer cache和swap cache。

六、SwapCached

其实上一节已经提及swap cache了，也归属于page cache中，具体计算如下：

> unsigned long total_swapcache_pages(void)  
> {  
>     int i;  
>     unsigned long ret = 0;
> 
>     for (i = 0; i < MAX_SWAPFILES; i++)  
>         ret += swapper_spaces[i].nrpages;  
>     return ret;  
> }

和其他的page cache不一样，swap cache并不是为了加快磁盘IO的性能，它是为了解决page frame和swap area之间的同步问题而引入的。例如：一个进程准备swap in一个page的时候，内核的内存回收模块可能同时也在试图将这个page swap out。为了解决这些这些同步问题，内核引入了swap cache这个概念，在任何模块进行swap in或者swap out的时候，都必须首先去swap cache中去看看，而借助page descriptor的PG_locked的标记，我们可以避免swap中的race condition。

swap cache在具体实现的时候，仍然借用了page cache的概念，每一个swap area都有一个address space，管理该swap device（或者swap file）的page cache。因此，一个swap device的所有swap cache仍然是保存在对应address space的radix tree中（仍然是熟悉的配方，仍然是熟悉的味道啊）。

原创文章，转发请注明出处。蜗窝科技，[www.wowotech.net](http://www.wowotech.net/gpio_subsystem/pinctrl-and-gpio.html)。

标签: [meminfo](http://www.wowotech.net/tag/meminfo)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [页面回收的基本概念](http://www.wowotech.net/memory_management/page_reclaim_basic.html) | [linux内核中的GPIO系统之（5）：gpio subsysem和pinctrl subsystem之间的耦合](http://www.wowotech.net/gpio_subsystem/pinctrl-and-gpio.html)»

**评论：**

**xiaobingqiu**  
2021-12-13 14:52

我统计了一下linux中kmalloc和kfree的次数，发现kfree远远大于kmalloc

[回复](http://www.wowotech.net/memory_management/meminfo_1.html#comment-8409)

**[linuxer](http://www.wowotech.net/)**  
2021-12-14 06:57

@xiaobingqiu：理论上kmalloc和kfree应该是一一对应的。你统计的是linux中运行时的kmalloc和kfree的次数吗？

[回复](http://www.wowotech.net/memory_management/meminfo_1.html#comment-8418)

**xiaobingqiu**  
2021-12-14 10:02

@linuxer：对，我是在调用kmalloc的时候将申请的内存的大小记下来，kfree的时候将释放的内存记下来，(并且会记录调用次数)，用于统计kmalloc一共申请了多少内存，但是发现kfree的值大于kmalloc的值，在统计过程中发现kfree调用的次数远远大于kmalloc，所以我就怀疑是不是kfree没有释放成功，然后多次循环调用。

[回复](http://www.wowotech.net/memory_management/meminfo_1.html#comment-8421)

**[linuxer](http://www.wowotech.net/)**  
2021-12-14 21:55

@xiaobingqiu：建议看看是否有double free的bug

[回复](http://www.wowotech.net/memory_management/meminfo_1.html#comment-8423)

**xiaobingqiu**  
2021-12-14 22:14

@linuxer：inuxer，您好，非常感谢您能回答我的问题，我感觉应该是我的理解有问题，我今下午又测试了一下，发现kmalloc申请的次数大于kfree的次数，但是发现最后统计的使用的内存还是小于0；  
used: -4842552(总的内存)  
malloc_num:  219211（申请的次数）  
free_num: 188045（释放的次数）  
我计算的是实际分配的内存，目前主要是没有啥思路来检测是哪里出的问题，（有可能还是没有很好的理解内存机制导致的）。  
代码中添加的方式是：  
kmalloc:  
       struct kmem_cache *s;  
    void * res = NULL;  
  
    if (unlikely(size > KMALLOC_MAX_CACHE_SIZE)) {  
        res = kmalloc_large(size, flags);  
        if (res ) {  
            all_bytes += 1 << (get_order(size) + PAGE_SHIFT);  
            malloc_num ++;  
        }  
        return res;  
    }  
  
    s = kmalloc_slab(size, flags);  
  
    if (unlikely(ZERO_OR_NULL_PTR(s)))  
        return s;  
      
    res = slab_alloc(s, flags, _RET_IP_);  
      
    if (res) {  
        all_byte += s->size;  
        malloc_num ++;  
    }  
      
    trace_kmalloc(_RET_IP_, res, size, s->size, flags);  
  
    kasan_kmalloc(s, res, size, flags);  
    return res;  
kfree:  
    struct page *page;  
    void *object = (void *)x;  
    size_t size = 0;  
    trace_kfree(_RET_IP_, x);  
      
    if (unlikely(ZERO_OR_NULL_PTR(x)))  
        return;  
  
    page = virt_to_head_page(x);  
    if (unlikely(!PageSlab(page))) {  
        
        kfree_hook(x);  
        __free_pages(page, compound_order(page));  
        all_bytes -= 1 << (compound_order(page) + PAGE_SHIFT);  
        free_num ++;  
        return;  
    }  
    size = page->slab_cache->size;  
    slab_free(page->slab_cache, page, object, NULL, 1, _RET_IP_);  
    all_bytes -= size;  
    free_num ++;

[回复](http://www.wowotech.net/memory_management/meminfo_1.html#comment-8426)

**xiaobingqiu**  
2021-12-15 17:36

@xiaobingqiu：linuxer您好，问题找到了，是我实现的有问题，我少统计了__kmalloc申请的内存。（O(∩_∩)O哈哈~），再次感谢，谢谢。

[回复](http://www.wowotech.net/memory_management/meminfo_1.html#comment-8430)

**xiaobingqiu**  
2021-12-13 14:50

linuxer 您好，我想问一下kfree一定能释放成功吗？如果失败了，会怎么处理呀？

[回复](http://www.wowotech.net/memory_management/meminfo_1.html#comment-8408)

**[linuxer](http://www.wowotech.net/)**  
2021-12-14 07:06

@xiaobingqiu：怎么定义成功？怎么定义失败？double free算是失败吗？  
如果内存管理模块本身不存在问题，那么kfree的失败应该用户调用接口的问题，那么需要提供机制检测到这种用户使用接口的bug。  
当然也可能内存管理模块本身存在bug，那么debug就OK了，不需要做什么特别处理。

[回复](http://www.wowotech.net/memory_management/meminfo_1.html#comment-8419)

**xiaobingqiu**  
2021-12-14 10:09

@linuxer：我的简单理解是：成功是指将申请的内存加入到了空闲链表中，失败是指没有将其加入到空闲链表中。double free应该是使用错误了。

[回复](http://www.wowotech.net/memory_management/meminfo_1.html#comment-8422)

**cjw**  
2021-12-25 10:20

@xiaobingqiu：kasan/slab  quarantine 会先把内存放在隔离期里，不会马上加回空闲链表

[回复](http://www.wowotech.net/memory_management/meminfo_1.html#comment-8444)

**rikeyone**  
2018-04-18 17:18

而在这个过程中，memory type的那些page frame被一个个的注入到各个zone的free list中，同时用totalram_pages 来记录那个时间点中空闲的page frame数量。这个点也就是伙伴系统的一个初始内存数量的起点。  
  
--------------------------------------------------------------  
memory type中的page并不是都会被注入到freelist吧？因为其中是有一些和reserved type mem重合的，这部分也会受伙伴系统管理吗？即使是受管理，那也会被当作free page吗？

[回复](http://www.wowotech.net/memory_management/meminfo_1.html#comment-6678)

**Zhenhua**  
2018-05-11 19:21

@rikeyone：是的，注入伙伴系统的是memory-reserved   除了CMA

[回复](http://www.wowotech.net/memory_management/meminfo_1.html#comment-6745)

**lcbj**  
2017-12-13 15:46

感谢解答

[回复](http://www.wowotech.net/memory_management/meminfo_1.html#comment-6360)

**fomalhaut**  
2017-10-27 22:27

对buffer cache的讲解言简意赅，赞！

[回复](http://www.wowotech.net/memory_management/meminfo_1.html#comment-6149)

**gnx**  
2017-09-01 12:22

这篇文章讲/proc/meminfo讲得也不错~  
http://linuxperf.com/?p=142

[回复](http://www.wowotech.net/memory_management/meminfo_1.html#comment-5967)

**fomalhaut**  
2017-10-31 21:28

@gnx：这篇也写的相当详细！

[回复](http://www.wowotech.net/memory_management/meminfo_1.html#comment-6156)

**lcbj**  
2017-08-21 15:57

linuxer 你好，  
目前遇到项目需要把异常日志压缩转存到BMC，请教内核中有现成的压缩函数供使用吗？

[回复](http://www.wowotech.net/memory_management/meminfo_1.html#comment-5937)

**[TonyHo](http://tonyiot.com/)**  
2017-10-11 15:40

@lcbj：内核自带了gzip，lzma，lzo等压缩，也可以自己添加lz4。  
具体可以根据压缩的ratio与speed自己根据情况选择。

[回复](http://www.wowotech.net/memory_management/meminfo_1.html#comment-6101)

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
    
    - ja  
        [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
    - 元神高手  
        [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
    - 十七  
        [内核空间的映射在系统启动时就已经设定好，并且在所有进程的页表...](http://www.wowotech.net/process_management/context-switch-arch.html#8920)
    - lw  
        [sparse模型和disconti模型没看出来有什么本质区别...](http://www.wowotech.net/memory_management/memory_model.html#8919)
    - 肥饶  
        [一个没设置好就出错](http://www.wowotech.net/linux_kenrel/516.html#8918)
    - orange  
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
    
    - [CFS调度器（3）-组调度](http://www.wowotech.net/process_management/449.html)
    - [ACCESS_ONCE宏定义的解释](http://www.wowotech.net/process_management/access-once.html)
    - [X-010-UBOOT-使用booti命令启动kernel(Bubblegum-96平台)](http://www.wowotech.net/x_project/bubblegum_uboot_booti.html)
    - [Linux设备模型(4)_sysfs](http://www.wowotech.net/device_model/dm_sysfs.html)
    - [X-026-KERNEL-Linux gpio driver的移植之gpio range](http://www.wowotech.net/x_project/kernel_gpio_driver_porting_2.html)
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
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

作者：[itrocker](http://www.wowotech.net/author/295) 发布于：2015-11-2 10:24 分类：[内存管理](http://www.wowotech.net/sort/memory_management)

Linux kernel组织管理物理内存的方式是buddy system（伙伴系统），而物理内存碎片正式buddy system的弱点之一，为了预防以及解决碎片问题，kernel采取了一些实用技术，这里将对这些技术进行总结归纳。

**1** **低内存时整合碎片**

从buddy申请内存页，如果找不到合适的页，则会进行两步调整内存的工作，compact和reclaim。前者是为了整合碎片，以得到更大的连续内存；后者是回收不一定必须占用内存的缓冲内存。这里重点了解comact，整个流程大致如下：

__alloc_pages_nodemask

    -> __alloc_pages_slowpath

        -> __alloc_pages_direct_compact

            -> try_to_compact_pages

                -> compact_zone_order

                    -> compact_zone

                        -> isolate_migratepages

                        -> migrate_pages

                        -> release_freepages

并不是所有申请不到内存的场景都会compact，首先要满足order大于0，并且gfp_mask携带__GFP_FS和__GFP_IO；另外，需要zone的剩余内存情况满足一定条件，kernel称之为“碎片指数”（fragmentation index），这个值在0~1000之间，默认碎片指数大于500时才能进行compact，可以通过proc文件extfrag_threshold来调整这个默认值。fragmentation index通过fragmentation_index函数来计算：

  

1.         /*
2. 	 * Index is between 0 and 1000
3. 	 *
4. 	 * 0 => allocation would fail due to lack of memory
5. 	 * 1000 => allocation would fail due to fragmentation
6. 	 */
7. 	return 1000 - div_u64( (1000+(div_u64(info->free_pages * 1000ULL, requested))), info->free_blocks_total)

在整合内存碎片的过程中，碎片页只会在本zone的内部移动，将位于zone低地址的页尽量移到zone的末端。申请新的页面位置通过compaction_alloc函数实现。

移动过程又分为同步和异步，内存申请失败后第一次compact将会使用异步，后续reclaim之后将会使用同步。同步过程只移动当面未被使用的页，异步过程将遍历并等待所有MOVABLE的页使用完成后进行移动。

**2** **按可移动性组织页**

按照可移动性将内存页分为以下三个类型：

UNMOVABLE：在内存中位置固定，不能随意移动。kernel分配的内存基本属于这个类型；

RECLAIMABLE：不能移动，但可以删除回收。例如文件映射内存；

MOVABLE：可以随意移动，用户空间的内存基本属于这个类型。

申请内存时，根据可移动性，首先在指定类型的空闲页中申请内存，每个zone的空闲内存组织方式如下：

1. struct zone {
2. ......
3. struct free_area    free_area[MAX_ORDER];
4. ......
5. }

7. struct free_area {
8. 	struct list_head	free_list[MIGRATE_TYPES];
9. 	unsigned long		nr_free;
10. };

当在指定类型的free_area申请不到内存时，可以从备用类型挪用，挪用之后的内存就会释放到新指定的类型列表中，kernel把这个过程称为“盗用”。

备用类型优先级列表如下定义：

1. static int fallbacks[MIGRATE_TYPES][4] = {
2. 	[MIGRATE_UNMOVABLE]   = { MIGRATE_RECLAIMABLE, MIGRATE_MOVABLE,     MIGRATE_RESERVE },
3. 	[MIGRATE_RECLAIMABLE] = { MIGRATE_UNMOVABLE,   MIGRATE_MOVABLE,     MIGRATE_RESERVE },
4. #ifdef CONFIG_CMA
5. 	[MIGRATE_MOVABLE]     = { MIGRATE_CMA,         MIGRATE_RECLAIMABLE, MIGRATE_UNMOVABLE, MIGRATE_RESERVE },
6. 	[MIGRATE_CMA]         = { MIGRATE_RESERVE }, /* Never used */
7. #else
8. 	[MIGRATE_MOVABLE]     = { MIGRATE_RECLAIMABLE, MIGRATE_UNMOVABLE,   MIGRATE_RESERVE },
9. #endif
10. 	[MIGRATE_RESERVE]     = { MIGRATE_RESERVE }, /* Never used */
11. #ifdef CONFIG_MEMORY_ISOLATION
12. 	[MIGRATE_ISOLATE]     = { MIGRATE_RESERVE }, /* Never used */
13. #endif
14. };

  

值得注意的是并不是所有场景都适合按可移动性组织页，当内存大小不足以分配到各种类型时，就不适合启用可移动性。有个全局变量来表示是否启用，在内存初始化时设置：

1. void __ref build_all_zonelists(pg_data_t *pgdat, struct zone *zone)
2. {
3. ......
4.     if (vm_total_pages < (pageblock_nr_pages * MIGRATE_TYPES))
5. 		page_group_by_mobility_disabled = 1;
6. 	else
7. 		page_group_by_mobility_disabled = 0;
8. ......
9. }

如果page_group_by_mobility_disabled，则所有内存都是不可移动的。

其中有个参数决定了每个内存区域至少拥有的页，pageblock_nr_pages，它的定义如下：

  

#define pageblock_order HUGETLB_PAGE_ORDER

1. #else /* CONFIG_HUGETLB_PAGE */
2. /* If huge pages are not used, group by MAX_ORDER_NR_PAGES */
3. #define pageblock_order		(MAX_ORDER-1)
4. #endif /* CONFIG_HUGETLB_PAGE */
5. #define pageblock_nr_pages	(1UL << pageblock_order)

在系统初始化期间，所有页都被标记为MOVABLE：

1. void __meminit memmap_init_zone(unsigned long size, int nid, unsigned long zone,
2. 		unsigned long start_pfn, enum memmap_context context)
3. {
4. ......
5. 		if ((z->zone_start_pfn <= pfn)
6. 		    && (pfn < zone_end_pfn(z))
7. 		    && !(pfn & (pageblock_nr_pages - 1)))
8. 			set_pageblock_migratetype(page, MIGRATE_MOVABLE);
9. ......
10. }

其它可移动性类型的页都是后来产生的，也就是前面说的“盗取”。在这种情况发生时，通常会“盗取”fallback中更高优先级、更大块连续的页，从而避免小碎片的产生。

1. /* Remove an element from the buddy allocator from the fallback list */
2. static inline struct page *
3. __rmqueue_fallback(struct zone *zone, int order, int start_migratetype)
4. {
5. ......
6. 	/* Find the largest possible block of pages in the other list */
7. 	for (current_order = MAX_ORDER-1; current_order >= order;
8. 						--current_order) {
9. 		for (i = 0;; i++) {
10. 			migratetype = fallbacks[start_migratetype][i];
11. ......
12. }

可以通过/proc/pageteypeinfo查看当前系统各种类型的页分布。

**3** **虚拟可移动内存域**

在依据可移动性组织页的技术之前，还有一个方法已经合入kernel，那就是虚拟内存域：ZONE_MOVABLE。基本思想很简单：把内存分为两部分，可移动的和不可移动的。

1. enum zone_type {
2. #ifdef CONFIG_ZONE_DMA
3. 	ZONE_DMA,
4. #endif
5. #ifdef CONFIG_ZONE_DMA32
6. 	ZONE_DMA32,
7. #endif
8. 	ZONE_NORMAL,
9. #ifdef CONFIG_HIGHMEM
10. 	ZONE_HIGHMEM,
11. #endif
12. 	ZONE_MOVABLE,
13. 	__MAX_NR_ZONES
14. };

  

ZONE_MOVABLE的启用需要指定kernel参数kernelcore或者movablecore，kernelcore用来指定不可移动的内存数量，movablecore指定可移动的内存大小，如果两个都指定，取不可移动内存数量较大的一个。如果都不指定，则不启动。

与其它内存域不同的是ZONE_MOVABLE不关联任何物理内存范围，该域的内存取自高端内存域或者普通内存域。

find_zone_movable_pfns_for_nodes用来计算每个node中ZONE_MOVABLE的内存数量，采用的内存区域通常是每个node的最高内存域，在函数find_usable_zone_for_movable中体现。

在对每个node分配ZONE_MOVABLE内存时，kernelcore会被平均分配到各个Node：

kernelcore_node = required_kernelcore / usable_nodes;

在kernel alloc page时，如果gfp_flag同时指定了__GFP_HIGHMEM和__GFP_MOVABLE，则会从ZONE_MOVABLE内存域申请内存。

  

标签: [内存碎片](http://www.wowotech.net/tag/%E5%86%85%E5%AD%98%E7%A2%8E%E7%89%87)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [實作 spinlock on raspberry pi 2](http://www.wowotech.net/231.html) | [ARM64的启动过程之（五）：UEFI](http://www.wowotech.net/armv8a_arch/UEFI.html)»

**评论：**

**davidlamb**  
2019-08-08 15:50

碎片整合对系统的性能有没什么影响？

[回复](http://www.wowotech.net/memory_management/memory-fragment.html#comment-7580)

**Bob**  
2018-03-07 17:46

请问个问题，我们有个程序要申请大块内存的DMA，总的free内存是够得，但就是申请速度特别慢，分钟级。  
  
把系统 extfrag_threshold 改成 1000后，申请速度快了很多。  
  
按照你文章的意思，是慢在做内存 compact 上，改成 1000 后优先 reclaim 了，所以变快了。  
  
但我想不通的是，不 compact 怎么有连续内存可用呢？  
  
或者是说 extfrag_threshold 改成 1000 还有个隐含意思是，系统会去释放 buffer/cache 内存？

[回复](http://www.wowotech.net/memory_management/memory-fragment.html#comment-6598)

**[komex](http://www.wowotech.net/)**  
2016-10-25 10:43

您好，请教个问题，在实际的驱动程序中，有会用到MIGRATE_UNMOVABLE这个类型的吗？看了下code，好像没看到给page标志MIGRATE_UNMOVABLE这个类型的code. 谢谢！

[回复](http://www.wowotech.net/memory_management/memory-fragment.html#comment-4775)

**[wowo](http://www.wowotech.net/)**  
2016-10-25 12:21

@komex：你可以参考kernel中的这段代码：  
        /*                                                                        
         * Disable grouping by mobility if the number of pages in the            
         * system is too low to allow the mechanism to work. It would be          
         * more accurate, but expensive to check per-zone. This check is          
         * made on memory-hotadd so a system can start with mobility              
         * disabled and enable it later                                          
         */                                                                      
        if (vm_total_pages < (pageblock_nr_pages * MIGRATE_TYPES))                
                page_group_by_mobility_disabled = 1;                              
        else                                                                      
                page_group_by_mobility_disabled = 0;

[回复](http://www.wowotech.net/memory_management/memory-fragment.html#comment-4777)

**[komex](http://www.wowotech.net/)**  
2016-10-25 14:28

@wowo：这段是初始化的时候的代码吧，一般page_group_by_mobility_disabled 都等于 0 吧，那都初始化好之后，后面申请的都没有用到 MIGRATE_UNMOVABLE 这个type吗？  
谢谢！

[回复](http://www.wowotech.net/memory_management/memory-fragment.html#comment-4778)

**[wowo](http://www.wowotech.net/)**  
2016-10-25 14:35

@komex：看代码应该是这样。如果系统的page比较少，少到不足以支撑相应的机制的时候，就使用MIGRATE_UNMOVABLE，也就变相的禁止了内存整理的功能。

[回复](http://www.wowotech.net/memory_management/memory-fragment.html#comment-4779)

**[komex](http://www.wowotech.net/)**  
2016-10-25 14:38

@wowo：在分配时，有一个gfpflags_to_migratetype(const gfp_t gfp_flags)函数  
  
实现从 gfp_flags 转成 migrate_type。但分配的时候，一般我们给的都是GFP_KERNEL，照这个code看的话，用到的migrate_type都是同一个类型，所以就有点疑惑，不知道具体其他几种type的使用情况，求指点！  
谢谢！  
  
code贴不上来，提示说黑客攻击，不知道是不是code问题，删了试下~ 囧~~~~

[回复](http://www.wowotech.net/memory_management/memory-fragment.html#comment-4780)

**[wowo](http://www.wowotech.net/)**  
2016-10-26 21:48

@komex：code贴不上的问题已经解决了。话说内存这一块的东西实在是太多了，还是等linuxer同学的分析文章吧，我相信他会把大家的问题讲明白的:-)

[回复](http://www.wowotech.net/memory_management/memory-fragment.html#comment-4784)

**[komex](http://www.wowotech.net/)**  
2016-10-27 11:04

@wowo：好的，谢谢！期待后续的分析文章~~  
谢谢！

**看客**  
2019-11-29 08:22

@komex：事实上gfpflags_to_migratetype（）的代码已经把这个事情说清楚了。当kernel想要分配内存的时候，其传入的GFP_KERNEL:  
#define GFP_KERNEL      (__GFP_RECLAIM | __GFP_IO | __GFP_FS)  
#define __GFP_RECLAIM ((__force gfp_t)(___GFP_DIRECT_RECLAIM|___GFP_KSWAPD_RECLAIM))  
  
也就是说如果你没有显式传入__GFP_RECLAIMABLE或者__GFP_MOVABLE，那么gfp_flags中关于migratetype的值就是0,0不就是MIGRATE_UNMOVABLE的值吗。  
  
#define GFP_MOVABLE_MASK (__GFP_RECLAIMABLE|__GFP_MOVABLE)  
#define GFP_MOVABLE_SHIFT 3

[回复](http://www.wowotech.net/memory_management/memory-fragment.html#comment-7768)

**jixing**  
2016-01-19 09:04

linuxer：  
    膜拜  
能加个好友吗？我对linux内核有比较强的兴趣， 但是接触时间较短， 想您指导一下。。。

[回复](http://www.wowotech.net/memory_management/memory-fragment.html#comment-3414)

**[郭健](http://www.wowotech.net/)**  
2016-01-19 12:19

@jixing：可以的，我没有QQ，不过有微信，如果你足够仔细，应该可以搜索到我，不过其实这个网站也很合适交流的。

[回复](http://www.wowotech.net/memory_management/memory-fragment.html#comment-3417)

**zhuochong**  
2015-11-11 17:24

大神，如何避免出现内核碎片的问题？可以从哪几个方面下手？

[回复](http://www.wowotech.net/memory_management/memory-fragment.html#comment-3029)

**[itrocker](http://www.wowotech.net/)**  
2015-11-12 19:06

@zhuochong：这个问题好大：）    
碎片问题难就难在无法完全避免，即便现在已经有了上述几个技术，问题还是存在，我们能做的就是尽量降低概率、延迟时间。  
对于开发者而言，就要尽量减少在buddy system中持续地申请释放page。  
可以考虑在上层实现内存池；复用slab技术；优化malloc库；甚至在用户不感知的情况下重启系统服务等等

[回复](http://www.wowotech.net/memory_management/memory-fragment.html#comment-3039)

**[logv](http://www.wowotech.net/)**  
2015-11-05 09:01

请教一下，“首先要满足order大于0，并且gfp_mask携带__GFP_FS和__GFP_IO”，这的order，以及这两个mask，实际意义是什么呢？谢谢

[回复](http://www.wowotech.net/memory_management/memory-fragment.html#comment-2999)

**[itrocker](http://www.wowotech.net/)**  
2015-11-05 14:25

@logv：（1）order大于0。buddy system是以page为单位组织的，所谓“碎片整合”就是把分散在各个地方的不连续页调整在一起，形成连续的多个页。如果申请的order=0，不要求多个连续的页，也就没必要整合了；  
（2）要求__GFP_FS和__GFP_IO。是因为在碎片整合过程中，有可能进行“页迁移”，这个操作需要进行页面回写，操作VFS并访问磁盘IO，因此需要携带这两个flag。

[回复](http://www.wowotech.net/memory_management/memory-fragment.html#comment-3000)

**[logv](http://www.wowotech.net/)**  
2015-11-05 16:15

@itrocker：多谢指点

[回复](http://www.wowotech.net/memory_management/memory-fragment.html#comment-3001)

**rainys**  
2016-01-04 16:57

@itrocker：不带着两个标记，有可能造成申请流程的嵌套死锁，这个是实现层面的意义

[回复](http://www.wowotech.net/memory_management/memory-fragment.html#comment-3337)

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
    
    - [Meltdown论文翻译](http://www.wowotech.net/basic_subject/meltdown.html)
    - [玩转BLE(1)_Eddystone beacon](http://www.wowotech.net/bluetooth/eddystone_test.html)
    - [Linux时间子系统之（十四）：tick broadcast framework](http://www.wowotech.net/timer_subsystem/tick-broadcast-framework.html)
    - [串口通信技术浅析](http://www.wowotech.net/basic_tech/serial_intro.html)
    - [Linux reset framework](http://www.wowotech.net/pm_subsystem/reset_framework.html)
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
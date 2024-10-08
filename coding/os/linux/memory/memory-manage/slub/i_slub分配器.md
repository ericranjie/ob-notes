作者：[itrocker](http://www.wowotech.net/author/295) 发布于：2015-12-21 18:51 分类：[内存管理](http://www.wowotech.net/sort/memory_management)

Linux的物理内存管理采用了以页为单位的buddy system（伙伴系统），但是很多情况下，内核仅仅需要一个较小的对象空间，而且这些小块的空间对于不同对象又是变化的、不可预测的，所以需要一种类似用户空间堆内存的管理机制(malloc/free)。然而内核对对象的管理又有一定的特殊性，有些对象的访问非常频繁，需要采用缓冲机制；对象的组织需要考虑硬件cache的影响；需要考虑多处理器以及NUMA架构的影响。90年代初期，在Solaris 2.4操作系统中，采用了一种称为“slab”（原意是大块的混凝土）的缓冲区分配和管理方法，在相当程度上满足了内核的特殊需求。

多年以来，SLAB成为linux kernel对象缓冲区管理的主流算法，甚至长时间没有人愿意去修改，因为它实在是非常复杂，而且在大多数情况下，它的工作完成的相当不错。但是，随着大规模多处理器系统和 NUMA系统的广泛应用，SLAB 分配器逐渐暴露出自身的严重不足：

1). 缓存队列管理复杂；
2). 管理数据存储开销大；
3). 对NUMA支持复杂；
4). 调试调优困难；
5). 摒弃了效果不太明显的slab着色机制；

针对这些SLAB不足，内核开发人员Christoph Lameter在Linux内核2.6.22版本中引入一种新的解决方案：SLUB分配器。SLUB分配器特点是简化设计理念，同时保留SLAB分配器的基本思想：每个缓冲区由多个小的slab 组成，每个 slab 包含固定数目的对象。SLUB分配器简化kmem_cache，slab等相关的管理数据结构，摒弃了SLAB 分配器中众多的队列概念，并针对多处理器、NUMA系统进行优化，从而提高了性能和可扩展性并降低了内存的浪费。为了保证内核其它模块能够无缝迁移到SLUB分配器，SLUB还保留了原有SLAB分配器所有的接口API函数。

（注：本文源码分析基于linux-4.1.x）

整体数据结构关系如下图所示：
![[Pasted image 20241008093809.png]]
# **1 SLUB****分配器的初始化**

SLUB初始化有两个重要的工作：第一，创建用于申请struct kmem_cache和struct kmem_cache_node的kmem_cache；第二，创建用于常规kmalloc的kmem_cache。
## **1.1** **申请kmem_cache****的kmem_cache**

第一个工作涉及到一个“先有鸡还是先有蛋”的问题，因为创建kmem_cache需要从kmem_cache的缓冲区申请，而这时候还没有创建kmem_cache的缓冲区。kernel的解决办法是先用两个静态变量boot_kmem_cache和boot_kmem_cache_node来保存struct kmem_cach和struct kmem_cache_node缓冲区管理数据，以两个静态变量为基础申请大小为struct kmem_cache和struct kmem_cache_node对象大小的slub缓冲区，随后再从这些缓冲区中分别申请两个kmem_cache，然后把boot_kmem_cache和boot_kmem_cache_node中的内容拷贝到新申请的对象中，从而完成了struct kmem_cache和struct kmem_cache_node管理结构的bootstrap（自引导）。
```cpp
1. void __init kmem_cache_init(void)
2. {
3. 	//声明静态变量，存储临时kmem_cache管理结构
4. 	static __initdata struct kmem_cache boot_kmem_cache,
5. 		boot_kmem_cache_node; 
6. •••
7. 	kmem_cache_node = &boot_kmem_cache_node;
8. 	kmem_cache = &boot_kmem_cache;

10. 	//申请slub缓冲区，管理数据放在临时结构中
11. 	create_boot_cache(kmem_cache_node, "kmem_cache_node",
12. 		sizeof(struct kmem_cache_node), SLAB_HWCACHE_ALIGN);
13. 	create_boot_cache(kmem_cache, "kmem_cache",
14. 			offsetof(struct kmem_cache, node) +
15. 				nr_node_ids * sizeof(struct kmem_cache_node *),
16. 		       SLAB_HWCACHE_ALIGN);

18. 	//从刚才挂在临时结构的缓冲区中申请kmem_cache的kmem_cache，并将管理数据拷贝到新申请的内存中
19. 	kmem_cache = bootstrap(&boot_kmem_cache);

21. 	//从刚才挂在临时结构的缓冲区中申请kmem_cache_node的kmem_cache，并将管理数据拷贝到新申请的内存中
22. 	kmem_cache_node = bootstrap(&boot_kmem_cache_node);
23. •••
24. }
```
## **1.2** **创建kmalloc常规缓存**

原则上系统会为每个2次幂大小的内存块申请一个缓存，但是内存块过小时，会产生很多碎片浪费，所以系统为96B和192B也各自创建了一个缓存。于是利用了一个size_index数组来帮助确定小于192B的内存块使用哪个缓存
```cpp
1. void __init create_kmalloc_caches(unsigned long flags)
2. {
3. •••
4. 	/*使用SLUB时KMALLOC_SHIFT_LOW=3，KMALLOC_SHIFT_HIGH=13
5. 	也就是说使用kmalloc能够申请的最小内存是8B，最大内存是8KB
6. 	申请内存是向上对其2的n次幂，创建对应大小缓存保存在kmalloc_caches [n]*/
7. 	for (i = KMALLOC_SHIFT_LOW; i <= KMALLOC_SHIFT_HIGH; i++) {
8. 		if (!kmalloc_caches[i]) {
9. 			kmalloc_caches[i] = create_kmalloc_cache(NULL,
10. 							1 << i, flags);
11. 		}

13. 		/*
14. 		 * Caches that are not of the two-to-the-power-of size.
15. 		 * These have to be created immediately after the
16. 		 * earlier power of two caches
17. 		 */
18. 		/*有两个例外，大小为64~96B和128B~192B，单独创建了两个缓存
19. 		保存在kmalloc_caches [1]和kmalloc_caches [2]*/
20. 		if (KMALLOC_MIN_SIZE <= 32 && !kmalloc_caches[1] && i == 6)
21. 			kmalloc_caches[1] = create_kmalloc_cache(NULL, 96, flags);

23. 		if (KMALLOC_MIN_SIZE <= 64 && !kmalloc_caches[2] && i == 7)
24. 			kmalloc_caches[2] = create_kmalloc_cache(NULL, 192, flags);
25. 	}
26. •••
27. }
```
# **2** **缓存的创建与销毁**
## **2.1** **缓存的创建**

创建缓存通过接口kmem_cache_create进行，在创建新的缓存以前，尝试找到可以合并的缓存，合并条件包括对对象大小以及缓存属性的判断，如果可以合并则直接返回已存在的kmem_cache，并创建一个kobj链接指向同一个节点。

创建新的缓存主要是申请管理结构暂用的空间，并初始化，这些管理结构包括kmem_cache、kmem_cache_nodes、kmem_cache_cpu。同时在sysfs创建kobject节点。最后把kmem_cache加入到全局cahce链表slab_caches中。
![[Pasted image 20241008093858.png]]
## **2.2** **缓存的销毁**

销毁过程比创建过程简单的多，主要工作是释放partial队列所有page，释放kmem_cache_cpu，释放每个node的kmem_cache_node，最后释放kmem_cache本身。

![](http://www.wowotech.net/content/uploadfile/201512/10fb1450695162.png)
# **3** **申请对象**

对象是SLUB分配器中可分配的内存单元，与SLAB相比，SLUB对象的组织非常简洁，申请过程更加高效。SLUB没有任何管理区结构来管理对象，而是将对象之间的关联嵌入在对象本身的内存中，因为申请者并不关心对象在分配之前内存的内容是什么。而且各个SLUB之间的关联，也利用page自身结构进行处理。

每个CPU都有一个slab作为本地高速缓存，只要slab所在的node与申请者要求的node匹配，同时该slab还有空闲对象，则直接在cpu_slab中取出空闲对象，否则就进入慢速路径。

每个对象内存的offset偏移位置都存放着下一个空闲对象，offset通常为0，也就是复用对象内存的第一个字来保存下一个空闲对象的指针，当满足条件(flags & (SLAB_DESTROY_BY_RCU | SLAB_POISON)) 或者有对象构造函数时，offset不为0，每个对象的结构如下图。

cpu_slab的freelist则保存着当前第一个空闲对象的地址。

![](http://www.wowotech.net/content/uploadfile/201512/09dd1450695162.png)

  

如果本地CPU缓存没有空闲对象，则申请新的slab；如果有空闲对象，但是内存node不相符，则deactive当前cpu_slab，再申请新的slab。

![](http://www.wowotech.net/content/uploadfile/201512/82661450695162.png)

deactivate_slab主要进行两步工作：

第一步，将cpu_slab的freelist全部释放回page->freelist；
第二部，根据page(slab)的状态进行不同操作，如果该slab有部分空闲对象，则将page移到kmem_cache_node的partial队列；如果该slab全部空闲，则直接释放该slab；如果该slab全部占用，而且开启了CONFIG_SLUB_DEBUG编译选项，则将page移到full队列。

page的状态也从frozen改变为unfrozen。（frozen代表slab在cpu_slub，unfroze代表在partial队列或者full队列）。

申请新的slab有两种情况，如果cpu_slab的partial队列不为空，则取出队列中下一个page作为新的cpu_slab，同时再次检测内存node是否相符，还不相符则循环处理。

如果cpu_slab的partial队列为空，则查看本node的partial队列是否为空，如果不空，则取出page；如果为空，则看一定距离范围内其它node的partial队列，如果还为空，则需要创建新slab。

创建新slab其实就是申请对应order的内存页，用来放足够数量的对象。值得注意的是其中order以及对象数量的确定，这两者又是相互影响的。order和object数量同时存放在kmem_cache成员kmem_cache_order_objects中，低16位用于存放object数量，高位存放order。order与object数量的关系非常简单：((PAGE_SIZE << order) - reserved) / size。

下面重点看calculate_order这个函数
```cpp
1. static inline int calculate_order(int size, int reserved)
2. {
3. •••
4. 	//尝试找到order与object数量的最佳配合方案
5. 	//期望的效果就是剩余的碎片最小
6. 	min_objects = slub_min_objects;
7. 	if (!min_objects)
8. 		min_objects = 4 * (fls(nr_cpu_ids) + 1);
9. 	max_objects = order_objects(slub_max_order, size, reserved);
10. 	min_objects = min(min_objects, max_objects);

12. 	//fraction是碎片因子，需要满足的条件是碎片部分乘以fraction小于slab大小
13. 	// (slab_size - reserved) % size <= slab_size / fraction
14. 	while (min_objects > 1) {
15. 		fraction = 16;
16. 		while (fraction >= 4) {
17. 			order = slab_order(size, min_objects,
18. 					slub_max_order, fraction, reserved);
19. 			if (order <= slub_max_order)
20. 				return order;
21. 	        //放宽条件，容忍的碎片大小增倍
22. 			fraction /= 2;
23. 		}
24. 		min_objects--;
25. 	}

27. 	//尝试一个slab只包含一个对象
28. 	order = slab_order(size, 1, slub_max_order, 1, reserved);
29. 	if (order <= slub_max_order)
30. 		return order;

32. 	//使用MAX_ORDER且一个slab只含一个对象
33. 	order = slab_order(size, 1, MAX_ORDER, 1, reserved);
34. 	if (order < MAX_ORDER)
35. 		return order;
36. 	return -ENOSYS;
37. }
```
# **4** **释放对象**

从上面申请对象的流程也可以看出，释放的object有几个去处：

1）cpu本地缓存slab，也就是cpu_slab；
2）放回object所在的page（也就是slab）中；另外要处理所在的slab：
2.1）如果放回之后，slab完全为空，则直接销毁该slab；
2.2）如果放回之前，slab为满，则判断slab是否已被冻结；如果已冻结，则不需要做其他事；如果未冻结，则将其冻结，放入cpu_slab的partial队列；如果cpu_slab partial队列过多，则将队列中所有slab一次性解冻到各自node的partial队列中。

值得注意的是cpu partial队列的功能是个可选项，依赖于内核选项CONFIG_SLUB_CPU_PARTIAL，如果没有开启，则不使用cpu partial队列，直接使用各个node的partial队列。

标签: [slub](http://www.wowotech.net/tag/slub) [slab](http://www.wowotech.net/tag/slab)

---

« [perfbook memory barrier（14.2章节）的中文翻译（上）](http://www.wowotech.net/kernel_synchronization/memory-barrier-1.html) | [Linux graphic subsytem(1)_概述](http://www.wowotech.net/graphic_subsystem/graphic_subsystem_overview.html)»

**评论：**

**张小花**  
2017-08-08 11:01

//http://elixir.free-electrons.com/linux/latest/source/mm/slub.c#L2636  
static __always_inline void *slab_alloc_node(struct kmem_cache *s,  
        gfp_t gfpflags, int node, unsigned long addr)  
{  
    void *object;  
    struct kmem_cache_cpu *c;  
    struct page *page;  
    unsigned long tid;  
    //钩子函数 可以做一些事情 比如测试和mmcgroup?  
    s = slab_pre_alloc_hook(s, gfpflags);  
    if (!s)  
        return NULL;  
redo:  
    /*  
     * Must read kmem_cache cpu data via this cpu ptr. Preemption is  
     * enabled. We may switch back and forth between cpus while  
     * reading from one cpu area. That does not matter as long  
     * as we end up on the original cpu again when doing the cmpxchg.  
     *  
     * We should guarantee that tid and kmem_cache are retrieved on  
     * the same cpu. It could be different if CONFIG_PREEMPT so we need  
     * to check if it is matched or not.  
     */  
    //看了一段时间 看不懂这里为何这么写  
    do {  
        tid = this_cpu_read(s->cpu_slab->tid);  
        c = raw_cpu_ptr(s->cpu_slab);  
    } while (IS_ENABLED(CONFIG_PREEMPT) &&  
         unlikely(tid != READ_ONCE(c->tid)));  
  
这个问题是不是可以归纳为:内核是如何解决在slab函数期间发生中断导致抢占与进程迁移的问题?

[回复](http://www.wowotech.net/memory_management/247.html#comment-5888)

**[wowo](http://www.wowotech.net/)**  
2017-08-08 14:14

@张小花：是的。  
这个注释已经说的很明白了：  
         * We should guarantee that tid and kmem_cache are retrieved on          
         * the same cpu. It could be different if CONFIG_PREEMPT so we need      
         * to check if it is matched or not.    
就是要保证tid和kmem_cache属于同一个CPU。  
无论此时发生了多少抢占，一直等到相同，再配合后面的barrier()，就能达到目的。

[回复](http://www.wowotech.net/memory_management/247.html#comment-5894)

**张小花**  
2017-08-08 14:54

@wowo：谢谢wowo  
原来它的目的是为了解决slab_alloc_node中Fastpath下进程迁移的问题.  
对于Slowpath则用另一套方案:关中断并重新读取当前CPU的kmem_cache.  
  
但是 while (IS_ENABLED(CONFIG_PREEMPT) &&  
         unlikely(tid != READ_ONCE(c->tid))); 中,  
tid != READ_ONCE(c->tid)不同的CPU也可能相等啊,也就是说此While循环并不能百分百的保证tid和kmem_cache属于同一个CPU吧?

[回复](http://www.wowotech.net/memory_management/247.html#comment-5896)

**[wowo](http://www.wowotech.net/)**  
2017-08-08 16:13

@张小花：可以保证的。如果使能了preempt，tid在不同CPU之间是可以区分，因此是唯一的，可参考：  
  
mm/slub.c  
  
#ifdef CONFIG_PREEMPT                                                            
/*                                                                                
* Calculate the next globally unique transaction for disambiguiation            
* during cmpxchg. The transactions start with the cpu number and are then        
* incremented by CONFIG_NR_CPUS.                                                
*/                                                                              
#define TID_STEP  roundup_pow_of_two(CONFIG_NR_CPUS)                              
#else

[回复](http://www.wowotech.net/memory_management/247.html#comment-5897)

**李勇**  
2016-04-06 19:27

你好，能请问下文章里面的图是用什么软件来绘制的吗？

[回复](http://www.wowotech.net/memory_management/247.html#comment-3802)

**[wowo](http://www.wowotech.net/)**  
2016-04-07 08:46

@李勇：visio吧

[回复](http://www.wowotech.net/memory_management/247.html#comment-3806)

**[wittyman](http://www.wowotech.net/)**  
2016-03-15 19:32

楼主，描述slab object layout那张图中，关于red zone的部分不是很准确，obj align前面那个red zone，里面实际上是pad数据（POSION_INUSE）,和前面个red zone不一样。这样画可能会引起误解。

[回复](http://www.wowotech.net/memory_management/247.html#comment-3672)

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
    
    - [X-026-KERNEL-Linux gpio driver的移植之gpio range](http://www.wowotech.net/x_project/kernel_gpio_driver_porting_2.html)
    - [CFS调度器（5）-带宽控制](http://www.wowotech.net/process_management/451.html)
    - [Linux MMC framework(1)_软件架构](http://www.wowotech.net/comm/mmc_framework_arch.html)
    - [显示技术介绍(3)_CRT技术](http://www.wowotech.net/display/crt_intro.html)
    - [Process Creation（一）](http://www.wowotech.net/process_management/Process-Creation-1.html)
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
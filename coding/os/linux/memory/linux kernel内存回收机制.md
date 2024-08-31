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

作者：[itrocker](http://www.wowotech.net/author/295) 发布于：2015-11-12 20:37 分类：[内存管理](http://www.wowotech.net/sort/memory_management)

无论计算机上有多少内存都是不够的，因而linux kernel需要回收一些很少使用的内存页面来保证系统持续有内存使用。页面回收的方式有页回写、页交换和页丢弃三种方式：如果一个很少使用的页的后备存储器是一个块设备（例如文件映射），则可以将内存直接同步到块设备，腾出的页面可以被重用；如果页面没有后备存储器，则可以交换到特定swap分区，再次被访问时再交换回内存；如果页面的后备存储器是一个文件，但文件内容在内存不能被修改（例如可执行文件），那么在当前不需要的情况下可直接丢弃。

**1** **回收的时机**

![](http://www.wowotech.net/content/uploadfile/201511/4a471447331961.png)

**2** **哪些内存可以回收**

**2.1** **页框的回收**

LRU(Least Recently Used)，近期最少使用链表，是按照近期的使用情况排列的，最少使用的存在链表末尾，通过以下宏定义即可看出：

#define lru_to_page(_head) (list_entry((_head)->prev, struct page, lru))

每个zone有5个LRU链表用以存放各种最近使用状态的页面。

enum lru_list {

         LRU_INACTIVE_ANON = LRU_BASE,

         LRU_ACTIVE_ANON = LRU_BASE + LRU_ACTIVE,

         LRU_INACTIVE_FILE = LRU_BASE + LRU_FILE,

         LRU_ACTIVE_FILE = LRU_BASE + LRU_FILE + LRU_ACTIVE,

         LRU_UNEVICTABLE,

         NR_LRU_LISTS

};

其中INACTIVE_ANON、ACTIVE_ANON、INACTIVE_FILE、ACTIVE_FILE 4个链表中的页面是可以回收的。ANON代表匿名映射，没有后备存储器；FILE代表文件映射。

页面回收时，会优先回收INACTIVE的页面，只有当INACTIVE页面很少时，才会考虑回收ACTIVE页面。

为了评估页的活动程度，kernel引入了PG_referend和PG_active两个标志位。为什么需要两个位呢？假定只使用一个PG_active来标识页是否活动，在页被访问时，设置该位，但是何时清楚呢？为此需要维护大量的内核定时器，这种方法注定是要失败的。

使用两个标志，可以实现一种更精巧的方法，其核心思想是：一个表示当前活动程度，一个表示最近是否被引用过，下图说明了基本算法。

![](http://www.wowotech.net/content/uploadfile/201511/fb5c1447331983.png)

基本上有以下步骤：

（1）如果页是活动的，设置PG_active位，并保存在ACTIVE LRU链表；反之在INACTIVE；

（2）每次访问页时，设置PG_referenced位，负责该工作的是mark_page_accessed函数；

（3）PG_referenced以及由逆向映射提供的信息用来确定页面活动程度，每次清除该位时，都会检测页面活动程度，page_referenced函数实现了该行为；

（4）再次进入mark_page_accessed。如果发现PG_referenced已被置位，意味着page_referenced没有执行检查，因而对于mark_page_accessed的调用比page_referenced更频繁，这意味着页面经常被访问。如果该页位于INACTIVE链表，将其移动到ACTIVE，此外还会设置PG_active标志位，清除PG_referenced；

（5）反向的转移也是有可能的，在页面活动程度减少时，可能连续调用两次page_referenced而中间没有mark_page_accessed。

如果对内存页的访问是稳定的，那么对page_referenced和mark_page_accessed的调用在本质上是均衡的，因而页面保持在当前LRU链表。这种方案同时确保了内存页不会再ACTIVE与INACTIVE链表间快速跳跃。

**2.2 slab****缓存回收**

slab缓存回收相对比较灵活，所有注册到shrinker_list中的方法都会被执行。

内核默认针对每个文件系统都注册了prune_super方法，这个函数用来回收文件系统中不再使用的dentry和inode缓存；

android的lowmemorykiller机制注册了选择性杀死进程的方法，回收进程使用的内存。

**3****怎样回收页框**

![](http://www.wowotech.net/content/uploadfile/201511/10fb1447331995.png)

其中shrink_page_list是真正回收页面的过程

![](http://www.wowotech.net/content/uploadfile/201511/09dd1447331995.png)

**4****周期性回收的频率**

**4.1 kswapd**

kswapd是内核为每个内存node创建的内存回收线程，为什么有了紧缺回收机制还需要周期性回收呢？因为有些内存分配是不允许阻塞等待回收的，比如中断和异常处理程序中的内存分配；还有些内存分配不允许激活I/O访问的。只有少数情况的内存紧缺可以完整执行回收过程，所以利用系统空闲时间回收内存非常必要。

该函数记录了上一次均衡操作时所用的分配order，如果kswapd_max_order大于上一次的值，或者classzone_idx小于上一次的值，则调用balance_pgdat再次均衡该内存域，否则可以进行短暂休眠，休眠的时间是HZ/10，对于arm(HZ=100)来说，休眠的时间就是1ms。

balance_pgdat均衡操作直到该内存域的zone_wartermark_ok为止。

**4.2 cache_reap**

cache_reap用来回收slab中的空闲对象，如果空闲对象可以还原成一个页面，则释放回buddy system。每次调用cache_reap会把所有的slab_caches遍历一遍，之后休眠2*HZ，对于arm(HZ=100)来说，周期就是20ms。

**5** **参考文献**

(1)《understanding the linux kernel》

(2)《professional linux kernel architecture》

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [进化论、人工智能和外星人](http://www.wowotech.net/tech_discuss/toe_ai_et.html) | [linux cpufreq framework(5)_ARM big Little driver](http://www.wowotech.net/pm_subsystem/arm_big_little_driver.html)»

**评论：**

**Second2None**  
2021-03-30 17:40

page_referenced通过PTE中一个硬件标志位来判断页面的访问频率。对于KVM虚拟机访问某个页面，如果此时对应的ept页表项已经存在，TLB脱靶，访问的是gpte和spte，并没有用到宿主机的pte。page_referenced有考虑这种情况吗？KVM虚拟机页面回收还有第二次机会吗？宿主机是如何得知虚拟机访问过这个物理页面被访问过的呢？

[回复](http://www.wowotech.net/memory_management/233.html#comment-8212)

**[bsp](http://www.wowotech.net/)**  
2021-03-31 09:41

@Second2None：arm64的虚拟化中有stage2映射，mmu可以认为也有两级；  
虚拟机通过VA得到IPA（你说的gpte/spte/pte）后，宿主机中的stage2页表负责IPA到PA（同理有另外一套页表map机制），host可以感知到IPA的PA的访问，同理加个硬件标志位 即可。

[回复](http://www.wowotech.net/memory_management/233.html#comment-8213)

**Second2None**  
2021-04-06 11:03

@bsp：是的，host硬件可以感知到IPA到PA的访问过程。但是，对于一个虚拟机的页面，在宿主机上是存在两个对应的页表项的，s2pte：IPA到PA的映射，pte：HVA到PA的映射，我看源码好像并没有判断某个页面是虚拟机使用的还是普通应用在使用。如果是虚拟机在使用那么page_referenced应该去判断stage2页表，如果是普通应用在使用,那么应该去判断stage1页表。

[回复](http://www.wowotech.net/memory_management/233.html#comment-8217)

**firo**  
2018-09-10 23:32

我查了4.19的代码, page_referenced逻辑没变, 依然适用.

[回复](http://www.wowotech.net/memory_management/233.html#comment-6944)

**tigerbear**  
2018-07-19 19:38

请教个问题，  
1. 你提到kswapd会周期性的回收，并且正常情况下回收的周期是100ms。但是我代码中（kswapd_try_to_sleep）看到似乎是kswapd会先睡眠100ms，如果睡眠过程中没有被唤醒的话就永久进入睡眠，直到下一次被唤醒。这是否与你说的周期性回收相矛盾呢？  
2. 我这边碰到一个问题，就是有时候会有突发性的GFP_ATOMIC标志的大量分配内存的需求，虽然每次分配只要一个页面，但会一下子要求分配很多。这种情况下我经常看到即使kswapd在工作，似乎回收速度也赶不上分配的速度，因此会造成分配失败。这个有什么办法能改善么？谢谢！

[回复](http://www.wowotech.net/memory_management/233.html#comment-6850)

**zhenhua**  
2018-06-04 15:50

貌似现在page_referenced不会做ClearPageReferenced的动作，而是在page_check_references去做  
而且现在inactive -> active 的转移除了mark_page_accessed，在shrink_page_list也有体现。  
所以上面那张图貌似是老了，期待更新哈

[回复](http://www.wowotech.net/memory_management/233.html#comment-6780)

**tiger**  
2018-03-26 16:43

没太明白page_referenced是在什么时候调用的？是每次回收页面的时候遍历这些可回收的页面时候调用的么？

[回复](http://www.wowotech.net/memory_management/233.html#comment-6623)

**[bsp](http://www.wowotech.net/)**  
2018-06-05 11:38

@tiger：有一处是在shrink_active_list中调用的，也就是内存不够时 先尝试从inactive中回收，但是inactive也不够 再调用shrink_active_list去move_active_pages_to_lru。

[回复](http://www.wowotech.net/memory_management/233.html#comment-6782)

**linux小白**  
2017-07-07 11:25

@itrocker  之后休眠2*HZ，对于arm(HZ=100)来说，周期就是20ms。 休眠的时间是HZ/10，对于arm(HZ=100)来说，休眠的时间就是1ms。  
我理解HZ ARM中定义是100,  相当于1S内产生100个时钟中断，jiffies = 1S/100 = 10, 所以一个jiffes代表10ms， ＨＺ是100个jiffes 等于1s，所以2HZ=2Ｓ，  但看您的计算HZ=1Ｓ/100 = 10MS,  难道我之前理解都错了么。。。请帮忙再普及下HZ，JIFFIES概念，谢谢~

[回复](http://www.wowotech.net/memory_management/233.html#comment-5784)

**davidlamb**  
2019-08-08 10:29

@linux小白：我觉得你的理解是正确的。

[回复](http://www.wowotech.net/memory_management/233.html#comment-7576)

**linux小白**  
2017-07-07 11:17

休眠的时间是HZ/10，对于arm(HZ=100)来说，休眠的时间就是1ms。  
应该是10ms吧

[回复](http://www.wowotech.net/memory_management/233.html#comment-5783)

**[bsp](http://www.wowotech.net/)**  
2018-06-05 11:33

@linux小白：休眠的时间是HZ/10，对于arm(HZ=100)来说，休眠的时间就是100ms。  
1HZ是1s，HZ/10当然是100ms。  
而且不管对于arm还是x86，HZ/10都是100ms。

[回复](http://www.wowotech.net/memory_management/233.html#comment-6781)

**zhaoge.zhang**  
2019-05-26 10:21

@linux小白：休眠的时间是HZ/10，对于arm(HZ=100)来说，休眠的时间就是1ms。  
1.对于arm(HZ=100)，所以可以得出100 * HZ = 1 S，所以 HZ = 10 ms  
2.休眠的时间是HZ/10，所以休眠时间 = HZ / 10 = 10 ms / 10 = 1 ms

[回复](http://www.wowotech.net/memory_management/233.html#comment-7440)

**keiviw**  
2016-05-12 10:05

您好 有个关于dentrycache和inodecache的内存回收问题请教您，在内存回收分配的dentry对象时，回收条件是什么？回收时针对的是memory_area还是page

[回复](http://www.wowotech.net/memory_management/233.html#comment-3925)

**[wowo](http://www.wowotech.net/)**  
2016-05-13 09:18

@keiviw：文件系统有关的回收操作，是由vm触发的（可参考shrink_slab-->do_shrinker_shrink-->shrinker->shrink），但具体的回收动作，是由文件系统负责的（可参考fs/super.c中的prune_super接口）。  
也就是说，decache和inode回收时机，是文件系统自己的事情，和vm无关。而具体的回收行为，我也没有研究过，你可以试试自己看一下。

[回复](http://www.wowotech.net/memory_management/233.html#comment-3926)

**keiviw**  
2016-05-14 11:17

@wowo：您看的源码是什么版本？我的理解是shrink_dcache_memory回收dentry对象到slab分配池中，而您介绍的cache_reap是针对slab3链表的空链表回收页帧

[回复](http://www.wowotech.net/memory_management/233.html#comment-3935)

**[wowo](http://www.wowotech.net/)**  
2016-05-16 09:00

@keiviw：我看的内核版本是linux-4.0.5，里面已经没有你说的shrink_dcache_memory接口了。

[回复](http://www.wowotech.net/memory_management/233.html#comment-3939)

**[andy01011501](http://www.wowotech.net/)**  
2016-03-09 11:42

我想保持一个普通文件在内存中的cache不被置换，也就是永久保留在内存中，有没有什么好办法啊？

[回复](http://www.wowotech.net/memory_management/233.html#comment-3621)

**[itrocker](http://www.wowotech.net/)**  
2016-03-11 14:06

@andy01011501：可以使用mlock机制

[回复](http://www.wowotech.net/memory_management/233.html#comment-3649)

**[andy01011501](http://www.wowotech.net/)**  
2016-03-15 15:21

@itrocker：对，非常感谢

[回复](http://www.wowotech.net/memory_management/233.html#comment-3666)

**[ericzhou](http://www.wowotech.net/)**  
2016-03-05 10:10

你好，请教一个问题。  
当回收一个页框时，怎么知道这个页框是否在被使用呢？  
如果用户程序映射到这个页框，并正在使用，是不是就不能回收？

[回复](http://www.wowotech.net/memory_management/233.html#comment-3589)

**[郭健](http://www.wowotech.net/)**  
2016-03-06 21:50

@ericzhou：你好，请教一个问题。  
当回收一个页框时，怎么知道这个页框是否在被使用呢？  
-------------------------------------------  
struct page中有reference counter，可以用来确定该page是否被引用。  
  
如果用户程序映射到这个页框，并正在使用，是不是就不能回收？  
-------------------------------------------  
如果用户程序的正文段mapping到了某个物理page上来，实际上，在内存比较紧俏的时候，仍然会回收该page，只不过，但该用户程序再次执行的时候，访问正文段产生异常，然后再次分配page并从backup的磁盘中加载该page的数据，返回异常的现场。

[回复](http://www.wowotech.net/memory_management/233.html#comment-3595)

**[qkhhyga](http://www.wowotech.net/)**  
2017-04-19 19:55

@郭健：Hi,linuxer  
但该用户程序再次执行的时候，访问正文段产生异常，然后再次分配page并从backup的磁盘中加载该page的数据，返回异常的现场。  
》》》我想问问，在回收page时，会把page 的内存backup到磁盘吗？

[回复](http://www.wowotech.net/memory_management/233.html#comment-5475)

**[linuxer](http://www.wowotech.net/)**  
2017-04-21 10:41

@qkhhyga：正文段是read only的，不会被修改，因此不需要讲page上的内容backup到磁盘。

[回复](http://www.wowotech.net/memory_management/233.html#comment-5478)

**[qkhhyga](http://www.wowotech.net/)**  
2017-04-27 19:56

@linuxer：Ｈi，linux,应用层的函数栈在该函数末退出时，这种情况的栈page 是否会被回收？又是如何来判断这种page 是否可以被回收呢？  
别外，我们都知道，应用程序在堆分配出来的内存属于匿名页面，这种页面又是如何设置page的count和mapcount参数的？

[回复](http://www.wowotech.net/memory_management/233.html#comment-5491)

**[linuxer](http://www.wowotech.net/)**  
2017-04-28 08:43

@qkhhyga：栈上的page属于匿名映射，只要内核配置了swap on，那么所有匿名page都会拥有了backup file或者backup device，只要有了backup file或者backup device，一般情况下（没有锁住）都可以被回收。  
  
mapcount参数是和地址映射相关的，在为该page创建新的地址映射关系的时候，会累加mapcount。  
  
count是真正的，通用意义上的reference counter。

[回复](http://www.wowotech.net/memory_management/233.html#comment-5494)

**[qkhhyga](http://www.wowotech.net/)**  
2017-04-28 14:11

@linuxer：Hi,linuxer  
   非常感谢你的回复  
　　count 在什么时候加一，减一又是什么时候？初始时，count是多少？当上层应用在执行free 的时候是否会对count 进行减一操作？

**[linuxer](http://www.wowotech.net/)**  
2017-04-28 16:35

@linuxer：一个物理页面（page frame）是由struct page来抽象，其成员_count表示了该page frame被使用的次数。刚开始，一个物理页面不被任何进程使用，_count等于-1，一旦被分配给了某个进程（我们称之A进程），例如保存了data段的数据，那么该page frame的_count等于0。在进程fork的时候，新进程B产生了，由于COW的原因，两个进程的地址空间都指向了一个page frame，那么这时候，该page frame的_count就会加一。当B进程向data段写入的时候，会产生memory fault，然后会为B进程分配新的page frame，从而使得A和B不再共享一个page frame，那么旧的page的_count就会减一。  
  
应用调用memory和free的时候，分配的都是虚拟地址，和物理页帧没有任何的关系

**[qkhhyga](http://www.wowotech.net/)**  
2017-04-28 18:00

@linuxer：我在看shrink_page_list 的时候，发现只有在page—>_count 等于０时才会被释放，这里有个疑问，既然还有进程引用这个page，那么为什么还会释放？  
当然shrink_page_list是在shrink_inactive_list 中调用的，是否在内存回收时并不在乎_count 的值，如果等于１或者２，只要try_to_unmap就可以了？

1 [2](http://www.wowotech.net/memory_management/233.html/comment-page-2#comments)

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
    
    - [Linux I2C framework(3)_I2C consumer](http://www.wowotech.net/comm/i2c_consumer.html)
    - [Linux2.6.23 ：sleepable RCU的实现](http://www.wowotech.net/kernel_synchronization/linux2-6-23-RCU.html)
    - [CFS调度器（1）-基本原理](http://www.wowotech.net/process_management/447.html)
    - [Linux电源管理(14)_从设备驱动的角度看电源管理](http://www.wowotech.net/pm_subsystem/device_driver_pm.html)
    - [快讯：蓝牙5.0发布（新特性速览）](http://www.wowotech.net/bluetooth/bluetooth_5_0_overview.html)
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
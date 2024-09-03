
奇小葩 Linux阅码场

 _2021年12月22日 10:14_

![图片](https://mmbiz.qpic.cn/mmbiz_png/nOQDlo8iaKME6rViaWWN1QI31mvoEeGscGBJsQwZoDI5efUdwichQoCicg0ReJNAa0ZhehGofR4a9o2oI16TVW7rEA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

_**原文作者：奇小葩**_

_**原文链接：https://blog.csdn.net/u012489236/article/details/120587124**_

  

  

在linux操作系统中，当内存充足的时候，内核会尽量使用内存作为文件缓存(page cache)，从而提高系统的性能。例如page cache缓冲硬盘中的内容，dcache、icache缓存文件系统的数据，这些内容是为了提升性能而设计的，还可以再次从硬盘中重新读取来构建对象，这部分内容可以在内存紧张的时候可以直接释放。  

  

所以内存回收在Linux内存管理中占据非常重要的地位，系统的内存毕竟是有限的，跑的进程成百上千，系统的内存越来越小，必须提供内存回收的机制，以满足别的任务的需求。在内存回收的过程中，会遇到以下问题

  

- 有哪些内存可以回收
    
- 什么时候回收，就需要了解回收解决什么问题？回收内存的策略是如何的
    
- 回收内存时，如何尽可能的减小对系统的性能的影响
    

  

**1 内存回收的目标**

对于内核并不是所有的物理内存都可以参与回收，比如内核的代码段，如果被内核回收了，系统就无法正常运行了，所以一般内核代码段、数据段、内核申请的内存、内核线程占用的内存等都是不可以回收的，除此之外的内存都可以是我们要回收的目标。

  

- _内核空间是所有进程公用的，内核中使用的页通常是伴随整个系统运行周期的，频繁的页换入和换出是非常影响性能的，所以内核中的页基本上不能回收，不是技术上实现不了而是这样做得不偿失。_
    
- _同时，另外一种是应用程序主动申请锁定的页，它的实时性要求比较高，频繁的换入换出和缺页异常处理无法满足它对于时间上的要求，所以这部分程序可能使用mlock api将页主动锁定，不允许它进行回收。_
    

  

我们下面对用户空间中的页面和内核空间中的页面给出进一步的分类讨论。可以把用户空间中的页面按其内容和性质分为以下几种：

- 进程映像所占的页面，包括进程的代码段、数据段、堆栈段以及动态分配的“存储堆，这部分进程的代码段和数据段所占用的内存页面是可以被换入换出的
    
- 通过系统调用mmap()把文件的内容映射到用户空间，这些页面所使用的交换区就是被映射的文件本身
    
- 进程间共享内存区，其页面的换入换出，只是整个过程比较复杂
    

  

除此之外，内核在执行过程中使用的页面要经过动态分配，但永驻内存，此类页面根据其内容和性质可以分为两类：

- 内核调用kmalloc()或vmalloc()为内核中临时使用的数据结构而分配的页，使用完毕后于是立即释放。但是，由于一个页面中存放有多个同种类型的数据结构，所以要到整个页面都空闲时才把该页面释放。
    
- 内核中通过调用alloc_pages()，为某些临时使用和管理目的而分配的页面，例如，每个进程的内核栈所占的两个页面、从内核空间复制参数时所使用的页面等等。这些页面也是一旦使用完毕便无保存价值，所以立即释放。
    

同时在内核中还有一种页面，虽然使用完毕，但其内容仍有保存价值，因此，并不立即释放。这类页面“释放”之后进入一个LRU队列，经过一段时间的缓冲让其“老 化”。如果在此期间又要用到其内容了，就又将其投入使用，否则便继续让其老化，直到条件不再允许时才加以回收。这部分主要是通过slab机制申请的内存，这种用途的内核页面大致有以下这些：

  

- 文件系统中用来缓冲存储一些文件目录结构dentry的空间
    
- 文件系统中用来缓冲存储一些索引节点inode的空间
    
- 用于文件系统读／写操作的缓冲区
    

按照以上所述，对于内存回收，大致可以分为以下两类：

- **文件映射的页**，包括page cache、slab中的dcache、icache、用户进程的可执行程序的代码段，文件映射页面。
    
- **匿名页**，括进程使用各种api（malloc,mmap,brk/sbrk）申请到的物理内存(这些api通常只是申请虚拟地址，真实的页分配发生在page fault中)，包括堆、栈，进程间通信中的共享内存，pipe，bss段，数据段，tmpfs的页。这部分没有办法直接回写，为他们创建swap区域，这些页也转化成了文件映射的页，可以回写到磁盘。
    

其中page cache包括文件系统的page，还包括块设备的buffer cache，万物皆文件，block也是一种文件，它也有关联的file、inode等。另外根据页是否是脏的，在回收的时候处理有所不同，脏页需要先回写到磁盘再回收，干净的页可以直接释放。

  

所以对于内存管理，我们回收的目的主要是基于用户空间进行回收，其主要回收的策略如下：

- 用户空间内存:原则上应该都可以参与内存回收，除非那些被进程锁定的页
    
- 内核空间内存:一般内核代码段，数据段，内核kmalloc()/vmalloc()出来的内存，内核线程占用的内存等都是不可以回收的，除此之外的内存都是我们要回收，所以大致为磁盘高速缓存(如索引节点，目录项高速缓存），页面高速缓存(访问文件时系统生成的页面cache)，mmap()文件时所用的有名映射所使用的物理内存
    

  

**2 内存回收策略**

内核之所以要进行内存回收，主要原因有两个：

  

- 内核需要为任何时刻突发到来的内存申请提供足够的内存，以便cache的使用和其他相关内存的使用不至于让系统的剩余内存长期处于很少的状态。
    
    _内核使用内存中的page cache对部分文件进行缓存，以便提升文件的读写效率。所以内核有必要设计一个周期性回收内存的机制，以便cache的使用和其他相关内存的使用不至于让系统的剩余内存长期处于很少的状态。_
    
- 当真的有大于空闲内存的申请到来的时候，会触发强制内存回收。
    

  

所以内核针对这两种回收的需求，分别实现了两种不同的机制。

  

- 针对第①种，Linux系统设计了kswapd后台程序，当内核分配物理页面时，由于系统内存短缺，没法在低水位情况下分配内存，因此会唤醒kswapd内核线程来异步回收内存
    
- 针对第②种，Linux系统会触发直接内存回收(direct reclaim)，在内核调用页分配函数分配物理页面时，由于系统内存短缺，不能满足分配请求，内核就会直接触发页面回收机制，尝试回收内存来解决问题
    

这两种回收的触发方式不同，其区别如下图所示

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**3 kswapd初始化**

kswpad本身是内核线程，它和调用者的关系是异步的，如test进程尝试调用alloc_pages()来分配内存，当发现在低水位情况下无法分配出内存时，它唤醒kswapd内核线程。这时，kswapd内核线程就开始执行页面回收工作了。同时test进程会尝试其他的方法来分配内存，如调用直接内存回收。

  

Linux内核中有一个非常重要的内核线程kswapda，它负责在内存不足的情况下回收页面。kswapd内核线程初始化时会为系统每个NUMA内存结点创建一个名为kswap%d的内核线程。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

- swap_setup函数根据物理内存大小设定全局变量page_cluster，当megs小于16时候，page_cluster为2，否则为3
    
    _page_cluster为每次swap in或者swap out操作多少内存页 为2的指数，当为0的时候，为1页，为1的时候，2页，2的时候4页，通过/proc/sys/vm/page-cluster 查看_
    
- 然后通过for_each_node_state遍历所有 节点，kswapd_run中kthread_run为每个节点创建一 个kswapd%d线程
    

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

在NUMA系统中，每个内存结点都通过一个pg_data_t数据结构来描述物理内存的布局，pg_data_t数据结构定义在include/linux/mmzone.h中，kswapd传递的参数就是pg_data_t数据结构

  

typedef struct pglist_data {

struct zone node_zones[MAX_NR_ZONES];

struct zonelist node_zonelists[MAX_ZONELISTS];

int nr_zones;

...

int node_id;

wait_queue_head_t kswapd_wait;

wait_queue_head_t pfmemalloc_wait;

struct task_struct *kswapd; /* Protected by

   mem_hotplug_begin/end() */

int kswapd_order;

enum zone_type kswapd_classzone_idx;

...

}

  

和kswapd相关的参数有kswapd_order，kswapd_wait和kswapd_classzone_idx等。kswapd_wait是一个等待队列，每个pg_data_t数据结构都有一个等待队列，它是在free_area_init_core函数中初始化的。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

- Kswapd的主循环是一个死循环，首先设置kswapd内核线程的进程描述符的标志位，只有当kthread _should_stop的时候才会break跳出循环体，会kswapd_try_to_sleep中睡眠，并让出CPU控制权。当系统内存紧张时，这时内存分配函数会调
    
- wakeup_kswapd()来唤醒kswapd内核线程，此时kswapd内核线程在kswapd_try_to_sleep函数中被唤醒，然后调用balance_pgdat()函数来回收页面。
    
- _PF_MEMALLOC：用于内存分配，一般在直接内存压缩，直接内存回收和kswapd中设置，这些场景下可能会有少量的内存分配行为，因此设置PF_MEMALLOC标志位，表示允许他们使用系统预留的内存，不用考虑zone水位问题_
    
- _PF_SWAPWRITE：允许写交换分区_
    
- _PF_KSWAPD：表明这是一个kswapd内核线程_
    

下面重点是看看kswapd_try_to_sleep，kswapd_try_to_sleep用于判断kswapd线程是否sleep，该函数是内核线程kswapd睡眠时让出cpu控制权的地方，同时也是睡眠kswapd被唤醒时进入的地方

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

- 通过prepare_kswapd_sleep判断kswap是否可以睡眠
    
- 可以睡眠则先尝试睡眠0.1s
    
- 如果中途没有被唤醒，说明kswap可以睡眠，让出CPU，schedule出去
    
- 如果中途被唤醒则返回上层函数，执行内存回收
    

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

- 唤醒所有等待内存回收的进程
    
- 如果此前kswapd回收失败达到16次了，没必要再唤醒kswapd
    
- 如果此时刚好有zone能满足high_wmark水位，那也没必要唤醒kswapd了
    
    对于该过程reclaim_order和classzone_idx用于判断当前节点是能否进入sleep状态，
    
- 若当前节点中0到classzone_idx中每个zone区域在分配了reclaim_order阶内存后空闲内存值都高于high水线阈值则当前节点的kswapd线程可进入sleep状态(空闲内存要减去该zone的为classzone_idx保留的内存)
    
- 若0到classzone_idx中有一个zone区域在分配了reclaim_order阶内存后空闲内存值低于了high水线阈值则当前节点的kswapd线程不能进入sleep状态
    

  

前面介绍了，kswapd内核线程初始化会在kswapd_try_to_sleep函数中睡眠，当内核线程被唤醒后，会调用最关键的balance_pgdat()来回收页面。首先，在kswapd回收内存过程中有一个扫描控制结构体，用于控制这个回收过程。既然是回收内存，就需要明确要回收多少内存，在哪里回收，以及回收时的操作权限等，我们看下这个控制结构struct scan_control主要的一些变量，

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

scan_control数据结构用于控制页面回收的参数，在balance_pgdat中会使用到这些参数，那么重点关注这个函数

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

主要流程：

  

- 定义kswap进程的内存回收控制结构，允许umap和swap操作，使用sc.priority为12优先级进行页面扫描
    
- 判断buffer_head缓存是否太多，太多就从最高内存域开始回收
    
- 判断是否有zone能满足此次内存分配，有则此次kswap回收可以停止
    
- 如果非活动匿名页太少，对匿名active链表做老化处理
    
- 会调用mem_cgroup_soft_limit_reclaim函数。该函数的目的是回收该zone上超过soft_limit最多的mem_cgroup在该zone上mem_cgroup_per_zone对应的lru链表，这个在cgroup限制回收中单独学习
    
- 调用kswapd_shrink_node开始回收内存
    
- 回收完毕判断是否需要唤醒等待内存的进程
    
- 此次kswap没有回收到页面，失败次数加1，达到16次就放弃
    

  

**4 kswapd_shrink_node**

pgdat_balanced()检查这个内存节点中是否有合格的zone，遍历这个内存节点中可用的zone的顺序为从最低zone到classzone_idx指向的zone，classzone_idx通常是页分配器传递过来的参数，然后调用zone_watermark_ok_safe函数来检查这个zone的水位释放高于WMARK_HIGH水位并且是否可以分配出2的order次幂个连续的物理页面。如果不够，就调用kswapd_shrink_node函数用于扫描内存节点中所有可回收的页面。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

- 计算high_wmark水位值，回收页面数量要大于该值
    
- 对node进行内存回收，包括LRU链表和slab缓存
    
- 如果已经回收到两倍order大小的内存，设置检测内存阈值检测odrer为0阶，避免过度回收
    

  

**5 shrink_node**

shrink_node函数用于扫面内存节点中所有可用于回收的页面，其主要的操作如下：

- do_while循环的判断条件为should_continue_reclaim，通过这一轮中回收页面的数量和扫面页面的数量来判断是否需要继续扫面
    
- 首先遍历mem_cgroup，调用shrink_node_memcg回收页面
    
- shrink_slab函数调用内存管理系统的shrinker接口，很多子系统会注册shrinker接口来回收slab对象
    
- vmpressure函数通过计算nr_scanned/nr_reclaimed比例来判断内存压力
    

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**6 shrink_node_memcg**

shrink_node_memcg函数使基于内存节点的页面回收函数，它被kswapd内核线程和直接页面回收机制调用，其有4个参数

- pgdat表示页面回收的内存节点
    
- memcg表示要页面回收的memory cgroup
    
- sc是页面回收的控制参数
    
- lru_pages表示已经扫面的页面数量
    

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

shrink_node_memcg函数中，调用了get_scan_count函数之后，获取到了扫描页面的信息后，就开始进入主题对LRU链表进行扫描处理了。它会对匿名页和文件页做平衡处理，选择更合适的页面来进行回收。当回收的页面超过了目标页面数后，将停止对文件页和匿名页两者间LRU页面数少的那一方的扫描，并调整对页面数多的另一方的扫描速度。最后，如果不活跃页面少于活跃页面，则需要将活跃页面迁移到不活跃页面链表中。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**7 get_scan_count**

内存回收最后会调到shrink_node, 从代码可以看出回收的对象就两种

- 一种是以lru list组织的用户可见的page, 包括文件的page cache, 进程的heap, stack等
    
- 另外一种是内核自己使用slab，shrink_slab系统中能提供内存回收功能的slab用户都会通过register_shrinker注册自己的内存回收函数
    

get_scan_count函数，它根据swapiness和priority优先级计算4个LRU链表中需要扫描的页面的个数，结果保存到nr数组中，函数原型如下：

  

static void get_scan_count(struct lruvec *lruvec, struct mem_cgroup *memcg,

   struct scan_control *sc, unsigned long *nr,

   unsigned long *lru_pages)

- nr[0]: 存放要扫描的不活跃的匿名页面个数
    
- nr[1]: 存放要扫描的活跃的匿名页面个数
    
- nr[2]: 存放要扫描的不活跃的文件页面个数
    
- nr[3]: 存放要扫描的活跃的文件页面个数
    

这个函数用于获取针对文件页和匿名页的扫描页面数。这个函数决定内存回收每次扫描多少页，匿名页和文件页分别是多少，比例如何分配等。

在函数的执行过程中，根据四种扫描平衡的方法标签来最终选择计算方式，四种扫描平衡标签如下：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

对于下面重点关注zone_reclaim_stat，其中匿名页面存放在数组[0]中，文件缓存页存放在数组[1]中。举个例子，如果recent_rotated[1]/recent_scanned[1]越小，说明LRU中的文件页面价值较小，那么更应该多扫描一些文件页面，尽量把没有价值的文件页面释放掉。根据公式，文件页面的recent_rotated越小，fp值越大，那么最后扫描的scan_file需要扫描的文件页面数量也就越大。也可以理解为：在扫描总量一定的情况下，扫描文件页面的比重更大。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在get_scan_count()函数中会计算每个lru链表需要扫描的页框数量，然后将它们保存到nr数组中，在此，有两个因素会影响这4个lru链表需要扫描的数量，一个是sc->priority(扫描优先级)，一个是swapiness。

  

- **sc->priority：**影响的是这4个lru链表扫描页框数量的基准值，当sc->priority越小，每个lru链表需要扫描的页框数量就越多，当sc->priority为0时，则本次会对每个lru链表都完全扫描一遍。在不同内存回收过程中，使用的sc->priority不同，而sc->priority默认值为12。
    
- **swapiness：**影响的是在基准值的基础上，是否做调整，让系统更多地去扫描文件页lru链表，或者更多地去扫描匿名页lru链表。当swapiness为100时，扫描文件页lru链表与扫描匿名页lru链表是平衡的，并不倾向与谁，也就是它们需要扫描的页框就是就是sc->priority决定的基准值，当swapiness为0,时，就不会去扫描匿名页lru链表，只扫描文件页lru链表。
    

  

**8 shrink_list**

计算好每个lru链表需要扫描的页框数量后，就以活动匿名页lru链表、非活动匿名页lru链表、活动文件页lru链表、非活动文件页lru链表的顺序对每个链表进行一次最多32个页框的扫描，然后将对应的nr数组的数值进行减少，当对这4个lru链表都进行过一次扫描后，判断是否回收到了足够页框，如果没有回收到足够页框，则继续扫描，而如果已经回收到了足够页框的话，并且nr数组中的数还有剩余的情况下，那么就进入到最重要的回收函数中，根据LRU链表的类型来调用不同的处理函数

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**9 inactive_list_is_low**

该函数的目的是判断传入的lru链表类型对应的不活跃lru链表上的页框数量是否过低，返回true表明过低。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

系统总是希望不活跃的匿名页面数量应该保持比较少，因为这样保证系统没有太多的工作可做，可以让页面回收的工作变得少。同时也希望而不活跃的文件映射页面也应该相对少一些，这样可以在活跃的LRU链表中预留更多的内存，更多的page cache，更高效的文件读取工作。

但是站在内存回收的角度，系统页不希望不活跃的lru链表的页面数量尽可能多一些，这样可以在回收页面前更多的页面得以有第二次被访问到的机会，得到重生的机会。

为了平衡上述的关系，linux系统提出了一个不活跃比例(inactive_ratio)，以此让同类型lru链在活跃页面数量和不活跃页面数量的比例达到平衡状态。

在某类型的lru链表，设置活跃页面数量为active，不活跃页面数量为inactive，不活跃比例为inactive_ratio

- _若inactive * inactive_ratio < active 该类型lru链表不活跃页面数量过低_
    
- _若inactive * inactive_ratio = active 理想状态_
    
- _若inactive * inactive_ratio > active 该类型lru链表活跃页面数量过低_
    

对于匿名页面，不活跃的比例为

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**10 shrink_active_list**

shrink_active_list用于扫描活跃LRU链表，包括匿名页面或文件映射页面，把最近一直没有人访问的页面添加到不活跃的LRU链表中。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**11 shrink_inactive_list**

对于shrink_inactive_list函数扫描不活跃LRU链表以尝试回收页面，并且返回已经回收的页面数量。其大致的过程如下：

  

- 定义一个临时链表page_list，并且通过too_many_isolated最初如下判断
    
     _当前页面回收是kswapd还是直接页面回收者(direct reclaimer)_
    
     _已经分离的页面数量是否大于不活跃的页面数量_
    
- 调用isolate_lru_pages以分离页面到临时链表page_list
    
- 调用shrink_page_list来扫描页面并回收页面，nr_reclaimed表示成功回收的页面数量
    

我们关注最重要的函数shrink_page_list，它决定在zone->inactive_list中的页面最后是否能被回收释放掉。这个函数的处理流程总结如下

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

对于页面回收过程中，我们会遇到很多意想不到的情况，如大量脏页、大量正在回写的页面堵塞了块设备的I/O通道等问题，这些问题都会严重的影响页面回收的机制，甚至用户的体验。为了捕获这些信息，页面回收机制使用以下方式

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

对于上面的页面，内核有几个标志位在内核节点pglist_datea的flags成员中，如下所示

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

- PGDAT_WRITEBACK，表示正在回收时发现大量正在回写页面，kswapd内核线程就应该跳过该页面，但是对于直接内存回收来说，需要等待这个页面回收完成
    

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

- PGDAT_CONGESTED表示系统中有大量页面堵塞在块设备的I/O操作上，应对措施是让系统等待一段时间。在扫面内存节点的页面时，每次扫面完一轮，需要判断当前是否设置了PGDAT_CONGESTED标志位。若直接页面回收者发现系统有大量回写页面堵塞，那么调用wait_iff_congested函数让页面等待一会儿
    

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- PGDAT_DIRTY表示发现LRU链表中有大量的葬爷，对于匿名页面的葬爷，都会调用pageout函数回写脏页，对于文件映射的脏页，就需要分为两种情况
    
- _对于kswapd内核线程，不管是否有大量脏页，都会调用pageout函数回写脏页_
    
- _对于直接内存回收者，只有发现大量葬爷，即设置了PGDAT_DIRTY标志位，才会调用pageout函数回写脏页，否则就直接跳过该页面_
    

  

**12 总结**

页面回收，并不是回收得越多越好，而是力求达到一种balanced。因为页面回收总是以cache丢弃、内存swap等为代价的，对系统 性能会有一定程度的影响。而balanced，就是既要保证性能，又要应付好新来的页面分配请求。而kswapd也是以此为原则进行内存的回收，本章大致梳理了下kswapd的初始化和内存回收的流程，至于具体的LRU链表的类型来调用不同的处理放到后面章节中进行学习，本章主要理解kswapd流程，参考网上一张图，其处理流程如下图所示

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**13 参考文档**

_[内核内存] [arm64] 内存回收4—shrink_node函数详解_

_https://blog.csdn.net/u010923083/article/details/116278456?utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EOPENSEARCH%7Edefault-10.no_search_link&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EOPENSEARCH%7Edefault-10.no_search_link_

_奔跑吧linux内核_

![](https://res.wx.qq.com/op_res/NN_GToMiIjsXzgPzF9-74ZzwR3cA9-fv3o9eWo8f5gQWqx71CmGlY8kFxuIxZaG0TB1bFeMCmh1DGN_pWMRg0A)

**帐号已迁移**

2篇原创内容

公众号

  

阅读 2926

​

写留言

**留言 1**

- Multilin
    
    2021年12月29日
    
    赞
    
    文章有一处错误：shrink_list扫描顺序应该是以非活动匿名页lru链表、活动匿名页lru链表、非活动文件页lru链表、活动文件页lru链表的顺序对每个链表进行一次最多32个页框的扫描。而且是先减少nr数组的值再执行shrink_list。
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/YHBSoNHqDiaH5V6ycE2TlM9McXhAbaV9aUBaTAibVhLibXH1rL4JTMb2eoQfdCicTm4wycOsopO82jG3szvYF3mTJw/300?wx_fmt=png&wxfrom=18)

Linux阅码场

1067

1

写留言

**留言 1**

- Multilin
    
    2021年12月29日
    
    赞
    
    文章有一处错误：shrink_list扫描顺序应该是以非活动匿名页lru链表、活动匿名页lru链表、非活动文件页lru链表、活动文件页lru链表的顺序对每个链表进行一次最多32个页框的扫描。而且是先减少nr数组的值再执行shrink_list。
    

已无更多数据
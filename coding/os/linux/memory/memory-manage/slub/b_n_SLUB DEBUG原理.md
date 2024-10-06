作者：[smcdef](http://www.wowotech.net/author/531) 发布于：2018-2-22 21:49 分类：[内存管理](http://www.wowotech.net/sort/memory_management)
# 1. 前言

在工作中，经常会遇到由于越界导致的各种奇怪的问题。为什么越界访问导致的问题很奇怪呢？在工作差不多半年的时间里我就遇到了很多越界访问导致的问题（不得不吐槽下IC厂商提供的driver，总是隐藏着bug）。比如说越界访问导致的死机问题，这种问题的出现一般需要长时间测试才能发现，而且发现的时候即使有panic log。你也没什么头绪。这是为什么呢？假设驱动A通过kmalloc()申请了一段内存，不注意越界改写了与其相邻的object的数据（经过我之前一篇SLUB的文章分析，你应该明白kmalloc基于kmem_cache实现的），假设被改写的object是B驱动使用的，巧合B驱动使用object存储的是地址数据，如果B驱动访问这个地址。那么完了，B驱动死了，panic也是怪B驱动。试想一下，这块被改写的object是哪个驱动使用，是不是哪个驱动就倒霉了？并且每一次死机的log中panic极有可能发生在不同的模块。但是真正的元凶却是A驱动，他没事你还不知道，是不是很恐怖？简直是借刀杀人啊！当然，越界访问也不一定会死机。之前就遇到一个很奇怪的问题。有两个全局数组变量（用作存储字符串）分别被模块C和D使用。这两个数组是上层需要显示的name信息。当C和D模块都工作的时候，发现C模块的name显示不对，但是D模块的name显示正常。将D模块remove，发现C模块的name显示正确。当时看了下System.map文件，发现这两个全局数组变量分配的内存是在一起的，由于D模块越界写导致的。而这种情况就不会死机。但是当你遇到这种情况的时候，你很惊讶，怎么会这样？两个模块之间根本就没关系啊！如果完全不借助检测工具去查找问题是相当费时间的。而且有可能还没什么头绪。这种问题我们该怎么定位？因此我们遇到一种debug的手段，可以检测out-of-bounds（oob）问题。刚才的第一种情况就可以SLUB自带debug功能。针对第二种情况就需要借助更加强大的KASAN工具（后续会有文章介绍）。  
因此，我们需要一种debug手段帮助我们定位问题。SLUB DEBUG就是其中的一种。但是SLUB DEBUG仅仅针对从slub分配器分配的内存，如果你需要检测从栈中或者数据区分配内存的问题，就不行了。当然了，你可以选择KASAN。本文主要关注SLUB DEBUG的原理，如何定位这些问题的。  
SLUB DEBUG检测oob问题原理也很简单，既然为了发现是否越界，那么就在分配出去的内存尾部添加一段额外的内存，填充特殊数字（magic num）。我们只需要检测这块额外的内存的数据是否被修改就可以知道是否发生了oob情况。而这段额外的内存就叫做Redzone。直译过来“红色区域”是不是有种神圣不可侵犯的感觉。  
说明：slab是最早加入linux的,在那时只有slab的存在。随着时间的推移slub出现了，slub是在slab基础上进行的改进，在大型机上表现出色。而slob是针对小型系统设计的。由于slub实现的接口和slab接口保持一致（虽然你用的是slub分配器，但是很多函数名称和数据结构还是依然和slab一致），所以有时候用slab来统称slab, slub和slob。slab, slub和slob仅仅是分配内存策略不同。管理的思想基本一致。本篇文章中说的是slub分配器debug原理。但是针对分配器管理的内存，下文统称为slab缓存池。所以文章中slub和slab会混用，表示同一个意思。  
注：文章代码分析基于linux-4.15.0-rc3。图片有点走形，请单独点开图片查看。 
# 2. SLUB DEBUG功能 

SLUB DEBUG可以检测内存越界（out-of-bounds）和访问已经释放的内存（use-after-free）等问题。  
## 2.1. 如何打开功能  
重新配置kernel选项，打开如下选项即可。  
```cpp
CONFIG_SLUB=y  
CONFIG_SLUB_DEBUG=y  
CONFIG_SLUB_DEBUG_ON=y  
```
## 2.2. 如何使用  
程序中的bug如果想用SLUB DEBUG去检测，还需要slabinfo命令。因为，SLUB内存检测功能在某些情况下不能立刻检测出来，必须主动触发，因此我们需要借助slabinfo命令触发SLUB allocator检测功能。和KASAN相比较而言，这也是SLUB DEBUG的一个劣势。毕竟KASAN可以做到在越界问题出现时就报出问题。  
slabinfo工具源码位于tools/vm目录。可以使用如下命令编译slabinfo工具（针对ARM64 architecture）。  
aarch64-linux-gnu-gcc -o slabinfo slabinfo.c  
当系统开机之后，就可以运行slabinfo –v命令触发SLUB allocator检测所有的object，并将log信息输出到syslog。接下来的任务就是查看log信息是否包含SLUB allocator输出的bug log。其实有些bug是不需要运行slabinfo命令即可捕捉，但是有些却必须使用slabinfo –v命令才可以。下一节将会介绍SLUB DEBUG的原理，为你揭开哪些bug不需要slabinfo命令。
# 3. object layout 

配置kernel选项CONFIG_SLUB_DEBUG_ON后，在创建kmem_cache的时候会传递很多flags（SLAB_CONSISTENCY_CHECKS、SLAB_RED_ZONE、SLAB_POISON、SLAB_STORE_USER）。针对这些flags，SLUB allocator管理的object对象的format将会发生变化。如下图所示。  
 [![12.png](http://www.wowotech.net/content/uploadfile/201802/9eb61519307945.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201802/9eb61519307945.png)  
SLUBU DEBUG关闭的情况下，free pointer是内嵌在object之中的，但是SLUB DEBUG打开之后，free pointer是在object之外，并且多了很多其他的内存，例如red zone、trace和red_left_pad等。这里之所以将FP后移就是因为为了检测use-after-free问题,当free object时会在将object填充magic num(0x6b)。如果不后移的话，岂不是破坏了object之间的单链表关系。  
## 3.1. Red zone有什么用  

从图中我们可以看到在object后面紧接着就是Red zone区域，那么Red zone有什么作用呢?既然紧随其后，自然是检测右边界越界访问（right out-of-bounds access）。原理很简单，在Red zone区域填充magic num，检查Red zone区域数据是否被修改即可知道是否发生right oob。  
可能你会想到如果越过Red zone，直接改写了FP，岂不是检测不到oob了，并且链表结构也被破坏了。其实在check_object()函数中会调用check_valid_pointer()来检查FP是否valid，如果invalid，同样会print error syslog。  
## 3.2. padding有什么用  

padding是sizeof(void ) bytes的填充区域，在分配slab缓存池时，会将所有的内存填充0x5a。同样在free/alloc object的时候作为检测的一种途径。如果padding区域的数据不是0x5a，就代表发生了“Object padding overwritten”问题。这也是有可能，越界跨度很大。  
## 3.3. red_left_pad有什么用

red_left_pad和Red zone的作用一致。都是为了检测oob。区别就是Red zone检测right oob，而red_left_pad是检测left oob。如果仅仅看到上面图片中object layout。你可能会好奇，如果发生left oob，那么应该是前一个object的red_left_pad区域被改写，而不是当前object的red_left_pad。如果你注意到这个问题，还是很机智的，这都被你发现了。为了避免这种情况的发生，SLUB allocator在初始化slab缓存池的时候会做一个转换。  
 [![13.png](http://www.wowotech.net/content/uploadfile/201802/c00b1519307827.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201802/c00b1519307827.png)  
如果你去追踪kmem_cache_create()，在calculate_sizes()中布局object。区域划分的layout就如同你看到上图的上半部分。当我第一次看到这段代码的时候，我也这么认为。实际上却不是这样的。在struct page结构中有一个freelist指针，freelist会指向第一个available object。在构建object之间的单链表的时候，object首地址实际上都会加上一个red_left_pad的偏移，这样实际的layout就如同图片中转换之后的layout。为什么会这样呢？因为在有SLUB DEBUG功能的时候，并没有检测left oob功能。这种转换是后续一个补丁的修改。补丁就是为了增加left oob检测功能。  
做了转换之后的red_left_pad就可以检测left oob。检测的方法和Red zone区域一样，填充的magic num也一样，差别只是检测的区域不一样而已。

# 4. SLUB DEBUG原理 

经过上一节分析应该很清楚了大概的原理了。从high level考虑，SLUB就是利用特殊区域填充特殊的magic num，在每一次alloc/free的时候检查magic num是否被意外修改。  
## 4.1. magic num  
SLUB 中有哪些magic num呢?所有使用的magic num都宏定义在include/linux/poison.h文件。
```cpp
1. #define SLUB_RED_INACTIVE   0xbb
2. #define SLUB_RED_ACTIVE     0xcc
3. /* ...and for poisoning */
4. #define POISON_INUSE         0x5a    /* for use-uninitialised poisoning */
5. #define POISON_FREE          0x6b    /* for use-after-free poisoning */
6. #define POISON_END           0xa5    /* end-byte of poisoning */
```
SLUB_RED_INACTIVE和SLUB_RED_ACTIVE用来填充Red zone和red_left_pad，目的是检测oob。POISON_INUSE用来填充padding区域，同样可以用来检测oob，只不过是poison overwrite。POISON_FREE作用是检测use-after-free问题。POISON_END是object可用区域的最后一个字节填充。  
4.2. slab缓存池填充  
当SLUB allocator申请一块内存作为slab 缓存池的时候，会将整块内存填充POISON_INUSE。如下图所示。  
 [![14.png](http://www.wowotech.net/content/uploadfile/201802/7b6f1519307828.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201802/7b6f1519307828.png)  
然后通过init_object()函数将相关的区域填充成free object的情况，并且建立单链表。注意freelist指针指向的位置，SLUB_DEBUG on和off的情况下是不一样的。主要就是3.3节提到的转换关系。为什么这里填充成free object的情况呢？其实就是为了假装我这里都是free的object，也是符合情理的。object初始化流程如下。  
 [![15.png](http://www.wowotech.net/content/uploadfile/201802/d6421519307830.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201802/d6421519307830.png)  
4.3. free object layout  
刚分配slab缓存池和free object之后，系统都会通过调用init_object()函数初始化object各个区域，主要是填充magic num。free object layout如下图所示。  
 [![16.png](http://www.wowotech.net/content/uploadfile/201802/1e411519307832.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201802/1e411519307832.png)  
1) red_left_pad和Red zone填充了SLUB_RED_INACTIVE（0xbb）；  
2) object填充了POISON_FREE（0x6b），但是最后一个byte填充POISON_END（0xa5）；  
3) padding在allocate_slab的时候就已经被填充POISON_INUSE（0x5a），如果程序意外改变，当检测到padding被改变的时候，会output error syslog并继续填充0x5a。  
4.4. alloc object layout  
当从SLUB allocator申请一个object时，系统同样会调用init_object()初始化成想要的模样。alloc object layout如下图所示。  
 [![17.png](http://www.wowotech.net/content/uploadfile/201802/c9ba1519307834.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201802/c9ba1519307834.png)  
1) red_left_pad和Red zone填充了SLUB_RED_ACTIVE（0xcc）；  
2) object填充了POISON_FREE（0x6b），但是最后一个byte填充POISON_END（0xa5）；  
3) padding在allocate_slab的时候就已经被填充POISON_INUSE（0x5a），如果程序意外改变，当检测到padding被改变的时候，会output error syslog并继续填充0x5a。  
alloc object layout和free object layout相比较而言，也仅仅是red_left_pad和Red zone的不同。既然该填充的数据都搞定了，下面就是如何检查oob、use-after-free等问题了。  
4.5. out-of-bounds bugs detect  
下面使用demo例程来说明oob检测。我们使用kmalloc分配32 bytes内存，然后制造越界访问第33个元素，必然会越界访问。由于kmalloc是基于SLUB allocator，因此此bug可以检测。  
```cpp
1. void right_oob(void)
2. {
3.     char *p = kmalloc(32, GFP_KERNEL);
4.     if (!p)
5.         return;
6.     p[32] = 0x88;
7.     kfree(p);
8. }
```
运行后的object layout如下图所示。  
 [![18.png](http://www.wowotech.net/content/uploadfile/201802/88391519307837.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201802/88391519307837.png)  
我们可以看到，Red zone区域本来应该0xcc的地方被修改成了0x88。很明显这是一个Redzone overwritten问题。那么系统什么时候会检测到这个严重的bug呢？就在你kfree()之后。kfree()中会去检测释放的object中各个区域的值是否valid。Red zone区域的值全是0xcc就是valid，因此这里会检测0x88不是0xcc，进而输出error syslog。kfree()最终会调用free_consistency_checks()检测object。free_consistency_checks()函数如下。  

1. static inline int free_consistency_checks(struct kmem_cache *s,
2.         struct page *page, void *object, unsigned long addr)
3. {
4.     if (!check_valid_pointer(s, page, object)) {
5.         slab_err(s, page, "Invalid object pointer 0x%p", object);
6.         return 0;
7.     }
8.     if (on_freelist(s, page, object)) {
9.         object_err(s, page, object, "Object already free");
10.         return 0;
11.     }
12.     if (!check_object(s, page, object, SLUB_RED_ACTIVE))
13.         return 0;
14.     return 1;
15. }

1) check_valid_pointer()负责检测object的free pointer指针数据是否valid。oob是有可能导致这种情况得到发生。  
2) on_freelist()检测object是否已经free，可以检测多次free的bug。  
3) check_object()会检测Red zone区域的数值是否被改变，因此这里就会报出bug。  
如果是左边界越界访问，是否也同样可以检测出呢？可以测试以下demo例程。  

1. void left_oob(void)
2. {
3.     char *p = kmalloc(32, GFP_KERNEL);
4.     if (!p)
5.         return;
6.     p[-1] = 0x88;
7.     kfree(p);
8. }

运行后的object layout如下图所示。  
 [![19.png](http://www.wowotech.net/content/uploadfile/201802/ba6b1519307842.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201802/ba6b1519307842.png)  
检测方法大同小异，这里也是最终在free_consistency_checks()函数中通过检测red_left_pad区域发现left oob问题。  
可能你会想如果我只申请内存不释放的话，这个bug还能检测到吗？其实这里是不行的。我们只能借助slabinfo工具主动触发检测功能。所以，这也是SLUB DEBUG的一个劣势，它不能做到动态监测。它的检测机制是被动的。  
4.6. use-after-free bugs detect  
如果是use-after-free问题，我们该如何检测呢？首先上demo例程。  

1. void use_after_free(void)
2. {
3.     char *p = kmalloc(32, GFP_KERNEL);
4.     if (!p)
5.         return;

7.     kfree(p);
8.     memset(p, 0x88, 32);
9. }

运行之后object layout如下图所示。  
 [![20.png](http://www.wowotech.net/content/uploadfile/201802/079f1519307843.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201802/079f1519307843.png)  
还记得上面说的吗？SLUB DEBUG是被动的。因此这里就要选择slabinfo工具了。命令中断输入slabinfo –v即可。slabinfo检测的原理也很简单，便利所有已经释放的object，检查object区域是否全是0x6b（最后一个字节oxa5）即可，如果不是的话，自然就是use-after-free。

5. slabinfo 

我们看一下slabinfo –v命令的实现方式以及检查的流程。slabinfo源码位于tools/vm/slabinfo.c文件。slabinfo –v命令执行流程如下图所示。  
 [![21.png](http://www.wowotech.net/content/uploadfile/201802/71341519307845.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201802/71341519307845.png)  
针对系统中每一个slab都会执行set_obj()函数。set_obj()代码如下：

1. static void set_obj(struct slabinfo *s, const char *name, int n)
2. {
3.     char x[100];
4.     FILE *f;
5.     snprintf(x, 100, "%s/%s", s->name, name);
6.     f = fopen(x, "w");
7.     if (!f)
8.         fatal("Cannot write to %s\n", x);
9.     fprintf(f, "%d\n", n);
10.     fclose(f);
11. }

set_obj()参数name传递的是“validate”，n传递的是1。作用就是向/sys/kernel/slab/<slab name>/ validate节点写“1”触发slab检测功能。根据validate节点找到写入口函数validate_store()。调用函数执行流如下图所示。  
 [![22.png](http://www.wowotech.net/content/uploadfile/201802/75c11519307847.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201802/75c11519307847.png)  
validate_slab()代码如下：  

1. static int validate_slab(struct kmem_cache *s, struct page *page, unsigned long *map)
2. {
3.     void *p;
4.     void *addr = page_address(page);
5.     if (!check_slab(s, page) ||
6.             !on_freelist(s, page, NULL))
7.         return 0;
8.     /* Now we know that a valid freelist exists */
9.     bitmap_zero(map, page->objects);
10.     get_map(s, page, map);
11.     for_each_object(p, s, addr, page->objects) {
12.         if (test_bit(slab_index(p, s, addr), map))
13.             if (!check_object(s, page, p, SLUB_RED_INACTIVE))
14.                 return 0;
15.     }
16.     for_each_object(p, s, addr, page->objects)
17.         if (!test_bit(slab_index(p, s, addr), map))
18.             if (!check_object(s, page, p, SLUB_RED_ACTIVE))
19.                 return 0;
20.     return 1;
21. }

1) check_slab()会调用slab_pad_check()检查slab padding区域。slab padding和object里面的pading不是一回事。如果说从buddy system分配的页按照SLUB规则平分成很多object，那么有可能不能整除，那么剩下的unused区域就是slab padding。valid的数值是0x5a。如下图所示。  
2) get_map()利用bitmap标记所有的available object。例如，slab缓存池一共有10个对象，按地址大小排序标号0-9（相当于index）。假设5和8号object已经被分配出去。那么bitmap中除了bit5和bit8以为，其余位为1。  
3) 第一个for循环遍历所有的available object是否有oob、use-after-free、object padding overwritten等问题发生。  
4) 第二个for循环遍历所有已经分配出去的object是否发生oob问题。

 [![23.png](http://www.wowotech.net/content/uploadfile/201802/7ae51519307847.png "点击查看原图")](http://www.wowotech.net/content/uploadfile/201802/7ae51519307847.png)

  

_原创文章，转发请注明出处。蜗窝科技，www.wowotech.net。_

  

标签: [slub](http://www.wowotech.net/tag/slub) [内存管理](http://www.wowotech.net/tag/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [致驱动工程师的一封信](http://www.wowotech.net/device_model/429.html) | [图解slub](http://www.wowotech.net/memory_management/426.html)»

**评论：**

**卷心菜**  
2022-12-01 17:45

请问红区的size是多大？

[回复](http://www.wowotech.net/memory_management/427.html#comment-8712)

**benjoÿing**  
2020-06-08 10:54

赞！图形解析很清楚！

[回复](http://www.wowotech.net/memory_management/427.html#comment-8015)

**Dongwei**  
2019-06-26 19:57

您好，  
   free掉的内存，再使用就会提示use_after_free,既然如此，是不是意味着该块内存被浪费了，如果不是被浪费，那么内核需要做些什么事情，来证明这段内存又恢复了可用性。

[回复](http://www.wowotech.net/memory_management/427.html#comment-7491)

**[smcdef](http://www.wowotech.net/)**  
2019-07-09 15:52

@Dongwei：free的内存，其他用户可以继续申请，申请后就可以使用了。不存在浪费现象。我们编程的规范流程是：  
1. 申请内存  
2. 使用内存  
3. 释放内存  
  
凡是已经释放的，一律不能使用，如果需要继续使用，必须重复以上3个步骤。当然，刚刚释放的内存，现在申请有可能还是同一块内存。所以，这块内存没有浪费。而是在重复利用。

[回复](http://www.wowotech.net/memory_management/427.html#comment-7521)

**Dongwei**  
2019-06-26 19:53

您好，我想问一下，内核中，什么情况下会使用slab呢，一般的数据结构都会用到slab么？

[回复](http://www.wowotech.net/memory_management/427.html#comment-7490)

**[smcdef](http://www.wowotech.net/)**  
2019-07-09 15:49

@Dongwei：驱动以及内核代码大量的调用kmalloc()接口，kmalloc()就是对slab接口的封装，所以内存分配大部分都是使用slab。当申请内存超过2页，就是走buddy system。

[回复](http://www.wowotech.net/memory_management/427.html#comment-7520)

**bob**  
2018-05-16 12:27

这种转换是后续一个补丁的修改。补丁就是为了增加left oob检测功能。请问这个补丁什么时候加入的，我这边4.4.15， 用slabinfo -v check left oob的信息，感觉还是red_left_pad是在尾部的

[回复](http://www.wowotech.net/memory_management/427.html#comment-6749)

**[smcdef](http://www.wowotech.net/)**  
2018-05-17 22:17

@bob：commit d86bd1bece6fc41d59253002db5441fe960a37f6  
Author: Joonsoo Kim <iamjoonsoo.kim@lge.com>  
Date:   Tue Mar 15 14:55:12 2016 -0700  
  
    mm/slub: support left redzone

[回复](http://www.wowotech.net/memory_management/427.html#comment-6752)

**[smcdef](http://www.wowotech.net/)**  
2018-05-17 22:19

@bob：注意这个函数的使用，你就明白了  
void *fixup_red_left(struct kmem_cache *s, void *p)  
{  
    if (kmem_cache_debug(s) && s->flags & SLAB_RED_ZONE)  
        p += s->red_left_pad;  
  
    return p;  
}

[回复](http://www.wowotech.net/memory_management/427.html#comment-6753)

**hai**  
2018-05-11 15:24

请问一下，打开slub_debug_on一定会在free object的时候将object标记为"6b"吗？

[回复](http://www.wowotech.net/memory_management/427.html#comment-6744)

**[smcdef](http://www.wowotech.net/)**  
2018-05-15 21:47

@hai：是啊！可以看下这个函数init_object()

[回复](http://www.wowotech.net/memory_management/427.html#comment-6748)

**hai**  
2018-05-17 09:53

@smcdef：这里的if条件并不一致，而且查看代码__OBJECT_POISON成立需要s->ctor == 0,碰到一个slab(proc_inode_cache),它的s->ctor ！= 0，free过后object仍然遗留数据，不知道这个是特例还是什么？  
  
static void init_object(struct kmem_cache *s, void *object, u8 val)  
{  
    u8 *p = object;  
  
    if (s->flags & SLAB_RED_ZONE)  
        memset(p - s->red_left_pad, val, s->red_left_pad);  
  
    if (s->flags & __OBJECT_POISON) {  
        memset(p, POISON_FREE, s->object_size - 1);  
        p[s->object_size - 1] = POISON_END;  
    }  
  
    if (s->flags & SLAB_RED_ZONE)  
        memset(p + s->object_size, val, s->inuse - s->object_size);  
}  
  
    if ((flags & SLAB_POISON) && !(flags & SLAB_DESTROY_BY_RCU) &&  
            !s->ctor)  
        s->flags |= __OBJECT_POISON;

[回复](http://www.wowotech.net/memory_management/427.html#comment-6750)

**[smcdef](http://www.wowotech.net/)**  
2018-05-17 22:13

@hai：通用slab（kmalloc的kmem_cache）s->ctor是NULL，会填充“6b”。你说的slab就是用户自定义的，因此不会填充“6b”。slab debug更多的意图是检测通过kmalloc接口申请的内存。

[回复](http://www.wowotech.net/memory_management/427.html#comment-6751)

**rafal**  
2018-03-30 14:18

碰到内存被踩的问题真是头疼，费了好大的力气查到某段地址被改，结果问题来了，who干的？似乎没有好的方法，先看看是不是被踩地址附近的对象越界踩的，如果不是，麻烦就来了，楼至有什么好方法吗？

[回复](http://www.wowotech.net/memory_management/427.html#comment-6629)

**[smcdef](http://www.wowotech.net/)**  
2018-04-01 21:21

@rafal：踩踏问题不是很容易解决的，可以尝试一下文章所说的工具，当然还有kasan工具。也是不错的。kasan的文章可以参考另一篇文章。

[回复](http://www.wowotech.net/memory_management/427.html#comment-6632)

**rafal**  
2018-04-02 10:45

@smcdef：正在看你写的kasan原理，真的太好了，看来这个工具可以在踩踏前报bug,强大，为楼主的付出点赞。

[回复](http://www.wowotech.net/memory_management/427.html#comment-6633)

**rafal**  
2018-03-30 14:07

拜读，佩服楼主的文档输出，尤其插图，直观易懂。

[回复](http://www.wowotech.net/memory_management/427.html#comment-6628)

**[linuxer](http://www.wowotech.net/)**  
2018-02-23 08:23

宋少侠文档产量很高啊！一次两篇，哈哈～～～

[回复](http://www.wowotech.net/memory_management/427.html#comment-6560)

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
    
    - [进程切换分析（2）：TLB处理](http://www.wowotech.net/process_management/context-switch-tlb.html)
    - [/proc/meminfo分析（一）](http://www.wowotech.net/memory_management/meminfo_1.html)
    - [串口通信技术浅析](http://www.wowotech.net/basic_tech/serial_intro.html)
    - [SLUB DEBUG原理](http://www.wowotech.net/memory_management/427.html)
    - [支持wowo，有这样一块小天地享受技术，感觉棒棒哒！](http://www.wowotech.net/147.html)
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
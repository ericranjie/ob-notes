# 

Linux阅码场

 _2021年09月29日 09:04_

The following article is from Linux内核远航者 Author Linux内核远航者

[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM6TOGoqicdx0ebU9icGjViaHIGsVEF4ofJPmCWY4pFxBPNaw/0)

**Linux内核远航者**.

探索Linux内核的奥秘，涉及处理器架构，内存管理，进程管理，容器和虚拟化等底层干货分享，当沉浸在庞大的内核源代码中难免会迷失自己，但参透某个子系统的设计思想时又会欣喜若狂，Linux内核远航者乘风破浪，在探索的道路上唯有热爱可抵岁月漫长。

](https://mp.weixin.qq.com/s?__biz=Mzg2OTc0ODAzMw==&mid=2247502772&idx=1&sn=6421e35ae01844d2d405eff6fc402bae&source=41&key=daf9bdc5abc4e8d0a0a2ad6eed4ed1a13b043ec3718f49704bf3ac6d5bd258857daff0d603683be12d708a9283f3a1c04638682e317b0906c23551ae91c4a06be02ac3dd6e1625559f08d64d1dc0fc5f5440c2c611b3c6bda506b60ec939e6158a4b17821d20b47e5f5860fd88bf75b79cdb783953decc568447ed0f17c7f18b&ascene=14&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=iMac+MacBookAir10%2C1+OSX+OSX+14.6.1+build(23G93)&version=13080710&nettype=WIFI&lang=en&session_us=gh_89d7b88c9033&countrycode=CN&fontScale=100&exportkey=n_ChQIAhIQqaqw0STgliQ1JLlTg1etYhKUAgIE97dBBAEAAAAAAPIzBhVi2Q0AAAAOpnltbLcz9gKNyK89dVj0AkE%2BiUpnFLSuLsA5Je5TSWx%2F3jxXKhLV2itU0qKNmUsfwAwFSE3mcest2QYDYg9MtbzSGVpNWdcJ236Q6oZ7OQvnHxPO7lZpEKbRmYiR4sLoa3ZQlloyKcQw1LEzjRzREnawS5rSxnz00V9qmTzC5lHdamCwPkIGM6CwyjH6eexXV7tRQZn%2BwLAXQ6cAUe0yjPMBwKNC0A%2B%2Fq%2FdtPZW31Ut8DqfTCtyRBkgYpjOS9neSr9TdbFYIfCbwAm5Vcjz%2F4Xo5eHE9R%2FIp91a7RICMzgATxOOsD9P%2BIJxUmIA7tacyv8KvZ%2BPfNrPpk%2Feokw%3D%3D&acctmode=0&pass_ticket=TrnVfrQqmBxi2hB%2BlDOlWzc%2FGcbrhv02sZSqWBz%2Bm1zTRBRCbnJtAMWQd0rwd5Bz&wx_header=0#)

首先祝大家中秋节快乐，阖家欢乐，节日之余记得学习哟！

Linux中有后备文件支持的页称为文件页，如属于进程的代码段、数据段的页，内存回收的时候这些页面只需要做脏页的同步即可（干净的页面可以直接丢弃掉）。反之为匿名页，如进程的堆栈使用的页，内存回收的时候这些页面不能简单的丢弃掉，需要交换到交换分区或交换文件。本文中，主要分析匿名页的访问将发生哪些可能颠覆我们认知的"化学反应"。

# 1.实例代码

首先以一个简单的示例代码来说明：

```
#include <stdio.h>#include <stdlib.h>#include <unistd.h>#include <string.h>#include <sys/mman.h>#define MAP_SIZE (100 * 1024 * 1024)int main(int argc, char *argv[]){ char *p; char val; int i; puts("before mmap ok, pleace exec 'free -m'!"); sleep(5); //mmap p = mmap(NULL, MAP_SIZE, PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0); if(p == NULL) {  perror("fail to malloc");  return -1; }  puts("after mmap ok, pleace exec 'free -m'!"); sleep(5); //read for (i = 0; i < MAP_SIZE; i++) {  val = p[i]; } puts("read ok, pleace exec 'free -m'!"); sleep(5);#if 1 //write memset(p, 0x55, MAP_SIZE); puts("write ok, pleace exec 'free -m'!");#endif //sleep pause(); return 0;}
```

代码非常简单：首先通过mmap分配100M的私有可读可写匿名页面，然后进行读写访问，分别在提示的时候在另外一个窗口执行free -m命令查看输出结果。

程序执行结果如下：

```
$ ./anon_rw_demobefore mmap ok, pleace exec 'free -m'!after mmap ok, pleace exec 'free -m'!read ok, pleace exec 'free -m'!write ok, pleace exec 'free -m'!
```

命令执行结果如下：

```
$ free -m              总计         已用        空闲      共享    缓冲/缓存    可用内存：       15729        8286        1945         895        5497        6220交换：       16290        1599       14691$ free -m              总计         已用        空闲      共享    缓冲/缓存    可用内存：       15729        8286        1945         895        5497        6220交换：       16290        1599       14691$ free -m              总计         已用        空闲      共享    缓冲/缓存    可用内存：       15729        8286        1945         895        5497        6220交换：       16290        1599       14691$ free -m              总计         已用        空闲      共享    缓冲/缓存    可用内存：       15729        8383        1848         895        5497        6123交换：       16290        1599       14691
```

可以看到：

第一次提示执行free命令的时候，我们还没有开始通过mmap分配内存，此时free命令输出作为参考。

第二次提示执行free命令的时候，我们已经通过mmap分配了100M的内存，此时发现free命令输出内存消耗基本没有变化。

第三次提示执行free命令的时候，我们对于分配的匿名页面进行了读操作，此时发现free命令输出内存消耗页基本没有变化, **这基本上会颠覆我们的认知**。

第四次提示执行free命令的时候，我们对于分配的匿名页面进行了写操作，此时发现free命令输出内存消耗大概为100M。

# 2.内核原理

下面我们从Linux内核的层面来解析发生以上神奇现象的原理。

## 2.1 mmap的内存消耗

mmap申请匿名页的时候，只是申请了虚拟内存（通过vm_area_struct结构来描述，如描述虚拟内存区域的地址范围、访问权限等，以下简称vma），实际的物理内存并没有申请（除了用于管理虚拟内存区域的vma等结构内存的申请），当前虚拟内存和物理内存并没有建立页表映射关系，而真正的申请的匿名页所对应的物理页在实际访问的时候按需分配获得，所以此时我们看不到内存的消耗情况。

## 2.2 第一次读匿名页的内存消耗

通过mmap申请完虚拟内存之后，进程就可以按照之前申请vma的访问权限进行访问，第一发生读访问，这个时候由于虚拟内存和物理内存并没有建立页表映射关系，通过虚拟地址并不能查找到物理内存，所以会发生处理器的异常，最终分析是因为数据访问异常导致，就由处理器架构相关的代码进入了我们通用的缺页异常处理例程中。

缺页异常调用链如下：

```
"mm/memory.c"处理器架构相关异常处理代码-> handle_mm_fault    -> __handle_mm_fault        -> handle_pte_fault            ->  if (!vmf->pte) {   ------------------- 1                     if (vma_is_anonymous(vmf->vma))  ------------------- 2                             return do_anonymous_page(vmf);   ------------------- 3
```

缺页异常进入handle_pte_fault后，在1标签代码处，来判断访问的虚拟内存页的页表项是否为空，为空说明这个这个虚拟页没有和物理页建立映射关系。然后在2标签代码处判断是否为匿名页缺页异常（实际上是判断是否为私有的匿名页，当前当前示例代码场景申请的为私有匿名页面）。在3标签代码处，进行真正的私有匿名页缺页异常处理。

下面主要看下第一次读匿名页的处理：

```
do_anonymous_page->pte_alloc(vma->vm_mm, vmf->pmd)   ------------------- 1->/* Use the zero-page for reads */if (!(vmf->flags & FAULT_FLAG_WRITE) &&     ------------------- 2                !mm_forbids_zeropage(vma->vm_mm)) { ------------------- 3        entry = pte_mkspecial(pfn_pte(my_zero_pfn(vmf->address),                                        vma->vm_page_prot));  ------------------- 4        vmf->pte = pte_offset_map_lock(vma->vm_mm, vmf->pmd,                        vmf->address, &vmf->ptl);  ------------------- 5               ...        goto setpte;}->  page = alloc_zeroed_user_highpage_movable(vma, vmf->address); ------------------- 6-> entry = mk_pte(page, vma->vm_page_prot);  ------------------- 7 entry = pte_sw_mkyoung(entry); ------------------- 8 if (vma->vm_flags & VM_WRITE)         entry = pte_mkwrite(pte_mkdirty(entry));  ------------------- 9 vmf->pte = pte_offset_map_lock(vma->vm_mm, vmf->pmd, vmf->address,                 &vmf->ptl); ------------------- 10                  ->set_pte_at(vma->vm_mm, vmf->address, vmf->pte, entry);  ------------------- 11
```

1标签处：判断虚拟地址对应的pmd表项是否为空，为空来分配直接页表设置到pmd表项中。

2标签处：判断是否是进行读访问。

3标签处：判断是否没有禁止0页。

4标签处：就是对于没有禁止0页的匿名页读访问设置页表，这里通过0页的页帧号和mmap映射时指定的访问权限组合页表项的值。

5标签处：通过发生缺页的虚拟地址来计算出页表项的地址保存在 vmf->pte。

最11标签处：将4标签初组合出的页表项的值写入到5标签初计算出的页表项中。

以上分析可知：对于私有的匿名页，第一次读访问的时候都会发生缺页异常，然后通过页表映射0页，这个0页没有什么特殊之处，只不过它是在系统启动过程中初始化好的一块内容全为0的页面，这样做可以为进程分配了内存只进行读访问节省大量物理内存。

## 2.3 第一次写匿名页的内存消耗

大家可以将示例代码中，读访问屏蔽掉只进行写访问，观察内存消耗。

这个时候发生缺页异常时，不会在走2 3 4 5 便签处代码，而在6处分配了一个物理页面，在7 8 9组合页表项的值， 10处计算出页表项的地址，最后把组合的值设置到页表项中。

需要注意第9处，如果是**写访问会设置页表项的可写标志位**。

以上分析可知：对于私有的匿名页，第一次写访问的时候都会发生缺页异常，会真正分配一个物理页面，然后将虚拟页面通过页面映射到物理页面，所以我们能观察到写之后发生了大量内存消耗。

## 2.4 第一次读然后写匿名页的内存消耗

这种场景就是示例代码中所做的实验，可以看到读的时候基本上没有内存消耗，写的时候发生了大量内存消耗。

关于第一次读，上面已经做过解释，下面主要看读完之后的页面发生写访问的情况。

## 2.4.1 从mmap说起

实际上，对于一个私有的内存映射，在mmap的时候为页表映射准备访问权限的时候并不是给予所有的权限，而是**把可写属性去掉**了。

我们可以从源代码找到答案：

```
"mm/mmap.c"do_mmap->mmap_region    ->vma_set_page_prot(vma)        ->vm_page_prot = vm_pgprot_modify(vma->vm_page_prot, vm_flags);  ---------1            ->pgprot_modify(oldprot, vm_get_page_prot(vm_flags))        ->WRITE_ONCE(vma->vm_page_prot, vm_page_prot);  ---------------2                 /* description of effects of mapping type and prot in current implementation.  * this is due to the limited x86 page protection hardware.  The expected  * behavior is in parens:  *  * map_type     prot  *              PROT_NONE       PROT_READ       PROT_WRITE      PROT_EXEC  * MAP_SHARED   r: (no) no      r: (yes) yes    r: (no) yes     r: (no) yes  *              w: (no) no      w: (no) no      w: (yes) yes    w: (no) no  *              x: (no) no      x: (no) yes     x: (no) yes     x: (yes) yes  *  * MAP_PRIVATE  r: (no) no      r: (yes) yes    r: (no) yes     r: (no) yes  *              w: (no) no      w: (no) no      w: (copy) copy  w: (no) no  *              x: (no) no      x: (no) yes     x: (no) yes     x: (yes) yes  */->vm_get_page_prot     pgprot_t protection_map[16] __ro_after_init = {        __P000, __P001, __P010, __P011, __P100, __P101, __P110, __P111,        __S000, __S001, __S010, __S011, __S100, __S101, __S110, __S111};
```

对于__Pxxx，  最后一个x表示vma属性是否可读，倒数第二个x表示vma属性是否可写，P后面的x表示是否可执行。

1标签处根据mmap传递的访问权限来构造最终的访问权限标识。

2标签处将构造好的访问权限标识记录到vma->vm_page_prot中，供缺页异常设置页表使用。

注释中已经做了详细的解释，具体页表属性如何表示由各自的处理器架构相关代码来做（eg: 对于x86架构 #define __P111  PAGE_COPY_EXEC），我们只需要知道：**无论我们想让vma具备那些属性组合，都会屏蔽掉写属性**，具体可以查看相关的处理器架构实现。

所以，再次回到缺页异常处理代码中。在2.2小节的4标签处，使用mmap设置好的页表访问权限设置页表属性，当前场景我们知道，mmap中指定为私有的可读可写属性，而页表中只是**设置为了只读属性**。

## 2.4.2 写时复制的触发

读访问将虚拟页以只读的方式映射到了0页，当再次发生写操作时，就会再次触数据访问异常，最终进入缺页异常处理例程中。

下面给出调用链：

```
"mm/memory.c"handle_pte_fault->if (vmf->flags & FAULT_FLAG_WRITE) {  -----------1        if (!pte_write(entry)) -----------2                return do_wp_page(vmf); -----------3
```

可以看到最终也是在handle_pte_fault中处理：在1标签处判断是否为写访问。在2标签处判断页表项的属性是否是只读。在3标签处进行实际的写时复制处理。

以上分析可知：发生写访问操作时，如果vma可写，但是页表属性标识不可写（只读），会发生写时复制缺页异常，对于当前场景的0页的写访问就是如此，在do_wp_page中会重新分配物理页面映射到虚拟页面，然后页表设置为可写属性，就完成了缺页处理。

# 3.总结

1）mmap分配私有匿名内存时，会设置vma的vm_page_prot成员，去除掉页表的写访问标识。

2）第一次读匿名页时，对于可读可写的vma，虚拟页会以只读的方式映射到0页。

3）第一次写匿名页时，对于可读可写的vma，会申请物理页面，虚拟页以可读可写的方式映射到此物理页。

4）第一次读匿名页后，然后写匿名页，先只读方式映射到0页，然后发生写时复制，分配物理页，虚拟页以可读可写的方式映射到此物理页。

可以发现，访问匿名页面时发生的“化学反应”并不是那么的简单，其中会涉及mmap的映射原则，0页的映射，匿名页面的处理，写时复制的处理等等，而且读写顺序不一样，产生的结果也会不一样，大家可以结合内核源代码进行分析，希望对大家理解匿名页缺页异常有所帮助。

  

对缺页异常感兴趣的小伙伴欢迎订阅视频课程：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

  

![](http://mmbiz.qpic.cn/mmbiz_png/ljlJkD7iaUC3zZW7k0blicNMumaBicacNCbQuAxwYWY6REDSI6yRqicpmNicDQO74ddzboHpxhvFvHzTRXfTJaGvib7g/300?wx_fmt=png&wxfrom=19)

**Linux内核远航者**

探索Linux内核的奥秘，涉及处理器架构，内存管理，进程管理，容器和虚拟化等底层干货分享，当沉浸在庞大的内核源代码中难免会迷失自己，但参透某个子系统的设计思想时又会欣喜若狂，Linux内核远航者乘风破浪，在探索的道路上唯有热爱可抵岁月漫长。

32篇原创内容

公众号

  

Reads 2060

​

Comment

**留言 1**

- 胡海鹏
    
    2021年9月29日
    
    Like
    
    mmap失败不是返回MAP_FAILED（-1）吗？不是返回NULL吧
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/YHBSoNHqDiaH5V6ycE2TlM9McXhAbaV9aUBaTAibVhLibXH1rL4JTMb2eoQfdCicTm4wycOsopO82jG3szvYF3mTJw/300?wx_fmt=png&wxfrom=18)

Linux阅码场

9Share2

1

Comment

**留言 1**

- 胡海鹏
    
    2021年9月29日
    
    Like
    
    mmap失败不是返回MAP_FAILED（-1）吗？不是返回NULL吧
    

已无更多数据
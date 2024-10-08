Linux阅码场
 _2022年03月08日 08:02_
以下文章来源于内核工匠 ，作者Cheetah

一般用户空间关联的物理页面是按需通过缺页异常的方式分配和调页，当系统物理内存不足时页面回收算法会回收一些最近很少使用的页面，但是有时候我们需要锁住一些物理页面防止其被回收（如时间有严格要求的应用），Linux中提供了mlock相关的系统调用供用户空间使用来锁住部分或全部的地址空间关联的物理页面。

本文的分析基于arm64处理器架构，内核版本为Linux-5.10.27，我们会结合重点内核源代码来解析mlock是如何做到锁住进程地址空间关联的物理内存的，又是如何防止相关的物理页面被交换出去的。
# 一、主动缺页

mlock的主要代码处理流程如下，这里我们主要关注主动缺页部分：

![[Pasted image 20240906121156.png]]

mlock处理路径中，会将VM_LOCKED标志加入到vma->vm_flags中（由于设置的地址区域有可能跨越多个vma,所以代码中会涉及到分裂和合并的操作，实质上都会设置相关的vma->vm_flags的VM_LOCKED标志），然后会调用__mm_populate来填充虚拟页对应的物理页，最终在faultin_page函数中试图查找vma中的每个虚拟页对应的物理页面（对应于follow_page_mask函数），如果没有找到会调用handle_mm_fault主动触发缺页处理。

handle_mm_fault函数是内核通用的缺页异常处理例程，如vma是匿名映射的则分配物理页面然后建立页表映射关系，vma是文件映射则会从磁盘读取对应的文件页（如果page cache没有对应页面时）到内存的page cache,然后建立虚拟页面建立页表映射关系。
# **二、内存回收处理
## 1. 扫描活跃的lru链表**

内存回收扫描活跃的lru链表时，对于设定了VM_LOCKED的vma处理链路如下：
  ![[Pasted image 20240906121015.png]]

可以看到：当扫描活跃的lru链表的时候，会通过反向映射机制查找到映射这个物理页面的每个vma, 对于设置了vma->vm_flags 的VM_LOCKED标志的vma来说直接退出反向映射处理即可，不需要进行访问计数的统计工作，本身这样的物理页面就需要常驻内存不要进行回收。
## **2. 扫描不活跃的lru链表**

内存回收扫描不活跃的lru链表时，对于设定了VM_LOCKED的vma处理链路如下：

  ![[Pasted image 20240906121023.png]]

可以看到：调用链中也会调用page_referenced 函数通过反向映射机制查找到映射这个物理页面的每个vma, 对于设置了vma->vm_flags 的VM_LOCKED标志的vma来说**直接退出反向映射处理**即可，返回到page_check_references函数时，判断如果有vma设置了VM_LOCKED标志就会返回PAGEREF_RECLAIM到shrink_page_list函数接着处理。

shrink_page_list函数在处理完page_check_references之后，就进行回收处理，对于页表映射页会调用try_to_unmap来解除页表映射。
## **3. 反向映射处理**

shrink_page_list在回收物理页面之前会调用try_to_unmap来解除映射到这个页面所有页表项，相关处理如下：
  ![[Pasted image 20240906121035.png]]

对于映射到这个物理页的每个vma来说，如果vma->vm_flags设置了VM_LOCKED标志，则会调用mlock_vma_page来做mlock处理，然后返回false，结束反向映射处理。

下面我们来看mlock_vma_page做了什么事情：
  ![[Pasted image 20240906121042.png]]

可以看到：mlock_vma_page首先设置页描述符的PG_mlocked标志，然后会zone的NR_MLOCK页面记账，然后会将页面从原来的lru链表中隔离出来，最后会将页面加入不可回收的lru中（这个代码大家自行阅读，实际上是判断页描述符的PG_mlocked标志）。

mlock_vma_page处理的重点就是将页面**加入到不可回收的lru链表**，这样内存回收的时候就不会在扫描到这样的页面了。

 mlock的整个过程如下图所示：
  ![[Pasted image 20240906121050.png]]
## **三、munlock处理**

munlock会解除原来锁住的页面，处理路径如下：

  ![[Pasted image 20240906121056.png]]

当然代码中也会有对应的vma的分裂处理，主要处理为：**清除vma的VM_LOCKED标志，清除页描述符的PG_mlocked标志，最后就会将原来在不可回收的lru中的页面重新加入对应的lru链表中**。

这里还有一个细节，那就是有可能这个页面对多个vma共享，所以会通过try_to_munlock来处理，处理路径如下：
  ![[Pasted image 20240906121104.png]]

会通过反向映射机制，遍历这样页对应的所有vma，如果传递的ttu_flags为TTU_MUNLOCK且vma->vm_flags没有设置VM_LOCKED标志，则直接返回，检查下一个vma；如果有一个vma设置了VM_LOCKED标志，说明这个页面还不能被回收，就会通过mlock_vma_page函数**重新将页面加入到不可回收的lru链表**。

munlock的整个处理过程如下图：
  ![[Pasted image 20240906121111.png]]
# 四、总结

对于一些对时间有严格要求的应用场景，访问时按需分配和调页机制的时延可能是未知的，内核中提供了mlock相关的系统调用，用于将虚拟内存区域对应的物理页面“锁在”内存中。内核对应mlock锁住的页面实际上它主要做了两步比较重要的操作：1，调用mlock的时候就将所需要的物理页面准备好；2，内存回收时当扫描到相关的物理页面时，将其放入不可回收的lru链表。第一步保证访问的虚拟地址对应的物理页面在内存中，第二步保证了锁住的页面不会被回收。
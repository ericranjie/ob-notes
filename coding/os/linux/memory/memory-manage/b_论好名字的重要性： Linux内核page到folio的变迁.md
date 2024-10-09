Original Barry OPPO内核工匠
_2023年06月30日 17:01_ _广东_

# **一、引子**

Once upon a time，Netscape的大拿 Phil Karlton曾经说过：“There are only two hard things in Computer Science: cache invalidation and naming things”，成为程序界流传甚广的名言，可见取名是计算机科学中最难的两件事之一。取名，要用名字恰到好处地描述其想描述的事物，要体现代码注释的最高原则——自注释，这其实一点都不轻松。

取名，一般都是从生僻的变为大众的，这样才能朗朗上口，为人民群众所喜闻乐见，比如陈港生更名为成龙，杨旎奥改名为杨紫，刘福荣改名为刘德华。而内核从page到folio的一次改变，似乎是反其道而行之了。感觉有相当数量的童鞋可能都不见得认识folio这个单词。金山词霸曾经曰过，folio是这个意思：

![Image](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjM6gt8rLM5X88WyrG4b8hcckHDXly9O8q0D921aVYtkoNt7523L105bia4eL2p6nlGdOpsoqssQMrA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

感觉大概意思，就是通过封面和封底夹在一起的一本书或者一套文献。这个名字，有点古典生僻，它的目标在于解决内核面临的一个纠结状况。至于这个名字叫folio、pageset、superpage还是head_page，其实都没有那么重要了，背后真正重要的是，它要解决什么问题。

# **二、乱局**

下面我们来看folio出现之前，Linux内核的情况。众所周知，在Linux内核中，我们用page来描述一页，这一页通常是4KB。这个世界如果所有人都是4KB的单页，那就简单归一了。但是，在晴朗的天空中，却漂浮中一朵乌云，这朵乌云就是compound page以及由compound page衍生出的hugepage，它们并非总是单页的。

在Linux中，我们并不总是以单一的4KB basepage为单位来获取、映射和释放内存。我们有时候，会把多个4KB复合在一起，进行申请、映射和释放：

## **1.用户态的透明大页（THP）和HugeTLB大页**

我们可能直接用PMD而不是PTE进行映射，把一个2MB的连续物理内存映射到用户态，这样用户态使用它，可以大量减小TLB miss。
!\[\[Pasted image 20240927115704.png\]\]

在内核中，我们描述内存的单元是page，这个page一般是4KB。但是在THP/HugeTLB的场景下，整个2MB其实是一个整体的概念，这个时候，我们诞生了一种需求：

- 有时候我们关心的是这个2MB的整体，但是page其实是描述它的4KB的部分，用page来描述整体似乎不太适合；
- 有时候，我们确实想描述2MB整个整体里面4KB的某个部分，这个时候page似乎比较适合。

## **2.内核态也可能直接申请和释放compound page**

比如一些内核driver会通过\_\_GFP_COMP标记申请order大于0的连续页，形成所谓的compound page（比如2页，4页，8页，16页等组成的复合页，前面的THP/HugeTLB其实也是一种order较大的compound page，由2MB/4KB个页面组成的compound page）。这样的透过\_\_GFP_COMP标记，来向buddy申请内存的driver还是比较多的：
!\[\[Pasted image 20240927115711.png\]\]

这样的compound page，可以透过内核统一的gc机制进行管理，比如在refcount即将归0的时候，put_page(compound page)可以整体释放compound page。

这显然和前面描述的THP/HugeTLB的情况是一样的，也存在一个在部分和整体两种语义中纠结的问题。

由于多个page构成了一个整体，这些page之间会有关联，我们需要某种方法解决如下的问题：

1. N个page是否组成了一个整体？
1. 这些page哪些是head（第0个page）？
1. 这些page哪些是tail（第1 ~ N - 1个）？
1. 这些page一共有多少个？
1. 如果我是一个tail，那我的head是谁？
1. 这些page如何整体释放？释放的时候需要什么析构动作？

....

在folio出现之前，内核采用如下的方法来解决上述的问题：

- 在由N个4KB组成的compound page的第0个page结构体（page\[0\]，即head page）上安置一个PG_head标记，逻辑如下：

page->flags |= (1UL \<\< PG_head);

所以，如果传给PageHead() API的是第0个page结构体，由于PG_head为真，这个API返回true。

- 在由N个4KB组成的compound page的第1~N-1的page结构体(page\[1\] ~ Page\[N-1\]，即tail page)的compound_head上的最后一位设置1，逻辑如下：

page->compound_head |=  1UL；

而除0位以外的位，则指向真正的head的page即page\[0\]，于是逻辑上，如果传入的是1~N-1这些page结构体，如下两个API分别可以取出head page和判断相关的page是否是tail page（一个compound page除page\[0\]以外的page）：

!\[\[Pasted image 20240927115721.png\]\]

- 在page\[1\]这个结构体的compound_order成员上，放置这个compound page的order，比如如果是连续4个4KB组成的复合页，则page\[1\].compound_order = 2。所以，如果我们把head传入compound_order这个API，则可以取到compound page的order数：

!\[\[Pasted image 20240927115727.png\]\]

- 在page\[1\]这个结构体的compound_dtor成员上，放置这个compound page的析构函数，此析构函数，在put_page\[page\[0\]\]并且refcount即将归0的时候会被执行。不同类型的compound page的析构函数可能会不一样:

!\[\[Pasted image 20240927115732.png\]\]

整个组织关系如下图：
!\[\[Pasted image 20240927115740.png\]\]

!\[\[Pasted image 20240927115745.png\]\]

当然，在HugeTLB和THP的场景下，page\[2\]还有更多的兼职功能（HugeTLB和THP不可能是只有2页，它们存在2MB/4KB，所以一定存在page\[2\]）。

比如HugeTLB借用page\[2\]->mapping成员：
!\[\[Pasted image 20240927115751.png\]\]

而THP借用page\[2\]的deferred_list：
!\[\[Pasted image 20240927115757.png\]\]

通过page\[0\]~page\[n-1\]中flags、compound_head、compound_dtor成员的特殊串联关系，把这N个page结构体联系在了一起。这产生了一个混乱，很多时候，我们真正想操作的，其实只是compound page的整体，比如get_page()、put_page()、lock_page()、unlock_page()等。于是这样的API里面，广泛地存在这样的compound_head()操作：
!\[\[Pasted image 20240927115804.png\]\]

就以get_page()为例，传入get_page()的page结构体，其实可能是三种情况：

1. 就是一个普通的非compound page的4KB page，这个时候，compound_head() API实际还是返回那个page；
1. 传入的是一个compound page的page\[0\]（**也即head page**），这个时候，compound_head()返回的还是page\[0\]；
1. 传入的是compound page的page\[1\] ~ page\[n\]（**也即tail page**），这个时候，compound_head()返回的是compound_head - 1，也就是page\[0\]。

另外，我们一般是用操作一组page的page\[0\]来操作整个compound page的。

!\[\[Pasted image 20240927115811.png\]\]

我们能不能把这些含混的语义扯清了呢？比如get_xxx()，这个xxx就是表示我要get一个整体呢？再比如get_yyy()就是表示我要操作一个basepage的yyy呢？另外，get_xxx()这个语义下，函数的参数就不可能是yyy呢？让天堂的归天堂，让尘土的归尘土，丁是丁，卯是卯，不香吗？

```cpp
get_xxx(struct xxx *x);
get_yyy(struct yyy *y);
```

而不是
!\[\[Pasted image 20240927115817.png\]\]

这种混乱的局面，很容易对程序员进行错误的向导，因为程序员写代码的时候，究竟在操作xxx，还是yyy，自己都拎不清了。所以需要在函数体内进行区分操作，相似的问题还存在于lock_page()、unlock_page()之类的API，比如：
!\[\[Pasted image 20240927115825.png\]\]

其实，优秀的代码都是拎得清的代码，优先的API都是强迫调用者拎清的API。

如果你看最新的内核，则可以看到两组不同的APIs:

```cpp
void folio_get(struct folio *folio);
void get_page(struct page *page);
void folio_lock(struct folio *folio);
void lock_page(struct page *page);
```

拎清楚的调用者，如果觉得自己在操作一个整体，它应该调用folio_get、folio_lock，另外，我们也强迫它搞清楚自己的参数是folio而不是page。这对于代码的读者而言，也是赏心悦目的，无需猜测的。因为，代码编写的一个基本原则就是：Don’t make me think!代码的读者并不想猜你究竟是想干xxx还是yyy，你直截了当地告诉我就好。

当我们明确地知道我们在操作一个整体/集合，我们在操作一个folio。那么这个folio和page是什么关系呢？page是folio的一部分。但是，folio结构体的定义是什么呢？在最开始的patch版本里，其实它就是：
!\[\[Pasted image 20240927115835.png\]\]

所以就数据结构本身而言，folio本质上还是一个page结构体，只是被正名了。folio本质上是一个集合的概念，比如它代表一个班级，但是它的数据结构的字长又和表示班上每个学生的数据结构是一样的。比如你的名字叫黄晓明，你是一个开发组的组长你是个工程师，你这个数据结构，其实和一般的工程师是一样的。但是，有时候，领导说，\*\*这个事情让黄晓明这边来干。他其实说的是黄晓明这个小组来干，黄晓明这个时候成为一个集体的概念。\*\*最终这个黄晓明其实和其他工程师的数据结构是一样的，但是领导说，让黄晓明干，会比说“让工程师干”要清晰明了的多。逻辑就是这么个逻辑，这体现了内核社区的洁癖，也是代码自注释的原则的体现。

早期的patch长成这样的话：
!\[\[Pasted image 20240927115843.png\]\]

这多少有点不方便，因为我们为了操作一个folio的flags、LRU、private之类的成员，我们还要先来一次folio->page的操作，比如：
!\[\[Pasted image 20240927115847.png\]\]

所以正式合入Linux 5.16 内核的folio是长下面这样的，把一些page里面常用字段，提取到了和page同等位置的union里面：
!\[\[Pasted image 20240927115853.png\]\]

如果你还没看明白呢，也许把folio和page并排列会更明白：
!\[\[Pasted image 20240927115859.png\]\]

说白了，就是同名成员在同样offset位置的简单数学游戏。这样，类似前面的folio的private的访问，就可以直接是：
!\[\[Pasted image 20240927115904.png\]\]

# **三、破局**

由此，我们搞清楚了folio并不是什么新生事物，而是一个有着集合概念的，数据结构与page对等的东西 。这样我们至少破除了folio的神秘感。

我们看看内核里面关于folio的注释：

_A folio is a physically, virtually and logically contiguous set of bytes.  It is a power-of-two in size, and it is aligned to that same power-of-two.  It is at least as large as %PAGE_SIZE.  If it is in the page cache, it is at a file offset which is a multiple of that  power-of-two.  It may be mapped into userspace at an address which is  at an arbitrary page offset, but its kernel virtual address is aligned  to its size._

其实folio就是物理连续、虚拟连续的2^n次的PAGE_SIZE的一些bytes的集合，当然这个n也是允许是0的。这个时候，有的童鞋就跳出来，为什么单页的集合也可以叫folio？你问这个问题是伤了广大单身群众的心，难道单身自己一个人过就不叫一个家庭了吗？家庭成员数量是一，侬晓得伐？

\*\*folio有一点是确定的，它必然不会是一个tail page。\*\*从而避免了前面的xxx、yyy的语义混乱（也就是Linux社区说的page结构体 的mess）。

理解理念之后，在实践环节，其实就比较简单了。Folio的开发，分了好多个阶段完成，而第一个阶段的git pull request，就有90个patch：
!\[\[Pasted image 20240927115913.png\]\]

简单地看到这个数字，可能就有的童鞋直接从入门到放弃了。但是，实际点进去看，真的都是非常简单的替换游戏。比如我们随机点开看一个mm: Add folio_pfn()  【1】这个是求folio的pfn的，它究竟是个什么样子呢？不要太简单好吧：
!\[\[Pasted image 20240927115920.png\]\]

所以，理解folio，最本质的是理解什么时候用folio，把该用folio的，当成folio用，破除心中的迷雾。

在Linux的层面，至少但是不限于如下这些应该是一个集合：

1.加入lruvec进行内存回收管理的应该是一个集合，它或者是compound page或者就是一个普通的单页“集合”。在内核透明大页THP的场景下，其实THP都是以整体加入lruvec的，将lruvec的相关参数改为folio，可以适应更广泛的情况：THP和非THP进入lruvec。比如，著名的shrink_page_list()函数，现在就叫shrink_folio_list()，从lruvec里面拿到的，也是folio:
!\[\[Pasted image 20240927115927.png\]\]

2.refcount计数、lock等的应该是一个集合

比如：
!\[\[Pasted image 20240927115936.png\]\]

哪怕你传的是folio中的某一个page，我lock的还是一个集合：
!\[\[Pasted image 20240927115941.png\]\]

3.mem_cgroup等的记账charge应该是一个集合；
!\[\[Pasted image 20240927115947.png\]\]

4.wait writeback、bit等应该是一个集合，比如：

folio_wait_bit(struct folio \*folio, int bit_nr);

void folio_wait_writeback(struct folio \*folio);
!\[\[Pasted image 20240927115953.png\]\]

5.与address_space绑定的Page cache的查找、插入、删除等操作应该是一个集合，因为page cache也是可以是THP的。相关代码比如：
!\[\[Pasted image 20240927120000.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
!\[\[Pasted image 20240927120008.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

6.rmap相关的单元应该是一个集合

鉴于文件页page cache以及进程的匿名页都可以是THP，所以在反向映射等API中，操作的应该也是folio，比如，在 do_anonymous_page(struct vm_fault \*vmf)这个经典的匿名页page fault处理函数中，最后rmap和lruvec相关的操作都是folio：
!\[\[Pasted image 20240927120018.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

当然，历史的伟大变革不会在一瞬间完成。从page语义向folio语义的转换并非是一蹴而就的，所以可以看看最新的Linux kernel提交，仍然也一些在转义过程中。这些转义发生在文件系统、内存管理等各个领域：
!\[\[Pasted image 20240927120023.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

鉴于本质上folio和page数据结构在内存意义上相等，所以基于历史原因短期内难以改掉的代码，如果使用的仍然是page，其实在folio和page之间还是可以比较轻松地转换的。比如最简单就是强行转换：

struct page \*p = (struct page \*) folio;

这当然是代码的“bad smell”。

由于folio是一个集合语义，所以，在我们关心的是集合的一部分的时候，或者说一个部分是否属于一个compound集合的时候，我们仍然关心的是page，比如，下面的API分别判断page是否是一个tail，page是否属于一个compound：
!\[\[Pasted image 20240927120031.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在比如下面的API，copy一整个folio集合，则需要里面的page一个部分一个部分的copy：
!\[\[Pasted image 20240927120038.png\]\]
!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

folio_page(folio, n)这个API可以取出一个folio中的第n个page。

**四、结语**

让集合的是集合，让个体的是个体，条分缕析，是folio设计的根本出发点。面向的问题根源，比怎么解决更加重要。尽管Linus Torvalds也不喜欢folio这个名，但是他认可folio要解决的问题，这比叫folio还是刘麻子更关键。

**参考文献：**

【1】mm:Addfolio_pfn()https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=bf6bd276b374d44

【2】Clarifying memory management with page folioshttps://lwn.net/Articles/849538/

【3】The kernel radar: folios, multi-generational LRU, and Rust

https://lwn.net/Articles/881675/

往

期

推

荐

[ShaderNN 2.0 ：基于GPU全图形栈的高效轻量移动端推理引擎](http://mp.weixin.qq.com/s?__biz=MzAxMDM0NjExNA==&mid=2247489495&idx=1&sn=63f369ddd2cbbbb58d1765fe257b12bc&chksm=9b509c3aac27152cb289128a8cd7f9718cd66cf4b99326e534516c6a22f3cd901a2aa935edcd&scene=21#wechat_redirect)

[Chromium多进程架构，你知道多少？](http://mp.weixin.qq.com/s?__biz=MzAxMDM0NjExNA==&mid=2247489443&idx=1&sn=5c22f9d63fa3741228a217a6b1160302&chksm=9b509c4eac271558f4997f95c871724a8a235ba30b90adf5c2197cff7676a1644767e0d5a9d6&scene=21#wechat_redirect)

[十分钟读懂HEVC-SCC原理](http://mp.weixin.qq.com/s?__biz=MzAxMDM0NjExNA==&mid=2247489425&idx=1&sn=6392efe5bf6a23d818b98da5124339dd&chksm=9b509c7cac27156acc4fc006177b8dcd8a7f8a9f8ff16914ca7325296f05325bde51a3f1b124&scene=21#wechat_redirect)

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

长按关注内核工匠微信

Linux内核黑科技| 技术文章| 精选教程

Reads 3195

​

Send Message

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjNfxSer7sH7b1yJBvlpWyx1AdCugCLScXKu60Ezh9oSCrZw36x9GTL1qvIzptqlefgS1vkQwBE7OA/300?wx_fmt=png&wxfrom=18)

OPPO内核工匠

42158

Send Message

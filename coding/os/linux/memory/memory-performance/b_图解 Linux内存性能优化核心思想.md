
Linux内核那些事 _2021年11月19日 09:00_

hi，大家好，今天分享一篇**内存性能优化**的文章，文章用了大量精美的图深入浅出地分析了Linux内核slab性能优化的**核心思想**，**slab**是Linux内核小对象内存分配最重要的算法，文章分析了内存分配的各种性能问题（在不同的场景下面），并给出了这些问题的优化方案，这个对我们实现**高性能内存池算法**，或以后遇到内存性能问题的时候，有一定的启发，值得我们学习。

**Linux内核的slab**来自一种很简单的思想，即事先准备好一些会频繁分配，释放的数据结构。然而标准的slab实现太复杂且维护开销巨大，因此便分化出了更加小巧的slub，因此本文讨论的就是slub，后面所有提到slab的地方，指的都是slub。另外又由于本文主要描述内核优化方面的内容，因此想了解slab细节以及代码实现的请查看源码。

# 单CPU上单纯的slab

下图给出了单CPU上slab在分配和释放对象时的情景序列：

![[Pasted image 20241021201748.png]]

可以看出，非常之简单，而且完全达到了slab设计之初的目标。

# 扩展到多核心CPU

现在我们简单的将上面的模型扩展到多核心CPU，同样差不多的分配序列如下图所示：

![[Pasted image 20241021201831.png]]

我们看到，在只有单一slab的时候，如果多个CPU同时分配对象，冲突是不可避免的，解决冲突的几乎是唯一的办法就是加锁排队，然而这将大大增加延迟，我们看到，申请单一对象的整个时延从T0开始，到T4结束，这太久了。

多CPU无锁化并行化操作的直接思路-复制给每个CPU一套相同的数据结构。不二法门就是增加“每CPU变量”。对于slab而言，可以扩展成下面的样子：

![[Pasted image 20241021201844.png]]

如果以为这么简单就结束了，那这就太没有意义了。

# 问题

首先，我们来看一个简单的问题，如果单独的某个CPU的slab缓存没有对象可分配了，但是其它CPU的slab缓存仍有大量空闲对象的情况，如下图所示：

![[Pasted image 20241021201910.png]]

这是可能的，因为对单独一种slab的需求是和该CPU上执行的进程/线程紧密相关的，比如如果CPU0只处理网络，那么它就会对skb等数据结构有大量的需求，对于上图最后引出的问题，如果我们选择从伙伴系统中分配一个新的page(或者pages，取决于对象大小以及slab cache的order)，那么久而久之就会造成slab在CPU间分布的不均衡，更可能会因此吃掉大量的物理内存，这都是不希望看到的。

在继续之前，首先要明确的是，我们需要在CPU间均衡slab，并且这些必须靠slab内部的机制自行完成，这个和进程在CPU间负载均衡是完全不同的，对进程而言，拥有一个核心调度机制，比如基于时间片，或者虚拟时钟的步进速率等，但是对于slab，完全取决于使用者自身，只要对象仍然在使用，就不能剥夺使用者继续使用的权利，除非使用者自己释放。因此slab的负载均衡必须设计成合作型的，而不是抢占式的。

好了。现在我们知道，从伙伴系统重新分配一个page(s)并不是一个好主意，它应该是最终的决定，在执行它之前，首先要试一下别的路线。

现在，我们引出第二个问题，如下图所示：

![[Pasted image 20241021201923.png]]

谁也不能保证分配slab对象的CPU和释放slab对象的CPU是同一个CPU，谁也不能保证一个CPU在一个slab对象的生命周期内没有分配新的page(s)，这期间的复杂操作谁也没有规定。这些问题该怎么解决呢？事实上，理解了这些问题是怎么解决的，一个slab框架就彻底理解了。

# 问题的解决-分层slab cache

无级变速总是让人向往。如果一个CPU的slab缓存满了，直接去抢同级别的别的CPU的slab缓存被认为是一种鲁莽且不道义的做法。那么为何不设置另外一个slab缓存，获取它里面的对象不像直接获取CPU的slab缓存那么简单且直接，但是难度却又不大，只是稍微增加一点消耗，这不是很好吗？事实上，**CPU的L1，L2，L3 cache**不就是这个方案设计的吗？这事实上已经成为cache设计的不二法门。这个**设计思想**同样作用于slab，就是Linux内核的slub实现，现在可以给出概念和解释了。

1. **Linux kernel slab cache**：一个分为3层的对象cache模型。

1. **Level 1 slab cache**：一个空闲对象链表，每个CPU一个的独享cache，分配释放对象无需加锁。

1. **Level 2 slab cache**：一个空闲对象链表，每个CPU一个的共享page(s) cache，分配释放对象时仅需要锁住该page(s)，与Level 1 slab cache互斥，不互相包容。

1. **Level 3 slab cache**：一个page(s)链表，每个NUMA NODE的所有CPU共享的cache，单位为page(s)，获取后被提升到对应CPU的Level 1 slab cache，同时该page(s)作为Level 2的共享page(s)存在。

1. **共享page(s)**：该page(s)被一个或者多个CPU占有，每一个CPU在该page(s)上都可以拥有互相不充图的空闲对象链表，该page(s)拥有一个唯一的Level 2 slab cache空闲链表，该链表与上述一个或多个Level 1 slab cache空闲链表亦不冲突，多个CPU获取该Level 2 slab cache时必须争抢，获取后可以将该链表提升成自己的Level 1 slab cache。

该**slab cache**的图示如下：

![[Pasted image 20241021201938.png]]

其行为如下图所示：

![[Pasted image 20241021201952.png]]

### 2个场景

对于常规的对象分配过程，下图展示了其细节：

![[Pasted image 20241021202006.png]]

事实上，对于多个CPU共享一个page(s)的情况，还可以有另一种玩法，如下图所示：

![[Pasted image 20241021202019.png]]

# 伙伴系统

前面我们简短的体会了Linux内核的slab设计,不宜过长,太长了不易理解.但是最后,如果Level 3也没有获取page(s)，那么最终会落到终极的伙伴系统，伙伴系统是为了防内存分配碎片化的，所以它尽可能地做两件事：

1. **尽量分配尽可能大的内存**

1. **尽量合并连续的小块内存成一块大内存**

我们可以通过下面的图解来理解上面的原则：

![[Pasted image 20241021202041.png]]

注意，本文是关于优化的，不是伙伴系统的科普，所以我假设大家已经理解了伙伴系统。

鉴于slab缓存对象大多数都是不超过1个页面的小结构(不仅仅slab系统，超过1个页面的内存需求相比1个页面的内存需求，很少)，因此会有大量的针对1个页面的内存分配需求。从伙伴系统的分配原理可知，如果持续大量分配单一页面，会有大量的order大于0的页面分裂成单一页面，在单核心CPU上，这不是问题，但是在多核心CPU上，由于每一个CPU都会进行此类分配，而伙伴系统的分裂，合并操作会涉及大量的链表操作，这个锁开销是巨大的，因此需要优化！

Linux内核对伙伴系统针对单一页面的分配需求采取的批量分配“每CPU单一页面缓存”的方式！每一个CPU拥有一个单一页面缓存池，需要单一页面的时候，可以无需加锁从当前CPU对应的页面池中获取页面。而当池中页面不足时，系统会批量从伙伴系统中拉取一堆页面到池中，反过来，在单一页面释放的时候，会择优将其释放到每CPU的单一页面缓存中。

为了维持“每CPU单一页面缓存”中页面的数量不会太多或太少(太多会影响伙伴系统，太少会影响CPU的需求)，系统保持了两个值，当缓存页面数量低于low值的时候，便从伙伴系统中批量获取页面到池中，而当缓存页面数量大于high的时候，便会释放一些页面到伙伴系统中。

# **小结**

多CPU操作系统内核中，关键的开销就是锁的开销。我认为这是一开始的设计导致的，因为一开始，多核CPU并没有出现，单核CPU上的共享保护几乎都是可以用“禁中断”，“禁抢占”来简单实现的，到了多核时代，操作系统同样简单平移到了新的平台，因此同步操作是在单核的基础上后来添加的。简单来讲，目前的主流操作系统都是在单核年代创造出来的，因此它们都是顺应单核环境的，对于多核环境，可能它们一开始的设计就有问题。

不管怎么说，优化操作的不二法门就是禁止或者尽量减少锁的操作。随之而来的思路就是为共享的关键数据结构创建"**每CPU的缓存**“，而这类缓存分为两种类型：

**1. 数据通路缓存**

比如路由表之类的数据结构，你可以用RCU锁来保护，当然如果为每一个CPU都创建一个本地路由表缓存，也是不错的，现在的问题是何时更新它们，因为所有的缓存都是平级的，因此一种批量同步的机制是必须的。

**2. 管理机制缓存**

比如slab对象缓存这类，其生命周期完全取决于使用者，因此不存在同步问题，然而却存在管理问题。采用分级cache的思想是好的，这个非常类似于CPU的L1/L2/L3缓存，采用这种平滑的开销逐渐增大，容量逐渐增大的机制，并配合以设计良好的换入/换出等算法，效果是非常明显的。

转自：https://zhuanlan.zhihu.com/p/426763007

阅读 2882


​---


写留言

**留言 1**

- Vincent

  2021年11月19日

  赞1

  tcmalloc

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/ciab8jTiab9J6dgBc9aqzhEyz7LkUJ812dSOibgAHcHicR8zE8PyD3bvkyicjTSGfFsF1racDTDviayU3Mbcra30sacw/300?wx_fmt=png&wxfrom=18)

Linux内核那些事

13111

1

写留言

**留言 1**

- Vincent

  2021年11月19日

  赞1

  tcmalloc

已无更多数据

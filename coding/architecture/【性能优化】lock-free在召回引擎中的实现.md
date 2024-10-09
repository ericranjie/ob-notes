# 

原创 雨乐 高性能架构探索

_2021年12月13日 12:08_

![](http://mmbiz.qpic.cn/mmbiz_png/p3sYCQXkuHhKgtwWvzaYZodgfpphdA6WWKEMXTn6ImCCCuEzlPKicNBcpzBUyjK1XicWwqIwusqLGpwyyOc87JPQ/300?wx_fmt=png&wxfrom=19)

**高性能架构探索**

专注于分享干货，硬货，欢迎关注😄

88篇原创内容

公众号

> ❝
>
> 对本文有疑问的，可以公众号留言、私信，也可以加笔者微信直接交流(文末留有微信二维码)；另外还有批量免费计算机电子书，后台回复\[pdf\]免费获取。
>
> ❞

大家好，我是雨乐！

在我们的工作中，多线程编程是一件太稀松平常的事。在多线程环境下操作一个变量或者一块缓存，如果不对其操作加以限制，轻则变量值或者缓存内容不符合预期，重则会产生异常，导致进程崩溃。为了解决这个问题，操作系统提供了锁、信号量以及条件变量等几种线程同步机制供我们使用。如果每次操作都使用上述机制，在某些条件下(系统调用在很多情况下不会陷入内核)，系统调用会陷入内核从而导致上下文切换，这样就会对我们的程序性能造成影响。

今天，借助此文，分享一下去年引擎优化的一个点，最终优化结果就是在多线程环境下访问某个变量，实现了无锁(lock-free)操作。

## 背景

对于后端开发者来说，服务稳定性第一，性能第二，二者相辅相成，缺一不可。

作为IT开发人员，秉承着一句话:只要程序正常运行，就不要随便动。所以程序优化就一直被搁置，因为没有压力，所以就没有动力嘛😁。在去年的时候，随着广告订单数量越来越多，导致服务rt上涨，光报警邮件每天都能收到上百封，于是痛定思痛，决定优化一版。

秉承小步快跑的理念，决定从各个角度逐步优化，从简单到困难，逐个击破。所以在分析了代码之后，准备从锁这个角度入手，看看能否进行优化。

在进行具体的问题分析以及优化之前，先看下现有召回引擎的实现方案，后面的方案是针对现有方案的优化。

![图片](https://mmbiz.qpic.cn/mmbiz_png/p3sYCQXkuHgYWMLkicCMPg4KWdItWu5Vz90bZMRlBiayxx04jc7QNWVQrAmgntrJGrwY9icXh9SpwAdGOBMDNOpzA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

- 广告订单以HTTP方式推送给消息系统

- 消息系统收到广告订单消息后

- 将广告订单消息格式化后推送给消息队列kafka(第1步)

- 将广告订单消息持久化到DB(第2步)

- 召回引擎订阅kafka的topic

- 从kafka中实时获取广告订单消息，建立并实时更建立维度索引(第3步)

- 召回引擎接收pv流量，实时计算，并返回满足定向后的广告候选集(第4步)

从上面图中可以看出，召回引擎是一个多线程应用，一方面有个线程专门从kafka中获取最新的广告订单消息建立维度索引(此为写线程)，另一方面，接收线上流量，根据流量属性，获取广告候选集(此为读线程)。因为召回引擎涉及到同时读和写同一块变量，因此读写不能同时操作。

## 概述

在多线程环境下，对同一个变量访问，大致分为以下几种情况：

- 多个线程同时读

- 多个线程同时写

- 一个线程写，一个线程读

- 一个线程写，多个线程读

- 多个线程写，一个线程读

- 多个线程写，多个线程读

在上述几种情况中，多个线程同时读显然是线程安全的，而对于其他几种情况，则需要保证其_互斥排他_性，即读写不能同时进行，管他几个线程读几个线程写，代码走起。

`thread1   {     std::lock_guard<std::mutex> lock(mtx);     // do sth(read or write)   }      thread2   {     std::lock_guard<std::mutex> lock(mtx);     // do sth(read or write)   }      threadN   {     std::lock_guard<std::mutex> lock(mtx);     // do sth(read or write)   }   `

在上述代码中，每一个线程对共享变量的访问，都会通过mutex来加锁操作，这样完全就避免了共享变量竞争的问题。

![图片](https://mmbiz.qpic.cn/mmbiz_png/p3sYCQXkuHgYWMLkicCMPg4KWdItWu5VzGEEr4T2DDCmm1xmqlkvJLjTwl4pjSo4z0Hpas4mArvzG6KNAYgHSDg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

如果对于性能要求不是很高的业务，上述实现完全满足需求，但是对于性能要求很高的业务，上述实现就不是很好，所以可以考虑通过其他方式来实现。

我们设想一个场景，假如某个业务，写操作次数远远小于读操作次数，例如我们的召回引擎，那么我们完全可以使用读写锁来实现该功能，换句话说\*\*_读写锁适合于读多写少_\*\*的场景。

> ❝
>
> 读写锁其实还是一种锁，是给一段临界区代码加锁，但是此加锁是在进行写操作的时候才会互斥，而在进行读的时候是可以共享的进行访问临界区的，其本质上是一种自旋锁。
>
> ❞

![图片](https://mmbiz.qpic.cn/mmbiz_png/p3sYCQXkuHgYWMLkicCMPg4KWdItWu5VzDAWuXV4cxiaBYiaCdH5bAnmiaibShV5wJB5IxhD62DqW5VqZ5LFfyXicCzQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

代码实现也比较简单，如下：

`writer thread {    pthread_rwlock_wrlock(&rwlock);     // do write operation     pthread_rwlock_unlock(&rwlock);   }      reader thread2 {     pthread_rwlock_rdlock(&rwlock);     // do read operation     pthread_rwlock_unlock(&rwlock);   }      reader threadN {     pthread_rwlock_rdlock(&rwlock)     // do read operation     pthread_rwlock_unlock(&rwlock);   }   `

在此，说下读写锁的特性：

- 读和读指针没有竞争关系

- 写和写之间是互斥关系

- 读和写之间是同步互斥关系(这里的同步指的是写优先，即读写都在竞争锁的时候，写优先获得锁)

那么，对于一写多读的场景，还有没有可能进行再次优化呢？

答案是：有的。

下面，我们将针对一写多读，读多写少的场景，进行优化。

## 方案

在上一节中，我们提到对于多线程访问，可以使用mutex对共享变量进行加锁访问。对于一写多读的场景，使用读写锁进行优化，使用读写锁，在读的时候，是不进行加锁操作的，但是当有写操作的时候，就需要加锁，这样难免也会产生性能上的影响，在本节，我们提供终极优化版本，目的是在写少读多的场景下实现lock-free。

如何在读写都存在的场景下实现lock-free呢？假设如果有两个共享变量，一个变量用来专供写线程来写，一个共享变量用来专供读线程来读，这样就不存在读写同步的问题了，如下所示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在上节中，我们有提到，多个线程对一个变量同时进行读操作，是线程安全的。一个线程对一个变量进行写操作也是线程安全的(这不废话么，都没人跟它竞争)，那么结合上述两点，上图就是线程安全的(多个线程读一个资源，一个线程写另外一个资源)。

好了，截止到现在，我们lock-free的雏形已经出来了，就是_使用双变量_来实现lock-free的目标。那么reader线程是如何第一时间能够访问writer更新后的数据呢？

> ❝
>
> 假设有两个共享资源A和B，当前情况下，读线程正在读资源A。突然在某一个时刻，写线程需要更新资源，写线程发现资源A正在被访问，那么其更新资源B，更新完资源B后，进行切换，让读线程读资源B，然后写线程继续写资源A，这样就能完全实现了lock-free的目标，此种方案也可以称为\*\*_双buffer_\*\*方式。
>
> ❞

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 实现

在上节中，我们提出了使用双buffer来实现lock-free的目标，那么如何实现读写buffer无损切换呢？

#### 指针互换

假设有两个资源，其指针分别为ptrA和ptrB，在某一时刻，ptrA所指向的资源正在被多个线程读，而ptrB所指向的资源则作为备份资源，此时，如果有写线程进行写操作，按照我们之前的思路，写完之后，马上启用ptrA作为读资源，然后写线程继续写ptrB所指向的资源，这样会有什么问题呢？

我们就以std::vector为例，如下图所示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在上图左半部分，假设ptr指向读对象的指针，也就是说读操作只能访问ptr所指向的对象。

某一时刻，需要对对象进行写操作(删除对象Obj4)，因为此时ptr = ptrA，因此写操作只能操作ptrB所指向的对象，在写操作执行完后，将ptr赋值为ptrB(保证后面所有的读操作都是在ptrB上)，即保证当前ptr所指向的对象永远为最新操作，然后写操作去删除ptrA中的Obj4，但是此时，有个线程正在访问ptrA的Obj4，自然而然会轻则当前线程获取的数据为非法数据，重则程序崩溃。

> ❝
>
> 此方案不可行，主要是因为在写操作的时候，没有判断当前是否还有读操作。
>
> ❞

#### 原子性

在上述方案中，简单的变量交换，最终仍然可能存在读写同一个变量，进而导致崩溃。那么如果保证在写的时候，没有读是不是就能解决上述问题了呢？如果是的话，那么应该如何做呢？

显然，此问题就转换成如何判断一个对象上存在线程读操作。

用过std::shared_ptr的都知道，其内部有个成员函数use_count()来判断当前智能指针所指向变量的访问个数，代码如下：

`long         _M_get_use_count() const noexcept         {           // No memory barrier is used here so there is no synchronization           // with other threads.           return __atomic_load_n(&_M_use_count, __ATOMIC_RELAXED);         }   `

那么，我们可以考虑采用智能指针的方案，代码也比较简单，如下：

`std::atomic_size_t curr_idx = 0;      std::vector<std::shared_ptr<Obj>> obj_buffers;   obj_buffers.emplace_back(std::make_shared<Obj>(...));   obj_buffers.emplace_back(std::make_shared<Obj>(...));      // write thread    {     size_t prepare = 1 - curr_idx.load();     while (obj_buffers[prepare].use_count() > 1) {      continue;     }         obj_buffers[prepare]->load();     curr_idx = prepare;    }        // read thread     {      auto tmp = obj_buffers[curr_idx.load()];      // do sth    }`

在上述代码中

- 首先创建一个vector，其内有两个Obj的智能指针，这俩智能指针所指向的Obj对象一个供读线程进行读操作，一个供写线程进行写操作

- curr_idx代表当前可供读操作对象在obj_buffers的索引，即obj_buffers\[curr_idx.load()\]所指对象供读线程进行读操作

- 那么相应的，obj_buffers\[1- curr_idx.load()\]所指对象供写线程进行写操作

- 在读线程中

- 通过auto tmp = obj_buffers\[curr_idx.load()\];获取一个拷贝，由于obj_buffers中存储的是shared_ptr那么，该对象的引用计数+1

- 在tmp上进行读操作

- 在写线程中

- prepare = 1 - curr_idx.load();在上面我有提到curr_idx指向可读对象在obj_buffers的索引，换句话说，1 - curr_idx.load()就是另外一个对象即可写对象在obj_buffers中的索引

- 通过while循环判断另外一个对象的引用计数是否大于1(如果大于1证明还有读线程正在进行读操作)

好了，截止到此，lock-free的实现目标基本已经完成。实现原理也也相对来说比较简单，重点是要保证\*\*_写的时候没有读操作_\*\*即可。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

上图是召回引擎做了lock-free优化后的效果图，从图上来看，效果还是很明显的。

## 扩展

双buffer方案在“一写多读”的场景下能够实现lock-free的目标，那么对于“多写一读”或者“多写多读”场景，是否也能够满足呢？

答案是\*\*_不太适合_\*\*，主要是以下两个原因：

- 在多写的场景下，多个写之间需要通过锁来进行同步，虽然避免了对读写互斥情况加锁，但是多线程写时通常对数据的实时性要求较高，如果使用双buffer，所有新数据必须要等到索引切换时候才能使用，很可能达不到实时性要求

- 多线程写时若用双buffer模式，则在索引切换时候也需要给对应的对象加锁，并且也要用类似于上面的while循环保证没有现成在执行写入操作时才能进行指针切换，而且此时也要等待读操作完成才能进行切换，这时候就对备用对象的锁定时间过长，在数据更新频繁的情况下是不合适的。

## 缺点

通过前面的章节，我们知道通过双buffer方式可以实现在一写多读场景下的lock-free，该方式要求两个对象或者buffer最终持有的数据是完全一致的，也就是说在单buffer情况下，只需要一个buffer持有数据就行，但是双buffer情况下，需要持有两份数据，所以存在内存浪费的情况。

其实说白了，双buffer实现lock-free，就是采用的空间换时间的方式。

## 结语

双buffer方案在多线程环境下能较好的解决 “一写多读” 时的数据更新问题，特别是适用于数据需要定期更新，且一次更新数据量较大的情形。

性能优化是一个漫长的不断自我提升的过程，项目中的一点点优化往往就可以使得性能得到质的提升。

好了，今天的文章就到这，我们下期见。

如果对本文有疑问可以加笔者**微信**直接交流，笔者也建了C/C++相关的技术群，有兴趣的可以联系笔者加群。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**往期****精彩****回顾**

[](http://mp.weixin.qq.com/s?__biz=Mzk0MzI4OTI1Ng==&mid=2247485953&idx=1&sn=f8cd484607ab07f15247ecde773d2e1c&chksm=c3376cc6f440e5d047f7e648c951fd583df82ab4e3dab5767baeddef9fe7c1270f05b039d8c4&scene=21#wechat_redirect)[【性能优化】高效内存池的设计与实现](http://mp.weixin.qq.com/s?__biz=Mzk0MzI4OTI1Ng==&mid=2247486254&idx=1&sn=ebce58aa6b547af2a818faa5a6412e89&chksm=c3376de9f440e4ffea267926b7ce09ac439ab33a1da9dc6b4d631c971053f628cf91202d0f53&scene=21#wechat_redirect)

[2万字|30张图带你领略glibc内存管理精髓](http://mp.weixin.qq.com/s?__biz=Mzk0MzI4OTI1Ng==&mid=2247485953&idx=1&sn=f8cd484607ab07f15247ecde773d2e1c&chksm=c3376cc6f440e5d047f7e648c951fd583df82ab4e3dab5767baeddef9fe7c1270f05b039d8c4&scene=21#wechat_redirect)

[【万字长文】吃透负载均衡](http://mp.weixin.qq.com/s?__biz=Mzk0MzI4OTI1Ng==&mid=2247484826&idx=1&sn=810c83bfd63ddaf1500e3ece97f71a8c&chksm=c337635df440ea4bc21c4fbb252acc3a17f6c23cb963be75cbfe3dd722a33700d5f1fe3f0d18&scene=21#wechat_redirect)

[流量控制还能这么搞。。。](http://mp.weixin.qq.com/s?__biz=Mzk0MzI4OTI1Ng==&mid=2247484677&idx=1&sn=dbf3afa848db5026227c1d87cfc3b3dc&chksm=c33763c2f440ead481486c8dd256b620d7b415fec4b334981465347087ef1a7ba13980fbf073&scene=21#wechat_redirect)

[技术十年](http://mp.weixin.qq.com/s?__biz=Mzk0MzI4OTI1Ng==&mid=2247484671&idx=1&sn=9c677de01f9dd4d0292c35f79fad660c&chksm=c3376238f440eb2e0600e86b6bd853606d7a8de1b0e5e661c81366e627adb67a8acb88cea508&scene=21#wechat_redirect)\
[聊聊服务注册与发现](http://mp.weixin.qq.com/s?__biz=Mzk0MzI4OTI1Ng==&mid=2247484674&idx=1&sn=81ee9f363b4a7019ff1e77bcaecf234b&chksm=c33763c5f440ead351ebdb39a47fa922180dc2fc647dfd31de6407d8f97dfcb526332df463e6&scene=21#wechat_redirect)

**点个关注吧!**

![](http://mmbiz.qpic.cn/mmbiz_png/p3sYCQXkuHhKgtwWvzaYZodgfpphdA6WWKEMXTn6ImCCCuEzlPKicNBcpzBUyjK1XicWwqIwusqLGpwyyOc87JPQ/300?wx_fmt=png&wxfrom=19)

**高性能架构探索**

专注于分享干货，硬货，欢迎关注😄

88篇原创内容

公众号

多线程1

编程3

c++48

架构4

C/C++系列56

阅读 1255

​

写留言

**留言 5**

- 程序喵

  2021年12月13日

  赞2

  大佬这文章是真的硬核

  置顶

  高性能架构探索

  作者2021年12月13日

  赞2

  喵大人见笑了![😂](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 秦时明月

  2022年1月14日

  赞1

  和shared_mutex啥区别

  高性能架构探索

  作者2022年1月14日

  赞

  这是17吧，我们最多11，所以没用过![😂](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 程序员小寒

  2021年12月13日

  赞

  大佬干货真多

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/p3sYCQXkuHhKgtwWvzaYZodgfpphdA6WWKEMXTn6ImCCCuEzlPKicNBcpzBUyjK1XicWwqIwusqLGpwyyOc87JPQ/300?wx_fmt=png&wxfrom=18)

高性能架构探索

19811

5

写留言

**留言 5**

- 程序喵

  2021年12月13日

  赞2

  大佬这文章是真的硬核

  置顶

  高性能架构探索

  作者2021年12月13日

  赞2

  喵大人见笑了![😂](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 秦时明月

  2022年1月14日

  赞1

  和shared_mutex啥区别

  高性能架构探索

  作者2022年1月14日

  赞

  这是17吧，我们最多11，所以没用过![😂](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- 程序员小寒

  2021年12月13日

  赞

  大佬干货真多

已无更多数据

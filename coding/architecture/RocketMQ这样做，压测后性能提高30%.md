# 

原创 丁威 中间件兴趣圈

_2021年11月29日 08:32_

点击上方“中间件兴趣圈”，选择“设为星标”

越努力越幸运，唯有坚持不懈！[![图片](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFtRsQop4YtGM7EiaiaXNibdElEDFKEfJmfXo7yxtC5GRPOJveia0HbBMRAW3NUV8qW77U0B5RF2uoxE8w/640?wx_fmt=png&wxfrom=13&tp=wxpic)](http://mp.weixin.qq.com/s?__biz=MzIzNzgyMjYxOQ==&mid=2247486570&idx=1&sn=25afad26c7986aaaf32e0be8de2fc334&chksm=e8c3fb9edfb4728831bd29cbc6b9805baea8b2670401cf54f21f0675c6026811bea511dac811&scene=21#wechat_redirect)

从官方这边获悉，RocketMQ在4.9.1版本中对消息发送进行了大量的优化，性能提升十分显著，接下来请跟着我一起来欣赏大神们的杰作。

根据RocketMQ4.9.1的更新日志，我们从中提取到关于消息发送性能优化的【Issues:2883】,详细链接如下：具体优化点如截图所示：

![图片](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFuQV5wLSzxwWx9IU4pJkXh7Xoiapf1u6XIsRot08whI0jQQT1PbZHYDJZ101UIMBMjXEibCWDPxLwcw/640?wx_fmt=png&wxfrom=13&tp=wxpic)

首先先尝试对上述优化点做一个简单的介绍：

- 对WaitNotifyObject的锁进行优化(item2)

- 移除HAService中的锁(item3)

- 移除GroupCommitService中的锁(item4)

- 消除HA中不必要的数组拷贝(item5)

- 调整消息发送几个参数的默认值(item7)

- sendMessageThreadPoolNums

- useReentrantLockWhenPutMessage

- flushCommitLogTimed

- endTransactionThreadPoolNums

- 减少琐的作用范围(item8-12)

通过阅读上述的变更，总结出优化手段主要包括如下三点：

- 移除不必要的锁

- 降低锁粒度(范围)

- 修改消息发送相关参数

接下来结合源码，从中挑选具有代表性功能进行详细剖析，一起领悟Java高并发编程的魅力。

## 1、移除不必要的锁

本次性能优化，主要针对的是RocketMQ同步复制场景。

我们首先先来简单介绍一下RocketMQ主从同步在编程方面的技巧。

RocketMQ主节点将消息写入内存后， 如果采用同步复制，需要等待从节点成功写入后才能向消息发送客户端返回成功，在代码编写方面也极具技巧性，时许图如下图所示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

> 温馨提示：在RocketMQ4.7版本开始对消息发送进行了优化，同步消息发送模型引入了jdk的CompletableFuture实现消息的异步发送。

核心步骤解读：

1. 消息发送线程调用Commitlog的aysncPutMessage方法写入消息。

1. Commitlog调用submitReplicaRequest方法，将任务提交到GroupTransferService中，并获取一个Future，实现异步编程。值得注意的是这里需要等待，待数据成功写入从节点（内部基于CompletableFuture机制的内部线程池ForkJoin）。

1. GroupTransferService中对提交的任务依次进行判断，判断对应的请求是否已同步到从节点。

1. 如果已经复制到从节点，则通过Future唤醒，并将结果返回给消息发送端。

GroupTransferService代码如下图所示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

为了更加方便大家理解接下来的优化点，首先再总结提炼一下GroupTransferService的设计理念：

- 首先引入两个List结合，分别命名为读、写链表。

- 外部调用GroupTransferService的putRequest请求，将存储在写链表中(requestWrite)。

- GroupTransferService的run方法从requestRead链表中获取任务，判断这些任务对应的请求的数据是否成功写入到从节点。

- 每当requestRead中没有数据可读时，两个队列进行交互，**从而实现读写分离，降低锁竞争**。

新版本的优化点主要包括：

- 更改putRequest的锁类型，用自旋锁替换synchronized

- 去除doWaitTransfer方法中多余的锁

#### 1.1 使用自旋锁替换synchronized

正入下图所示，GroupTransferService向外提供接口putRequest，用来接受外部的同步任务，需要对ArrayList加锁进行保护，往ArrayList中添加数据属于一个内存操作，操作耗时小。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

故这里没必要采取synchronized这种**synchronized**，而是可以自旋锁，自旋锁的实现非常轻量级，其实现如下图所示：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

整个锁的实现就只需引入一个AtomicBoolean，加锁、释放锁都是基于CAS操作，非常的轻量，并且**自旋锁不会发生线程切换**。

#### 1.2 去除多余的锁

“锁”的滥用是一个非常普遍的现象，多线程环境编程是一个非常复杂的交互过程，在编写代码过程中我们可能觉得自己无法预知这段代码是否会被多个线程并发执行，为了谨慎起见，就直接简单粗暴的对其进行加锁，带来的自然是性能的损耗，这里将该锁去除，我们就要结合该类的调用链条，判断是否需要加锁。

整个GroupTransferService中在多线程环境中运行需要被保护的主要是requestRead与requestWrite集合，引入的锁的目的也是确保这两个集合在多线程环境下安全访问，故我们首先应该梳理一下GroupTransferService的核心方法的运作流程：!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

doWaitTransfer方法操作的主要对象是requestRead链表，而且该方法只会被GroupTransferService线程调用，并且requestRead中方法会在swapRequest中被修改，但这两个方法是串行执行，而且在同一个线程中，故无需引入锁，该锁可以移除。

但由于该锁被移除，在swapRequests中进行加锁，因为requestWrite这个队列会被多个线程访问，优化后的代码如下：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)从这个角度来看，其实主要是将锁的类型由synchronized替换为更加轻量的自旋锁。

## 2、降低锁的范围

被锁包裹的代码块是串行执行，即无法并发，在无法避免锁的情况下，降低锁的代码块，能有效提高并发度，图解如下：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

如果多个线程区访问lock1,lock2，在lock1中domSomeThing1、domSomeThing2这两个方法都必须串行执行，而多个线程同时访问lock2方法，doSomeThing1能被多个线程同时执行，只有doSomething2时才需要串行执行，其整体并发效果肯定是lock2，基于这样理论：得出一个锁使用的最佳实践：**被锁包裹的代码块越少越好**。

在老版本中，消息写入加锁的代码块比较大，一些可以并发执行的动作也被锁包裹，例如生成offsetMsgId。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)新版本采用函数式编程的思路，只是定义来获取msgId的方法，在进行消息写入时并不会执行，降低锁的粒度，使得offsetMsgId的生成并行化，其编程手段之巧妙，值得我们学习。

## 3、调整消息发送相关的参数

1. sendMessageThreadPoolNums

   Broker端消息发送端线程池数量，该值在4.9.0版本之前默认为1，新版本调整为操作系统的CPU核数，并且不小于4。该参数的调整有利有弊。提高了消息发送的并发度，但同时会导致消息顺序的乱序,其示例图如下同步发送下不会有顺序问题，可放心修改

   !\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)在顺序消费场景，该参数不建议修改。在实际过程中应该对RocketMQ集群进行治理，顺序消费的场景使用专门集群。

1. useReentrantLockWhenPutMessage MQ消息写入时对内存加锁使用的锁类型，低版本之前默认为false,表示默认使用自旋锁；新版本使用ReentrantLock。自旋主要的优势是没有线程切换成本，但自旋容易造成CPU的浪费，内存写入大部分情况下是很快，但RocketMQ比较依赖页缓存，如果出现也缓存抖动，带来的CPU浪费是非常不值得，在sendMessageThreadPoolNums设置超过1之后，锁的类型使用ReentrantLock更加稳定。

1. flushCommitLogTimed 首先我们通过观察源码了解一下该参数的含义：!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

   其主要作用是控制刷盘线程阻塞等待的方式，低版本flushCommitLogTimed为false，默认使用CountDownLatch，而高版本则直接使用Thread.sleep。猜想的原因是刷盘线程比较独立，无需与其他线程进行直接的交互协作，故无需使用CountDownLatch这种专门用来线程协作的“外来和尚”。

1. endTransactionThreadPoolNums

   主要用于设置事务消息线程池的大小。!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)新版本主要是可通过调整发送线程池来动态调节事务消息的值，这个大家可以根据压测结果动态调整。

### 最后说一句(求关注，别白嫖我)

如果这篇文章对您有所帮助，或者有所启发的话，帮忙扫描下发二维码关注一下，您的支持是我坚持写作最大的动力。

求一键三连：点赞、转发、在看。

![](http://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFvs5UnzjhzrS6h29QXbfK4I0GLibNP4Qlpt1ovSdmwfsoY7D4JYUZzkACtqe3wrKh6icG7oHUTMibJbA/300?wx_fmt=png&wxfrom=19)

**中间件兴趣圈**

《RocketMQ技术内幕》作者维护，主打成体系剖析JAVA主流中间件架构与设计原理，为构建完备的互联网分布式架构体系而努力，助力突破职场瓶颈。

232篇原创内容

公众号

关注公众号：「中间件兴趣圈」，在公众号中回复：「PDF」可获取大量学习资料，回复「专栏」可获取15个主流Java中间件源码分析专栏，另外回复：加群，可以跟很多BAT大厂的前辈交流和学习。

走进作者

[10年IT老兵给职场新人的一些建议](http://mp.weixin.qq.com/s?__biz=MzIzNzgyMjYxOQ==&mid=2247484618&idx=1&sn=e24d7d19006f0d66e697e8d2be4aa508&chksm=e8c3f33edfb47a286f4515c4b11e822c35eab9b6c7ada25ac2cce3d2f7e5dac0230b54c56646&scene=21#wechat_redirect)

[“我”被阿里巴巴宠幸了](http://mp.weixin.qq.com/s?__biz=MzIzNzgyMjYxOQ==&mid=2247486100&idx=1&sn=3166338465f9b4a47ad93ecf13df6e48&chksm=e8c3fd60dfb47476e0c3ff65673eee47a5b99c7455f70252d08d6d0330828ea9050b27526a7d&scene=21#wechat_redirect)

[程序员如何提高影响力](http://mp.weixin.qq.com/s?__biz=MzIzNzgyMjYxOQ==&mid=2247485407&idx=1&sn=0e0de515b3a66ac91e55fdf583be5c0d&chksm=e8c3f02bdfb4793daecebbead9c5cdf6e64da25b80f2fd3f2bcfcc52a6c3a57b2414298bd0b5&scene=21#wechat_redirect)

[优秀程序员必备技能之如何高效阅读源码](http://mp.weixin.qq.com/s?__biz=MzIzNzgyMjYxOQ==&mid=2247485573&idx=1&sn=4d936fa700b38b5158316bdaf1aeac68&chksm=e8c3ff71dfb476675613afe09c682bc5fbd454b35f8d3d6d0458360149d5f0d673965c8852c4&scene=21#wechat_redirect)

[我的另一种参与 RocketMQ 开源社区的方式](http://mp.weixin.qq.com/s?__biz=MzIzNzgyMjYxOQ==&mid=2247484822&idx=1&sn=ecaada01b1bcf73b3a9fb750872b8e9d&chksm=e8c3f262dfb47b74d6f03be903dc734953e83ee720ac5b98e7ffcd92da39df5d68308b26bf85&scene=21#wechat_redirect)

点击查看“阅读原文”，**可直接进入\[中间件兴趣圈\]文章合集。**

RocketMQ原理与实战44

RocketMQ原理与实战 · 目录

上一篇7张图揭晓RocketMQ存储设计的精髓下一篇我擦，RocketMQ的tag还有这个“坑”！

阅读原文

阅读 4049

修改于2021年11月29日

​

写留言

**留言 8**

- 丁威

  2021年11月29日

  赞5

  纠错一下：同步发送，是可以保证顺序的

  置顶

- Noname

  2021年11月29日

  赞3

  对于单客户端使用sendSync方式来说，即使服务端多线程，依旧是顺序发送，因为需要等到上个消息ack才会进行下条发送。而对于多个客户端来说，本来就很难去判断消息的先后顺序

  中间件兴趣圈

  作者2021年11月29日

  赞2

  有些场景必须保证顺序性的

  Noname

  2021年11月29日

  赞1

  顺序一般为分区顺序和全局顺序两种场景，分区顺序通过不同的key hash到不同队列，全局顺序可以通过每个broker topic创建一个队列即可，然后发送消费通过一些策略保证高可用，实现全局顺序。所以我理解基本上这个参数对顺序影响不太大，合理能提高不少性能，个人愚见![[社会社会]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  中间件兴趣圈

  作者2021年11月29日

  赞1

  ![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)是的，你是对的，我后面去优化一下这个参数，在我们公司落地实践一下

- 时间去哪儿

  2021年11月29日

  赞1

  ‍‌‭‎‫‬‬‌‮‭⁠‌楼主分析的很细致![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)，rocketmq的性能核心在于请求大量异步‫‎‭‌​‪‫‬‌‏‏‏​‪过来时，commitlog通过加锁串行写入mmap的byterbuffer‫‬‌‏‏‏对象，其他所有动作围绕这个bytebuffer做异步操作。优化细都是围绕上面这两大节奏来的。

- 张勇

  2021年12月7日

  赞

  牛 学习了

- 金牌🥇猎头Amy

  2021年11月30日

  赞

  厉害![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFvs5UnzjhzrS6h29QXbfK4I0GLibNP4Qlpt1ovSdmwfsoY7D4JYUZzkACtqe3wrKh6icG7oHUTMibJbA/300?wx_fmt=png&wxfrom=18)

中间件兴趣圈

47215

8

写留言

**留言 8**

- 丁威

  2021年11月29日

  赞5

  纠错一下：同步发送，是可以保证顺序的

  置顶

- Noname

  2021年11月29日

  赞3

  对于单客户端使用sendSync方式来说，即使服务端多线程，依旧是顺序发送，因为需要等到上个消息ack才会进行下条发送。而对于多个客户端来说，本来就很难去判断消息的先后顺序

  中间件兴趣圈

  作者2021年11月29日

  赞2

  有些场景必须保证顺序性的

  Noname

  2021年11月29日

  赞1

  顺序一般为分区顺序和全局顺序两种场景，分区顺序通过不同的key hash到不同队列，全局顺序可以通过每个broker topic创建一个队列即可，然后发送消费通过一些策略保证高可用，实现全局顺序。所以我理解基本上这个参数对顺序影响不太大，合理能提高不少性能，个人愚见![[社会社会]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  中间件兴趣圈

  作者2021年11月29日

  赞1

  ![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)是的，你是对的，我后面去优化一下这个参数，在我们公司落地实践一下

- 时间去哪儿

  2021年11月29日

  赞1

  ‍‌‭‎‫‬‬‌‮‭⁠‌楼主分析的很细致![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)，rocketmq的性能核心在于请求大量异步‫‎‭‌​‪‫‬‌‏‏‏​‪过来时，commitlog通过加锁串行写入mmap的byterbuffer‫‬‌‏‏‏对象，其他所有动作围绕这个bytebuffer做异步操作。优化细都是围绕上面这两大节奏来的。

- 张勇

  2021年12月7日

  赞

  牛 学习了

- 金牌🥇猎头Amy

  2021年11月30日

  赞

  厉害![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

已无更多数据

# 

程序喵大人

_2022年03月01日 08:36_

以下文章来源于sowhat1412 ，作者SoWhat1412

\[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM7315IP4VUZVneF7u4hYJXqRma0CMwemKXWCQDiaPmAsRg/0)

**sowhat1412**.

吾生也有涯，而知也无涯，一个IT从业人员的点滴分享。

\](https://mp.weixin.qq.com/s?\_\_biz=MzkyODU5MTYxMA==&mid=2247493393&idx=1&sn=e08a63639073705be6b92403d8f0ccac&source=41&key=daf9bdc5abc4e8d03ef84d973a87bd9cfe33e8ff8a5e1204c881beb93fdd6bce7ee855c53f9058f37bea8a5cef65e771e584fa0107228eb9c4b7d434d79a78dc688df19c2049d12c7098cd705c8b3474195a7a39f89ba485f416fa321209891e3da1323c0b374a43291d3973b64913ee7f130cb9ae6d98a4c2dbb392c43aa570&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQAYwiQHNaVkKZ7hgzHf%2FTuBLmAQIE97dBBAEAAAAAAIu9NCwkX1YAAAAOpnltbLcz9gKNyK89dVj0%2BKMulA%2FKZ%2FPxVaMLeMGSVl26MIRr3fjHcJFNdhlt8do3pPYO0r3AkYT9BL0ZZpnMvsQcxoUCXT0UFFm4sXSMpsWaiJjHHPgVsiT9M%2BTpG6SfoNnjgmaA0CpTxbhVcFTAo3hu0ViEDy%2FGLvjJt9rgJcTtgU5Wn7V%2Fn8oK4%2F6QUHggR3BLr1j2W70nmqtYfM%2FnyYVoeUNiwMFPr2FQlP1FHbUv855GlwTf5V%2B%2BLGmsnlH%2F4qVYkeIkaPCK17bD98xr&acctmode=0&pass_ticket=ypnK%2FcaJ8EUD%2F8Whsq%2F9Vw32dTDWSJ9Cu2jrLIjGcVblVjRrPgYCVWeYvexOTKrC&wx_header=1#)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

话不多说，干货走起。

# 1、HashMap

![图片](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dV0QkuqNxTFYEpn7icYvTCVLTP2sO3msyNuGoicebmNHI1uEIEMxwHMqpBStl82g9hbkRnNxOzKWpxw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

面试第一题必问的 [HashMap](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247485513&idx=1&sn=340e879f3197ae9e3d8789a1ad55a76e&scene=21#wechat_redirect)，挺考验**Javaer**的基础功底的，别问为啥放在这，因为重要！HashMap具有如下**特性**：

> 1. HashMap 的存取是没有顺序的。
>
> 1. KV 均允许为 NULL。
>
> 1. 多线程情况下该类不安全，可以考虑用 HashTable。
>
> 1. JDk8底层是数组 + 链表 + 红黑树，JDK7底层是数组 + 链表。
>
> 1. 初始容量和装载因子是决定整个类性能的关键点，轻易不要动。
>
> 1. HashMap是**懒汉式**创建的，只有在你put数据时候才会 build。
>
> 1. 单向链表转换为红黑树的时候会先变化为**双向链表**最终转换为**红黑树**，切记双向链表跟红黑树是`共存`的。
>
> 1. 对于传入的两个`key`，会强制性的判别出个高低，目的是为了决定向左还是向右放置数据。
>
> 1. 链表转红黑树后会努力将红黑树的`root`节点和链表的头节点 跟`table[i]`节点融合成一个。
>
> 1. 在删除的时候是先判断删除节点红黑树个数是否需要转链表，不转链表就跟`RBT`类似，找个合适的节点来填充已删除的节点。
>
> 1. 红黑树的`root`节点`不一定`跟`table[i]`也就是链表的头节点是同一个，三者同步是靠`MoveRootToFront`实现的。而`HashIterator.remove()`会在调用`removeNode`的时候`movable=false`。

常见HashMap考点：

> 1. HashMap原理，内部数据结构。
>
> 1. HashMap中的put、get、remove大致过程。
>
> 1. HashMap中 hash函数实现。
>
> 1. HashMap如何扩容。
>
> 1. HashMap几个重要参数为什么这样设定。
>
> 1. HashMap为什么线程不安全，如何替换。
>
> 1. HashMap在JDK7跟JDK8中的区别。
>
> 1. HashMap中链表跟红黑树切换思路。
>
> 1. JDK7中 [HashMap环产生原理](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247485348&idx=1&sn=fed45652fa26f374b5229b401e74ca2c&scene=21#wechat_redirect)。

# 2、ConcurrentHashMap

[ConcurrentHashMap](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247487563&idx=1&sn=0a223ae2bba963e3ac40b7ce6d9ecd56&scene=21#wechat_redirect) 是多线程模式下常用的并发容器，它的实现在**JDK7**跟**JDK8**区别挺大的。

### 2.1 JDK7

**JDK7**中的 [ConcurrentHashMap](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247487563&idx=1&sn=0a223ae2bba963e3ac40b7ce6d9ecd56&scene=21#wechat_redirect) 使用 **Segment** + **HashEntry** 分段锁实现并发，它的**缺点**是并发程度是由**Segment** 数组个数来决定的，并发度一旦初始化无法扩容，扩容的话只是**HashEntry**的扩容。![图片](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dV0QkuqNxTFYEpn7icYvTCVLia3Gjbib7IkODJjrFzYgF8btMw18cLAj5M55r83eDoOZ8UVBJHU3bjOw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)**Segment** 继承自 **ReentrantLock**，在此扮演锁的角色。可以理解为我们的每个**Segment**都是实现了**Lock**功能的**HashMap**。如果我们同时有多个**Segment**形成了**Segment**数组那我们就可以实现并发咯。

大致的**put**流程如下：![图片](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dV0QkuqNxTFYEpn7icYvTCVLNs7cm9bRjN1ltMUtVS0ztWNicChnv3Gb84JkKYiagaRBpibwezGNB6Cow/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

1. **ConcurrentHashMap**底层大致实现？

> ConcurrentHashMap允许多个修改操作**并发进行**，其关键在于使用了`锁分离技术`。它使用了多个锁来控制对hash表的不同部分进行的修改。内部使用段(Segment)来表示这些不同的部分，每个段其实就是一个小的HashTable，只要多个修改操作发生在不同的段上就可以并发进行。

2. **ConcurrentHashMap**在并发下的情况下如何保证取得的元素是最新的？

> 用于存储键值对数据的**HashEntry**，在设计上它的成员变量**value**跟**next**都是`volatile`类型的，这样就保证别的线程对value值的修改，get方法可以马上看到，并且get的时候是**不用加锁的**。

3. **ConcurrentHashMap**的**弱一致性**体现在clear和get方法，原因在于**没有加锁**。

> 比如迭代器在遍历数据的时候是一个**Segment**一个**Segment**去遍历的，如果在遍历完一个**Segment**时正好有一个线程在刚遍历完的**Segment**上插入数据，就会体现出不一致性。**clear**也是一样。**get**方法和**containsKey**方法都是遍历对应索引位上所有节点，都是不加锁来判断的，如果是修改性质的因为可见性的存在可以直接获得最新值，不过如果是新添加值则**无法保持一致性**。

4. **size** 统计个数**不准确**

> size方法比较有趣，先无锁的统计所有的数据量看下前后两次是否数据一样，如果一样则返回数据，如果不一样则要把全部的segment进行加锁，统计，解锁。并且size方法只是返回一个**统计性**的数字。

### 2.2 JDK8

[ConcurrentHashMap](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247487563&idx=1&sn=0a223ae2bba963e3ac40b7ce6d9ecd56&scene=21#wechat_redirect) 在**JDK8**中抛弃了分段锁，转为用 **CAS** + **synchronized**，同时将**HashEntry**改为**Node**，还加入了**红黑树**的实现，主要还是看**put**的流程(如果看了**扩容**这块，绝对可以好好吹逼一番)。!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**ConcurrentHashMap** 是如果来做到`高效并发安全`？

1. 读操作

> get方法中根本没有使用同步机制，也没有使用unsafe方法，所以**读操作是支持并发**操作的。

2. 写操作

> 基本思路跟**HashMap**的写操作类似，只不过用到了**CAS** + **syn** 实现加锁，同时还涉及到扩容的操作。**JDK8**中锁已经细化到 **table\[i\]** 了，数组位置不同可并发，位置相同则去帮忙扩容。

3. 同步处理主要是通过**syn**和**unsafe**的硬件级别原子性这两种方式完成

> 1. 当我们对某个**table\[i\]**操作时候是用**syn**加锁的。
>
> 1. 取数据的时候用的是**unsafe**硬件级别指令，直接获取指定内存的最新数据。

# 3 、并发基础知识

并发编程的出发点：**充分利用CPU计算资源**，多线程并不是一定比单线程快，要不为什么[Redis](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247488832&idx=1&sn=5999893d7fe773f54f7d097ac1c2074d&scene=21#wechat_redirect)6.0版本的核心操作指令仍然是单线程呢？对吧！

多线程跟单线程的性能都要具体任务具体分析，**talk is cheap, show me the picture**。!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 3.1 进程跟线程

**进程**：

> 进程是操作系统调用的最小单位，是系统进行资源分配和调度的独立单位。

**线程**：

> 1. 因为进程的创建、销毁、切换产生大量的时间和空间的开销，进程的数量不能太多，而线程是比进程更小的能独立运行的基本单位，他是进程的一个实体，是CPU调度的最小单位。线程可以减少程序并发执行时的时间和空间开销，使得操作系统具有更好的并发性。
>
> 1. 线程基本不拥有系统资源，只有一些运行时必不可少的资源，比如程序计数器、寄存器和栈，进程则占有堆、栈。线程，Java默认有两个线程  main 跟GC。Java是没有权限开线程的，无法操作硬件，都是调用的  **native** 的  **start0** 方法 由 **C++** 实现

### 3.2 并行跟并发

并发：

> **concurrency** : 多线程操作同一个资源，单核CPU极速的切换运行多个任务

并行：

> **parallelism** ：多个CPU同时使用，CPU多核 真正的同时执行

### 3.3 线程几个状态

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)Java中线程的状态分为6种：

1. 初始(New)：

> 新创建了一个线程对象，但还没有调用start()方法。

2. 可运行(Runnable)：

> 1. 调用线程的**start**()方法，此线程进入就绪状态。就绪状态只是说你资格运行，调度程序没有给你CPU资源，你就永远是就绪状态。
>
> 1. 当前线程**sleep**()方法结束，其他线程**join**()结束，等待用户输入完毕，某个线程拿到对象锁，这些线程也将进入就绪状态。
>
> 1. 当前线程时间片用完了，调用当前线程的**yield**()方法，当前线程进入就绪状态。
>
> 1. 锁池里的线程拿到对象锁后，进入**就绪状态**。

3. 运行中(Running)

> 就绪状态的线程在获得CPU时间片后变为运行中状态（running）。这也是线程进入运行状态的唯一的一种方式。

4. 阻塞(Blocked)：

> 阻塞状态是线程阻塞在进入**synchronized**关键字修饰的方法或代码块(获取锁)时的状态。

5. 等待(Waiting)  跟 超时等待(Timed_Waiting)：

> 1. 处于这种状态的线程不会被分配CPU执行时间，它们要等待被显式地唤醒(通知或中断)，否则会处于无限期等待的状态。
>
> 1. 处于这种状态的线程不会被分配CPU执行时间，不过无须无限期等待被其他线程显示地唤醒，在达到一定时间后它们会自动唤醒。

6. 终止(Terminated)：

> 当线程正常运行结束或者被异常中断后就会被终止。线程一旦终止了，就不能复生。

PS：

> 1. 调用 **obj.wait** 的线程需要先获取 **obj** 的 **monitor**，**wait**会释放 **obj** 的 **monitor** 并进入等待态。所以  **wait()/notify()**  都要与 **synchronized** 联用。
>
> 1. 其实线程从阻塞/等待状态 到 可运行状态都涉及到**同步队列**跟**等待队列的**，这点在 **[AQS](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247489331&idx=1&sn=89e47f0f93ddf3e1089ab11a2b42ce20&scene=21#wechat_redirect)** 有讲。

### 3.4. 阻塞与等待的区别

**阻塞**：

> 当一个线程试图获取对象锁（非JUC库中的锁，即**synchronized**），而该锁被其他线程持有，则该线程进入阻塞状态。它的特点是使用简单，由JVM调度器来决定唤醒自己，而不需要由另一个线程来显式唤醒自己，不响应中断。

**等待**：

> 当一个线程等待另一个线程通知调度器一个条件时，该线程进入等待状态。它的特点是需要等待另一个线程显式地唤醒自己，实现灵活，语义更丰富，可响应中断。例如调用：**Object.wait()**、\*\*Thread.join()\*\*以及等待 **Lock** 或 **Condition**。

虽然 **synchronized** 和 **JUC** 里的 **Lock** 都实现锁的功能，但线程进入的状态是不一样的。**synchronized** 会让线程进入**阻塞态**，而 **JUC** 里的 **Lock**是用**park()/unpark()** 来实现**阻塞/唤醒** 的，会让线程进入**等待状态**。虽然等锁时进入的状态不一样，但被唤醒后又都进入**Runnable**状态，从行为效果来看又是一样的。

### 3.5 yield 跟 sleep 区别

1. **yield** 跟 **sleep** 都能暂停当前线程，都**不会释放锁资源**，**sleep** 可以指定具体休眠的时间，而 **yield** 则依赖 **CPU** 的时间片划分。

1. **sleep**方法给其他线程运行机会时不考虑线程的优先级，因此会给低优先级的线程以运行的机会。**yield**方法只会给相同优先级或更高优先级的线程以运行的机会。

1. 调用 **sleep** 方法使线程进入**等待状态**，等待休眠时间达到，而调用我们的 **yield**方法，线程会进入**就绪状态**，也就是**sleep**需要等待设置的时间后才会进行**就绪状态**，而**yield**会立即进入**就绪状态**。

1. **sleep**方法声明会抛出 **InterruptedException**，而 **yield** 方法没有声明任何异常

1. **yield** 不能被中断，而 **sleep** 则可以接受中断。

1. **sleep**方法比**yield**方法具有更好的移植性(跟操作系统CPU调度相关)

### 3.6 wait 跟 sleep 区别

1. 来源不同

> **wait** 来自**Object**，**sleep** 来自 **Thread**

2. 是否释放锁

> **wait** 释放锁，**sleep** 不释放

3. 使用范围

> **wait** 必须在同步代码块中，**sleep** 可以任意使用

4. 捕捉异常

> **wait** 不需要捕获异常，**sleep** 需捕获异常

### 3.7 多线程实现方式

1. 继承 **Thread**，实现**run**方法

1. 实现 **Runnable**接口中的**run**方法，然后用**Thread**包装下。**Thread** 是线程对象，**Runnable** 是任务，线程启动的时候一定是对象。

1. 实现 **Callable**接口，**FutureTask** 包装实现接口，**Thread** 包装 **FutureTask**。**Callable** 与 **Runnable** 的区别在于**Callable**的**call**方法有返回值，可以抛出异常，**Callable**有缓存。

1. 通过**线程池**调用实现。

1. 通过**Spring**的注解 **@Async** 实现。

### 3.8 死锁

死锁是指两个或两个以上的线程互相持有对方所需要的资源，由于某些锁的特性，比如syn使用下，一个线程持有一个资源，或者说获得一个锁，在该线程释放这个锁之前，其它线程是获取不到这个锁的，而且会一直死等下去，因此这便造成了死锁。

> 面试官：你给我解释下死锁是什么，解释好了我就录用你。
>
> 应聘者：先发Offer，发了Offer我给你解释什么是死锁。

**产生条件**：

> 1. **互斥条件**：一个资源，或者说一个锁只能被一个线程所占用，当一个线程首先获取到这个锁之后，在该线程释放这个锁之前，其它线程均是无法获取到这个锁的。
>
> 1. **占有且等待**：一个线程已经获取到一个锁，再获取另一个锁的过程中，即使获取不到也不会释放已经获得的锁。
>
> 1. **不可剥夺条件**：任何一个线程都无法强制获取别的线程已经占有的锁
>
> 1. **循环等待条件**：线程A拿着线程B的锁，线程B拿着线程A的锁。。

**检查**：

> 1、jps -l 定位进程号
>
> 2、jstack 进程号找到死锁问题

**避免**：

> 1. **加锁顺序**：线程按照相同的顺序加锁。
>
> 1. **限时加锁**：线程获取锁的过程中限制一定的时间，如果给定时间内获取不到，就算了，这需要用到Lock的一些API。

# 4、JMM

### 4.1 JMM由来

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)随着**CPU**、**内存**、**磁盘**的高速发展，它们的访问速度差别很大。为了提速就引入了L1、L2、L3三级缓存。以后程序运行获取数据就是如下的步骤了。!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)这样虽然提速了但是会导致`缓存一致性问题`跟`内存可见性问题`。同时编译器跟CPU为了加速也引入了指令重排。指令重排的大致意思就是你写的代码运行运算结果会按照你看到的逻辑思维去运行，但是在JVM内部系统是智能化的会进行加速排序的。

> 1、编译器优化的重排序：编译器在不改变单线程程序语义的前提下，可以重新安排语句的执行顺序。
>
> 2、指令级并行的重排序：现代处理器采用了指令级并行技术在不影响数据依赖性前提下重排。
>
> 3、内存系统的重排序：处理器使用缓存和读/写缓冲区 进程重排。

指令重排这种机制会导致`有序性问题`，而在并发编程时经常会涉及到线程之间的通信跟同步问题，一般说是`可见性`、`原子性`、`有序性`。这三个问题对应的底层就是 **缓存一致性**、**内存可见性**、**有序性**。

> **原子性**：原子性就是指该操作是不可再分的。不论是多核还是单核，具有原子性的量，同一时刻只能有一个线程来对它进行操作。在整个操作过程中不会被线程调度器中断的操作，都可认为是原子性。比如 a = 1。
>
> **可见性**：指当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值。Java保证可见性可以认为通过**volatile**、**synchronized**、**final**来实现。
>
> **有序性**：程序执行的顺序按照代码的先后顺序执行，Java通过**volatile**、**synchronized**来保证。

为了保证共享内存的正确性(可见性、有序性、原子性)，内存模型定义了共享内存模式下**多线程程序读写操作行为的规范**，既**JMM**模型，注意**JMM**只是一个**约定概念**，是用来保证效果一致的**机制**跟**规范**。它作用于**工作内存和主存之间数据同步过程**，规定了如何做数据同步以及什么时候做数据同步。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)在JMM中，有两条规定：

> 1. 线程对共享变量的所有操作都必须在自己的工作内存中进行，不能直接从主内存中读写。
>
> 1. 不同线程之间无法访问其他线程工作内存中的变量，线程间变量值的传递需要通过主内存来完成。

共享变量要实现可见性，必须经过如下两个步骤：

> 1. 把本地内存1中更新过的共享变量刷新到主内存中。
>
> 1. 把主内存中最新的共享变量的值更新到本地内存2中。

同时人们提出了`内存屏障`、`happen-before`、`af-if-serial`这三种概念来保证系统的`可见性`、`原子性`、`有序性`。

### 4.2 内存屏障

[内存屏障](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247488184&idx=1&sn=20f001915d5504da77d753fe7d4a38c0&scene=21#wechat_redirect) (Memory Barrier) 是一种CPU指令，用于控制特定条件下的**重排序**和**内存可见性**问题。Java编译器也会根据内存屏障的规则禁止重排序。Java编译器在生成指令序列的适当位置会插入内存屏障指令来禁止特定类型的处理器重排序，从而让程序按我们预想的流程去执行。具有如下功能：

> 1. 保证特定操作的执行顺序。
>
> 1. 影响某些数据（或则是某条指令的执行结果）的内存可见性。

在 [volatile](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247485656&idx=1&sn=03c2eedc1f749e719230cfa87952c51c&scene=21#wechat_redirect) 中就用到了内存屏障，[volatile](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247485656&idx=1&sn=03c2eedc1f749e719230cfa87952c51c&scene=21#wechat_redirect)部分已详细讲述。

### 4.3 happen-before

因为有`指令重排`的存在会导致难以理解CPU内部运行规则，JDK用 [happens-before](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247488184&idx=1&sn=20f001915d5504da77d753fe7d4a38c0&scene=21#wechat_redirect) 的概念来阐述操作之间的内存可见性。在[JMM](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247488184&idx=1&sn=20f001915d5504da77d753fe7d4a38c0&scene=21#wechat_redirect) 中如果一个操作执行的结果需要对另一个操作可见，那么这两个操作之间必须要存在`happens-before`关系 。其中CPU的`happens-before`无需任何同步手段就可以保证的。

> - 程序顺序规则：一个线程中的每个操作，happens-before于该线程中的任意后续操作。
>
> - 监视器锁规则：对一个锁的解锁，happens-before于随后对这个锁的加锁。
>
> - volatile变量规则：对一个volatile域的写，happens-before于任意后续对这个volatile域的读。
>
> - 传递性：如果A happens-before B，且B happens-before C，那么A happens-before C。
>
> - start()规则：如果线程A执行操作ThreadB.start()（启动线程B），那么A线程的ThreadB.start()操作happens-before于线程B中的任意操作。
>
> - join()规则：如果线程A执行操作ThreadB.join()并成功返回，那么线程B中的任意操作happens-before于线程A从ThreadB.join()操作成功返回。
>
> - 线程中断规则:对线程interrupt方法的调用happens-before于被中断线程的代码检测到中断事件的发生。

### 4.4 af-if-serial

**af-if-serial** 的含义是不管怎么重排序(编译器和处理器为了提高并行度)，单线程环境下程序的执行结果不能被改变且必须正确。该语义使单线程环境下程序员无需担心重排序会干扰他们，也无需担心内存可见性问题。

# 5、volatile

[volatile](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247485656&idx=1&sn=03c2eedc1f749e719230cfa87952c51c&scene=21#wechat_redirect) 关键字的引入可以保证变量的**可见性**，但是无法保证变量的**原子性**，比如 **a++** 这样的是无法保证的。这里其实涉及到[JMM](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247488184&idx=1&sn=20f001915d5504da77d753fe7d4a38c0&scene=21#wechat_redirect) 的知识点，**Java**多线程交互是通过**共享内存**的方式实现的。当我们读写volatile变量时具有如下规则：

> 1. 当写一个**volatile**变量时，**JMM**会把该线程对应的本地中的共享变量值**刷新到主内存**。
>
> 1. 当读一个**volatile**变量时，**JMM**会把该线程对应的本地内存置为无效。线程接下来将**从主内存中读取共享变量**。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**volatile**就会用到上面说到的内存屏障，目前有四种内存屏障：

> 1. StoreStore屏障，保证普通写不和volatile写发生重排序
>
> 1. StoreLoad屏障，保证volatile写与后面可能的volatile读写不发生重排序
>
> 1. LoadLoad屏障，禁止volatile读与后面的普通读重排序
>
> 1. LoadStore屏障，禁止volatile读和后面的普通写重排序

[volatile](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247485656&idx=1&sn=03c2eedc1f749e719230cfa87952c51c&scene=21#wechat_redirect)原理：用**volatile**变量修饰的共享变量进行写操作的时候会使用**CPU**提供的**Lock**前缀指令，在**CPU**级别的功能如下：

> 1. 将当前处理器缓存行的数据写回到 系统内存。
>
> 1. 这个写回内存的操作会告知在其他CPU你们拿到的变量是无效的下一次使用时候要重新共享内存拿。

# 6、单例模式 DCL + volatile

### 6.1 标准单例模式

**高频考点单例模式**：就是将类的构造函数进行**private**化，然后只留出一个静态的 **Instance** 函数供外部调用者调用。单例模式一般标准写法是 **DCL** + **volatile**：

`public class SingleDcl {       private volatile static SingleDcl singleDcl; //保证可见性       private SingleDcl(){       }       public static SingleDcl getInstance(){           // 放置进入加锁代码，先判断下是否已经初始化好了        if(singleDcl == null) {         // 类锁 可能会出现 AB线程都在这卡着，A获得锁，B等待获得锁。         synchronized (SingleDcl.class) {        if(singleDcl == null) {        // 如果A线程初始化好了，然后通过vloatile 将变量复杂给住线程。        // 如果此时没有singleDel === null,判断 B进程 进来后还会再次执行 new 语句        singleDcl = new SingleDcl();       }      }        }           return singleDcl;       }   }   `

### 6.2 为什么用Volatile修饰

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

不用**Volatile**则代码运行时可能存在指令重排，会导致线程一在运行时执行顺序是 1-->2--> 4 就赋值给instance变量了，然后接下来再执行构造方法初始化。**问题是**如果构造方法初始化执行**没完成前** 线程二进入发现instance != null，直接给线程二个半成品，加入volatile后底层会使用内存屏障强制按照你以为的执行。

单例模式几乎是面试必考点,，一般有如下特性：

> **懒汉式**：在需要用到对象时才实例化对象，正确的实现方式是 Double Check + Lock + volatile，解决了并发安全和性能低下问题，对内存要求非常高，那么使用懒汉式写法。
>
> **饿汉式**：在类加载时已经创建好该单例对象，在获取单例对象时直接返回对象即可，对内存要求不高使用饿汉式写法，因为简单不易出错，且没有任何并发安全和性能问题。
>
> **枚举式**：Effective Java 这本书也列举了使用枚举，其代码精简，没有线程安全问题，且 **Enum** 类内部防止反射和反序列化时破坏单例。

# 7、线程池

### 7.1 五分钟了解线程池

**老王**是个深耕在帝都的一线码农，辛苦一年挣了点钱，想把钱存储到银行卡里，拿钱去银行办理遇到了如下的遭遇

1. 老王银行门口取号后发现有柜台营业ing  但是没人办理业务就`直接办理`了。

1. 老王取号后发现柜台上都有人在办理，等待席有空地，去`坐着等`办理去了。

1. 老王取号后发现柜台都有人办理，等待席也人坐满了，这个时候银行经理看到老王是老实人本着关爱老实人的态度，新开一个**临时**窗口给他办理了。

1. 老王取号后发现柜台都满了，等待座位席也满了，**临时**窗口也人满了。这个时候银行经理给出了若干`解决策略`。

> 1. 直接告知人太多不给你办理了。
>
> 1. 采用冷暴力模式，也不给不办理也不让他走。
>
> 1. 经理让老王取尝试跟座位席中最前面的人聊一聊看是否可以加塞，可以就办理，不可以还是被踢走。
>
> 1. 经理直接跟老王说谁让你来的你找谁去我这办理不了。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

上面的这个流程几乎就跟JDK线程池的大致流程类似，其中7大参数：

> 1. 营业中的3个窗口对应核心线程池数：**corePoolSize**
>
> 1. 银行总的营业窗口数对应：**maximumPoolSize**
>
> 1. 打开的临时窗口在多少时间内无人办理则关闭对应：**keepAliveTime**
>
> 1. 临时窗口存货时间单位：**TimeUnit**
>
> 1. 银行里的等待座椅就是等待队列：**BlockingQueue**
>
> 1. **threadFactory** 该参数在JDK中是 线程工厂，用来创建线程对象，一般不会动。
>
> 1. 无法办理的时候银行给出的解决方法对应：**RejectedExecutionHandler**

当线程池的任务缓存队列**已满**并且线程池中的线程数目达到**maximumPoolSize**，如果还有任务到来就会采取任务拒绝策略，一般有**四大拒绝策略**：

> 1. ThreadPoolExecutor.AbortPolicy ：丢弃任务，并抛出 **RejectedExecutionException** 异常。
>
> 1. ThreadPoolExecutor.CallerRunsPolicy：该任务被线程池拒绝，由调用 **execute**方法的线程执行该任务。
>
> 1. ThreadPoolExecutor.DiscardOldestPolicy ：抛弃队列最前面的任务，然后重新尝试执行任务。
>
> 1. ThreadPoolExecutor.DiscardPolicy：丢弃任务，也不抛出异常。

### 7.2 正确创建方式

使用**Executors**创建线程池可能会导致**OOM**。原因在于线程池中的**BlockingQueue**主要有两种实现，分别是**ArrayBlockingQueue** 和 **LinkedBlockingQueue**。

> 1. **ArrayBlockingQueue** 是一个用数组实现的有界阻塞队列，必须设置容量。
>
> 1. **LinkedBlockingQueue** 是一个用链表实现的有界阻塞队列，容量可以选择进行设置，不设置的话，将是一个无边界的阻塞队列，最大长度为Integer.MAX_VALUE，极易容易导致线程池OOM。

正确创建线程池的方式就是自己直接调用**ThreadPoolExecutor**的构造函数来自己创建线程池。在创建的同时，给**BlockQueue**指定容量就可以了。

`private static ExecutorService executor = new ThreadPoolExecutor(10, 10,           60L, TimeUnit.SECONDS,           new ArrayBlockingQueue(10));   `

### 7.3 常见线程池

罗列几种常见的[线程池创建](https://mp.weixin.qq.com/s?__biz=MzU1NTkwODE4Mw==&mid=2247492339&idx=1&sn=3667ed5b8e437b40024f5c3ba9f16789&scene=21#wechat_redirect)方式。

1. Executors.newFixedThreadPool

> 定长的线程池，有核心线程，核心线程的即为最大的线程数量，没有非核心线程。 使用的无界的等待队列是LinkedBlockingQueue。使用时候小心堵满等待队列。

2. Executors.newSingleThreadExecutor

> 创建单个线程数的线程池，它可以保证先进先出的执行顺序

3. Executors.newCachedThreadPool

> 创建一个可缓存线程池，如果线程池长度超过处理需要，可灵活回收空闲线程，若无可回收，则新建线程。

4. Executors.newScheduledThreadPool

> 创建一个定长的线程池，而且支持定时的以及周期性的任务执行，支持定时及周期性任务执行

5. ThreadPoolExecutor

> 最原始跟常见的创建线程池的方式，它包含了 7 个参数、4种拒绝策略 可用。

### 7.4 线程池核心点

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)[线程池](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247485573&idx=1&sn=d1c7a57cf12a531cdcfb9522be7f1175&scene=21#wechat_redirect) 在工作中常用，面试也是必考点。关于线程池的细节跟使用在以前举例过一个 [银行排队](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247485573&idx=1&sn=d1c7a57cf12a531cdcfb9522be7f1175&scene=21#wechat_redirect) 办业务的例子了。线程池一般主要也无非就是下面几个考点了：

> 1. 为什么用线程池。
>
> 1. 线程池的作用。
>
> 1. **7大重要参数**。
>
> 1. **4大拒绝策略**。
>
> 1. 常见线程池任务队列，如何理解有界跟无界。
>
> 1. 常用的线程池模版。
>
> 1. 如何分配线程池个数，**IO密集型** 还是 **CPU密集型**。
>
> 1. 设定一个线程池优先级队列，**Runable** 类要实现可对比功能，任务队列使用优先级队列。

# 8、ThreadLocal

[ThreadLocal](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247485381&idx=1&sn=cdcb9c6744e287cb37ac16102471ebc6&scene=21#wechat_redirect)  可以简单理解为线程本地变量，相比于  [synchronized](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247488320&idx=1&sn=6fd5cddf2a0ff68fe00ccc834e90521b&scene=21#wechat_redirect)  是用空间来换时间的思想。他会在每个线程都创建一个副本，在线程之间通过访问内部副本变量的形式做到了线程之间互相隔离。这里用到了 [弱引用](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247489183&idx=1&sn=02ab3551c473bd2c8429862e3689a94b&scene=21#wechat_redirect) 知识点：

> 如果一个对象只具有弱引用，那么GC回收器在扫描到该对象时，**无论内存充足与否，都会回收该对象的内存**。

### 8.1 核心点

每个**Thread**内部都维护一个**ThreadLocalMap**字典数据结构，字典的**Key**值是**ThreadLocal**，那么当某个**ThreadLocal**对象不再使用(没有其它地方再引用)时，每个已经关联了此**ThreadLocal**的线程怎么在其内部的**ThreadLocalMap**里做清除此资源呢？**JDK**中的**ThreadLocalMap**没有继承**java.util.Map**类，而是自己实现了一套专门用来**定时清理无效资源的字典结构**。其内部存储实体结构**Entry\<ThreadLocal, T>**继承自**java.lan.ref.WeakReference**，这样当**ThreadLocal**不再被引用时，因为弱引用机制原因，当**jvm**发现内存不足时，会自动回收弱引用指向的实例内存，即其线程内部的**ThreadLocalMap**会释放其对**ThreadLocal**的引用从而让**jvm**回收**ThreadLocal**对象。这里是重点强调下，回收的是**Key** 也就是**ThreadLocal**对象，而非整个**Entry**，所以线程变量中的值**T**对象还是在内存中存在的，所以内存泄漏的问题还没有完全解决。

接着分析底层代码会发现在调用ThreadLocal.get() 或者 ThreadLocal.set() 都会 **定期回收无效的Entry** 操作。!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

# 9、CAS

[Compare And Swap](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247488426&idx=1&sn=705cace6ce7fbc2d6f141e8b03623fff&scene=21#wechat_redirect)：比较并交换，主要是通过**处理器的指令**来保证操作的`原子性`，它包含三个操作数：

> V：变量内存地址
>
> A：旧的预期值
>
> B：准备设置的新值

当执行[CAS](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247488426&idx=1&sn=705cace6ce7fbc2d6f141e8b03623fff&scene=21#wechat_redirect)指令时，只有当 V 对应的值等于 A 时才会用 B 去更新V的值，否则就不会执行更新操作。[CAS](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247488426&idx=1&sn=705cace6ce7fbc2d6f141e8b03623fff&scene=21#wechat_redirect)可能会带来**ABA问题**、**循环开销过大问题**、一个共享变量原子性操作的**局限性**。如何解决以前写过，在此不再重复。

# 10、Synchronized

### 10.1 Synchronized 讲解

[Synchronized](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247488320&idx=1&sn=6fd5cddf2a0ff68fe00ccc834e90521b&scene=21#wechat_redirect) 是 JDK自带的线程安全关键字，该关键字可以修饰**实例方法**、**静态方法**、**代码块**三部分。该关键字可以保证**互斥性**、**可见性**、**有序性**(不解决重排)但保证**有序性**。

[Syn](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247488320&idx=1&sn=6fd5cddf2a0ff68fe00ccc834e90521b&scene=21#wechat_redirect)的底层其实是**C++**代码写的，**JDK6**前是重量级锁，调用的时候涉及到用户态跟内核态的切换，挺耗时的。**JDK6**之前 **Doug Lea**写出了**JUC**包，可以方便的让用于在用户态实现锁的使用，**Syn**的开发者被激发了斗志所以在**JDK6**后对**Syn**进行了各种性能升级。

### 10.2 Synchronized 底层

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Syn里涉及到了 [对象头](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247488320&idx=1&sn=6fd5cddf2a0ff68fe00ccc834e90521b&scene=21#wechat_redirect) 包含对象头、填充数据、实例变量。这里可以看一个美团面试题：

问题一：new Object()占多少字节

> 1. markword 8字节 + classpointer 4字节(默认用calssPointer压缩) + padding 4字节  = **16**字节
>
> 1. 如果没开启classpointer压缩：markword 8字节 + classpointer 8字节 = **16**字节

问题二：User (int id,String name) User u = new User(1,"李四")

> markword 8字节 + 开启classPointer压缩后classpointer 4字节 + instance data int 4字节 + 开启普通对象指针压缩后String4字节 + padding 4  = **24字节**

### 10.3 Synchronized  锁升级

[synchronized](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247488320&idx=1&sn=6fd5cddf2a0ff68fe00ccc834e90521b&scene=21#wechat_redirect) 锁在JDK6以后有四种状态，`无锁`、`偏向锁`、`轻量级锁`、`重量级锁`。这几个状态会随着竞争状态逐渐升级，**锁可以升级但不能降级**，但是偏向锁状态可以被重置为无锁状态。大致升级过程如下(原图公众号回复`syn`)：!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**锁对比**：

|锁状态|优点|缺点|适用场景|
|---|---|---|---|
|偏向锁|加锁解锁无需额外消耗，跟非同步方法时间相差纳秒级别|如果竞争线程多，会带来额外的锁撤销的消耗|基本没有其他线程竞争的同步场景|
|轻量级锁|竞争的线程不会阻塞而是在自旋，可提高程序响应速度|如果一直无法获得会自旋消耗CPU|少量线程竞争，持有锁时间不长，追求响应速度|
|重量级锁|线程竞争不会导致CPU自旋跟消耗CPU资源|线程阻塞，响应时间长|很多线程竞争锁，切锁持有时间长，追求吞吐量时候|

### 10.4 Synchronized 无法禁止指令重排，却能保证有序性

指令重排是程序运行时 **解释器** 跟 **CPU** 自带的加速手段，可能导致语句执行顺序跟预想不一样情况，但是无论如何重排 也必须遵循 **as-if-serial**。

避免重排的最简单方法就是**禁止处理器优化跟指令重排**，比如[volatile](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247485656&idx=1&sn=03c2eedc1f749e719230cfa87952c51c&scene=21#wechat_redirect)中用内存屏障实现，syn是关键字级别的**排他且可重入锁**，当某个线程执行到一段被syn修饰的代码之前，会先进行加锁，执行完之后再进行解锁。

当某段代码被**syn**加锁后跟解锁前，其他线程是无法再次获得锁的，只有这条加锁线程可以重复获得该锁。所以代码在执行的时候是单线程执行的，这就满足了**as-if-serial**语义，正是因为有了**as-if-serial**语义保证，单线程的**有序性**就天然存在了。

### 10.5 wait 虚假唤醒

虚假唤醒定义：

> 1. 当一个条件满足时，很多线程都被唤醒了，但只有其中部分是有用的唤醒，其它的唤醒是不对的，
>
> 1. 比如说买卖货物，如果商品本来没有货物，所有消费者线程都在**wait**状态卡顿呢。这时突然生产者进了一件商品，唤醒了所有挂起的消费者。可能导致所有的消费者都继续执行wait下面的代码，出现错误调用。

虚假唤醒原因：

> 因为 **if** 只会执行一次，执行完会接着向下执行 **if** 下面的。而 **while** 不会，直到条件满足才会向下执行 **while**下面的。

[虚假唤醒](https://mp.weixin.qq.com/s?__biz=MzU1NDU4NDQ1OQ==&mid=2247484609&idx=1&sn=d8b3ec0f2b768b6aade60861021d373f&scene=21#wechat_redirect) 解决办法：

> 在调用 **wait** 的时候要用 **while** 不能用 **if**。

### 10.6  notify()底层

1. 为何**wait**跟**notify**必须要加**synchronized**锁

> **synchronized** 代码块通过 **javap** 生成的字节码中包含**monitorenter**  和  **monitorexit**  指令线程，执行 **monitorenter** 指令可以获取对象的 **monitor**，而 **wait** 方法通过调用 **native** 方法 **wait(0)** 实现，该注释说：**The current thread must own this object's monitor**。

2. **notify** 执行后立马唤醒线程吗?

> **notify/notifyAll** 调用时并不会真正释放对象锁，只是把等待中的线程唤醒然后放入到对象的锁池中，但是锁池中的所有线程都不会立马运行，只有拥有锁的线程运行完代码块释放锁，别的线程拿到锁才可以运行。

`public void test()   {       Object object = new Object();       synchronized (object){           object.notifyAll();           while (true){             // TODO 死循环会导致 无法释放锁。           }       }   }   `

# 11、AQS

### 11.1 高频考点线程交替打印

目标是实现两个线程交替打印，实现字母在前数字在后。你可以用信号量、Synchronized关键字跟Lock实现，这里用 [ReentrantLock](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247489331&idx=1&sn=89e47f0f93ddf3e1089ab11a2b42ce20&scene=21#wechat_redirect)简单实现：

`import java.util.concurrent.CountDownLatch;   import java.util.concurrent.locks.Condition;   import java.util.concurrent.locks.Lock;   import java.util.concurrent.locks.ReentrantLock;       public class Main {    private static Lock lock = new ReentrantLock();    private static Condition c1 = lock.newCondition();    private static Condition c2 = lock.newCondition();    private static CountDownLatch count = new CountDownLatch(1);        public static void main(String[] args) {     String c = "ABCDEFGHI";     char[] ca = c.toCharArray();     String n = "123456789";     char[] na = n.toCharArray();         Thread t1 = new Thread(() -> {      try {       lock.lock();       count.countDown();       for(char caa : ca) {        c1.signal();        System.out.print(caa);        c2.await();       }       c1.signal();      } catch (InterruptedException e) {       e.printStackTrace();      } finally {       lock.unlock();      }     });         Thread t2 = new Thread(() -> {      try {       count.await();       lock.lock();       for(char naa : na) {        c2.signal();        System.out.print(naa);        c1.await();       }       c2.signal();      } catch (InterruptedException e) {       e.printStackTrace();      } finally {       lock.unlock();      }     });          t1.start();      t2.start();    }   }   `

### 11.2 AQS底层

上题我们用到了**ReentrantLock**、**Condition** ，但是它们的底层是如何实现的呢？其实他们是基于[AQS](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247489331&idx=1&sn=89e47f0f93ddf3e1089ab11a2b42ce20&scene=21#wechat_redirect)的 **同步队列** 跟 **等待队列** 实现的！

##### 11.2.1 AQS 同步队列

学[AQS](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247489331&idx=1&sn=89e47f0f93ddf3e1089ab11a2b42ce20&scene=21#wechat_redirect) 前 **CAS** + **自旋** + **LockSupport**  + **模板模式** 必须会，目的是方便理解源码，感觉比 [Synchronized](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247488320&idx=1&sn=6fd5cddf2a0ff68fe00ccc834e90521b&scene=21#wechat_redirect) 简单，因为是单纯的 **Java** 代码。个人理解[AQS](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247489331&idx=1&sn=89e47f0f93ddf3e1089ab11a2b42ce20&scene=21#wechat_redirect)具有如下几个特点：

> 1. **在AQS 同步队列中 -1 表示线程在睡眠状态**
>
> 1. **当前Node节点线程会把前一个Node.ws = -1。当前节点把前面节点ws设置为-1，你可以理解为：你自己能知道自己睡着了吗？只能是别人看到了发现你睡眠了！**
>
> 1. **持有锁的线程永远不在队列中。**
>
> 1. **在AQS队列中第二个才是最先排队的线程。**
>
> 1. **如果是交替型任务或者单线程任务，即使用了Lock也不会涉及到AQS 队列。**
>
> 1. **不到万不得已不要轻易park线程，很耗时的！所以排队的头线程会自旋的尝试几个获取锁。**
>
> 1. **并不是说 CAS 一定比SYN好，如果高并发执行时间久 ，用SYN好， 因为SYN底层用了wait() 阻塞后是不消耗CPU资源的。如果锁竞争不激烈说明自旋不严重 此时用CAS。**
>
> 1. **在AQS中也要尽可能避免调用CLH队列，因为CLH可能会调用到park，相对来耗时。**

**ReentrantLock底层**：!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

##### 11.2.2 AQS 等待队列

当我们调用 [Condition](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247489331&idx=1&sn=89e47f0f93ddf3e1089ab11a2b42ce20&scene=21#wechat_redirect) 里的 **await** 跟 **signal** 时候底层其实是这样走的。!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

# 12、线程思考

### 12.1. 变量建议使用栈封闭

> 所有的变量都是在方法内部声明的，这些变量都处于栈封闭状态。方法调用的时候会有一个栈桢，这是一个独立的空间。在这个独立空间创建跟使用则绝对是安全的，但是注意不要返回该变量哦！

### 12.2. 防止线程饥饿

> 优先级低的线程总是得不到执行机会，一般要保证资源充足、公平的分配资源、防止持有锁的线程长时间执行。

### 12.3  开发步骤

多线程编程不要为了用而用，引入多线程后会引入额外的开销。量应用程序性能一般：服务时间、延迟时间、吞吐量、可伸缩性。做应用的时候可以一般按照如下步骤：

> 1. 先确保保证程序的正确性跟健壮性，确实达不到性能要求再想如何提速。
>
> 1. 一定要以测试为基准。
>
> 1. 一个程序中串行的部分永远是有的.
>
> 1. 装逼利器：**阿姆达尔定律** S=1/(1-a+a/n)

阿姆达尔定律中 a为并行计算部分所占比例，n为并行处理结点个数：

> 1. 当1-a=0时，(即没有串行，只有并行)最大加速比s=n；
>
> 1. 当a=0时（即只有串行，没有并行），最小加速比s=1；
>
> 1. 当n无穷大时，极限加速比s→ 1/（1-a），这就是加速比的上限。例如，若串行代码占整个代码的25%，则并行处理的总体性能不可能超过4。

### 12.4 影响性能因素

1. 缩小锁的范围，能锁方法块尽量不要锁函数

1. 减少锁的粒度跟锁分段，比如ConcurrentHashMap的实现。

1. 读多写少时候用读写锁，可提高十倍性能。

1. 用CAS操作来替换重型锁。

1. 尽量用JDK自带的[常见并发容器](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247488541&idx=1&sn=442f57533fcd9a9d9241ef955fd6301f&scene=21#wechat_redirect)，底层已经足够优化了。

# 13、End

都看到这了，送你几个高频面试题吧。

> 1. **synchronized**跟**ReentrantLock**使用区别跟底层实现以及重入底层原理
>
> 1. 描述下锁的四种状态跟升级过程
>
> 1. **CAS**是什么？**CAS**的弊端是什么？
>
> 1. 你对**volatile**的理解，可见性跟指令重排咋实现的。
>
> 1. 一个对象创建过程是怎么样的。对象在内存中如何分布的，看 [JVM](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247489183&idx=1&sn=02ab3551c473bd2c8429862e3689a94b&scene=21#wechat_redirect) 即可。
>
> 1. 聊一聊单例模式，为什么**DCL**要用**volatile**
>
> 1. Object 0 = new Object() 在内存中占据几个字节
>
> 1. 你对**as-if-serial**跟**happpends-before**的理解
>
> 1. **ThreadLocal**说一说，咋解决内存泄露
>
> 1. 自旋锁一定比重量级锁效率高吗？偏向锁是否效率一定提高。
>
> 1. 线程池聊一聊如何用 注意细节，如何实现。
>
> 1. 你对**JMM**理解？
>
> 1. **Synchronized** 可以实现指令重排么？它是如何保证有序性的？
>
> 1. 聊一聊**AQS**，为什么 **AQS** 底层是 **CAS** + **Volatile**

______________________________________________________________________

往期推荐

\[

面试HashMap看这篇就够了

\](http://mp.weixin.qq.com/s?\_\_biz=MzI4NjI1OTI4Nw==&mid=2247485513&idx=1&sn=340e879f3197ae9e3d8789a1ad55a76e&chksm=ebdee971dca960676449424d5ae384bc1084a2e93109903e929bd60bc7f30758ebb8ce2edcf2&scene=21#wechat_redirect)

\[

讲真 这次绝对让你轻松学习线程池

\](http://mp.weixin.qq.com/s?\_\_biz=MzI4NjI1OTI4Nw==&mid=2247485573&idx=1&sn=d1c7a57cf12a531cdcfb9522be7f1175&chksm=ebdee9bddca960ab8f94c7de544683655d523d112dd1c02b9edb6b43952c8666d1b10639d789&scene=21#wechat_redirect)

\[

三面阿里竟然败在了volatile关键字上

\](http://mp.weixin.qq.com/s?\_\_biz=MzI4NjI1OTI4Nw==&mid=2247485656&idx=1&sn=03c2eedc1f749e719230cfa87952c51c&chksm=ebdee9e0dca960f6f2ca21c76b044ae25790770014a140ba063cfdd6c0dde156eca1f053bc1c&scene=21#wechat_redirect)

\[

我是这样给阿里面试官吹 ConcurrentHashMap的

\](http://mp.weixin.qq.com/s?\_\_biz=MzI4NjI1OTI4Nw==&mid=2247487563&idx=1&sn=0a223ae2bba963e3ac40b7ce6d9ecd56&chksm=ebdef173dca9786555ec9f5b3c9dd4f2fe125e778deba8bfd9a60b7821149480ddcb956c7d12&scene=21#wechat_redirect)

\[

侃侃JMM 助你面试锦上添花

\](http://mp.weixin.qq.com/s?\_\_biz=MzI4NjI1OTI4Nw==&mid=2247488184&idx=1&sn=20f001915d5504da77d753fe7d4a38c0&chksm=ebdef380dca97a965ad565b9561546fb8904bd8852045097dfef3db347529f464e736e925d92&scene=21#wechat_redirect)

\[

由浅入深逐步了解 Synchronized

\](http://mp.weixin.qq.com/s?\_\_biz=MzI4NjI1OTI4Nw==&mid=2247488320&idx=1&sn=6fd5cddf2a0ff68fe00ccc834e90521b&chksm=ebdef278dca97b6e0767abd7b5b8650a2a337bb9abedf8f8b0d0f71ed69caa8774c68b044cd7&scene=21#wechat_redirect)

\[

由浅入深讲解JUC下 CAS + Lock

\](http://mp.weixin.qq.com/s?\_\_biz=MzI4NjI1OTI4Nw==&mid=2247488426&idx=1&sn=705cace6ce7fbc2d6f141e8b03623fff&chksm=ebdef292dca97b840f0ba8ac900160f1b98f2a4ebb55d1757585a2731710b8bb43944887c2a3&scene=21#wechat_redirect)

\[

快速上手JUC下常见并发容器

\](http://mp.weixin.qq.com/s?\_\_biz=MzI4NjI1OTI4Nw==&mid=2247488541&idx=1&sn=442f57533fcd9a9d9241ef955fd6301f&chksm=ebdef525dca97c33db030f46cde0ed9a4f0a4fa56455916eb0c42c4c42ea238b88744999c658&scene=21#wechat_redirect)

\[

由浅入深逐步讲解Java并发的半壁江山AQS

\](http://mp.weixin.qq.com/s?\_\_biz=MzI4NjI1OTI4Nw==&mid=2247489331&idx=1&sn=89e47f0f93ddf3e1089ab11a2b42ce20&chksm=ebdef60bdca97f1dc3cb431832055e41c658703c9e354dd95c543fbe5c5a9230732a78210db8&scene=21#wechat_redirect)

\[

头条面试官手把手教学 ThreadLocal

\](http://mp.weixin.qq.com/s?\_\_biz=MzI4NjI1OTI4Nw==&mid=2247485381&idx=1&sn=cdcb9c6744e287cb37ac16102471ebc6&chksm=ebdee6fddca96feb1171b8ae69811da75d7f724d216f23dff56a5d019a2385cbb89e2799cc9d&scene=21#wechat_redirect)

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

阅读 2362

​

写留言

**留言 22**

- 阿秀

  2022年3月1日

  赞4

  爷青结，喵喵不发C++了，取关了![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  程序喵大人

  作者2022年3月1日

  赞

  没有爱了？

- 豆芽

  2022年3月1日

  赞3

  明天还没有像这样高质量的c++就取关

  程序喵大人

  作者2022年3月1日

  赞

  明天真没有

- Santiago

  2022年3月1日

  赞1

  强哥发的都感兴趣

  程序喵大人

  作者2022年3月1日

  赞

  真爱

- 秦时明月

  2022年3月9日

  赞

  模块管理功能进来了就足以无敌

- 秦时明月

  2022年3月9日

  赞

  23来了不用怕

  程序喵大人

  作者2022年3月9日

  赞

  能加入什么特性还真不一定呢

- Xman

  2022年3月3日

  赞

  java什么都好, 就是太封闭. 要是有python的开放性, 立刻无脑投入java

- 周术辉 Jacky

  2022年3月1日

  赞

  cppfather要变成Java father了，爷青结![[奸笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  程序喵大人

  作者2022年3月1日

  赞

  ![[偷笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[偷笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[偷笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- jojogh得木

  2022年3月1日

  赞

  对java毫无爱，只有恨

  程序喵大人

  作者2022年3月1日

  赞

  被伤过？

- 雷小帅

  2022年3月1日

  赞

  没有c++ 有意思

  程序喵大人

  作者2022年3月1日

  赞

  java大佬说java没意思

- 花儿冲我笑

  2022年3月1日

  赞

  JAVA cpp 都喜欢！

  程序喵大人

  作者2022年3月1日

  赞

  赞👍🏻

- (╯‵□′)╯︵┻━┻

  2022年3月1日

  赞

  java的尽头一定会走向c++![[奸笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  程序喵大人

  作者2022年3月1日

  赞

  c++也有可能走到java

- 寂之清修者

  2022年3月1日

  赞

  JAVA转c++，谁让我喜欢造轮子呢![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  程序喵大人

  作者2022年3月1日

  赞

  666

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/Av3j9GXyMmGmg0HNK7fR6Peq3uf9pUoN5Qt0s0ia8IH97pvIiaADsEvy9nm3YnR5bmA13O8ic0JJOicQO0IboKGdZg/300?wx_fmt=png&wxfrom=18)

程序喵大人

17分享8

22

写留言

**留言 22**

- 阿秀

  2022年3月1日

  赞4

  爷青结，喵喵不发C++了，取关了![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  程序喵大人

  作者2022年3月1日

  赞

  没有爱了？

- 豆芽

  2022年3月1日

  赞3

  明天还没有像这样高质量的c++就取关

  程序喵大人

  作者2022年3月1日

  赞

  明天真没有

- Santiago

  2022年3月1日

  赞1

  强哥发的都感兴趣

  程序喵大人

  作者2022年3月1日

  赞

  真爱

- 秦时明月

  2022年3月9日

  赞

  模块管理功能进来了就足以无敌

- 秦时明月

  2022年3月9日

  赞

  23来了不用怕

  程序喵大人

  作者2022年3月9日

  赞

  能加入什么特性还真不一定呢

- Xman

  2022年3月3日

  赞

  java什么都好, 就是太封闭. 要是有python的开放性, 立刻无脑投入java

- 周术辉 Jacky

  2022年3月1日

  赞

  cppfather要变成Java father了，爷青结![[奸笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  程序喵大人

  作者2022年3月1日

  赞

  ![[偷笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[偷笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[偷笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- jojogh得木

  2022年3月1日

  赞

  对java毫无爱，只有恨

  程序喵大人

  作者2022年3月1日

  赞

  被伤过？

- 雷小帅

  2022年3月1日

  赞

  没有c++ 有意思

  程序喵大人

  作者2022年3月1日

  赞

  java大佬说java没意思

- 花儿冲我笑

  2022年3月1日

  赞

  JAVA cpp 都喜欢！

  程序喵大人

  作者2022年3月1日

  赞

  赞👍🏻

- (╯‵□′)╯︵┻━┻

  2022年3月1日

  赞

  java的尽头一定会走向c++![[奸笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  程序喵大人

  作者2022年3月1日

  赞

  c++也有可能走到java

- 寂之清修者

  2022年3月1日

  赞

  JAVA转c++，谁让我喜欢造轮子呢![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

  程序喵大人

  作者2022年3月1日

  赞

  666

已无更多数据

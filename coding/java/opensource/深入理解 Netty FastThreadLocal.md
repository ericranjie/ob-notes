# 

原创 Jiang Zhu vivo互联网技术

_2023年10月18日 20:59_ _广东_

**vivo互联网技术**

分享 vivo 互联网技术干货与沙龙活动，推荐最新行业动态与热门会议。

431篇原创内容

公众号

作者：vivo 互联网服务器团队- Jiang Zhu

本文以线上诡异问题为切入点，通过对比JDK ThreadLocal和Netty FastThreadLocal实现逻辑以及优缺点，并深入解读源码，由浅入深理解Netty FastThreadLocal。

一、前言

最近在学习Netty相关的知识，在看到Netty FastThreadLocal章节中，回想起一起线上诡异问题。

**问题描述**：外销业务获取用户信息判断是否支持https场景下，获取的用户信息有时候竟然是错乱的。

**问题分析**：使用ThreadLocal保存用户信息时，未能及时进行remove()操作，而Tomcat工作线程是基于线程池的，会出现线程重用情况，所以获取的用户信息可能是之前线程遗留下来的。

**问题修复**：ThreadLocal使用完之后及时remove()、ThreadLocal使用之前也进行remove()双重保险操作。

接下来，我们继续深入了解下JDK ThreadLocal和Netty FastThreadLocal吧。

二、JDK ThreadLocal介绍

ThreadLocal是JDK提供的一个方便对象在本线程内不同方法中传递、获取的类。用它定义的变量，仅在本线程中可见，不受其他线程的影响，与其他线程**相互隔离**。

那具体是如何实现的呢？如图1所示，每个线程都会有个ThreadLocalMap实例变量，其采用懒加载的方式进行创建，当线程第一次访问此变量时才会去创建。

ThreadLocalMap使用**线性探测法**存储ThreadLocal对象及其维护的数据，具体操作逻辑如下：

假设有一个新的ThreadLocal对象，通过hash计算它应存储的位置下标为x。

此时发现下标x对应位置已经存储了其他的ThreadLocal对象，则它会往后寻找，步长为1，下标变更为x+1。

接下来发现下标x+1对应位置也已经存储了其他的ThreadLocal对象，同理则它会继续往后寻找，下标变更为x+2。

直到寻找到下标为x+3时发现是空闲的，然后将该ThreadLocal对象及其维护的数据构建一个entry对象存储在x+3位置。

在ThreadLocalMap中数据很多的情况下，很容易出现hash冲突，解决冲突需要不断的向下遍历，该操作的时间复杂度为**O(n)，效率较低**。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图1

从下面的代码中可以看出：

Entry 的 key 是**弱引用**，value 是强引用。在 JVM 垃圾回收时，只要发现弱引用的对象，不管内存是否充足，都会被回收。

但是当 ThreadLocal 不再使用被 GC 回收后，ThreadLocalMap 中可能出现 Entry 的 key 为 NULL，那么 Entry 的 value 一直会强引用数据而得不到释放，只能等待线程销毁，从而造成**内存泄漏**。

```
static class ThreadLocalMap {
```

综上所述，既然JDK提供的ThreadLocal可能存在效率较低和内存泄漏的问题，为啥不做相应的优化和改造呢？

1.从ThreadLocal类注释看，它是JDK1.2版本引入的，早期可能不太关注程序的性能。

2.大部分多线程场景下，线程中的ThreadLocal变量较少，因此出现hash冲突的概率相对较小，及时偶尔出现了hash冲突，对程序的性能影响也相对较小。

3.对于内存泄漏问题，ThreadLocal本身已经做了一定的保护措施。作为使用者，在线程中某个ThreadLocal对象不再使用或出现异常时，立即调用 remove() 方法删除 Entry 对象，养成良好的编码习惯。

三、Netty FastThreadLocal介绍

FastThreadLocal是Netty中对JDK提供的ThreadLocal优化改造版本，从名称上来看，它应该比ThreadLocal更快了，以应对Netty处理并发量大、数据吞吐量大的场景。

那具体是如何实现的呢？如图2所示，每个线程都会有个InternalThreadLocalMap实例变量。

每个FastThreadLocal实例创建时，都会采用AtomicInteger保证顺序递增生成一个不重复的下标index，它是该FastThreadLocal对象维护的数据应该存储的位置。

读写数据的时候通过FastThreadLocal的下标 index 直接定位到该FastThreadLocal的位置，时间复杂度为 O(1)，效率较高。

如果该下标index递增到特别大，InternalThreadLocalMap维护的数组也会特别大，所以FastThreadLocal是通过空间换时间来提升读写性能的。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图2

四、Netty FastThreadLocal源码分析

4.1 构造方法

```
public class FastThreadLocal<V> {
```

```
public final class InternalThreadLocalMap extends UnpaddedInternalThreadLocalMap {
```

上面这两段代码在Netty FastThreadLocal介绍中已经讲解过，这边就不再重复介绍了。

4.2 get 方法

```
public class FastThreadLocal<V> {
```

```
public final class InternalThreadLocalMap extends UnpaddedInternalThreadLocalMap {
```

源码中 **get()** 方法主要分为下面3个步骤处理：

```
通过InternalThreadLocalMap.get()方法获取当前线程的InternalThreadLocalMap。
```

下面我们继续分析一下

\*\*InternalThreadLocalMap.get()\*\*方法的实现逻辑。

```
首先判断当前线程是否是FastThreadLocalThread类型，如果是FastThreadLocalThread
```

4.3 set 方法

```
public class FastThreadLocal<V> {
```

```
public final class InternalThreadLocalMap extends UnpaddedInternalThreadLocalMap {
```

源码中 **set()** 方法主要分为下面3个步骤处理：

```
判断value是否是缺省值UNSET，如果value不等于缺省值，则会通过InternalThreadLocalMap.get()方法获取当前线程的InternalThreadLocalMap，具体实现3.2小节中get()方法已做讲解。
```

接下来我们看下

**InternalThreadLocalMap.setIndexedVariable**方法的实现逻辑。

判断index是否超出存储绑定到当前线程的数据的数组indexedVariables的长度，如果没有超出，则获取index位置的数据，并将该数组index位置数据设置新value。

如果超出了，绑定到当前线程的数据的数组需要扩容，则扩容该数组并将它index位置的数据设置新value。

扩容数组以index 为基准进行扩容，将数组扩容后的容量向上取整为 2 的次幂。然后将原数组内容拷贝到新的数组中，空余部分填充缺省值UNSET，最终把新数组赋值给 indexedVariables。

下面我们再继续看下

**FastThreadLocal.addToVariablesToRemove**方法的实现逻辑。

1.取下标index为0的数据（用于存储待清理的FastThreadLocal对象Set集合中），如果该数据是缺省值UNSET或null，则会创建FastThreadLocal对象Set集合，并将该Set集合填充到下标index为0的数组位置。

2.如果该数据不是缺省值UNSET，说明Set集合已金被填充，直接强转获取该Set集合。

3.最后将FastThreadLocal对象保存到待清理的Set集合中。

4.4 remove、removeAll方法

```
public class FastThreadLocal<V> {
```

```
public final class InternalThreadLocalMap extends UnpaddedInternalThreadLocalMap {
```

源码中 **remove()** 方法主要分为下面2个步骤处理：

```
通过InternalThreadLocalMap.getIfSet()获取当前线程的InternalThreadLocalMap。具体和3.2小节get()方法里面获取当前线程的InternalThreadLocalMap相似，这里就不再重复介绍了。
```

源码中 **removeAll()** 方法主要分为下面3个步骤处理：

```
通过InternalThreadLocalMap.getIfSet()获取当前线程的InternalThreadLocalMap。
```

五、总结

**那么使用ThreadLocal时最佳实践又如何呢？**

每次使用完ThreadLocal实例，在线程运行结束之前的finally代码块中主动调用它的remove()方法，清除Entry中的数据，避免操作不当导致的内存泄漏。

**使⽤Netty的FastThreadLocal一定比JDK原生的ThreadLocal更快吗？**

不⼀定。当线程是FastThreadLocalThread，则添加、获取FastThreadLocal所维护数据的时间复杂度是 O(1)，⽽使⽤ThreadLocal可能存在哈希冲突，相对来说使⽤FastThreadLocal更⾼效。但如果是普通线程则可能更慢。

**使⽤FastThreadLocal有哪些优点？**

正如文章开头介绍JDK原生ThreadLocal存在的缺点，FastThreadLocal全部优化了，它更⾼效、而且如果使⽤的是FastThreadLocal，它会在任务执⾏完成后主动调⽤removeAll⽅法清除数据，避免潜在的内存泄露。

END

猜你喜欢

- [记一次Redis Cluster Pipeline导致的死锁问题](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247497343&idx=1&sn=959b66ceb9a8c2fe060b6981b41a807e&chksm=ebdb86eddcac0ffb229636ec51ec94433af75c7bc58aaa5ae52b4a2bf6a2d6d2ab11583b3174&scene=21#wechat_redirect)

- [MySQL到TiDB：Hive Metastore横向扩展之路](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247497336&idx=1&sn=777059b19e224f1a4fbb1550ad1de7e8&chksm=ebdb86eadcac0ffc78dc019c5685ad137fdddd6572238525d707152c102790f82917a473d52a&scene=21#wechat_redirect)

- [开源框架中的责任链模式实践](http://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247497332&idx=1&sn=17f1ebb821156d32208bae85a909d09d&chksm=ebdb86e6dcac0ff0a4a9646ce1ad2130ec2aed7655ad357f5b45965adcf392acd02530c5067b&scene=21#wechat_redirect)

![](http://mmbiz.qpic.cn/mmbiz_png/4g5IMGibSxt45QXJZicZ9gaNU2mRSlvqhQd94MJ7oQh4QFj1ibPV66xnUiaKoicSatwaGXepL5sBDSDLEckicX1ttibHg/0?wx_fmt=png)

**vivo互联网技术**

分享 vivo 互联网技术干货与沙龙活动，推荐最新行业动态与热门会议。

431篇原创内容

公众号

服务器149

服务器 · 目录

上一篇记一次Redis Cluster Pipeline导致的死锁问题下一篇Dubbo 路由及负载均衡性能优化

阅读 2564

​

写留言

**留言 2**

- momo

  广东2023年10月26日

  赞

  它更⾼效、而且如果使⽤的是FastThreadLocal，它会在任务执⾏完成后主动调⽤removeAll⽅法清除数据，避免潜在的内存泄露。 ------------- 这里面的FastThreadLocal应该是FastThreadLocalRunnable吧？

  vivo互联网技术

  作者2023年10月27日

  赞

  Netty封装了 FastThreadLocalRunnable，FastThreadLocalRunnable 最后会执行 FastThreadLocal.removeAll()

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/4g5IMGibSxt45QXJZicZ9gaNU2mRSlvqhQd94MJ7oQh4QFj1ibPV66xnUiaKoicSatwaGXepL5sBDSDLEckicX1ttibHg/300?wx_fmt=png&wxfrom=18)

vivo互联网技术

12274

2

写留言

**留言 2**

- momo

  广东2023年10月26日

  赞

  它更⾼效、而且如果使⽤的是FastThreadLocal，它会在任务执⾏完成后主动调⽤removeAll⽅法清除数据，避免潜在的内存泄露。 ------------- 这里面的FastThreadLocal应该是FastThreadLocalRunnable吧？

  vivo互联网技术

  作者2023年10月27日

  赞

  Netty封装了 FastThreadLocalRunnable，FastThreadLocalRunnable 最后会执行 FastThreadLocal.removeAll()

已无更多数据


冰河 冰河技术

 _2022年02月08日 10:55_

**大家好，我是冰河~~**

各位小伙伴应该都上班了吧，好啦，大家把心收回来好好工作吧，今天冰河继续给大家分享一篇关于JVM的文章。

![](http://mmbiz.qpic.cn/mmbiz_png/2hHcUic5FEwEsQmOPMjJS0EZKCJ66T4VwM2ia7E7Dxj8pWyco8fSSt6EeUOTeRHM64TE2MGkxHibrXYibUALgGLncA/300?wx_fmt=png&wxfrom=19)

**冰河技术**

分享各种编程语言、开发技术、分布式与微服务架构、分布式数据库、分布式事务、云原生、大数据与云计算技术和渗透技术。另外，还会分享各种面试题和面试技巧。

673篇原创内容

公众号

点击上方卡片关注我

在《[JVM垃圾回收机制](https://mp.weixin.qq.com/s?__biz=Mzg4MjU0OTM1OA==&mid=2247499370&idx=1&sn=9584ccbeb437823a59f4a6af7058d4a8&chksm=cf56496bf821c07dc09d199169e0a3057bf3a400420837594dc37d0cc2edfab5ca6fd82b041d&token=737594716&lang=zh_CN&scene=21#wechat_redirect)》一文中，我们知道了哪些类是需要清除的，那在java虚拟机中，他有哪些垃圾收集算法呢？

## 标记-清除

标记-清除算法就是，先标记哪些对象是存活的，哪些对象是可以回收的，然后再把标记为可回收的对象清除掉。

从下面的图可以看到，回收后，产生了大量的不连续的内存碎片，如果这个时候，有一个比较大的对象进来，却没有一个连续的内存空间给他使用，又会触发一次垃圾收集动作。

![图片](https://mmbiz.qpic.cn/mmbiz_png/2hHcUic5FEwELfoOicV02JexQG6EEyPa2kGuHAiaicB20OJXD0oYvSzxkb22yrZBbNb8IhficHq1xrwYT7l9dJCUuBw/640?wx_fmt=png&wxfrom=13&tp=wxpic)

## 复制算法

复制算法是通过两个内存空间，把一方存活的对象复制到另外一个内存空间上，然后再把自己的内存空间直接清理。标记后，此时情况如下：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

然后再把左边的存活对象复制到右边：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

复制完后，再清理自己的内存空间：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

右边的空间开始回收，再复制到坐标，如此往复。这样就可以让存活的对象紧密的靠在一起，腾出了连续的内存空间。

缺点就是空间少了一半，这少了一半的空间用于复制存活的对象。但是在实际过程中，大部分的对象的存活时间都非常短，也就是说这些对象都可以被回收的。

比如说左边有100M空间，但是只有1M的对象是存活的，那我们右边就不需要100M来存放这个1M的存活对象。

因此我们的新生代是分成3个内存块的：Eden空间、From Survivor空间、To Survivor空间，他们的默认比例是8:1:1。

也就是说，平常的时候有Eden+Survivor=90M可以使用，10M用于存放存活对象（假设100M空间）。这个算法在《[JVM堆内存分配机制](https://mp.weixin.qq.com/s?__biz=Mzg4MjU0OTM1OA==&mid=2247499346&idx=1&sn=fd0b892c55177cc78cc69be0ff7e84c1&chksm=cf564953f821c0454f31bd284748b26ae2aec1795cbf62625fddc64fae99c4687795de3d066e&token=737594716&lang=zh_CN&scene=21#wechat_redirect)》一文中的对象优先在Eden分配中提过了。

## 标记-整理

除了新生代，老年代的内存也要清理的，但是上面的算法并不适合老年代。

因为老年代对象的生命周期都比较长，那就不能像新生代一样仅浪费10%的内存空间，而是浪费一半的内存空间。

标记-整理与标记-清除都是先标记哪些对象存活，哪些对象可以回收，不同的是他并没有直接清理可回收的对象，而是先把存活的对象进行移动，这样存活的对象就紧密的靠在一起，最后才把垃圾回收掉。

回收前，存活对象是不连续的。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

回收中，存活对象是连续的。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

回收后，回收垃圾对象。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 性能与优化

由于每次GC，都会Stop The World，也就是说，此时虚拟机会把所有工作的线程都停掉，会给用户带来不良的体验及影响，所以我们要尽量减少GC的次数。

针对新生代，Minor GC触发的原因就是新生代的Eden区满了，所以为了减少Minor GC，我们新生代的内存空间不要太小。如果说我们给新生代的内存已经到达机器的极限了，那只能做集群了，把压力分担出去。

老年代的垃圾回收速度比新生代要慢10倍，所以我们也要尽量减少发生Full GC。

《[JVM堆内存分配机制](https://mp.weixin.qq.com/s?__biz=Mzg4MjU0OTM1OA==&mid=2247499346&idx=1&sn=fd0b892c55177cc78cc69be0ff7e84c1&chksm=cf564953f821c0454f31bd284748b26ae2aec1795cbf62625fddc64fae99c4687795de3d066e&token=737594716&lang=zh_CN&scene=21#wechat_redirect)》一文中我们提到了几种进入老年代的方式，我们看看这些是怎么处理的：

- 大对象直接进入老年代：这个没办法优化，总不能调大对象大小吧，那这些大对象在新生代的复制就会很慢了。
    
- 长期存活的对象将进入老年代:年龄的增加，是每次Minor GC的时候，所以我们可以减少Minor GC（这个方法上面提到了），这样年龄就不会一直增加。
    
- 动态年龄判断:有一个重要的条件就是在Survivor空间中相同年龄所有对象大小的总和大于Survivor空间的一半，所以要加大新生代的内存空间。
    
- 空间分配担保:这里面有一个条件是Minor GC后，Survivor空间放不下的就存放到老年代，为了让存活不到老年代，我们可以加大Survivor空间。
    

上面的方法，既有加大新生代的内存空间，也有加大Survivor空间，实时上，怎么优化，需要根据我们的实际场景来看，JVM的优化并没有银弹。

**好了，今天就到这儿吧，我是冰河，我们下期见~~**

_转自：segmentfault.com/a/1190000038339421_  

![](http://mmbiz.qpic.cn/mmbiz_png/2hHcUic5FEwEsQmOPMjJS0EZKCJ66T4VwM2ia7E7Dxj8pWyco8fSSt6EeUOTeRHM64TE2MGkxHibrXYibUALgGLncA/300?wx_fmt=png&wxfrom=19)

**冰河技术**

分享各种编程语言、开发技术、分布式与微服务架构、分布式数据库、分布式事务、云原生、大数据与云计算技术和渗透技术。另外，还会分享各种面试题和面试技巧。

673篇原创内容

公众号

点击上方卡片关注我  

精通JVM系列18

精通JVM系列 · 目录

上一篇JVM垃圾回收机制（建议收藏）下一篇JVM - CMS垃圾收集器（建议收藏）

阅读原文

阅读 2225

​

写留言

**留言 7**

- 冰河
    
    2022年2月8日
    
    赞4
    
    JVM走起，![[嘿哈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[嘿哈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[嘿哈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    置顶
    
- forever
    
    2022年2月8日
    
    赞2
    
    jvm表示，你越来越爱学习了
    
- You're ready(休养中，勿扰)
    
    2022年2月19日
    
    赞
    
    你好，大神，请问gc是多少秒执行一次。![[发呆]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)原理我看着都懂了，就是垃圾回收啥时候执行谁决定的
    
    冰河技术
    
    作者2022年2月19日
    
    赞1
    
    没有具体的时间，根据内存情况执行
    
- Sherlock
    
    2022年2月8日
    
    赞1
    
    尽量减少GC的次数
    
- 麦田花香
    
    2022年2月10日
    
    赞
    
    还是数你可能深入浅出讲技术
    
- 探险家Criminal
    
    2022年2月10日
    
    赞
    
    之前也看过gc的文章，一直没有理解空间不连续的意思。冰河大佬，这配图完美解释我的疑惑！![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/2hHcUic5FEwEsQmOPMjJS0EZKCJ66T4VwM2ia7E7Dxj8pWyco8fSSt6EeUOTeRHM64TE2MGkxHibrXYibUALgGLncA/300?wx_fmt=png&wxfrom=18)

冰河技术

1517

7

写留言

**留言 7**

- 冰河
    
    2022年2月8日
    
    赞4
    
    JVM走起，![[嘿哈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[嘿哈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[嘿哈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    置顶
    
- forever
    
    2022年2月8日
    
    赞2
    
    jvm表示，你越来越爱学习了
    
- You're ready(休养中，勿扰)
    
    2022年2月19日
    
    赞
    
    你好，大神，请问gc是多少秒执行一次。![[发呆]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)原理我看着都懂了，就是垃圾回收啥时候执行谁决定的
    
    冰河技术
    
    作者2022年2月19日
    
    赞1
    
    没有具体的时间，根据内存情况执行
    
- Sherlock
    
    2022年2月8日
    
    赞1
    
    尽量减少GC的次数
    
- 麦田花香
    
    2022年2月10日
    
    赞
    
    还是数你可能深入浅出讲技术
    
- 探险家Criminal
    
    2022年2月10日
    
    赞
    
    之前也看过gc的文章，一直没有理解空间不连续的意思。冰河大佬，这配图完美解释我的疑惑！![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    

已无更多数据
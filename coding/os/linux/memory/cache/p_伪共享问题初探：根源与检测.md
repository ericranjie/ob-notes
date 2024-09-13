# 

Linux内核之旅

 _2024年09月05日 12:01_ _陕西_

The following article is from 程栩的性能优化笔记 Author 程栩

[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM4zuU31ibxk2hXrLSO8s7BlzpZARHVyjr5o3AjCfNU7iabA/0)

**程栩的性能优化笔记**.

一个专注于性能的大厂程序员，分享包括但不限于计算机体系结构、性能优化、云原生的知识。

](https://mp.weixin.qq.com/s?__biz=MzI3NzA5MzUxNA==&mid=2664617923&idx=1&sn=8347bae1d2e6f9b3e9dd1bc38eaac0cd&chksm=f1b8d35bbc976781abc34cc5413242c76b1ca2f4d38c6422633364bec774734393a6ef3398ec&mpshare=1&scene=24&srcid=0905YN8kJOP9vbf3UCq5rYhr&sharer_shareinfo=0662d57d64f628efd8d2ea46e0b65cb7&sharer_shareinfo_first=0662d57d64f628efd8d2ea46e0b65cb7&key=daf9bdc5abc4e8d08f506593bdfe16f29044ad7bce4cd7b156546e863d1bef2d22d69f4a07c277d0733c28bb7f94a80f4a84567aab179aa246d37d64f48b8e2329276ba2c5d47377f2034250149c31a9455698b891f22202fbfbfd9c4860646ebf97df734937a152a6cff300a23f59fbb226f78d4d2889c5acc0fa104a88c22d&ascene=14&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=iMac+MacBookAir10%2C1+OSX+OSX+14.6.1+build(23G93)&version=13080710&nettype=WIFI&lang=en&session_us=gh_3e85a6de261f&countrycode=CN&fontScale=100&exportkey=n_ChQIAhIQf%2BeR4Rx8ocws3WzbvdCbmRKUAgIE97dBBAEAAAAAAG6gAvVApYsAAAAOpnltbLcz9gKNyK89dVj0stHs0CxOBjURHNZMGqgpLwKO8Ddp6%2BtfBFRWC7VAKU7OazXxwVwqSJHwx5mmwthkz32kj7sa6aJIFnyd3ZvPRhmnSqvbH6%2FMjA%2FU%2BLaVpSjS1ExhinPGEeyKTC2RIeOPbXi3cm6%2FTaE%2FTGJOi2A35UDf1dOsi7qp1ADOyIEq8cQxICYtE4%2FS4RpNYdurxhU8lmVFsWsmukXJfka3i2ii48g62lG8CcfFU8EPseB3W4rYZzxyb5Cw4Qo8HkT554Fpxx7mYVkqhRBrEnhfzgD4Z%2FxsxsFbzprdjjlHX%2FOARbBwgVFU%2FBDD3X1VmnM3Pg%3D%3D&acctmode=0&pass_ticket=Kj2gBxQtkyURGRqNICP0lnNgX1%2FSpg2FTnrrzQHZOL5mu6OhroSTyDUV%2Frznjw2Q&wx_header=0#)

### cache

在我们的程序访问内存的时候，如果每一次都去从主存里面获取数据的话，程序就会经常需要去等待访存，这会导致程序的运行比较缓慢：

![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/0A9N2rUnQ9P8FsqzAr4dO5OibATgAzicqIztXZtCibOsicaUwW6Hnes0lCGuIoZ3fIiaYggNTjaibcwVeZmwMCshlF3A/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

内存访问

为了加快程序的执行，出于一种简单的考虑：如果访问了A地址，那么下次极有可能去访问A或者A附近的地址，也就是所谓的局部性原理，我们设计了缓存(`cache`)。**缓存简单来说就是主存的一个子集，映射到主存上，每次基于规则去判断是否命中，如果命中的话就直接从缓存中取数据来加速程序的执行，一般我们将一个cache line作为一个最小区域**：

![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/0A9N2rUnQ9P8FsqzAr4dO5OibATgAzicqIsDmL9iat462noibiatsGibkP72zq3Mr73X6vqnkXEo8YpjagAu7H3nC1xQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

cache示意图

走缓存能够极大的加速内存的访问，我们用几张图来展示单`CPU`缓存（不考虑多级缓存）命中与不命中的流程：

#### 命中缓存

![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/0A9N2rUnQ9P8FsqzAr4dO5OibATgAzicqIVP9YsmcMh19BibKibRxs5AwS7KdN24p65SlhncicEkhibNrpmNOL9v0vzQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

hit

#### 缓存未命中

![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/0A9N2rUnQ9P8FsqzAr4dO5OibATgAzicqIH7Kic5wjKoKx6zhdplHiar1R1DU8gTf8z3lic163gH7tFLUwwkutz1bIw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

cache miss

![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/0A9N2rUnQ9P8FsqzAr4dO5OibATgAzicqI6KdBVcogOiciba46qCwayJnMB6UPVodTKicdcYicr036XVAM6TNqDTPh7g/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

直接访问

### 多线程与一致性协议

随着单核性能到达瓶颈，业界开始往多核性能方向发展。当我们编写单进程/多进程程序的时候，由于进程的隔离性，所以不太需要去考虑一致性问题；但是当涉及到多线程程序的时候，由于多个`CPU`可能会去访问同一个数据，就需要我们保证一致性。如果没有缓存的存在，每次我们都直接从内存中进行读取，那么每一次变化都是可以直接被另一个`CPU`感知到的：

![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/0A9N2rUnQ9P8FsqzAr4dO5OibATgAzicqIVoCy8XKfV0U2umG2ZWiahv0YYlv5vibhfVFX1VqPmxgtiaIicD4eEly7jg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

没有cache直接修改内存

![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/0A9N2rUnQ9P8FsqzAr4dO5OibATgAzicqIHaMk3xJxvDKdoViaFbun2fRGo3M21Eas0fM16tzzCQCoTMGMz5Q8icoA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

没有cache直接访问内存

但是当有了`cache`以后，由于加了一个中间层，如果一个线程在`CPU0`修改了变量A，而另一个线程在`CPU1`想基于A的值做进一步更新，如果还是使用原来的值，就会出现错误：

![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/0A9N2rUnQ9P8FsqzAr4dO5OibATgAzicqIxmRic2N0Siau81wCL0IHxnbna2XjNeg7NgnhI0QibVwFljpJq6paOEhPw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

cache不一致

为了解决这样的问题，我们在`cache`这里加上了一致性协议，比如`MSI`、`MESI`等，简单来说就是**可以让不同CPU上的cache能够感知到别的CPU针对同一个cache line的修改**：

![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/0A9N2rUnQ9P8FsqzAr4dO5OibATgAzicqIhvTvXXyiceymlSUPYvEe1RuK2Jic8iamI2MBVoSb82ny0fKEzDsnGT51w/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

一致性协议

### false sharing

由于我们将`cache line`作为一个最小区域，所以只要这个区域中有一个变量出现了变化，一致性协议就会要求其他的`CPU`进行同步；而如果这个区域中有多个变量在不同`CPU`持续更新，就会导致频繁的`cache miss`，从而降低性能：`看起来像是使用了cache，但是实际上cache经常失效，我们一般称之为伪共享(false sharing)`：

![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/0A9N2rUnQ9P8FsqzAr4dO5OibATgAzicqIsfbFY55gfib2icUz8h1mNiagkQweamWsIP7libNZ7NuKazFOTCmgz4J93A/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

false sharing

如果我们将同步区域变小是不是可以解决这个问题呢？并不能解决，因为只要存在失效就存在着同步，同步4字节也是一次同步。

### 测试程序

在实际的编程中，我们也会写出存在伪共享的代码，下面将通过一段代码的优化来介绍一些针对伪共享问题的处理。

测试代码：

![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/0A9N2rUnQ9P8FsqzAr4dO5OibATgAzicqIyh01iaFppl5DIsYqsxY3aLAymKXmNMwjkI3uJiaSO2uoS0hU1sIwBr6A/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

测试代码

构建与运行：

![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/0A9N2rUnQ9P8FsqzAr4dO5OibATgAzicqIxBkafpvCBibjaN5sd7nuhImtlIdcIYJj8zanfEF5ly1cTdxzDjWWaYQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

构建

### USE method

作为分析的第一步，笔者习惯先简单的了解系统上发生了什么（例如`USE method`），例如我们可以尝试从资源的角度去看。笔者习惯先用`top`、`free`来看一下系统大致的一些情况。

> USE method可以参见[USE方法：系统性能分析第一步](http://mp.weixin.qq.com/s?__biz=Mzg3NzkxMjA1MA==&mid=2247484424&idx=1&sn=56e30282b96a4dc1f6e83a6add4a561d&chksm=cf1a8e95f86d078367433f51b50e57800078a10e84bc5c674369979da75e2531de6f1f476c6e&scene=21#wechat_redirect)

执行`top`：

![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/0A9N2rUnQ9P8FsqzAr4dO5OibATgAzicqI0Ik4G3ibaqe1BWc5FF3kXulQbwrVMANCumxRUaDicCBfO45QVWh3hS9A/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

top

可以看到，我们开了四个现场，占了四个CPU，整体的执行时间是：

![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/0A9N2rUnQ9P8FsqzAr4dO5OibATgAzicqIuAjb4CxLpCuU9ypXwv6AfPkuwffBn35J2ELXuOw4iaqiboMnz7HM9nGg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

初始耗时

至此，我们可以简单的认为这里运行的时间都是在`CPU`上，或者说是一个**计算密集型**任务。

从`top`里看起来内存占用并不多，我们可以尝试用`perf`采集一些基础内存指标看看：

![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/0A9N2rUnQ9P8FsqzAr4dO5OibATgAzicqIanlaeHTceeMzBvNxHJ9VXSAhS2V8BPV1yRb2dicPz1icKLbhromLgObw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

perf stat查看访问相关

![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/0A9N2rUnQ9P8FsqzAr4dO5OibATgAzicqIQ0ucJ2okwibvME2L1jTJ4OG7JdQRFk2h1y7qAUZyx6xqSlXE0St3gOA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

访存相关指标

可以看到，程序的`ipc`偏低，一个时钟周期只能执行`0.31`条指令，存在明显的性能问题；同时我们还可以看到存在比较多的`cache miss`现象。**我们需要关注访存上可能存在的缓存失效现象。**

> 这里也可以使用TopDown方法来查看性能瓶颈。TopDown参考[Top-Down性能分析方法（原理篇）：揭秘代码运行瓶颈](http://mp.weixin.qq.com/s?__biz=Mzg3NzkxMjA1MA==&mid=2247484373&idx=1&sn=6b9a0776398e910a6d33f7a0f4ecbbcc&chksm=cf1a8948f86d005e4be710d42868e3235baeabbcb8369c2331130bf8e7acdc871c592d36e703&scene=21#wechat_redirect)

### perf record

针对这种`CPU`比较密集的程序，我们可以用`perf record`和`perf report`进行`Profiling`性能剖析来查看`CPU`上具体热点，了解进程在`CPU`上在干嘛：

![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/0A9N2rUnQ9P8FsqzAr4dO5OibATgAzicqIWibkq1YGvg1oOawibM7ShjJAicj0RTMXHrE0Va7kFjgr3qZibuTLI2GTnA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

perf record

收集以后用`perf report`先简单看一下

![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/0A9N2rUnQ9P8FsqzAr4dO5OibATgAzicqILJljGme2OY5yMdpdF8JdZJvqVPC2zwwTQgyjZJX5hWV1Gh9ZdPMib2Q/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

perf report

发现主要的热点都在四个线程上。**我们需要聚焦到四个线程的代码上。**

### perf c2c

从前面我们可以知道问题大概率是在四个子线程上，并且存在比较多的`cache miss`。因此我们可以尝试用`perf c2c`来查看是否存在伪共享情况：

![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/0A9N2rUnQ9P8FsqzAr4dO5OibATgAzicqIyF2KYLT4GQCTtILDcNWpuJR6iceiagEz680KFX3eW8b64TEHv73nkibZA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

perf c2c

开始收集：

![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/0A9N2rUnQ9P8FsqzAr4dO5OibATgAzicqI3LRqiaQX8Hfkl4W26OUR0ILBjgiaz88YCBnSLXNOHthREBwed5CuePbw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

perf c2c record

查看结果：
![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/0A9N2rUnQ9P8FsqzAr4dO5OibATgAzicqI1V6enj9gYp0HWOxbq3dPOhlf90XJEcw2icm7XI4s8Z5P7ibNhIVo5xUg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

perf c2c report

这里我们重点看`HITM`(`LLC Misses to Remote cache`)，表示的是由`cache`的修改导致远端的`cache`失效的占比，由`Load Remote HITM/Load LLC Misses`得到，值越大说明伪共享的问题越严重。**判断可能存在伪共享问题**。

再往下看，我们可以看到缓存行里面具体的数据：
![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/0A9N2rUnQ9P8FsqzAr4dO5OibATgAzicqIIxaz7WicGVhW6mZJPcESd9aJ8Xggtex8AawZd0VzfzOeFM8tvrra03A/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

cache line具体情况

我们去查看针对的代码地址：
![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/0A9N2rUnQ9P8FsqzAr4dO5OibATgAzicqItCm52VleCgefg8Wwv6AFXHqQ7dV9Ib4FEibU10j7uRuSicPYI5naYW1g/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

相关源代码

这里汇编的逻辑就是先从数组中把数据取到寄存器，然后操作以后再写回到对应的内存地址中去(`data[index].value++`)。这样我们就定位到了伪共享发生的位置，就是我们申明的这个结构体数组：
![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/0A9N2rUnQ9P8FsqzAr4dO5OibATgAzicqIbScUibf64UbCXBglAScdOYuMOMqMP3HdZicl0m54PQcp8fkOLFl4dXeA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

伪共享地址

### 原理解析与修复

知道了是哪里形成了伪共享以后，我们用一张图来直观的展示当前伪共享的过程：
![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/0A9N2rUnQ9P8FsqzAr4dO5OibATgAzicqI5R2Kbx8sjd2rAvjBs99YkGAcEibd0u9BGDoxpBCJvlUbkyNficaYI9IA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

false sharing图示

如果想减少伪共享，最简单的就是将这几个数据放到不同的`cache line`中：
![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/0A9N2rUnQ9P8FsqzAr4dO5OibATgAzicqIG5BibaqUGxN2z90W7D2d6AIWVia5oyod9nMolribKjvrRoJuDkibNuq39A/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

分开cache line

为了放到不同的`cache line`中，我们需要先查看系统上缓存行的大小：
![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/0A9N2rUnQ9P8FsqzAr4dO5OibATgAzicqIXe3bEvUQnuDLSCT119sM4MqX6oGdJicaTaNfNKWSmJqqfJF9oqtk3uw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

查看cache line大小
![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/0A9N2rUnQ9P8FsqzAr4dO5OibATgAzicqIOEp8FAicBT2fRunDbvGYicDbowbuTyzG314CY4p1Un5U1MxSYGZiaTCZA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

cache line = 64bytes

接着我们可以进行如下的尝试：
![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/0A9N2rUnQ9P8FsqzAr4dO5OibATgAzicqI2bdjhnACAibKdb5bGZnJeSyiazQyshkjW2iasonxJZpyh9kFh6ShDmJxQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

按照64字节对齐

简单来说，就是让数据按照64字节进行对齐，这样每一个进程都可以独享一个缓存行，从而减少伪共享导致的`cache miss`。在完成修改后，我们重新运行尝试：
![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/0A9N2rUnQ9P8FsqzAr4dO5OibATgAzicqIicQwibwLSUpia4ibyQIqxE2MgPwuXeuE8Vic2OQPichd7822NbpkJL9YNVoA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

修改后重新运行

查看`cache miss`情况：
![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/0A9N2rUnQ9P8FsqzAr4dO5OibATgAzicqIxNNKuuDNPicX8EicfkOtlbwaOMPUicQIYZ1RTI5GSndBCvY1gEHNSKCGw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

ipc明显提升

可以看到整体的`ipc`升到了2.32，时间变成了原来的1/6，说明我们的优化还是有一定效果的。

### 总结

本文主要简单介绍了伪共享产生的背景和原因，以及通过一个简单的例子展示了如何使用`perf`等工具解决伪共享问题。后续我们将继续深入的介绍`perf c2c`等工具的实现原理和使用细节。

> 本文所使用的测试代码均已上传到https://github.com/AshinZ/perf-workshop。

### 参考资料

- perf c2c(https://blog.csdn.net/aolitianya/article/details/138312017)  
    

Reads 1339

​

Comment

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/SeWfibBcBT0EibtIWVNvshnuWMN1AoJw3poFIsbpaIVyZibCCqwBUR21rcDfrQgoqYzaYNdS14IIXzvmzvibdDa5Rw/300?wx_fmt=png&wxfrom=18)

Linux内核之旅

181197

Comment

Comment

**Comment**

暂无留言
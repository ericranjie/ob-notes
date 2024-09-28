

Original Gracker Android Performance  _2022年06月30日 08:00_ _四川_

本文是 Systrace 线程 CPU 运行状态分析技巧系列的第三篇，本文主要讲了使用 Systrace 分析 CPU 状态时遇到的 **Sleep** 与 **Uninterruptible Sleep** 状态的原因排查方法与优化方法，这两个状态导致性能变差概率非常高，而且排查起来也比较费劲，网上也没有系统化的文档。

本系列的目的是通过 Systrace 这个工具，从另外一个角度来看待 Android 系统整体的运行，同时也从另外一个角度来对 Framework 进行学习。也许你看了很多讲 Framework 的文章，但是总是记不住代码，或者不清楚其运行的流程，也许从 Systrace 这个图形化的角度，你可以理解的更深入一些。Systrace 基础和实战系列大家可以在 Systrace 基础知识 - Systrace 预备知识[1] 或者 博客文章目录[2] 这里看到完整的目录

1. [Systrace 线程 CPU 运行状态分析技巧 - Runnable 篇](http://mp.weixin.qq.com/s?__biz=MzIwNTQxMjM5MA==&mid=2247485497&idx=1&sn=1f76cd615518322ba214ee41c1cd99b7&chksm=97300970a0478066a46867085c92251c42c91199b532de574773a78b409d528c7359ddcef053&scene=21#wechat_redirect)[3]
    
2. [Systrace 线程 CPU 运行状态分析技巧 - Running 篇](http://mp.weixin.qq.com/s?__biz=MzIwNTQxMjM5MA==&mid=2247485698&idx=1&sn=54bad9edbde9f2ee137d2719cc31943b&chksm=9730084ba047815d655a4602eac17ddaeae104cdd4cb9afb82fc8edfd8829338223f04c4f2fd&scene=21#wechat_redirect)[4]
    
3. Systrace 线程 CPU 运行状态分析技巧 - Sleep 和 Uninterruptible Sleep 篇[5]
    

# TASK_INTERUPTIBLE vs TASK_UNINTERRUPTIBLE

一个线程的状态不属于 Running 或者 Runnable 的时候，那就是 Sleep 状态了（严谨来说，还有其他状态，不过对性能分析来说不常见，比如 STOP、Trace 等）。

在 Linux 中的 Sleep 状态可以细分为 3 个状态：

- **TASK_INTERUPTIBLE** → 可中断
    
- **TASK_UNINTERRUPTIBLE** → 不可中断
    
- **TASK_KILLABLE** → 等同于 TASK_WAKEKILL | TASK_UNINTERRUPTIBLE
    

![Image](https://mmbiz.qpic.cn/mmbiz_png/HjA9ygCONWmMD0pDHlibbTfYYicfWKgCKMAWr7oPYic8AMVNXWVXoTqhbS8Makictgk5wicds8stv1YnGdORTbkrQpw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

"图 1: 性能之巅 2 CPU 优化"

在 Systrace/Perfetto 中，Sleep 状态指的是 Linux 中的 TASK_INTERUPTIBLE，trace 中的颜色为白色。Uninterruptible Sleep 指的是 Linux 中的 TASK_UNINTERRUPTIBLE，trace 中的颜色为橙色。

本质上他们都是处于睡眠状态，拿不到 CPU 的时间片，只有满足某些条件时才会拿到时间片，即变为 Runnable，随后是 Running。

TASK_INTERRUPTIBLE 与 TASK_UNINTERRUPTIBLE 本质上都是 Sleep，**区别在于前者是可以处理 Signal 而后者不能，即使是 Kill 类型的 Signal**。因此，除非是拿到自己等待的资源之外，没有其他方法可以唤醒它们。TASK_WAKEKILL 是指可以接受 Kill 类型的 Signal 的 TASK_UNINTERRUPTIBLE。

Android 中的 Looper、Java/Native 锁等待都属于 TAKS_INTERRUPTIBLE，因为他们可以被其他进程唤醒，应该说绝大部分的程序都处于 TAKS_INTERRUPTIBLE 状态，即 Sleep 状态。看看 Systrace 中的一大片进程的白色状态就知道了（trace 中表现为白色块），它们绝大部分时间都是在 Runnning 跟 Sleep 状态之间转换，零星会看到几个 Runnable 或者 UninterruptibleSleep，即蓝色跟橙色。

## TASK_UNINTERRUPTIBLE 作用

似乎看来 TASK_INTERUPTIBLE 就可以了，那为什么还要有 TASK_UNINTERRUPTIBLE 状态呢？

中断来源有两个，一个是硬件，另一个就是软件。硬件中断是外围控制芯片直接向 CPU 发送了中断信号，被 CPU 捕获并调用了对应的硬件处理函数。软件中断，前面说的 Signal、驱动程序里的 softirq 机制，主要用来在软件层面触发执行中断处理程序，也可以用作进程间通讯机制。

一个进程可以随时处理软中断或者硬件中断，他们的执行是在当前进程的上下文上，意味着共享进程的堆栈。但是在某种情况下，程序不希望有任何打扰，它就想等待自己所等待的事情执行完成。比如与硬件驱动打交道的流程，如 IO 等待、网络操作。这是为了保护这段逻辑不会被其他事情所干扰，`避免它进入不可控的状态`。

Linux 处理硬件调度的时候也会临时关闭中断控制器、调度的时候会临时关闭抢占功能，本质上为了 `防止程序流程进入不可控的状态`。这类状态本身执行时间非常短，但系统出异常、运行压力较大的时候可能会影响到性能。

https://elixir.bootlin.com/linux/latest/ident/TASK_UNINTERRUPTIBLE[6]

可以看到内核中使用此状态的情况，典型的有 Swap 读数据、信号量机制、mutex 锁、内存慢路径回收等场景。

## 分析时候的注意点

首先要认识到 TASK_INTERUPTIBLE、TASK_UNINTERRUPTIBLE 状态的出现是正常的，但是如果这些这些状态的累计占比达到了一定程度，就要引起注意了。特别是在关键操作路径上这类状态的占比较多的时候，需要排查原因之后做相应的优化。分析问题以及做优化的时候需要牢牢把握两个关键点，它类似于内功心法一样:

1. **原因的排查方法**
    
2. **优化方法论**
    

你需要知道是什么原因导致了这次睡眠，是主动的还是被动的？如果是主动的，通过走读代码调查是否是正常的逻辑。如果是被动的，故事的源头是什么？这需要你对系统有足够多的认识，以及分析问题的经验，你需要经常看案例以增强自己的知识。

以下把 TASK_INTERUPTIBLE 称之为 Sleep，TASK_UNINTERRUPTIBLE 称之为 UninterruptibleSleep，目的是与 Systrac 中的用词保持一致。

初期分析 Sleep 与 UninterruptibleSleep 状态的经验不足时你会感到困惑，这种困惑主要是来自于对系统的不了解。你需要读大量的框架层、内核层的代码才能从 Trace 中找出蛛丝马迹。目前并没有一种 Trace 工具能把整个逻辑链路描述的很清楚，而且他们有时候还有不准的时候，比如 Systrace 中的 wakeup_from 信息。只有广泛的系统运行原理做为支持储备，再结合 Trace 工具分析问题，才能做到准确定位问题根因。否则就是我经常说的「性能优化流氓」，你说什么是什么，别人也没法证伪。反复折磨测试同学复测，没测出来之后，这个问题也就不了了之了。

本文没办法列举完所有状态的原因，因此只能列举最为常见的类型，以及典型的实际案例。更重要的是，你需要掌握诊断方法，并结合源代码来定位问题。

## Trace 中的可视化效果
![[Pasted image 20240928120519.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Pefetto 中支持显示的状态
![[Pasted image 20240928120526.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Systrace 支持显示的状态

# Sleep 状态分析
![[Pasted image 20240928120533.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图 1: UIThread 等待 RenderThread
![[Pasted image 20240928120542.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图 2: Binder 调用等待

## 诊断方法

### 通过 `wakeup from tid: ***`查看唤醒线程

Sleep 最常见的有图 1（UIThread 与 RenderThread 同步）的情况与图 2（Binder 调用）的情况。Sleep 状态一般是由程序主动等待某个事件的发生而造成的，比如锁等待，因此它有个比较明确的唤醒源。比如图 1，UIThread 等待的是 RenderThread，你可以通过阅读代码来了解这种多线程之间的交互关系。虽然最直接，但是对开发者的要求非常高，因为这需要你熟读图形栈的代码。这可不是一般的难度，是追求的目标，但不具备普适性。

更简单的方法是通过所谓的 `wakeup from tid: ***` 来调查线程之间的交互关系。从前面的 Runnable 文章[7] 中讲过，任何线程进入 Running 之前会先进入到 Runnable 状态，由此再转换成 Running。从 Sleep 状态切换到 Running，必然也要经过 Runnable。

进入到 Runnable 有两种方式，一种是 Running 中的程序被抢占了，暂时进入到 Runnable。还有一种是由另外一个线程将此线程（处于 Sleep 的线程）变成了 Runnable。

我们在调查 Sleep 唤醒线程关系的时候，应用到的原理是第二种情况。在 Systrace 中这种是被 `wakeup from tid: ***` 信息所呈现。线程被抢占之后变成 Runnable，在 Systrace 中是被 `Running Instead` 呈现。
![[Pasted image 20240928120610.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

需要特别注意的是 `wakeupfrom` 这个有时候不准，原因是跟具体的 tracepoint 类型有关。分析的时候要注意甄别，不要一味地相信这个数据是对的。

### 其他方法

1. Simpleperf 还原代码执行流
    
2. 在 Systrace 寻找时间点对齐的事件
    

方法 1 适合用来看程序到底在执行什么操作进入到这种状态，是 IO 还是锁等待？球里连载 Simpleperf 工具的使用方法，其中「Simpleperf 分析篇 (1): 使用 Firefox Profiler 可视分析 Simpleperf 数据[8]」介绍了可以按时间顺序看函数调用的可视化方法。其他使用也会陆续更新，直接搜关键字即可。

方法 2 是个比较笨的方法，但有时候也可以通过它找到蛛丝马迹，不过缺点是错误率比较高。

## 耗时过长的常见原因

- **Binder 操作** → 通过打开 Binder 对应的 trace，可方便地观察到调用到远端的 Binder 执行线程。如果 Binder 耗时长，要分析远端的 Binder 执行情况，是否是锁竞争？得不到 CPU 时间片？要具体问题具体分析
    
- **Java\futex 锁竞争等待** → 最常见也是最容易引起性能问题，当负载较高时候特别容易出现，特别是在 SystemServer 进程中。这是 Binder 多线程并行化或抢占公共资源导致的弊端。
    
- **主动等待** → 线程主动进入 Sleep 状态，等待其它线程的唤醒，比如等待信号量的释放。优化建议：需要看代码逻辑分析等待是否合理，不合理就要优化掉。
    
- **等待 GPU 执行完毕** → 等 GPU 任务执行完毕，Trace 中可以看到等 GPU fence 时间。常见的原因有渲染任务过重、 GPU 能力弱、GPU 频率低等。优化建议：提升 GPU 频率、降低渲染任务复杂度，比如精简 Shader、降低渲染分辨率、降低 Texture 画质等。
    

# UninterruptibleSleep 状态分析

## 诊断方法

本质上 UninterruptibleSleep 也是一种 Sleep，因此分析 Sleep 状态时用到的方法也是通用的。不过此状态有两个特殊点与 Sleep 不同，因此在此特别说明。

1. **UninterruptibleSleep 分为 IOWait 与 Non-IOWait**
    
2. **UninterruptibleSleep 有 Block reason**
    

### UninterruptibleSleep 分为 IOWait 与 Non-IOWait

IO 等待好理解，就是程序执行了 IO 操作。最简单的，程序如果没法从 PageCache 缓存里快速拿到数据，那就要与设备进行 IO 操作。CPU 内部缓存的访问速度是最快的，其次是内存，最后是磁盘。它们之间的延迟差异是数量级差异，因此系统越是从磁盘中读取数据，对整体性能的影响就越大。

非 IO 等待主要是指内核级别的锁等待，或者驱动程序中人为设置的等待。Linux 内核中某些路径是热点区域，因此不得不拿锁来进行保护。比如 Binder 驱动，当负载大到一定程度，Binder 的内部的锁竞争导致的性能瓶颈就会呈现出来。

### Block Reason

谷歌的 Riley Andrews(riandrews@google.com) 15 年左右往内核里提交了一个 tracepoint 补丁，用于记录当发生 UninterruptibleSleep 的时候是否是 IO 等待、调用函数等信息。Systrace 中展示的 IOWait 与 BlockReason，就是通过解析这条 tracepoint 而来的。这条代码提交的介绍如下（由于这笔提交未合入到 Linux 上游主线，因此要注意你用的内核是否单独带了此补丁）:

```c
sched: add sched blocked tracepoint which dumps out context of sleep.Decare war on uninterruptible sleep. Add a tracepoint whichwalks the kernel stack and dumps the first non-scheduler functioncalled before the scheduler is invoked.Change-Id: [I19e965d5206329360a92cbfe2afcc8c30f65c229](https://android-review.googlesource.com/#/q/I19e965d5206329360a92cbfe2afcc8c30f65c229 "I19e965d5206329360a92cbfe2afcc8c30f65c229")Signed-off-by: Riley Andrews [riandrews@google.com](mailto:riandrews@google.com "riandrews@google.com")
```

在 ftrace（Systrace 使用的数据抓取机制） 中的被记录为

```c
sched_blocked_reason: pid=30235 iowait=0 caller=get_user_pages_fast+0x34/0x70
```

这句话被 Systrace 可视化的效果为:
![[Pasted image 20240928120629.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

主线程中有一段 Uninterruptible Sleep 状态，它的 BlockReason 是 `get_user_pages_fast`。它是一个 Linux 内核中函数的名字，代表着是线程是被它切换到了 UninterruptibleSleep 状态。为了查看具体的原因，需要查看这个函数的具体实现。

```c
/** * get_user_pages_fast() - pin user pages in memory * @start:      starting user address * @nr_pages:   number of pages from start to pin * @gup_flags:  flags modifying pin behaviour * @pages:      array that receives pointers to the pages pinned. *              Should be at least nr_pages long. * * Attempt to pin user pages in memory without taking mm->mmap_lock. * If not successful, it will fall back to taking the lock and * calling get_user_pages(). * * Returns number of pages pinned. This may be fewer than the number requested. * If nr_pages is 0 or negative, returns 0. If no pages were pinned, returns * -errno. */int get_user_pages_fast(unsigned long start, int nr_pages,      unsigned int gup_flags, struct page **pages){  if (!is_valid_gup_flags(gup_flags))    return -EINVAL;  /*   * The caller may or may not have explicitly set FOLL_GET; either way is   * OK. However, internally (within mm/gup.c), gup fast variants must set   * FOLL_GET, because gup fast is always a "pin with a +1 page refcount"   * request.   */  gup_flags |= FOLL_GET;  return internal_get_user_pages_fast(start, nr_pages, gup_flags, pages);}EXPORT_SYMBOL_GPL(get_user_pages_fast);
```

从函数解释上可以看到，函数首先是通过无锁的方式 pin 应用侧的 pages，如果失败的时候不得不尝试持锁后走慢速执行路径。此时，无法持锁的时候那就要等待了，直到先前持锁的人释放锁。那之前被谁持有了呢？这时候可以利用之前介绍的 Sleep 诊断方法，如下图。
![[Pasted image 20240928120704.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

UninterruptibleSleep 状态相比 Sleep 有点复杂，因为它涉及到 Linux 内部的实现。可能是内核本身的机制有问题，也有可能是应用层使用不对，因此要联合上层的行为综合诊断才行。毕竟内核也不是万能的，它也有自己的能力边界，当应用层的使用超过其边界的时候，就会出现影响性能的现象。

## IOWait 常见原因与优化方法

### 1. **主动 IO 操作**

- 程序进行频繁、大量的读或者写 IO 操作，这是最常见的情况。
    
- 多个应用同时下发 IO 操作，导致器件的压力较大。同时执行的程序多的时候 IO 负载高的可能性也大。
    
- 器件本身的 IO 性能较差，可通过 IO Benchmark 来进行排查。常见的原因有磁盘碎片化、器件老化、剩余空间较少（越是低端机越明显）、读放大、写放大等等。
    
- 文件系统特性，比如有些文件系统的内部操作会表现为 IO 等待。
    
- 开启 Swap 机制的内核下，数据从 Swap 中读取。
    

**优化方法**

- 调优 Readahead 机制
    
- 指定文件到 PageCache，即 PinFile 机制
    
- 调整 PageCache 回收策略
    
- 调优清理垃圾文件策略
    

### 2. 低内存导致的 IO 变多

内存是个非常有意思的东西，由于它的速度比磁盘快，因此 OS 设计者们把内存当做磁盘的缓存，通过它来避免了部分 IO 操作的请求，非常有效的提升了整体 IO 性能。有两个极端情况，当系统内存特别大的时候，绝大部分操作都可以在内存中执行，此时整体 IO 性能会非常好。当系统内存特别低，以至于没办法缓存 IO 数据的时候，几乎所有的 IO 操作都直接与器件打交道，这时候整体性能相比内存多的时候而言是非常差的。

所以系统中的内存较少的时候 IO 等待的概率也会变高。所以，这个问题就变成了如何让系统中有足够多的内存？如何调节磁盘缓存的淘汰算法？

**优化方法**

- 关键路径上减少 IO 操作
    
- 通过 Readahead 机制读数据
    
- 将热点数据尽量聚集在一起，使被 Readahead 机制命中的概率高
    
- 最后一个老生常谈的，减少大量的内存分配、内存浪费等操作
    

系统中的内存是被各个进程所共用。当 app 只考虑自己，肆无忌惮的使用计算资源，必然会影响到其他程序。这时候系统还是会回来压制你，到头来亏损的还是自己。不过能想到这一步的开发者比较少，也不现实。明文化的执行系统约定，可能是个终极解决方案。

## Non-IOWait 常见原因

- **低内存导致等待** → 低内存的时候要回收其他程序或者缓存上的内存。
    
- **Binder 等待** → 有大量 Binder 操作的时候出现概率较高。
    
- **各种各样的内核锁，不胜枚举**。结合「诊断方法」来分析。
    

## 系统调度与 UninterruptibleSleep 耦合的问题

当线程处于 UninterruptibleSleep 非 IO 等待状态（即内核锁），而持有该锁的其他线程因 CPU 调度原因，较长时间处于 Runnable 状态。这时候就出现了有意思的现象，即使被等待的线程处于高优先级，它的依赖方没有被调度器及时的识别到，即使是非常短的锁持有，也会出现较长时间的等待。

规避或者彻底解决这类问题都是件比较难的事情，不同厂家实现了不同的解决方案，也是比较考虑厂家技术能力的一个问题。

# 附录

## Linux 线程状态释义

|线程状态|描述|
|---|---|
|S|SLEEPING|
|R、R+|RUNNABLE|
|D|UNINTR_SLEEP|
|T|STOPPED|
|t|DEBUG|
|Z|ZOMBIE|
|X|EXIT_DEAD|
|x|TASK_DEAD|
|K|WAKE_KILL|
|W|WAKING|
|D|K|
|D|W|

## 案例: 从 Swap 读取数据时的等待
![[Pasted image 20240928120718.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 案例: 同进程的多个线程进行 mmap
![[Pasted image 20240928120724.png]]
![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

共享同一个 mm_struct 的线程同时执行 mmap() 系统调用进行 vma 分配时发生锁竞争。

mmap_write_lock_killable() 与 mmap_write_unlock() 包起来的区域就是由锁受保护的区域。

```c
unsigned long vm_mmap_pgoff(struct file *file, unsigned long addr,  unsigned long len, unsigned long prot,  unsigned long flag, unsigned long pgoff){  unsigned long ret;  struct mm_struct *mm = current->mm;  unsigned long populate;  LIST_HEAD(uf);  ret = security_mmap_file(file, prot, flag);  if (!ret) {    if (mmap_write_lock_killable(mm))      return -EINTR;    ret = do_mmap(file, addr, len, prot, flag, pgoff, &populate,            &uf);    mmap_write_unlock(mm);    userfaultfd_unmap_complete(mm, &uf);    if (populate)      mm_populate(ret, populate);  }  return ret;}
```

### 参考资料

[1]

Systrace 基础知识 - Systrace 预备知识: _https://www.androidperformance.com/2019/07/23/Android-Systrace-Pre/_

[2]

博客文章目录: _https://www.androidperformance.com/2019/12/01/BlogMap/_

[3]

Systrace 线程 CPU 运行状态分析技巧 - Runnable 篇: _https://www.androidperformance.com/2022/01/21/android-systrace-cpu-state-runnable/_

[4]

Systrace 线程 CPU 运行状态分析技巧 - Running 篇: _https://www.androidperformance.com/2022/03/13/android-systrace-cpu-state-running/_

[5]

Systrace 线程 CPU 运行状态分析技巧 - Sleep 和 Uninterruptible Sleep 篇: _https://www.androidperformance.com/2022/03/13/android-systrace-cpu-state-sleep/_

[6]

https://elixir.bootlin.com/linux/latest/ident/TASK_UNINTERRUPTIBLE: _https://elixir.bootlin.com/linux/latest/ident/TASK_UNINTERRUPTIBLE_

[7]

Runnable 文章: _https://articles.zsxq.com/id_wzkkiwop5pgm.html_

[8]

Simpleperf 分析篇 (1): 使用 Firefox Profiler 可视分析 Simpleperf 数据: _https://articles.zsxq.com/id_xc0a7uwmsf3z.html_

[Read more](https://www.androidperformance.com/2022/03/13/android-systrace-cpu-state-sleep)

​

![](https://mp.weixin.qq.com/rr?timestamp=1727495968&src=11&ver=1&signature=QHdCCOZg7j*CwPA*-Qomo7Vdu8z6VJzhhnhnuvi00mGVN0OJCXaeo0lNzXg1ulAAd89A8rUgkjXPbxwBzhe0cTtWpS6h-BIdrOX-Lw9meHY=)

Scan to Follow
#

CPP 与系统软件联盟

_2022 年 01 月 20 日 11:40_

**点击蓝字**

**关注我们**

编者按：本文摘录自「全球 C++及系统软件技术大会」Boolan 资深咨询师冉昕老师的专题演讲。

Scott Meyers 曾说到过，如果你不在乎性能，为什么要在 C++这里，而不去隔壁的 Python room 呢？

今天我们就从“低延迟的概述”、“低延迟系统调整”、“低延迟系统编译选项”、“低延迟软件设计与编码”四个部分来聊聊低延迟场景下的性能优化实践。

1.

**低延迟概述**

**低延迟场景**

很多系统都会关注延迟，比如：电信系统、游戏行业、音视频解码，或者一些金融系统。这里我们就以金融场景为例。

在程序化交易系统下，为什么需要关注低延迟？

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/YQ55UxynrohnO3qrP9XOJ93MvJcn1yLAEIBiaKg0WeicX3ZtIetcR7vW4zboH8QYILQZ0kyiaSXcicvULkTqf1CECA/640?wx_fmt=jpeg&wxfrom=13&tp=wxpic)

程序化交易系统是接收市场的行情再去进行运算，然后发出交易信号。发出交易信号越早，就越可能挣到钱，如果晚了，钱都被别人挣了，自己可能就会亏钱。所以在这种场景下，**低延迟是第一需求，\*\***不会追求吞吐量**。交易所都有流速权，即每秒的报单速度是有限的，不允许做很大的吞吐，所以金融对低延迟的要求是比较高的，也**不在意资源利用率\*\*。因为我们的  CPU 会进行绑核，绑核会让 CPU 处于 busy looping，让它的占有率达到 100%，那么资源利用率就没有任何参考价值。

当然，程序化交易系统资源都是超配的，比如内存、硬盘，虽然 CPU 没有超配这一说，但尽可能配最好的。

**低延迟优化特点**

**![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)**

常用的性能优化就是做一些压力测试、关注一下 QPS、看看系统负载需不需要内存、使用率怎么样，用 perf 工具去找出程序的热点。“不成熟的优化是万恶之源。”Profile 就是一个非常好的优化工具。

但对低延迟性能优化来说，Profile 可能就不是特别关键了。低延迟系统有追求延迟的线程，也有不追求延迟的、没那么 critical 的线程。critical 线程在我们系统整个代码量中并不是特别大，这种情况下用 Profile 的数据是不准的，Profile 工具是采样的，延迟很低就更难采到。所以在系统、设计、编码的层次上需要提前考虑低延迟，也会提前规划好哪些代码要走 critical path 并对它进行优化。还要测试各单元的延迟，这个延迟可能是一个 tick-to-trade，即从行情开始到最后交易完成的整个系统的延迟，也可能是各个模块、各个 function、各个语句块、甚至各条语句的延迟，最后再去优化 critical path。

**常见操作时延**

我们来看一组以前的操作数据。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/YQ55UxynrohnO3qrP9XOJ93MvJcn1yLAEpJAsdCK0YGVzoDiaeTRlBibkicURibmfhibIaXoqGhuTVrH6TmsBodtiaMw/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

从最底下开始看，Disk read 一旦涉及到磁盘就和延迟无关了，这个结果显然是不允许的。Context switch 是系统调用，在内核中会做很多操作，线程被调度出去再被调度回来，本身切换过程的耗时就非常大，再加上运行其他的线程，cache 可能都已经冷了，这里的其他开销可能就更难衡量。假如是 10K 的 CPU cycle，即便是 10GHz 的超频服务器耗时也需要一个微秒，这在低延迟系统里已经是非常大的开销。这里的异常抛出和 cache 处理占的时间也比较长，如果代码进了内核态再切换回来，这个延迟也是非常可观的。

Allocation deallocation pair，这个延迟是指用 malloc/free  或 delete，申请内存的过程中会有内存管理器这一层，比如 Glibc ptmalloc，大多数情况下是不会系统调用，但它本身开销也很可观。如果你申请的内存本身 core 比较大，直接调用 mindmap，或者 Glibc 的缓存里没有 free 内存去分配，就会走到 kernel 再回来，这个时间开销就更大了。

内存读取包括主内存读取、NUMA 去读取另外一个节点的数据，性能开销都是很大的。Mutex 在低延迟代码里也基本不会用。至于函数调用，不可能一个都不用，但可以用 inline 来减少函数调用。除了普通的函数调用，还有多态调用，即 vptr、vtable。Div 操作是 CPU 不喜欢的。

CPU 都是流水线执行的，"wrong" branch of "if"和“right" branch of "if"，就是 CPU 执行到一个 if-else 时会自己去猜，如果猜对了，就几乎可以忽略，如果猜错了，代价就比较惨。

2.

**低延迟系统调整**

**硬件&系统**

首先既对处理器的核数有要求，同时也对单核的频率有要求，但这两点是矛盾的。想要一个核数又多、频率又高的，就要用到超频服务器，执行效率越高越好，不需要虚拟化功能。内存也要充足。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

超线程一般是关闭的，同一个程序在开超线程和不开超线程的机器跑的话，肯定是不开超线程的更快。另外，如果进行内核绑定，Critical thread 会独占一个核，如果绑定一个开了超线程的核就相当于绑定了同一个核，或者是一个核不用扔在那儿，这是没有意义的。

操作系统是 64 位的 Linux，一般是  CentOS 或是  RHEL，最小化安装，toolchain 升级，因为默认自带的可能是比较老的 GCC，我一般都习惯升级到 9 或 10。

最小化安装还有一个比较有意思的点，因为我个人是坚定的  Emacs 党，不喜欢 vim，但 Emacs 会默认安装一些图形化插件，所以要在你的生产机器上装 Emacs 的话就要装 Emacs-nw 版本。Rtkernel 看起来好像和低延迟实施有关系，但实际上它是保证一种硬抢占的内核 patch，这个对我们来说是完全不需要的。RHEL 一般都可以照着 Tuning guide 去对系统进行调整，如果是买服务器或超频服务器，vendor 也会有 guide，可以斟酌一下要不要打开。

**CPU 相关优化**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

CPU 优化最核心的就是要让 Critical 线程独占 CPU，不能被打断，要求极致的低延迟，而普通线程就无所谓。我们要做的就是先把一定数量的核 isolate，这样操作系统就不会把任何的用户态线程再调度到这个核上，然后再做 thread bonding，把 Critical thread 绑定到这些 isolate 的核里去，这样就保证了 thread 可以独占这个核。也可以设置  scheduler，对于 Critical thread 我们一般都是设置 FIFO 这种实时的优先级调度策略，对于普通的线程用 default(CFS)  就可以。

**中断**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

当遇到 kernel 中断、时钟中断或 workqueue 等情况时还是可能会侵占 CPU 时间，可以把中断的 balance 关掉，设置中断 affinity 到非 isolate 核心，这样可以让中断对你的影响尽可能地小。这里要提一下，时钟中断是不可能完全关闭的，除非改内核。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

网卡要绑定到相应的 slot，一般和 Critical 线程绑定的 slot 是同一个。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

内存优化也要避免进入内核态，一方面是分配的时候可能进入， 另一方面是触发 fault 的时候。

fault，对于 Linux 操作系统来说，在内核层面上是不区分线程和进程的，都是用 task_struct 来表示线程。进程和线程唯一的区别就是进程的 tid 和 pid 是相等的，因为一个进程的内存是共享的，所以每一个 task_struct 里其实都有一个 mm_struct 指向同一个内存 object，这个内存的 object 分各个 area，每个 area 都标识了这块内存的虚拟地址是否合法。

我们平时写代码的时候，不考虑 Glibc 有缓存内存，假如 malloc 或 new 一块内存的请求到了操作系统，那么操作系统做的一件事就是在刚才所说的 mm_struct 里的  vm_area 里划分并标识一块合法的区域，这些操作都是在虚拟地址层面上，并没有真实的物理地址层面，然后做完这个操作以后它就返回了。但实际上虚拟地址和物理地址之间需要有一个映射，即虚拟页面。假如说是一个 4K 页，和一个物理 4K 页之间的映射关系没有建立，那什么时候建立呢？当 CPU 访问这块内存的时候就会触发一个 fault，因为 CPU 在 MMU 单元通过虚拟地址去找这块物理地址找不到，这个 fault 交到操作系统，操作系统再进行处理，这相当于是一种操作系统 lazy 处理的模式。但这些过程都是需要内核深度参与的，一旦出现要在内核态做这么多事情的情况，和低延迟就差得很远了。

major fault 是指当内存不够时，内存可能被交换到磁盘，再用到这块内存时再从磁盘交换回去 。major fault 比较好解决，一种是禁用 swap 分区 ，而且内存比较充足的话一般也不会触发，我们在系统里还有一个 mlock 调用，mlock 调用以后就可以阻止你这个进程的内存被 swap 到硬盘上。

minor fault 就比较难搞了，这个过程中可能有多个手段，但也不保证能百分百把它消除掉。一种是用  huge page。因为 fault 是以页面为单位的，huge page 可以把一个页从 4K 变成 2M，这样的好处是页面 fault 的几率就明显会小很多。另外，虚拟地址页面去找物理地址页面需要 CPU 的 MMU 单元去找，它会优先去找 TLB。TLB 相当于映射的缓存，你可以认为它是一个哈希，如果找不到就会到页表里面一级一级去找，可能是两级，可能是三级，TLB 可以大大的提升这个这个寻找的时间。用了 huge page 以后，页表总体更少了，TLB miss 几率也就更低了。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

对于 NUMA 来说，尽可能要它访问自己的线程，不要跨 slot 访问。NUMA 有多个内存的分配策略，一般默认的就是  localalloc，让这个槽的线程分配的内存在 local 分配，不要到 remote 分配。还有一种是 Interleave，即平均在几个 slot 里面分配，这种是我们不想要的。

prefault 是很大的一个话题，就是可以分配内存，但是分配了之后要想办法在真正使用之前先触发它的 minor fault。这有多个层面去解决，一是可以 hack 内存管理器，可以自己写，也可以优化 ptmalloc，当然如果有第三方的内存管理器可能会更好地解决这个问题。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**网络**

现在  TCP 延迟较高基本是业界共识，大家都在想怎么去解决这个问题，现在有趋势就是交易这方面也往 UDP 转，尤其是行情部分会越来越多地转到 UDP。无论怎么优化，你的缓冲区也好，中断也好，还是会有硬件的中断触发，陷入内核态，只要你的协议栈在内核态，性能就不会很好，所以这种情况下就要用用户态的协议栈。还有一些  FPGA 解决方案的，一般是券商或期货公司在用。

3.

**低延迟系统编译选项**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

先说一下编译器选择。Linux 主流的编译器无非就是 GCC、Clang、ICC。ICC 一直作为性能标杆的存在，但 ICC 在 C++ 的标准支持是比较落后的。ICC 做的比较好的是 CPU patch，它会针对不同的 CPU 的指令集生成很多份代码，运行的时候会根据具体的指令去选择最优的 function。

我们用的更多的还是 GCC，GCC 现在讨论最多的就是-O2 和-O3。这个在选择上没有标准答案，我们就来看看 -O3 比 -O2 多了什么吧。

首先，-finline-functions 除了代码里写了 inline，或者用 GCC 的扩展 always_inline，GCC-O2 还会默认开一个 inline call_once function，还有一个 inline_function 我个人觉得是很有用的。GCC 10 开始就已经 include -O2 了，也会针对它不同的优化，不断地把 -O2 move 到 -O3。但-O3  不一定整个项目都能用，可以只针对某个 function 或某个 file 来打开。

-floop-unroll-and-jam 是指如果有多层循环的话会把外层循环展开。

-fipa-cp-clone 是指如果有多个参数，其中一个传了常数的话，它有可能把这个 function clone 两份，其中一份会去做一些常量展开、常量传播，这个有时能用得上。也许你会说“我代码写得比较好，我用  (const expression)  之类也能达到相似效果”，但是你不能保证所有人写代码或第三方库都能做到这一点。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这张图中上面两段代码都不是 cache friendly 的代码，都是比较低效的内存访问模式，但如果开了  -floop-interchange ，编译器就帮我们优化到我们想要的样子，cache friendly 就没有问题了。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

可能有的编码规范上说不要在 foo 里面加 branch，但这段代码中看起来每个 foo 里都加了 branch，其实如果开了编译选项以后，GCC 会自动把 if 放到 foo 外面，如果这个 foo 里面有一条赋值语句且和 foo 无关的话，也会被移到外面。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E "幻灯片 18.jpg")

loop distribution 是目前的热点话题，distribution 是把一个循环展开成两个，但在这个例子中，展开成两个看起来是反优化的：a[i] = b[i]， b[i] = 0，在 cache 里肯定是最快的，那为什么要拆开呢？loop distribution pattern 能发现这个代码有一定的 pattern，上面用  memcpy() 搞定，下面用 memset() 搞定。

其他情况比如 a[i] = b[i]，但对 b[i] + 1 有一些依赖，那对流水线是不友好的，这种情况也有可能拆开。

当然还有  loop fusion 这种相反的情况，本来写的两个循环，它发现合并了以后更有利于 cache friendly，可能就会做合并，但在 GCC 里没有做合并这个选项，我们自己写代码的时候需要注意一下。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

loop-vectorize 是我认为最关键的一个，这里源自 Stack Overflow 的一个问题：在执行过程中有没有 sort，性能差异是巨大的，为什么？

有了 sort 以后，CPU Branch Prediction 更好了，成功率很高，性能就很高。GCC-O3 比 -O2 更慢，核心原因是最初 cmov 指令在老的处理器架构上比较慢，而现代新的编译器都用 cmov 做优化，不再用条件 jump 语句了，执行效率非常高。有了 cmov 以后就没有分支了，也就不存在 sort 和 unsort 的区别了，也不存在 -O3 比 -O2 更慢的情况。

如果我们把  sum += data[c] 改成 sum += data[c] + data[c]，那么无论 GCC 还是 Clang 都不会再用 cmov，而是用传统条件跳转的方式，这种情况下性能就又有差别了，loop-vectorize 就可以起作用了。

如果启用了 loop vectorize，它就会用 SSE 指令集 去优化这个循环的过程，也就是说，这个性能和 cmov 版本相比不会差，甚至是更好，所以说 loop vectorize 很多时候是非常有用的。如果是针对 -march=native，让编译器针对当前的处理器架构做一些优化的话，如果你的处理器支持 AVX 2 或是更高的 AVX-512 指令集，那它可以给你做更进一步的优化，性能提升得更大。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

O3 与 Ofast 的主要区别在于  -ffast-math 是针对浮点数进行运算的。

Profile-Guided Optimisations 和 Profile 有点像，对于金融来说是测不准的。

-funroll-loops 在 Clang-O2 就有这个优化，但基于 GCC 只有开了 Profile Guided 优化才会把循环展开，这种情况下如果希望强制展开，可以用 #pragma。

-march 要么=native，要么等于目标架构。

-flto 一般也是打开的，可以减少 binary size，跨文件单元进行优化。

irace 是一个开源工具，是把各个编译选项排列组合，你提供一个测试程序，看一看哪个性能最高。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

loop-vectorize 对于 int 是能提升性的，但如果把 type 改成 double，由于 floating 运算不支持结合律，loop vectorize 就做不了。那要如何进行优化呢？

如果你对浮点数的要求的精度运算没有那么高，可以打开  -ffast-math 下 -funsafe-math-optimizations 三个选项：

- -fassociative-math
- -fno-signed-zeros
- -fno-trapping-math

但要注意打开 -ffast-math 有可能产生一些精度问题，一定要对你的程序进行一些精确的测量，否则会出现一些莫名其妙的运算错误，对交易来说，这个运算错误肯定是非常致命的。

总结来说，开哪个编译并没有标准答案，我个人会开 Ofast 也会开  -march=native，需要结合你的具体项目需要。

4.

**低延迟软件设计与编码**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

由于我们相当于是一个 client，不会用多进程，线程也都是提前创建的，因为创建线程肯定是要进内核态，而且内核态开销比较大，线程池也不太用，静态链接会比动态链接有百分之几的性能提升，建议用静态链接。数据拷贝和数据共享也都尽可能避免。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

因为我们有数很多数据，即使是 critical 线程，critical path，系统刚启动时你总有一定的时间可以进行一些比较耗时的操作，你可以把这些计算量比较大的东西先算好，然后每来一些新的数据就可以用一些增量的方法来更新。最简单的，比如算一些均线、布林线、MACD 指标等都可以考虑怎样增量计算，毕竟预算量越小、指令越少，性能就越高，让代码尽可能减少间接层，慎用第三方库。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

运行时多态，通过 vptr 去找 vtable 会有一定的性能开销，另外如果是虚函数调用就没法 inline，这个带来的性能损失可能更大。这里有一些模板去解决这个问题，比如 CRTP、Policy based class design 之类。如果集成 path 上虚函数和具体函数只有一个实现的话，这个编译选项有可能会把虚函数间接的调用优化掉，还会有一个 vtable 的比较，我觉得这个可有可无，性能开销也不会很大，仅有一个实现的话还是可以考虑的。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这个案例中上面是基类，下面是派生类。基类里有一些 virtual function，这里用了一个 Strategy 设计模式，可能也是一个抽象类，运行时指向几个具体的类，通常写代码时这样写肯定是没有问题的，但如果这里面的触发至少有两次的虚函数都用开销，并且也不能 inline ，我们一般都会用 CRTP 这种模板的方式去解决。首先，基类 OnTick 会调用派生类的 OnTickImpl，这个 OMType\* \_om 也不再作为 Strategy 设计模式，直接继承下来并作为一个模板参数，在编译时就把它决议掉，然后在具体的类再去实现 OnTickImpl，这样就没有虚函数的开销，也可以 inline。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

我们做这个模板的时候有很多类型信息可能拿不到，所以就用一些 traits 的方式，比如各个接口之间用这个 concept 定一个 traits，每个接口都把这个 traits 实现好，基类就可以去根据这个 traits 去取派生类的一些信息。这里 if constexpr 也相当于一个编译态的 if，和 enable_if 很像，通过这个也可以针对类型做一些分支处理，这些在运行时都是没有任何开销的。enable_if 这有一个参数，来判断它是不是 bool Critical 线程，如果是，就直接用 write(msg) ， 因为有另外一个线程会 busy loop 去做这件事情，可能就需要 \_cv.notify。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

系统调用要尽可能避免，除非就是 vdso 支持的可以考虑派出。我们知道 vdso 就相当于是内核开辟了一块空间，方便系统调用拿数据。vdso 只支持几个时间相关的以及 getcpu 有限的几个。

rdtsc 是最方便的取时间的方式，而 clock 是真正的系统调用，它就不在 vdso 里，性能开销非常大。

打日志也是要非常谨慎，我们很多地方注意了低延迟，但其实打日志的延迟是非常大的，它开销主要是两部分：一部分是 format 开销，一部分是获取时间的开销 。format 开销有一种方式是编译的时候生成一些信息，运行时把这些 format 延迟推后，然后通过一些离线的工具，在生成 log 的时候再去做这个 format，比如说一些低开销的日志库就能做这些事情。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

动态内存分配有可能触发系统调用，并且会引发配置 fault，所以要尽量避免。避免的方式有很多，你可以用 placement new、memory pool。但 STL 及第三方库带来的内存分配很难避免，要避免的话，一是要提前分配内存，用 ring buffer 之类，map/set  数量少的话可以用 sorted array 替代。pool allocator 也可以实现内存池。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

减少内存分配就是让 Glibc 的内存分配了以后尽可能不回收，一旦回收给操作系统，下次再申请就比较麻烦。

这里有一个例子。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

vector 和 string 一样是可以提前分配内存的，你可以先 alloc 一块 memory，目的就是 presort，每隔 4096 都写一下，sort 完之后 clear，如果系统调整好了，即使是默认的，Glibc 也不会回收这块内存。然后下次你再一个个 push_back，也不会触发 page fault。你可以通过工具，包括这个 perf 来看  minor-faults  有多少来验证。这仅限于 Glibc，对于谷歌 tcmalloc 和 Facebook jemalloc，因为是互联网环境，它们对内存回收都比较积极，所以这个方法对于它们都不适用。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

string 的开销其实也是比较大的，但好处是它引起的开销就是堆上开销。它在堆上分配内存，在堆上分配一个差的数组，如果 std::string 比较小，就会在栈上分配内存，那速度就是比较快的。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

hashmap 也很关键，因为里面的数据也会触发堆内存分配，这个也是不可以接受的。因此不能用那种链式的，都是用线性搜索。如果一块连续的地址填满了一个 hashmap，就会到紧邻的位置去找。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

消息传递会用一些 lockfree queue 无锁编程，其实它内部也是有 lock 指令的，也是有开销的，所以如非必要也不建议用。如果极端情况下确实需要 lock

的话，用 spinlock，不要用 mutex/semaphore，因为 spinlock 是一种懵懂的状态，它不会进内核态等你的内核唤醒，所以它的开销还是比较小的。对于 mutex，现在是用 futex 优化，如果没有发生锁冲突，它也不会进内核态，但即使这样，mutex 的实现也比较复杂，开销也是比较大的。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这张图中我们可以看到 spinlock 开销还是比较低的，atomic 的操作也不高，当然原生的肯定是最高的。对于 mutax，我这里测的都是在没有资源竞争的情况下，这个数据已经非常不好了。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

代码尽可能少用 branch，比如这里是一个 lookup table，我们在 table 里写的这些 function  就可以减少一些 switch case 或者 else 的分支判断。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

写代码的时候，为了节省时间资源，往往都倾向于鼓励提前退出，都会把比较高的 if 放到前面。但对于我们的情况可能正好相反。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在低延迟场景下，大多数时候我们最终的信号是不会发出执行的，这种情况下如果让它过早退出，那这段很大的代码，包括数据，中间的 cache 都是冷的，那下次真正执行的效率就会比较低，这种是我们不想要的。

我们会在不 crash 的情况下，尽可能多执行代码，让它把一个完整的流程走完，不在意 CPU 时间浪费。在这里我们不是用或操作符，而是用按位操作符。用这种操作来换取 branch 只会有一个分支，就不会有三个分支了。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

分支预测基本上是  [[likely]], [[unlikely]]，相当于是给编译器的一个参考。但实际上它只是决定静态的分支预测，到实际 CPU 运行的时候会按照实际的 branch 是否 take 来决定下一次怎么预测。

这和刚才那个例子一样，可能 99.9%的情况下我们这个交易信号是触发不了的，那我永远走的是不触发的那个 branch，这样 CPU 也记住了，每次都会走到不触发的 branch。当我真正想要触发时是最需要低延迟的时候，这个 branch 就给我预测错，并且所有 cache 都是冷的。这种情况有一些技巧，比如，可以用一些假的程序尽可能地往下执行，到最后一步停。还有最简单的办法就是我这个订单真的发到柜台，只不过我把精度扩大一位，比如说有效价格是两位，我给它设三位，那这个单子发出去了也会被柜台拒绝，但我所有的 branch 都走到了。但这样做可能券商或期货公司不喜欢，因为会有大量的废单进来。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

异常如果不触发，对性能基本上没有什么影响，大家就没有什么心理负担。写代码的时候，作用域尽可能小，尽可能用 const ，连接性也尽可能低。这样目的就是让编译器知道更多，编译器知道更多的信息，就可能帮你做更多的优化。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

智能指针  unique_ptr 的开销基本可以忽略，开销本身是动态分配内存的开销，shared_ptr 里面有两个 atomic 变量，当然它底层不是用 atomic，而是用的更原始的操作去做的，但这个也会有性能开销，传参的时候也不要觉得它是智能指针可以自动加一减一就直接传了，还是按照引用的方式传比较好。

最后，C++ 20 的一些新特性对低延迟有一些帮助。atomic shared_ptr 目前内部还是用锁实现的，也是暂时不能用，希望以后可能有更优化的实现。

**关于我们**

CPP 与系统软件联盟是 Boolan 旗下连接 30 万+中高端 C++与系统软件研发骨干、技术经理、总监、CTO 和国内外技术专家的高端技术交流平台。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

2022 年 3 月 11-12 日，「全球 C++及系统软件技术大会」将于上海隆重召开，“C++之父”Bjarne Stroustrup 将发布主题演讲。来自海内外近 40 位专家将围绕包括：现代 C++、系统级软件、架构与设计演化、高性能与低时延、质量与效能、工程与工具链、嵌入式开发、分布式与网络应用 共 8 大主题，深度探讨最佳工程实践和前沿方法。大会官网：www.cpp-summit.org

**点击“阅读原文”获取大会演讲全套 PPT**

C++9

系统软件 6

C++ · 目录

上一篇从纳秒级优化谈 CPU 眼里的好代码下一篇使用 Boost 库实现自动数量的虚函数

阅读原文

阅读  3217

修改于 2022 年 01 月 21 日

​

高可用架构

_2022年01月27日 14:35_

以下文章来源于云加社区 ，作者郭剑池

\[

![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM4VrfJGRic9cMlydQkzsTsDFptqtib3k4CxI3TOVia4Nmicpw/0)

**云加社区**.

腾讯云官方社区公众号，汇聚技术开发者群体，分享技术干货，打造技术影响力交流社区。

\](https://mp.weixin.qq.com/s?\_\_biz=MzAwMDU1MTE1OQ==&mid=2653558778&idx=1&sn=1db38f8446fd7c2a61d8e3f6e18e4cde&chksm=81398662b64e0f740f255ec7ce063de77d0af1773a5551245bf77f56e9d9dc4c04335204ee50&mpshare=1&scene=24&srcid=0127BaM6Kfx9TAQbbTTz8LL3&sharer_sharetime=1643288448483&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d016f8d1b89e7960af3ab9f206e1417192d7116c6648a62fa1dd794253088d90a7d63d03cf312c69259980db51b78a3e70230b65f08b790fe58b02ba081a0aeac9300943dfb699aaf10704e7ef75673a9872373ee9892d70ea9b1b5f9dc758d9adb2ed238a6676977332542d46ae15bb7c82fe762a8908d44f&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQMnqYWyfc08Jz0GSWnwuzPhLmAQIE97dBBAEAAAAAAHxAFrCioGkAAAAOpnltbLcz9gKNyK89dVj05V7TT3wu6R%2FRiByzkv1s7N4nDD4e3yB1X4Csrcb%2F2cyykzsiijnCy%2BpdPgjyBckUq6yo1Kkt0nhqwdy1GOLRMGVWZfj9OphjowbMBY6lQhu246DXW3rdo6e9QgRuGN3JkeKmI%2BkBIN80ljqBVZQCwuxh4p11ydPoZyS6dJupURX9P0CsAsioHUftUgdKoIadKU2VGtlt3lhtJsDLacc1yjwFoDA0Xb2FyO%2BSn9kfHR9bYGwA2Vs2HMttRflyK4Gj&acctmode=0&pass_ticket=y6h8uiglmtc5ZXWKtvNdRnAk7gXvn4%2Bi9PIbaxnZ5HJGYtZcdtoCnVhu%2BArQCnZU&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1#)

导语 | 并发编程，可以说一直都是开发者们关注最多的主题之一。而Golang作为一个出道就自带“高并发”光环的编程语言，其并发编程的实现原理肯定是值得我们深入探究的。本文主要介绍Goroutine和channel的实现。

Go并发编程模型在底层是由操作系统所提供的线程库支撑的，这里先简要介绍一下线程实现模型的相关概念。

**一、线程的实现模型**

线程的实现模型主要有3个，分别是：**用户级线程模型**、**内核级线程模型和两级线程模型**。它们之间最大的差异在于用户线程与内核调度实体（KSE）之间的对应关系上。内核调度实体就是可以被操作系统内核调度器调度的对象，也称为内核级线程，是操作系统内核的最小调度单元。

### **（一）用户级线程模型**

![](https://mmbiz.qpic.cn/mmbiz_png/VY8SELNGe95xtElNsKdc7hfa48Ol5JBa48FNoFXdJPwrKBZhtqtFLCwKqQ9jFg2VoYw0ZMVnycbxn9yEu4Ddqw/640?wx_fmt=png&wxfrom=13&tp=wxpic)

用户线程与KSE为多对一（N:1）的映射关系。此模型下的线程由用户级别的线程库全权管理，线程库存储在进程的用户空间之中，这些线程的存在对于内核来说是无法感知的，所以这些线程也不是内核调度器调度的对象。一个进程中所有创建的线程都只和同一个KSE在运行时动态绑定，内核的所有调度都是基于用户进程的。对于线程的调度则是在用户层面完成的，相较于内核调度不需要让CPU在用户态和内核态之间切换，这种实现方式相比内核级线程模型可以做的很轻量级，对系统资源的消耗会小很多，上下文切换所花费的代价也会小得多。许多语言实现的协程库基本上都属于这种方式。但是，此模型下的多线程并不能真正的并发运行。例如，如果某个线程在I/O操作过程中被阻塞，那么其所属进程内的所有线程都被阻塞，整个进程将被挂起。

### **（二）内核级线程模型**

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

用户线程与KSE为一对一（1:1)的映射关系。此模型下的线程由内核负责管理，应用程序对线程的创建、终止和同步都必须通过内核提供的系统调用来完成，内核可以分别对每一个线程进行调度。所以，一对一线程模型可以真正的实现线程的并发运行，大部分语言实现的线程库基本上都属于这种方式。但是，此模型下线程的创建、切换和同步都需要花费更多的内核资源和时间，如果一个进程包含了大量的线程，那么它会给内核的调度器造成非常大的负担，甚至会影响到操作系统的整体性能。

### **（三）两级线程模型**

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

用户线程与KSE为多对多（N:M）的映射关系。两级线程模型吸收前两种线程模型的优点并且尽量规避了它们的缺点，区别于用户级线程模型，两级线程模型中的进程可以与多个内核线程KSE关联，也就是说一个进程内的多个线程可以分别绑定一个自己的KSE，这点和内核级线程模型相似；其次，又区别于内核级线程模型，它的进程里的线程并不与KSE唯一绑定，而是可以多个用户线程映射到同一个KSE，当某个KSE因为其绑定的线程的阻塞操作被内核调度出CPU时，其关联的进程中其余用户线程可以重新与其他KSE绑定运行。所以，两级线程模型既不是用户级线程模型那种完全靠自己调度的也不是内核级线程模型完全靠操作系统调度的，而是一种自身调度与系统调度协同工作的中间态，**即用户调度器实现用户线程到KSE的调度，内核调度器实现KSE到CPU上的调度**。

**二、Go的并发机制**

在Go的并发编程模型中，不受操作系统内核管理的独立控制流不叫用户线程或线程，而称为Goroutine。Goroutine通常被认为是协程的Go实现，实际上Goroutine并不是传统意义上的协程，传统的协程库属于用户级线程模型，而Goroutine结合Go调度器的底层实现上属于两级线程模型。

Go搭建了一个特有的两级线程模型。由Go调度器实现Goroutine到KSE的调度，由内核调度器实现KSE到CPU上的调度。Go的调度器使用G、M、P三个结构体来实现Goroutine的调度，也称之为**GMP模型**。

### **（一）GMP模型**

**G**：表示Goroutine。每个Goroutine对应一个G结构体，G存储Goroutine的运行堆栈、状态以及任务函数，可重用。当Goroutine被调离CPU时，调度器代码负责把CPU寄存器的值保存在G对象的成员变量之中，当Goroutine被调度起来运行时，调度器代码又负责把G对象的成员变量所保存的寄存器的值恢复到CPU的寄存器。

**M**：OS底层线程的抽象，它本身就与一个内核线程进行绑定，每个工作线程都有唯一的一个M结构体的实例对象与之对应，它代表着真正执行计算的资源，由操作系统的调度器调度和管理。M结构体对象除了记录着工作线程的诸如栈的起止位置、当前正在执行的Goroutine以及是否空闲等等状态信息之外，还通过指针维持着与P结构体的实例对象之间的绑定关系。

**P**：表示逻辑处理器。对G来说，P相当于CPU核，G只有绑定到P(在P的local runq中)才能被调度。对M来说，P提供了相关的执行环境(Context)，如内存分配状态(mcache)，任务队列(G)等。它维护一个局部Goroutine可运行G队列，工作线程优先使用自己的局部运行队列，只有必要时才会去访问全局运行队列，这可以大大减少锁冲突，提高工作线程的并发性，并且可以良好的运用程序的局部性原理。

一个G的执行需要P和M的支持。一个M在与一个P关联之后，就形成了一个有效的G运行环境（内核线程+上下文）。每个P都包含一个可运行的G的队列（runq）。该队列中的G会被依次传递给与本地P关联的M，并获得运行时机。

M与KSE之间总是一一对应的关系，一个M仅能代表一个内核线程。M与KSE之间的关联非常稳固，一个M在其生命周期内，会且仅会与一个KSE产生关联，而M与P、P与G之间的关联都是可变的，M与P也是一对一的关系，P与G则是一对多的关系。

- #### **G**

运行时，G在调度器中的地位与线程在操作系统中差不多，但是它占用了更小的内存空间，也降低了上下文切换的开销。它是Go语言在用户态提供的线程，作为一种粒度更细的资源调度单元，使用得当，能够在高并发的场景下更高效地利用机器的CPU。

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

g结构体部分源码（src/runtime/runtime2.go）：

```
type g struct {
```

gobuf中保存的内容会在调度器保存或恢复上下文时使用，其中栈指针和程序计数器会用来存储或恢复寄存器中的值，改变程序即将执行的代码。

atomicstatus字段存储了当前Goroutine的状态，Goroutine主要可能处于以下几种状态：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Goroutine的状态迁移是一个十分复杂的过程，触发状态迁移的方法也很多。这里主要介绍一下比较常见的**五种状态_Grunnable、\_Grunning、\_Gsyscall、\_Gwaiting和_Gpreempted**。

可以将这些不同的状态聚合成三种：等待中、可运行、运行中，运行期间会在这三种状态来回切换：

- **等待中**：Goroutine正在等待某些条件满足，例如：系统调用结束等，包括_Gwaiting、\_Gsyscall和_Gpreempted几个状态；

- **可运行**：Goroutine已经准备就绪，可以在线程运行，如果当前程序中有非常多的Goroutine，每个Goroutine就可能会等待更多的时间，即_Grunnable；

- **运行中**：Goroutine正在某个线程上运行，即_Grunning。

G常见的状态转换图：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

进入死亡状态的G可以重新初始化并使用。

- #### **M**

Go语言并发模型中的M是操作系统线程。调度器最多可以创建10000个线程，但是最多只会有GOMAXPROCS（P的数量）个活跃线程能够正常运行。在默认情况下，运行时会将 GOMAXPROCS设置成当前机器的核数，我们也可以在程序中使用runtime.GOMAXPROCS来改变最大的活跃线程数。

例如，对于一个四核的机器，runtime会创建四个活跃的操作系统线程，每一个线程都对应一个运行时中的runtime.m结构体。在大多数情况下，我们都会使用Go的默认设置，也就是线程数等于CPU数，默认的设置不会频繁触发操作系统的线程调度和上下文切换，所有的调度都会发生在用户态，由Go语言调度器触发，能够减少很多额外开销。

m结构体源码（部分）：

```
type m struct {
```

g0表示一个特殊的Goroutine，由Go运行时系统在启动之处创建，它会深度参与运行时的调度过程，包括Goroutine的创建、大内存分配和CGO函数的执行。curg是在当前线程上运行的用户Goroutine。

- #### **P**

调度器中的处理器P是线程和Goroutine的中间层，它能提供线程需要的上下文环境，也会负责调度线程上的等待队列，通过处理器P的调度，每一个内核线程都能够执行多个Goroutine，它能在Goroutine进行一些I/O操作时及时让出计算资源，提高线程的利用率。

P的数量等于GOMAXPROCS，设置GOMAXPROCS的值只能限制P的最大数量，对M和G的数量没有任何约束。当M上运行的G进入系统调用导致M被阻塞时，运行时系统会把该M和与之关联的P分离开来，这时，如果该P的可运行G队列上还有未被运行的G，那么运行时系统就会找一个空闲的M，或者新建一个M与该P关联，满足这些G的运行需要。因此，M的数量很多时候都会比P多。

p结构体源码（部分）：

```
type p struct {
```

P可能处于的状态如下：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**三、调度器**

两级线程模型中的一部分调度任务会由操作系统之外的程序承担。在Go语言中，调度器就负责这一部分调度任务。调度的主要对象就是G、M和P的实例。每个M（即每个内核线程）在运行过程中都会执行一些调度任务，他们共同实现了Go调度器的调度功能。

#### **（一）g0和m0**

运行时系统中的每个M都会拥有一个特殊的G，一般称为M的g0。M的g0不是由Go程序中的代码间接生成的，而是由Go运行时系统在初始化M时创建并分配给该M的。M的g0一般用于执行调度、垃圾回收、栈管理等方面的任务。M还会拥有一个专用于处理信号的G，称为gsignal。

除了g0和gsignal之外，其他由M运行的G都可以视为用户级别的G，简称用户G，g0和gsignal可称为系统G。Go运行时系统会进行切换，以使每个M都可以交替运行用户G和它的g0。这就是前面所说的“**每个M都会运行调度程序**”的原因。

除了每个M都拥有属于它自己的g0外，还存在一个runtime.g0。runtime.g0用于执行引导程序，它运行在Go程序拥有的第一个内核线程之中，这个线程也称为runtime.m0，runtime.m0的g0就是runtime.g0。

#### **（二）核心元素的容器**

上面讲了Go的线程实现模型中的**3个核心元素——G、M和P**，下面看看承载这些元素实例的容器：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

和G相关的四个容器值得我们特别注意，任何G都会存在于全局G列表中，其余四个容器只会存放当前作用域内的、具有某个状态的G。两个可运行的G列表中的G都拥有几乎平等的运行机会，只不过不同时机的调度会把G放在不同的地方，例如，从Gsyscall状态转移出来的G都会被放入调度器的可运行G队列，而刚刚被初始化的G都会被放入本地P的可运行G队列。此外，这两个可运行G队列之间也会互相转移G，例如，本地P的可运行G队列已满时，其中一半的G会被转移到调度器的可运行G队列中。

调度器的空闲M列表和空闲P列表用于存放暂时不被使用的元素实例。运行时系统需要时，会从中获取相应元素的实例并重新启用它。

#### **（三）调度循环**

调用runtime.schedule进入调度循环：

```
func schedule() {
```

runtime.schedule函数会从下面几个地方查找待执行的Goroutine：

- 为了保证公平，当全局运行队列中有待执行的Goroutine时，通过schedtick保证有一定几率会从全局的运行队列中查找对应的Goroutine；

- 从处理器本地的运行队列中查找待执行的Goroutine；

- 如果前两种方法都没有找到G，会通过findrunnable函数去其他P里面去“偷”一些G来执行，如果“偷”不到，就阻塞查找直到有可运行的G。

接下来由runtime.execute执行获取的Goroutine：

```
func execute(gp *g, inheritTime bool) {
```

当开始执行execute后，G会被切换到_Grunning状态，并将M和G进行绑定，最终调用runtime.gogo将Goroutine调度到当前线程上。runtime.gogo会从runtime.gobuf中取出runtime.goexit的程序计数器和待执行函数的程序计数器，并将：

- runtime.goexit的程序计数器被放到栈SP上；

- 待执行函数的程序计数器被放到了寄存器BX上。

```
MOVL gobuf_sp(BX), SP  // 将runtime.goexit函数的PC恢复到SP中
```

当Goroutine中运行的函数返回时，程序会跳转到runtime.goexit所在位置，最终在当前线程的g0的栈上调用runtime.goexit0函数，该函数会将Goroutine转换为_Gdead状态、清理其中的字段、移除Goroutine和线程的关联并调用runtime.gfput将G重新加入处理器的Goroutine空闲列表gFree中：

```
func goexit0(gp *g) {
```

最后runtime.goexit0会重新调用runtime.schedule触发新一轮的Goroutine调度，调度器从runtime.schedule开始，最终又回到runtime.schedule，这就是Go语言的调度循环。

**四、Channel**

Go中经常被人提及的一个设计模式：不要通过共享内存的方式进行通信，而是应该通过通信的方式共享内存。Goroutine之间会通过 channel传递数据，作为Go语言的核心数据结构和Goroutine之间的通信方式，channel是支撑Go语言高性能并发编程模型的重要结构。

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

channel在运行时的内部表示是runtime.hchan，该结构体中包含了用于保护成员变量的互斥锁，从某种程度上说，channel是一个用于同步和通信的有锁队列。hchan结构体源码：

```
type hchan struct {
```

waitq中连接的是一个sudog双向链表，保存的是等待中的Goroutine。

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

#### **（一）创建chan**

使用make关键字来创建管道，make(chan int，3)会调用到runtime.makechan函数中：

```
const (
```

上述代码根据channel中收发元素的类型和缓冲区的大小初始化runtime.hchan和缓冲区：

- 若缓冲区所需大小为0，就只会为hchan分配一段内存；

- 若缓冲区所需大小不为0且elem不包含指针，会为hchan和buf分配一块连续的内存；

- 若缓冲区所需大小不为0且elem包含指针，会单独为hchan和buf分配内存。

#### **（二）发送数据到chan**

发送数据到channel，ch\<-i会调用到runtime.chansend函数中，该函数包含了发送数据的全部逻辑：

```
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
```

block表示当前的发送操作是否是阻塞调用。如果channel为空，对于非阻塞的发送，直接返回false，对于阻塞的发送，将goroutine挂起，并且永远不会返回。对channel加锁，防止多个线程并发修改数据，如果channel已关闭，报错并中止程序。

runtime.chansend函数的执行过程可以分为以下三个部分：

- 当存在等待的接收者时，通过runtime.send直接将数据发送给阻塞的接收者；

- 当缓冲区存在空余空间时，将发送的数据写入缓冲区；

- 当不存在缓冲区或缓冲区已满时，等待其他Goroutine从channel接收数据。

- ##### **直接发送**

如果目标channel没有被关闭且recvq队列中已经有处于读等待的Goroutine，那么runtime.chansend会从接收队列 recvq中取出最先陷入等待的Goroutine并直接向它发送数据，注意，由于有接收者在等待，所以如果有缓冲区，那么缓冲区一定是空的：

```
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
```

直接发送会调用runtime.send函数：

```
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
```

sendDirect方法调用memmove进行数据的内存拷贝。goready方法将等待接收数据的Goroutine标记成可运行状态（Grunnable）并把该Goroutine发到发送方所在的处理器的runnext上等待执行，该处理器在下一次调度时会立刻唤醒数据的接收方。注意，只是放到了runnext中，并没有立刻执行该Goroutine。

- **发送到缓冲区**

如果缓冲区未满，则将数据写入缓冲区：

```
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
```

找到缓冲区要填充数据的索引位置，调用typedmemmove方法将数据拷贝到缓冲区中，然后重新设值sendx偏移量。

- **阻塞发送**

当channel没有接收者能够处理数据时，向channel发送数据会被下游阻塞，使用select关键字可以向channel非阻塞地发送消息：

```
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
```

对于非阻塞的调用会直接返回，对于阻塞的调用会创建sudog对象并将sudog对象加入发送等待队列。调用gopark将当前Goroutine转入waiting状态。调用gopark之后，在使用者看来向该channel发送数据的代码语句会被阻塞。

发送数据整个流程大致如下：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**注意**，发送数据的过程中包含几个会触发Goroutine调度的时机：

- 发送数据时发现从channel上存在等待接收数据的Goroutine，立刻设置处理器的runnext属性，但是并不会立刻触发调度；

- 发送数据时并没有找到接收方并且缓冲区已经满了，这时会将自己加入channel的sendq队列并调用gopark触发Goroutine的调度让出处理器的使用权。

#### **（三）从chan接收数据**

从channel获取数据最终调用到runtime.chanrecv函数：

```
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
```

当从一个空channel接收数据时，直接调用gopark让出处理器使用权。如果当前channel已被关闭且缓冲区中没有数据，直接返回。

runtime.chanrecv函数的具体执行过程可以分为以下三个部分：

- 当存在等待的发送者时，通过runtime.recv从阻塞的发送者或者缓冲区中获取数据；

- 当缓冲区存在数据时，从channel的缓冲区中接收数据；

- 当缓冲区中不存在数据时，等待其他Goroutine向channel发送数据。

- **直接接收**

当channel的sendq队列中包含处于发送等待状态的Goroutine时，调用runtime.recv直接从这个发送者那里提取数据。注意，由于有发送者在等待，所以如果有缓冲区，那么缓冲区一定是满的。

```
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
```

主要看一下runtime.recv的实现：

```
func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
```

该函数会根据缓冲区的大小分别处理不同的情况：

- 如果channel不存在缓冲区：直接从发送者那里提取数据。

- 如果channel存在缓冲区：

1. 将缓冲区中的数据拷贝到接收方的内存地址；

1. 将发送者数据拷贝到缓冲区，并唤醒发送者。

无论发生哪种情况，运行时都会调用goready将等待发送数据的Goroutine标记成可运行状态（Grunnable）并将当前处理器的runnext设置成发送数据的Goroutine，在调度器下一次调度时将阻塞的发送方唤醒。

- **从缓冲区接收**

如果channel缓冲区中有数据且发送者队列中没有等待发送的Goroutine时，直接从缓冲区中recvx的索引位置取出数据：

```
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
```

- **阻塞接收**

当channel的发送队列中不存在等待的Goroutine并且缓冲区中也不存在任何数据时，从管道中接收数据的操作会被阻塞，使用 select 关键字可以非阻塞地接收消息：

```
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
```

如果是非阻塞调用，直接返回。阻塞调用会将当前Goroutine封装成sudog，然后将sudog添加到等待接收队列中，调用gopark让出处理器的使用权并等待调度器的调度。

注意，接收数据的过程中包含几个会触发Goroutine调度的时机：

- 当channel为空时

- 当channel的缓冲区中不存在数据并且sendq中也不存在等待的发送者时

#### **（四）关闭chan**

关闭通道会调用到runtime.closechan方法：

```
func closechan(c *hchan) {
```

将recvq和sendq两个队列中的Goroutine加入到gList中，并清除所有sudog上未被处理的元素。最后将所有glist中的Goroutine加入调度队列，等待被唤醒。注意，发送者在被唤醒之后会panic。

总结一下发送/接收/关闭操作可能引发的结果：

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Goroutine+channel的组合非常强壮，两者的实现共同支撑起了Go语言的并发机制。

## **参考资料：**

1.Go并发编程实战

2.Go语言设计与实现

**作者简介**

!\[\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**郭剑池**

腾讯游戏后台开发工程师

腾讯游戏后台开发工程师，毕业于北京邮电大学。目前负责魂斗罗归来手游服务器端的相关开发工作。在学习和钻研Go语言的过程中，希望能和大家分享更多的心得体会，共同进步。

**参考阅读：**

- [事件驱动架构在 vivo 内容平台的实践](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653558735&idx=1&sn=f503421864acb826308c62e7d49b7481&chksm=81398657b64e0f417a4b694b1d682e1621c58f689002c718f59f5e23c4108566541b8ba338a2&scene=21#wechat_redirect)

- [带你彻底击溃跳表原理及其Golang实现！（内含图解）](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653558709&idx=1&sn=ab23e81819d9cd7f4e39696fee0812da&chksm=8139862db64e0f3b65475f7979c7550fb57986eb4b2588fc2b5ad37f3d1fcd451bd1e5250768&scene=21#wechat_redirect)

- [从0到1：美团端侧CDN容灾解决方案](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653558679&idx=1&sn=12edc6ac041ee80e8ec76adc4f63061d&chksm=8139860fb64e0f19e5c31782e7a595cd606a4c4a138eb40900553e9a7ec86561d615de0822a8&scene=21#wechat_redirect)

- [百度搜索中台新一代内容架构：FaaS化和智能化实战](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653558653&idx=1&sn=b25187f806a4f181d47e801821c1288c&chksm=813986e5b64e0ff3ebc70bff1d78db3be85e5586aecb08eb65a1a88161a527daa9a0cf39448d&scene=21#wechat_redirect)

- [五人基础架构组如何掌控千万DAU云原生架构](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653558635&idx=1&sn=36704fd67482209bb3a658fb7d99bc35&chksm=813986f3b64e0fe575e03a4891d91579689c466f0dfeb65f921e29130b869846190d8409c5a9&scene=21#wechat_redirect)

技术原创及架构实践文章，欢迎通过公众号菜单「联系我们」进行投稿。

**高可用架构**

**改变互联网的构建方式**

![](http://mmbiz.qpic.cn/mmbiz_png/8XkvNnTiapONl06YmHad4csRU93kcbJ76JIWzEAmOSVooibFHHkzfWzzkc7dpU4H06Wp9F6Z687vIghdawxvl47A/300?wx_fmt=png&wxfrom=19)

**高可用架构**

高可用架构公众号。

439篇原创内容

公众号

阅读 4244

​
